# Office — AI Agent Office

## What This Is

A self-hostable, portable "AI Agent Office" — a single Docker Compose stack you bring up on any VPS, laptop, or workstation that becomes a 2D top-down virtual office where each desk hosts an AI coding agent (Claude Code, OpenAI Codex/GPT, Gemini, more) bound to a specific git repository or scratch workspace. You watch every agent's live status (model, context %, current task, cost), chat with any of them from a phone or browser, and optionally hand any agent its own Telegram bot for remote control. Designed for a single owner today, the architecture is wired for inter-agent task handoff in MVP2.

## Core Value

**One command brings up an entire portable office of multi-model coding agents — anywhere — and the owner controls them all from a phone.** If everything else breaks, this must work: `docker compose up -d` → log in → add agent against a repo → talk to it from phone or web.

## Requirements

### Validated

(None yet — ship to validate)

### Active

#### Multi-AI Providers
- [ ] Spawn agents using Claude Code SDK (`@anthropic-ai/claude-agent-sdk`)
- [ ] Spawn agents using OpenAI (Codex/GPT via Responses API)
- [ ] Pluggable provider interface so Gemini, DeepSeek, Ollama can be added without rewrites

#### Secrets Vault
- [ ] Encrypt API keys at rest on the host with AES-256-GCM, master key from env or KMS
- [ ] Web UI to add/rotate/revoke provider keys and Telegram bot tokens
- [ ] Per-agent key assignment; keys never leave the backend (browser never sees them)

#### Agent Instances
- [ ] Create agent against a remote git repo URL (clones it), a local workspace path, or a temporary scratch dir
- [ ] Each agent runs inside an isolated Docker container with restricted egress
- [ ] Lifecycle: create → start → pause → resume → archive → delete
- [ ] State and session transcripts persist across host restarts

#### 2D Office Frontend
- [ ] PixiJS canvas rendering a desk grid; each agent is a sprite avatar at a desk
- [ ] Color status: green = idle/healthy, yellow = working, red = context > 70% OR error
- [ ] Card overlay per desk shows project, model, context %, turns, cost, current task
- [ ] Click a desk to open a chat/task panel for that agent
- [ ] Mobile-responsive PWA installable from a phone

#### Telegram Integration
- [ ] Attach a Telegram bot to any agent on the fly, no restart
- [ ] One bot can switch which agent it controls via slash command
- [ ] Bot tokens stored in the same secrets vault

#### Portable Deployment
- [ ] `docker compose up -d` brings the entire stack online in under 2 minutes on a clean host
- [ ] All state (DB, secrets, agent workspaces, session logs) in a single mountable volume
- [ ] Backup = tar of the volume; restore = untar + compose up on any other host
- [ ] First-run auto-bootstrap: generates master key, creates admin user, prints access URL
- [ ] Bundled Caddy reverse proxy for automatic HTTPS via Let's Encrypt (optional toggle)

#### Inter-Agent Comms Interface (MVP1 scaffold for MVP2 feature)
- [ ] Internal message bus interface (in-process EventEmitter or Redis pub/sub, swappable)
- [ ] In MVP1 only humans move tasks between agents — but the API surface is in place

### Out of Scope (MVP1)

- Multi-user / team accounts — single admin user only; multi-user comes post-MVP
- Agents talking to agents autonomously — MVP2 feature; only the interface is scaffolded in MVP1
- Native iOS/Android apps — web PWA is enough for phone control
- Built-in code review / CI integration — out of scope for the office itself
- Marketplace of prebuilt agent templates — post-MVP
- Non-Docker host support (PTY fallback) — Docker is a hard requirement; trivial to install anywhere

## Context

### Existing Assets to Reuse

