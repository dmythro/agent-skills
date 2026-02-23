# Configuration Reference

## bunfig.toml

Bun's configuration file. Located in the project root or `$HOME/.bunfig.toml` for global config.

### Runtime

```toml
# Run configuration
[run]
bun = true                          # Always use Bun runtime (not Node.js)
shell = "bun"                       # Default shell: "bun" or "system"
silent = false                      # Suppress script name echo
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
coverageReporter = ["text"]         # "text", "lcov", "json"
coverageDir = "./coverage"
coverageIgnore = ["node_modules", "test"]
coverageThreshold = { line = 0, function = 0, statement = 0 }
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
