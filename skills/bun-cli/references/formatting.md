# Formatting (bun fmt)

> `bun fmt --help --all` is authoritative for flags. Full guide:
> `node_modules/bun-types/docs/format/index.mdx`, migration guides in
> `docs/guides/format/migrate-from-prettier.mdx` and `migrate-from-oxfmt.mdx`, and the `[fmt]` keys
> in `docs/runtime/bunfig.mdx` (v1.4.4+).

`bun fmt` (v1.4.4+) is a Prettier-compatible formatter built into Bun. It formats JavaScript,
TypeScript, JSX, Flow, CSS, SCSS, Less, HTML, Vue, Svelte, Angular, LWC, MJML, JSON, JSONC, JSON5,
YAML, TOML, Markdown, MDX, GraphQL, and Handlebars, including embedded code (`<style>` in HTML,
code blocks in Markdown, `` css`...` `` templates). It reads an existing Prettier or oxfmt config,
needs no `prettier` package, and replaces `prettier --write`, `oxfmt`, and `lint-staged` for
formatting. Bun 1.4.3 and older have no formatter: `bun fmt` there fails with
`Script not found "fmt"`.

## Commands

```bash
bun fmt                              # format the current directory in place
bun fmt src "test/**/*.ts"           # only these paths or globs
bun fmt --check                      # write nothing; list files that need formatting
bun fmt -l                           # --list-different: the same, without the summary line
bun fmt --list-files                 # print what would be formatted, change nothing
bun fmt --staged                     # only staged files, then stage them again (pre-commit)
bun fmt --changed                    # files that differ from the default branch, untracked included
bun fmt --changed=origin/dev         # ... from another ref
bun fmt --stdin-filepath src/a.ts < src/a.ts   # format stdin as this file, print the result
bun fmt --init                       # add a [fmt] section to bunfig.toml
bun fmt --doctor                     # how Bun reads your config, which packages you can uninstall
bun fmt --debug-check                # write nothing; fail if formatting would change a file's meaning
bun fmt --no-script --check          # the formatter, even when package.json has a "fmt" script
```

| Instead of | Run |
|---|---|
| `prettier --write .` | `bun fmt` |
| `prettier --check .` | `bun fmt --check` |
| `oxfmt` / `oxfmt --check` | `bun fmt` / `bun fmt --check` |
| `lint-staged` running Prettier | `bun fmt --staged` in the pre-commit hook |

`bun format` is not an alias: it runs the project's `format` script, and prints
`Script not found "format"` plus `note: The formatter is bun fmt` when there is none.

## Exit Codes

| Code | Meaning |
|---|---|
| `0` | Every file is formatted (with `--check`), or formatting succeeded |
| `1` | `--check` / `-l`: at least one file needs formatting |
| `2` | Something went wrong: a syntax error, an unknown `[fmt]` key, an unsupported Prettier plugin, a partially staged file under `--staged`, or an unparseable `package.json` that may hold a `fmt` script |

A file with a syntax error is reported and left unchanged; the other files are still formatted,
and the run exits `2`.

## A `fmt` Script Shadows the Formatter

When `package.json` defines a `fmt` script, `bun fmt` runs that script and appends your flags:
`bun fmt --check` runs `<fmt script> --check`. Inside the `fmt` script, `bun fmt` is the
formatter. Anywhere else -- the shell and other scripts alike -- it is the script, so a
`"format": "bun fmt --check"` script runs the `fmt` script. This mirrors `bun check` and the
`check` script.

```bash
bun fmt --no-script            # always the formatter
bun fmt --no-script --check    # what tools and CI steps in projects you do not control should run
```

**`--write` beats `--check`.** `-w` / `--write` (accepted for Prettier compatibility, and the
default) wins in any order: `bun fmt --check --write` formats the files and exits `0`. Never pass
both when you mean to check.

## Configuration

Options go in the `[fmt]` section of `bunfig.toml`. `bun fmt` uses the nearest `bunfig.toml` with a
`[fmt]` section, walking up from the working directory, so a monorepo formats the same from any
directory. Paths and patterns in the section are relative to that file.

