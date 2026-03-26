# Testing Patterns

**Analysis Date:** 2026-03-25

## Test Framework

**Runner:**
- Vitest — config: `vitest.config.ts` (root), `vitest.unit.config.ts`, `vitest.e2e.config.ts`, `vitest.live.config.ts`, `vitest.gateway.config.ts`, `vitest.channels.config.ts`, `vitest.extensions.config.ts`
- Pool: `forks` (process forks, not vmForks, for isolation)
- Timeout: 120,000ms default; 180,000ms for Windows hooks

**Assertion Library:**
- Built-in Vitest `expect` (Chai-compatible)

**Run Commands:**
```bash
pnpm test                      # Run all unit tests (parallel via scripts/test-parallel.mjs)
pnpm test -- <path-or-filter>  # Scoped run (e.g., pnpm test -- src/utils.test.ts)
pnpm test:coverage             # Unit tests with V8 coverage report
pnpm test:e2e                  # E2E tests only (*.e2e.test.ts)
pnpm test:gateway              # Gateway tests (src/gateway/**/*.test.ts)
pnpm test:channels             # Channel tests
pnpm test:extensions           # Extension tests
pnpm test:live                 # Live tests (requires real API keys)
```

## Test File Organization

**Location:**
- Unit tests: co-located next to source file (`src/utils.ts` → `src/utils.test.ts`)
- Cross-cutting tests: `test/` directory at repo root
- Test utilities: `src/test-utils/` for shared test helpers
- E2E tests: co-located as `*.e2e.test.ts` in `src/` or `extensions/`, or `test/` root

**Naming:**
- `<source-name>.test.ts` — standard unit/integration test
- `<source-name>.e2e.test.ts` — end-to-end test (spawns real processes, live CLI)
- `<source-name>.live.test.ts` — live test (requires real external API keys)
- `<source-name>.<subsystem>.test.ts` — focused subsystem tests (e.g., `server.auth.compat-baseline.test.ts`)

**Structure:**
```
src/
├── utils.ts
├── utils.test.ts              # co-located unit test
├── infra/
│   ├── device-bootstrap.ts
│   ├── device-bootstrap.test.ts
│   └── archive-helpers.test.ts
├── test-utils/                # shared test helpers (not tests themselves)
│   ├── tracked-temp-dirs.ts
│   ├── frozen-time.ts
│   ├── mock-http-response.ts
│   └── temp-home.ts
test/                          # cross-cutting and e2e tests
├── setup.ts                   # global test setup (runs before all tests)
├── test-env.ts                # environment isolation helpers
├── cli-json-stdout.e2e.test.ts
└── architecture-smells.test.ts
```

## Test Structure

**Suite Organization:**
```typescript
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { myFunction } from "./my-module.js";

describe("myFunction", () => {
  afterEach(() => {
    vi.useRealTimers();
    // cleanup
  });

  it("does the expected thing", () => {
    const result = myFunction(input);
    expect(result).toBe(expected);
  });

  it("throws for invalid input", () => {
    expect(() => myFunction(bad)).toThrow(/message pattern/);
  });
});
```

**Patterns:**
- `afterEach` for cleanup (restoring timers, resetting module state, removing temp files)
- `beforeEach` for per-test setup (env stubs, module reloads via `vi.resetModules()`)
- `afterAll` for suite-level cleanup (e.g., `await tempDirs.cleanup()`)
- All lifecycle hooks at module scope when cleanup is per-suite, inside `describe` when per-group

## Mocking

**Framework:** Vitest `vi` (built-in)

**Patterns:**
```typescript
// Module mock (hoisted, factory-based)
vi.mock("../infra/device-bootstrap.js", () => ({
  issueDeviceBootstrapToken: vi.fn(async () => ({
    token: "bootstrap-123",
    expiresAtMs: 123,
  })),
}));

// Spy on specific method
const spy = vi.spyOn(fs, "readFileSync").mockImplementation((...args) => {
  if (args[0] === mappingPath) return `"5551234"`;
  return original(...args);
});
// Cleanup:
spy.mockRestore();

// Inline vi.fn() for callback stubs
const sendPairingReply = vi.fn(async () => {});
expect(sendPairingReply).not.toHaveBeenCalled();
expect(upsert).toHaveBeenCalledWith({ id: "42", meta: { name: "alice" } });

// Environment variable stubs
vi.stubEnv("OPENCLAW_HOME", "/srv/openclaw-home");
vi.stubEnv("HOME", "/home/other");
vi.unstubAllEnvs(); // restore; or rely on unstubEnvs: true in config (auto-restored between tests)

// Fake timers
vi.useFakeTimers();
vi.setSystemTime(new Date("2026-03-14T12:00:00Z"));
vi.advanceTimersByTime(1000);
vi.useRealTimers(); // or afterEach(() => { vi.useRealTimers(); })
```

**What to Mock:**
- External modules that make network calls or hit real filesystems in unit tests
- File system reads when testing behavior dependent on specific file content
- Timers (`vi.useFakeTimers`) when testing TTL logic or time-dependent behavior
- `process.env` via `vi.stubEnv` (auto-restored by `unstubEnvs: true` config)

**What NOT to Mock:**
- The subject under test itself
- Pure utility functions — test them directly
- File I/O in integration tests that use real temp directories

## Fixtures and Factories

