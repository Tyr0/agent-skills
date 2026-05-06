---
name: pr-review
description: Use this skill whenever you need to comment on, review, approve, or request changes on a pull request on GitHub or Bitbucket. Also use it for posting inline code comments on specific lines, replying to existing comment threads, resolving or unresolving conversations, creating tasks on Bitbucket PRs, or reading all comments and review feedback on a PR. Triggers on 'comment on this PR', 'approve the pull request', 'request changes', 'post an inline comment', 'reply to a PR comment', 'resolve this thread', 'read PR comments', 'add a task to this PR', or any question about reviewing, commenting on, or giving feedback on a pull request.
---

# Pull Request Review and Comments

Post comments, submit reviews, approve or request changes, and manage comment threads on GitHub and Bitbucket Cloud PRs.

---

## Prerequisites

Before calling any endpoint, complete the steps in the `pr-setup` skill:
1. Detect the platform from `git remote get-url origin`
2. Read credentials via the `credential-storage` skill (entries `github` and `bitbucket`)
3. Set the correct base URL and auth headers

---

## Post a General Comment

A top-level comment on the PR conversation thread (not tied to a specific line of code).

### GitHub

GitHub treats PRs as issues for general comments:

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/{owner}/{repo}/issues/{pr_number}/comments" \
     -d '{ "body": "Looks good overall. One suggestion below." }'
```

### Bitbucket

```bash
curl -X POST -u "$BB_EMAIL:$BB_API_TOKEN" \
     -H "Content-Type: application/json" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/comments" \
     -d '{
       "content": { "raw": "Looks good overall. One suggestion below." }
     }'
```

---

## Post an Inline Code Comment

A comment attached to a specific file and line in the diff.

### GitHub

Inline comments are "pull request review comments":

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{pr_number}/comments" \
     -d '{
       "body": "Consider renaming this variable for clarity.",
       "commit_id": "abc123def456...",
       "path": "src/auth/login.rb",
       "line": 42,
       "side": "RIGHT"
     }'
```

| Field | Required | Description |
|---|---|---|
| `body` | Yes | Comment text (markdown) |
| `commit_id` | Yes | Full SHA of the commit being commented on |
| `path` | Yes | File path relative to repo root |
| `line` | Yes | Line number in the diff (use `side` to specify which version) |
| `side` | No | `LEFT` (old/removed) or `RIGHT` (new/added, default) |
| `start_line` | No | For multi-line comments: first line of the range |
| `start_side` | No | Side for the start line |
| `subject_type` | No | `"file"` for file-level comments (omit `line` in that case) |

### Bitbucket

```bash
curl -X POST -u "$BB_EMAIL:$BB_API_TOKEN" \
     -H "Content-Type: application/json" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/comments" \
     -d '{
       "content": { "raw": "Consider renaming this variable for clarity." },
       "inline": {
         "path": "src/auth/login.rb",
         "to": 42
       }
     }'
```

| `inline` field | Description |
|---|---|
| `path` | File path relative to repo root (required) |
| `to` | Line number in the new version of the file (comment on added/current lines) |
| `from` | Line number in the old version (comment on removed lines) |

Use `to` for new/current lines, `from` for deleted lines. Do not set both simultaneously.

---

## Read Comments on a PR

### GitHub

GitHub has two separate comment collections — fetch both for complete coverage:

```bash
# General conversation comments
curl -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/issues/{pr_number}/comments?per_page=100"

# Inline review comments
curl -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{pr_number}/comments?per_page=100"
```

### Bitbucket

All comments (general and inline) are in a single collection:

```bash
curl -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/comments?pagelen=100"
```

Inline comments have an `inline` object in the response; general comments do not.

---

## Reply to an Existing Comment

### GitHub — Reply to an inline review comment

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{pr_number}/comments/{comment_id}/replies" \
     -d '{ "body": "Good point, fixed in the latest commit." }'
