# Cloud Coding Agent Tasks

Driving CodeRabbit's cloud Coding Agent from the CLI (`coderabbit code`, beta since v0.9.0) in agent mode: the record stream, what to do with each outcome, and which reruns are safe. Distilled from the [cloud task docs](https://docs.coderabbit.ai/cli/cloud-tasks) and v0.9.1 `--help`; only the read-only `code ls` was exercised here. The cloud task guide does not cover the v0.9.1 additions yet -- `goal`, `archive`, `new --goal`/`--private`/`--access`, `resume --interactive` -- so their notes come from `--help` and the [CLI command reference](https://docs.coderabbit.ai/cli/reference), which is generated from it.

## Before Scripting

- **A person signs in first**: `coderabbit auth login` (browser), then `coderabbit auth org` in an interactive terminal. There is no non-interactive path -- API-key, self-hosted and SSO-workspace logins fail with `unsupported_auth`, a missing organization fails with `organization_required`, and `auth org --agent` only lists organizations.
- **Coding Agent must be enabled for the organization**, and the first task starts its trial if it has not begun. `billing_required` means a person must start a trial or set up billing in the web app.
- **A task starts from the branch on `origin`** that the current branch tracks (or the same-named branch), never from the working tree: push first. A detached HEAD is refused, and `new` warns with `unpushed_commits`, `uncommitted_changes` or `branch_not_on_origin`.
- **`handoff` needs a clean checkout on a pushed branch.** Untracked files count as changes, so keep the summary and plan files outside the repository. Each run creates another task.
- **`--access full`** (v0.9.1, on `new` and `handoff`) requests full repository access, which needs a GitHub authorization in the browser -- a person's step, like `authorize_github`.
- **Pass `--agent` right after the command name**, and before `--plan` on `new`. Every `code` command except `skills import` accepts it; only an explicit `--agent` turns argument errors into JSON.

## The Record Stream

One JSON object per line, each with a `type`. The first record is a `status` with `status: "beta_notice"`.

| `type` | Carries | Notes |
|--------|---------|-------|
| `status` | `status` | Progress. `resume`, `new --resume` and `ask` write `waiting` after 45 s of silence; `push` writes `delivery_waiting` |
| `trace` | `kind` (`agent_message`, `command`, `file_change`, `turn_end`) | `resume` only |
| `action_required` | `action`, `message`, and per action `command`, `url` or `answerTemplate` | A person, or a follow-up command, must act |
| `complete` | `status` | Success -- but `push_in_progress` and `cancel_pending` are not final |
| `error` | `status` (the error code) | Carries no `taskId`: keep it from earlier records |

**The last record decides.** Branch on its `type`, then its `status` or `action`; use the exit code only as a cross-check:

| Exit | Means |
|------|-------|
| `0` | `complete` |
| `1` | `error`, or `action_required` for `authenticate` / `authorize_github` |
| `3` | `action_required`: `answer_question` |
| `4` | `action_required`: `resume` (rerun `code resume --agent <task>`) or `read_answer` (never rerun `ask`) |
| `130`, `143` | Killed by a signal; no last record |

A last record whose `status` or `action` is not covered here: stop and ask a person. `agentMinutes` on the last record of a turn is gross agent minutes, not billed minutes, and `null` when the run did not see the turn end.

## Commit Points and Safe Reruns

**Never rerun a command after it wrote one of these records** -- the rerun creates a second task, message or billed question: `task_submitted` (`new`, carries `taskId` and `url`), `creating_task` (`handoff`), `message_sent`, `steer_sent`, `answer_sent`, `question_sent`. Save them as they arrive and follow the task with `coderabbit code resume --agent <task>` instead.

- `message_dropped` and `steer_not_delivered` mean the send did not land: send again.
- No last record (killed, or exit `130`/`143`) and no commit point seen from `new` or `handoff`? A task may still exist: check `coderabbit code ls --agent --repo .` before rerunning.
- After `-m`, keep `clientOperationId` from `message_sent`: the turn is yours when its `turn.queueItemId` matches. Never resend `-m`.
- **Safe to rerun**: `ls`, `show`, `plan` without `--approve`, `goal` without `--set` or `--clear`, `archive` (a repeat succeeds), and `push` with identical flags -- unless its last record was `delivery_failed`, `delivery_conflicts` or `push_rejected`. Never rerun `cancel` to wait: it stops whatever turn runs then, and queued messages do not run after it.

## Outcomes by Command

### new

| Last record | Next step |
|-------------|-----------|
| `complete` `task_submitted` | Done; keep `taskId` and `url`. With `--resume` the stream continues as for `resume` |
| `error` after a `task_submitted` record | The task exists: use that `taskId`, don't rerun |
| `error` `operation_conflict` | A task may be being created: run `code ls --agent --repo .` and let the user decide |
| `error` `new_failed`, no `task_submitted` | The request may have arrived: check `ls`/`show` before running `new` again |
| `error` `task_rejected` with `branch_not_on_origin` | Push the branch, then retry |

### resume

