# External Integrations

**Analysis Date:** 2026-03-25

## AI / LLM Providers

**Anthropic:**
- Integration: Embedded via `@mariozechner/pi-ai` and `@mariozechner/pi-agent-core` (core agent runtime)
- Direct HTTP: `src/agents/` (streaming, compaction, tool calls)
- Auth: API key; `extensions/anthropic/` plugin for setup
- Vertex AI variant: `@anthropic-ai/vertex-sdk` — `src/agents/anthropic-vertex-provider.ts`

**OpenAI:**
- Integration: `extensions/openai/openai-provider.ts`, OpenAI Responses API at `src/gateway/openresponses-http.ts`
- WebSocket streaming: `src/agents/openai-ws-connection.ts`, `src/agents/openai-ws-stream.ts`
- Codex variant: `extensions/openai/openai-codex-provider.ts`
- Auth: API key; `OPENAI_API_KEY`

**Google (Gemini):**
- Integration: `extensions/google/` plugin, `src/agents/` (Gemini streaming, batch embeddings)
- Auth: Service account / OAuth via `src/infra/gemini-auth.ts`; Vertex AI via `src/agents/models-config.providers.ts`
- Batch embeddings: `src/memory/batch-gemini.ts`, `src/memory/embeddings-gemini.ts`

**AWS Bedrock:**
- Discovery: `@aws-sdk/client-bedrock` — `src/agents/bedrock-discovery.ts`
- Auth: AWS credentials (standard SDK chain)

**Ollama (local):**
- Integration: `src/agents/ollama-models.ts`, `src/agents/ollama-stream.ts`
- Auto-discovery: `src/agents/models-config.providers.ts` (local network scan)
- Auth: No auth required; local endpoint

**Microsoft / GitHub Copilot:**
- GitHub Copilot: `src/providers/github-copilot-auth.ts`, `src/providers/github-copilot-models.ts`
- Azure OpenAI / Copilot proxy: `extensions/copilot-proxy/`, `extensions/microsoft/`
- Auth: GitHub OAuth tokens, Azure API keys

**Other LLM Providers (extension plugins):**
- Mistral: `extensions/mistral/` — `src/memory/embeddings-mistral.ts` for embeddings
- Cloudflare AI Gateway: `extensions/cloudflare-ai-gateway/`, `src/agents/cloudflare-ai-gateway.ts`
- Vercel AI Gateway: `extensions/vercel-ai-gateway/`, `src/agents/vercel-ai-gateway.ts`
- Hugging Face: `extensions/huggingface/` — `src/agents/huggingface-models.ts`
- Together AI: `extensions/together/`
- OpenRouter: `extensions/openrouter/`
- XAI (Grok): `extensions/xai/`
- Perplexity: `extensions/perplexity/`
- NVIDIA: `extensions/nvidia/`
- Chutes: `extensions/chutes/` — OAuth flow at `src/agents/chutes-oauth.ts`
- Kilocode: `extensions/kilocode/`
- FAL.ai: `extensions/fal/`
- Venice AI: `extensions/venice/`
- BytePlus / VolcEngine: `extensions/byteplus/`, `extensions/volcengine/`
- MiniMax: `extensions/minimax/`
- Moonshot (Kimi): `extensions/moonshot/`, `extensions/kimi-coding/`
- ModelStudio: `extensions/modelstudio/`
- Qwen Portal: `extensions/qwen-portal-auth/`
- Qianfan (Baidu): `extensions/qianfan/`
- Xiaomi: `extensions/xiaomi/`
- ZAI: `extensions/zai/`
- SgLang: `extensions/sglang/`
- vLLM: `extensions/vllm/`
- Voyage AI: `src/memory/embeddings-voyage.ts`, `src/memory/batch-voyage.ts` (embeddings)
- OpenCode: `extensions/opencode/`, `extensions/opencode-go/`
- node-llama-cpp (local): peer dependency for local inference
- Pi AI (embedded): `@mariozechner/pi-agent-core`, `@mariozechner/pi-coding-agent`

## Messaging Channels

**Telegram:**
- SDK: `grammy` + `@grammyjs/runner` + `@grammyjs/transformer-throttler`
- Extension: `extensions/telegram/`
- Auth: Bot token from `@BotFather`; stored in gateway state

**Discord:**
- SDK: `@buape/carbon` (HTTP interactions) + `@discordjs/voice` + `discord-api-types`
- Extension: `extensions/discord/`
- Auth: Discord bot token; raw token stored in state

**Slack:**
- SDK: `@slack/bolt` + `@slack/web-api`
- Extension: `extensions/slack/`
- Auth: Slack app OAuth tokens (Socket Mode)

