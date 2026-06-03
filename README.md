# tcalderone-agent-skills

An open-source library of plugins and skills for AI coding agents.

Each plugin groups one or more **skills** — self-contained reference documents that compatible coding agents load on demand when a user's question matches the skill's trigger description. Skills are agent-neutral by design: the same skill source feeds multiple tools through tool-specific registries.

## Supported Agents

| Agent | Status | Discovery |
|---|---|---|
| Claude Code | Supported | Reads `.claude-plugin/marketplace.json` |
| OpenAI Codex | Supported | Reads `.agents/plugins/marketplace.json` and each plugin's `.codex-plugin/plugin.json` |
| Gemini CLI | Planned | Discovery format TBD |

The repository is intentionally structured so that adding a new agent means adding a new entrypoint or registry — not forking the underlying skill content.

## Repository Layout

```
.agents/plugins/marketplace.json     Codex marketplace registry
.claude-plugin/marketplace.json      Claude Code marketplace registry
plugins/                             Plugin sources (one directory per plugin)
AGENTS.md                            Entrypoint for AI coding agents
ARCHITECTURE.md                      How plugins, skills, and registries fit together
README.md                            This file
```

See `ARCHITECTURE.md` for the plugin and skill format conventions.

## Plugins

### `credential-expert`

Local credential storage for AI coding agents. A non-secret index file at `~/.agents/credentials.json` (mode 0600) maps credential names to OS-keystore references; the actual secrets live in the macOS Keychain as per-credential `generic-password` items. Other plugins (e.g. `pr-expert`) consume credentials through this skill instead of reading plaintext token files.

| Skill | Description |
|---|---|
| `credential-storage` | Index file schema, `security` CLI commands for set/read/delete/rotate, the safe-consumption pattern for downstream skills, a global pre-commit hook for accidental-commit defense, and the threat model |

---

### `fly-expert`

