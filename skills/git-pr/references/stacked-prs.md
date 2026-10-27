# Stacked PRs

**A stacked PR targets another PR's branch instead of the default branch** (`main <- parent <- child`). Opening one is easy. Merging is where it goes wrong, and in two ways that cost real money or work: the child PR gets **closed** instead of moving to `main`, and every merge into the default branch starts whatever runs on a push to it (deploys, releases, publishing), so a stack merged one PR at a time runs those pipelines once per PR. **Never merge a PR in a stack without asking the user which way** (SKILL.md, Critical Rule 11).

GitHub only. Commands checked against `gh` v2.103.2 `--help` (2026-10-27), the skill's pin.

## Create a Stack

```bash
gh pr create --base {parent_branch} --title "feat: ..." --body "Stacked on #{parent}. ..."
```

- **CodeRabbit skips the child by default.** Auto-review fires only on PRs based on the default branch (plus `auto_review.base_branches` patterns), so the child's `CodeRabbit` status reads `Review skipped: reviews are disabled for this base branch`, behind a green check (`bot_status` exit `5`). An explicit `@coderabbitai review` comment still runs a full review on it (observed 2026-08-27). Post it when it is the child's turn in the bucket, since it spends a PR review like any other. To review stacks automatically instead, add `base_branches: [".*"]` under `auto_review` in `.coderabbit.yaml`.
- **Name the parent in the child's body.** The base branch is the only other link, and it changes when the stack is merged.

## Before Every Merge: Detect the Stack

```bash
# PRs stacked on this one: their base is this PR's head branch
gh pr list --base "$(gh pr view {pr} --json headRefName --jq .headRefName)" --json number,title,headRefName,isDraft

# Is this PR itself stacked? Its base is not the default branch
gh pr view {pr} --json baseRefName,headRefName,headRefOid
gh repo view --json defaultBranchRef,deleteBranchOnMerge --jq '{default: .defaultBranchRef.name, deleteBranchOnMerge}'
```

| Result | Action |
|--------|--------|
| No PR targets its head branch, base is the default branch | Normal merge (SKILL.md, Merge) |
| Other PRs target its head branch | It is a parent: **stop and ask** (Ask Which Way, below) |
| Its base is another PR's branch | It is a child: merging it lands it on the parent's branch, **not** the default branch. Say so, then ask as below |

## What Goes Wrong

### The Child Gets Closed

**Deleting the parent's branch can close the child instead of retargeting it.** GitHub documents that when a merged PR's head branch is deleted, open PRs based on it are retargeted to the merged PR's base. Do not rely on that: observed in October 2026, merging the parent closed the child PR outright. `gh pr merge --delete-branch` deletes the remote branch through the REST refs API as soon as the merge call returns (gh v2.103.2 source).

Leaving out `--delete-branch` does not protect the child either. With the repo's auto-delete on (`deleteBranchOnMerge: true`), GitHub deletes the branch at merge anyway. **Retargeting the child before the parent merges is the one order that is safe under every setting.**

### Squash and Rebase Merges Leave the Parent's Commits in the Child

A squash or rebase merge lands the parent on the default branch under new SHAs. The child still carries the original commits, so even after it is retargeted, its diff repeats the parent's changes or conflicts until it is rebased:

```bash
git fetch origin
git rebase --onto origin/{default} {parent_head_sha} {child_branch}   # keep only the child's own commits
git push --force-with-lease origin {child_branch}
```

`{parent_head_sha}` is the parent PR's last head (`gh pr view {parent} --json headRefOid --jq .headRefOid`, still readable after the merge). The range `{parent_head_sha}..{child_branch}` is exactly the child's own commits, even if the child was cut from an older parent commit. A merge-commit merge (`--merge`) keeps the parent's SHAs and needs no rebase.

The force-push is a push. It re-runs the child's PR CI, and once the child is based on the default branch it can start a CodeRabbit round on an auto-review repo (One Push Per Round in `bot-review-loop.md`).

### Every Merge to the Default Branch Is a Pipeline Run

Whatever a push to the default branch starts runs once per merge. Merging a two-PR stack bottom-up runs it twice, and the first run ships a state (the parent without the child) that nobody planned to release. Find out what runs before asking:

