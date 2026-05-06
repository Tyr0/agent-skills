---
name: rails-testing
description: Use this skill whenever the user asks about testing in Ruby on Rails, including setting up Minitest, writing model tests, integration tests, system tests, mailer tests, or job tests. Also use it for questions about fixtures, test helpers, SimpleCov coverage, Capybara system tests, parallel test execution, or RuboCop-Minitest style enforcement. Triggers on 'how do I write a test', 'what is a fixture', 'how do I test a controller', 'how do I set up SimpleCov', 'how do I run system tests', or any question about the test/ directory structure.
---

# Rails Testing Reference (Minitest)

A dense reference for Rails testing with Minitest — the framework built into Rails. Covers setup, fixtures, all test types, coverage, and anti-patterns.

> **Framework:** Minitest (Rails default). No additional gems required for basic testing.

---

## Test Directory Structure

```
test/
├── controllers/          # ActionDispatch::IntegrationTest (preferred over functional tests)
├── fixtures/             # YAML fixture files
├── helpers/              # Custom test helper modules (auto-loaded via test_helper.rb)
├── integration/          # Multi-step request flows (ActionDispatch::IntegrationTest)
├── mailers/              # ActionMailer::TestCase
├── models/               # ActiveSupport::TestCase
├── system/               # ActionDispatch::SystemTestCase (Capybara)
├── validators/           # ActiveSupport::TestCase for custom validators
├── application_system_test_case.rb
└── test_helper.rb
```

---

## Test Helper Setup

```ruby
# test/test_helper.rb
require "simplecov"
SimpleCov.start "rails" do
  add_group "Validators", "app/validators"
  enable_coverage :branch
  primary_coverage :branch
end

ENV["RAILS_ENV"] ||= "test"
require_relative "../config/environment"
require "rails/test_help"

Rails.application.eager_load!

# Auto-load all test helpers
Dir[Rails.root.join("test", "helpers", "**", "*.rb")].each { |file| require file }

module ActiveSupport
  class TestCase
    parallelize(workers: :number_of_processors)

    parallelize_setup do |worker|
      SimpleCov.command_name "#{SimpleCov.command_name}-#{worker}"
    end

    parallelize_teardown do |worker|
      SimpleCov.result
    end

    fixtures :all
  end
end
```

---

## Writing Tests

### Test naming — use descriptive strings, not `def test_`

```ruby
# Good
test "should require a valid email address" do
  # ...
end

# Avoid
def test_email_validation
  # ...
end
```

### Core assertions

```ruby
assert expr                       # expr is truthy
assert_not expr                   # expr is falsy (prefer over assert !)
assert_equal expected, actual     # == equality
assert_not_equal a, b
assert_nil value
assert_not_nil value
assert_includes collection, item
assert_empty collection
assert_raises(ErrorClass) { ... }
assert_difference "Model.count", 1 { ... }     # count changes by 1
assert_difference "Model.count", -1 { ... }    # count decreases by 1
assert_no_difference "Model.count" { ... }
assert_difference -> { A.count } => 1, -> { B.count } => -1 { ... }  # multiple
```

---

## Model Tests

```ruby
# test/models/user_test.rb
require "test_helper"

class UserTest < ActiveSupport::TestCase
  test "should be valid with valid attributes" do
    user = users(:john_doe)
    assert user.valid?
  end

  test "should require email" do
    user = users(:john_doe)
    user.email = nil
    assert_not user.valid?
    assert_includes user.errors[:email], "can't be blank"
  end

  test "should destroy associated posts on deletion" do
    user = users(:john_doe)
    assert_difference "Post.count", -user.posts.count do
      user.destroy!
    end
  end
end
```

---

## Integration / Controller Tests

`ActionDispatch::IntegrationTest` is preferred for all request-level tests — it exercises the full stack including middleware.

```ruby
# test/controllers/articles_controller_test.rb
require "test_helper"

class ArticlesControllerTest < ActionDispatch::IntegrationTest
  test "should get index" do
    get articles_path
    assert_response :success
  end

  test "should create article" do
    assert_difference "Article.count", 1 do
      post articles_path, params: { article: { title: "Hello", body: "World" } }
    end
    assert_redirected_to article_path(Article.last)
  end

  test "should return 404 for missing article" do
    get article_path(id: 99999)
    assert_response :not_found
  end
end
```

