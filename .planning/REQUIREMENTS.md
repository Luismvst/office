# Requirements: Office — AI Agent Office

**Defined:** 2026-05-13
**Core Value:** One command brings up an entire portable office of multi-model coding agents — anywhere — and the owner controls them all from a phone.

## v1 Requirements

Requirements for MVP1. Each maps to exactly one roadmap phase.

### Foundation & Bootstrap

- [ ] **FND-01**: User can clone the repo and run `docker compose up -d` on a clean Ubuntu/Debian/Mac/Windows host with Docker; the full stack (backend + Redis + Caddy + socket-proxy + web) starts in under 2 minutes.
- [ ] **FND-02**: On first run, the system auto-generates a master encryption key and admin password, writes them to `${VOLUME}/INITIAL_SECRETS.txt` (mode 0600), prints to stdout, and renders a phone-scannable QR code with the access URL.
- [ ] **FND-03**: System refuses to start when its data volume sits on NFS/SMB/FUSE (boot-time `statfs.f_type` check) to prevent SQLite WAL corruption.
- [ ] **FND-04**: System verifies host clock skew is under 60 seconds vs UTC at boot and refuses to start otherwise (prevents JWT/TLS breakage).
- [ ] **FND-05**: All persistent state (DB, secrets, agent workspaces, session JSONLs, logs) lives in a single Docker named volume `office_data` with fixed UID 10001 ownership.
- [ ] **FND-06**: Backend runs Node 22 LTS + Fastify 5 + TypeScript + Drizzle + better-sqlite3 in a single container.

### Authentication & Access

- [ ] **AUTH-01**: Admin can log in with the auto-generated password via the web UI; session persists across browser refresh.
- [ ] **AUTH-02**: Admin password is hashed with Argon2id (memory ≥64 MiB, iterations ≥3, parallelism 4).
- [ ] **AUTH-03**: Sessions use `@fastify/jwt` with HS256, 7-day expiry, sliding renewal, and `clockTolerance: 30s`.
- [ ] **AUTH-04**: Admin can change their password from the settings page.
- [ ] **AUTH-05**: `users` and `sessions` tables match Better-Auth schema for the multi-user migration path.

### Secrets Vault

- [ ] **VAULT-01**: Admin can add a provider API key (Anthropic, OpenAI, Gemini, Ollama) via the web UI; the key is encrypted at rest with AES-256-GCM via `node:crypto`.
- [ ] **VAULT-02**: Vault uses a two-layer Bitwarden-style model: `MASTER_KEY` (from env) wraps an internal `DATA_KEY`.
- [ ] **VAULT-03**: Admin can rotate or revoke any key from the web UI.
- [ ] **VAULT-04**: The UI never displays the plaintext key — only a SHA-256 fingerprint + last-used timestamp.
- [ ] **VAULT-05**: Vault writes an audit log entry on every read, write, rotate, or revoke (with reason + actor).
- [ ] **VAULT-06**: API keys are injected into agent containers as env vars at spawn time and never written to agent container disk.
- [ ] **VAULT-07**: Backend logs use Pino `redact` to scrub keys, tokens, and authorization headers from all output streams.

### Provider Abstraction

- [ ] **PROV-01**: System exposes a `Provider` interface (TypeScript) with: `spawnAgent`, `sendMessage`, `getStatus`, `terminate`, `parseUsage`.
- [ ] **PROV-02**: Claude Code provider implemented using `@anthropic-ai/claude-agent-sdk` ^0.2.
- [ ] **PROV-03**: OpenAI provider implemented using `openai` ^6 Responses API.
- [ ] **PROV-04**: A new provider can be added by writing a single `ProviderDescriptor` + adapter file without touching core code.
- [ ] **PROV-05**: An ESLint rule bans `provider === 'claude'` / `'openai'` outside the adapter layer.
- [ ] **PROV-06**: Contract tests run identical scenarios against every registered provider.

### Agent Runtime & Sandbox

