# Roadmap: Office — AI Agent Office

## Overview

Seven phases deliver MVP1 in dependency-correct order. Each phase ends in a demoable end-to-end capability the owner can use from a phone or browser — no horizontal layers, no theoretical work. Phase 1 makes `docker compose up -d` a real first-run experience (portable + secure-by-default + clock-safe + admin login + Docker socket proxy). Phase 2 lands the encrypted vault and the `Provider` plug-in surface (no spawning yet). Phase 3 spawns the first real Claude agent — the largest phase, owning six of the top-ten pitfalls (malicious-repo quarantine, egress allowlist, session resume, cost caps, context % accuracy, OOM control). Phase 4 lights up the 2D office on a phone: WebSocket gateway, Redis message bus, PixiJS canvas, click-to-chat, PWA shell. Phase 5 proves the provider abstraction by adding OpenAI **before** anything else gets piled on top (catches retrofit pain early). Phase 6 attaches Telegram bots per agent through the multi-channel FIFO queue, exercising the bus as the second consumer. Phase 7 hardens for production (Caddy LE+Tailscale, `office update`/`backup`/`restore` CLI, CSP, 2FA, web push) and polishes mobile (PWA install, haptics, voice, 60fps target). Inter-agent activation (A2A) is explicitly out of MVP1 — the bus interface is scaffolded in Phase 4 so v2 activation costs days, not weeks.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: foundations-and-first-run** - Portable compose stack, first-run bootstrap, admin login, Docker socket proxy, FS+clock guards
- [ ] **Phase 2: vault-and-provider-abstraction** - AES-256-GCM two-layer vault, secrets UI, `Provider` interface + Claude descriptor (no spawn yet)
- [ ] **Phase 3: single-agent-spawn-claude** - First Claude agent end-to-end: sandboxed container, repo quarantine, session resume, status pipeline, cost caps
- [ ] **Phase 4: websocket-bus-and-2d-office** - WS gateway with replay, Redis message bus, PixiJS office canvas, click-to-chat side panel, PWA shell
- [ ] **Phase 5: second-provider-openai** - `agent-openai` image + adapter, contract tests, `provider===` lint guard — proves plug-in surface
- [ ] **Phase 6: telegram-sidecar-and-queue** - Python sidecar, `/attach`, vault-backed bot tokens, polling lock, per-agent FIFO across web+TG
- [ ] **Phase 7: production-hardening-and-mobile-polish** - Caddy LE/Tailscale, `office` CLI (backup/restore/update), 2FA, web push, haptics, CSP, 60fps

## Phase Details

### Phase 1: foundations-and-first-run
**Goal**: Owner runs `docker compose up -d` on a clean host and reaches a working admin login in under two minutes, with the entire security/portability foundation already locked in.
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: FND-01, FND-02, FND-03, FND-04, FND-05, FND-06, AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05, AGENT-08, DEP-01, DEP-03
**Success Criteria** (what must be TRUE):
  1. Owner runs `docker compose up -d` on a clean Ubuntu/Debian/Mac/Windows-Docker host and the full stack (backend + Redis + Caddy + socket-proxy) starts in under 2 minutes.
  2. On first run, the system writes `${VOLUME}/INITIAL_SECRETS.txt` (0600), prints the master key + admin password + access URL to stdout, and renders a phone-scannable QR code — owner can log in within the same session.
  3. Owner can log in via the web UI with the auto-generated Argon2id-hashed admin password; the session persists across browser refresh; password change works from a settings page.
  4. System refuses to start if the data volume sits on NFS/SMB/FUSE (FS-type guard) or if host clock skew exceeds 60 seconds vs UTC — both with clear actionable error messages.
  5. Backend connects to Docker only through `Tecnativa/docker-socket-proxy` (verifiable: `curl --unix-socket /var/run/docker.sock` from backend fails; `curl http://socket-proxy:2375/containers/json` succeeds); all images are SHA-pinned in `docker-compose.yml`.
**Plans**: TBD
**UI hint**: yes

