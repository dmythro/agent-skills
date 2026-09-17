# Access and Roles

Who is allowed to work the board, as opposed to which token scope your automation holds. These are different problems: a scope error blocks **you**, a role gap blocks the **teammate** you set the board up for -- and it surfaces as "the buttons are missing", never as an error message.

## The two gates

Repo role and project role are **independent**. Neither implies the other.

| What the person does | Gate | Minimum role |
|----------------------|------|--------------|
| Open an issue, comment | Repo | Read |
| Apply/dismiss labels | Repo | **Triage** |
| Assign people, close/reopen, apply milestones | Repo | **Triage** |
| Add or remove sub-issues (the epic tree) | Repo | **Triage** |
| Create, edit, delete labels or milestones | Repo | Write |
| Edit anyone's comments, transfer an issue | Repo | Write |
| Delete an issue | Repo | Admin |
| See the board | Project | Read |
| Set Status, Priority, any project field | Project | **Write** |
| Add fields, edit views, toggle workflows | Project | Admin |

Two consequences worth stating plainly:

- Repo **Write** grants nothing on the project. A teammate with push access still cannot drag a card between Status columns.
- Project **Write** grants nothing on the repo. A teammate who can set Priority still cannot apply a label or assign anyone.

A non-coder running the roadmap needs **repo Triage + project Write**. That pair is the default grant; Write on the repo only adds code push and the power to rewrite other people's issue text.

Sub-issues sit at Triage by documented rule: *"People with at least triage permissions for a repository can add sub-issues."* This is the gate that makes an epic tree unbuildable for a Read-only collaborator -- and the one most often mistaken for a bug in the sub-issue UI.

## Diagnose "they cannot set labels / assign / relate"

```bash
# THE authoritative read -- role_name, not permission
gh api repos/{owner}/{repo}/collaborators/<user>/permission --jq '{permission, role_name}'
```

**Trap:** `.permission` is the legacy field and only ever reports `read`/`write`/`admin` -- a triage collaborator reads back as `"permission": "read"`. Always judge by `.role_name`. Checking the wrong field makes a correct triage grant look like it failed.

Then find where the access comes from:

```bash
gh api orgs/<org> --jq '.default_repository_permission'                  # base role for every member
gh api orgs/<org>/teams/<team>/repos --paginate --jq '.[] | {repo: .full_name, role: .role_name}'
gh api "repos/{owner}/{repo}/collaborators?affiliation=direct" --paginate --jq '.[] | {login, role: .role_name}'
```

Effective permission is the **highest** of org base, team grant, and direct grant. A stale direct `read` is harmless; a stale direct `write` silently outranks the team and hides a downgrade.

**Paginate every list read.** REST returns 30 items per page, so a bare `gh api .../collaborators` or `.../members` answers for the first page only and quietly omits the rest -- which turns an access audit or a 2FA preflight into a false all-clear. Add `--paginate`, and emit one record per line (`--jq '.[].login'`) rather than per-page arrays (`--jq '[.[].login]'`), which `--paginate` would hand back as several separate arrays to reassemble.

## Grant repo access (prefer teams)

```bash
gh api --method PUT orgs/<org>/teams/<team>/memberships/<user> -f role=member
gh api --method PUT orgs/<org>/teams/<team>/repos/{owner}/{repo} -f permission=triage
```

`permission` takes `pull | triage | push | maintain | admin`. Grant **per repo** -- the team's own `permission` field (`PATCH /orgs/<org>/teams/<team>`) is only the default applied to repos added later, and it accepts just `pull|push|admin`, so triage is unreachable there.

Direct grant, when a team is overkill:

```bash
gh api --method PUT repos/{owner}/{repo}/collaborators/<user> -f permission=triage
```

Grant before you revoke. Add the team access first, verify `role_name`, then drop redundant direct grants so the team is the single source of truth:

```bash
gh api --method DELETE repos/{owner}/{repo}/collaborators/<user>
gh api repos/{owner}/{repo}/collaborators/<user>/permission --jq .role_name   # still triage, via the team
```

## Grant project access (GraphQL only)

There is **no `gh project` subcommand for access**. Use `updateProjectV2Collaborators`; roles are `NONE | READER | WRITER | ADMIN`, and each entry takes either a `userId` or a `teamId`.

```bash
projectId=$(gh project view <num> --owner <org> --format json --jq .id)
teamId=$(gh api graphql -f query='{ organization(login:"<org>"){ team(slug:"<team>"){ id } } }' \
  --jq '.data.organization.team.id')

gh api graphql -f query='mutation($p:ID!,$t:ID!){
  updateProjectV2Collaborators(input:{projectId:$p, collaborators:[{teamId:$t, role:WRITER}]}){
    collaborators(first:20){ nodes{ __typename ... on Team { slug } ... on User { login } } } } }' \
  -f p=$projectId -f t=$teamId
```

Grant an individual with `{userId: "<id>", role: WRITER}` (`gh api graphql -f query='{ user(login:"<login>"){ id } }'`).

