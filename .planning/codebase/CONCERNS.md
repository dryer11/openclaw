# Codebase Concerns

**Analysis Date:** 2026-03-25

## Tech Debt

**Import boundary violations — core importing bundled extensions directly:**
- Issue: Core `src/` files directly import from `extensions/` bypassing the `openclaw/plugin-sdk/*` contract. This creates tight coupling between core and bundled channel plugins.
- Files:
  - `src/infra/state-migrations.ts:4` — imports `listTelegramAccountIds` from `../../extensions/telegram/api.js`
  - `src/infra/outbound/outbound-session.ts:1-2` — imports `parseDiscordTarget`, `normalizeIMessageHandle`, `parseIMessageTarget` directly
  - `src/config/plugin-auto-enable.ts:1` — imports `hasAnyWhatsAppAuth` from WhatsApp extension
  - `src/config/sessions/explicit-session-key-normalization.ts:1` — imports from Discord extension
  - `src/auto-reply/reply/directive-handling.model.ts:1` — imports from Telegram extension
  - `src/auto-reply/templating.ts:1` — imports Telegram type directly
  - `src/gateway/server-http.ts:11` — imports `handleSlackHttpRequest` from Slack extension
  - `src/cron/isolated-agent/delivery-target.ts:1` — imports from WhatsApp extension
  - `src/agents/pi-embedded-runner/run/attempt.ts:13` — imports Telegram-specific APIs
- Impact: Adding or removing a bundled channel plugin requires changes across core. Violates stated import boundary policy in CLAUDE.md. Prevents clean extraction of channels into installable plugins.
- Fix approach: Expose shared APIs through `openclaw/plugin-sdk/<subpath>` subpaths or move shared utilities to a channel-agnostic layer in `src/channels/`.

**Legacy config migration layer — large and still growing:**
- Issue: Config has three separate migration part files plus multiple supporting files totalling ~2,500 LOC of migration logic that runs on every gateway start.
- Files: `src/config/legacy.migrations.part-1.ts`, `src/config/legacy.migrations.part-2.ts`, `src/config/legacy.migrations.part-3.ts`, `src/config/legacy-migrate.ts`, `src/config/legacy-web-search.ts`, `src/config/legacy.ts`, `src/config/legacy.shared.ts`, `src/config/legacy.rules.ts`
- Impact: Migration code accumulates technical debt over time. No mechanism to retire old migrations once adoption is complete.
- Fix approach: Version migrations with a minimum-version gate so old migrations can be pruned once users are past a known-good version threshold.

**`CLAWDBOT_*` legacy env var aliases still in production code:**
- Issue: Old `CLAWDBOT_` environment variable names are still accepted alongside `OPENCLAW_` equivalents in multiple production paths. The fallback chain reads both.
- Files: `src/config/paths.ts:65`, `src/pairing/setup-code.ts:157,162,223`, `src/security/audit.ts:387,390,774,786`, `src/wizard/setup.ts:309,330`, `extensions/device-pair/index.ts:91,198,204`, `src/infra/bonjour.ts:111`
- Impact: Documentation ambiguity for users, surface area for config mistakes. Dual-name logic must be maintained everywhere env vars are read.
- Fix approach: Define a deprecation horizon; emit one-time migration warnings when CLAWDBOT_ vars are used and remove fallbacks in a future major version.

**Gaxios prototype mutation shim:**
- Issue: `src/infra/gaxios-fetch-compat.ts` patches `Gaxios.prototype._defaultAdapter` at runtime to inject an SSRF-aware fetch. This is a monkey-patch on an internal method (`_defaultAdapter`) of a third-party library.
- Files: `src/infra/gaxios-fetch-compat.ts:286-301`
- Impact: Any gaxios upgrade that renames or removes `_defaultAdapter` will silently break the SSRF protection for Vertex AI. The method name check (`hasGaxiosConstructorShape`) mitigates detection but does not prevent regression.
- Fix approach: Track gaxios releases for `_defaultAdapter` stability. The `installLegacyWindowFetchShim` fallback path mitigates some risk but the core patch is still fragile.

**Stale fallback gateway context snapshot:**
- Issue: `setFallbackGatewayContext` stores a startup-time snapshot of the gateway context. The TODO comment acknowledges it can become stale if runtime config/context changes.
- Files: `src/gateway/server-plugins.ts:49`
- Impact: Plugin code using the fallback gateway context gets outdated config values after runtime config changes (e.g., `openclaw config set`).
- Fix approach: Replace the snapshot with a lazy accessor or a live reference that queries current context at call time.

