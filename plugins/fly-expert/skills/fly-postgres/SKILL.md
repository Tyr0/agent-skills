---
name: fly-postgres
description: Use this skill whenever the user asks about setting up, connecting to, or managing Postgres on Fly.io, including creating a Fly Postgres cluster, attaching a database to a Rails or other app, DATABASE_URL configuration, running database migrations on Fly, proxying to a Fly Postgres database locally, connection pooling, backing up or restoring a Fly Postgres database, scaling a Postgres cluster, or troubleshooting database connection issues on Fly.io. Also use it for questions about Fly Managed Postgres (fly mpg) vs unmanaged Postgres clusters, or local development workflows that connect to a remote Fly Postgres database.
---

# Fly.io Postgres Reference

A dense reference for provisioning, connecting, and operating Postgres on Fly.io. Covers both unmanaged Fly Postgres clusters and Managed Postgres (MPG).

---

## Postgres Option Decision Guide

| Option | Use when | Command |
|---|---|---|
| **Fly Managed Postgres (MPG)** | Production; want Fly-supported backups, upgrades, HA | `fly mpg create` |
| **Fly Postgres (unmanaged)** | Want full control; dev/staging; cost-sensitive | `fly postgres create` |
| **External Postgres** | Already have Supabase/RDS/Neon/etc. | Set `DATABASE_URL` secret manually |

> Fly.io provides no support or guidance for unmanaged Postgres clusters. For production, prefer **Fly Managed Postgres**.

---

## 1. Fly Postgres Cluster (Unmanaged)

### Create a cluster
```bash
fly postgres create
```

Interactive prompts:
- **App name** — e.g. `my-rails-app-db` (leave blank for auto-generated)
- **Region** — match your app's `primary_region` for lowest latency
- **Configuration:**
  | Preset | Nodes | CPU | RAM | Disk | Use |
  |---|---|---|---|---|---|
  | Development | 1 | 1x shared | 256MB | 1GB | Local dev / staging |
  | Production HA | 3 | 2x shared | 4GB | 40GB | Production |
  | Custom | — | configurable | configurable | configurable | — |

**Save the output credentials immediately** — the password is shown once and cannot be retrieved:
```
Username:    postgres
Password:    <generated>
Hostname:    my-rails-app-db.internal
Flycast:     fdaa:...
Proxy port:  5432
Postgres URL: postgres://postgres:<password>@my-rails-app-db.flycast:5432
```

### Attach to your app
```bash
fly postgres attach my-rails-app-db --app my-rails-app
```

What `attach` does automatically:
1. Creates a new database named after your app (hyphens → underscores): `my_rails_app`
2. Creates a Postgres user named after your app: `my_rails_app`
3. Generates a random password for that user
4. Sets `DATABASE_URL` as a secret on your app pointing to the new database

**After attach, your app's `DATABASE_URL` is set and ready.** No manual configuration required.

### Detach
```bash
fly postgres detach my-rails-app-db --app my-rails-app
```

Does NOT drop the database or user — only removes the `DATABASE_URL` secret and platform association.

---

## 2. Fly Managed Postgres (MPG)

```bash
# Create managed cluster
fly mpg create

# List MPG clusters
fly mpg list

# Attach to app (sets DATABASE_URL secret)
fly mpg attach <mpg-name> --app <app-name>
```

MPG handles:
- Automated backups and point-in-time recovery
- Automatic minor version upgrades
- HA with automatic failover
- Fly.io support for operational issues

---

## 3. DATABASE_URL Format

```
postgres://USER:PASSWORD@HOSTNAME:5432/DATABASE?sslmode=disable
```

### Internal connection (app → DB within Fly network)
```
postgres://my_rails_app:password@my-rails-app-db.flycast:5432/my_rails_app?sslmode=disable
```

- Use `.flycast` hostname for internal connections (private networking, encrypted by WireGuard)
- `sslmode=disable` is safe on Fly's private network; connections are already WireGuard-encrypted

### Rails `database.yml` with `DATABASE_URL`
Rails automatically uses `DATABASE_URL` when set; no changes to `database.yml` required for basic use.

For connection pool tuning, override in `config/database.yml`:
```yaml
production:
  url: <%= ENV["DATABASE_URL"] %>
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  checkout_timeout: 5
  connect_timeout: 5
```

---

## 4. Database Migrations

### Recommended: `release_command` in `fly.toml`
```toml
[deploy]
  release_command = "bin/rails db:prepare"
```

- `db:prepare` = `db:create` (if needed) + `db:migrate` (idempotent)
- Runs in a **temporary Machine** using the new image, before traffic switches
- Deploy is **aborted** if the release command exits non-zero
- Runs as a single instance (no parallel migration risk)

### Alternative: manual migration via SSH
```bash
fly ssh console -C "bin/rails db:migrate"
```

### Verify migration status
```bash
fly ssh console -C "bin/rails db:version"
fly ssh console -C "bin/rails db:migrate:status"
```

---

## 5. Connecting Locally (Proxy)

Connect to your Fly Postgres database from your local machine for development or debugging.

