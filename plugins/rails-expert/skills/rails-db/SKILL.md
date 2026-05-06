---
name: rails-db
description: Use this skill whenever the user asks about Rails database management, including running or rolling back migrations, checking migration status, seeding the database, resetting or dropping the database, schema loading, or any `bin/rails db:*` task. Also use it for questions about the schema.rb vs structure.sql choice, multi-database setups, annotating models with schema, or troubleshooting common migration errors.
---

# Rails Database Management Reference

A dense reference for all `bin/rails db:*` tasks and database lifecycle management.

---

## Task Quick Reference

| Task | What it does |
|---|---|
| `db:create` | Create the database(s) defined in `database.yml` |
| `db:drop` | Drop the database(s) |
| `db:migrate` | Run pending migrations |
| `db:rollback` | Revert the last migration (or last N with `STEP=N`) |
| `db:migrate:status` | Show each migration and whether it has run |
| `db:version` | Print the current schema version (timestamp of last migration) |
| `db:seed` | Run `db/seeds.rb` |
| `db:schema:load` | Load `db/schema.rb` into the database (skips migrations) |
| `db:schema:dump` | Regenerate `db/schema.rb` from the live database |
| `db:structure:load` | Load `db/structure.sql` (for `config.active_record.schema_format = :sql`) |
| `db:structure:dump` | Regenerate `db/structure.sql` |
| `db:reset` | `db:drop` + `db:setup` |
| `db:setup` | `db:create` + `db:schema:load` + `db:seed` |
| `db:prepare` | Create if missing + `db:migrate` (idempotent; use in CI and release commands) |
| `db:purge` | Truncate all tables without dropping the database |
| `db:environment:set` | Write `ar_internal_metadata` to mark the current Rails env |

---

## Migrations

### Run pending migrations
```bash
bin/rails db:migrate
```

Run for a specific environment:
```bash
RAILS_ENV=production bin/rails db:migrate
```

Migrate to a specific version (timestamp):
```bash
bin/rails db:migrate VERSION=20240101120000
```

### Roll back
```bash
bin/rails db:rollback          # revert last migration
bin/rails db:rollback STEP=3   # revert last 3 migrations
```

Roll back to a specific version:
```bash
bin/rails db:migrate:down VERSION=20240101120000
```

Re-run a specific migration (down then up):
```bash
bin/rails db:migrate:redo VERSION=20240101120000
bin/rails db:migrate:redo STEP=2   # redo last 2
```

### Check status
```bash
bin/rails db:migrate:status
```

Output:
```
 Status   Migration ID    Migration Name
--------------------------------------------------
   up     20240101000001  Create users
   up     20240215000001  Add email to users
  down    20240310000001  Add index on users email
```

### Migration file anatomy
```ruby
class AddIndexOnUsersEmail < ActiveRecord::Migration[7.2]
  def change
    add_index :users, :email, unique: true
  end
end
```

- Use `change` when the migration is reversible (Rails auto-generates `down`).
- Use `up` / `down` explicitly for irreversible operations (e.g., `execute`, data transforms).

### Safe migration practices
| Pattern | Notes |
|---|---|
| `add_column` with a default | In Postgres 11+, safe — no table rewrite. In older versions, consider `add_column` then `change_column_default` separately |
| `remove_column` | Deploy code that no longer references the column first; then remove |
| Adding a NOT NULL constraint | Add column as nullable, backfill, then add constraint — never one-step on large tables |
| `add_index` | Use `algorithm: :concurrently` in Postgres to avoid table lock. Requires `disable_ddl_transaction!` |

```ruby
class AddIndexConcurrently < ActiveRecord::Migration[7.2]
  disable_ddl_transaction!

  def change
    add_index :users, :email, algorithm: :concurrently
  end
end
```

---

## Schema Format

`db/schema.rb` (default) — Ruby DSL, database-agnostic, loads fast.

`db/structure.sql` — Raw SQL, preserves triggers, custom types, views, functions.

To switch to SQL format, in `config/application.rb`:
```ruby
config.active_record.schema_format = :sql
```

**Commit schema files to version control.** When in doubt, commit the regenerated file after every migration run.

---

## Seeding

```bash
bin/rails db:seed
```

`db/seeds.rb` is plain Ruby. Use `find_or_create_by` to make seeds idempotent:

```ruby
# db/seeds.rb
User.find_or_create_by(email: "admin@example.com") do |u|
  u.password = "changeme"
  u.role = :admin
end
```

Run seed after schema load:
```bash
bin/rails db:setup   # create + schema:load + seed
```

Only seed (no create/migrate):
```bash
bin/rails db:seed
```

---

## Database Lifecycle

### Development reset (destructive)
```bash
bin/rails db:reset   # drop + create + schema:load + seed
```

### Purge without dropping (faster for large schemas)
```bash
bin/rails db:purge db:schema:load db:seed
```

### CI setup (idempotent)
```bash
bin/rails db:prepare   # creates if needed, then migrates
```

---

## Multi-Database

Rails 6+ supports multiple databases. Tasks accept a `:<db>` suffix:

```bash
bin/rails db:migrate                    # all databases
bin/rails db:migrate:primary            # primary only
bin/rails db:migrate:animals            # named db only
bin/rails db:rollback:primary STEP=1
bin/rails db:migrate:status:animals
```

`database.yml` for multiple databases:
```yaml
development:
  primary:
    adapter: postgresql
    database: myapp_development
  animals:
    adapter: postgresql
    database: myapp_animals_development
    migrations_paths: db/animals_migrate
```

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `PendingMigrationError` | Migrations haven't been run | `bin/rails db:migrate` |
| `ActiveRecord::NoDatabaseError` | Database doesn't exist | `bin/rails db:create db:migrate` |
| `already exists` on `db:create` | Database was already created | Safe to ignore; use `db:prepare` to avoid |
| Migration stuck (lock) | Another process holds a DB lock | Kill the other process; check `pg_stat_activity` |
| Schema mismatch in test env | `db/schema.rb` not loaded | `bin/rails db:test:prepare` or `RAILS_ENV=test bin/rails db:schema:load` |
| `down` migration fails | `change` method not reversible | Implement explicit `up`/`down` methods |
