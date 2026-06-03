---
name: credential-storage
description: Use this skill whenever you need to store, read, rotate, or delete a local credential (API token, PAT, app password) on behalf of another plugin. Also use it when an authenticated workflow asks "where do I keep this token", when another skill encounters a missing or invalid credential, or when a user wants to review which credentials are configured. Triggers on 'save my GitHub token', 'set up Bitbucket credentials', 'rotate my API token', 'where is the credentials file', 'how do I store an API key for an agent', or any task that requires reading a secret from local credential storage.
---

# Credential Storage

Local credential storage for AI coding agents. A non-secret index file at `~/.agents/credentials.json` records which credentials exist; the actual secrets live in the operating-system keystore (macOS Keychain on Darwin) and are looked up by convention.

This skill is for interactive workstation use. CI and other headless contexts should use environment-variable secrets and bypass this storage entirely.

---

## Quick Reference

| Operation | Command |
|---|---|
| Set or rotate (macOS) | `security add-generic-password -U -s 'agent-skills:<name>' -a '<account>' -w "$value"` |
| Read (macOS) | `security find-generic-password -w -s 'agent-skills:<name>' -a '<account>'` |
| Delete (macOS) | `security delete-generic-password -s 'agent-skills:<name>' -a '<account>'` |
| List configured | Read names from `~/.agents/credentials.json` |

`<name>` is the credential name (e.g. `github`, `bitbucket`); `<account>` is the entry's `account` field, or `default` when absent. See **Keying Convention** below.

---

## Index File

### Location and permissions

| Path | Mode | Contents |
|---|---|---|
| `~/.agents/` | `0700` | Directory; user-only access |
| `~/.agents/credentials.json` | `0600` | Non-secret index of configured credentials |

The index file MUST be mode `0600`. The parent directory MUST be mode `0700`. Verify before every write; refuse to operate if either is wrong.

```bash
mkdir -p ~/.agents
chmod 700 ~/.agents
touch ~/.agents/credentials.json
chmod 600 ~/.agents/credentials.json
```

### Schema

```json
{
  "version": 1,
  "credentials": {
    "github": {},
    "bitbucket": {
      "account": "tyler@calder.one"
    }
  }
}
```

| Field | Required | Description |
|---|---|---|
| `version` | yes | Schema version. Currently `1`. |
| `credentials` | yes | Map of credential name → entry. Names are kebab-case identifiers used by consuming plugins (e.g. `github`, `bitbucket`, `github-work`). |
| `credentials.<name>.account` | no | Per-credential account/username. Used for disambiguation and as the basic-auth username where applicable. Defaults to `"default"` when absent. |
| `credentials.<name>.backend` | no | Override the platform-default keystore. Rare — only when the user wants a non-default store like `pass` on Linux. Values: `macos-keychain`, `secret-service`, `windows-credential-manager`, `pass-cli`. |
| `credentials.<name>.<other>` | no | Additional non-secret metadata (e.g. `scopes`, `created_at`). Never store the secret itself in the index. |

The index file is non-secret by design but is still mode `0600` because it discloses *which* services the user has configured.

---

## Keying Convention

Consuming skills derive the keystore lookup from `(plugin-prefix, credential-name, account)` rather than reading keying material out of the index. The convention is identical across backends:

| Backend | Keystore call shape |
|---|---|
| `macos-keychain` (default on Darwin) | `service = "agent-skills:<name>"`, `account = entry.account or "default"` |
| `secret-service` (default on Linux) | attrs `{application: "agent-skills", credential: "<name>", account: entry.account or "default"}` |
| `windows-credential-manager` (default on Windows) | `target = "agent-skills:<name>"`, `username = entry.account or "default"` |

Backend selection: if `entry.backend` is present, use it; otherwise infer from `uname -s`. The index file is never required to know the platform.

Do not edit `~/.agents/credentials.json` to change a service name. The mapping is conventional — diverging from it makes the entry unreadable by every consuming skill.

---

## Initial Setup

If `~/.agents/credentials.json` does not exist, create it and the parent directory with the correct modes before adding any credential.

```bash
mkdir -p ~/.agents && chmod 700 ~/.agents
if [ ! -f ~/.agents/credentials.json ]; then
  printf '{"version":1,"credentials":{}}\n' > ~/.agents/credentials.json
  chmod 600 ~/.agents/credentials.json
fi
```

Verify modes after creation:

```bash
stat -f '%A %N' ~/.agents ~/.agents/credentials.json
# Expected:
# 700 /Users/<you>/.agents
# 600 /Users/<you>/.agents/credentials.json
```

---

## Adding or Rotating a Credential (macOS)

Two steps: write the secret to the keystore, then update the index.

### Step 1 — store the secret

```bash
# Read the secret interactively. Do NOT pass it via shell history.
read -rs value && echo

security add-generic-password \
  -U \
  -s 'agent-skills:bitbucket' \
  -a 'tyler@calder.one' \
  -w "$value"

unset value
```