### Phase 2: vault-and-provider-abstraction
**Goal**: Owner can add and rotate encrypted provider keys via the web UI, and the codebase has a `Provider` plug-in surface ready for the first spawning implementation — keys never reach the browser and never leak into logs.
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: VAULT-01, VAULT-02, VAULT-03, VAULT-04, VAULT-05, VAULT-06, VAULT-07, PROV-01, PROV-04, PROV-05, SEC-04, SEC-05
**Success Criteria** (what must be TRUE):
  1. Owner can add an Anthropic / OpenAI / Gemini / Ollama API key via the web UI; the key is encrypted at rest with AES-256-GCM under a two-layer master-key/data-key model and shows as ciphertext in SQLite.
  2. Owner can rotate or revoke any key from the UI; UI displays only a SHA-256 fingerprint + last-used timestamp — never the plaintext — and every read/write/rotate/revoke produces an audit log entry.
  3. Backend logs (Pino `redact`) scrub Authorization headers, API key fields, and bot tokens from every output stream; `/auth/login` request bodies never appear in logs; cookies are `Secure`, `HttpOnly`, `SameSite=Lax`.
  4. The `Provider` TypeScript interface (`spawnAgent`, `sendMessage`, `getStatus`, `terminate`, `parseUsage`) exists; a Claude `ProviderDescriptor` is registered (no spawning yet); an ESLint rule fails the build if `provider === 'claude' | 'openai'` appears outside the adapter layer.
  5. Owner can see a list of registered providers + models in the UI populated from the descriptor registry — a new descriptor file is enough to make it appear (no core code touched).
**Plans**: TBD
**UI hint**: yes

