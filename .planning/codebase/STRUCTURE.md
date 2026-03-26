# Codebase Structure

**Analysis Date:** 2026-03-25

## Directory Layout

```
openclaw/                       # Repo root
├── src/                        # Core TypeScript source
│   ├── entry.ts                # Primary CLI entry point
│   ├── index.ts                # Legacy entry + library consumer exports
│   ├── library.ts              # Public library re-exports
│   ├── runtime.ts              # RuntimeEnv abstraction
│   ├── acp/                    # Agent Client Protocol bridge (external agent integration)
│   ├── agents/                 # AI agent runner (pi-embedded), tools, auth profiles, sandbox
│   ├── auto-reply/             # Inbound-message → agent-run → reply orchestration
│   ├── bindings/               # Low-level Node.js native bindings
│   ├── browser/                # Embedded browser control
│   ├── canvas-host/            # Canvas / a2ui host server
│   ├── channels/               # Channel plugin contracts, routing, allowlists
│   ├── cli/                    # CLI wiring (Commander.js program, option parsers)
│   ├── commands/               # All CLI command implementations
│   ├── compat/                 # Legacy compatibility shims
│   ├── config/                 # Config file IO, types, validation, session store
│   ├── context-engine/         # Context assembly and compaction registry
│   ├── cron/                   # Scheduled/isolated agent tasks
│   ├── daemon/                 # OS daemon management (launchd, systemd, schtasks)
│   ├── docs/                   # Inline documentation helpers
│   ├── gateway/                # Gateway HTTP+WebSocket server + protocol
│   ├── hooks/                  # Plugin hook runner, bundled hooks
│   ├── i18n/                   # Internationalization helpers
│   ├── image-generation/       # Image generation provider abstraction
│   ├── infra/                  # Platform utilities (ports, exec, retry, TLS, outbound delivery…)
│   ├── interactive/            # Interactive prompts helper
│   ├── line/                   # LINE channel message formatting
│   ├── link-understanding/     # URL/link content extraction
│   ├── logging/                # Structured logging, subsystem loggers
│   ├── markdown/               # Markdown rendering utilities
│   ├── media/                  # Media file handling (audio, image, PDF, MIME)
│   ├── media-understanding/    # Media understanding provider abstraction + provider impls
│   ├── memory/                 # Vector memory store, embeddings, search
│   ├── node-host/              # Remote node invocation
│   ├── pairing/                # Device pairing protocol
│   ├── plugin-sdk/             # Public plugin SDK surface (aliased as openclaw/plugin-sdk/*)
│   ├── plugins/                # Plugin registry, loader, runtime context, provider catalog
│   ├── process/                # Process exec, queue, supervisor, child-process bridge
│   ├── providers/              # Provider-specific shared utilities (google, github-copilot…)
│   ├── routing/                # Inbound routing: resolve agent + session key
│   ├── scripts/                # Internal build/maintenance scripts as TypeScript
│   ├── secrets/                # Secret input handling, runtime snapshots
│   ├── security/               # Security path guards, exec obfuscation detection
│   ├── sessions/               # Session ID, label, lifecycle events, transcript events
│   ├── shared/                 # Generic shared utilities (net, text, lazy-runtime)
│   ├── terminal/               # Terminal palette, progress lines, table rendering
│   ├── test-helpers/           # Test setup helpers (used from test files, not shipped)
│   ├── test-utils/             # Test utility modules
│   ├── tts/                    # Text-to-speech provider abstraction + provider impls
│   ├── tui/                    # Terminal UI components and theme
│   ├── types/                  # Shared TypeScript type declarations
│   ├── utils/                  # Miscellaneous utility functions
│   ├── web-search/             # Web search provider abstraction + provider impls
│   ├── whatsapp/               # WhatsApp-specific helpers
│   └── wizard/                 # Interactive setup wizard prompts
├── extensions/                 # Plugin workspace packages (77 extensions)
│   ├── telegram/               # Telegram channel plugin
│   ├── discord/                # Discord channel plugin
│   ├── slack/                  # Slack channel plugin
│   ├── anthropic/              # Anthropic AI provider plugin
│   ├── openai/                 # OpenAI provider plugin
│   ├── google/                 # Google provider plugin
│   ├── matrix/                 # Matrix channel plugin
│   ├── whatsapp/               # WhatsApp channel plugin
│   ├── signal/                 # Signal channel plugin
│   ├── imessage/               # iMessage channel plugin
│   ├── memory-core/            # Core memory plugin
│   ├── memory-lancedb/         # LanceDB memory backend
│   ├── shared/                 # Shared extension utilities
│   └── ...                     # 63 more provider/channel/tool extensions
├── ui/                         # Web control UI (Vite + vanilla TS)
│   └── src/ui/                 # UI component files
├── apps/
│   ├── android/                # Android app (Kotlin, Jetpack Compose)
│   ├── ios/                    # iOS app (Swift, SwiftUI + Observation framework)
│   └── macos/                  # macOS menubar app (Swift, SwiftUI + Observation framework)
├── packages/
│   ├── clawdbot/               # Bot package
│   └── moltbot/                # Bot package
├── test/                       # Top-level integration and architecture tests
├── test-fixtures/              # Shared test fixtures
├── docs/                       # Mintlify documentation source
│   └── zh-CN/                  # Generated Chinese translations (do not edit manually)
├── scripts/                    # Shell and TS build/maintenance scripts
├── Swabble/                    # Swabble integration
├── skills/                     # Built-in agent skills
├── assets/                     # Static assets
├── vendor/                     # Vendored dependencies
├── .agents/skills/             # Agent skill definitions for maintainer workflows
├── .pi/                        # Pi configuration and extensions
├── .planning/codebase/         # GSD codebase analysis documents
├── openclaw.mjs                # Thin wrapper that invokes dist/entry.js
├── package.json                # Root package (also main npm published package)
├── pnpm-workspace.yaml         # pnpm workspace members
├── tsconfig.json               # TypeScript config with openclaw/plugin-sdk/* path aliases
├── tsdown.config.ts            # tsdown bundle configuration
├── vitest.config.ts            # Main vitest config
└── Dockerfile                  # Production Docker image
```