### JSON API tests

```ruby
class API::ArticlesControllerTest < ActionDispatch::IntegrationTest
  test "should return articles as JSON" do
    get api_articles_path, as: :json
    assert_response :ok
    assert_equal "application/json", response.content_type.split(";").first

    body = response.parsed_body
    assert body.key?("articles")
  end

  test "should create article via API" do
    post api_articles_path,
      params: { article: { title: "Test" } },
      headers: { "Authorization" => "Bearer #{token}" },
      as: :json

    assert_response :created
  end
end
```

### Authentication in integration tests

```ruby
# Helper module for signing in
module AuthTestHelper
  def sign_in(user)
    post session_path, params: { email: user.email_address, password: "Password123!" }
  end
end

class ArticlesControllerTest < ActionDispatch::IntegrationTest
  include AuthTestHelper

  setup do
    sign_in users(:john_doe)
  end
end
```

---

## Fixtures

YAML fixtures are the Rails-native way to seed test data. Loaded automatically with `fixtures :all`.

```yaml
# test/fixtures/users.yml
john_doe:
  first_name: John
  last_name: Doe
  email_address: john.doe@example.com
  password_digest: <%= BCrypt::Password.create("Password123!") %>
  admin: false

chuck_norris:
  first_name: Chuck
  last_name: Norris
  email_address: chuck.norris@example.com
  password_digest: <%= BCrypt::Password.create("IDareYou!") %>
  admin: true
```

```yaml
# test/fixtures/posts.yml
hello_world:
  title: Hello World
  user: john_doe       # references users.yml by label (FK resolved automatically)
  published: true
  published_at: <%= 1.day.ago.to_fs(:db) %>
```

```ruby
# Access fixtures in tests
users(:john_doe)        # returns the User AR object
users(:john_doe).id     # resolved primary key
```

**Fixture rules:**
- Use `<%= %>` for dynamic values (timestamps, digests).
- Reference associations by label — Rails resolves the FK automatically.
- Prefer a small, stable fixture set; add fixtures as you add features.
- Do not compute expensive values inline — use `BCrypt::Password.create` only when auth matters.

---

## System Tests (Capybara)

System tests drive a real browser via Selenium. Use for critical user flows.

```ruby
# test/application_system_test_case.rb
require "test_helper"

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  driven_by :selenium, using: :chrome, screen_size: [1400, 1400]
end
```

```ruby
# test/system/articles_test.rb
require "application_system_test_case"

class ArticlesTest < ApplicationSystemTestCase
  test "creating an article" do
    visit new_article_path

    fill_in "Title", with: "My Article"
    fill_in "Body", with: "Article content"
    click_on "Create Article"

    assert_text "My Article"
    assert_current_path article_path(Article.last)
  end
end
```

---

## Mailer Tests

```ruby
# test/mailers/user_mailer_test.rb
require "test_helper"

class UserMailerTest < ActionMailer::TestCase
  test "welcome email" do
    user = users(:john_doe)
    mail = UserMailer.welcome(user)

    assert_equal "Welcome to the app", mail.subject
    assert_equal [user.email_address], mail.to
    assert_match user.first_name, mail.body.encoded
  end

  test "welcome email is delivered" do
    user = users(:john_doe)
    assert_difference "ActionMailer::Base.deliveries.size", 1 do
      UserMailer.welcome(user).deliver_now
    end
  end
end
```

In integration tests, assert on deliveries:

```ruby
assert_emails 1 do
  post users_path, params: { user: { email: "new@example.com" } }
end
```

---

## Job Tests

```ruby
# test/jobs/cleanup_job_test.rb
require "test_helper"

class CleanupJobTest < ActiveJob::TestCase
  test "should delete old records" do
    assert_difference "OldRecord.count", -1 do
      CleanupJob.perform_now
    end
  end

  test "should be enqueued on create" do
    assert_enqueued_with(job: WelcomeJob) do
      User.create!(email: "new@example.com", password: "Password1!")
    end
  end
end
```

---