The `-U` flag updates the entry if it already exists, which makes set and rotate the same command.

The `-w "$value"` form briefly exposes the secret in the process argument list visible to `ps`. For rotation on a single-user workstation this is acceptable. To avoid even that exposure, omit `-w` and let `security` prompt interactively:

```bash
security add-generic-password -U -s '...' -a '...'
# security prompts for the password
```

### Step 2 — update the index

```bash
python3 - <<'PY'
import json, os, pathlib

path = pathlib.Path.home() / ".agents" / "credentials.json"
data = json.loads(path.read_text())
data["credentials"]["bitbucket"] = {"account": "tyler@calder.one"}
tmp = path.with_suffix(".json.tmp")
tmp.write_text(json.dumps(data, indent=2) + "\n")
os.chmod(tmp, 0o600)
tmp.replace(path)
PY
```

For credentials with no natural account identifier (e.g. a single GitHub PAT), the entry can be empty:

```python
data["credentials"]["github"] = {}
```

The atomic-replace pattern (`tmp.replace(path)`) avoids partially-written index files.

---

## Reading a Credential (macOS)

The canonical read pattern. Consuming plugins MUST use this shape:

```bash
# 1. Resolve the account from the index (defaulting if absent).
name="bitbucket"
account=$(python3 -c "
import json, pathlib
data = json.loads((pathlib.Path.home() / '.agents/credentials.json').read_text())
entry = data['credentials'].get('$name')
if entry is None:
    raise SystemExit(f'credential not configured: $name')
print(entry.get('account', 'default'))
")

# 2. Fetch the secret. Never echo, printf, or cat it.
service="agent-skills:$name"
bb_token=$(security find-generic-password -w -s "$service" -a "$account")

# 3. Use it via a flag that reads from a variable. Unset immediately after.
curl -u "$account:$bb_token" "https://api.bitbucket.org/2.0/user"
unset bb_token
```

### When to prefer environment variables

If the consuming command supports an environment-variable form for credentials (most HTTP clients and SDKs do), pass via the environment of the child process rather than a command-line flag — flag arguments appear in `ps`, environment variables of a child process do not appear in another user's `ps` output.

```bash
GITHUB_TOKEN=$(security find-generic-password -w -s 'agent-skills:github' -a 'default') \
  gh pr list
unset GITHUB_TOKEN
```

---

## Listing Configured Credentials

```bash
python3 -c "
import json, pathlib
data = json.loads((pathlib.Path.home() / '.agents/credentials.json').read_text())
for name, entry in data['credentials'].items():
    extras = ', '.join(f'{k}={v}' for k, v in entry.items())
    print(f'{name}\t{extras}')
"
```

Listing reveals only non-secret metadata. Never include any secret in listing output.

---

## Deleting a Credential

```bash
name="bitbucket"
account=$(python3 -c "
import json, pathlib
data = json.loads((pathlib.Path.home() / '.agents/credentials.json').read_text())
print(data['credentials'].get('$name', {}).get('account', 'default'))
")

security delete-generic-password -s "agent-skills:$name" -a "$account"

python3 - <<'PY'
import json, os, pathlib

path = pathlib.Path.home() / ".agents" / "credentials.json"
data = json.loads(path.read_text())
data["credentials"].pop("bitbucket", None)
tmp = path.with_suffix(".json.tmp")
tmp.write_text(json.dumps(data, indent=2) + "\n")
os.chmod(tmp, 0o600)
tmp.replace(path)
PY
```

---

## First Access — Keychain ACL Prompt

The first time a process reads a Keychain item, macOS shows a dialog asking whether to allow access. Three options:

| Choice | Effect |
|---|---|
| Always Allow | Future reads from the same calling binary are silent |
| Allow | Grants this single read; future reads prompt again |
| Deny | This read fails; future reads prompt again |

Recommended choice: **Always Allow**, scoped per credential. The dialog binds the grant to the calling binary (typically `/usr/bin/security`), not to a session. One grant per credential per machine.

ACLs are per-Keychain-item, so granting "Always Allow" on the Bitbucket credential does not grant access to the GitHub credential. This isolation is the primary security property this design provides.

---

## Defense in Depth — Pre-Commit Hook

The most likely failure mode for any credential file is not cryptographic — it is a developer running `git add -A` from `$HOME` or accidentally committing a copy of `~/.agents/credentials.json` to a repo. Install a global pre-commit hook that refuses such commits.

```bash
mkdir -p ~/.config/git/hooks
cat > ~/.config/git/hooks/pre-commit <<'HOOK'
#!/usr/bin/env bash
# Refuse commits that touch ~/.agents/ or look like credential indexes.
set -euo pipefail
staged=$(git diff --cached --name-only)
if echo "$staged" | grep -qE '(^|/)\.agents/|(^|/)credentials\.json$'; then
  echo "pre-commit: refusing to commit ~/.agents/ or a credentials.json file" >&2
  echo "If this is a false positive, bypass with: git commit --no-verify" >&2
  exit 1
fi
HOOK
chmod +x ~/.config/git/hooks/pre-commit

git config --global core.hooksPath ~/.config/git/hooks
```

