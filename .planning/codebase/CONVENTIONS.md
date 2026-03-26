# Coding Conventions

**Analysis Date:** 2026-03-25

## Naming Patterns

**Files:**
- `kebab-case.ts` for all TypeScript source files (e.g., `device-bootstrap.ts`, `pairing-store.ts`, `archive-helpers.ts`)
- Test files match source with `.test.ts` suffix: `device-bootstrap.test.ts`
- E2E tests use `.e2e.test.ts` suffix: `cli-json-stdout.e2e.test.ts`
- Live (real-key) tests use `.live.test.ts` suffix: `byteplus.live.test.ts`
- Config files use `.config.ts` suffix: `vitest.config.ts`, `agents.config.ts`
- Runtime boundary files use `.runtime.ts` suffix to separate lazy-load surfaces

**Functions:**
- `camelCase` for all function names (e.g., `normalizeE164`, `ensureDir`, `resolveBootstrapPath`)
- `create*` prefix for factory functions (e.g., `createDefaultDeps`, `createTrackedTempDirs`, `createStubPlugin`)
- `resolve*` prefix for functions that derive/compute values (e.g., `resolveConfigDir`, `resolveNativeCommandsEnabled`)
- `build*` prefix for functions that construct data (e.g., `buildPairingReply`)
- `normalize*` prefix for normalization functions (e.g., `normalizePollInput`, `normalizeE164`)
- `assert*` prefix for assertion/guard functions that throw (e.g., `assertWebChannel`)
- `is*`/`has*` prefix for boolean predicates (e.g., `isRecord`, `isVerbose`, `hasConfiguredSecretInput`)

**Variables:**
- `camelCase` for local variables and parameters
- `UPPER_SNAKE_CASE` for module-level constants (e.g., `PAIRING_CODE_LENGTH`, `DEVICE_BOOTSTRAP_TOKEN_TTL_MS`, `OPENCLAW_CLI_ENV_VAR`)

**Types:**
- `PascalCase` for all types and interfaces (e.g., `PollInput`, `TempHomeEnv`, `RegistryState`)
- Type aliases preferred over interface in most cases (but both are used)
- `*Deps` suffix for dependency injection parameter types (e.g., `CliDeps`, `OutboundSendDeps`)
- `*Config` suffix for configuration types (e.g., `CommandsConfig`, `OpenClawConfig`)
- `*Result` / `*Params` / `*Options` suffixes for scoped input/output types

## Code Style

**Formatting:**
- Tool: `oxfmt` (Oxide formatter) — run `pnpm format:fix` to apply, `pnpm format:check` to check
- ESM with `.js` extensions on all local imports (even in `.ts` source, e.g., `import { foo } from "./utils.js"`)
- All Node.js built-ins use `node:` protocol prefix (e.g., `import fs from "node:fs"`, `import path from "node:path"`)

**Linting:**
- Tool: `oxlint --type-aware` with plugin sets `unicorn`, `typescript`, `oxc`
- Config: `.oxlintrc.json`
- Key enforced rules:
  - `typescript/no-explicit-any: error` — never use `any`; fix the root cause
  - `curly: error` — always use braces for control flow
  - `correctness`, `perf`, `suspicious` categories all set to `error`
- Inline disable: `// oxlint-disable-next-line typescript/no-explicit-any` (for rare, documented exceptions)
- Never add `@ts-nocheck`; never disable `no-explicit-any` globally

## Import Organization

**Order:**
1. Node.js built-ins with `node:` prefix (e.g., `import fs from "node:fs"`)
2. Third-party packages (e.g., `import { defineConfig } from "vitest/config"`)
3. Internal project imports using relative paths with `.js` extension (e.g., `import { foo } from "./utils.js"`)

**Type-only imports:**
- Use `import type { ... }` when importing only type definitions (enforced by TypeScript strict mode)
- Example: `import type { ChannelId } from "../channels/plugins/types.js"`

