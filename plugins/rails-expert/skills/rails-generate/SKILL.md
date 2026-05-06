---
name: rails-generate
description: Use this skill whenever the user asks about the `bin/rails generate` command (or `bin/rails g`), including generating models, controllers, migrations, scaffolds, mailers, jobs, channels, concerns, serializers, or any other Rails generator. Also use it for questions about generator options and flags, undoing a generator with `bin/rails destroy`, creating custom generators, or understanding what files a generator creates.
---

# Rails Generate Reference

A dense reference for `bin/rails generate` (alias: `bin/rails g`). Covers the most common built-in generators, their options, and how to undo them.

---

## Generator Basics

```bash
bin/rails generate <generator> <arguments> [options]
bin/rails g <generator> <arguments> [options]   # alias
```

**Dry run** — see what files would be created without writing anything:
```bash
bin/rails g model User name:string --dry-run
bin/rails g model User name:string -p          # -p is short for --pretend
```

**Undo a generator** — destroy all files it created:
```bash
bin/rails destroy model User
bin/rails d model User                         # alias
```

**List all available generators:**
```bash
bin/rails generate --help
bin/rails g                                    # same
```

---

## Model

```bash
bin/rails g model <ModelName> [field:type ...]
```

```bash
bin/rails g model User name:string email:string:uniq age:integer active:boolean
```

Creates:
- `app/models/user.rb`
- `db/migrate/<timestamp>_create_users.rb`
- `test/models/user_test.rb` (or `spec/models/user_spec.rb` with RSpec)
- `test/fixtures/users.yml`

### Field types

| Type | Column |
|---|---|
| `string` | `VARCHAR` |
| `text` | `TEXT` |
| `integer` | `INTEGER` |
| `bigint` | `BIGINT` |
| `float` | `FLOAT` |
| `decimal` | `DECIMAL` |
| `boolean` | `BOOLEAN` |
| `date` | `DATE` |
| `datetime` | `DATETIME` / `TIMESTAMP` |
| `time` | `TIME` |
| `binary` | `BLOB` |
| `json` | `JSON` / `JSONB` (Postgres) |
| `references` / `belongs_to` | Foreign key column + index |

### Field modifiers

Append modifiers with `{modifier}`:
```bash
bin/rails g model User email:string{uniq} status:string{null:false}
```

| Modifier | Effect |
|---|---|
| `uniq` | Adds a unique index |
| `null:false` | Adds NOT NULL constraint in migration |
| `index` | Adds a plain index |

### Associations

```bash
bin/rails g model Post title:string body:text user:references
```

`user:references` creates a `user_id:bigint` column, a `belongs_to :user` in the model, and an index on `user_id`.

### Skipping files

```bash
bin/rails g model User name:string --no-migration   # skip migration
bin/rails g model User name:string --skip-test-framework
```

---

## Migration

```bash
bin/rails g migration <MigrationName> [field:type ...]
```

Rails infers the migration action from the name:

| Name pattern | Generated action |
|---|---|
| `AddXxxToYyy` | `add_column :yyy, :xxx` |
| `RemoveXxxFromYyy` | `remove_column :yyy, :xxx` |
| `CreateYyy` | `create_table :yyy` |
| `AddIndexToYyy` | `add_index :yyy` |
| `CreateJoinTableXxxYyy` | `create_join_table :xxx, :yyy` |

```bash
# Add columns
bin/rails g migration AddEmailToUsers email:string
bin/rails g migration AddTimestampsToOrders shipped_at:datetime completed_at:datetime

# Remove a column
bin/rails g migration RemoveAgeFromUsers age:integer

# Add an index
bin/rails g migration AddIndexToUsersEmail   # add manually in generated file

# Rename a column (no auto-inference; write manually)
bin/rails g migration RenameUsernameToHandle
# Then edit: rename_column :users, :username, :handle

# Rename a table
bin/rails g migration RenameUsersToMembers
# Then edit: rename_table :users, :members
```

Always run after generating:
```bash
bin/rails db:migrate
```

---

## Controller

```bash
bin/rails g controller <ControllerName> [action ...]
```

```bash
bin/rails g controller Users index show new create edit update destroy
```