## Directory Purposes

**`src/entry.ts`:**
- Purpose: CLI bootstrap — respawn management, fast-path version/help, lazy CLI load
- Key files: `src/entry.ts`, `src/index.ts`, `src/library.ts`

**`src/cli/`:**
- Purpose: Commander.js program construction, lazy command registration, all CLI-specific utilities
- Contains: `program/` (program builder, command registry, context), `send-runtime/` (per-channel send adapters), `shared/` (shared CLI helpers), many `*-cli.ts` modules for each CLI feature surface
- Key files: `src/cli/run-main.ts`, `src/cli/program/build-program.ts`, `src/cli/program/command-registry.ts`, `src/cli/deps.ts`

**`src/commands/`:**
- Purpose: Business logic for every CLI command
- Contains: `onboard*.ts`, `doctor*.ts`, `status*.ts`, `sessions*.ts`, `agents*.ts`, `channels*.ts`, `configure*.ts`, `auth-choice*.ts`, `sandbox*.ts`, `backup*.ts` and many more
- Key files: `src/commands/onboard.ts`, `src/commands/doctor.ts`, `src/commands/status.ts`

**`src/gateway/`:**
- Purpose: Long-running WebSocket+HTTP gateway server
- Contains: `server.impl.ts` (main server factory), `server-channels.ts` (channel lifecycle manager), `server-methods.ts` (JSON-RPC dispatch), `server-methods/` (sub-handlers), `protocol/` (AJV-validated message schemas), `server/` (HTTP listen, WS connection)
- Key files: `src/gateway/server.impl.ts`, `src/gateway/server-channels.ts`, `src/gateway/server-methods.ts`, `src/gateway/client.ts`

**`src/agents/`:**
- Purpose: Pi (claude-cli) agent runner and all agent-related logic
- Contains: `pi-embedded-runner.ts`, `pi-embedded-subscribe.ts`, `pi-embedded-helpers.ts`, `pi-tools.ts`, sandbox management, auth profiles, subagent registry, context engine, skills
- Key files: `src/agents/pi-embedded-runner.ts`, `src/agents/pi-embedded-subscribe.ts`, `src/agents/pi-tools.ts`, `src/agents/subagent-registry.ts`

**`src/auto-reply/`:**
- Purpose: Translate an inbound `MsgContext` into a `ReplyPayload` via an agent run
- Contains: `reply/get-reply.ts` (main entry), directive parsers, queue, typing controller
- Key files: `src/auto-reply/reply/get-reply.ts`, `src/auto-reply/reply.ts`

**`src/channels/`:**
- Purpose: Channel plugin contract types, allowlists, routing utilities, run-state machine
- Contains: `plugins/types.ts` (channel contracts), `plugins/types.plugin.ts` (ChannelPlugin type), `allowlists/`, `transport/`, `web/`
- Key files: `src/channels/plugins/types.ts`, `src/channels/plugins/types.plugin.ts`, `src/channels/run-state-machine.ts`