- [ ] **AGENT-01**: User can create an agent against a remote git URL (clones it), a local workspace path, or an empty scratch workspace.
- [ ] **AGENT-02**: Each agent runs in a dedicated Docker container with `--read-only` root, `cap-drop=ALL`, no Docker socket access, and an iptables egress allowlist (`api.anthropic.com`, `api.openai.com`, the repo's git host) — everything else blocked including `169.254.169.254` metadata.
- [ ] **AGENT-03**: On clone, the system quarantines `.claude/`, `.mcp.json`, `.cursor/`, `.devcontainer/`, `.git/hooks/`, `.vscode/tasks.json`, and package install scripts; quarantined files are moved to `.office-quarantine/` until admin reviews and approves them.
- [ ] **AGENT-04**: Each agent container has memory cap (default 512 MiB) and CPU cap (default 1.0 vCPU); backend refuses to spawn an agent if host has < 1 GiB free RAM.
- [ ] **AGENT-05**: Agent lifecycle: `create` → `start` → `pause` → `resume` → `archive` → `delete`. All transitions exposed via REST API and web UI.
- [ ] **AGENT-06**: Claude Code agents persist session state in `CLAUDE_CONFIG_DIR=/data/claude/<agent-id>/` (inside the named volume) so sessions resume across host restart.
- [ ] **AGENT-07**: Session resume verifies `(cwd, JSONL path, model)` match the original session and fails loudly on mismatch (prevents the silent-new-session bug, issue #555).
- [ ] **AGENT-08**: Backend uses `dockerode` against the `Tecnativa/docker-socket-proxy` sidecar with allowlist `CONTAINERS,IMAGES,NETWORKS,EXEC,POST=1` and denylist `SWARM,SECRETS,VOLUMES,BUILD,PLUGINS`.
- [ ] **AGENT-09**: Each agent enforces per-session cost cap (default $1) and per-agent daily cost cap (default $10); breaching either pauses the agent.

### Workspace & Git

- [ ] **WS-01**: Admin can browse the agent's workspace tree from the web UI (read-only listing).
- [ ] **WS-02**: System supports both clone-per-agent and worktree-per-agent (final default decided in Phase 3 spike).
- [ ] **WS-03**: Admin can view git status, branch, and diff for an agent's workspace.
- [ ] **WS-04**: Admin can trigger commit + push to a branch from the web UI (using a vault-stored git credential).

### Live Monitoring

- [ ] **MON-01**: For every agent, the dashboard shows: project name, current model, context % used, turns, cost (USD), current task description, last activity timestamp.
- [ ] **MON-02**: Context % comes from the Claude status-line JSON (`context_window.used_percentage`) for Claude agents and from SDK `usage.input_tokens / context_window_size` for non-Claude agents — never from char-count estimation.
- [ ] **MON-03**: System subscribes to Claude's `compact_boundary` `SystemMessage` and renders a visual marker in the agent timeline when compaction occurs.
- [ ] **MON-04**: Agent state machine: `idle | working | needs-input | completed | failed | stopped`; mirrors Anthropic Agent View naming.

### 2D Office Frontend

- [ ] **OFFICE-01**: Web UI renders a PixiJS v8 canvas with a desk grid; each agent occupies one desk.
- [ ] **OFFICE-02**: Each desk shows a sprite avatar with color overlay: green = idle/healthy, yellow = working, red = context > 70% OR error state.
- [ ] **OFFICE-03**: A card overlay anchored to each desk displays the live status fields from MON-01.
- [ ] **OFFICE-04**: Clicking a desk opens a side panel with the agent's chat/task interface.
- [ ] **OFFICE-05**: Office canvas renders at 60 fps on a $200 Android phone with up to 12 active agents.
- [ ] **OFFICE-06**: Adding or removing an agent updates the canvas without a full page reload.
- [ ] **OFFICE-07**: PixiJS resources (textures, sprites) are explicitly destroyed when an agent is removed (Pixi v8 memory leak mitigation).

### Chat / Task Interface

- [ ] **CHAT-01**: Admin can send a message to any agent from the web UI side panel; the agent's reply streams back in real time.
- [ ] **CHAT-02**: Code diffs in agent replies render as a unified diff with syntax highlight.
- [ ] **CHAT-03**: Admin can stop the current turn at any time (sends SIGINT to the agent process).
- [ ] **CHAT-04**: Admin can start a fresh session for any agent (resets context).
- [ ] **CHAT-05**: Per-tool approval: when an agent wants to run a destructive tool (`Bash`, `Write`), the UI shows an approve/deny dialog with the tool input.
- [ ] **CHAT-06**: Approval mode is per-agent: `bypassPermissions` (auto-approve, default for trusted repos) or `acceptEdits` (prompt on destructive).
- [ ] **CHAT-07**: Markdown rendering uses a sanitized renderer (DOMPurify) — no inline scripts, no unsafe attributes.

### Message Bus (MVP1 scaffold, MVP2 activation)

- [ ] **BUS-01**: System exposes a `MessageBus` interface with `publish(topic, message)` and `subscribe(topic, handler)`.
- [ ] **BUS-02**: Implementation is Redis pub/sub (events) + Redis Streams (durable per-agent inboxes); `ioredis` 5 on Node, `redis-py` 5 on Python.
- [ ] **BUS-03**: Every `Task` message carries a `fromAgentId` discriminator; in MVP1 only `fromAgentId="human"` is allowed (guard rejects others).
- [ ] **BUS-04**: WebSocket gateway broadcasts bus events to all connected clients; supports `lastEventId` resume from Streams on reconnect.
- [ ] **BUS-05**: Per-agent FIFO inbox queues messages from multiple senders (web + Telegram) and serializes them to the agent.

### WebSocket Gateway

- [ ] **WS-API-01**: Backend exposes WebSocket endpoint at `/ws` using `@fastify/websocket` 11.2.
- [ ] **WS-API-02**: Heartbeat ping/pong every 25 seconds; clients reconnect on missed pong (WebKit 228296 mitigation).
- [ ] **WS-API-03**: Client supplies `lastEventId` on reconnect; server replays missed events from Redis Streams.
- [ ] **WS-API-04**: Every API response carries `X-Office-Protocol-Version` header; clients detect mismatch and prompt "update available."

### Telegram Integration

- [ ] **TG-01**: Python sidecar container (image `office-telegram`) reuses the existing `claude-telegram-agent` code (python-telegram-bot 22.7) and connects to Redis.
- [ ] **TG-02**: Admin can register Telegram bot tokens in the vault and attach a bot to any agent without restarting any container.
- [ ] **TG-03**: A bot can be re-attached to a different agent on the fly via `/attach <agent-id>` slash command (atomic switch with `deleteWebhook(drop_pending_updates=true)`).
- [ ] **TG-04**: Telegram commands: `/start`, `/status`, `/new`, `/stop`, `/projects`, `/cd`, `/attach`, `/model`. Free-text messages route to the attached agent.
- [ ] **TG-05**: Only whitelisted Telegram user IDs (set per bot) can interact with the bot; everyone else gets a deny + log entry.
- [ ] **TG-06**: Only one Telegram poller per bot token at any time; enforced by Redis lock `SET tg:lock:<botid> NX PX 30000` with 25-second renewal.

### Deployment & Portability

- [ ] **DEP-01**: `docker compose up -d` brings the full stack online with no manual config beyond an optional `.env` file (only needed for `OFFICE_DOMAIN` / `OFFICE_TAILSCALE`).
- [ ] **DEP-02**: Reverse proxy is Caddy 2 with three auto-detected modes: internal CA (default LAN), Let's Encrypt (when `OFFICE_DOMAIN` is set), Tailscale (when `OFFICE_TAILSCALE=1` — best-effort, optional).
- [ ] **DEP-03**: All container images are pinned by SHA in `docker-compose.yml`.
- [ ] **DEP-04**: Multi-arch images (amd64 + arm64) built and pushed to GHCR on every release tag.

### Operations

- [ ] **OPS-01**: CLI command `office backup` streams a `tar.gz` of the `office_data` volume to stdout (or a path); checkpoints SQLite WAL before tar.
- [ ] **OPS-02**: CLI command `office restore <file>` extracts a backup into a fresh volume and runs DB migration if schema versions mismatch.
- [ ] **OPS-03**: CLI command `office update` pulls latest images, runs DB migrations, and restarts containers — preceded by an automatic pre-update backup.
- [ ] **OPS-04**: CLI command `office reset` (with `--confirm`) deletes the volume; only path that performs `docker compose down -v`.
- [ ] **OPS-05**: Healthcheck endpoint `/health` returns 200 with backend / Redis / DB status.

### Mobile / PWA

- [ ] **MOB-01**: Web client ships as a PWA (`vite-plugin-pwa` 0.21, Workbox); installable from Chrome/Safari "Add to home screen."
- [ ] **MOB-02**: PWA service worker uses `registerType: 'autoUpdate'` + `skipWaiting()` + `clientsClaim()`; mismatched protocol version triggers an "update available" toast.
- [ ] **MOB-03**: Office canvas, agent cards, and chat interface are usable in portrait orientation on a 375px-wide phone screen.
- [ ] **MOB-04**: Web push notifications via VAPID for: agent finished, agent needs approval, agent error, context red (> 70%).
- [ ] **MOB-05**: Chat input supports voice dictation via the Web Speech API (Chrome/Edge/Safari iOS).
- [ ] **MOB-06**: Approve/deny taps trigger haptic feedback via `navigator.vibrate`.

### Security Hardening

- [ ] **SEC-01**: HTTP responses set strict CSP (no `unsafe-inline`, no `unsafe-eval`, nonce-based script-src).
- [ ] **SEC-02**: 2FA/TOTP available as an optional second factor for the admin account (RFC 6238).
- [ ] **SEC-03**: Per-agent egress rules verified by a CI integration test (spawn container, attempt connect to `1.1.1.1` → must fail).
- [ ] **SEC-04**: All cookies are `Secure`, `HttpOnly`, `SameSite=Lax`.
- [ ] **SEC-05**: Server logs no secret values, no request bodies for `/auth/login`, no Authorization headers.

## v2 Requirements (Deferred)

### Inter-Agent Coordination (MVP1.5 / MVP2)
- **A2A-01**: Agents can publish tasks to the bus targeting other agents (flip `fromAgentId` guard).
- **A2A-02**: "Assign to" UI lets admin route an in-progress task from one agent to another.
- **A2A-03**: Agent runner exposes a `bus.publish` tool so an agent can delegate sub-tasks.
- **A2A-04**: Canvas renders delegation arrows showing active task handoffs.

### Multi-User & RBAC
- **MU-01**: Admin can invite team members; each gets isolated `CLAUDE_CONFIG_DIR` and vault namespace.
- **MU-02**: RBAC: admin, owner, contributor, viewer.
- **MU-03**: Per-user API tokens for programmatic access.
- **MU-04**: SSO via OIDC (Google, GitHub).

### Additional Providers
- **PROV-V2-01**: Gemini provider via `@google/genai` ^2.
- **PROV-V2-02**: Ollama provider via `ollama` ^0.5 for local-only models.
- **PROV-V2-03**: DeepSeek / Mistral / Anthropic-via-Bedrock / OpenAI-via-Azure routes.

### Visual & UX Polish
- **UX-V2-01**: Sprite animations (walking between desks, typing, idle bobs).
- **UX-V2-02**: Multi-room layouts (per-team rooms, "lab" zone, "archive" zone).
- **UX-V2-03**: Office layout editor (drag desks, save layout).
- **UX-V2-04**: Light/dark theme + per-user accent color.

### Advanced Ops
- **OPS-V2-01**: Auto-update via Watchtower with admin-confirmation gate.
- **OPS-V2-02**: Postgres migration path with a one-shot `office migrate-to-postgres` command.
- **OPS-V2-03**: Per-agent log retention policies (rotate, compress, archive to S3).
- **OPS-V2-04**: Built-in Prometheus metrics + Grafana dashboard recipes.

## Out of Scope (MVP1)

| Feature | Reason |
|---------|--------|
| Multi-user / team accounts | Single admin in MVP; multi-tenant adds RBAC and namespace complexity that obscures the core thesis. v2. |
| Native iOS/Android apps | Web PWA is sufficient for phone control; native adds App Store friction without value. |
| Built-in code review / CI integration | The office is for running agents, not gating their output. Out of scope; agents can be told to open PRs. |
| Agent template marketplace | Adds publishing/distribution mechanics distracting from MVP. Post-MVP. |
| Cost-aware model auto-routing | Models cost differently and people care, but it's a UX rabbit hole. Manual model selection in MVP. |
| Embedded Monaco editor | Agents edit code, humans review diffs. Embedded editor is huge complexity for marginal value. |
| Embedded PTY / terminal | Same as above; this is a control panel, not VS Code. |
| Multi-floor / 3D / isometric canvas | One floor, top-down, 2D. Anything more is feature creep. |
| Autonomous agent-to-agent action | Scaffolded only; activation is MVP1.5 / MVP2 behind an explicit guard. |
| Cluster / Kubernetes / Swarm deployment | Single host is the deployment unit by design (portability story). Cluster mode breaks the volume-tar story. |
| Non-Docker host support (PTY fallback) | Docker is a hard requirement. Trivial to install everywhere. Reduces attack surface. |
| Shared RAG / knowledge base across agents | Tempting but expensive; defer to v2 once we know what users actually need. |
| Auto-rotate keys | Manual rotation is fine for single admin; auto-rotate requires lifecycle integration with provider dashboards. v2. |
| Native push to iOS via APNs | Web Push (VAPID) covers Android + recent iOS Safari; APNs adds Apple-specific cert mgmt. v2. |

## Traceability

Filled by the roadmapper. Empty for now.

| Requirement | Phase | Status |
|-------------|-------|--------|
| _(populated after roadmap creation)_ | | |

**Coverage:**
- v1 requirements: 95 total
- Mapped to phases: 0 (pending roadmap)
- Unmapped: 95

---
*Requirements defined: 2026-05-13*
*Last updated: 2026-05-13 after initial definition*
