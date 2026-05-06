---
name: fly-rails-deployment
description: Use this skill whenever the user asks about deploying Ruby on Rails to Fly.io, configuring fly.toml for Rails, using Docker with Rails on Fly, setting up environment variables or secrets on Fly.io, configuring Sidekiq or background workers on Fly, setting up health checks, running rake tasks or Rails console on Fly, scaling Rails apps on Fly.io, or troubleshooting Rails deployments on Fly. Also use it for questions about local Docker development that mirrors a Fly.io Rails deployment, multi-process setups (web + worker), Action Cable, Redis on Fly, or Fly.io Dockerfile generation via `bin/rails generate dockerfile`.
---

# Fly.io Rails Deployment Reference

A dense reference for deploying and operating Ruby on Rails applications on Fly.io with Docker. Covers the full workflow from initial launch through production operations.

---

## Deployment Priority Order

1. `fly launch` — detect, scaffold, first deploy
2. Commit generated files (`Dockerfile`, `fly.toml`, `bin/docker-entrypoint`, `.dockerignore`, `config/dockerfile.yml`)
3. Attach Postgres and set secrets before first deploy if skipped at launch
4. Verify `release_command` runs migrations (`bin/rails db:prepare`)
5. Add health check endpoint (`/up` is Rails default)
6. Add process groups for workers before scaling

---

## 1. Initial Setup

### Install flyctl
```bash
brew install flyctl          # macOS
curl -L https://fly.io/install.sh | sh   # Linux/CI
fly auth login
```

### Launch a new Rails app
```bash
fly launch
```

`fly launch` auto-detects Rails and:
- Generates `Dockerfile`, `fly.toml`, `bin/docker-entrypoint`, `.dockerignore`, `config/dockerfile.yml`
- Prompts for org, region, machine size
- Optionally creates a Postgres cluster and attaches it (sets `DATABASE_URL` secret)
- Builds and deploys immediately if you accept

**All five generated files must be committed to version control.**

### Redeploy after changes
```bash
fly deploy
```

`fly deploy` builds the Docker image remotely (on Fly builders), pushes it, runs the `release_command`, then updates Machines.

---

## 2. Generated Files

### `Dockerfile`
Rails' built-in generator produces a production-ready multi-stage Dockerfile:

```dockerfile
# Stage 1: build gems and assets
FROM ruby:3.3-slim AS build
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev nodejs npm
WORKDIR /rails
COPY Gemfile Gemfile.lock ./
RUN bundle install --without development test
COPY . .
RUN bundle exec rails assets:precompile

# Stage 2: minimal runtime image
FROM ruby:3.3-slim AS final
RUN apt-get update -qq && apt-get install -y libpq5
WORKDIR /rails
COPY --from=build /usr/local/bundle /usr/local/bundle
COPY --from=build /rails /rails
ENTRYPOINT ["/rails/bin/docker-entrypoint"]
EXPOSE 3000
CMD ["./bin/rails", "server", "-b", "0.0.0.0"]
```

**Regenerate when dependencies change:**
```bash
bin/rails generate dockerfile
```

Common generator flags:
| Flag | Effect |
|---|---|
| `--postgresql` | Add `libpq5` runtime dependency |
| `--redis` | Add Redis client |
| `--sidekiq` | Add Sidekiq worker entrypoint |
| `--nginx` | Add nginx for asset serving |
| `--node` | Add Node.js for JS bundling |
| `--swap=512` | Add swap space (MB) for memory spikes |
| `--tigris` | Add Tigris object storage support |
| `--ci` | Build for CI environments |

### `bin/docker-entrypoint`
Runs database preparation on every container start (idempotent):
```bash
#!/bin/bash -e
if [ "${*}" == "./bin/rails server" ]; then
  ./bin/rails db:prepare
fi
exec "${@}"
```

Remove `db:prepare` from entrypoint if you use `release_command` in `fly.toml` instead (preferred for migrations).

### `config/dockerfile.yml`
Tracks generator preferences so `bin/rails generate dockerfile` regenerates consistently:
```yaml
args:
  postgresql: true
  redis: true
```

