# Release Notes and Changelogs

How to get from a package name and a version range to the breaking changes, migration steps and adoptable features it contains -- without scraping, and without burning context on patch bumps nobody needs to read about.

## Budget First

Release-note retrieval is the most expensive phase of a validation. Spend it by risk tier from the delta:

| Tier | Packages | Depth |
|------|----------|-------|
| 1 | Majors, `0.x` minors, framework and ORM/CMS packages | Every version crossed, full body |
| 2 | Minors on direct runtime dependencies | Grep the bodies for `BREAKING`, `deprecat`, `removed`, `migrat`, `renamed` |
| 3 | Patches on direct dependencies | Only if a gate fails or an advisory is involved |
| 4 | Transitive-only bumps | None. Confirm no major was crossed and move on |

State in the report which tiers were read. A validation that silently skipped tier 3 reads identical to one that covered everything.

## Step 1: Enumerate the Versions Crossed

Notes for the target version alone are not enough -- a breaking change shipped mid-range stays broken.

```bash
npm view '<pkg>@>16.8.0 <=17.0.2' version
```

Quote the spec so the shell does not consume the comparison operators. Output is one `<pkg>@<version> '<version>'` line per release, oldest first. Pair with publish dates when the timeline matters:

```bash
npm view <pkg> time --json | jq -r 'to_entries | map(select(.key | test("^[0-9]"))) | sort_by(.value) | .[-15:] | .[] | "\(.key)  \(.value)"'
```

Prereleases appear in both listings. Filter them out unless the project deliberately runs one.

## Step 2: Resolve the Repository

```bash
npm view <pkg> repository.url homepage --json
```

`repository.url` arrives as `git+https://github.com/owner/repo.git` -- strip the `git+` prefix and `.git` suffix to get `owner/repo`.

Two traps:

- **Renamed orgs.** React's metadata still says `github.com/react/react`; the repo is `facebook/react`. `gh` follows the redirect, so pass the value through rather than validating it by hand.
- **Monorepos.** Scoped packages usually point at one shared repo (`@payloadcms/db-postgres` -> `payloadcms/payload`). Whether that repo tags per release or per package determines Step 3.

For packages whose metadata carries no repository, `homepage` and the npm page are the fallback, and the docs site is usually where the upgrade guide lives.

## Step 3: Retrieve the Notes

Source order, cheapest first.

### Local changelog (free, no network)

```bash
cat node_modules/<pkg>/CHANGELOG.md 2>/dev/null | head -200
```

Always try first, but expect misses: many popular packages ship no changelog in the tarball at all.

### GitHub releases (primary)

```bash
gh release list --repo <owner>/<repo> --limit 30 --exclude-pre-releases
gh release view <tag> --repo <owner>/<repo> --json tagName,name,publishedAt,body
```

`--exclude-pre-releases` matters on canary-heavy repos -- an unfiltered `gh release list --repo vercel/next.js` returns almost nothing but canaries.

**Tag naming has no standard**, so resolve tags from the repo's actual list rather than constructing them:

| Pattern | Example | Seen in |
|---------|---------|---------|
| `v<version>` | `v17.0.2` | Most repos |
| `<version>` | `17.0.2` | Some monorepos and release-please setups |
| `<pkg>@<version>` | `@tanstack/solid-query@6.0.0-rc.0` | Package-tagged monorepos |
| `<unscoped-name>@<version>` | `payload@3.88.0` | Some monorepos |

A release's display *name* may differ from its tag entirely -- React tags `v19.0.8` and names the release `19.0.8 (July 21st, 2026)`. Match on `tagName`, not `name`.

To pull several versions at once and keep them attributable, fetch the tag list once and resolve each version against it:

```bash
REPO=<owner>/<repo>; PKG=<pkg>
gh release list --repo "$REPO" --limit 100 --json tagName --jq '.[].tagName' > /tmp/tags.txt

for v in 16.9.0 16.10.0 17.0.0; do
  esc=${v//./\\.}
  tag=$(grep -E "^(v?${esc}|.*${PKG}@${esc})$" /tmp/tags.txt | head -1)
  if [ -z "$tag" ]; then echo "### $v -- no release found"; continue; fi
  echo "### $v ($tag)"
  gh release view "$tag" --repo "$REPO" --json body --jq '.body' | head -60
done
```

The alternation covers all four naming patterns in one pass, and anchoring `${PKG}@` disambiguates sibling packages that share a version -- in a package-tagged monorepo a bare version match returns every package released that day.

A version with no matching tag is a finding, not an error to swallow: it usually means the package publishes to npm without cutting GitHub releases, so the changelog or docs fallback applies.

When a repo's tags do not fit any pattern, list and filter instead:

```bash
gh release list --repo <owner>/<repo> --limit 100 --json tagName,publishedAt,isPrerelease \
  --jq '.[] | select(.tagName | test("16\\.(9|1[0-4])"))'
```

### Repository changelog files (fallback)

```bash
gh api repos/<owner>/<repo>/contents/CHANGELOG.md --jq '.download_url' | xargs curl -sL | head -300
```

Common locations when the root has none: `packages/<name>/CHANGELOG.md`, `docs/`, `UPGRADING.md`, `MIGRATION.md`. Next.js keeps `UPGRADING.md` at its repo root; Payload keeps no changelog anywhere and puts everything in releases. Check what exists before fetching:

```bash
gh api repos/<owner>/<repo>/contents --jq '.[].name' | grep -iE 'change|upgrad|migrat'
```

### Commit range (last resort)

When a repo publishes no notes at all:

```bash
gh api repos/<owner>/<repo>/compare/v16.8.0...v17.0.2 --jq '.commits[].commit.message' | head -60
```

Noisy, but conventional-commit repos yield readable `feat:` / `fix:` / `BREAKING CHANGE:` lines.

### Docs site (for guides, not notes)

Framework upgrade guides usually live on the docs site rather than in the repo. Fetch them with WebFetch once the release notes have named the migration; searching the docs blind is slower and less accurate than following the link the notes give.

## Step 4: Extract Only Four Things

Read for these and discard the rest:

1. **Breaking changes** -- removed or renamed APIs, changed defaults, changed config shape, dropped runtime or peer support. Bumped minimum Node versions belong here.
2. **Migration steps** -- codemods, required config edits, schema changes. Hand these to `migrations.md`.
3. **Deprecations** -- with the version they are scheduled for removal, so they can be prioritised rather than merely noted.
4. **Adoptable additions** -- features that could replace something the project currently hand-rolls.

Everything else -- internal refactors, dependency bumps, contributor lists, docs fixes -- is noise. Do not quote it into the report.

Grep pattern for tier-2 scanning:

```bash
gh release view <tag> --repo <owner>/<repo> --json body --jq '.body' \
  | grep -inE 'break|deprecat|remov|renam|migrat|no longer|must now|drop' | head -20
```

## Step 5: Attribute Every Finding

Each extracted item carries the package and the version that introduced it. "Config option renamed" is not actionable; "`serverExternalPackages` replaced `serverComponentsExternalPackages` in 15.0" tells the reader what to grep for. Then check the project actually uses the affected API before reporting it as required work:

```bash
rg -n 'serverComponentsExternalPackages' --glob '!node_modules'
```

A breaking change the project never touches is worth one line under a "not applicable" note, not a task.
