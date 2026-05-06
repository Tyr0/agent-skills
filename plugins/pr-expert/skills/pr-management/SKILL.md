---
name: pr-management
description: Use this skill whenever you need to create, list, view, update, merge, or decline a pull request on GitHub or Bitbucket. Also use it for viewing PR diffs, checking merge conflict status, viewing CI/build statuses, listing PR commits, syncing a PR branch with its base, or declining/closing a PR. Triggers on 'create a PR', 'open a pull request', 'merge this PR', 'list open PRs', 'check if PR has conflicts', 'view the diff', 'what is the CI status', 'close this PR', 'update the PR description', or any question about managing the lifecycle of a pull request on GitHub or Bitbucket.
---

# Pull Request Management

Create, list, update, merge, decline, and inspect pull requests on GitHub and Bitbucket Cloud via their REST APIs.

---

## Prerequisites

Before calling any endpoint, complete the steps in the `pr-setup` skill:
1. Detect the platform from `git remote get-url origin`
2. Read credentials via the `credential-storage` skill (entries `github` and `bitbucket`)
3. Set the correct base URL and auth headers

All examples below use placeholder variables: `$TOKEN` (GitHub), `$BB_EMAIL:$BB_API_TOKEN` (Bitbucket), `{owner}` / `{workspace}`, `{repo}`, and `{id}` (PR number/ID).

---

## List Pull Requests

### GitHub

```bash
curl -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/{owner}/{repo}/pulls?state=open&per_page=50"
```

Query params: `state` (open/closed/all), `head`, `base`, `sort` (created/updated/popularity/long-running), `direction` (asc/desc).

### Bitbucket

```bash
curl -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests?state=OPEN&pagelen=50"
```

Query params: `state` (OPEN/MERGED/DECLINED/SUPERSEDED — repeatable), `q` (filter expression), `sort` (e.g., `-updated_on`).

**Bitbucket filter examples:**
```
q=source.branch.name="feature/login"
q=destination.branch.name="main"
q=author.uuid="{user-uuid}"
q=state="OPEN" AND source.branch.name="feature/login"
```

---

## Get a Single Pull Request

### GitHub

```bash
curl -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{id}"
```

Key response fields: `number`, `title`, `body`, `state`, `head`, `base`, `mergeable` (boolean or null), `mergeable_state`, `draft`, `user`, `requested_reviewers`.

### Bitbucket

```bash
curl -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}"
```

Key response fields: `id`, `title`, `description`, `state`, `source`, `destination`, `author`, `reviewers`, `participants`, `close_source_branch`, `links`.

---

## Create a Pull Request

### GitHub

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/{owner}/{repo}/pulls" \
     -d '{
       "title": "Fix login timeout",
       "body": "## Summary\n- Resolves session expiration bug\n\n## Test Plan\n- Manual login test",
       "head": "tcalderone/fix-login-timeout",
       "base": "main",
       "draft": false
     }'
```

Required: `title`, `head` (source branch), `base` (target branch). Optional: `body`, `draft`, `maintainer_can_modify`.

### Bitbucket

```bash
curl -X POST -u "$BB_EMAIL:$BB_API_TOKEN" \
     -H "Content-Type: application/json" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests" \
     -d '{
       "title": "Fix login timeout",
       "description": "## Summary\n- Resolves session expiration bug\n\n## Test Plan\n- Manual login test",
       "source": { "branch": { "name": "tcalderone/fix-login-timeout" } },
       "destination": { "branch": { "name": "main" } },
       "close_source_branch": true,
       "reviewers": [ { "uuid": "{user-uuid}" } ]
     }'
```

Required: `title`, `source.branch.name`. Defaults to repo default branch for `destination` if omitted.

---

## Update a Pull Request

### GitHub

```bash
curl -X PATCH -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{id}" \
     -d '{
       "title": "Updated title",
       "body": "Updated description",
       "state": "open"
     }'
```

Updatable: `title`, `body`, `state` (open/closed), `base`, `maintainer_can_modify`.

### Bitbucket

```bash
curl -X PUT -u "$BB_EMAIL:$BB_API_TOKEN" \
     -H "Content-Type: application/json" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}" \
     -d '{
       "title": "Updated title",
       "description": "Updated description",
       "reviewers": [ { "uuid": "{user-uuid}" } ],
       "close_source_branch": true
     }'
```

Updatable: `title`, `description`, `reviewers`, `close_source_branch`, `destination.branch.name`.

---

## Merge a Pull Request

### GitHub

```bash
curl -X PUT -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{id}/merge" \
     -d '{
       "merge_method": "squash",
       "commit_title": "Fix login timeout (#42)",
       "commit_message": "Resolves session expiration bug"
     }'
```

| `merge_method` | Behavior |
|---|---|
| `merge` | Standard merge commit (`--no-ff`) |
| `squash` | Squash all commits into one |
| `rebase` | Rebase onto base branch |

Optional `sha` field: if provided, must match HEAD of the PR branch (prevents merging stale state).

### Bitbucket

```bash
curl -X POST -u "$BB_EMAIL:$BB_API_TOKEN" \
     -H "Content-Type: application/json" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/merge" \
     -d '{
       "merge_strategy": "squash",
       "close_source_branch": true,
       "message": "Fix login timeout (PR #42)"
     }'
```

| `merge_strategy` | Behavior |
|---|---|
| `merge_commit` | Standard merge (`--no-ff`) |
| `squash` | Squash all commits into one |
| `fast_forward` | Fast-forward only (`--ff-only`) |

---

## Close / Decline a Pull Request

### GitHub

Close by setting the state to `closed`:

```bash
curl -X PATCH -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{id}" \
     -d '{ "state": "closed" }'
