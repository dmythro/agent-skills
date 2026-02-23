# Testing Reference

## bun test

Built-in test runner with Jest-compatible API.

```bash
bun test [flags] [file/dir patterns...]
```

### CLI Flags

| Flag | Description |
|---|---|
| `--filter pattern` | Filter by test name (string or /regex/) |
| `--timeout ms` | Per-test timeout in milliseconds (default: 5000) |
| `--bail [count]` | Stop after N failures (default: 1 if no count) |
| `--rerun-each N` | Run each test N times |
| `--only` | Only run tests marked with `.only` |
| `--todo` | Include `.todo` tests |
| `--watch` | Re-run on file changes |
| `--update-snapshots` | Update snapshot files |
| `--preload module` | Preload script before tests |
| `--coverage` | Enable code coverage |
| `--coverage-reporter type` | Reporter: `text`, `lcov`, `json` |
| `--coverage-dir path` | Coverage output directory |
| `--reporter name` | Test reporter: `default`, `spec`, `tap`, `junit`, `json` |
| `--verbose` | Verbose output |
| `--silent` | Suppress output |
| `--no-color` | Disable color output |
| `--env-file path` | Load env file |
| `--cwd path` | Set working directory |

### Test File Discovery

Patterns searched (in order):
1. `*.test.{ts,tsx,js,jsx,mts,mjs}`
2. `*_test.{ts,tsx,js,jsx,mts,mjs}`
3. `*.spec.{ts,tsx,js,jsx,mts,mjs}`
4. `*_spec.{ts,tsx,js,jsx,mts,mjs}`
5. `__tests__/**/*.{ts,tsx,js,jsx,mts,mjs}`

## Test API

### Defining Tests

```typescript
import { describe, test, it, expect, beforeAll, afterAll, beforeEach, afterEach } from 'bun:test'

// Basic test
test('description', () => {
  expect(1 + 1).toBe(2)
})

// Alias
it('also works', () => {
  expect(true).toBeTruthy()
})

// Grouped tests
describe('group', () => {
  test('nested', () => {})
})

// Async test
test('async', async () => {
  const result = await fetchData()
  expect(result).toBeDefined()
})

// Timeout per test
test('slow test', () => {}, { timeout: 30000 })

// Skip/only/todo
test.skip('skipped', () => {})
test.only('only this runs', () => {})
test.todo('not implemented yet')

// Conditional
test.if(process.platform === 'darwin')('mac only', () => {})
test.skipIf(process.env.CI)('skip in CI', () => {})

// Parameterized
test.each([
  [1, 2, 3],
  [2, 3, 5],
])('add(%i, %i) = %i', (a, b, expected) => {
  expect(a + b).toBe(expected)
})
```

### Lifecycle Hooks

```typescript
beforeAll(() => {
  // Runs once before all tests in this describe block
})

afterAll(() => {
  // Runs once after all tests
})

beforeEach(() => {
  // Runs before each test
})

afterEach(() => {
  // Runs after each test
})
```

All hooks support async functions. `beforeAll`/`afterAll` run once per describe block.

### Matchers (expect)

```typescript
// Equality
expect(x).toBe(y)                    // Strict equality (===)
expect(x).toEqual(y)                 // Deep equality
expect(x).toStrictEqual(y)           // Deep + type equality

// Truthiness
expect(x).toBeTruthy()
expect(x).toBeFalsy()
expect(x).toBeNull()
expect(x).toBeUndefined()
expect(x).toBeDefined()
expect(x).toBeNaN()

// Numbers
expect(x).toBeGreaterThan(y)
expect(x).toBeGreaterThanOrEqual(y)
expect(x).toBeLessThan(y)
expect(x).toBeLessThanOrEqual(y)
expect(x).toBeCloseTo(y, numDigits)

// Strings
expect(str).toMatch(/regex/)
expect(str).toMatch('substring')
expect(str).toContain('substring')
expect(str).toStartWith('prefix')
expect(str).toEndWith('suffix')
expect(str).toHaveLength(n)

// Arrays/Iterables
expect(arr).toContain(item)
expect(arr).toContainEqual(item)     // Deep equality
expect(arr).toHaveLength(n)
expect(arr).toInclude(item)
expect(arr).toIncludeAllMembers([a, b])
expect(arr).toSatisfy(fn)

// Objects
expect(obj).toHaveProperty('key')
expect(obj).toHaveProperty('key', value)
expect(obj).toMatchObject(subset)

// Exceptions
expect(() => fn()).toThrow()
expect(() => fn()).toThrow('message')
expect(() => fn()).toThrow(/regex/)
expect(() => fn()).toThrow(ErrorType)
expect(asyncFn()).rejects.toThrow()

// Snapshots
expect(value).toMatchSnapshot()
expect(value).toMatchInlineSnapshot(`expected`)

// Negation
expect(x).not.toBe(y)

// Asymmetric matchers
expect.anything()
expect.any(Constructor)
expect.stringContaining(str)
expect.stringMatching(regex)
expect.arrayContaining(arr)
expect.objectContaining(obj)
```

## Mocking

```typescript
import { mock, spyOn, jest } from 'bun:test'

// Mock function
const fn = mock(() => 42)
fn()
expect(fn).toHaveBeenCalled()
expect(fn).toHaveBeenCalledTimes(1)
expect(fn).toHaveBeenCalledWith(/* args */)
expect(fn).toHaveReturnedWith(42)

// Spy on existing method
const spy = spyOn(object, 'method')
spy.mockReturnValue(42)
spy.mockResolvedValue(42)
spy.mockImplementation(() => 42)

// Module mocking
mock.module('module-name', () => ({
  default: 42,
  namedExport: 'mocked',
}))

// Timer mocking
jest.useFakeTimers()
jest.advanceTimersByTime(1000)
jest.runAllTimers()
jest.useRealTimers()

// Reset
mock.restore()        // Restore all mocks
fn.mockClear()        // Clear call history
fn.mockReset()        // Clear history + implementation
fn.mockRestore()      // Restore original
```

## Coverage Configuration

In `bunfig.toml`:
```toml
[test]
coverage = true
coverageReporter = ["text", "lcov"]
coverageDir = "./coverage"
coverageThreshold = { line = 80, function = 80, statement = 80 }

# Ignore patterns for coverage
coverageIgnore = ["node_modules", "test", "**/*.test.ts"]
```

Or via CLI:
```bash
bun test --coverage --coverage-reporter lcov --coverage-dir ./coverage
```

## Test Configuration in bunfig.toml

```toml
[test]
# Test runner options
root = "./"                     # Root directory for test discovery
preload = ["./setup.ts"]        # Scripts to run before tests
timeout = 5000                  # Default test timeout (ms)
bail = 0                        # Stop after N failures (0 = no limit)
rerunEach = 1                   # Run each test N times
smol = false                    # Reduce memory usage

# Coverage
coverage = false
coverageReporter = ["text"]
coverageDir = "./coverage"
coverageIgnore = []
```