Creates:
- `app/controllers/users_controller.rb`
- `app/views/users/index.html.erb`, `show.html.erb`, etc.
- `test/controllers/users_controller_test.rb`
- Route helpers (does NOT add resourceful routes — do that manually in `routes.rb`)

For API controllers (no views):
```bash
bin/rails g controller Api::V1::Users --no-template-engine --skip-template-engine
```

---

## Scaffold

Generates the full CRUD stack: model + migration + controller + views + routes.

```bash
bin/rails g scaffold Post title:string body:text published:boolean user:references
```

Creates:
- Model + migration
- `app/controllers/posts_controller.rb` (full CRUD)
- `app/views/posts/` (index, show, new, edit, _form partials)
- Route: adds `resources :posts` to `config/routes.rb`
- Tests and fixtures

**For JSON APIs:**
```bash
bin/rails g scaffold Post title:string body:text --api
```

Skips views; generates controller with `render json:` responses only.

**Scaffold resource (no model/migration — assumes table exists):**
```bash
bin/rails g scaffold_controller Post title:string body:text
```

---

## Resource

Like scaffold but only adds the route and controller stub — no views, no model:
```bash
bin/rails g resource Post title:string body:text
```

---

## Mailer

```bash
bin/rails g mailer UserMailer welcome password_reset
```

Creates:
- `app/mailers/user_mailer.rb`
- `app/views/user_mailer/welcome.html.erb` + `welcome.text.erb`
- `app/views/user_mailer/password_reset.html.erb` + `password_reset.text.erb`
- `test/mailers/user_mailer_test.rb`
- `test/mailers/previews/user_mailer_preview.rb`

---

## Job

```bash
bin/rails g job ProcessPayment
```

Creates:
- `app/jobs/process_payment_job.rb`
- `test/jobs/process_payment_job_test.rb`

With a specific queue:
```bash
bin/rails g job ProcessPayment --queue urgent
```

---

## Channel (Action Cable)

```bash
bin/rails g channel Chat speak
```

Creates:
- `app/channels/chat_channel.rb`
- `app/javascript/channels/chat_channel.js` (or `.coffee`)

---

## Concern

```bash
bin/rails g concern Archivable          # model concern
bin/rails g concern Authenticatable     # controller concern (place manually)
```

Creates `app/models/concerns/archivable.rb` with the `ActiveSupport::Concern` skeleton.

---

## Serializer

If using `ActiveModel::Serializers`:
```bash
bin/rails g serializer User name email role
```

---

## Dockerfile (Rails 7.1+)

```bash
bin/rails generate dockerfile
```

Generates a production-ready multi-stage `Dockerfile`, `bin/docker-entrypoint`, `.dockerignore`, and `config/dockerfile.yml`.

Flags:
| Flag | Effect |
|---|---|
| `--postgresql` | Add `libpq5` runtime dep |
| `--redis` | Add Redis client |
| `--sidekiq` | Configure Sidekiq entrypoint |
| `--node` | Add Node.js for JS bundling |
| `--swap=512` | Add swap space (MB) |

---

## Custom Generators

Create a generator skeleton:
```bash
bin/rails g generator MyThing
```

Creates `lib/generators/my_thing/my_thing_generator.rb`. Generators inherit from `Rails::Generators::Base` and use Thor actions.

---

## Common Flags (All Generators)

| Flag | Effect |
|---|---|
| `--skip-namespace` | Don't namespace the generated files |
| `--force` | Overwrite existing files without prompting |
| `--quiet` / `-q` | Suppress output |
| `--pretend` / `-p` | Dry run — show what would be created |
| `--no-test-framework` | Skip test files |
| `--no-helper` | Skip helper file (controllers) |
| `--no-assets` | Skip asset files (controllers) |
| `--api` | Generate API-mode files (no views) |

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Using `scaffold` in a mature app | Overwrites existing routes, controller, views | Use `scaffold_controller` or `g model` + `g migration` individually |
| Committing without reviewing generated files | Generated code is a starting point, not final | Always review and edit before committing |
| Generating without `--dry-run` on an existing app | Risk of overwriting existing files | Use `-p` first to preview |
| Forgetting `bin/rails db:migrate` after `g model` | Schema not applied | Always migrate immediately after generating a model or migration |
| Not using `references` for associations | Missing index on foreign key | Use `field:references` or add `add_index` manually |