**Test Data:**
```typescript
// Factory pattern for stub objects
function createStubPlugin(params: {
  id: ChannelId;
  label?: string;
  deliveryMode?: ChannelOutboundAdapter["deliveryMode"];
}): ChannelPlugin {
  return {
    id: params.id,
    meta: { id: params.id, label: params.label ?? String(params.id), ... },
    // ...
  };
}

// Helper functions for assertion logic
function expectResolvedSetupOk(resolved: ResolvedSetup, params: { authLabel: string }) {
  expect(resolved.ok).toBe(true);
  if (!resolved.ok) throw new Error("expected setup resolution to succeed");
  expect(resolved.authLabel).toBe(params.authLabel);
}
```

**Location:**
- Shared fixtures: `src/test-utils/` (e.g., `model-auth-mock.ts`, `imessage-test-plugin.ts`)
- Suite-local helpers: defined as functions inside the test file itself
- Global registry/channel stubs: `test/setup.ts` (loaded by all unit tests)

**Temp Directory Handling:**
```typescript
// Recommended: use createTrackedTempDirs() from src/test-utils/tracked-temp-dirs.ts
const tempDirs = createTrackedTempDirs();
const createTempDir = () => tempDirs.make("openclaw-mymodule-test-");

afterEach(async () => {
  await tempDirs.cleanup();
});

// For isolated home environments: withIsolatedTestHome() from test/test-env.ts
// (called automatically in test/setup.ts for all unit tests)
```

## Coverage

**Requirements:**
- Lines: 70%
- Functions: 70%
- Branches: 55%
- Statements: 70%
- Provider: V8
- Only `./src/**/*.ts` counts (not `extensions/`, `apps/`, `ui/`, `test/`)

**View Coverage:**
```bash
pnpm test:coverage     # Runs unit tests with text + lcov reporters
```

**Excluded from coverage:** Entry points (`src/entry.ts`, `src/index.ts`), CLI wiring (`src/cli/**`), commands (`src/commands/**`), large integration surfaces (`src/agents/**`, `src/gateway/**`, `src/channels/**`), interactive TUI/wizard flows.

## Test Types

**Unit Tests (`*.test.ts`):**
- Co-located with source; test pure logic, data transforms, and module behavior
- Use temp dirs, fake timers, and env stubs — never real network/real user state
- Run by default via `pnpm test`

**Integration Tests:**
- Also `*.test.ts`, but may use real filesystem with temp dirs and real child processes
- Distinguished by their use of `createTrackedTempDirs()` or `withTempHome()`

**E2E Tests (`*.e2e.test.ts`):**
- Spawn real CLI processes with `spawnSync` / `execFileSync`
- Test CLI contract (JSON output shape, exit codes, stdout/stderr)
- Run via `pnpm test:e2e` using `vitest.e2e.config.ts`
- Use isolated temp homes (`withTempHome`) to avoid touching real state

**Live Tests (`*.live.test.ts`):**
- Require real API keys via env vars
- Gate with `describe.skip` when env vars are absent:
  ```typescript
  const LIVE = isTruthyEnvValue(process.env.BYTEPLUS_LIVE_TEST) || isTruthyEnvValue(process.env.LIVE);
  const describeLive = LIVE && BYTEPLUS_KEY ? describe : describe.skip;
  ```
- Run via `CLAWDBOT_LIVE_TEST=1 pnpm test:live`

**Architecture / Boundary Tests (`test/*.test.ts`):**
- Verify import boundaries and structural rules (e.g., `test/architecture-smells.test.ts`, `test/extension-plugin-sdk-boundary.test.ts`)
- Do not test business logic; test that the codebase structure is compliant

## Common Patterns

**Async Testing:**
```typescript
// Use resolves/rejects matchers for cleaner async assertions
await expect(verifyDeviceBootstrapToken({ ... })).resolves.toEqual({ ok: true });
await expect(resolvePackedRootDir(emptyDir)).rejects.toThrow(/unexpected archive layout/i);

// Standard async/await with expect
const issued = await issueDeviceBootstrapToken({ baseDir });
expect(issued.token).toMatch(/^[A-Za-z0-9_-]+$/);
```

**Error Testing:**
```typescript
// Sync throws
expect(() => assertWebChannel("bad" as string)).toThrow();
expect(() => normalizePollInput({ question: "Q", options: ["A", "B", "C"] }, { maxOptions: 2 }))
  .toThrow(/at most 2/);

// Async rejects
await expect(resolvePackedRootDir(emptyDir)).rejects.toThrow(/unexpected archive layout/i);
```

**Parameterized Tests:**
```typescript
// Use it.each for table-driven tests
it.each([
  { input: "/tmp/file.zip", expected: "zip" },
  { input: "/tmp/file.TAR.GZ", expected: "tar" },
  { input: "/tmp/file.txt", expected: null },
])("detects archive kind for $input", ({ input, expected }) => {
  expect(resolveArchiveKind(input)).toBe(expected);
});
```

**Platform-Conditional Tests:**
```typescript
const describeUnix = process.platform === "win32" ? describe.skip : describe;
describe.skipIf(isWindows)("restart-stale-pids", () => { ... });
const maybeIt = process.platform === "win32" ? it.skip : it;
```

**Environment Isolation (global):**
- `test/setup.ts` runs `withIsolatedTestHome()` for all unit tests — sets `HOME` to a temp dir, clears sensitive env vars, restores after suite
- `unstubEnvs: true` and `unstubGlobals: true` in `vitest.config.ts` — `vi.stubEnv` changes are automatically reverted between tests

---

*Testing analysis: 2026-03-25*