### Phase 3: single-agent-spawn-claude
**Goal**: Owner can create a Claude Code agent against a real git repo or scratch dir, watch it work safely inside a hardened container, and resume its session across host restart — the riskiest end-to-end loop in the product.
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: PROV-02, AGENT-01, AGENT-02, AGENT-03, AGENT-04, AGENT-05, AGENT-06, AGENT-07, AGENT-09, WS-01, WS-02, WS-03, WS-04, MON-01, MON-02, MON-03, MON-04
**Success Criteria** (what must be TRUE):
  1. Owner creates an agent against a remote git URL, a local workspace path, or an empty scratch workspace; the agent container spawns with `--read-only`, `cap-drop=ALL`, no Docker socket, memory/CPU caps, and an iptables egress allowlist (`api.anthropic.com`, `api.openai.com`, repo git host) — connecting to `1.1.1.1` from inside the container fails.
  2. On clone, the system quarantines `.claude/`, `.mcp.json`, `.cursor/`, `.devcontainer/`, `.git/hooks/`, `.vscode/tasks.json`, and package install scripts to `.office-quarantine/`; owner approves or rejects each via a Workspace-Trust-style dialog before the agent starts using them.
  3. Owner can issue `create → start → pause → resume → archive → delete` from the UI / REST API; sessions persist at `CLAUDE_CONFIG_DIR=/data/claude/<agent-id>/` and resume after a container restart with `(cwd, JSONL path, model)` integrity verified — silent-new-session bug (#555) is impossible.
  4. Dashboard surfaces per-agent: project name, current model, context % (from Claude status-line JSON or SDK `usage.input_tokens / context_window_size` — never char-count), turns, cost USD, current task, last activity, state (`idle | working | needs-input | completed | failed | stopped`); a visual marker fires on `compact_boundary` SystemMessage.
  5. Per-session cost cap ($1 default) and per-agent daily cap ($10 default) pause the agent at the threshold; owner can view git status / branch / diff for the workspace and trigger commit-and-push using vault-stored git credentials.
**Plans**: TBD

### Phase 4: websocket-bus-and-2d-office
**Goal**: Owner installs the PWA on a phone, opens the office, sees a colored sprite for the live agent, taps the desk, and chats with the agent in real time — the "wow" moment of the product.
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: BUS-01, BUS-02, BUS-03, BUS-04, BUS-05, WS-API-01, WS-API-02, WS-API-03, WS-API-04, OFFICE-01, OFFICE-02, OFFICE-03, OFFICE-04, OFFICE-06, OFFICE-07, CHAT-01, CHAT-02, CHAT-03, CHAT-04, CHAT-05, CHAT-06, CHAT-07
**Success Criteria** (what must be TRUE):
  1. Owner opens the web UI on a phone or laptop and sees a PixiJS v8 canvas with a desk grid; each active agent occupies a desk with a sprite avatar colored green/yellow/red (red = context > 70% OR error) and a card overlay showing the MON-01 status fields; adding or removing an agent updates the canvas without a full page reload, and removed sprites have their textures explicitly destroyed.
  2. Owner taps/clicks a desk and a side chat panel opens; sending a message streams the agent reply back in real time with code diffs rendered as unified diff (syntax-highlighted) and markdown rendered through DOMPurify (no inline scripts).
  3. Owner can stop the current turn (SIGINT to the agent process), start a fresh session, and on per-tool approval mode (`acceptEdits`) gets an approve/deny dialog with the tool input before `Bash`/`Write` runs; `bypassPermissions` mode is available per-agent for trusted repos.
  4. WebSocket gateway at `/ws` heartbeats every 25 seconds, replays missed events from Redis Streams when the client supplies `lastEventId` on reconnect (phone-background recovery works without manual refresh), and every API response carries `X-Office-Protocol-Version`.
  5. The Redis `MessageBus` interface (`publish/subscribe` over pub/sub, durable per-agent inbox over Streams) is wired and enforces a `fromAgentId="human:<userId>"` guard — the durable FIFO inbox per agent already exists so Phase 6's Telegram is a "second consumer" not a "first integration."
**Plans**: TBD
**UI hint**: yes

### Phase 5: second-provider-openai
**Goal**: Owner can spawn an OpenAI (Codex/GPT) agent next to a Claude agent in the same office — proves the `Provider` abstraction holds before MVP1 ships.
**Mode:** mvp
**Depends on**: Phase 4
**Requirements**: PROV-03, PROV-06
**Success Criteria** (what must be TRUE):
  1. Owner adds an OpenAI key in the vault, picks "OpenAI" + a model in the create-agent form, and an `agent-openai` container spawns; the OpenAI Responses API streams responses back through the same WS pipeline as Claude.
  2. The OpenAI agent reports normalized status (model, context % from `usage.input_tokens / context_window`, cost USD, turns, current task) — the same fields the dashboard already renders for Claude — without any new branching in the dashboard or chat code.
  3. Provider contract tests run identical scenarios (streaming, tool use, usage parsing, session resume, error) against both Claude and OpenAI adapters and pass; CI fails if `provider === 'claude'` or `'openai'` is added outside the adapter layer.
  4. Owner sees both a Claude desk and an OpenAI desk side-by-side in the 2D office with correct color status and card overlays — visually indistinguishable framework-wise.
**Plans**: TBD
**UI hint**: yes

### Phase 6: telegram-sidecar-and-queue
**Goal**: Owner registers a Telegram bot token, attaches the bot to any agent, and drives that agent from a phone messenger with chat parity to the web UI — and can hot-switch the bot to a different agent without restarting anything.
**Mode:** mvp
**Depends on**: Phase 5
**Requirements**: TG-01, TG-02, TG-03, TG-04, TG-05, TG-06
**Success Criteria** (what must be TRUE):
  1. The Python sidecar container (`office-telegram`, reuses existing `claude-telegram-agent` code with `python-telegram-bot` 22.7) connects to Redis on boot, exchanges its shared secret for a sidecar JWT against the backend, and registers presence.
  2. Owner adds a bot token to the vault and attaches the bot to agent A from the web UI; sending a free-text message to the bot in Telegram routes to agent A; agent replies stream back to Telegram in chat — no container restart required.
  3. Owner sends `/attach <agent-B-id>` in Telegram; the sidecar atomically releases polling on the old binding (`deleteWebhook(drop_pending_updates=true)`), switches the target, and resumes; subsequent messages reach agent B. The complete command set works: `/start`, `/status`, `/new`, `/stop`, `/projects`, `/cd`, `/attach`, `/model`.
  4. Only whitelisted Telegram user IDs (configured per bot) get responses; everyone else gets a silent deny + audit log entry; per-agent FIFO inbox serializes inputs so a simultaneous web + Telegram send to the same agent produces deterministic transcript order with source-channel chips.
  5. Exactly one polling loop runs per bot token at any moment, enforced by a Redis lock (`SET tg:lock:<botid> NX PX 30000` renewed every 25s); a second sidecar instance detects the lock and waits — no 409 conflicts.
**Plans**: TBD
**UI hint**: yes

### Phase 7: production-hardening-and-mobile-polish
**Goal**: Office is hardened for real-world deployment on a public VPS or Tailscale tailnet, fully phone-installable as a PWA with web push and haptics, and operable through a tight `office` CLI for backup/restore/update.
**Mode:** mvp
**Depends on**: Phase 6
**Requirements**: DEP-02, DEP-04, OPS-01, OPS-02, OPS-03, OPS-04, OPS-05, MOB-01, MOB-02, MOB-03, MOB-04, MOB-05, MOB-06, SEC-01, SEC-02, SEC-03, OFFICE-05
**Success Criteria** (what must be TRUE):
  1. Owner deploys to a public VPS with `OFFICE_DOMAIN=office.example.com`; Caddy auto-provisions a Let's Encrypt certificate (staging-first guard prevents rate-limit lockout); alternative `OFFICE_TAILSCALE=1` mode reaches the office at `https://<host>.<tailnet>.ts.net` without port-forwarding. Multi-arch (amd64 + arm64) images are pushed to GHCR on every release tag.
  2. `office backup` streams a checkpointed `tar.gz` of `office_data` to stdout/path; `office restore <file>` rebuilds a fresh volume and runs DB migration on version mismatch; `office update` pulls latest images, runs a pre-update backup, runs migrations, restarts containers; `office reset --confirm` is the only path that performs `docker compose down -v`; `/health` returns 200 with backend/Redis/DB status.
  3. Owner installs the PWA from Chrome/Safari "Add to home screen"; service worker uses `registerType: 'autoUpdate'` + `skipWaiting()` + `clientsClaim()`; a protocol-version mismatch shows an "update available" toast; the office canvas, agent cards, and chat are usable in portrait on a 375px-wide screen at 60fps with up to 12 active agents on a $200 Android phone.
  4. Owner receives web push notifications (VAPID) for: agent finished, agent needs approval, agent error, context red >70%; chat input supports voice dictation via Web Speech API; approve/deny taps trigger `navigator.vibrate` haptic feedback.
  5. HTTP responses set strict CSP (no `unsafe-inline`, no `unsafe-eval`, nonce-based `script-src`); admin can enable optional TOTP 2FA (RFC 6238) on the account; a CI integration test spawns an agent container and confirms a connect to `1.1.1.1` is dropped by the egress allowlist.
**Plans**: TBD
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6 → 7

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. foundations-and-first-run | 0/TBD | Not started | - |
| 2. vault-and-provider-abstraction | 0/TBD | Not started | - |
| 3. single-agent-spawn-claude | 0/TBD | Not started | - |
| 4. websocket-bus-and-2d-office | 0/TBD | Not started | - |
| 5. second-provider-openai | 0/TBD | Not started | - |
| 6. telegram-sidecar-and-queue | 0/TBD | Not started | - |
| 7. production-hardening-and-mobile-polish | 0/TBD | Not started | - |

---

## Notes for Plan-Phase

- **Phase 3 is the largest phase (17 requirements)** and owns 6 of the Top-10 pitfalls (#1 malicious-repo, #4 cost runaway, #5 context %, #6 session resume, #10 memory, plus partial overlap with #2 via socket-proxy already in Phase 1). Expect 4-5 plans. Sub-slicing into 3.0 (sandbox + spawn), 3.1 (status pipeline), 3.2 (lifecycle + cost caps), 3.3 (workspace/git) is acceptable.
- **Phase 4 is large (22 requirements)** but cohesive — it's the "WS + bus + 2D + chat" big-bang. Expect 4 plans (WS+heartbeat, Bus+queue, PixiJS canvas, Chat panel).
- **Phase 5 is intentionally small (2 requirements)** — it's the abstraction validation. Do not pad. A single plan with strong contract tests is the win condition.
- **Phase 7 may sub-slice** into 7.0 (production hardening: Caddy LE/TS, CLI, CSP, 2FA, push) and 7.5 (mobile polish: 60fps target, haptics, voice, PWA install) to keep plans tight.
- **Inter-agent comms activation (A2A-01..04) is v2 / MVP1.5** — explicitly tracked in REQUIREMENTS.md v2 section, NOT in v1 phases.

---

*Roadmap created: 2026-05-13*
*Coverage: 90 / 90 v1 requirements mapped, no orphans*
