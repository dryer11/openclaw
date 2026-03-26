# Architecture

**Analysis Date:** 2026-03-25

## Pattern Overview

**Overall:** Gateway-centric, plugin-extensible AI messaging platform

**Key Characteristics:**
- Long-running gateway process (`src/gateway/`) handles all real-time channel connections, WebSocket clients, and agent runs
- CLI (`src/cli/`) is the user-facing surface; connects to the gateway via WebSocket or runs operations directly
- Everything external (messaging channels, AI providers, tools) is a plugin registered via `openclaw/plugin-sdk`
- ACP (Agent Client Protocol) bridge at `src/acp/` allows external AI agents/tools (e.g. Claude) to connect to the gateway
- Extensions in `extensions/*` are pnpm workspace packages that register themselves using `defineChannelPluginEntry` or `definePluginEntry`

## Layers

**Entry Points:**
- Purpose: Bootstrap and start the CLI or hand off to gateway
- Location: `src/entry.ts` (primary), `src/index.ts` (legacy + library consumer surface)
- Contains: Process respawn logic, fast-path version/help, lazy imports of CLI
- Depends on: `src/cli/run-main.ts`, `src/infra/`
- Used by: End users, npm package consumers

**CLI Layer:**
- Purpose: Parses argv, dispatches to commands, wraps gateway operations
- Location: `src/cli/`
- Contains: Commander.js program construction, lazy command registration, option parsing
- Depends on: `src/commands/`, `src/gateway/`, `src/infra/`, `src/plugins/`
- Used by: `src/entry.ts`

**Commands Layer:**
- Purpose: Business logic for each CLI subcommand
- Location: `src/commands/`
- Contains: `onboard`, `setup`, `configure`, `doctor`, `status`, `sessions`, `agents`, `channels`, `message`, and 40+ other commands
- Depends on: `src/config/`, `src/channels/`, `src/gateway/`, `src/agents/`, `src/plugins/`
- Used by: CLI layer

**Gateway Layer:**
- Purpose: Long-running server; manages channels, agents, WebSocket clients, hooks, and cron
- Location: `src/gateway/`
- Contains: HTTP+WebSocket server (`server.impl.ts`), channel manager (`server-channels.ts`), method handlers (`server-methods.ts`, `server-methods/`), protocol schema (`protocol/`)
- Depends on: `src/channels/`, `src/agents/`, `src/plugins/`, `src/config/`, `src/infra/`
- Used by: CLI via `src/gateway/client.ts`; ACP bridge `src/acp/`

**Channels Layer:**
- Purpose: Abstractions for all messaging platforms
- Location: `src/channels/`
- Contains: Plugin contract types (`plugins/types.ts`, `plugins/types.plugin.ts`), allowlists, routing, run-state machine, session handling
- Depends on: `src/routing/`, `src/config/`, `src/infra/`
- Used by: Gateway layer; channel extension plugins

**Agents / Pi Layer:**
- Purpose: AI agent execution — wraps pi-agent-core (Claude) with OpenClaw-specific tooling
- Location: `src/agents/`
- Contains: `pi-embedded-runner.ts` (run loop), `pi-embedded-subscribe.ts` (stream processing), tool definitions, sandbox management, auth profiles, subagent registry, context engine
- Depends on: `src/plugins/`, `src/context-engine/`, `src/memory/`, `src/hooks/`, `src/infra/`
- Used by: Gateway layer (requests), `src/auto-reply/reply/` (inbound message processing)

**Auto-Reply Layer:**
- Purpose: Orchestrates an inbound message into an agent run and reply
- Location: `src/auto-reply/`
- Contains: `reply/get-reply.ts` (main entry), directive parsing, queue, typing controller, sandbox media staging
- Depends on: `src/agents/`, `src/channels/`, `src/config/`, `src/media/`
- Used by: Channel gateway adapters, CLI `send` command