Watch with no send option, or send at most one of `-m`, `--steer`, `--answer` (all need `--agent`; `-` reads stdin). The run follows the turn until it ends, the agent asks a blocking question, or 9 minutes pass.

`--interactive` (v0.9.1, with `--agent`) keeps one bidirectional NDJSON session open on piped stdin instead: `message`, `steer`, `answer`, `cancel`, `show` and `close` requests go in, correlated responses and live trace events come out, and EOF detaches without stopping the task. A `message`, `steer` or `answer` sent through it starts billed work like `-m`; the request shapes are not documented yet, so prefer one send per `resume` run.

| Last record | Next step |
|-------------|-----------|
| `complete` `turn_completed` | Report `turn.finalReply` |
| `error` `turn_failed`, `turn_canceled`, `turn_interrupted` | Report it; a leftover patch is pushed only with the user's OK |
| `action_required` `answer_question` | With an `answerTemplate`: fill it in and run `code resume --agent <task> --answer '<json>'` -- never the record's `command`, which holds a placeholder. With a `null` template the question is sensitive: give the user the interactive `command`, never answer it yourself |
| `action_required` `resume` | Not final: rerun without a send option |
| `error` `turn_not_running` | Send with `-m` |
| `error` `turn_active`, `pending_queue_exists`, `task_busy`, `task_stopping` | Wait, then retry |
| `error` `resume_failed` after a `*_sent` record | The send probably landed: follow, don't resend |

### ask

Billed from `question_sent` on. A task holds 3 open side chats, an idle one expires after 24 hours, and the CLI cannot close one: continue with `--follow-up <turnId>`.

| Last record | Next step |
|-------------|-----------|
| `complete` `question_answered` | Report `answer`; keep `turnId` |
| `error` after `question_sent` (`answer_failed`, `answer_interrupted`, `side_chat_closed`) | Report it; never ask again |
| `action_required` `read_answer` | No answer within 15 minutes, but it is billed and still coming: read it in the web app, don't rerun |
| `error` `ask_failed` | Billing unknown: treat it as billed and hand the task ID and question to a person |
| `error` `side_chat_limit_reached` | Use `--follow-up` |

### push

| Last record | Next step |
|-------------|-----------|
| `complete` `task_pushed` | Done; `alreadyPushed: true` when nothing new was sent |
| `complete` `push_in_progress` | Not final: rerun with identical flags (it joins the running push) or poll `code show --agent <task>`. Never toggle `--stacked` between runs |
| `action_required` `authorize_github` | Give the `url` to a person, never open it yourself; rerun `command` after they approve |
| `error` `task_busy`, `turn_active`, `pending_queue_exists` | Nothing was pushed: wait, then rerun |
| `error` `delivery_failed`, `delivery_conflicts`, `push_rejected` | Report it; fix the cause (conflicts: on the task page) before pushing again |

### cancel, plan, autofix, goal, archive

- **`cancel`**: `complete` `cancel_pending` is not final -- poll `code show --agent <task>` until `task.turnStatus` is not `in_progress` and `task.cancellationPending` is false. Queued messages do not run afterwards: CodeRabbit pauses them where queue pause has rolled out (resume them from the task page in the web app), and automated tasks such as pull request monitoring drop them.
- **`plan`**: `plan_stale` -- show the plan again and get approval again; `plan_not_found` -- no finished plan yet, and a code-mode task never has one.
- **`autofix`** (v0.9.1; `autopilot` on v0.9.0, an alias since): the records keep the `autopilot` field and `autopilot_*` status codes. `autopilot_paused` (`on` only) -- run `code autofix resume --agent` on the same target; `billing_required` -- a person acts.
- **`goal`** (v0.9.1): reads the goal by default. `--set <objective>` (1--4,000 characters, `-` for stdin) keeps a paused or limited state but restarts a completed goal; `--clear` removes it. Get the user's OK for either.
- **`archive`** (v0.9.1): hides the task from `ls` (`show` still reads it), makes it read-only and pauses its schedules; it can be unarchived later. Running, starting or stopping turns must finish first, and it neither cancels turns nor deletes the task.

### Any command

`action_required` `authenticate` -- a person runs the login `command`. `unsupported_auth`, `organization_required` -- a person signs in or picks an organization. `invalid_arguments` -- fix the call. Workspace tasks are read-only (`task_read_only`), tasks on the previous runtime fail `resume` and `cancel`, and automation tasks fail with `unsupported_task`: continue those in the web app.

## Limits

| Command | Limit |
|---------|-------|
| `resume --agent` | Follows for up to 9 minutes, then exits `4` |
| `ask` | Up to 3 minutes for the task's environment, then 15 for the answer |
| `push` | Waits up to 15 minutes for delivery; the push continues afterwards |
| `cancel` | Up to 60 seconds for the turn to stop |
| `show` | Up to 20 seconds for pull request health, then warns and still succeeds |
| Prompts, messages, steers, questions | 65,536 characters; stdin 262,144 bytes |

`cancel`, `handoff`, `plan`, `autofix`, `ls` and `show` write no keep-alive records -- size CI idle timeouts from this table.
