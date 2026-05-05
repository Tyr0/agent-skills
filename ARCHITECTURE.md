# Architecture

## Overview

`open-agent-skills` is a multi-agent skills library. Each plugin groups one or more *skills* — self-contained reference documents that compatible coding tools load when a user's question matches the skill's trigger description.

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