**Plugin SDK (`openclaw/plugin-sdk`):**
- Purpose: Public API surface for extensions; re-exports from `src/plugin-sdk/*.ts`
- Location: `src/plugin-sdk/` (203 files), aliased as `openclaw/plugin-sdk/*` via tsconfig paths
- Contains: Channel contracts, provider auth helpers, runtime types, tool helpers, webhook utilities
- Depends on: `src/channels/plugins/`, `src/plugins/types.ts`, `src/config/`
- Used by: All `extensions/*` packages

**Plugins Layer:**
- Purpose: Registry, loader, runtime, and hooks for all extension plugins
- Location: `src/plugins/`
- Contains: `registry.ts` (registration), `runtime/` (runtime context passed to plugin code), `loader.ts`, `hooks.ts`, channel plugin IDs, provider catalog
- Depends on: `src/channels/plugins/`, `src/hooks/`, `src/context-engine/`
- Used by: Gateway layer, CLI layer

**Infrastructure Layer (`src/infra/`):**
- Purpose: Platform utilities with no domain knowledge
- Location: `src/infra/`
- Contains: Outbound delivery (`outbound/`), port management, file locking, exec approval, heartbeat runner, TLS helpers, process management, restart logic, update checking, pairing utilities
- Depends on: Node.js built-ins only (plus a few stable third-party utilities)
- Used by: All layers

**Config Layer:**
- Purpose: Read/write/validate `openclaw.json` config file
- Location: `src/config/`
- Contains: `config.ts` (barrel), `io.ts` (load/write), `types.ts` (schema), `validation.ts`, session store
- Depends on: `src/infra/`
- Used by: All layers

**Routing Layer:**
- Purpose: Resolve which agent+session key to use for an inbound message
- Location: `src/routing/`
- Contains: `resolve-route.ts` (agent binding resolution), `session-key.ts` (key building), `account-lookup.ts`
- Depends on: `src/config/`
- Used by: Auto-reply layer, channel gateway adapters

## Data Flow

**Inbound Message → Reply:**

1. Channel extension gateway adapter receives inbound event (e.g. Telegram update)
2. Adapter calls into `src/channels/plugins/channel-plugin-common.ts` / `channel-reply-pipeline.ts`
3. `src/routing/resolve-route.ts` determines `agentId` + `sessionKey`
4. `src/auto-reply/reply/get-reply.ts` (`getReplyFromConfig`) orchestrates the agent run
5. `src/agents/pi-embedded-runner.ts` calls pi-agent-core with model, tools, and system prompt
6. `src/agents/pi-embedded-subscribe.ts` streams and chunks the response
7. Reply payload is delivered via `src/infra/outbound/deliver.ts` → channel outbound adapter

**CLI Command → Gateway:**

1. `src/entry.ts` bootstraps, `src/cli/run-main.ts` dispatches argv
2. Lazy command module is loaded and registered via `src/cli/program/command-registry.ts`
3. Command creates a `GatewayClient` (`src/gateway/client.ts`) and sends a JSON-RPC method call
4. Gateway processes the request in `src/gateway/server-methods.ts` and responds via WebSocket

**ACP Agent Integration:**

1. External agent (e.g. Claude) connects via ACP protocol (JSON-RPC over stdin/stdout)
2. `src/acp/server.ts` / `AcpGatewayAgent` translates ACP protocol to gateway WebSocket calls
3. Gateway processes the session and streams back events

**State Management:**
- Config: JSON5 file, cached in memory with `getRuntimeConfigSnapshot()` (`src/config/io.ts`)
- Sessions: JSONL transcript files at `~/.openclaw/agents/<agentId>/sessions/`
- Auth profiles: `~/.openclaw/credentials/` (per-profile JSON files)
- Plugin state: Extension-specific stores under `~/.openclaw/plugins/<pluginId>/`

## Key Abstractions

**ChannelPlugin:**
- Purpose: The interface every messaging channel must implement
- Examples: `extensions/telegram/src/channel.ts`, `extensions/discord/src/channel.ts`
- Pattern: `ChannelPlugin<ResolvedAccount>` with optional adapter fields (`gateway`, `outbound`, `lifecycle`, `auth`, `status`, etc.)