Skills for deploying and operating applications on [Fly.io](https://fly.io).

| Skill | Description |
|---|---|
| `fly-rails-deployment` | Deploying Ruby on Rails to Fly.io — `fly launch`, `fly.toml` configuration, the generated Dockerfile, secrets management, multi-process setups (Sidekiq), health checks, scaling, one-off commands, and local Docker development that mirrors production |
| `fly-postgres` | Postgres on Fly.io — Managed Postgres (MPG) vs unmanaged clusters, `fly postgres attach`, `DATABASE_URL` configuration, running migrations via `release_command`, local proxy access, connection pooling (PgBouncer), backups, and HA |

---

### `pr-expert`

Skills for managing pull requests on GitHub and Bitbucket Cloud via their REST APIs. Credentials are consumed through `credential-expert/credential-storage`.

| Skill | Description |
|---|---|
| `pr-setup` | Authentication and configuration — platform detection from git remote, the `github` and `bitbucket` credential entries, base URLs, auth headers, pagination patterns, and 401/403 troubleshooting |
| `pr-management` | PR lifecycle — create, list, get, update, merge (squash/rebase/merge commit), decline/close, view diffs (raw and structured), check merge conflicts and CI/build status, list commits, and sync branches |
| `pr-review` | Reviewing and commenting — post general and inline code comments, reply to threads, read all PR feedback, submit reviews (approve/request changes), dismiss reviews, resolve conversations, and manage Bitbucket tasks |

---

### `rails-expert`

Skills for working with Ruby on Rails.

| Skill | Description |
|---|---|
| `rails-active-job` | Background jobs with Active Job — job structure, queue adapters (Solid Queue, GoodJob, Sidekiq), `retry_on`, `discard_on`, mailer delivery, and testing jobs |
| `rails-activerecord-queries` | ActiveRecord query optimization — N+1 prevention, `includes`/`eager_load`/`preload`, batch processing (`find_each`), bulk operations (`update_all`, `insert_all`), `pluck`, `exists?`, and the Bullet gem |
| `rails-caching` | Caching — fragment and Russian doll caching, low-level `Rails.cache` usage, HTTP caching (`ETag`, `Cache-Control`), cache store selection (Redis, Memcached), cache key design, and invalidation strategies |
| `rails-console` | Using `bin/rails console` — sandbox mode, `bin/dev` context, querying and mutating data, reloading code, production console access (Fly, Heroku, Kubernetes), IRB configuration, and common debugging patterns |
| `rails-db` | Database management via `bin/rails db:*` — running and rolling back migrations, checking migration status, seeding, schema formats (`schema.rb` vs `structure.sql`), multi-database setups, and safe migration patterns |
| `rails-db-best-practices` | Schema design best practices — indexing strategies, association patterns, join tables, Postgres-specific features (JSONB, GIN/GiST indexes, range types, advisory locks, upsert), and migration anti-patterns |
| `rails-generate` | Code generation via `bin/rails generate` — models, migrations, controllers, scaffolds, mailers, jobs, channels, concerns, the Dockerfile generator, and custom generators |
| `rails-hotwire` | Hotwire — Turbo Drive (page navigation without reloads), Turbo Frames (scoped partial updates), Turbo Streams (DOM mutations from HTTP responses and Action Cable broadcasts), and Stimulus (lightweight JS controllers with targets, values, actions, and outlets) |
| `rails-routing` | Routing — `resources`/`resource`, nested routes, shallow nesting, namespaces, scopes, member/collection routes, named routes, constraints, catch-alls, and `draw` for large route files |
| `rails-tailwind` | Tailwind CSS — `tailwindcss-rails` setup, `tailwind.config.js`, purge-safe dynamic classes, extracting patterns with partials and ViewComponent, responsive design, dark mode, design tokens, the forms and typography plugins, and `@apply` guidance |
| `rails-testing` | Testing with Minitest — fixtures, model tests, integration/controller tests, system tests (Capybara), mailer tests, job tests, SimpleCov, parallel tests, and RuboCop-Minitest |
| `rails-views` | View layer — ERB partials, collection rendering, layouts and `content_for`/`yield`, view helpers, ViewComponent (slots, testing, previews, sidecar assets, Stimulus integration), and Phlex (pure-Ruby views) |

---

### `swift-expert`

Skills for writing Swift with a focus on correctness and performance.

| Skill | Description |
|---|---|
| `swift-ipc` | Inter-process communication on Apple platforms — XPC, `NSXPCConnection`, shared memory (`mmap`, POSIX `shm`), lock-free SPSC ring buffers, Mach messages, and cross-process synchronization patterns |
| `swift-performance` | Performance and optimization — method dispatch, existentials, ARC, copy-on-write, value vs reference types, noncopyable types (`~Copyable`), typed throws, collections, strings, struct layout, and Swift Concurrency overhead |
| `swift-synchronization` | Threading and synchronization — `DispatchQueue` ownership pattern, `OSAllocatedUnfairLock`, Swift `Mutex` and `Atomic` (Synchronization module), actors and `nonisolated`, memory ordering, priority inversion, and deadlock |
| `swift-testing` | Testing with Swift Testing (Swift 5.9+ / Xcode 16+) — `@Suite`, `@Test`, `#expect`, `#require`, parameterized tests, async tests, tags, traits, dependency injection, and XCTest migration |

---

### `swiftui-expert`

Skills for writing SwiftUI on iOS 17+ / macOS 14+, with the Observation framework as the default data-flow model.

| Skill | Description |
|---|---|
| `swiftui-state-management` | State and data flow — `@State`, `@Binding`, `@Bindable`, `@Environment`, the Observation framework (`@Observable`), view re-rendering rules, identity-keyed state lifetime, and migration from `ObservableObject`/`@StateObject`/`@EnvironmentObject` |
| `swiftui-performance` | Performance — body invalidation rules, `EquatableView`, view identity, lazy containers (`LazyVStack`, `List`), image decoding strategy, `drawingGroup()` and `Canvas`, `.task` lifecycle, and profiling via the SwiftUI Instruments template |
| `swiftui-layout` | Layout — the proposed-size / required-size negotiation, `frame` vs `fixedSize`, stacks and `Spacer`, alignment guides, safe area, `containerRelativeFrame`, `onGeometryChange`, grids, `ScrollView` APIs, and the custom `Layout` protocol |
| `swiftui-navigation` | Navigation — `NavigationStack`, `NavigationSplitView`, `NavigationPath`, value-based navigation with `navigationDestination(for:)`, deep linking, sheets/popovers/covers/inspectors, the `dismiss` environment, and migration from `NavigationView` |
| `swiftui-animation` | Animation — `withAnimation` vs `.animation(_:value:)`, `Transaction`, transitions, `matchedGeometryEffect`, `PhaseAnimator`, `KeyframeAnimator`, custom `Animatable` types, and `contentTransition` |
| `swiftui-uikit-interop` | Bridging — `UIViewRepresentable`/`UIViewControllerRepresentable` (and AppKit equivalents), the Coordinator pattern, `UIHostingController`, `UIHostingConfiguration` for cells, sizing with `sizeThatFits`, and lifecycle management |
| `swiftui-accessibility` | Accessibility — labels/values/hints/traits, grouping with `accessibilityElement(children:)`, custom actions and rotors, Dynamic Type with `@ScaledMetric`, Reduce Motion, VoiceOver focus, and an audit checklist |

## Installing

### Claude Code

Add this repository as a marketplace, then install the plugins you want:

```bash
claude plugin marketplace add Tyr0/tcalderone-agent-skills
claude plugin install <plugin-name>
```

### OpenAI Codex

Codex resolves plugins through `.agents/plugins/marketplace.json`. Point Codex at this repository per its plugin documentation and install the plugins you want.

## Contributing

Contributions are welcome. Before opening a pull request:

1. Read `ARCHITECTURE.md` for the plugin and skill format conventions.
2. Add or update both marketplace registries when introducing a new plugin.
3. Keep skill content agent-neutral — tool-specific guidance belongs in the entrypoint files (`AGENTS.md`, etc.), not in the shared skill.

## License

MIT. See `LICENSE`.