## Custom Validator Tests

Test validators in isolation with an inline test model to avoid coupling to AR lifecycle:

```ruby
# test/validators/presence_validator_test.rb
require "test_helper"

class PresenceValidatorTest < ActiveSupport::TestCase
  class TestModel
    include ActiveModel::Validations
    attr_accessor :name
    validates :name, presence: true
    def initialize(name) = @name = name
  end

  test "should be invalid when name is blank" do
    model = TestModel.new("")
    assert_not model.valid?
    assert_includes model.errors[:name], "can't be blank"
  end

  test "should be valid when name is present" do
    model = TestModel.new("Alice")
    assert model.valid?
  end
end
```

---

## Custom Test Helpers

Extract shared setup into modules in `test/helpers/` (auto-loaded by `test_helper.rb`):

```ruby
# test/helpers/api_test_helper.rb
module APITestHelper
  TEST_USER_AGENT = "TestAgent/1.0"

  def api_headers(token: nil)
    headers = { "User-Agent" => TEST_USER_AGENT, "Accept" => "application/json" }
    headers["Authorization"] = "Bearer #{token}" if token
    headers
  end
end
```

Include per-test-class:

```ruby
class API::TokensControllerTest < ActionDispatch::IntegrationTest
  include APITestHelper
end
```

---

## Parallel Tests

Enabled by default in the generated `test_helper.rb`:

```ruby
parallelize(workers: :number_of_processors)
```

- Each worker gets its own database: `test-0`, `test-1`, etc. (created automatically).
- SimpleCov requires the parallel setup/teardown hooks to merge results (see test_helper.rb above).
- Fixtures are loaded per-worker — no shared state between workers.

---

## Code Coverage with SimpleCov

```ruby
# Gemfile
group :test do
  gem "simplecov", require: false
end
```

Configure in `test_helper.rb` before loading the app:

```ruby
require "simplecov"
SimpleCov.start "rails" do
  add_group "Validators", "app/validators"
  add_group "Services", "app/services"
  enable_coverage :branch     # track branch coverage (if/else, case)
  primary_coverage :branch    # report on branch coverage (not just line)
  minimum_coverage 90         # optional: fail CI if below threshold
end
```

Run: `bin/rails test` — SimpleCov generates `coverage/index.html` after the suite.

---

## RuboCop-Minitest

```ruby
# Gemfile
group :rubocop do
  gem "rubocop-minitest", require: false
end
```

```yaml
# .rubocop.yml
plugins:
  - rubocop-minitest

Rails/AssertNot:
  Include:
    - "**/test/**/*"

Rails/RefuteMethods:
  Include:
    - "**/test/**/*"

Minitest/UnreachableAssertion:
  Enabled: true
```

Key enforced rules:
- `assert_not` over `assert !`
- `assert_not_nil` over `refute_nil`
- No unreachable assertions after early returns

---

## Running Tests

```bash
bin/rails test                             # run all tests
bin/rails test test/models/               # run a directory
bin/rails test test/models/user_test.rb   # run a single file
bin/rails test test/models/user_test.rb:8 # run a single test by line
bin/rails test:system                     # run system tests only
bin/rails test:all                        # models + integration + system
```

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Mocking the database | Diverges from production behavior; misses real query bugs | Hit a real test DB; use transactions to roll back |
| No `fixtures :all` | Fixture helpers unavailable | Add `fixtures :all` to `ActiveSupport::TestCase` |
| `User.create` in tests instead of fixtures | Slow; non-deterministic; fixture associations break | Use fixtures for stable data; `create` only when testing creation logic |
| Testing implementation details (private methods) | Brittle; breaks on refactor | Test behavior through the public interface |
| System tests for every scenario | Slow Selenium startup for simple logic | Reserve system tests for critical UI flows; use integration tests for the rest |
| No assertions in a test body | Test always passes; silent breakage | Every test must have at least one `assert_*` |
| Calling `deliver_now` in tests without checking delivery | Email bugs silently pass | Use `assert_emails` or check `ActionMailer::Base.deliveries` |
| Giant fixture files | Hard to understand; slow to parse | Keep fixtures minimal and focused; 2-4 records per model is usually enough |
