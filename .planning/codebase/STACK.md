# Technology Stack

**Analysis Date:** 2026-03-25

## Languages

**Primary:**
- TypeScript 5.9+ - All core source code (`src/**`, `extensions/**`, `ui/**`)
- Swift - macOS app (`apps/macos/Sources/`), iOS app (`apps/ios/Sources/`), shared SDK (`apps/shared/OpenClawKit/Sources/`)
- Kotlin - Android app (`apps/android/app/`)

**Secondary:**
- JavaScript (ESM) - Build scripts (`scripts/*.mjs`), config files
- Python 3.10+ - Skills automation scripts (`skills/`), linted via Ruff
- Shell / Bash - Packaging, CI, e2e helpers (`scripts/*.sh`)

## Runtime

**Environment:**
- Node.js 22.16.0+ (minimum) — runtime for CLI, gateway, and built output (`dist/`)
- Node.js 24 (Docker images) — pinned in `Dockerfile` via `node:24-bookworm`
- Bun — preferred for TypeScript execution in dev, tests, and scripts; kept in sync with pnpm lockfile

**Package Manager:**
- pnpm 10.23.0 (canonical)
- Lockfile: `pnpm-lock.yaml` — present and committed
- Bun patching supported; patches live in `patches/`

## Frameworks

**Core (Node.js/Gateway):**
- Hono 4.12.8 — HTTP server framework for gateway endpoints (`src/gateway/`)
- Express 5.2.1 — secondary HTTP server for Teams/extension webhooks (`extensions/msteams/`)
- ws 8.19.0 — WebSocket server/client for gateway protocol
- Zod 4.3.6 — schema validation throughout (`src/**`, `extensions/**`)
- `@sinclair/typebox` 0.34.48 — JSON schema / tool schema definitions; pinned exact version

**Control UI (Web frontend):**
- Lit 3.3.2 — web components for control UI (`ui/src/`)
- Vite 8.0.0 — build tool (`ui/vite.config.ts`)

**Testing:**
- Vitest 4.1.0 — test runner (`vitest.config.ts`, `vitest.unit.config.ts`, `vitest.gateway.config.ts`, etc.)
- V8 coverage — via `@vitest/coverage-v8`
- Playwright 1.58.2 — browser and e2e testing (`playwright-core` in core deps; `playwright` in UI devDeps)
- jsdom 29.0.0 — DOM environment for unit tests

**Build/Dev:**
- tsdown 0.21.4 — primary bundler, replaces rollup for core and plugin SDK (`tsdown.config.ts`)
- tsx 4.21.0 — TypeScript execution via `--import tsx` for build scripts
- TypeScript native preview (`@typescript/native-preview` 7.0.0-dev) — used for fast type-check via `pnpm tsgo`
- XcodeGen — iOS project generation (`scripts/ios-configure-signing.sh`)
- Gradle — Android build system (`apps/android/build.gradle.kts`)

**CLI / UX:**
- Commander 14.0.3 — CLI argument parsing (`src/cli/`)
- `@clack/prompts` 1.0+ — interactive CLI prompts (onboarding, wizard)
- osc-progress 0.3.0 — progress bars/spinners (`src/cli/progress.ts`)
- chalk 5.6.2 — terminal colors
- tslog 4.10.2 — structured logging (`src/logger.ts`)

## Key Dependencies

**Critical:**
- `@mariozechner/pi-agent-core` 0.60.0 — embedded agent runtime (pi agent core)
- `@mariozechner/pi-ai` 0.60.0 — AI session primitives used by agent runner
- `@mariozechner/pi-coding-agent` 0.60.0 — coding agent integration
- `@mariozechner/pi-tui` 0.60.0 — TUI interface for agent sessions
- `@agentclientprotocol/sdk` 0.16.1 — ACP protocol SDK (`src/acp/`)
- `@modelcontextprotocol/sdk` 1.27.1 — MCP tool integration (`src/agents/mcp-stdio.ts`)
- `jiti` 2.6.1 — runtime TypeScript import for plugin loading (plugin-sdk alias)