```bash
# Actions workflows that pushes to the default branch started recently
gh run list --branch {default} --event push --limit 30 --json workflowName --jq '[.[].workflowName] | unique'

# Deployment environments, including external CD apps (Vercel, Netlify, ...) that run no Actions workflow
gh api repos/{owner}/{repo}/deployments --jq '[.[].environment] | unique'
```

A workflow that has not run lately does not show up in the first command. For a complete list, read the `on:` blocks in `.github/workflows/` for `push` to the default branch, `release` and `workflow_run`. A deploy workflow with `concurrency: { cancel-in-progress: true }` cancels the first run when the second merge lands quickly, but both runs still start, and a cancelled deploy can be left half-applied. That does not make bottom-up free.

## Ask Which Way

Unless the repo's instructions already fix the merge order, stop and give the user:

1. The stack: PR numbers, titles, which PR is based on which.
2. What a push to the default branch starts here (the two commands above). If that list is empty, say so.
3. The two ways to merge, with a recommendation:

| | Merge as one (top-down) | Merge one by one (bottom-up) |
|---|---|---|
| Order | Child into the parent's branch, then parent into the default branch | Retarget the child, parent into the default branch, rebase the child, then child into the default branch |
| Pushes to the default branch | 1 | One per PR |
| Default-branch history | One squash commit for the stack (with `--rebase` on the parent: its commits plus the child's squash commit) | One commit per PR, each revertable alone |
| Extra cost | The child's merge is a push to the parent PR: it re-runs the parent's CI and can be a CodeRabbit round there. The parent's merge message has to cover both | The rebase force-push re-runs the child's CI and review. Push pipelines run once per PR |
| Pick when | A push to the default branch deploys or releases, and the stack ships together | Nothing costly runs on push, or each layer must ship or be revertable on its own |

**Recommend top-down whenever the push list contains a deploy or release.** Recommend bottom-up when it is empty or only cheap CI.

## Merge as One (Top-Down)

```bash
gh pr merge {child} --squash --delete-branch      # child lands on the parent's branch; nothing is stacked on it
gh pr checks {parent} --json name,state,bucket,description   # the push re-ran the parent's checks: wait until required ones pass
gh pr merge {parent} --squash --delete-branch --subject "{type}: {summary of both layers} (#{parent})"
```

- **Check the child before its merge.** Its CI and review state matter even though its base is not protected: a non-default branch usually has no branch protection, so nothing else stops the merge.
- **The parent's squash commit is the record of both layers.** Set `--subject`/`--body`, or edit the parent's title and body first (`gh pr edit {parent} --title ... --body ...`). Use `--rebase` on the parent to keep the child's commit separate in the default branch, where rebase merges are allowed (`gh repo view --json rebaseMergeAllowed`).
- **Deeper stacks fold from the top:** merge the top PR into its parent, repeat downward, then merge the bottom PR into the default branch.

## Merge One by One (Bottom-Up)

```bash
gh pr view {parent} --json headRefOid --jq .headRefOid   # 0. note the parent's last head for the rebase
gh pr edit {child} --base {default}                      # 1. detach the child before its base can be deleted
gh pr merge {parent} --squash --delete-branch            # 2. deleting the parent's branch now closes nothing
git fetch origin
git rebase --onto origin/{default} {parent_head_sha} {child_branch}   # 3. squash/rebase merges only
git push --force-with-lease origin {child_branch}
# 4. the child's CI (and review round, where one fires) on the new base; then merge it like any PR
```

Retarget immediately before the parent's merge, not earlier. Until the parent lands, the child's diff against the default branch includes the parent's changes, so anyone reading the child sees both layers. In a deeper stack, repeat per layer from the bottom.

## If the Child Was Already Closed

Not exercised here. Reopening needs the PR's base branch to exist again, so restore it first:

```bash
gh api repos/{owner}/{repo}/git/refs -f ref=refs/heads/{parent_branch} -f sha={parent_head_sha}   # restore at the parent's last head
gh pr reopen {child}
gh pr edit {child} --base {default}
git push origin --delete {parent_branch}
```

Then rebase the child as in step 3 of Bottom-Up. If the reopen is refused, open a new PR from the child's branch (`gh pr create --base {default} --head {child_branch}`) and link the closed one in its body. Its review threads stay on the closed PR.