**WhatsApp:**
- SDK: `@whiskeysockets/baileys` (WhatsApp Web reverse-engineered protocol)
- Extension: `extensions/whatsapp/`
- Auth: QR code pairing flow; session stored in gateway state

**Microsoft Teams:**
- SDK: `@microsoft/agents-hosting` (Bot Framework)
- Extension: `extensions/msteams/`
- Auth: Azure Bot credentials (App ID + Password)

**Matrix:**
- SDK: `matrix-js-sdk` + `@matrix-org/matrix-sdk-crypto-nodejs` (E2EE)
- Extension: `extensions/matrix/`
- Auth: Matrix homeserver credentials; crypto state persisted

**Signal:**
- Integration: signal-cli linked device (no first-party SDK)
- Extension: `extensions/signal/`
- Auth: signal-cli REST API; linked device setup

**LINE:**
- SDK: `@line/bot-sdk` (in root `package.json`)
- Code: `src/line/` (accounts, send, template-messages)
- Auth: LINE channel access token

**iMessage:**
- Integration: `src/imessage/` — macOS-only via applescript/jxi
- Auth: macOS Contacts / system access

**IRC:**
- Extension: `extensions/irc/`
- Auth: IRC server credentials

**Feishu (Lark):**
- Extension: `extensions/feishu/`
- Auth: Feishu app credentials

**Google Chat:**
- Extension: `extensions/googlechat/`
- Auth: Google service account

**Mattermost:**
- Extension: `extensions/mattermost/`

**Nextcloud Talk:**
- Extension: `extensions/nextcloud-talk/`

**Nostr:**
- Extension: `extensions/nostr/`

**Synology Chat:**
- Extension: `extensions/synology-chat/`

**Twitch:**
- Extension: `extensions/twitch/`

**Tlon (Urbit):**
- Extension: `extensions/tlon/` — SDK: `@tloncorp/api`

**Zalo / ZaloUser:**
- Extensions: `extensions/zalo/`, `extensions/zalouser/`

**BlueBubbles (iMessage bridge):**
- Extension: `extensions/bluebubbles/`

**Voice Call:**
- Extension: `extensions/voice-call/`, `extensions/talk-voice/`
- Integration: PTY + ElevenLabs TTS (see Speech section)

**Web Chat (built-in):**
- Built-in channel: `src/channels/web/` (HTTP + WebSocket)
- Control UI: `ui/` (Lit/Vite web app)

## Speech / Voice

**ElevenLabs:**
- Extension: `extensions/elevenlabs/` (TTS)
- Auth: ElevenLabs API key

**Microsoft Edge TTS:**
- SDK: `node-edge-tts` (built-in dependency)
- Code: `src/tts/`
- Auth: None (free Edge TTS endpoint)

**Sherpa-ONNX TTS:**
- Integration: `src/agents/skills/` — binary-based local TTS

## Web Search / Browsing

**Brave Search:**
- Extension: `extensions/brave/`
- Auth: Brave Search API key

**Tavily:**
- Extension: `extensions/tavily/`
- Auth: Tavily API key

**Firecrawl:**
- Extension: `extensions/firecrawl/`
- Auth: Firecrawl API key

**Perplexity (web search mode):**
- Extension: `extensions/perplexity/`
- Auth: Perplexity API key

**Playwright (browser automation):**
- SDK: `playwright-core` 1.58.2 (built-in dep)
- Code: `src/browser/`, `src/agents/` web tools
- Auth: None (local browser)

## Data Storage

**Databases:**
- SQLite (via `sqlite-vec`): local vector + key-value store for memory subsystem
  - Files: `src/memory/sqlite.ts`, `src/memory/sqlite-vec.ts`
  - Connection: file-based, stored at `OPENCLAW_STATE_DIR`
- LanceDB (`@lancedb/lancedb`): optional vector database for long-term memory
  - Extension: `extensions/memory-lancedb/`
  - Connection: local directory (default) or remote

**File Storage:**
- Local filesystem: all session data, config, credentials stored under `~/.openclaw/` (or `OPENCLAW_STATE_DIR`)
  - Credentials: `~/.openclaw/credentials/`
  - Sessions: `~/.openclaw/agents/<agentId>/sessions/*.jsonl`
  - Config: JSON files managed by `src/infra/json-file.ts`
- No remote object storage

**Caching:**
- In-memory caches throughout (model catalog, channel status, heartbeat state)
- Filesystem-backed JSON stores for persistence (`src/infra/json-files.ts`)

## Authentication & Identity

**Auth Provider:**
- Custom device-based auth (`src/infra/device-identity.ts`, `src/infra/device-auth-store.ts`)
- Pairing token system: `src/infra/pairing-token.ts`, `src/infra/device-pairing.ts`
- Gateway connection auth: `src/gateway/auth.ts`, `src/gateway/connection-auth.ts`
- Auth profiles (multi-key rotation): `src/agents/auth-profiles.ts`
- Credentials stored at `~/.openclaw/credentials/` (custom format)

