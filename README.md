# open-agent-skills

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

### `swift-expert`

Skills for writing Swift with a focus on correctness and performance.

| Skill | Description |
|---|---|
| `swift-ipc` | Inter-process communication on Apple platforms — XPC, `NSXPCConnection`, shared memory (`mmap`, POSIX `shm`), lock-free SPSC ring buffers, Mach messages, and cross-process synchronization patterns |
| `swift-performance` | Performance and optimization — method dispatch, existentials, ARC, copy-on-write, value vs reference types, noncopyable types (`~Copyable`), typed throws, collections, strings, struct layout, and Swift Concurrency overhead |
| `swift-synchronization` | Threading and synchronization — `DispatchQueue` ownership pattern, `OSAllocatedUnfairLock`, Swift `Mutex` and `Atomic` (Synchronization module), actors and `nonisolated`, memory ordering, priority inversion, and deadlock |
| `swift-testing` | Testing with Swift Testing (Swift 5.9+ / Xcode 16+) — `@Suite`, `@Test`, `#expect`, `#require`, parameterized tests, async tests, tags, traits, dependency injection, and XCTest migration |

## Installing

### Claude Code

Add this repository as a marketplace, then install the plugins you want:

```bash
claude plugin marketplace add Tyr0/open-agent-skills
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
