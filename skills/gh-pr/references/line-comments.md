# Line-Specific PR Comments

`gh api` patterns for line-specific PR review comments. These are not available via `gh` subcommands.

## Single-Line Comment

```bash
# Comment on a specific line in the latest commit
gh api repos/{owner}/{repo}/pulls/{pr}/comments \
  -f body="Comment text" \
  -f path="src/file.ts" \
  -F line=42 \
  -f side=RIGHT \
  -f commit_id="$(gh pr view {pr} --json commits --jq '.commits[-1].oid')"
```

Parameters:
- `body` — comment text (markdown supported)
- `path` — file path relative to repo root
- `line` — line number in the diff
- `side` — `RIGHT` (new code) or `LEFT` (old code, for deletions)
- `commit_id` — the commit SHA to comment on (use latest commit)

## Multi-Line Range Comment

```bash
# Comment spanning multiple lines
gh api repos/{owner}/{repo}/pulls/{pr}/comments \
  -f body="This entire block should be refactored" \
  -f path="src/file.ts" \
  -F line=50 \
  -f side=RIGHT \
  -F start_line=42 \
  -f start_side=RIGHT \
  -f commit_id="$(gh pr view {pr} --json commits --jq '.commits[-1].oid')"
```

Additional parameters:
- `start_line` — first line of the range
- `start_side` — side for start line (`RIGHT` or `LEFT`)
- `line` — last line of the range (the comment anchor)

## Reply to a Comment Thread

```bash
# Reply to an existing review comment (creates thread)
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="Reply text"
```

The `comment_id` is the numeric ID of the comment you're replying to. Get it from listing comments.

## List Review Comments

```bash
# All review comments on a PR
gh api repos/{owner}/{repo}/pulls/{pr}/comments

# With pagination
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate

# Filter by path (client-side with jq)
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate \
  --jq '.[] | select(.path == "src/file.ts") | {id, line, body, user: .user.login}'
```

Response fields:
- `id` — comment ID (use for replies)
- `body` — comment text
- `path` — file path
- `line` — line number
- `side` — LEFT or RIGHT
- `start_line` — start of range (if multi-line)
- `user.login` — author
- `created_at` — timestamp
- `in_reply_to_id` — parent comment (if reply)

## Edit a Review Comment

```bash
gh api repos/{owner}/{repo}/pulls/comments/{comment_id} \
  --method PATCH \
  -f body="Updated comment text"
```

Note: the endpoint for editing uses `/pulls/comments/{id}` (not `/pulls/{pr}/comments/{id}`).

## Delete a Review Comment

```bash
gh api repos/{owner}/{repo}/pulls/comments/{comment_id} \
  --method DELETE
```

## Create a Review with Multiple Comments

Submit multiple line comments as part of a single review:

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/reviews \
  --method POST \
  -f event=COMMENT \
  -f body="Review summary" \
  --input <(echo '{
    "comments": [
      {
        "path": "src/file.ts",
        "line": 42,
        "side": "RIGHT",
        "body": "First comment"
      },
      {
        "path": "src/other.ts",
        "line": 10,
        "side": "RIGHT",
        "body": "Second comment"
      }
    ]
  }')
```

Events: `COMMENT`, `APPROVE`, `REQUEST_CHANGES`.

## Getting the Latest Commit SHA

Line comments require a `commit_id`. Get the latest commit on the PR:

```bash
# Latest commit SHA
gh pr view 123 --json commits --jq '.commits[-1].oid'

# Or from the HEAD ref
gh pr view 123 --json headRefOid --jq '.headRefOid'
```