**`role: NONE` removes only that one direct grant.** It does not touch access the person keeps through a team that holds a grant, or through the project's base role. Revoking someone for real is three checks: drop the direct grant, remove them from any granted team, and -- if the base role is `Read` or higher -- lower it. Dropping the direct grant alone and calling it revoked is the mistake this ordering exists to prevent.

**There is no API read for project access.** `ProjectV2` exposes no `collaborators` field, and the mutation's return payload echoes back **only the collaborators you passed in** -- verified: a one-team grant returns `totalCount: 1` on a project that also has three individual collaborators. Do not treat that payload as the access list; it will show you exactly what you just sent and nothing else. The UI's **Manage access** page is the only complete view, so audit access there and never conclude "nobody else has access" from the CLI.

**Base role** -- what every org member gets on the project -- is **UI-only**: Project -> ... -> Settings -> Manage access -> Base role (`No access | Read | Write | Admin`). It is neither readable nor settable through the API.

Effective access is the **maximum** of the base role and every explicit grant, so a grant can only add. The trap: if the base role is already `Write`, every org member can edit the board and your team grant changes nothing -- it looks like team-gated access while being open to the whole org. A project whose access is meant to come from teams needs base role `No access`, set in the UI, with the team grant in place first. Always read the base role off that page before concluding the board is gated.

## Scopes for these operations

| Operation | Scope |
|-----------|-------|
| Read repo roles, issues, sub-issues | `repo` |
| Read org + team membership | `read:org` |
| Team membership grants | `admin:org` (fine-grained: org `Members: write`); caller must be an org owner or a team maintainer |
| Team repo grants | `admin:org` (fine-grained: repo `Administration: write` + org `Members: read` + repo `Metadata: read`); caller must have admin on the repo and be able to see the team |
| Org settings (`PATCH /orgs/<org>`) | `admin:org` **or** `repo` (classic); caller must be an org owner |
| Project reads | `read:project` |
| Project writes, incl. collaborators | `project` |

`admin:org` is not part of a default `gh` login, and it is the team grants that need it -- an org-settings PATCH also goes through on a plain `repo` token, and `write:org` does not cover teams. Adding it needs an interactive device flow the **user** must run themselves (the fine-grained permissions above apply only to a token supplied via `gh auth login --with-token`; `gh auth refresh` deals in classic scopes):

```bash
gh auth refresh -h github.com -s admin:org
```

## Org settings that REST silently ignores

`GET /orgs/<org>` returns these, but `PATCH` **accepts the request, returns 200, and leaves the value unchanged** -- they are absent from the endpoint's request schema and are UI-only:

- `members_can_change_repo_visibility`
- `members_can_delete_repositories`
- `members_can_create_teams`, `members_can_delete_issues`
- `two_factor_requirement_enabled`

UI: Organization -> Settings -> Member privileges (2FA lives under Settings -> Authentication security).

One more is returned by GET and cannot be changed **anywhere** on a Free or Team org: **`members_can_invite_outside_collaborators`**. Restricting outside-collaborator invitations to owners is a **GitHub Enterprise Cloud** feature, so on other plans the toggle is absent from Member privileges and the field is permanently `true`. Do not report it as an unfinished hardening step -- check the org's plan (`gh api orgs/<org> --jq .plan.name`) first. It is also narrower than it sounds: only someone with **admin on a repository** can invite an outside collaborator, so an org whose members top out at triage/write and cannot create repositories has no one able to use it regardless.

PATCH **does** accept `default_repository_permission` (`read|write|admin|none`), `members_can_create_repositories`, and `members_allowed_repository_creation_type`. Because the silent-ignore failure mode exists, **always re-read after a PATCH** instead of trusting the response body:

```bash
gh api --method PATCH orgs/<org> -f default_repository_permission=none -F members_can_create_repositories=false
gh api orgs/<org> --jq '{default_repository_permission, members_can_create_repositories}'   # confirm
```

**Before enabling the 2FA requirement**, know that it lands differently on two populations:

- **Members and billing managers** without 2FA keep their membership but **lose access to org resources** until they enable it. They are not removed and need no re-invite.
- **Outside collaborators** without 2FA **are removed** and lose repository access. An owner can reinstate them within three months once they enable 2FA.

List both before flipping it:

```bash
gh api "orgs/<org>/members?filter=2fa_disabled" --paginate --jq '.[].login'               # locked out until they enable
gh api "orgs/<org>/outside_collaborators?filter=2fa_disabled" --paginate --jq '.[].login' # removed on enable
```

## Why a team, even for three people

A team is one grant point instead of N. It costs nothing and pays off immediately:

- One membership change grants repo role **and** project role -- a new teammate inherits both.
- `@org/team` mentions the whole group on an issue.
- One list to audit when someone leaves; removing them from the team revokes everything at once.
- `default_repository_permission: none` becomes safe: access is explicit and team-derived, so a repo added to the org later is not auto-readable by everyone.

Nested teams are worth it only once distinct roles appear (say a dev team on Write beside a product team on Triage). Until then, one team.