```

General/issue comments do not support threaded replies — post a new comment that quotes or references the original.

### Bitbucket — Reply to any comment

```bash
curl -X POST -u "$BB_EMAIL:$BB_API_TOKEN" \
     -H "Content-Type: application/json" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/comments" \
     -d '{
       "content": { "raw": "Good point, fixed in the latest commit." },
       "parent": { "id": 12345 }
     }'
```

Set `parent.id` to the ID of the comment being replied to.

---

## Update or Delete a Comment

### GitHub

```bash
# Update a general comment
curl -X PATCH -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/issues/comments/{comment_id}" \
     -d '{ "body": "Updated text." }'

# Update an inline review comment
curl -X PATCH -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/comments/{comment_id}" \
     -d '{ "body": "Updated text." }'

# Delete (same pattern with DELETE method)
curl -X DELETE -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/issues/comments/{comment_id}"
```

### Bitbucket

```bash
# Update
curl -X PUT -u "$BB_EMAIL:$BB_API_TOKEN" \
     -H "Content-Type: application/json" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/comments/{comment_id}" \
     -d '{ "content": { "raw": "Updated text." } }'

# Delete
curl -X DELETE -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/comments/{comment_id}"
```

---

## Submit a Review (Approve / Request Changes)

### GitHub — Create and submit a review

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{pr_number}/reviews" \
     -d '{
       "event": "APPROVE",
       "body": "Ship it."
     }'
```

| `event` value | Meaning |
|---|---|
| `APPROVE` | Approve the PR |
| `REQUEST_CHANGES` | Request changes (blocks merge in many configurations) |
| `COMMENT` | General review comment without approval/rejection |

### GitHub — Review with inline comments in a single request

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{pr_number}/reviews" \
     -d '{
       "event": "REQUEST_CHANGES",
       "body": "A few things to address.",
       "comments": [
         {
           "path": "src/auth/login.rb",
           "line": 42,
           "body": "This needs a nil check."
         },
         {
           "path": "src/auth/session.rb",
           "line": 15,
           "body": "Extract this into a private method."
         }
       ]
     }'
```

### GitHub — Dismiss a review

```bash
curl -X PUT -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{pr_number}/reviews/{review_id}/dismissals" \
     -d '{ "message": "Reviewer no longer on the project." }'
```

### Bitbucket — Approve

```bash
curl -X POST -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/approve"
```

### Bitbucket — Remove approval

```bash
curl -X DELETE -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/approve"
```

### Bitbucket — Request changes

```bash
curl -X POST -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/request-changes"
```

### Bitbucket — Remove request for changes

```bash
curl -X DELETE -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/request-changes"
```

---

## List Reviews on a PR (GitHub only)

```bash
curl -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{pr_number}/reviews"
```

Each review: `id`, `user`, `body`, `state` (APPROVED/CHANGES_REQUESTED/COMMENTED/DISMISSED/PENDING).

### List comments within a specific review

```bash
curl -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/repos/{owner}/{repo}/pulls/{pr_number}/reviews/{review_id}/comments"
```

### Bitbucket equivalent

Bitbucket does not have a separate "review" object. Approvals, change requests, and comments are tracked on the PR's `participants` list and comment collection. To see who approved:

```bash
curl -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}" \
  | python3 -c "
import sys, json
pr = json.load(sys.stdin)
for p in pr.get('participants', []):
    print(f\"{p['user']['display_name']}: approved={p.get('approved', False)} role={p.get('role', 'PARTICIPANT')} state={p.get('state', 'N/A')}\")
"
```

---

## Resolve / Unresolve Conversations

### GitHub — Requires GraphQL API

The REST API does not support resolving threads. Use the GraphQL endpoint:

```bash
# Resolve a thread
curl -X POST -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/graphql" \
     -d '{
       "query": "mutation { resolveReviewThread(input: { pullRequestReviewThreadId: \"THREAD_NODE_ID\" }) { thread { isResolved } } }"
     }'

