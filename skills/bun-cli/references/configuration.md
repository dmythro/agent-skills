# Configuration Reference

> Full key-by-key reference: `node_modules/bun-types/docs/runtime/bunfig.mdx`.

**Changed in 1.4 -- `bunfig.toml` is strict TOML.** The rewritten parser throws `SyntaxError`
at startup (rather than a `BuildMessage`) on input the old one accepted:

- unquoted string values -- `linker = isolated` must be `linker = "isolated"`
- missing newlines between key/value pairs
- integers past `Number.MAX_SAFE_INTEGER`

A `bunfig.toml` that loaded fine on 1.3.x can stop Bun from starting on 1.4 with
`TOML Parse error: Strings must be quoted`. Also new in 1.4: a project's `bunfig.toml` takes
precedence over any `.npmrc` setting for the same key (`.npmrc`-only settings such as
`//host/:_authToken` still attach to registries declared in `bunfig.toml`).

## bunfig.toml

Bun's configuration file. Located in the project root or `$HOME/.bunfig.toml` for global config.

### Runtime

```toml
# Run configuration
[run]
bun = true                          # Always use Bun runtime (not Node.js)
shell = "bun"                       # Default shell: "bun" or "system"
silent = false                      # Suppress script name echo
noOrphans = true                    # Exit when the parent dies; SIGKILL descendants on exit
```

Top-level (not under a section):

```toml
env = false                         # Disable automatic .env loading (v1.3.3+)
```

Serving static assets from `Bun.serve` HTML routes (v1.4+):

```toml
[serve.static]
sourcemap = "linked"                # Production no longer serves HTML-route sourcemaps
                                    #   by default; set this to opt back in
```

### Package Installation

```toml
[install]
auto = false                        # Auto-install missing imports
exact = false                       # Pin exact versions by default
peer = true                         # Auto-install peer dependencies
production = false                  # Skip devDependencies
optional = true                     # Install optional dependencies
frozenLockfile = false              # Error if lockfile needs update
dryRun = false                      # Preview without installing
globalDir = "~/.bun/install/global" # Global install location
globalBinDir = "~/.bun/bin"         # Global bin location
concurrentScripts = 8               # Max parallel lifecycle scripts
saveTextLockfile = true             # Use text lockfile (bun.lock)
linker = "isolated"                 # "isolated" or "hoisted"; default comes from the
                                    #   lockfile's configVersion, not the Bun version
globalStore = false                 # Share package files across projects via symlinks
                                    #   (v1.3.14+). Off by default and IGNORED unless
                                    #   linker = "isolated". See pm/global-store.mdx
publicHoistPattern = []             # Globs hoisted to the root node_modules (isolated linker)
hoistPattern = ["*"]                # Globs hoisted into node_modules/.bun/node_modules
hoist = true                        # false: skip that fallback dir entirely, so undeclared
                                    #   requires fail with MODULE_NOT_FOUND (v1.4+)
minimumReleaseAge = 0               # Only install packages published at least N seconds ago

[install.cache]
dir = "~/.bun/install/cache"        # Cache directory
disable = false                     # Disable cache

[install.lockfile]
save = true                         # Save lockfile
print = "default"                   # "default" or "yarn"
```

### Scoped Registries

```toml
[install.scopes]
# GitHub Packages
"@myorg" = { token = "$NPM_TOKEN", url = "https://npm.pkg.github.com/" }

# Private registry
"@company" = { token = "$REGISTRY_TOKEN", url = "https://registry.company.com/" }

# Basic auth
"@private" = { username = "user", password = "$PASS", url = "https://registry.example.com/" }
```

### Default Registry

```toml
[install.registry]
url = "https://registry.npmjs.org/"
token = "$NPM_TOKEN"
```

### Test Configuration

```toml
[test]
root = "./"                         # Test root directory
preload = ["./test/setup.ts"]       # Scripts to run before tests
timeout = 5000                      # Default test timeout (ms)
bail = 0                            # Stop after N failures (0 = unlimited)
rerunEach = 1                       # Run each test N times
smol = false                        # Reduce memory usage

# Coverage
coverage = false
coverageReporter = ["text"]         # "text" and/or "lcov" -- "json" is rejected
coverageDir = "./coverage"
coveragePathIgnorePatterns = ["node_modules", "test"]
# Fractions 0-1, PLURAL keys -- singular keys (line/function) are silently ignored.
# `statements` is accepted but not enforced. A bare number sets lines and functions.
coverageThreshold = { lines = 0.8, functions = 0.8 }
```

### Bundle Configuration

```toml
[bundle]
entryPoints = ["./src/index.ts"]
outdir = "./dist"
target = "browser"                  # "browser", "bun", "node"
format = "esm"                      # "esm", "cjs", "iife"
splitting = false
sourcemap = "none"                  # "external", "inline", "linked", "none"
minify = false
external = []
define = {}
loader = {}
publicPath = ""
naming = { entry = "[dir]/[name].[ext]", chunk = "[name]-[hash].[ext]", asset = "[name]-[hash].[ext]" }
```

### Module Resolution

```toml
[resolve]
conditions = []                     # Custom export conditions
mainFields = ["module", "main"]     # Package.json main field order
extensions = [".tsx", ".ts", ".jsx", ".js", ".mjs", ".cjs", ".json"]
```

### Development Server

```toml
[dev]
port = 3000                         # Default dev server port
hostname = "localhost"              # Default hostname
```

### Debugging

```toml
[debug]
# Log level for Bun's internal operations
logLevel = "info"                   # "debug", "info", "warn", "error"
```

### Macro Configuration

```toml
[macros]
# Remap imports at compile time
react-relay = { graphql = "bun-macro-relay/bun-macro-relay.tsx" }
```

### Trusted Dependencies

In `package.json` (not bunfig.toml):
```json
{
  "trustedDependencies": [
    "esbuild",
    "@prisma/client",
    "sharp"
  ]
}
```

Only these packages can run lifecycle scripts (postinstall, etc.). This is a security feature unique to Bun.
