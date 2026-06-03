# Architecture

## Overview

`tcalderone-agent-skills` is a multi-agent skills library. Each plugin groups one or more *skills* — self-contained reference documents that compatible coding tools load when a user's question matches the skill's trigger description.

The repository is structured so that one source of truth (`plugins/`) feeds multiple coding agents through tool-specific registries.

```
.agents/
  plugins/
    marketplace.json        # OpenAI Codex registry
.claude-plugin/
  marketplace.json          # Claude Code registry
plugins/
  <plugin-name>/
    .codex-plugin/
      plugin.json           # Codex per-plugin manifest
    skills/
      <skill-name>/
        SKILL.md            # The skill document (frontmatter + markdown)
AGENTS.md                   # AI coding agent entrypoint
README.md                   # Human-facing project overview
```

## Supported Agents

| Agent | Discovery |
|---|---|
| Claude Code | Reads `.claude-plugin/marketplace.json` to enumerate available plugins |
| OpenAI Codex | Reads `.agents/plugins/marketplace.json` and each plugin's `.codex-plugin/plugin.json` |
| Gemini CLI | Planned. Discovery format and entrypoint to be added when support lands |

When adding support for a new agent, add a new entrypoint or registry file rather than forking shared skill content.

## Plugin Registries

Both registries describe the same `plugins/` tree but use different schemas to match each tool's expectations.

### Claude Code marketplace (`.claude-plugin/marketplace.json`)

| Field | Purpose |
|---|---|
| `name` | Marketplace identifier |
| `owner` | Marketplace owner metadata |
| `plugins[].name` | Plugin identifier (matches directory name) |
| `plugins[].source` | Relative path to the plugin directory |
| `plugins[].description` | Human-readable plugin summary |
| `plugins[].version` | Semantic version |
| `plugins[].author` | Plugin author metadata |

### Codex marketplace (`.agents/plugins/marketplace.json`)

| Field | Purpose |
|---|---|
| `name` | Marketplace identifier |
| `interface.displayName` | Codex marketplace display name |
| `plugins[].name` | Plugin identifier (matches directory name) |
| `plugins[].source` | `{ "source": "local", "path": "./plugins/<plugin>" }` |
| `plugins[].policy` | Codex install and authentication policy |
| `plugins[].category` | Codex marketplace category |

### Codex per-plugin manifest (`plugins/<plugin>/.codex-plugin/plugin.json`)

| Field | Purpose |
|---|---|
| `name`, `version`, `description` | Required package identity metadata |
| `author` | Publisher metadata |
| `skills` | Relative path to the skill directory; currently always `./skills/` |
| `interface` | Codex install-surface display metadata |

Both registries should list the same plugins in the same order unless a tool-specific compatibility issue requires an exception.

## Skill Document Format

Every skill lives in `plugins/<plugin>/skills/<skill-name>/SKILL.md`.

```
---
name: <skill-name>          # Kebab-case; matches the directory name exactly
description: <trigger text> # Multi-sentence description of when a coding tool should use this skill
---

# <Skill Title>

<One-line intro sentence.>

---

## <Major Section>

...

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
```

### Frontmatter rules

- `name` must exactly match the containing directory name.
- `description` is the primary routing signal — it should enumerate trigger conditions, covered topics, and common user questions that should activate the skill.

### Content conventions

- Sections use `##`; subsections use `###`.
- Code blocks are language-tagged (e.g. `ruby`, `erb`, `bash`, `javascript`, `yaml`).
- Decision matrices and quick-reference tables use a three-column format.
- Every skill ends with an **Anti-Patterns** table (`Anti-pattern | Problem | Fix`).
- Target length: 300–500 lines. Dense and practical; no filler.

## Plugins

### `credential-expert`

Local credential storage for AI coding agents.

| Skill | Covers |
|---|---|
| `credential-storage` | Non-secret index at `~/.agents/credentials.json` (mode 0600); macOS Keychain via `security` for set/read/delete/rotate; canonical safe-consumption pattern for other skills; pre-commit hook to refuse accidental commits of the index file; threat model |

The index schema is forward-compatible with Linux (`secret_service`) and Windows (`credential_manager`) backends — only the per-entry backend object differs.

### `fly-expert`

Fly.io deployment knowledge.

| Skill | Covers |
|---|---|
| `fly-rails-deployment` | Rails on Fly — `fly launch`, `fly.toml`, generated Dockerfile, secrets, Sidekiq process groups, health checks, scaling, local Docker parity |
| `fly-postgres` | Postgres on Fly — MPG vs unmanaged, attach, `DATABASE_URL`, migrations via `release_command`, proxy, pooling, backups, HA |

### `pr-expert`

Pull request management across GitHub and Bitbucket Cloud. Consumes credentials through `credential-expert/credential-storage`.

| Skill | Covers |
|---|---|
| `pr-setup` | Auth, platform detection, credential names (`github`, `bitbucket`), pagination |
| `pr-management` | PR lifecycle — create, list, update, merge, decline, diffs, CI status, conflicts |
| `pr-review` | Comments (general + inline), reviews, approvals, thread resolution, Bitbucket tasks |

### `rails-expert`

Ruby on Rails best practices, from the database layer through the frontend.