Verify:

```bash
git config --global --get core.hooksPath
# /Users/<you>/.config/git/hooks
```

---

## Headless and CI Usage

This skill is for interactive workstations. In CI, on Fly.io machines, in containers, and in other headless contexts:

- Do not attempt to read from `~/.agents/credentials.json`.
- Do not attempt to call `security`. The Data Protection Keychain is not reliably available without a GUI session.
- Read credentials from environment variables provided by the CI runner's secret store.

Consuming plugins should branch on `[ -n "${CI:-}" ]` or an equivalent signal and prefer the environment-variable form when in CI.

---

## Platform Dispatch

The schema is platform-agnostic. Backend selection happens at read time. A consuming skill that wants to support more than macOS can dispatch like this:

```bash
read_credential() {
  local name="$1"
  local entry account backend

  entry=$(python3 -c "
import json, pathlib
data = json.loads((pathlib.Path.home() / '.agents/credentials.json').read_text())
e = data['credentials'].get('$name')
print('__missing__' if e is None else json.dumps(e))
")
  [ "$entry" = "__missing__" ] && { echo "credential not configured: $name" >&2; return 1; }

  account=$(echo "$entry" | python3 -c "import json,sys; print(json.load(sys.stdin).get('account','default'))")
  backend=$(echo "$entry" | python3 -c "import json,sys; print(json.load(sys.stdin).get('backend',''))")

  if [ -z "$backend" ]; then
    case "$(uname -s)" in
      Darwin) backend=macos-keychain ;;
      Linux)  backend=secret-service ;;
      *) echo "unsupported platform" >&2; return 1 ;;
    esac
  fi

  case "$backend" in
    macos-keychain)
      security find-generic-password -w -s "agent-skills:$name" -a "$account"
      ;;
    secret-service)
      secret-tool lookup application agent-skills credential "$name" account "$account"
      ;;
    *)
      echo "unsupported backend: $backend" >&2; return 1
      ;;
  esac
}
```

Linux and Windows backends are not yet exercised in this repo. The schema reserves room for them; implementations land when a contributor needs them.

---

## Threat Model

What this design protects against:

| Threat | Protection |
|---|---|
| Casual disk read of the home directory | Index file is `0600`; secrets are not on disk in plaintext |
| Backup theft (Time Machine, iCloud Drive sync) | Secrets live in Keychain, which is encrypted at rest under the user account |
| Accidental git commit of the index file | Pre-commit hook refuses; index discloses metadata only, not secrets |
| Malicious skill or plugin running briefly as the user | Per-credential ACL prompts force per-secret consent; one approval does not unlock other credentials |

What this design does NOT protect against:

- A fully compromised user account with persistent code execution. Such an attacker can repeatedly invoke `security`, harvest "Always Allow" grants the user has already given, and read every credential.
- Memory-scraping malware that reads other processes' memory while a secret is in scope.
- A consuming skill that violates the safe-consumption pattern (e.g. `echo "$bb_token"` into a log).

The security ceiling here is "raise the cost of opportunistic credential theft to roughly that of any other macOS application secret." It is not a hardware-security-module substitute.

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Storing the secret value in `~/.agents/credentials.json` | Promotes the index file from non-secret to secret; defeats the design | Index holds metadata only; secret stays in the keystore |
| `echo "$bb_token"`, `printf`, or `cat` on a line containing a secret | Plaintext leaks to stdout, logs, terminal scrollback, and agent transcripts | Capture into a shell variable; use directly via `-u "$user:$token"` or env var; `unset` after |
| Running `set -x` while a secret is in scope | xtrace prints every expansion, including the secret | Disable xtrace before the read; re-enable only after `unset` |
| Passing the secret as a literal `curl` flag value | Visible in `ps` output to the same user | Use `-u "$user:$token"` (variable expansion is short-lived in argv); prefer env-var auth where supported |
| Mode `0644` on `~/.agents/credentials.json` | Other users on the machine can enumerate configured services | Enforce `0600` on every write; verify with `stat` before reading |
| Editing the index to use a custom keystore service name | Diverges from the keying convention; entry becomes unreadable to every consuming skill | Service names are conventional — `agent-skills:<name>`. Don't fight it. |
| Syncing `~/.agents/` across machines (iCloud, Dropbox, syncthing) | Keystore items are not portable; the index ends up referencing secrets that don't exist on the target machine | Keep `~/.agents/` local; re-add credentials per machine |
| Falling back to plaintext on first-run failure | Quietly downgrades the security posture | Fail loud; require the user to fix the keystore problem before proceeding |
| Calling `security` in CI | Hangs or fails opaquely without a GUI session | Branch on the environment; read from CI-provided env vars instead |
| Committing the pre-commit hook bypass (`--no-verify`) into muscle memory | Defeats the accidental-commit defense | Reserve `--no-verify` for genuine false positives; investigate every trigger |