- **`../claude-telegram-agent/`** — A working Python Telegram bot that already wraps Claude Code with whitelist auth, session management, project switching, and a Windows NSSM service installer. It will be containerized as a Python sidecar (decision: keep Python, don't port to Node) and the Node.js backend will talk to it over HTTP / Redis.

### Prior Research Findings

**Closest existing OSS projects** (to study and borrow patterns from, not clone wholesale):
- `claude-office` (paulrobello) — PixiJS 2D office for Claude Code, FastAPI backend. Closest direct reference for the visualization layer.
- `agent-office` (harishkotra) — Phaser + Colyseus multi-agent office, multiplayer-style.
- `Octogent` (hesamsheikh) — web UI multi-Claude-Code manager with Node.js + WebSocket. Architectural reference for the backend.
- `Claude Squad` (smtg-ai, 5.6K stars) — tmux + git worktrees TUI manager. Reference for the worktree-per-agent pattern.
- `Claude Code Agent Monitor` (hoangsonww) — Express + WebSocket + React monitoring dashboard. Reference for the live-status pipeline.
- `claudebox` (RchGrav) — Docker isolation template for Claude Code.

**Claude Code SDK key facts:**
- Package: `@anthropic-ai/claude-agent-sdk` (renamed from `claude-code`)
- `query()` returns `AsyncGenerator<SDKMessage>`; the final `SDKResultMessage` has `total_cost_usd` and detailed `usage` tokens
- Session persistence lives at `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`
- `CLAUDE_CONFIG_DIR` env var enables per-tenant / per-agent isolation
- Context % is not directly exposed — must be estimated from `input_tokens` against the 200K window, OR read from the status-line JSON which exposes `context_window.used_percentage` directly
- HTTP hooks deliver lifecycle events; `--bg` runs background sessions
- The `SystemMessage` subtype `compact_boundary` fires on auto-compaction

**OpenAI integration:**
- Use OpenAI SDK with the Responses API for Codex/GPT agents
- `usage.input_tokens` / `usage.output_tokens` available per response; context window per model (e.g. 200K for GPT-4.1)

### Security Reality Check

CVE-2025-59536 and CVE-2026-21852 demonstrated that malicious git repos can achieve RCE and API-key exfiltration through Claude Code's `.claude/` hook system. **Implication:** every agent MUST run in a sandboxed Docker container with `--read-only` root, dropped Linux capabilities, and outbound network limited to the relevant provider hosts (`api.anthropic.com`, `api.openai.com`) plus the repo's git host. This is non-negotiable.

## Constraints

- **Tech stack — Backend**: Node.js 22 + TypeScript + Fastify + `@fastify/websocket` (validated against research; revisitable in Phase 1)
- **Tech stack — Frontend**: React 19 + Vite + PixiJS v8 + Tailwind + Zustand
- **Tech stack — DB**: SQLite (better-sqlite3) for MVP; migration path to PostgreSQL preserved via abstraction layer
- **Tech stack — Containers**: Docker (required, no PTY fallback). `dockerode` for the Docker Engine API.
- **Tech stack — Reverse proxy**: Caddy bundled into the compose stack for automatic HTTPS
- **Tech stack — Telegram**: Python sidecar container reusing existing `claude-telegram-agent` code; Node backend talks to it over HTTP or Redis
- **Auth**: Single admin user in MVP. bcrypt + JWT.
- **Portability**: All persistent state in a single named volume. The whole office must be relocatable with `docker compose down && tar volume && scp && tar -x && docker compose up`.
- **Security**: Each agent container runs read-only root, restricted egress, no host Docker socket access from inside agents.
- **Auto-bootstrap**: First run on a clean host must succeed without any manual config beyond `docker compose up`. Master key, admin password, and access URL are generated and printed once.
- **Phone-first UI**: The web UI must be usable from a phone in portrait orientation as the primary device.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Docker required on every host, no PTY fallback | Simpler architecture; Docker is trivial to install on any modern VPS/desktop; security model relies on container isolation | — Pending validation |
| Reuse `claude-telegram-agent` Python code as containerized sidecar (do not port to Node.js) | Code already works, has whitelist auth and session mgmt; less risk and faster than rewrite; Node backend talks to it over HTTP / Redis | — Pending |
| Quality model profile (Opus 4.7) for all GSD planning agents | This is an ambitious multi-component project; deeper analysis pays off here | — Pending |
| Single admin user for MVP, multi-user post-MVP | Reduces auth complexity; owner is sole user today; multi-tenant later via per-user config dirs | — Pending |
| State persistence via a single Docker named volume | Enables the "tar + scp + compose up" portability story | — Pending |
| Inter-agent message bus scaffolded in MVP1 (Redis or in-process EventEmitter behind an interface), only enabled in MVP2 | Avoids retrofit pain; humans move tasks today, agents talk tomorrow | — Pending |
| Auto-clear context between GSD phases | Keeps each phase's planning context fresh, reduces saturation risk | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-05-13 after initialization*