**Mobile Push:**
- Apple APNS: `src/infra/push-apns.ts`, `src/infra/push-apns.relay.ts`
  - Auth: APNS JWT / p8 key

## Networking / Infrastructure

**Service Discovery:**
- mDNS/Bonjour: `@homebridge/ciao` — `src/infra/bonjour-ciao.ts` (local network discovery)
- DNS-based wide-area: `src/infra/widearea-dns.ts`

**VPN / Overlay Networks:**
- Tailscale: `src/infra/tailscale.ts`, `src/gateway/server-tailscale.ts`
  - Used for remote gateway access without port forwarding

**SSH Tunneling:**
- `src/infra/ssh-tunnel.ts` — dynamic tunnel setup for remote connections

**WebSocket Gateway:**
- `src/gateway/server.ts` — core WebSocket server (WS protocol)
- `src/gateway/client.ts` — gateway client

## Monitoring & Observability

**OpenTelemetry (optional plugin):**
- Extension: `extensions/diagnostics-otel/`
- SDKs: Full `@opentelemetry` stack (traces, metrics, logs, OTLP exporters)
- Auth: OTLP endpoint + headers (configurable)

**Error Tracking:**
- No built-in cloud error tracking
- Structured logging via `tslog` (`src/logger.ts`)
- macOS unified logs via `scripts/clawlog.sh`

**Logs:**
- JSON-structured logs at runtime via `tslog`
- Gateway logs: `/tmp/openclaw-gateway.log` (self-hosted) or fly.io log streams

## CI/CD & Deployment

**Hosting:**
- Fly.io: `fly.toml` — primary cloud hosting (`openclaw` app, `iad` region)
- Render: `render.yaml` — alternative Docker hosting
- Self-hosted: Docker image (`Dockerfile`, `Dockerfile.sandbox`, `Dockerfile.sandbox-browser`)
- Fly.io secondary: `fly.private.toml` (private deployment)
- Signal bot: Fly.io `flawd-bot` machine

**CI Pipeline:**
- GitHub Actions: `.github/workflows/ci.yml`
- Runner: Blacksmith (`blacksmith-16vcpu-ubuntu-2404`, `blacksmith-32vcpu-windows-2025`) + `macos-latest`
- NPM release: `.github/workflows/openclaw-npm-release.yml`, `.github/workflows/plugin-npm-release.yml`
- Docker release: `.github/workflows/docker-release.yml`
- CodeQL security scanning: `.github/workflows/codeql.yml`

## Protocol Implementations

**ACP (Agent Client Protocol):**
- SDK: `@agentclientprotocol/sdk` 0.16.1
- Code: `src/acp/` — server, client, session mapping, persistent bindings

**MCP (Model Context Protocol):**
- SDK: `@modelcontextprotocol/sdk` 1.27.1
- Code: `src/agents/mcp-stdio.ts`, `src/agents/embedded-pi-mcp.ts`

**Image Generation:**
- Extension: `extensions/fal/` (FAL.ai), others via provider plugins
- Core: `src/image-generation/`

## Webhooks & Callbacks

**Incoming Webhooks:**
- Gateway HTTP server (`src/gateway/server-http.ts`) — receives provider callbacks
- Microsoft Teams: webhook endpoint via Express in `extensions/msteams/`
- LINE: webhook endpoint via `@line/bot-sdk`
- Plugin webhook ingress: `src/plugin-sdk/webhook-ingress.ts` (standardized path registration)

**Outgoing Webhooks:**
- Hooks system: `src/hooks/` — configurable outbound HTTP callbacks on agent events
- Heartbeat events: `src/infra/heartbeat-runner.ts` — scheduled outbound notifications

## Environment Configuration

**Required env vars (production):**
- `OPENCLAW_STATE_DIR` — persistent data directory (default `~/.openclaw`)
- `OPENCLAW_WORKSPACE_DIR` — agent workspace root
- `OPENCLAW_GATEWAY_TOKEN` — gateway auth token (auto-generated on Render)
- `NODE_ENV` — `production` for gateway deployments
- `NODE_OPTIONS` — e.g. `--max-old-space-size=1536` for memory limits

**Secrets location:**
- `~/.openclaw/credentials/` (runtime)
- Render: `SETUP_PASSWORD`, `OPENCLAW_GATEWAY_TOKEN` via Render secrets
- Fly.io: secrets via `fly secrets set`
- Android: `gradle.properties` (local, not committed)

---

*Integration audit: 2026-03-25*
