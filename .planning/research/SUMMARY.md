# Research Summary — Office (AI Agent Office)

**Project:** Office — AI Agent Office
**Domain:** Self-hostable multi-AI agent orchestration dashboard (Docker-native, single-volume portable, phone-first PWA, Telegram integration)
**Researched:** 2026-05-13
**Confidence:** HIGH overall (stack/architecture/pitfalls); MEDIUM on a few choice points (Telegram IPC, worktree-vs-clone, anti-feature thresholds)

## TL;DR

The Office is a single-VPS Docker Compose stack: one **backend** container (Node 22 + Fastify 5 + dockerode + better-sqlite3 + Drizzle) supervises N **agent** containers (Claude Code SDK / OpenAI Responses / Gemini / Ollama, each isolated with `--read-only`, dropped caps, and an iptables egress allowlist), with a **Caddy** reverse proxy doing zero-config HTTPS, a **Redis** pub/sub bus serving both inter-agent communication and the Python **Telegram sidecar**, and a **React 19 + Vite 6 + PixiJS 8** PWA rendering a 2D top-down office on phones first. All state lives in a single named volume so `tar volume + scp + compose up` relocates the whole office. The non-negotiable security frame is CVE-2025-59536 / CVE-2026-21852: agents must run sandboxed with egress allowlists, the Docker socket must be proxied (never mounted directly into the backend), and API keys must be vault-encrypted (Argon2id for passwords, AES-256-GCM for secrets, two-layer Bitwarden-style master/data key model) and never logged or shipped to the browser. Eight features differentiate this against Claude Squad, Octogent, Kanna, Devin, and Anthropic Agent View; the most defensible combination is **2D office + phone-first PWA + Telegram-per-agent + `docker compose up` portability + multi-provider + hardened sandbox** — no surveyed competitor ships all six.

## Recommended Stack

- **Runtime:** Node.js 22 LTS (`.nvmrc`, `engines.node >=22 <25`). Not 24, not Bun.
- **Backend:** Fastify 5 + `@fastify/websocket` 11.2 (ws@8). Not Socket.IO/Express/Hono/NestJS.
- **Docker control:** `dockerode` 5 via **socket-proxy sidecar** (`Tecnativa/docker-socket-proxy`). Backend never mounts `/var/run/docker.sock`.
- **DB:** `better-sqlite3` 12 + Drizzle ORM (dual-dialect SQLite→Postgres).
- **Cache/Bus:** Redis 7 via `ioredis` 5 (Node) + `redis-py` 5 (Python). Serves: message bus, Telegram IPC, WS replay streams, Telegram polling lock.
- **Frontend:** React 19 + Vite 6 + PixiJS 8.17+ + `@pixi/react` v8 + Zustand 5 + Tailwind v4 + shadcn/ui.
- **Crypto/Auth:** `node:crypto` AES-256-GCM (NOT libsodium); **Argon2id (not bcrypt)** via `argon2`; `@fastify/jwt` 9 with `clockTolerance: 30`.
- **AI SDKs:** `@anthropic-ai/claude-agent-sdk` ^0.2, `openai` ^6 (Responses API only), `@google/genai` ^2 (NOT `@google/generative-ai` — sunset 2025-08-31), `ollama` ^0.5. Thin in-house TS `Provider` interface, no LiteLLM.
- **Reverse proxy:** Caddy 2 + `caddy-docker-proxy`. Three modes: internal CA (default LAN), Let's Encrypt (with `OFFICE_DOMAIN`), Tailscale (optional sidecar).
- **Telegram sidecar:** Python 3.12 + `python-telegram-bot` 22.7 (existing claude-telegram-agent code containerized).
- **PWA:** `vite-plugin-pwa` 0.21 (Workbox; `registerType: 'autoUpdate'`).
- **Logging:** Pino 9 with redact paths; OTel-pino transport scaffolded.
- **Init:** Docker `--init` (tini) on every agent.
- **Monorepo:** pnpm 9 workspaces + Turborepo 2. `packages/{backend,web,shared}`, `services/telegram`, `images/agent-*`, `infra/compose`.
- **CI/CD:** GitHub Actions → GHCR multi-arch.
- **NOT using:** Lucia Auth (deprecated 2025-03) — use `@fastify/jwt` + Argon2id with Better-Auth-compatible schema.