| Skill | Covers |
|---|---|
| `rails-active-job` | Background jobs — Solid Queue, GoodJob, Sidekiq, `retry_on`, `discard_on` |
| `rails-activerecord-queries` | Query optimization — N+1, eager loading, batch processing, bulk operations |
| `rails-caching` | Caching — fragment/Russian doll, low-level cache, HTTP caching, Redis, invalidation |
| `rails-console` | `bin/rails console` — sandbox, querying, production access, IRB tips |
| `rails-db` | Database management — migrations, schema formats, multi-database, safe patterns |
| `rails-db-best-practices` | Schema design — indexes, associations, Postgres features (JSONB, GIN, advisory locks) |
| `rails-generate` | Code generation — models, migrations, controllers, scaffolds, custom generators |
| `rails-hotwire` | Hotwire — Turbo Drive, Turbo Frames, Turbo Streams (HTTP + broadcasts), Stimulus |
| `rails-routing` | Routing — resources, nested routes, namespaces, constraints, route helpers |
| `rails-tailwind` | Tailwind CSS — setup, config, dynamic classes, ViewComponent integration, dark mode |
| `rails-testing` | Minitest — fixtures, model/integration/system tests, SimpleCov, Capybara |
| `rails-views` | View layer — ERB partials, layouts, ViewComponent, Phlex, form helpers |

### `swift-expert`

Swift language knowledge focused on correctness and performance.

| Skill | Covers |
|---|---|
| `swift-ipc` | IPC on Apple platforms — XPC, shared memory, lock-free SPSC ring buffers, Mach messages, cross-process sync |
| `swift-performance` | Performance — dispatch, existentials, ARC, copy-on-write, `~Copyable`, typed throws, collections, layout, Concurrency overhead |
| `swift-synchronization` | Threading — `DispatchQueue` ownership, `OSAllocatedUnfairLock`, Swift `Mutex`/`Atomic`, actors, memory ordering, deadlock |
| `swift-testing` | Swift Testing — `@Suite`, `@Test`, `#expect`, `#require`, parameterized tests, traits, XCTest migration |

### `swiftui-expert`

SwiftUI knowledge targeting iOS 17+ / macOS 14+, with the Observation framework as the default data-flow model.

| Skill | Covers |
|---|---|
| `swiftui-state-management` | `@State`, `@Binding`, `@Bindable`, `@Environment`, the Observation framework (`@Observable`), view re-rendering rules, identity, migration from `ObservableObject` |
| `swiftui-performance` | Body invalidation, `EquatableView`, lazy containers, image performance, `drawingGroup`/`Canvas`, `.task` lifecycle, profiling with the SwiftUI Instruments template |
| `swiftui-layout` | Proposed/required size negotiation, `frame`/`fixedSize`, alignment guides, safe area, `containerRelativeFrame`, `onGeometryChange`, custom `Layout` |
| `swiftui-navigation` | `NavigationStack`, `NavigationSplitView`, `NavigationPath`, value-based `navigationDestination`, deep linking, sheets/popovers/covers, `dismiss` |
| `swiftui-animation` | `withAnimation`, `.animation(_:value:)`, transactions, transitions, `matchedGeometryEffect`, `PhaseAnimator`, `KeyframeAnimator`, `Animatable` |
| `swiftui-uikit-interop` | `UIViewRepresentable`/`UIViewControllerRepresentable` (and AppKit), Coordinator pattern, `UIHostingController`/`UIHostingConfiguration`, sizing, lifecycle |
| `swiftui-accessibility` | Labels/values/hints/traits, grouping with `accessibilityElement`, custom actions and rotors, Dynamic Type, Reduce Motion, focus, audit checklist |

## Conventions

### Local credentials

Plugins that need persistent secrets MUST consume them through `credential-expert/credential-storage`. Do not introduce a new plaintext token file or a parallel index. The index file is at `~/.agents/credentials.json` (mode `0600`); the parent directory is `~/.agents/` (mode `0700`). Keychain service names follow the convention `tcalderone-agent-skills:<plugin-name>`. Account values are either a natural identifier (email, username) or the literal string `default`.

In headless contexts (CI, daemons), consumers should branch on the environment and read from environment-variable secrets instead — `credential-storage` is for interactive workstations.

## Adding a Plugin

1. Create `plugins/<plugin-name>/` with a `skills/` subdirectory.
2. Add `plugins/<plugin-name>/.codex-plugin/plugin.json` with `skills` set to `./skills/`.
3. Add an entry to both `.claude-plugin/marketplace.json` and `.agents/plugins/marketplace.json`.
4. Document the plugin in `README.md`.
5. Document the plugin and its skills in this file.

## Adding a Skill

1. Create `plugins/<plugin>/skills/<skill-name>/SKILL.md` following the format above.
2. Update the plugin's skill table in `README.md` and this file.

## Maintenance Guidelines

- Keep skill content agent-neutral. Avoid naming a specific assistant unless the section is an adapter or entrypoint for that assistant.
- When adding support for a new agent, add or update an entrypoint file and document the discovery behavior here.
- When introducing a new convention that depends on a specific agent's discovery mechanism, document that dependency in this file rather than in shared skill bodies.