```

### Bitbucket

```bash
curl -X POST -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/decline"
```

---

## Check if a PR Is Merged

### GitHub

```bash
curl -s -o /dev/null -w "%{http_code}" \
     -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{id}/merge"
# 204 = merged, 404 = not merged
```

### Bitbucket

Check the `state` field on the PR object — `MERGED` if merged.

---

## View Diffs

### GitHub — Raw unified diff

```bash
curl -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github.diff" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{id}"
```

### GitHub — Structured file list with patches

```bash
curl -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{id}/files?per_page=100"
```

Each file entry: `filename`, `status` (added/removed/modified/renamed), `additions`, `deletions`, `changes`, `patch`.

### Bitbucket — Raw unified diff

```bash
curl -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/diff"
```

### Bitbucket — Structured diffstat (JSON)

```bash
curl -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/diffstat"
```

Each file entry: `status` (added/removed/modified/renamed), `old.path`, `new.path`, `lines_added`, `lines_removed`.

---

## Merge Conflicts and Mergeability

### GitHub

The `mergeable` field on a PR object indicates conflict status:

| Value | Meaning |
|---|---|
| `true` | No conflicts, can be merged |
| `false` | Has merge conflicts |
| `null` | GitHub is still computing — poll again after a short delay |

```bash
# Fetch PR and check mergeability
curl -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{id}" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'mergeable={d[\"mergeable\"]} state={d[\"mergeable_state\"]}')"
```

### GitHub — Sync PR branch with base

```bash
curl -X PUT -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{id}/update-branch"
```

### Bitbucket

Attempting to merge a PR with conflicts returns an HTTP error. Check for conflicts by attempting a merge with `--dry-run` logic or inspecting the PR `state` and error responses. The PR object itself does not expose a dedicated `mergeable` boolean — conflict detection is implicit in the merge attempt response.

---

## CI / Build Status

### GitHub — Combined commit status

```bash
curl -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/commits/{head_sha}/status"
```

Response includes `state`: `pending`, `success`, `failure`, or `error`.

### GitHub — Check runs (GitHub Actions, etc.)

```bash
curl -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/commits/{head_sha}/check-runs"
```

Each check run: `name`, `status` (queued/in_progress/completed), `conclusion` (success/failure/neutral/cancelled/skipped/timed_out/action_required).

### GitHub — Re-request a check run

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/check-runs/{check_run_id}/rerequest"
```

### Bitbucket — Build statuses on a PR

```bash
curl -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/statuses"
```

Each status: `state` (SUCCESSFUL/FAILED/INPROGRESS/STOPPED), `key`, `name`, `url`, `description`.

### Bitbucket — Build status on a specific commit

```bash
curl -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/commit/{hash}/statuses/build"
```

---

## List PR Commits

### GitHub

```bash
curl -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{id}/commits?per_page=100"
```

Max 250 commits per PR.

### Bitbucket

```bash
curl -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/commits"
```

Paginated — follow `next` for additional pages.

---

## PR Activity Feed (Bitbucket only)

Bitbucket provides a chronological feed of all PR events (comments, approvals, updates, merges):

```bash
curl -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/activity"
```

GitHub equivalent: use the Timeline API or list events on the issue.

---

## Cross-Platform Quick Reference

| Operation | GitHub | Bitbucket |
|---|---|---|
| List PRs | `GET /repos/{o}/{r}/pulls` | `GET /repositories/{w}/{r}/pullrequests` |
| Get PR | `GET /repos/{o}/{r}/pulls/{id}` | `GET /repositories/{w}/{r}/pullrequests/{id}` |
| Create PR | `POST /repos/{o}/{r}/pulls` | `POST /repositories/{w}/{r}/pullrequests` |
| Update PR | `PATCH /repos/{o}/{r}/pulls/{id}` | `PUT /repositories/{w}/{r}/pullrequests/{id}` |
| Merge PR | `PUT /repos/{o}/{r}/pulls/{id}/merge` | `POST /repositories/{w}/{r}/pullrequests/{id}/merge` |
| Close/Decline | `PATCH ...pulls/{id}` state=closed | `POST .../pullrequests/{id}/decline` |
| Diff (raw) | `Accept: application/vnd.github.diff` | `GET .../pullrequests/{id}/diff` |
| Diff (structured) | `GET ...pulls/{id}/files` | `GET .../pullrequests/{id}/diffstat` |
| Commits | `GET ...pulls/{id}/commits` | `GET .../pullrequests/{id}/commits` |
| CI status | `GET /repos/{o}/{r}/commits/{sha}/check-runs` | `GET .../pullrequests/{id}/statuses` |
| Mergeability | `.mergeable` field on PR object | Implicit in merge attempt response |
| Source branch | `head` field | `source.branch.name` field |
| Target branch | `base` field | `destination.branch.name` field |

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Merging without checking CI status | Broken builds land on main | Always check build/check-run status before merging |
| Not verifying `mergeable` (GitHub) before merging | Merge conflicts cause 405/409 errors | Fetch PR, check `mergeable` field; if `null`, poll briefly |
| Using `PUT` for Bitbucket updates when GitHub uses `PATCH` | 405 Method Not Allowed | GitHub uses `PATCH` for updates; Bitbucket uses `PUT` |
| Forgetting `Content-Type: application/json` on POST/PUT | 415 or silent failures on Bitbucket | Always include the header on requests with JSON bodies |
| Merging without `sha` validation (GitHub) | Merging outdated PR state | Pass the HEAD SHA in the merge request to prevent stale merges |
| Ignoring `close_source_branch` on Bitbucket | Stale feature branches accumulate | Set `close_source_branch: true` when creating or merging |
| Assuming `mergeable: null` means conflict | GitHub is still computing | Poll the PR endpoint again after a brief delay |