## Table Stakes vs Differentiators

### Table Stakes
Multi-provider (Claude SDK + OpenAI Responses) with per-agent model lock · encrypted vault, per-agent key assignment, keys never in browser · sandboxed Docker per agent (read-only, cap-drop ALL, egress allowlist) · lifecycle CRUD from git URL/local path/scratch · worktree-per-agent · live monitoring (ctx %, cost, tokens, current task) · 2D PixiJS office with status colors and card overlay · chat with diff view + tap-to-approve + stop-turn · Telegram sidecar with `/attach`/`/status`/`/stop`/`/new` parity · `docker compose up -d` <2 min on clean host with first-run bootstrap + single named volume + tar-backup + Caddy HTTPS · phone-first PWA + web push + 2FA · scaffolded inter-agent message bus.

### Eight Differentiators
1. 2D top-down PixiJS office with sprite animations & status colors
2. Phone-first PWA with web push + voice dictation + haptics
3. Telegram-bot-per-agent, hot-attachable, shared encrypted vault
4. `docker compose up` portability with `tar`-the-volume migration
5. Pluggable multi-provider backend via thin TS `Provider` interface
6. Sandboxed Docker + per-agent egress allowlist (CVE-2025-59536 response)
7. Scaffolded inter-agent message bus (Redis from day one)
8. Per-agent egress firewall + secret-vault audit log in admin UI

### Anti-Features (avoid in v1)
Monaco editor; embedded PTY; cost-aware auto-routing; agent marketplace; multi-floor canvas; autonomous A2A; auto-rotate keys; native mobile wrappers; multiplayer view; cluster/K8s; shared RAG; Arena mode; 3D/isometric view.

## Architecture in One Page

```
                                INTERNET / LAN
                                      |
                  +-------------------+--------------------+
                  |           [reverse-proxy]              |
                  |     Caddy 2 (LE / internal / TS)       |
                  +-------------------+--------------------+
                                      |
   +-------------------+--------------+--------------+------------------+
   |                   |                             |                  |
+--+----------+  +-----+---------+         +---------+------+   +-------+-------+
|  [web-ui]   |  |  [backend]    |         | [telegram]     |   |   [redis]     |
|  React 19   |<>|  Node 22      |<------->|  Python sidecar|<->| pub/sub +     |
|  PixiJS 8   |WS|  Fastify 5    | HTTP+WS |  python-tg-bot |   | streams + KV  |
|  PWA        |  |  Orchestrator |  Redis  |                |   |               |
+-------------+  +-+-----------+-+         +----------------+   +---------------+
                   | docker-   | better-sqlite3 + Drizzle
                   | socket-   | AES-256-GCM 2-layer vault
                   | proxy     | JWT + Argon2id
                   v           v
              +--------------------------------+
              |   DOCKER ENGINE (via proxy)    |
              +-+--------------+---------------+
                |              |
   +------------v--+    +------v--------------+         +---------------------+
   | [agent-N]     |    | [agent-M]           |   ...   | [agent-K]           |
   | Claude SDK    |    | OpenAI Responses    |         | Gemini / Ollama     |
   | read-only +   |    | read-only +         |         | read-only +         |
   | cap-drop=ALL +|    | cap-drop=ALL +      |         | cap-drop=ALL +      |
   | egress-fence  |    | egress-fence        |         | egress-fence        |
   +---------------+    +---------------------+         +---------------------+

  --- ALL STATE in single named volume `office_data` --------------------------
```

**Narrative:** Only the backend talks to Docker (via socket-proxy sidecar with endpoint allowlist `CONTAINERS/IMAGES/NETWORKS/EXEC/POST=1`, denying `SWARM/SECRETS/VOLUMES/BUILD/PLUGINS`). Agent containers never see the socket, other agents, or the vault. Keys decrypt in backend RAM, inject as `Env` at `createContainer` — never on disk, never logged, never to browser. Status flows via three normalized signal sources for Claude (status-line + HTTP hooks + SDK result) and one for non-Claude (`runner.js` reads `usage`), funneled into `MessageBus` (Redis pub/sub for events, Streams for durable inboxes). WebSocket gateway broadcasts to web clients with `lastEventId` resume; Telegram sidecar is the second consumer. Persistence anchored in a single named volume — `tar`-relocatable with fixed UID 10001 in agent images.