**defineChannelPluginEntry / definePluginEntry:**
- Purpose: Factory function called from extension `index.ts` to register the plugin
- Examples: `extensions/telegram/index.ts`, `extensions/anthropic/index.ts`
- Pattern: Returns a plugin entry with `id`, `name`, and a `register(api)` method

**PluginRuntime:**
- Purpose: Context object injected into plugin code at runtime by `src/plugins/runtime/index.ts`
- Examples: Used throughout `extensions/*/src/`
- Pattern: `createPluginRuntime()` constructs a `PluginRuntime` with `agent`, `channel`, `config`, `media`, `subagent`, `tools` sub-namespaces

**OpenClawPluginApi:**
- Purpose: Registration API passed to `register(api)` in plugin entries
- Examples: Used in `extensions/anthropic/index.ts`, `extensions/telegram/index.ts`
- Pattern: Provides `api.registerProvider()`, `api.registerChannel()`, `api.registerHook()`, etc.

**GatewayClient:**
- Purpose: CLI-side client that speaks the gateway WebSocket protocol
- Examples: `src/gateway/client.ts`
- Pattern: Created by commands; sends JSON-RPC method calls, receives event streams

**createDefaultDeps:**
- Purpose: Factory for lazy-loaded per-channel send functions used by CLI commands
- Examples: `src/cli/deps.ts`
- Pattern: Returns `CliDeps` map keyed by channel ID, each a proxy that dynamic-imports the channel's send runtime on first use

## Entry Points

**CLI Entry (`src/entry.ts`):**
- Location: `src/entry.ts`
- Triggers: Direct `node dist/entry.js` or `openclaw` binary (`openclaw.mjs` wrapper)
- Responsibilities: Respawn with `--disable-warning=ExperimentalWarning`, parse `--profile`, fast-path version/help, invoke `runCli()`

**Library Entry (`src/index.ts`):**
- Location: `src/index.ts`
- Triggers: `import { createDefaultDeps } from 'openclaw'` from external consumers
- Responsibilities: Lazy-load and re-export library functions; guards against double-execution when imported alongside `entry.ts`

**Gateway Server (`src/gateway/server.impl.ts`):**
- Location: `src/gateway/server.impl.ts`
- Triggers: `openclaw gateway run` CLI command
- Responsibilities: Start HTTP+WebSocket server, load plugins, start channel managers, initialize agent runtimes, register JSON-RPC method handlers

**ACP Bridge (`src/acp/server.ts`):**
- Location: `src/acp/server.ts`
- Triggers: `openclaw acp` CLI command
- Responsibilities: Connect to gateway, translate ACP protocol (stdin/stdout JSON-RPC) into gateway WebSocket calls

## Error Handling

**Strategy:** Errors bubble to the nearest command handler; gateway errors are logged and channel runtimes are restarted with exponential backoff (`CHANNEL_RESTART_POLICY` in `src/gateway/server-channels.ts`)

**Patterns:**
- CLI commands use `defaultRuntime.exit(1)` to terminate with an error code
- Gateway catches per-channel errors and schedules restarts with backoff (initial 5s, max 5m)
- Unhandled rejections are caught globally in `src/infra/unhandled-rejections.ts` and log+exit
- `src/runtime.ts` `RuntimeEnv` abstraction lets tests inject non-exiting error handlers

## Cross-Cutting Concerns

**Logging:** Subsystem loggers via `src/logging/subsystem.ts`; console capture enabled by `src/logging.ts` `enableConsoleCapture()`
**Validation:** Config validated via `src/config/validation.ts`; plugin contracts validated via `src/plugins/contracts/`
**Authentication:** Per-channel auth adapters in `ChannelPlugin.auth`; provider API keys / OAuth tokens in auth profile store at `~/.openclaw/credentials/`
**Hooks:** Plugin hooks registered via `api.registerHook()`; bundled hooks in `src/hooks/bundled/`; invoked around agent runs and session lifecycle events

---

*Architecture analysis: 2026-03-25*