### Method 1: flyctl proxy (recommended)
```bash
# Forward local port 5432 to Fly Postgres
fly proxy 5432 -a my-rails-app-db

# If local port 5432 is already in use
fly proxy 15432:5432 -a my-rails-app-db
```

Then connect:
```bash
psql postgres://postgres:<password>@localhost:5432
# Or with alternate port:
psql postgres://postgres:<password>@localhost:15432
```

### Method 2: flyctl postgres connect (interactive psql)
```bash
fly postgres connect -a my-rails-app-db
```

Opens a `psql` shell directly — no local Postgres client required beyond `flyctl`.

### Method 3: Use proxy for local Rails development
```bash
# Terminal 1: start proxy
fly proxy 5432 -a my-rails-app-db

# Terminal 2: run Rails pointing to proxied DB
DATABASE_URL=postgres://my_rails_app:<password>@localhost:5432/my_rails_app bin/rails server
```

---

## 6. Operational Commands

```bash
# List all Postgres apps
fly postgres list

# Status of a cluster
fly status -a my-rails-app-db

# View Postgres logs
fly logs -a my-rails-app-db

# List users
fly postgres users list -a my-rails-app-db

# Create a new user
fly postgres users create -a my-rails-app-db

# List databases
fly postgres db list -a my-rails-app-db

# SSH into the Postgres VM directly
fly ssh console -a my-rails-app-db
# Then: psql -U postgres
```

---

## 7. Backups and Snapshots

### Unmanaged Fly Postgres
```bash
# List snapshots (automatic daily snapshots)
fly volumes list -a my-rails-app-db
fly volumes snapshots list <volume-id>

# Restore from snapshot
fly volumes create --snapshot-id <snapshot-id> --size 10
```

### Manual pg_dump via proxy
```bash
# Start proxy
fly proxy 5432 -a my-rails-app-db &

# Dump
pg_dump postgres://my_rails_app:<password>@localhost:5432/my_rails_app > backup.sql

# Restore to a different DB
psql postgres://my_rails_app:<password>@localhost:5432/my_rails_app_staging < backup.sql
```

### Managed Postgres (MPG)
Point-in-time recovery and backup management are handled by Fly — use the dashboard or `fly mpg` commands.

---

## 8. Scaling and High Availability

```bash
# Scale VM size for the Postgres cluster
fly scale vm shared-cpu-2x -a my-rails-app-db
fly scale vm performance-1x --memory 4096 -a my-rails-app-db

# Enable scale-to-zero for development (single-node only)
fly scale count 0 -a my-rails-app-db   # stops all; starts on next connection
```

### HA Configuration
- Use 3-node setup (1 primary + 2 replicas) for production
- Fly Postgres uses **Stolon** for HA and automatic failover
- Replicas serve read-only queries on port 5433

```
# Read replica connection (port 5433)
postgres://my_rails_app:<password>@my-rails-app-db.flycast:5433/my_rails_app
```

Rails multi-database config for read replicas:
```yaml
production:
  primary:
    url: <%= ENV["DATABASE_URL"] %>
  primary_replica:
    url: <%= ENV["DATABASE_REPLICA_URL"] %>
    replica: true
    database_tasks: false
```

---

## 9. Connection Pooling

Rails' built-in connection pool (`pool` in `database.yml`) is sufficient for most apps.

For high concurrency, add **PgBouncer** (transaction-mode pooler):

```bash
# Fly Postgres has a built-in PgBouncer on port 6432
postgres://my_rails_app:<password>@my-rails-app-db.flycast:6432/my_rails_app
```

| Port | Mode | Use |
|---|---|---|
| 5432 | Direct Postgres | Migrations, long transactions, LISTEN/NOTIFY |
| 5433 | Read replica | Read-heavy queries |
| 6432 | PgBouncer (transaction) | High-concurrency web requests |

**Do not use PgBouncer (port 6432) for:**
- `db:migrate` — use port 5432 (migrations need persistent connections)
- ActiveRecord advisory locks (`with_advisory_lock`) — require session mode
- `LISTEN`/`NOTIFY` — require persistent connections

---

## 10. Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Running `db:migrate` in `ENTRYPOINT` | Runs on every Machine start; races on multi-instance deploys | Use `release_command = "bin/rails db:prepare"` |
| Committing the Postgres password | Security exposure | Always use `fly secrets set DATABASE_URL=...` |
| Connecting with `.internal` hostname from local machine | `.internal` only resolves inside Fly network | Use `fly proxy` for local access |
| Using PgBouncer port for migrations | Transactions pooled; migrations may fail | Use port 5432 for `release_command` |
| Single-node Postgres in production | No failover; downtime on node failure | Use 3-node HA cluster or Managed Postgres |
| Ignoring `pool` setting | Default `pool: 5` may be too low or too high | Set `pool` = `WEB_CONCURRENCY * RAILS_MAX_THREADS` |
| Forgetting `sslmode=disable` on internal connections | Connection refused or TLS handshake failures | Add `?sslmode=disable` to internal `DATABASE_URL` |