## Top 10 Pitfalls to Address in MVP1

1. **#1 Malicious-repo RCE via `.claude/`, `.mcp.json`, hooks** (CRITICAL × NEAR-CERTAIN) — CVE-2025-59536 + CVE-2026-21852. Quarantine `.claude/`, `.mcp.json`, `.cursor/`, `.devcontainer/`, `.git/hooks/` on clone; iptables egress allowlist; pin SDK ≥2.0.65.
2. **#2 Docker socket mounted directly into backend** (CRITICAL × HIGH) — CVE-2025-9074 class. Use `Tecnativa/docker-socket-proxy` with endpoint allowlist; `DOCKER_HOST=tcp://socket-proxy:2375`.
3. **#3 API keys / Telegram tokens leaking into logs and transcripts** (CRITICAL × HIGH) — May 2026 SIGMA-bot $200K drain. Pino `redact`; transcript regex-strip; key SHA-256 fingerprint for display.
4. **#4 Rate-limit and cost runaway with shared keys** (CRITICAL × HIGH) — Per-agent `max_usd_per_session` + sliding-window TPM; org circuit breaker at $20/day; loop detector.
5. **#11 Master key recoverability** (HIGH × MEDIUM) — Two-layer Bitwarden model: `MASTER_KEY` wraps internal `DATA_KEY`. Print to 3 sinks on first run (stdout + `${VOLUME}/INITIAL_SECRETS.txt` 0600 + QR).
6. **#10 Memory exhaustion** (HIGH × HIGH) — `--memory 512m --memory-swap 512m` per agent; pre-spawn admission control; document "8 GB for 4 agents".
7. **#7 SQLite WAL on networked storage** (HIGH × MEDIUM) — Boot-time `statfs.f_type` refuses NFS (0x6969) / SMB / FUSE; bundle SQLite ≥3.51.3.
8. **#6 Session resume edge cases** (HIGH × HIGH) — Issue #555: silent new session on cwd mismatch. Canonicalize cwd to `/work/repo`; pin `CLAUDE_CONFIG_DIR`; verify (cwd, JSONL, model) on resume and fail loudly.
9. **#13 Mobile WebSocket disconnects** (HIGH × NEAR-CERTAIN) — WebKit 228296. 25s heartbeat ping/pong; `visibilitychange` reconnect; Redis Streams sequence-replay using `lastEventId`.
10. **#9 Telegram 409 conflict + token security** (HIGH × HIGH) — Single Redis polling lock (`SET tg:lock:<botid> NX PX 30000`); atomic agent switch with `deleteWebhook(drop_pending_updates=true)`; @BotFather revoke deep-link.

**Honorable mention #5: Context % accuracy** — without trusting SDK `usage.input_tokens` and subscribing to `compact_boundary`, the 70% red dot lies.

## Suggested Phase Structure

The architecture researcher proposed 7 mandatory + 1 optional phases keyed to **technical de-risking order**; the features researcher proposed 6 phases keyed to **user-facing capability delivery order**. They disagree on two points:

- **Where Office Visualization sits** — features puts it between Workspace and Phone/Telegram; architecture puts the 2D frontend right after the first single-agent spawn.
- **Where the Second Provider sits** — architecture wants it BEFORE Telegram (to prove the `Provider` abstraction before MVP ships); features groups providers up-front.

### Recommendation: Resolve in favor of architecture's ordering, with a concession

**Reasoning:** Architecture's phasing is dependency-graph-correct. Validating the `Provider` abstraction with a second provider (OpenAI) before piling Telegram + polish on top catches the single biggest "would-have-to-rewrite" risk (Pitfall #17) before MVP ships. Stream shapes diverge a lot between Anthropic/OpenAI/Gemini, so this is cheap insurance.

**Concession:** the 2D canvas in Phase 4 ships in a deliberately minimal form (desk grid + colored sprite + card overlay + click-to-chat); polish (sprite animations, layout editor, room grouping) folds into a "Phase 7.5 — UI polish" slice running in parallel with Phase 7 hardening.

### Proposed Phases

