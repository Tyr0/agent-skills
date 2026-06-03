---
name: pr-setup
description: Use this skill whenever you need to authenticate with or configure access to a pull request hosting platform (GitHub or Bitbucket). Also use it to detect which platform a repository is hosted on, resolve authentication errors (401/403), set up tokens or API tokens, or register PR-platform credentials via the credential-storage skill. Triggers on 'how do I authenticate with Bitbucket', 'set up GitHub token', 'PR auth error', '401 from Bitbucket API', 'configure pull request access', or any question about connecting to a PR platform's REST API.
---

# Pull Request Platform Setup

Authentication, platform detection, and credential configuration for GitHub and Bitbucket Cloud REST APIs.

Credentials are managed by the `credential-storage` skill. This skill describes the platform-specific concerns: which credentials to register, how to detect the platform from the git remote, and how to construct authenticated requests.

---

## Platform Detection

Determine the hosting platform from the git remote URL before making any API calls.

```bash
# Get the remote URL
git remote get-url origin
```

| URL pattern | Platform | Workspace/Owner | Repo |
|---|---|---|---|
| `git@github.com:{owner}/{repo}.git` | GitHub | `{owner}` | `{repo}` |
| `https://github.com/{owner}/{repo}.git` | GitHub | `{owner}` | `{repo}` |
| `git@bitbucket.org:{workspace}/{repo}.git` | Bitbucket | `{workspace}` | `{repo}` |
| `https://bitbucket.org/{workspace}/{repo_slug}.git` | Bitbucket | `{workspace}` | `{repo_slug}` |
| `https://{user}@bitbucket.org/{workspace}/{repo_slug}.git` | Bitbucket | `{workspace}` | `{repo_slug}` |

### Extracting components

```bash
# Parse owner and repo from remote URL
remote_url=$(git remote get-url origin)

# GitHub SSH
# git@github.com:owner/repo.git -> owner repo
echo "$remote_url" | sed -n 's|git@github.com:\([^/]*\)/\(.*\)\.git|\1 \2|p'

# GitHub HTTPS
# https://github.com/owner/repo.git -> owner repo
echo "$remote_url" | sed -n 's|https://github.com/\([^/]*\)/\(.*\)\.git|\1 \2|p'

# Bitbucket SSH
# git@bitbucket.org:workspace/repo.git -> workspace repo
echo "$remote_url" | sed -n 's|git@bitbucket.org:\([^/]*\)/\(.*\)\.git|\1 \2|p'

# Bitbucket HTTPS (with or without username@)
echo "$remote_url" | sed -n 's|https://\([^@]*@\)\?bitbucket.org/\([^/]*\)/\(.*\)\.git|\2 \3|p'
```

---

## Credentials

This skill consumes two credentials registered via the `credential-storage` skill:

| Name | Account | Stores |
|---|---|---|
| `github` | `default` | GitHub fine-grained personal access token |
| `bitbucket` | Atlassian email | Bitbucket Cloud API token |

### Reading credentials (macOS)

GitHub:

```bash
TOKEN=$(security find-generic-password -w -s 'agent-skills:github' -a 'default')
```

Bitbucket — the email lives in the index entry's `account` field; the token is in Keychain under that account:

```bash
BB_EMAIL=$(python3 -c "
import json, pathlib
data = json.loads((pathlib.Path.home() / '.agents/credentials.json').read_text())
print(data['credentials']['bitbucket']['account'])
")
BB_API_TOKEN=$(security find-generic-password -w -s 'agent-skills:bitbucket' -a "$BB_EMAIL")
```

After API calls, `unset TOKEN BB_API_TOKEN` to remove secrets from the shell environment.

For non-macOS platforms, dispatch via `uname -s` to the matching keystore. See the `credential-storage` skill's **Platform Dispatch** section.

### First-time setup

If a credential is not yet registered, `security` exits with a non-zero status. Guide the user through obtaining the token, then register it.

**GitHub:**
1. Go to GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens.
2. Create a token with **Pull requests: read and write** permission, scoped to the target repositories.
3. Register the token (paste at the prompt):

   ```bash
   security add-generic-password -U -s 'agent-skills:github' -a 'default'
   ```

4. Add an index entry:

   ```bash
   python3 - <<'PY'
   import json, os, pathlib
   p = pathlib.Path.home() / ".agents" / "credentials.json"
   d = json.loads(p.read_text()) if p.exists() else {"version": 1, "credentials": {}}
   d["credentials"]["github"] = {}
   t = p.with_suffix(".json.tmp")
   t.write_text(json.dumps(d, indent=2) + "\n")
   os.chmod(t, 0o600)
   t.replace(p)
   PY
   ```