**`src/plugins/`:**
- Purpose: Plugin system — registry, loader, runtime context, hook runner
- Contains: `registry.ts`, `loader.ts`, `runtime/index.ts` (PluginRuntime factory), `hooks.ts`, `provider-catalog.ts`
- Key files: `src/plugins/registry.ts`, `src/plugins/runtime/index.ts`, `src/plugins/types.ts`

**`src/plugin-sdk/`:**
- Purpose: Public contract surface exposed to extensions via `openclaw/plugin-sdk/*` path alias
- Contains: 203 re-export files; channel helpers, provider auth, runtime types, webhook utilities
- Key files: `src/plugin-sdk/index.ts`, `src/plugin-sdk/core.ts`, `src/plugin-sdk/channel-plugin-common.ts`, `src/plugin-sdk/provider-auth.ts`

**`src/infra/`:**
- Purpose: Platform utilities — no domain knowledge
- Contains: `outbound/` (delivery pipeline), port utilities, exec approval, heartbeat runner, TLS, file locking, device pairing, update checking
- Key files: `src/infra/outbound/deliver.ts`, `src/infra/ports.ts`, `src/infra/exec-approvals.ts`, `src/infra/heartbeat-runner.ts`

**`src/config/`:**
- Purpose: Config file read/write/validation and session store
- Key files: `src/config/config.ts` (barrel), `src/config/io.ts` (load/write/cache), `src/config/types.ts` (schema), `src/config/sessions.ts`

**`src/routing/`:**
- Purpose: Resolve `agentId` + `sessionKey` from an inbound message context
- Key files: `src/routing/resolve-route.ts`, `src/routing/session-key.ts`

**`src/memory/`:**
- Purpose: Vector memory store with embeddings
- Key files: `src/memory/manager.ts`, `src/memory/embeddings.ts`, `src/memory/search-manager.ts`

**`src/context-engine/`:**
- Purpose: Pluggable registry for context assembly/compaction strategies
- Key files: `src/context-engine/index.ts`, `src/context-engine/registry.ts`

**`src/daemon/`:**
- Purpose: OS daemon/service management (launchd on macOS, systemd on Linux, schtasks on Windows)
- Key files: `src/daemon/launchd.ts`, `src/daemon/systemd.ts`, `src/daemon/schtasks.ts`

**`src/acp/`:**
- Purpose: ACP (Agent Client Protocol) bridge — translates external agent protocol to gateway calls
- Key files: `src/acp/server.ts`, `src/acp/translator.ts`

**`extensions/`:**
- Purpose: All channel and provider plugins as pnpm workspace packages
- Structure: Each extension has `package.json` (with `"openclaw"` manifest), `index.ts` (entry with `definePluginEntry` or `defineChannelPluginEntry`), `src/` (implementation), optional `api.ts` and `runtime-api.ts` barrels
- Key files: `extensions/telegram/index.ts`, `extensions/anthropic/index.ts`, `extensions/discord/index.ts`

**`ui/`:**
- Purpose: Web-based control UI (Vite, vanilla TypeScript)
- Key files: `ui/src/ui/app.ts`, `ui/src/ui/app-gateway.ts`, `ui/src/ui/app-chat.ts`

**`apps/`:**
- Purpose: Native mobile and desktop apps
- Key files: `apps/ios/Sources/`, `apps/android/app/src/main/java/ai/openclaw/`, `apps/macos/Sources/`

**`docs/`:**
- Purpose: Mintlify documentation source (English)
- Key files: `docs/channels/`, `docs/gateway/`, `docs/reference/`

**`test/`:**
- Purpose: Top-level architecture, integration, and e2e tests not colocated with source
- Key files: `test/architecture-smells.test.ts`, `test/extension-plugin-sdk-boundary.test.ts`

**`scripts/`:**
- Purpose: Build, release, packaging, and maintenance scripts
- Key files: `scripts/committer`, `scripts/package-mac-app.sh`, `scripts/bundle-a2ui.sh`

## Key File Locations

**Entry Points:**
- `src/entry.ts`: Primary CLI entry (respawn + bootstrap)
- `src/index.ts`: Legacy entry + library consumer exports
- `openclaw.mjs`: Shell wrapper that invokes `dist/entry.js`

**Configuration:**
- `tsconfig.json`: TypeScript config with `openclaw/plugin-sdk/*` path aliases
- `tsdown.config.ts`: Bundle configuration
- `vitest.config.ts`: Main vitest config
- `pnpm-workspace.yaml`: Workspace members
- `package.json`: Root package (CLI published to npm)

**Core Logic:**
- `src/gateway/server.impl.ts`: Gateway server factory
- `src/auto-reply/reply/get-reply.ts`: Main agent-run orchestrator
- `src/agents/pi-embedded-runner.ts`: Pi agent run loop
- `src/cli/run-main.ts`: CLI entry dispatcher
- `src/cli/program/command-registry.ts`: Lazy command registration
- `src/plugins/registry.ts`: Plugin registration
- `src/routing/resolve-route.ts`: Inbound route resolver