**ACP error-kind conflation — errors mapped as `end_turn`:**
- Issue: When the agent errors (timeout, rate-limit, etc.), `src/acp/translator.ts` maps the stop reason to `"end_turn"` because `ChatEventSchema` has no structured `errorKind` field. This prevents clients from distinguishing errors from deliberate completions.
- Files: `src/acp/translator.ts:824-829`
- Impact: ACP clients cannot distinguish a rate-limit error from a natural conversation end-turn, leading to potentially incorrect retry or UX behavior.
- Fix approach: Add `errorKind` field to `ChatEventSchema` in `src/gateway/protocol/schema/logs-chat.ts` and thread it through the translator.

**`no-await-in-loop` suppressed 18 times in production security/infra code:**
- Issue: Sequential awaits in loops appear frequently in `src/security/fix.ts` (9 suppressions) and `src/security/audit-extra.async.ts` (5 suppressions). These are individually annotated but represent latent performance bottlenecks on large permission sets.
- Files: `src/security/fix.ts`, `src/security/audit-extra.async.ts`
- Impact: Security audit and fix operations run O(n) sequential async steps. For large permission sets this could be slow but correctness is preserved.
- Fix approach: Audit each loop to determine whether parallelism is safe; convert independent iterations to `Promise.all` batches.

**`ensureExtensionRelayForProfiles` is a dead stub:**
- Issue: This function is a documented no-op kept alive only to avoid touching the call graph in a patch release. The Chrome extension relay path has been removed.
- Files: `src/browser/server-lifecycle.ts:9-16`
- Impact: Dead code that callers must maintain. The comment acknowledges it should be cleaned up.
- Fix approach: Remove the stub and its callers in the next breaking-change release.

## Known Bugs

**Telegram sticker e2e tests skipped — deterministic cache injection not implemented:**
- Symptoms: Two e2e test cases are marked `skip` due to non-deterministic static sticker fetch behavior.
- Files: `extensions/telegram/src/bot.media.stickers-and-fragments.e2e.test.ts:22,73`
- Trigger: Any test run that covers Telegram sticker delivery.
- Workaround: Tests are skipped with reference to issue #50185.

## Security Considerations

**Wildcard CORS origin (`["*"]`) allowed in config schema:**
- Risk: The config schema permits `["*"]` as a valid browser origin for the Control UI WebSocket. The schema help text warns against it but it is not blocked.
- Files: `src/config/schema.help.ts:388`, `src/security/audit.ts:517`
- Current mitigation: `src/security/audit.ts` emits a warning when wildcard is set, advising replacement with explicit origins.
- Recommendations: Consider treating `["*"]` as a configuration error rather than a warning for non-loopback bind addresses.

**SSRF protection depends on gaxios internal API:**
- Risk: SSRF protection for Vertex AI traffic routes through a prototype-level patch on `Gaxios._defaultAdapter`. If gaxios is upgraded and the internal method is renamed or removed, the patch silently becomes a no-op.
- Files: `src/infra/gaxios-fetch-compat.ts`, `src/infra/net/ssrf.ts`, `src/infra/net/fetch-guard.ts`
- Current mitigation: `hasGaxiosConstructorShape` checks for the method before patching; logs `"not-installed"` state on failure.
- Recommendations: Add an explicit integration test that verifies SSRF rejection for Vertex AI requests after gaxios upgrades.

**`CLAWDBOT_GATEWAY_TOKEN` / `CLAWDBOT_GATEWAY_PASSWORD` accepted as fallbacks:**
- Risk: The legacy `CLAWDBOT_*` env var names for gateway token and password are still accepted. Documentation/operational users may set the old names without realizing the current canonical names are `OPENCLAW_*`.
- Files: `src/pairing/setup-code.ts:157-162`, `src/security/audit.ts:387,390`, `src/wizard/setup.ts:309,330`
- Current mitigation: Both names read identically; no privilege escalation risk. Audit code checks both to avoid false-negatives.
- Recommendations: Emit a startup warning when `CLAWDBOT_GATEWAY_TOKEN` or `CLAWDBOT_GATEWAY_PASSWORD` is set without the matching `OPENCLAW_*` name.

**Silent swallowing of `stat()` error after writing allow-from file:**
- Risk: After persisting the allow-from store, a bare `catch {}` swallows any `stat()` error that follows. This means cache invalidation metadata (mtime, size) may be `null` even for a successfully written file.
- Files: `src/pairing/pairing-store.ts:474-476`
- Current mitigation: The cache still records `exists: true`, so subsequent reads will re-read the file rather than using stale data.
- Recommendations: Log the stat error instead of silently swallowing it.

## Performance Bottlenecks

