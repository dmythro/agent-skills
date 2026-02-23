# AGENTS.md

Guidelines for AI agents working in this repository.

## Repository Overview

This is a **documentation-only repository** containing agent skills -- structured
Markdown reference documents designed to be installed into AI coding agents
(OpenCode, Claude Code, Copilot, etc.) via the [skills.sh](https://skills.sh)
ecosystem. There is no application code, no build system, no dependencies, and
no tests.

### Structure

```
skills/
  <skill-name>/
    SKILL.md              # Main entry point (has YAML frontmatter)
    references/
      <topic>.md          # Deep-dive reference documents
      allowlist.md        # Tool permission patterns (optional)
```

Current skills: `bun-api`, `bun-cli`, `gh-pr`.

## Build / Lint / Test Commands

There are none. This repository contains only Markdown files. No `package.json`,
no CI/CD, no linter configuration, no test runner.

To validate changes, manually review Markdown rendering. If adding links,
verify they point to valid targets.

## Content Style Guidelines

### SKILL.md Files (Main Entry Points)

Every skill's `SKILL.md` must include YAML frontmatter:

```yaml
---
name: skill-name
description: One or two sentences describing what the skill covers, no trailing period
---
```

- `name`: kebab-case, matches the directory name
- `description`: concise, plain text, no period at end

### Document Structure

1. **H1 heading** immediately after frontmatter -- matches the skill's subject
2. **Opening paragraph** -- bold, concise, establishes context and scope
3. **"When to Use"** section -- bullet list of triggering scenarios
4. **"Critical Rule"** section -- bold, imperative, must-follow constraints
5. **Major topic sections** (H2) covering the skill's API/CLI surface
6. **Best Practices / Key Gotchas** -- numbered list at the end
7. **References** -- blockquote pointers to `references/*.md` files

### Reference Files (`references/*.md`)

- No YAML frontmatter
- H1 title matching the topic
- H2 for each major function, command, or concept
- H3 for sub-topics within those
- Deeper detail than SKILL.md; meant to be loaded on demand

### Allowlist Files (`references/allowlist.md`)

- H1 title "Suggested Allowlist Patterns"
- H2 sections grouping patterns by category (Read-Only, Write, etc.)
- JSON code blocks with `Bash(command:*)` permission patterns
- "All Patterns Combined" section with complete JSON config
- "Not Included" section explaining what requires manual approval and why

## Formatting Conventions

### Headings

- H1 (`#`) for document title only -- one per file
- H2 (`##`) for major sections
- H3 (`###`) for subsections within H2
- Do not skip heading levels (no H1 -> H3)

### Code Blocks

Always use fenced code blocks with a language tag:

- `typescript` for TypeScript/JavaScript code
- `bash` for shell commands
- `toml` for Bun/config TOML files
- `json` for JSON configuration
- `yaml` for YAML frontmatter examples

Code blocks should be practical, copy-paste-ready, and include inline
comments where the behavior is non-obvious.

### Tables

Use Markdown pipe tables for command mappings, flag references, and
comparison matrices. Align columns for readability in source.

### Lists

- **Bullet lists** for use cases, features, and unordered items
- **Numbered lists** for sequential steps, best practices, and ranked items

### Cross-References

Use blockquote format to point to reference files:

```markdown
> **Reference**: See `references/file-io.md` for full API details
```

### Emphasis

- **Bold** for critical rules, key terms, and callouts
- `Inline code` for commands, filenames, function names, and config keys
- Avoid italics -- they render poorly in terminal-based agents

### Horizontal Rules

Use `---` to separate major conceptual sections within a single document
when visual separation helps readability (used sparingly).

## Writing Style

- **Tone**: Direct, imperative, practical. No filler, no hedging.
- **Audience**: AI coding agents, not humans. Be precise and unambiguous.
- **Code-first**: Show the code, then explain if needed. Not the reverse.
- **Concise**: Every sentence should earn its place. Remove anything that
  does not help an agent produce correct code.
- **Opinionated**: State the best practice directly. Do not present
  multiple options without a clear recommendation.
- **No emojis**: Do not use emojis in any skill content.

## Naming Conventions

- **Skill directories**: `kebab-case` (e.g., `bun-api`, `gh-pr`)
- **Reference files**: `kebab-case.md` (e.g., `file-io.md`, `shell-and-process.md`)
- **Frontmatter `name`**: Must match the directory name exactly

## Adding a New Skill

1. Create `skills/<skill-name>/SKILL.md` with proper YAML frontmatter
2. Create `skills/<skill-name>/references/` directory
3. Add reference files for each major sub-topic
4. Optionally add `references/allowlist.md` for tool permission patterns
5. Update `README.md` to include the new skill in the "Available Skills" table
   with its install command: `npx skills add dmythro/agent-skills --skill <name>`

## Common Mistakes to Avoid

1. **Missing language tags** on code blocks -- always specify the language
2. **Overly verbose explanations** -- agents need facts, not tutorials
3. **Inconsistent frontmatter** -- `name` must be kebab-case, `description`
   must not end with a period
4. **Broken cross-references** -- verify `references/*.md` paths exist
5. **Skipping heading levels** -- go H1 -> H2 -> H3, never skip
6. **Including non-Markdown files** -- this repo is Markdown-only
7. **Forgetting README.md updates** -- every new skill needs a table entry

## License

MIT. See `LICENSE` in the repository root.
