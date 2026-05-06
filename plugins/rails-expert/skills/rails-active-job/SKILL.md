---
name: rails-active-job
description: Use this skill whenever the user asks about background jobs in Ruby on Rails, including creating jobs, enqueuing work, configuring queue adapters (Solid Queue, GoodJob, Sidekiq), retry behavior, discarding failures, scheduling jobs to run later, testing jobs, or managing job queues. Also use it for questions about `perform_later`, `perform_now`, `retry_on`, `discard_on`, job prioritization, or the difference between job adapters. Triggers on 'how do I run a background job', 'how do I configure Sidekiq', 'how do I retry a failed job', 'how do I test a job', or any question about ActiveJob or background processing.
---

# Rails Active Job Reference

A dense reference for background job processing with Active Job — Rails' unified interface over multiple queue backends.

> **Rails 8 default:** Solid Queue (DB-backed, no Redis required). Earlier apps commonly use Sidekiq or GoodJob.

---

## Job Anatomy

```ruby
# app/jobs/cleanup_expired_sessions_job.rb
class CleanupExpiredSessionsJob < ApplicationJob
  queue_as :default

  def perform(user_id: nil)
    scope = UserSession.where("expired_at < ?", Time.current)
    scope = scope.where(user_id: user_id) if user_id
    scope.delete_all
  end
end
```

- `ApplicationJob < ActiveJob::Base` — app-wide base class, add shared logic here (logging, error reporting).
- `queue_as` — sets the queue this job runs on. Override per-enqueue with `.set(queue:)`.
- `perform` arguments must be serializable: integers, strings, booleans, `GlobalID`-registered AR objects, arrays, hashes.

### Passing ActiveRecord objects

```ruby
# Good — GlobalID serializes/deserializes the record automatically
WelcomeJob.perform_later(user)           # serialized as "gid://app/User/42"

# Avoid — IDs are safe but require manual lookup in perform
WelcomeJob.perform_later(user_id: user.id)
```

If the record is deleted before the job runs, Active Job raises `ActiveJob::DeserializationError`. Handle this with `discard_on ActiveJob::DeserializationError` when record deletion is expected.

---

## Enqueuing Jobs

```ruby
# Enqueue to run ASAP
WelcomeJob.perform_later(user)

# Run synchronously (bypasses queue — useful in tests/rake tasks)
WelcomeJob.perform_now(user)

# Enqueue with options
WelcomeJob.set(wait: 5.minutes).perform_later(user)
WelcomeJob.set(wait_until: Date.tomorrow.noon).perform_later(user)
WelcomeJob.set(queue: :critical).perform_later(user)
WelcomeJob.set(priority: 10).perform_later(user)

# Chain set options
WelcomeJob.set(queue: :mailers, wait: 1.minute).perform_later(user)
```

### From a model callback

```ruby
class User < ApplicationRecord
  after_commit :enqueue_welcome_job, on: :create

  private

  def enqueue_welcome_job
    WelcomeJob.perform_later(self)
  end
end
```

Use `after_commit`, not `after_create` — ensures the record exists in the DB before the worker picks it up.

---

## Queue Adapters

### Solid Queue (Rails 8 default)

Database-backed. No Redis. Configured automatically in Rails 8.

```ruby
# config/application.rb (Rails 8)
config.active_job.queue_adapter = :solid_queue
```

Run the worker process:

```bash
bin/jobs                # foreground (dev)
bin/jobs start          # background (production)
```

In `Procfile.dev` for `bin/dev`:

```
web: bin/rails server
jobs: bin/jobs
```

### GoodJob (Rails 7 alternative)

Postgres-backed, thread-based. Runs inside the Rails process or standalone.

```ruby
gem "good_job"
```

```ruby
config.active_job.queue_adapter = :good_job
```

In-process (single-server):

```ruby
# config/environments/production.rb
config.good_job.execution_mode = :async
```

Standalone:

```bash
bundle exec good_job start
```

### Sidekiq

Redis-backed, process-based, high throughput.

```ruby
gem "sidekiq"
```

```ruby
config.active_job.queue_adapter = :sidekiq
```

```bash
bundle exec sidekiq
```

Sidekiq can also be used directly (bypassing Active Job) with `include Sidekiq::Job` for more control over retries and middleware.

### Async adapter (development / test)

```ruby
# Runs jobs in a thread pool within the Rails process — do not use in production
config.active_job.queue_adapter = :async
```

### Test adapter

```ruby
# config/environments/test.rb
config.active_job.queue_adapter = :test
# Jobs are not executed — they are recorded for assertion
```

---

## Retries and Failure Handling