**Infrastructure:**
- `undici` 7.24.4 — HTTP client, fetch polyfill
- `gaxios` 7.1.4 — Google API HTTP client (pinned exact)
- `ws` 8.19.0 — WebSocket
- `hono` 4.12.8 — HTTP framework (pinned exact via override)
- `chokidar` 5.0.0 — file watcher (config reload, canvas watcher)
- `croner` 10.0.1 — cron scheduling (`src/gateway/server-cron.ts`)
- `yaml` 2.8.2 — YAML config parsing
- `json5` 2.2.3 — relaxed JSON config parsing
- `uuid` 11.1.0 — UUID generation
- `dotenv` 17.3.1 — environment variable loading
- `sharp` 0.34.5 — image processing (media pipeline)
- `pdfjs-dist` 5.5.207 — PDF processing for media understanding
- `sqlite-vec` 0.1.7 — vector search over SQLite (memory subsystem, `src/memory/`)
- `tar` 7.5.11 — archive extraction (plugin install, skills)
- `jszip` 3.10.1 — ZIP file handling
- `file-type` 21.3.3 — MIME detection (pinned exact via override)
- `playwright-core` 1.58.2 — headless browser (web channel, browser tools)
- `linkedom` 0.18.12 — lightweight DOM for server-side HTML parsing
- `@mozilla/readability` 0.6.0 — web article extraction
- `markdown-it` 14.1.1 — markdown rendering
- `ipaddr.js` 2.3.0 — IP address parsing (SSRF protection)
- `node-edge-tts` 1.2.10 — Edge TTS integration
- `qrcode-terminal` 0.12.0 — QR code display in terminal (WhatsApp pairing)
- `@homebridge/ciao` 1.3.5 — mDNS/Bonjour service advertisement (`src/infra/bonjour-ciao.ts`)
- `long` 5.3.2 — 64-bit integer handling (protobuf)
- `@lydell/node-pty` 1.2.0-beta.3 — PTY/terminal emulation (agent bash tools)

**AI Provider SDKs (in extensions):**
- `@anthropic-ai/vertex-sdk` (root core) — Anthropic Vertex AI
- `@aws-sdk/client-bedrock` (root core) — AWS Bedrock discovery
- `@line/bot-sdk` 10.6.0 (root core) — LINE messaging
- `openai` (in `extensions/memory-lancedb`, OpenAI provider) — OpenAI API
- `grammy` + `@grammyjs/runner` (telegram extension) — Telegram Bot API
- `@buape/carbon` + `discord-api-types` (discord extension) — Discord Bot
- `@slack/bolt` + `@slack/web-api` (slack extension) — Slack Bot
- `@whiskeysockets/baileys` (whatsapp extension) — WhatsApp Web
- `matrix-js-sdk` + `@matrix-org/matrix-sdk-crypto-nodejs` (matrix extension) — Matrix protocol
- `@microsoft/agents-hosting` (msteams extension) — Microsoft Teams Bot Framework

**Optional/Peer:**
- `@napi-rs/canvas` — optional native canvas rendering
- `node-llama-cpp` 3.16.2 — optional local LLM inference

## Configuration

**Environment:**
- `.env.example` present at repo root (existence confirmed; contents not read)
- Key env vars: `OPENCLAW_STATE_DIR`, `OPENCLAW_WORKSPACE_DIR`, `OPENCLAW_GATEWAY_TOKEN`, `NODE_ENV`, `NODE_OPTIONS`
- Android signing: `OPENCLAW_ANDROID_STORE_FILE`, `OPENCLAW_ANDROID_STORE_PASSWORD`, `OPENCLAW_ANDROID_KEY_ALIAS`, `OPENCLAW_ANDROID_KEY_PASSWORD`
- Provider API keys injected via env vars (model-specific, e.g. `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`)
- Config loaded via `dotenv` at startup (`src/infra/dotenv.ts`)

**Build:**
- `tsdown.config.ts` — bundler config; produces `dist/`
- `tsconfig.json` — TypeScript config; strict mode, `NodeNext` module resolution, ES2023 target
- `tsconfig.plugin-sdk.dts.json` — separate DTS-only build for plugin SDK types
- `pnpm-workspace.yaml` — workspace packages: `.`, `ui`, `packages/*`, `extensions/*`
- `vitest.config.ts` — main test config; multiple variant configs (`vitest.gateway.config.ts`, `vitest.unit.config.ts`, etc.)
- `vite.config.ts` in `ui/` — control UI Vite build
- `knip.config.ts` — dead code detection

## Platform Requirements

**Development:**
- Node.js 22.16.0+, pnpm 10.23.0+ (or Bun)
- macOS: SwiftFormat, SwiftLint, XcodeGen (for `apps/macos/`, `apps/ios/`)
- Android: JDK + Gradle (for `apps/android/`)
- Git hooks via `prek install` (runs same checks as CI)

**Production:**
- Docker: `node:24-bookworm` or `node:24-bookworm-slim` base images (pinned by SHA256 digest in `Dockerfile`)
- Fly.io: `fly.toml` — `shared-cpu-2x`, 2048 MB RAM, `iad` region (primary)
- Render: `render.yaml` — Docker runtime, 1 GB persistent disk at `/data`
- Self-hosted: Node 22+ with `npm i -g openclaw` (Linux/macOS/Windows)
- macOS app: bundled in `dist/OpenClaw.app` via `scripts/package-mac-app.sh`
- iOS: App Store via Xcode + `xcodegen`
- Android: Google Play via Gradle AAB build

---

*Stack analysis: 2026-03-25*
