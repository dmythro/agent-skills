# Package Management Reference

## bun install

Install all dependencies from package.json.

```bash
bun install [flags]
```

| Flag | Description |
|---|---|
| `--frozen-lockfile` | Error if lockfile is out of date (CI) |
| `--no-save` | Don't update package.json or lockfile |
| `--production` | Skip devDependencies |
| `--dry-run` | Show what would be installed |
| `--force` | Reinstall all packages |
| `--verbose` | Verbose logging |
| `--silent` | No logging |
| `--no-progress` | Disable progress bar |
| `--ignore-scripts` | Skip lifecycle scripts |
| `--trust` | Run lifecycle scripts for new dependencies |
| `--concurrent-scripts N` | Max parallel lifecycle scripts |
| `--backend` | Install backend: `hardlink` (default), `symlink`, `copyfile`, `clonefile` |
| `--cwd path` | Set working directory |
| `-g, --global` | Install globally |
| `--config path` | Load specific bunfig.toml |
| `--no-verify` | Skip checksum verification |
| `--no-cache` | Disable package cache |
| `--cache-dir path` | Custom cache directory |

### Lockfile Behavior

- `bun.lock` (text, JSONC format) — default since Bun v1.2
- `bun.lockb` (binary) — legacy format
- Convert: `bun install` regenerates lockfile; delete old format first
- `--save-text-lockfile` — force text format
- `--frozen-lockfile` — CI mode, errors if lockfile needs changes

## bun add

Add one or more packages.

```bash
bun add [flags] pkg [@version] [pkg2...]
```

| Flag | Description |
|---|---|
| `-d, --dev` | Add to devDependencies |
| `--optional` | Add to optionalDependencies |
| `-g, --global` | Install globally |
| `--exact` | Pin exact version (no ^ or ~) |
| `--no-save` | Don't update package.json |
| `--trust` | Run lifecycle scripts for this package |
| `--verbose` | Verbose logging |
| `--cwd path` | Set working directory |
| `--filter pattern` | Add in specific workspace(s) |

### Version Specifiers

```bash
bun add pkg                  # Latest
bun add pkg@1.2.3            # Exact
bun add pkg@^1.2.0           # Compatible
bun add pkg@~1.2.0           # Patch-level
bun add pkg@latest           # Latest tag
bun add pkg@next             # Next tag
bun add user/repo            # GitHub
bun add https://example.com/pkg.tgz  # Tarball URL
bun add ./local-pkg          # Local path
bun add file:./local-pkg     # Local path (explicit)
```

## bun remove

Remove one or more packages.

```bash
bun remove pkg [pkg2...] [flags]
```

| Flag | Description |
|---|---|
| `-g, --global` | Remove globally |
| `--cwd path` | Set working directory |
| `--filter pattern` | Remove from specific workspace(s) |

## bun update

Update packages to latest compatible versions.

```bash
bun update [pkg] [flags]
```

| Flag | Description |
|---|---|
| `--latest` | Update to latest, ignoring semver ranges |
| `--filter pattern` | Update in specific workspace(s) |
| `--cwd path` | Set working directory |

## bun outdated

Show packages with newer versions available.

```bash
bun outdated [pkg] [flags]
```

| Flag | Description |
|---|---|
| `--filter pattern` | Check specific workspace(s) |
| `--cwd path` | Set working directory |

Output columns: Package, Current, Update (semver-compatible), Latest, Workspace.

## bun audit

Check for known vulnerabilities.

```bash
bun audit [flags]
```

| Flag | Description |
|---|---|
| `--level` | Minimum severity: `info`, `low`, `moderate`, `high`, `critical` |
| `--json` | JSON output |
| `--cwd path` | Set working directory |

## bun link

Create/use local package links for development.

```bash
bun link               # Register current package as linkable
bun link pkg-name      # Link a registered package into current project
bun unlink             # Unregister current package
```

## bun patch

Patch installed packages (modifies in node_modules, persists via patchedDependencies).

```bash
bun patch pkg[@version]         # Start patching (extracts to temp dir)
bun patch --commit patch-dir    # Apply and record patch
```

Patches stored in `patches/` directory, recorded in package.json `patchedDependencies`.

## bun pm

Package manager utilities.

```bash
bun pm ls [--all]       # List installed packages
bun pm hash             # Print lockfile hash (for CI caching)
bun pm hash-string      # Print lockfile hash as string
bun pm hash-print       # Print lockfile hash to stdout
bun pm cache            # Show global cache directory
bun pm cache rm         # Clear global cache
bun pm pack [--dry-run] [--destination dir]  # Create tarball
bun pm default-trusted  # Show default trusted dependencies
bun pm untrusted        # Show packages with blocked scripts
bun pm trust [pkg...]   # Trust packages (allow lifecycle scripts)
bun pm migrate          # Migrate from npm/yarn/pnpm lockfile
```

## bun info

Show package metadata from the registry.

```bash
bun info pkg                    # Show package info
bun info pkg versions           # List all published versions
bun info pkg version            # Show latest version
bun info pkg description        # Show description
bun info pkg dependencies       # Show dependencies
```

## bun publish

Publish package to npm registry.

```bash
bun publish [flags]
```

| Flag | Description |
|---|---|
| `--dry-run` | Preview without publishing |
| `--tag name` | Publish with dist-tag (default: latest) |
| `--access public/restricted` | Set package access level |
| `--otp code` | One-time password for 2FA |
| `--auth-type legacy/web` | Auth type |
| `--token token` | Auth token |
| `--registry url` | Custom registry URL |

## Workspace Commands

```bash
bun --filter 'pkg-name' install     # Install in specific workspace
bun --filter 'pkg-name' add dep     # Add dep to specific workspace
bun --filter '*' run build          # Run in all workspaces
bun --filter './apps/*' run dev     # Run with glob pattern
bun --filter '!pkg-name' test       # Exclude specific workspace
```

### Workspace Protocol in package.json

```json
{
  "dependencies": {
    "@scope/pkg": "workspace:*",
    "@scope/pkg2": "workspace:^1.0.0"
  }
}
```