# Unresolve a thread
curl -X POST -H "Authorization: Bearer $TOKEN" \
     "https://api.github.com/graphql" \
     -d '{
       "query": "mutation { unresolveReviewThread(input: { pullRequestReviewThreadId: \"THREAD_NODE_ID\" }) { thread { isResolved } } }"
     }'
```

The `THREAD_NODE_ID` is the `node_id` field from review comment objects (available in REST responses).

### Bitbucket

Comment resolution is tracked via the `resolution` property on the comment object. A resolved comment has a non-null `resolution` field.

---

## Tasks (Bitbucket only)

Bitbucket PRs support tasks — actionable items tied to the PR.

### List tasks

```bash
curl -u "$BB_EMAIL:$BB_API_TOKEN" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/tasks"
```

### Create a task

```bash
curl -X POST -u "$BB_EMAIL:$BB_API_TOKEN" \
     -H "Content-Type: application/json" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pullrequests/{id}/tasks" \
     -d '{
       "content": { "raw": "Add unit tests for the new login flow." }
     }'
```

**Note:** Task API support may be limited — tasks are primarily managed through the Bitbucket UI. Verify endpoint availability before relying on programmatic task creation.

---

## Cross-Platform Quick Reference

| Operation | GitHub | Bitbucket |
|---|---|---|
| General comment | `POST /repos/{o}/{r}/issues/{id}/comments` | `POST /repositories/{w}/{r}/pullrequests/{id}/comments` |
| Inline comment | `POST /repos/{o}/{r}/pulls/{id}/comments` | Same endpoint, add `inline` object |
| Reply to comment | `POST .../comments/{cid}/replies` | Same endpoint, add `parent.id` |
| Read general comments | `GET /repos/{o}/{r}/issues/{id}/comments` | `GET .../pullrequests/{id}/comments` (all in one) |
| Read inline comments | `GET /repos/{o}/{r}/pulls/{id}/comments` | Same endpoint (all in one) |
| Update comment | `PATCH .../comments/{cid}` | `PUT .../comments/{cid}` |
| Delete comment | `DELETE .../comments/{cid}` | `DELETE .../comments/{cid}` |
| Approve | `POST .../reviews` event=APPROVE | `POST .../approve` |
| Unapprove | Dismiss the review | `DELETE .../approve` |
| Request changes | `POST .../reviews` event=REQUEST_CHANGES | `POST .../request-changes` |
| Remove request for changes | Dismiss the review | `DELETE .../request-changes` |
| Resolve thread | GraphQL `resolveReviewThread` | `resolution` field on comment |
| Tasks | Not a native concept | `GET/POST .../tasks` |
| Review with inline comments | Single `POST .../reviews` with `comments` array | Post comments individually |

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Posting inline comments without `commit_id` (GitHub) | 422 validation error | Always include the full SHA of the commit being reviewed |
| Using `/pulls/{id}/comments` for general discussion (GitHub) | Creates orphaned review comments not visible in conversation tab | Use `/issues/{id}/comments` for general comments |
| Posting many individual inline comments instead of a review (GitHub) | Floods the PR author with separate notifications | Batch inline comments into a single review with `comments` array |
| Setting both `from` and `to` in Bitbucket inline comments | Ambiguous line reference | Use `to` for new lines, `from` for removed lines — not both |
| Approving without reading the diff | Rubber-stamp reviews erode trust | Always fetch and review the diff before approving |
| Ignoring the two-collection split on GitHub | Missing comments when reading PR feedback | Fetch both `/issues/{id}/comments` and `/pulls/{id}/comments` |
| Replying to a Bitbucket comment without `parent.id` | Creates a new top-level comment instead of a thread reply | Always set `parent.id` when replying to an existing comment |
| Using REST API for resolving GitHub threads | No such endpoint exists | Use the GraphQL API with `resolveReviewThread` mutation |
