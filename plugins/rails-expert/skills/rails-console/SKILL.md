---
name: rails-console
description: Use this skill whenever the user asks about the Rails console, including how to open it, sandbox mode, switching environments, querying or manipulating data interactively, reloading code, using the console in production or on a remote server, IRB tips, or common console patterns for debugging and inspection.
---

# Rails Console Reference

A dense reference for using `bin/rails console` effectively in development, staging, and production.

---

## Opening the Console

```bash
bin/rails console          # development environment (default)
bin/rails console -e production   # specific environment
RAILS_ENV=staging bin/rails console
```

Short alias:
```bash
bin/rails c
```

### `bin/dev` vs `bin/rails console`

`bin/dev` starts the full development stack (web server, background workers, CSS/JS watchers) via Foreman/Overmind. It does **not** open a console. To open a console while `bin/dev` is running, open a second terminal and run `bin/rails console` — the console connects to the same database as the running dev server.

```bash
# Terminal 1
bin/dev          # starts web + workers

# Terminal 2
bin/rails c      # connects to the development database
```

If your app requires environment variables loaded by `bin/dev` (e.g., via `.env`), use `dotenv-rails` or load the env explicitly:

```bash
dotenv bin/rails console         # if using dotenv gem
```

---

## Sandbox Mode

Wraps the entire session in a transaction that is **rolled back on exit**. Safe for experimenting with data mutations in production.

```bash
bin/rails console --sandbox
bin/rails c -s
```

- All writes (create, update, destroy) are rolled back when you exit.
- The database sees the changes during the session (you can query them).
- Do NOT use sandbox for anything that fires external side-effects (emails, webhooks, jobs) — those are not rolled back.

---

## Essential IRB Commands

| Command | What it does |
|---|---|
| `reload!` | Reload all application code without restarting the console |
| `app` | Access the routing/request helper (see below) |
| `helper` | Access view helpers |
| `exit` or `quit` | Exit the console |
| `_` | The result of the last expression |
| `show-source ClassName` | Print source of a class/method (requires `pry` or recent IRB) |

---

## Querying Data

```ruby
# Find by primary key (raises ActiveRecord::RecordNotFound if missing)
User.find(1)

# Find by attribute (returns nil if missing)
User.find_by(email: "alice@example.com")

# All records
User.all

# Chained scopes
User.where(active: true).order(created_at: :desc).limit(10)

# Count
User.where(role: :admin).count

# Pluck specific columns (avoids instantiating AR objects — fast)
User.pluck(:id, :email)

# exists?
User.exists?(email: "alice@example.com")

# First / last
User.first
User.last(3)

# Inspect associations
user = User.find(1)
user.posts.count
user.posts.first
```

---

## Mutating Data

```ruby
# Create
user = User.create!(name: "Bob", email: "bob@example.com")

# Update
user.update!(name: "Robert")
User.where(active: false).update_all(archived: true)

# Destroy
user.destroy
User.where("created_at < ?", 1.year.ago).destroy_all

# Toggle a boolean
user.toggle!(:active)
```

Always prefer `!` variants (`save!`, `create!`, `update!`) in the console — they raise on failure instead of returning false silently.

---

## Inspecting the Application

```ruby
# Check what routes exist
app.url_helpers   # the route helpers module
Rails.application.routes.url_helpers

# Make a fake HTTP request
app.get "/users/1"
app.response.status    # => 200
app.response.body      # => HTML/JSON string

# Inspect current environment
Rails.env               # => "development"
Rails.env.production?   # => false
Rails.root              # => Pathname to app root

# Read config values
Rails.application.config.action_mailer.delivery_method

# List all loaded models
ActiveRecord::Base.subclasses.map(&:name)

# Inspect a model's columns
User.columns.map { |c| [c.name, c.type] }
User.column_names

# Check table name
User.table_name
```

---

## Working with Jobs and Mailers

```ruby
# Enqueue a job
MyJob.perform_later(user_id: 1)

# Run a job inline (synchronously, bypasses queue)
MyJob.new.perform(user_id: 1)

# Preview a mailer (does not send)
UserMailer.welcome(user).to_s

# Deliver a mailer now
UserMailer.welcome(user).deliver_now
```

---

## Reloading Code

After editing a file, reload without restarting:
```ruby
reload!
```

- Re-evaluates all autoloaded constants.
- Local variables in your session are **not** cleared.
- Use when iterating on a model or service object without restarting the console.

---

## Production Console Access

### Local Rails app
```bash
RAILS_ENV=production bin/rails console
```

### Heroku
```bash
heroku run bin/rails console --app my-app
```

### Fly.io
```bash
fly console                           # if console_command is set in fly.toml
fly ssh console -C "bin/rails console"  # direct SSH
```

### Kubernetes / Docker
```bash
kubectl exec -it <pod-name> -- bin/rails console
docker exec -it <container> bin/rails console
```

---

## IRB Configuration Tips

`~/.irbrc` customizations apply to all `rails console` sessions:

```ruby
# ~/.irbrc

# Better output formatting
require "irb/completion"

# Show results without having to puts
IRB.conf[:INSPECT_MODE] = true

# Increase history size
IRB.conf[:HISTORY_FILE] = "#{ENV['HOME']}/.irb_history"
IRB.conf[:SAVE_HISTORY] = 1000
```

Rails also respects `.irbrc` in the app root.

---

## Common Debugging Patterns

```ruby
# Print all SQL queries in real time
ActiveRecord::Base.logger = Logger.new($stdout)

# Silence SQL output
ActiveRecord::Base.logger = nil

# Time a block
Benchmark.measure { User.all.to_a }.real

# Check what SQL a scope generates (without executing)
User.where(active: true).to_sql

# Explain a query
User.where(active: true).explain

# Inspect an object's methods (filtered)
user.methods.grep(/email/)

# Pretty-print a hash or object
pp user.attributes
```

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Running destructive queries without sandbox | Data loss in production | Always use `--sandbox` unless you intend to commit changes |
| `User.all` on a large table | Loads millions of AR objects into memory; OOM | Use `.find_each`, `.pluck`, or `.limit` |
| `destroy_all` without a scope | Deletes every row | Always scope: `User.where(...).destroy_all` |
| Firing emails/jobs in sandbox | Side-effects are not rolled back | Use sandbox only for DB inspection; fire side-effects deliberately |
| Ignoring silent failures (save returns false) | Data not persisted | Use `save!` / `update!` / `create!` in console |