**Path Aliases:**
- `openclaw/plugin-sdk` → `src/plugin-sdk/index.ts`
- `openclaw/plugin-sdk/*` → `src/plugin-sdk/*.ts`
- `openclaw/extension-api` → `src/extensionAPI.ts`
- Extensions must use these public paths; never import from `src/**` directly

## Error Handling

**Patterns:**
- Throw `new Error(message)` with descriptive messages for validation failures
  ```typescript
  if (!question) throw new Error("Poll question is required");
  if (maxSelections > cleaned.length) throw new Error("maxSelections cannot exceed option count");
  ```
- Return `null` (not `undefined`) for "not found" / best-effort lookups (e.g., `safeParseJson`, `jidToE164`)
- Use `try/catch` with `return false` or `return null` for existence checks (e.g., `pathExists`)
- Suppress expected errors silently using empty `catch` blocks with a comment (e.g., `// ignore cleanup errors`)
- Result objects with `{ ok: boolean, reason?: string }` for operations that may fail gracefully (e.g., `verifyDeviceBootstrapToken`)

## Logging

**Framework:** Custom logger via `src/logger.ts` and `src/logging/`

**Patterns:**
- `logInfo(message)` — informational, displayed via palette (info color)
- `logWarn(message)` — warnings, displayed via palette (warn color)
- `logSuccess(message)` — success state, displayed via palette (success color)
- `logError(message)` — errors, goes to stderr
- `logDebug(message)` — always written to file logger; only shown to console when verbose mode is on
- Use `src/terminal/palette.ts` shared CLI palette for ANSI colors — no hardcoded color codes
- Subsystem prefix pattern: messages starting with `discord: ...` or `[discord] ...` are routed through `createSubsystemLogger`

## Comments

**When to Comment:**
- Always add brief comments for tricky or non-obvious logic
- JSDoc `/** ... */` for exported functions and types that have non-obvious purpose or parameters
  ```typescript
  /**
   * Safely parse JSON, returning null on error instead of throwing.
   */
  export function safeParseJson<T>(raw: string): T | null { ... }
  ```
- Inline `//` comments for implementation notes
  - `// best-effort` when an operation silently degrades
  - `// ignore ...` when errors are intentionally suppressed

## Function Design

**Size:** Aim for functions under ~50 LOC; extract helpers for repeated logic. Files targeted under ~700 LOC (soft guideline).

**Parameters:**
- For functions with 3+ related parameters, use a single params object: `function foo(params: { a: string; b: number; c?: boolean })`
- Optional parameters default to `{}` pattern: `function normalizePollInput(input: PollInput, options: NormalizePollOptions = {})`

**Return Values:**
- Return explicit types on exported functions when non-trivial
- Use `Promise<boolean>` or `Promise<{ ok: boolean }>` for async operations with binary outcomes
- Async functions explicitly `async`; callbacks return `Promise<void>` not `void`

## Module Design

**Exports:**
- Named exports preferred; `export default` avoided in production code
- Re-export barrel files used at layer boundaries (e.g., `src/channels/plugins/types.ts` re-exports from sub-files)
- Keep each file focused on one domain; split large files rather than creating "V2" copies

**Dynamic Imports:**
- Never mix `await import("x")` and `import ... from "x"` for the same module in production paths
- Lazy-load boundaries: create a dedicated `*.runtime.ts` file that re-exports from `x`, then dynamically import that boundary
- After any lazy-loading refactor, run `pnpm build` and verify no `[INEFFECTIVE_DYNAMIC_IMPORT]` warnings

**Class Design:**
- Never use prototype mutation (`applyPrototypeMixins`, `Object.defineProperty` on `.prototype`)
- Use explicit inheritance/composition (`A extends B extends C`) or helper composition
- Prefer per-instance stubs in tests over `SomeClass.prototype.method = ...`

---

*Convention analysis: 2026-03-25*