### `retry_on` — retry on specific exceptions

```ruby
class ImportJob < ApplicationJob
  retry_on Net::TimeoutError, wait: :polynomially_longer, attempts: 5
  retry_on ActiveRecord::Deadlocked, wait: 2.seconds, attempts: 3

  def perform(file_id)
    # ...
  end
end
```

`wait` options:
- `:exponentially_longer` — 0s, 8s, 64s, ...
- `:polynomially_longer` — gentler backoff
- Fixed value: `5.seconds`
- Proc: `wait: ->(executions) { executions * 10.seconds }`

After exhausting `attempts`, the job is re-raised and handled by `discard_on` or the adapter's dead-letter queue.

### `discard_on` — silently drop specific exceptions

```ruby
class SendNotificationJob < ApplicationJob
  discard_on ActiveJob::DeserializationError   # record deleted before job ran
  discard_on Notification::AlreadySentError

  def perform(notification)
    notification.deliver!
  end
end
```

### `after_discard` — log or alert when a job is discarded (Rails 7.1+)

```ruby
class ApplicationJob < ActiveJob::Base
  after_discard do |job, exception|
    Rails.logger.warn "Discarded #{job.class}: #{exception.message}"
    Sentry.capture_exception(exception)
  end
end
```

---

## Callbacks

```ruby
class AuditedJob < ApplicationJob
  before_perform  { Rails.logger.info "Starting #{self.class}" }
  after_perform   { Rails.logger.info "Finished #{self.class}" }
  around_perform  do |job, block|
    start = Time.current
    block.call
    Rails.logger.info "#{job.class} took #{Time.current - start}s"
  end
end
```

---

## Mailer Jobs

`deliver_later` automatically enqueues an `ActionMailer::MailDeliveryJob`:

```ruby
UserMailer.welcome(user).deliver_later
UserMailer.welcome(user).deliver_later(wait: 5.minutes)
UserMailer.welcome(user).deliver_later(queue: :mailers)
```

---

## Queue Configuration

```ruby
# config/application.rb
config.active_job.default_queue_name = :default
config.active_job.queue_name_prefix = Rails.env   # => "production_default"
config.active_job.queue_name_delimiter = "_"       # default: "_"
```

### Priority

Lower numbers = higher priority (convention varies by adapter):

```ruby
class CriticalAlertJob < ApplicationJob
  queue_as :critical
  queue_with_priority 1
end
```

---

## Testing

Use `ActiveJob::TestCase` or include `ActiveJob::TestHelper`:

```ruby
require "test_helper"

class WelcomeJobTest < ActiveJob::TestCase
  test "delivers welcome email" do
    user = users(:john_doe)
    assert_emails 1 do
      WelcomeJob.perform_now(user)
    end
  end

  test "is enqueued on user creation" do
    assert_enqueued_with(job: WelcomeJob) do
      User.create!(email: "new@example.com", password: "Password1!")
    end
  end

  test "is enqueued on the mailers queue" do
    assert_enqueued_with(job: WelcomeJob, queue: "mailers") do
      User.create!(email: "new@example.com", password: "Password1!")
    end
  end
end
```

`assert_enqueued_jobs` helper:

```ruby
assert_enqueued_jobs 3 do
  Order.create_batch(3)
end

assert_no_enqueued_jobs do
  User.find(1).touch
end
```

Perform enqueued jobs immediately in integration tests:

```ruby
perform_enqueued_jobs do
  post users_path, params: { user: { email: "new@example.com" } }
end
assert_emails 1
```

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Passing entire AR objects as keyword args (not GlobalID) | Stale data if record changes between enqueue and execute | Pass the AR object directly — Active Job uses GlobalID |
| Serializing non-primitives (arbitrary objects, procs) | Raises `SerializationError` | Serialize only IDs and primitives; reconstruct objects in `perform` |
| Doing too much in one job | Slow, hard to retry partially | Split into focused jobs; chain via callbacks or a workflow gem |
| Using `after_create` instead of `after_commit` for enqueuing | Job runs before DB transaction commits; worker can't find record | Use `after_commit on: :create` |
| No `discard_on ActiveJob::DeserializationError` | Deleted records cause infinite retries | Discard when record deletion is expected |
| Using the `:async` adapter in production | Jobs are lost on process restart | Use Solid Queue, GoodJob, or Sidekiq in production |
| Long-running jobs blocking the queue | Starves other jobs on shared workers | Break into smaller jobs; use dedicated queues with worker concurrency limits |
| Calling `perform_now` in production code for latency-sensitive paths | Blocks the request thread | Use `perform_later`; only use `perform_now` in rake tasks or console |
