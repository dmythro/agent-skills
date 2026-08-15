# Migrations and Codemods

Release notes bury migration requirements in prose, and the two kinds fail in opposite ways. Both must be reported; neither runs without explicit approval.

| Kind | What it changes | How it fails | Reversible |
|------|-----------------|--------------|------------|
| **Code migration** | Source, config, imports, API calls | Loudly -- build or typecheck breaks immediately | Yes, via git |
| **Schema / data migration** | Database structure or stored content | Silently in development, destructively in production | Only with a written-down rollback |

A code migration missed costs an afternoon. A schema migration missed, or applied in the wrong order relative to a deploy, costs data.

## Detection

### From the release notes

Scan every tier-1 and tier-2 body for the vocabulary that signals required work:

```bash
gh release view <tag> --repo <owner>/<repo> --json body --jq '.body' \
  | grep -inE 'migrat|codemod|upgrade guide|breaking|schema|column|table|index|deprecat|must (now|be)|no longer' \
  | head -25
```

Links matter as much as prose: notes that point at an `UPGRADING.md`, a `/docs/*/upgrade*` page or a codemod package are declaring a migration exists even when the word never appears.

### From the project

Which migration systems are in play is a property of the dependency tree, not of the release notes:

```bash
# ORM / CMS packages that own a schema
node -p "Object.keys({...require('./package.json').dependencies, ...require('./package.json').devDependencies}).join('\n')" \
  | grep -iE 'payload|drizzle|prisma|typeorm|mikro-orm|sequelize|knex|mongoose|kysely'

# Existing migration directories -- their presence sets the expected workflow
ls -d migrations src/migrations drizzle prisma/migrations 2>/dev/null
```

A project with a migrations directory has committed to generated, reviewed, version-controlled migrations. An upgrade that changes the schema must produce one; `push`-style commands that sync a database directly bypass that record and must not be proposed there.

## Code Migrations

### Official codemods

Where a package ships one, it is more reliable than hand-editing and far more reliable than an agent's memory of the rename. Confirm the exact invocation from the release notes or upgrade guide for the target version -- these tools change their own CLI between majors.

| Ecosystem | Tool | Typical invocation |
|-----------|------|--------------------|
| Next.js | `@next/codemod` | `bunx @next/codemod@latest upgrade latest` |
| React | `codemod` registry | `bunx codemod@latest react/19/migration-recipe` |
| Tailwind | `@tailwindcss/upgrade` | `bunx @tailwindcss/upgrade` |
| MUI | `@mui/codemod` | `bunx @mui/codemod@latest <version>/preset-safe <path>` |
| ESLint | `@eslint/migrate-config` | `bunx @eslint/migrate-config .eslintrc.json` |
| Generic | `jscodeshift` | `bunx jscodeshift -t <transform> src/` |

Run order that keeps the diff reviewable:

1. Commit or stash everything first -- a codemod's value is that its diff is inspectable in isolation.
2. Run with the tool's dry-run flag where one exists.
3. Run for real, then read the entire diff. Codemods routinely mangle edge cases they do not understand: dynamic imports, re-exports, generated files, commented-out code.
4. Typecheck before anything else. Codemods produce syntactically valid, semantically wrong code.

### Manual migrations

When no codemod exists, work from the extracted breaking-change list rather than from the diff of failures -- fixing what the compiler reports finds the errors, not the silent behavior changes. For each item: grep for the old API, confirm the project uses it, apply the documented replacement, note anything the guide does not cover.

```bash
rg -n '<old-api>' --glob '!node_modules' --glob '!dist'
```

Renamed configuration keys deserve particular care: an unknown key in a config file is frequently ignored rather than rejected, so the old value silently stops applying and nothing breaks until production behaves differently.

## Schema and Data Migrations

### Detecting drift after an upgrade

An upgraded ORM or CMS can change the schema it expects without any project code changing -- new internal columns, changed index strategy, altered field storage. Detect it with the project's own tooling rather than assuming:

| System | Drift / status check | Generate | Apply |
|--------|---------------------|----------|-------|
| Payload | `payload migrate:status` | `payload migrate:create` | `payload migrate` |
| Drizzle | `drizzle-kit check` | `drizzle-kit generate` | `drizzle-kit migrate` |
| Prisma | `prisma migrate status` | `prisma migrate dev --create-only` | `prisma migrate deploy` |
| TypeORM | -- | `typeorm migration:generate` | `typeorm migration:run` |

Run the status or check command first, and report its output verbatim. "The upgrade may require a migration" is not a finding; "`payload migrate:status` reports 2 pending migrations" is.

### Rules

1. **Generate for anything deployed; push only against a local database.** `drizzle-kit push` and `prisma db push` sync a database directly and leave no migration file. That is a reasonable local iteration loop on a disposable dev database. Against a shared, staging or production database it can drop columns without asking, and on a project with a migrations directory it bypasses the audit trail that the directory exists to keep. Deployed environments get a committed migration, applied through the project's migrate command -- no exceptions during an upgrade validation, where the schema change was not the point of the work.
2. **Read the generated SQL before it runs.** Generators infer intent from a schema diff and cannot distinguish a rename from a drop-and-create. A rename inferred as drop-and-create loses the column's data.
3. **State the ordering against deploy.** Additive migrations (new nullable column, new table, new index) apply before the deploy safely. Destructive ones (drop, rename, narrow a type, add a NOT NULL constraint) break the running old code and need the expand/contract split: deploy code tolerating both shapes, migrate, then remove the old shape in a later release.
4. **Write the rollback before proposing the migration.** A `down` migration, a restore point, or an explicit "this is irreversible" -- one of the three, in the report.
5. **Never run against a non-local database without explicit per-environment approval.** Confirm which database the connection string actually points at; a `.env` on a developer machine pointing at staging is common enough to check every time.

## Reporting

Every migration finding carries five fields:

```markdown
- **Package**: payload 3.85 -> 3.88
- **Kind**: schema
- **Required**: yes -- `payload migrate:status` reports 2 pending
- **Command**: `payload migrate:create` then review, then `payload migrate`
- **Ordering**: additive only (two nullable columns) -- safe to apply before deploy; rollback via generated `down`
```

Optional migrations -- a deprecated API that still works, a codemod that only modernises style -- are listed separately from required ones. Mixing them makes the required set look negotiable.
