# Line-Specific PR/MR Comments

API patterns for line-specific review comments. These are not available via CLI subcommands on either provider.

## GitHub: Single-Line Comment

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments \
  -f body="Comment text" \
  -f path="src/file.ts" \
  -F line=42 \
  -f side=RIGHT \
  -f commit_id="$(gh pr view {pr} --json headRefOid --jq .headRefOid)"
```

Parameters:
- `body` -- comment text (markdown supported)
- `path` -- file path relative to repo root
- `line` -- line number in the diff
- `side` -- `RIGHT` (new code) or `LEFT` (old code, for deletions)
- `commit_id` -- the commit SHA to comment on (use latest commit)

## GitHub: Multi-Line Range Comment

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments \
  -f body="This entire block should be refactored" \
  -f path="src/file.ts" \
  -F line=50 \
  -f side=RIGHT \
  -F start_line=42 \
  -f start_side=RIGHT \
  -f commit_id="$(gh pr view {pr} --json headRefOid --jq .headRefOid)"
```

Additional parameters:
- `start_line` -- first line of the range
- `start_side` -- side for start line (`RIGHT` or `LEFT`)
- `line` -- last line of the range (the comment anchor)

## GitHub: Reply to a Comment Thread

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="Reply text"
```

The `comment_id` is the numeric ID of the comment you're replying to. Get it from listing comments.

## GitHub: List Review Comments

```bash
# All review comments on a PR
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate

# Filter by path (client-side with jq)
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate \
  --jq '.[] | select(.path == "src/file.ts") | {id, line, body, user: .user.login}'
```

Response fields:
- `id` -- comment ID (use for replies)
- `body` -- comment text
- `path` -- file path
- `line` -- line number
- `side` -- LEFT or RIGHT
- `start_line` -- start of range (if multi-line)
- `user.login` -- author
- `in_reply_to_id` -- parent comment (if reply)

## GitHub: Edit a Review Comment

```bash
gh api repos/{owner}/{repo}/pulls/comments/{comment_id} \
  --method PATCH \
  -f body="Updated comment text"
```

Note: the endpoint for editing uses `/pulls/comments/{id}` (not `/pulls/{pr}/comments/{id}`).

## GitHub: Delete a Review Comment

```bash
gh api repos/{owner}/{repo}/pulls/comments/{comment_id} \
  --method DELETE
```

## GitHub: Batch Review with Multiple Comments

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

## GitHub: Getting the Latest Commit SHA

Line comments require a `commit_id`. Get the latest commit on the PR:

```bash
# From HEAD ref (preferred)
gh pr view {pr} --json headRefOid --jq .headRefOid

# From commits list
gh pr view {pr} --json commits --jq '.commits[-1].oid'
```

---

## GitLab: Single-Line Comment

```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions --method POST \
  --field "body={comment}" \
  --field "position[base_sha]={base_sha}" \
  --field "position[head_sha]={head_sha}" \
  --field "position[start_sha]={base_sha}" \
  --field "position[position_type]=text" \
  --field "position[new_path]={file}" \
  --field "position[new_line]={line}"
```

Parameters:
- `body` -- comment text (markdown supported)
- `position[base_sha]` -- SHA of the base commit (target branch HEAD)
- `position[head_sha]` -- SHA of the head commit (source branch HEAD)
- `position[start_sha]` -- same as base_sha for simple diffs
- `position[position_type]` -- always `text` for code comments
- `position[new_path]` -- file path for new/modified lines
- `position[new_line]` -- line number in the new version

### Getting SHAs for GitLab

```bash
# Get diff refs from the MR
glab mr view {iid} -F json | jq '{base:.diff_refs.base_sha,head:.diff_refs.head_sha,start:.diff_refs.start_sha}'
```

## GitLab: Comment on Deleted Line

Use `old_path` and `old_line` instead of `new_path` and `new_line`:

```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions --method POST \
  --field "body={comment}" \
  --field "position[base_sha]={base_sha}" \
  --field "position[head_sha]={head_sha}" \
  --field "position[start_sha]={base_sha}" \
  --field "position[position_type]=text" \
  --field "position[old_path]={file}" \
  --field "position[old_line]={line}"
```

## GitLab: Reply to a Discussion

```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id}/notes \
  --method POST --field "body=Reply text"
```

## GitLab: List MR Discussions

```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions --paginate | jq '[.[] | select(.notes[0].type == "DiffNote") | {id:.id,path:.notes[0].position.new_path,line:.notes[0].position.new_line,body:.notes[0].body,author:.notes[0].author.username}]'
```

## GitLab: Resolve a Discussion

```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id} \
  --method PUT --field "resolved=true"
```