**Testing:**
- `vitest.config.ts`: Main vitest config
- `vitest.unit.config.ts`: Unit test config
- `vitest.e2e.config.ts`: E2E test config
- `vitest.extensions.config.ts`: Extension test config
- `vitest.channels.config.ts`: Channel-specific test config
- `test/global-setup.ts`: Global test setup

## Naming Conventions

**Files:**
- Source files: `kebab-case.ts` (e.g. `run-main.ts`, `server-channels.ts`)
- Test files: `<name>.test.ts` (unit/integration), `<name>.e2e.test.ts` (end-to-end), `<name>.live.test.ts` (live API tests)
- Integration tests: `<name>.integration.test.ts`
- Test helpers: `<name>.test-helpers.ts` or `<name>.test-utils.ts`
- Runtime boundaries: `<name>.runtime.ts` (lazy-loaded runtime modules)
- Plugin SDK subpaths: `src/plugin-sdk/<feature>.ts`
- Extension public APIs: `extensions/<id>/api.ts` (shared), `extensions/<id>/runtime-api.ts` (runtime-only)

**Directories:**
- Source domains: `kebab-case/` (e.g. `auto-reply/`, `context-engine/`)
- Extension packages: `extensions/<channel-or-provider-id>/` (e.g. `extensions/telegram/`)
- Test helpers within directories: `test-helpers/` subdirectory

## Where to Add New Code

**New CLI Command:**
- Primary code: `src/commands/<command-name>.ts`
- CLI wiring: `src/cli/<command-name>-cli.ts` or register in `src/cli/program/command-registry.ts`
- Tests: `src/commands/<command-name>.test.ts` (colocated)

**New Channel Plugin (extension):**
- Create: `extensions/<channel-id>/` as a new pnpm workspace package
- Entry: `extensions/<channel-id>/index.ts` with `defineChannelPluginEntry`
- Source: `extensions/<channel-id>/src/channel.ts` (implements `ChannelPlugin`)
- Public API barrel: `extensions/<channel-id>/api.ts`
- Runtime API barrel: `extensions/<channel-id>/runtime-api.ts`
- Manifest: `extensions/<channel-id>/openclaw.plugin.json` or `package.json` `"openclaw"` field
- Plugin SDK additions (if needed): `src/plugin-sdk/<feature>.ts` with export from `src/plugin-sdk/index.ts`

**New AI Provider Plugin:**
- Create: `extensions/<provider-id>/` as a new pnpm workspace package
- Entry: `extensions/<provider-id>/index.ts` with `definePluginEntry`
- Registration: Call `api.registerProvider({ ... })` inside `register(api)`
- No `src/` needed for simple providers

**New Gateway Method:**
- Add handler to `src/gateway/server-methods/` or `src/gateway/server-methods.ts`
- Add schema to `src/gateway/protocol/schema/`

**New Plugin SDK Subpath:**
- Add: `src/plugin-sdk/<subpath>.ts`
- Export from: `src/plugin-sdk/index.ts` (for root subpath)
- Add subpath in: `package.json` `exports` field

**Utilities:**
- Shared helpers: `src/shared/<feature>.ts`
- Infrastructure: `src/infra/<feature>.ts`
- Channel-specific helper: `src/plugin-sdk/<channel>-*.ts`

**Tests:**
- Unit tests: Colocated `*.test.ts` next to the source file
- Architecture guard tests: `test/*.test.ts`
- Test fixtures: `test-fixtures/` (top-level) or `src/<module>/test-fixtures/`

## Special Directories

**`dist/`:**
- Purpose: Built output
- Generated: Yes
- Committed: No

**`node_modules/`:**
- Purpose: Installed dependencies
- Generated: Yes
- Committed: No

**`src/plugin-sdk/`:**
- Purpose: Public API surface for extensions — consumed via `openclaw/plugin-sdk/*` path alias
- Generated: No (hand-authored re-exports)
- Committed: Yes

**`docs/zh-CN/`:**
- Purpose: Chinese documentation translations
- Generated: Yes (via `scripts/docs-i18n`)
- Committed: Yes (but do not edit manually)

**`.planning/codebase/`:**
- Purpose: GSD codebase analysis documents for AI planning and execution
- Generated: Yes (by `gsd:map-codebase`)
- Committed: Yes

**`skills/`:**
- Purpose: Built-in agent skills (markdown files)
- Generated: No
- Committed: Yes

---

*Structure analysis: 2026-03-25*