---

## 3. fly.toml Reference

Minimal production Rails configuration:

```toml
app = "my-rails-app"
primary_region = "ord"

[build]
  dockerfile = "Dockerfile"

[deploy]
  # Runs once per deploy in a temporary Machine BEFORE traffic switches
  release_command = "bin/rails db:prepare"
  strategy = "rolling"   # rolling | immediate | canary | bluegreen

[env]
  RAILS_ENV = "production"
  RAILS_LOG_TO_STDOUT = "true"
  LOG_LEVEL = "info"
  WEB_CONCURRENCY = "2"

[http_service]
  internal_port = 3000
  force_https = true
  auto_stop_machines = "stop"    # stop | suspend | off
  auto_start_machines = true

  [http_service.concurrency]
    type = "connections"
    hard_limit = 25
    soft_limit = 20

[[http_service.checks]]
  grace_period = "10s"
  interval = "30s"
  method = "GET"
  path = "/up"           # Rails 7.1+ built-in health check
  timeout = "5s"

[[vm]]
  size = "shared-cpu-1x"
  memory = "512mb"
```

### Key `[deploy]` options
| Option | Value | Notes |
|---|---|---|
| `release_command` | `"bin/rails db:prepare"` | Runs migrations; failure aborts deploy |
| `strategy` | `"rolling"` | Replaces Machines one by one (zero-downtime) |
| `wait_timeout` | `"5m"` | Time to wait for Machine health |

### Key `[http_service]` options
| Option | Value | Notes |
|---|---|---|
| `internal_port` | `3000` | Must match Rails `PORT` or `-p` flag |
| `auto_stop_machines` | `"stop"` | Shuts idle Machines to save costs |
| `auto_start_machines` | `true` | Wakes on incoming request |
| `force_https` | `true` | Always enable in production |

### `[env]` — non-sensitive only
Never put secrets in `[env]`. Use `fly secrets set` for anything sensitive.

### `[[vm]]` sizes
| Size | CPU | RAM | Use |
|---|---|---|---|
| `shared-cpu-1x` | 1 shared | 256MB | Dev / low traffic |
| `shared-cpu-1x` + `memory = "512mb"` | 1 shared | 512MB | Typical Rails |
| `shared-cpu-2x` | 2 shared | 512MB | Medium load |
| `performance-1x` | 1 dedicated | 2GB | CPU-bound workloads |
| `performance-2x` | 2 dedicated | 4GB | High traffic |

---

## 4. Secrets Management

Secrets are encrypted, stored in Fly vault, and injected as env vars at runtime.

```bash
# Set one or more secrets (triggers Machine restart by default)
fly secrets set SECRET_KEY_BASE=$(rails secret) RAILS_MASTER_KEY=$(cat config/master.key)

# Stage secrets (no immediate restart; applied at next deploy or manual restart)
fly secrets set DATABASE_URL=postgres://... --stage

# List secret names (values hidden)
fly secrets list

# Remove secrets
fly secrets unset OLD_SECRET
```

**Required Rails secrets on Fly:**
| Secret | How to generate |
|---|---|
| `SECRET_KEY_BASE` | `bin/rails secret` |
| `RAILS_MASTER_KEY` | Contents of `config/master.key` |
| `DATABASE_URL` | Set automatically by `fly postgres attach` |
| `REDIS_URL` | Set when provisioning Redis |

**Never commit `config/master.key`** — set it as a secret instead. Add to `.gitignore` if not already.

---

## 5. Multi-Process Setup (Web + Worker)

Add Sidekiq or other workers as a separate process group:

```toml
[processes]
  web    = "bundle exec rails server -b [::] -p 3000"
  worker = "bundle exec sidekiq -c 5"

[http_service]
  internal_port = 3000
  processes = ["web"]         # http_service only routes to web

[[vm]]
  size = "shared-cpu-1x"
  memory = "512mb"
  processes = ["web"]

[[vm]]
  size = "shared-cpu-1x"
  memory = "512mb"
  processes = ["worker"]
```

