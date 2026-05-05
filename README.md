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

Plugins are not yet populated. See `ARCHITECTURE.md` for the plugin and skill format that will land here.

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