```toml
[fmt]
printWidth = 100
semi = false
singleQuote = true
sortImports = true          # group, sort, and merge imports (off by default)
sortTailwindcss = true      # Tailwind class order, using the project's tailwindcss v3 or v4 (off by default)
sortPackageJson = true      # conventional package.json key order (off by default)
jsdoc = true                # format JSDoc comments (off by default)
ignorePatterns = ["vendor/", "**/*.gen.ts"]

[[fmt.overrides]]
files = ["*.test.ts", "scripts/**"]
options = { printWidth = 120 }
```

- **Every Prettier option works**, under the same name and default, and each is also a flag
  (`--print-width 100`, `--no-semi`). The flag wins over `bunfig.toml`.
- **A typo is an error**: an unknown key exits `2` with a suggestion
  (`Unknown key "printWidht" in [fmt]` / `Did you mean "printWidth"?`).
- `.editorconfig` supplies `indent_style`, `indent_size`, `tab_width`, `max_line_length`, and
  `end_of_line`; `bunfig.toml` overrides it.
- Tables tune the sorters: `[fmt.sortImports]` (`groups`, `customGroups`, `internalPattern`,
  `newlinesBetween`), `[fmt.sortTailwindcss]` (`stylesheet` for v4, `config` for v3, `functions`),
  `[fmt.svelte]`. `svelte = false` (or `--no-svelte`) leaves `.svelte` files alone.

### Prettier and oxfmt Configs

`bun fmt` reads `.prettierrc*`, `prettier.config.*`, `"prettier"` in `package.json`, `overrides`,
and `.prettierignore`, and also `.oxfmtrc.json`, `.oxfmtrc.jsonc`, and `oxfmt.config.ts`.
`// prettier-ignore` comments work. A `[fmt]` section still applies next to such a config, where a
key overrides the config file the way its flag would.

- **Built-in plugins.** Import sorting (`@trivago/prettier-plugin-sort-imports`,
  `@ianvs/prettier-plugin-sort-imports`, `prettier-plugin-organize-imports`),
  `prettier-plugin-tailwindcss`, `prettier-plugin-packagejson`, and `prettier-plugin-svelte` are
  handled natively; they may stay in `plugins`, and their options apply.
- **Astro and other plugin-only languages** go to the Prettier installed in the project.
- **An unsupported plugin** (such as `prettier-plugin-jsdoc`) is named in an error, the files it
  applies to are skipped, and the run exits `2`.
- `bun fmt --doctor` lists which files are formatted natively and which packages can be removed.

### What Gets Formatted

The current directory, minus `node_modules`, `.git/info/exclude`, `.prettierignore` entries,
`ignorePatterns`, minified files (`*.min.js`, `*.min.css`), lockfiles, and everything in every
`.gitignore` of the repository -- only the current directory's `.gitignore` when a Prettier config
is present, as Prettier reads it. A path passed explicitly is formatted anyway. JSON files,
`package.json` included, are formatted like any other source; without a Prettier config, TOML is
too. `bun fmt --list-files` shows the exact set before you write anything.

## Git Hooks and CI

```bash
#!/bin/sh
# .git/hooks/pre-commit
bun fmt --no-script --staged
```

`--staged` formats the staged version of each file and stages the result again. A file with both
staged and unstaged changes is not written: Bun names it and exits `2`
(`a.ts has both staged and unstaged changes, so it was not formatted`) -- stage the rest or
`git stash --keep-index` first.

```yaml
- uses: oven-sh/setup-bun@v2
- run: bun fmt --no-script --check
```

## Differences From Prettier

Output matches Prettier 3.9.9 byte for byte on 99.1% of Prettier's own tests (release notes).
Deliberate differences preserve meaning where Prettier changes it: `font-family: Font\ Awesome` in
CSS, `x |b` in a CSS selector, `<!DOCTYPE html>` and `// comment` lines in Handlebars, and trailing
blank lines in a YAML block scalar. Syntax that only Babel parses (stage 2 proposals and below) is
rejected as a syntax error, so a repository that relies on one cannot switch fully.