Scale each process group independently:
```bash
fly scale count web=2 worker=1
```

---

## 6. Running One-Off Commands

```bash
# Open Rails console on a running Machine
fly console

# SSH into a running Machine
fly ssh console

# Run a one-off command (temporary Machine, auto-destroyed)
fly machine run . --rm --command "bin/rails db:seed"
fly machine run . --rm --command "bin/rails runner 'User.reindex'"

# Run a rake task
fly ssh console -C "bin/rails db:version"
```

**`fly console` vs `fly ssh console`:**
- `fly console` — uses `console_command` from `fly.toml` (defaults to `/bin/bash`); set it to `bin/rails console` for direct Rails console
- `fly ssh console` — raw shell access

```toml
# fly.toml — set Rails console as default
console_command = "/rails/bin/rails console"
```

---

## 7. Monitoring & Troubleshooting

```bash
fly status                   # Machine health overview
fly logs                     # Tail live logs
fly logs --instance <id>     # Logs for a specific Machine
fly releases                 # Deployment history
fly releases show <version>  # Details of a specific release

# Check health of a deploy
fly checks list
fly checks list --app my-rails-app
```

### Common deploy failures
| Symptom | Cause | Fix |
|---|---|---|
| `release_command` fails | Migration error or missing DB | Check `fly logs`; verify `DATABASE_URL` is set |
| Machine never becomes healthy | Health check path wrong or app crashes | Check `/up` route exists; `fly logs` for stack trace |
| `SECRET_KEY_BASE` missing | Forgot to set secret | `fly secrets set SECRET_KEY_BASE=$(rails secret)` |
| Assets not found | Precompile step failed | Check Dockerfile build stage; verify `assets:precompile` runs |
| Memory OOM | Rails + gems exceed limit | Increase `memory` in `[[vm]]`; add swap with `--swap` in Dockerfile generator |

---

## 8. Scaling

```bash
# Scale Machine count
fly scale count 3                    # 3 Machines for default process
fly scale count web=2 worker=1       # Per process group

# Scale Machine size
fly scale vm shared-cpu-2x
fly scale vm shared-cpu-1x --memory 1024

# Show current scale
fly scale show
```

`auto_stop_machines = "stop"` and `auto_start_machines = true` are the recommended default for cost savings — Machines stop when idle and wake on the first request (~300ms cold start for Rails).

---

## 9. Local Docker Development (Mirror Production)

Run the exact production image locally:

```bash
# Build the production image locally
docker build -t my-rails-app .

# Run with local env vars
docker run --rm -it \
  -p 3000:3000 \
  -e DATABASE_URL=postgres://localhost:5432/myapp_dev \
  -e SECRET_KEY_BASE=local_dev_secret \
  -e RAILS_ENV=production \
  my-rails-app

# Or use docker compose for full stack (Rails + Postgres + Redis)
docker compose up
```

### Recommended `docker-compose.yml` for local Fly parity:

```yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://postgres:password@db:5432/myapp_development
      - SECRET_KEY_BASE=local_dev_secret_not_for_production
      - RAILS_ENV=development
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp_development
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

---

## 10. Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Secrets in `[env]` section | Visible in `fly.toml`; committed to git | Use `fly secrets set` |
| `db:migrate` in `ENTRYPOINT` instead of `release_command` | Runs on every Machine start; risk of parallel migration | Move to `release_command` in `fly.toml` |
| Hardcoded `PORT=3000` without `-b [::] ` | Won't bind to IPv6; unreachable on Fly | Use `rails server -b [::] -p 3000` |
| Single `shared-cpu-1x` with 256MB for Puma + multiple workers | OOM kills | Use 512MB+ or reduce `WEB_CONCURRENCY` |
| Not committing `Dockerfile` | Local and CI builds diverge from production | Always commit all generated files |
| Using `fly launch` on an existing app without `--no-deploy` | May reset fly.toml configuration | Use `fly deploy` for subsequent deploys |
| Omitting health check | Deploy succeeds even if app crashes silently | Always configure `[[http_service.checks]]` with `path = "/up"` |