**Bitbucket:**
1. Go to your Atlassian profile → Account settings → Security → **Create and manage API tokens**.
2. Create an API token with Bitbucket **Pull requests: Read** and **Pull requests: Write** scopes.
3. Register the token under your Atlassian email (replace `<email>`):

   ```bash
   security add-generic-password -U -s 'agent-skills:bitbucket' -a '<email>'
   ```

4. Add an index entry:

   ```bash
   python3 - <<'PY'
   import json, os, pathlib
   p = pathlib.Path.home() / ".agents" / "credentials.json"
   d = json.loads(p.read_text()) if p.exists() else {"version": 1, "credentials": {}}
   d["credentials"]["bitbucket"] = {"account": "<email>"}
   t = p.with_suffix(".json.tmp")
   t.write_text(json.dumps(d, indent=2) + "\n")
   os.chmod(t, 0o600)
   t.replace(p)
   PY
   ```

If `~/.agents/` does not yet exist, create it first per the `credential-storage` skill's setup section (`mkdir -p ~/.agents && chmod 700 ~/.agents`).

---

## Base URLs and Authentication Headers

### GitHub

```
Base URL: https://api.github.com
```

```bash
curl -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     -H "X-GitHub-Api-Version: 2022-11-28" \
     "https://api.github.com/repos/{owner}/{repo}/pulls"
```

### Bitbucket Cloud

```
Base URL: https://api.bitbucket.org/2.0
```

```bash
curl -u "$BB_EMAIL:$BB_API_TOKEN" \
     -H "Content-Type: application/json" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests"
```

---

## Error Handling

| HTTP Status | Meaning | Action |
|---|---|---|
| 401 | Invalid or expired token | Re-check the credential via `credential-storage`; rotate the token if needed |
| 403 | Insufficient permissions | Check token scopes — needs PR read/write |
| 404 | Repo not found or no access | Verify workspace/owner and repo slug; check token scope includes the repo |
| 422 | Validation error | Read the error message body for field-level details |
| 429 | Rate limited | Wait and retry; GitHub includes `X-RateLimit-Reset` header |

### Debugging auth issues

```bash
# GitHub — verify token works
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $TOKEN" \
  https://api.github.com/user

# Bitbucket — verify credentials work
curl -s -o /dev/null -w "%{http_code}" \
  -u "$BB_EMAIL:$BB_API_TOKEN" \
  https://api.bitbucket.org/2.0/user
```

---

## Pagination

### GitHub

Page-based. Use `per_page` (max 100) and `page` query params. Follow the `rel="next"` URL in the `Link` response header until absent.

```bash
# First page
curl -H "Authorization: Bearer $TOKEN" \
  "https://api.github.com/repos/{owner}/{repo}/pulls?per_page=100&page=1"
# Check Link header for next page URL
```

### Bitbucket

Page-based. Use `pagelen` (max 100) and `page` query params. Follow the `next` URL in the JSON response until absent.

```bash
# First page
curl -u "$BB_EMAIL:$BB_API_TOKEN" \
  "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests?pagelen=50&page=1"
# Check .next in JSON response for next page URL
```

---

## Quick Reference

| Concept | GitHub | Bitbucket |
|---|---|---|
| Base URL | `https://api.github.com` | `https://api.bitbucket.org/2.0` |
| Auth header | `Authorization: Bearer <token>` | Basic auth (`-u email:api_token`) |
| Credential name | `github` | `bitbucket` |
| Credential account | `default` | Atlassian email |
| PR path prefix | `/repos/{owner}/{repo}/pulls` | `/repositories/{workspace}/{repo_slug}/pullrequests` |
| PR identifier | `pull_number` (integer) | `pull_request_id` (integer) |
| Pagination | `page` + `per_page`; `Link` header | `page` + `pagelen`; `next` in JSON body |
| Token type | Fine-grained PAT with PR permissions | API token with PR read/write scopes |

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Hardcoding tokens in scripts or skill files | Tokens leak into version control | Read via `credential-storage` at runtime; never embed token values in code |
| Echoing or logging the token after fetch | Plaintext leaks to stdout, scrollback, and agent transcripts | Capture into a shell variable; use directly via header or `-u`; `unset` after |
| Assuming `gh` CLI is available | `gh` is GitHub-only and may not be installed | Use `curl` against the REST API for both platforms |
| Skipping platform detection | Wrong API endpoints, confusing errors | Always parse the git remote URL first |
| Ignoring pagination | Missing PRs/comments beyond the first page | Follow `next`/`Link` header until exhausted |
| Using classic GitHub tokens without repo scope | 404 errors on private repos | Use fine-grained tokens with explicit repo + PR permissions |
| Calling endpoints before credentials are registered | Cryptic curl errors with empty auth | Verify both `github` and `bitbucket` entries exist; guide the user through registration if missing |
| Falling back to plaintext token files on lookup failure | Quietly downgrades the security posture | Fail loud; require the user to register the credential properly |