**Sequential async iterations in security audit/fix:**
- Problem: `src/security/fix.ts` and `src/security/audit-extra.async.ts` contain 18 `no-await-in-loop` suppressions, meaning file permission fixes and audit checks run one-at-a-time.
- Files: `src/security/fix.ts`, `src/security/audit-extra.async.ts`
- Cause: Sequential design; each operation awaits before the next starts.
- Improvement path: Identify independent operations and batch with `Promise.all`; preserve sequential execution only where ordering or side-effects require it.

**Memory manager uses `setInterval` for periodic sync:**
- Problem: `src/memory/manager-sync-ops.ts` and `src/memory/qmd-manager.ts` each use a `setInterval` timer for periodic updates. Long-lived intervals can hold references and prevent GC.
- Files: `src/memory/manager-sync-ops.ts:651`, `src/memory/qmd-manager.ts:268`
- Cause: Polling-based sync rather than event-driven invalidation.
- Improvement path: Verify `clearInterval` is called in all teardown paths; consider event-driven invalidation for reduced idle overhead.

## Fragile Areas

**`src/agents/pi-embedded-runner/run/attempt.ts` — 3,212 LOC:**
- Files: `src/agents/pi-embedded-runner/run/attempt.ts`
- Why fragile: Largest single production file by line count. Combines Telegram-specific API calls, model-specific streaming, compaction injection, bootstrap budget, TTS hints, and run-loop logic. No dedicated test file — tested via integration tests only.
- Safe modification: Read the full file before making changes; search for related test suites in `src/agents/pi-embedded-runner/run/attempt.test.ts`.
- Test coverage: Covered by `src/agents/pi-embedded-runner/run/attempt.test.ts` (1,833 LOC integration test), but no unit-level tests for individual helpers.

**`src/agents/subagent-registry.ts` — 1,705 LOC, no direct test file:**
- Files: `src/agents/subagent-registry.ts`
- Why fragile: Large registry that tracks live subagent state, manages sweeper intervals, and gates subagent spawning. No corresponding `subagent-registry.test.ts`.
- Safe modification: Changes here affect all subagent lifecycle events. Run full test suite; search for coverage in `src/agents/` e2e tests.
- Test coverage: Indirect only.

**`src/gateway/server-methods/chat.ts` — 1,601 LOC, no direct test file:**
- Files: `src/gateway/server-methods/chat.ts`
- Why fragile: The main gateway chat message handler; partially covered by `chat.abort-*.test.ts` and `chat.directive-tags.test.ts` but not by a `chat.test.ts` that covers the full handler path.
- Safe modification: Changes to this file affect every chat message processed through the gateway. Add targeted tests for any new branches introduced.
- Test coverage: Partial — abort paths, directive tags, and parent-id injection are tested. Main dispatch logic is not directly unit-tested.

**`src/agents/agent-command.ts` — 1,333 LOC, no direct test file:**
- Files: `src/agents/agent-command.ts`
- Why fragile: Core agent run-loop command handler. No `agent-command.test.ts` found.
- Safe modification: Exercise changes through the agent e2e test suites.
- Test coverage: Indirect via e2e suites.

**`src/gateway/server.impl.ts` — 1,354 LOC, no direct test file:**
- Files: `src/gateway/server.impl.ts`
- Why fragile: Main gateway server implementation. No `server.impl.test.ts` found; tested indirectly through `src/gateway/server.sessions.gateway-server-sessions-*.test.ts`.
- Safe modification: Changes here affect every connected client. Run gateway session tests after changes.
- Test coverage: Indirect only.

**`src/acp/control-plane/manager.core.ts` — 1,421 LOC, no direct test file:**
- Files: `src/acp/control-plane/manager.core.ts`
- Why fragile: ACP control plane core logic. Tested via `src/acp/control-plane/manager.test.ts` (1,661 LOC) which covers the manager integration but not the core module in isolation.
- Safe modification: Run `src/acp/control-plane/manager.test.ts` after changes.
- Test coverage: Integration-level only.

**Matrix legacy crypto migration path:**
- Files: `src/infra/matrix-legacy-crypto.ts` (493 LOC), `src/infra/matrix-legacy-state.ts` (156 LOC)
- Why fragile: One-shot migration code that runs on startup to recover old Matrix crypto state. A failure mid-migration intentionally leaves migration state absent so the next startup can retry, but this means a repeated startup failure loop is possible.
- Safe modification: Do not change migration state file paths or formats without running the Matrix legacy migration tests. The recovery-key persistence failure path at line 440 of `matrix-legacy-crypto.ts` is a known incomplete error path.
- Test coverage: `src/infra/matrix-legacy-crypto.test.ts` (448 LOC) and `src/infra/matrix-legacy-state.test.ts` (244 LOC).

## Scaling Limits