| # | Phase | Research Flag |
|---|-------|---------------|
| 1 | **Foundations & First Run** — monorepo, compose with socket-proxy + Redis + Caddy + sqlite, first-run bootstrap (3-sink credentials), named-volume layout, `office backup`/`restore` CLI, FS-type guard (#7), clock-skew guard (#18), JWT + Argon2id, image-SHA pinning | **NEEDS RESEARCH** on docker-socket-proxy allowlist + iptables `DOCKER-USER` egress chain |
| 2 | **Vault + Provider Abstraction** — AES-256-GCM two-layer key model, secrets UI, `ProviderDescriptor` registry with Claude descriptor (no spawning), Pino redact, audit log | STANDARD PATTERN |
| 3 | **Single-Agent Spawn (Claude only)** — `AgentManager`, dockerode via socket-proxy, `agent-claude` image, `runner.js` with status-line + hooks + SDK result, lifecycle CRUD, egress allowlist iptables, `CLAUDE_CONFIG_DIR` persistence + resume, per-agent mem/CPU caps, `.claude/` quarantine (#1) | **NEEDS RESEARCH** on status-line JSON shape + hook callbacks + `.claude/` quarantine UX |
| 4 | **WebSocket Gateway + Bus + Minimal 2D Frontend** — Redis `MessageBus` (full interface), WS gateway with `lastEventId` resume (#13), PixiJS office (grid + sprites + card + click-to-chat), PWA shell + SW, voice + haptics scaffolded | **NEEDS RESEARCH** on iOS Safari WS suspension (WebKit 228296) + Pixi v8 destroy semantics |
| 5 | **Second Provider + Plug-in Validation (OpenAI)** — `agent-openai` image, OpenAI descriptor, normalized status (`usage`-based ctx%, no hooks), adapter contract tests, lint banning `provider===` outside adapter (#17) | **NEEDS RESEARCH** on OpenAI Responses streaming event shape + tool_call delta normalization |
| 6 | **Telegram Sidecar + Multi-Channel Queue** — containerized claude-telegram-agent, `RemoteAgentClient`, `/attach`, sidecar JWT, vault-backed tokens, per-agent FIFO queue (#16), Redis polling lock (#9), webhook fallback | **NEEDS RESEARCH** on python-telegram-bot 22.7 webhook/polling switchover + Redis pub/sub Python channel ack semantics |
| 7 | **Production Hardening + Mobile Polish** — Caddy mode B (LE staging-first) + mode C (Tailscale), `office update` CLI with pre-update backup, SQLite WAL checkpoint, CSP + sanitized markdown, 2FA/TOTP, web-push VAPID, per-tool timeline + cost roll-up | STANDARD PATTERNS — mode C (Tailscale) needs a 1-day spike |
| 8 (optional MVP1.5) | **Inter-Agent Comms Activation** — flip `fromAgentId` guard, "assign to" UI, runner.js `bus.publish` tool, delegation arrows on canvas | DEFERRED to MVP2 |

**Dogfood loop:** From Phase 3 onward, the Office can become its own development environment for Phases 4+. Recommend introducing this as a Phase 4 milestone gate.

## Cross-Cutting Decisions Made

- **Redis in compose stack from day one** — serves message bus, WS replay, Telegram polling lock, Telegram IPC. No EventEmitter detour.
- **Docker socket proxy from day one** — `Tecnativa/docker-socket-proxy`. Retrofitting impossible without breaking installs.
- **Caddy three modes**, auto-detected: internal CA (default LAN), LE (with `OFFICE_DOMAIN`), Tailscale (optional sidecar). Default internal CA avoids ACME rate-limit lockout.
- **Egress allowlist mandatory** — iptables `DOCKER-USER` on agent network. Drops everything including `169.254.169.254` metadata.
- **Argon2id (not bcrypt) from day one** — overrides PROJECT.md "bcrypt"; same JWT surface; `users` table matches Better Auth schema for MVP2 migration.
- **Single named Docker volume, NOT bind mount** — Fixed UID 10001 in agent images for cross-host tar/restore.
- **AES-256-GCM via `node:crypto`** with two-layer Bitwarden-style master/data key model. Master key written to 3 sinks on first run.
- **Pino redact in same phase as vault** — retroactive scrubbing impossible.
- **Status state computed from SDK-reported `usage`**, never char-counts. Subscribe to `compact_boundary`.
- **Per-agent FIFO queue interface in Phase 4** so Telegram (Phase 6) is "second consumer," not "first integration".
- **`docker compose down -v` is a foot-gun** — ship `office reset` CLI with confirmation; don't document `down -v` in README.
- **Worktree-vs-clone deferred to Phase 3 spike.**
- **Frontend bundle versioning + PWA `skipWaiting()` + `clientsClaim()` mandatory** — every `/api` response carries `X-Office-Protocol-Version`; SW shows "update available" toast on mismatch.

## Open Questions Deferred to Phase-Specific Research

1. **Phase 1 — Egress allowlist implementation:** netns + iptables `DOCKER-USER` vs proxy sidecar (mitmproxy/Envoy) vs egress gateway.
2. **Phase 3 — Worktree-per-agent vs clone-per-agent v1 decision** with explicit spike criteria.
3. **Phase 3 — `.claude/`/`.mcp.json`/`.cursor/`/`.devcontainer/` quarantine UX** (VS Code Workspace Trust-style approval dialog design).
4. **Phase 3 — Claude status-line wire format integration** against pinned SDK version.
5. **Phase 4 — iOS Safari PWA WebSocket suspension edge cases** on real iOS 18+ devices.
6. **Phase 4 — PixiJS v8 destroy semantics + low-end Android budget** (1-hour mobile soak).
7. **Phase 5 — OpenAI Responses streaming event normalization** (text delta, tool_call delta, completion).
8. **Phase 6 — Python sidecar ↔ Node backend transport finalization** (channel naming, ack semantics, ordering).
9. **Phase 7 — Better Auth schema column-naming compatibility verification.**
10. **Phase 7 — Caddy mode C (Tailscale) integration spike** (capability flags, cert flow, sidecar topology).
11. **Cross-phase — First-run bootstrap UX wording** for 3-sink credentials output.

## Confidence Scorecard

| Dimension | Confidence | Source |
|-----------|------------|--------|
| Stack — established layers (Fastify, dockerode, Pino, Caddy, PixiJS, Drizzle, better-sqlite3, Tailwind, pnpm/Turborepo) | HIGH | STACK.md — 2026 ecosystem reports + official docs |
| Stack — evolving layers (Better Auth schema-compat, Serwist vs vite-plugin-pwa, Argon2-vs-bcrypt) | MEDIUM-HIGH | STACK.md |
| Stack — Telegram IPC (Redis vs HTTP) | MEDIUM | STACK.md — right answer given inter-agent bus is also coming |
| Features — competitive landscape | HIGH | FEATURES.md — 13 reference products surveyed |
| Features — anti-feature thresholds | MEDIUM | FEATURES.md — competitor-observation only, no user research |
| Architecture — container topology, secrets injection, status pipeline, backup/restore, bootstrap | HIGH | ARCHITECTURE.md — grounded in Claude SDK + dockerode + Caddy docs |
| Architecture — MVP2 bus consumer-group details | MEDIUM | ARCHITECTURE.md — interface right, Stream choreography TBD |
| Architecture — egress allowlist mechanism | MEDIUM | ARCHITECTURE.md — pattern right, rule format TBD in Phase 1 |
| Pitfalls — CVE & incident-driven | HIGH | PITFALLS.md — public CVEs + named incidents |
| Pitfalls — library-specific | HIGH | PITFALLS.md — public issues + vendor docs |
| Pitfalls — UX/operational | MEDIUM-HIGH | PITFALLS.md — patterns from adjacent products |

**Overall confidence:** HIGH. The four research files agree on the dependency graph, stack choices, and security frame, and the two phasing proposals can be reconciled cleanly. Genuine open questions are concentrated at per-phase-research level.

### Gaps to Address
- No direct user research yet — feature anti-thresholds and phone-first UX are inferred. Mitigation: ship MVP1 narrow and measure.
- `.claude/` config quarantine UX needs a Phase 3 design pass.
- Cross-host UID/GID portability solved on paper but needs real restore-from-tar smoke test in CI.
- Tailscale mode C described but not prototyped; treat as "best-effort optional" in Phase 7.