**`Symbol.for()` global singleton registry — multi-instance hazard:**
- Current capacity: At least 19 `Symbol.for()` singletons are registered on `globalThis` to share state across hot-reloaded or multi-instance module loads.
- Limit: In environments where multiple versions of the core are loaded in the same process (e.g., during plugin installs or jiti-based hot reload), the singletons may be shared across incompatible versions.
- Files: `src/plugins/commands.ts`, `src/plugins/runtime.ts`, `src/plugins/hook-runner-global.ts`, `src/plugins/conversation-binding.ts`, `src/agents/session-write-lock.ts`, `src/agents/pi-embedded-runner/runs.ts`, and others.
- Scaling path: Namespace symbols with a version suffix when incompatible state shapes change between releases.

## Dependencies at Risk

**`@buape/carbon` pinned to a pre-release beta tag:**
- Risk: Discord extension depends on `@buape/carbon: 0.0.0-beta-20260317045421` — a dated beta. CLAUDE.md explicitly states "Never update the Carbon dependency," indicating prior breakage with updates. The pinned version is a pre-release build.
- Files: `extensions/discord/package.json:7`
- Impact: Security patches or Discord API changes may require Carbon updates that are currently blocked by policy.
- Migration plan: None currently. Requires explicit decision from owners before any upgrade.

**`@matrix-org/matrix-sdk-crypto-nodejs` — native binary dependency:**
- Risk: Requires compilation during install (`onlyBuiltDependencies`). Platform binary availability may lag Matrix SDK releases.
- Files: `extensions/matrix/package.json`, `pnpm-workspace.yaml`
- Impact: Matrix plugin install failures on unsupported platforms or Node version mismatches.
- Migration plan: Monitor `@matrix-org/matrix-sdk-crypto-nodejs` release cadence; keep version range conservative.

**`workspace:*` in `extensions/matrix` devDependencies:**
- Risk: `extensions/matrix/package.json` uses `workspace:*` for the `openclaw` devDependency. Per CLAUDE.md, `workspace:*` in `dependencies` breaks `npm install --omit=dev`. The current placement in `devDependencies` is correct but must not be moved.
- Files: `extensions/matrix/package.json:15`
- Impact: If accidentally moved to `dependencies`, production plugin installs will break.
- Migration plan: Keep as devDependency; validate in CI that `dependencies` never contains `workspace:*`.

## Missing Critical Features

**`ChatEventSchema` lacks structured `errorKind` field:**
- Problem: ACP clients cannot distinguish backend errors (rate-limit, timeout) from deliberate end-turn responses. All non-abort errors are surfaced as `"end_turn"`.
- Blocks: Proper retry logic and error UX in ACP consumers.
- Files: `src/acp/translator.ts:827-829`, `src/gateway/protocol/schema/logs-chat.ts`

**No retirement mechanism for config legacy migrations:**
- Problem: Legacy config migration code accumulates indefinitely. There is no version gate that allows old migration paths to be pruned after a minimum version floor is established.
- Blocks: Long-term maintainability of the config migration layer.
- Files: `src/config/legacy.migrations.ts`, `src/config/legacy.migrations.part-*.ts`

## Test Coverage Gaps

**`src/agents/subagent-registry.ts` — no unit test file:**
- What's not tested: Direct unit coverage of registry state management, sweeper interval logic, and subagent spawning gates.
- Files: `src/agents/subagent-registry.ts`
- Risk: Bugs in sweeper cleanup or registry lookup could go undetected between e2e runs.
- Priority: High

**`src/gateway/server-methods/chat.ts` — main dispatch path not unit-tested:**
- What's not tested: The main chat message routing logic (non-abort, non-directive-tag paths).
- Files: `src/gateway/server-methods/chat.ts`
- Risk: Regressions in chat dispatch are caught only by full e2e/integration tests.
- Priority: High

**`src/agents/agent-command.ts` — no unit test file:**
- What's not tested: Agent run-loop command handling, stop reason propagation, console error paths.
- Files: `src/agents/agent-command.ts`
- Risk: Changes to run-loop logic lack fast feedback.
- Priority: Medium

**`src/gateway/server.impl.ts` — no unit test file:**
- What's not tested: Direct unit coverage of gateway server connection handling.
- Files: `src/gateway/server.impl.ts`
- Risk: Connection-handling bugs in edge cases (race conditions, concurrent sessions) require full server tests to catch.
- Priority: Medium

**`src/gateway/server-methods/chat-transcript-inject.ts` — no test file:**
- What's not tested: Chat transcript injection logic.
- Files: `src/gateway/server-methods/chat-transcript-inject.ts`
- Risk: Transcript injection edge cases are untested.
- Priority: Low

---

*Concerns audit: 2026-03-25*
