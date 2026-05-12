# Stack Research — AI Agent Office

**Domain:** Self-hostable multi-AI agent orchestration dashboard (Docker-required, single-volume portable, mobile PWA, Telegram integration)
**Researched:** 2026-05-13
**Overall confidence:** HIGH for established layers (Fastify, dockerode, Pino, Caddy, PixiJS, Drizzle, better-sqlite3, Tailwind, pnpm/Turborepo); MEDIUM for evolving layers (Better Auth, Serwist vs Vite-PWA, LLM router); LOW for areas where adoption is mixed (Node 22 vs 24 choice, Telegram IPC).

---

## TL;DR — The Recommended Stack

| Layer | Pick | Version | One-line Why |
|-------|------|---------|--------------|
| Runtime | Node.js | **22.x LTS** | Battle-tested for native modules (better-sqlite3); 24 LTS valid but raises native-rebuild risk for MVP1 |
| Backend framework | Fastify | **5.x** | Lowest overhead per WS connection, mature plugin model, the `@fastify/websocket` plugin is officially maintained |
| WS plugin | `@fastify/websocket` | **11.2.x** | Built on `ws@8`, route-scoped handlers, integrates with Fastify hooks |
| Docker control | dockerode | **5.x** | De-facto standard, 4M weekly downloads, full Engine API |
| DB (MVP) | better-sqlite3 | **12.x** | Fastest synchronous SQLite driver, single-file portable, the Drizzle SQLite default |
| ORM / query builder | Drizzle ORM | **0.40.x / 1.0** | TS-native schema, dual-driver (SQLite + PG) lets the same schema migrate to Postgres later |
| WebSocket transport | `@fastify/websocket` + `ws@8` + custom heartbeat | — | Avoids Socket.IO overhead; phone reconnection via client-side exponential backoff |
| Frontend framework | React | **19.x** | Required for current `@pixi/react` v8 |
| Build tool | Vite | **6.x** | Required by `@pixi/react` v8; fastest DX |
| 2D renderer | PixiJS | **8.17.x+** | Unopinionated, WebGPU-ready, perfect for static desk grid + sprites with React UI overlays |
| Pixi React glue | `@pixi/react` | **8.x** | Official, React 19 native |
| State (client) | Zustand | **5.x** | Single store, ~3KB, ergonomic for an agent registry streamed over WS |
| Styling | Tailwind CSS | **v4** (Oxide) | Phone-first responsive utilities, shadcn/ui ecosystem, sub-500ms builds |
| Crypto / vault | `node:crypto` (built-in) AES-256-GCM | Node 22 native | Zero external dep, hardware-accelerated; libsodium-wrappers does NOT expose AES-GCM directly |
| Reverse proxy | Caddy | **v2.x** (+ `caddy-docker-proxy` v2.10+) | Automatic HTTPS by default with zero config — the single biggest "one command up" win |
| Telegram IPC | Redis Pub/Sub via `ioredis` (Node) + `redis-py` (Python) | ioredis 5.x / redis-py 5.x | Single dependency already useful for the inter-agent bus (PROJECT.md), trivial bidirectional |
| Python sidecar | `python-telegram-bot` | **22.7+** | Latest, already used by `claude-telegram-agent` |
| PWA tooling | `vite-plugin-pwa` | **0.21.x+** | Most mature for Vite + React; Workbox under the hood |
| Auth | `@fastify/jwt` + bcrypt + Argon2 hash + session cookie | latest | MVP single-admin: do not pull Better Auth's full surface for one user; design table schema for Better Auth migration |
| Logging | Pino | **9.x** | 7x faster than Winston, structured JSON, OTel transport ready |
| Observability | OpenTelemetry SDK | scaffold only in MVP1 | Hook the trace/span IDs into Pino now; wire to a collector later |
| Container init | `tini` | bundled via `docker run --init` | Built into Docker — no extra package |
| Monorepo | pnpm workspaces + Turborepo | pnpm 9.x, turbo 2.x | Polyglot-friendly enough (Python via Dockerfile); 80% of Nx value, 20% of complexity |
| CI/CD | GitHub Actions → GHCR | — | Standard for OSS multi-arch container publishing |

---

## Layer-by-Layer Recommendations

### 1. Runtime: Node.js 22 LTS

**Recommendation:** Node 22.x (Active LTS until April 2027). Pin via `.nvmrc` and `engines.node` ≥ 22.0.0 < 25.

**Why:**
- Node 22 has been the production default for native-addon ecosystems for a year — `better-sqlite3` and `dockerode` are guaranteed to compile cleanly.
- Node 24 LTS (current Active LTS, EOL 2028-04) is great but adds risk: native addons must be rebuilt per major version, and the project has at least one critical native dep (`better-sqlite3`).

**Do NOT use:**
- Node 20 (EOL 2026-04-30 — already past)
- Bun (not yet a 1:1 replacement for the `child_process` + `dockerode` + native-addon surface this project depends on)

**Confidence:** HIGH

---

### 2. Backend Framework: Fastify 5

**Recommendation:** Fastify 5.x with `@fastify/websocket` 11.2.x.

**Why:**
- The lowest per-WebSocket-connection overhead of any framework with first-class TS types — critical when the dashboard maintains one persistent WS per phone client and one to every agent container.
- Fastify's hook system (`onRequest`, `preHandler`, `onClose`) maps cleanly onto agent lifecycle events. Built-in JSON schema validation hardens the public API.
- `@fastify/websocket` is officially maintained, built on `ws@8`, and is route-scoped so admin endpoints and WS endpoints can share auth middleware without ceremony.

**Alternatives considered:**
| Option | Why not |
|--------|---------|
| Hono | Edge-native; not designed for persistent WebSocket servers with stateful container management — picks a different design point |
| NestJS | Built-in `@WebSocketGateway` is nice but DI overhead + decorator-heavy code are unjustified for a single-binary self-hosted app |
| Express + ws | Express is in maintenance; loses you JSON schema validation and the better hook model |
| Effect-TS | Powerful but a steep learning curve that adds risk to MVP1 timeline |

**Do NOT use:**
- Socket.IO on the server side. It's heavier, owns its own protocol, and the namespacing/room model is overkill for "1 admin client + N agent processes." The reconnection logic is what people reach for it for, and that can be done in the client (see WebSocket section).
- uWebSockets.js. Faster, but the API is C-like, types are sparse, and the maintenance posture is one-developer-driven. Not worth the operational risk for an OSS self-hosted product.

**Confidence:** HIGH — validated in the existing constraints and confirmed via package-popularity and framework-comparison research.

---

### 3. AI Provider SDKs

#### 3a. Claude

**Recommendation:** `@anthropic-ai/claude-agent-sdk` ^0.2 (current 0.2.139)

**Why:**
- The renamed SDK from `claude-code`. Returns `AsyncGenerator<SDKMessage>` which streams directly into Fastify WS handlers without buffering.
- The `SDKResultMessage` carries `total_cost_usd` and detailed `usage` for the cost overlay required by FEATURES (status card).
- `CLAUDE_CONFIG_DIR` env var is the per-agent isolation lever — every agent container sets its own.

**Pinning:** `^0.2` is acceptable (semver pre-1.0 allows minor breakage but Anthropic publishes a migration guide for the agent SDK transitions).

#### 3b. OpenAI

**Recommendation:** `openai` ^6 (current 6.37.0), Responses API only

**Why:**
- Responses API is now the primary surface for Codex/GPT — `client.responses.create({ model, input })`. Don't write new code against Chat Completions.
- `usage.input_tokens` / `usage.output_tokens` are returned per response and feed the context-% calculation against the model's context window.

#### 3c. Google Gemini

**Recommendation:** `@google/genai` ^2 (current 2.0.1)

**Why:**
- This is the actively maintained SDK. The legacy `@google/generative-ai` package was sunset on 2025-08-31 — using it now means picking up no fixes.

#### 3d. Ollama (local models)

**Recommendation:** `ollama` (official npm package, currently ~0.5.x)

**Why:**
- Official Ollama team package; streams via `AsyncGenerator`; tiny dependency surface.

#### 3e. Unified router

**Recommendation:** **Build a thin in-house TS interface (the existing project Decision)** — DO NOT pull in LiteLLM (Python) or any TS gateway in MVP1.

**Why:**
- The four providers above already use a similar surface (model name → stream of tokens → final usage block). A 200-line `Provider` interface is more maintainable than a Python sidecar gateway.
- LiteLLM is Python-only and would force a second sidecar with its own lifecycle. Bifrost (Go, OpenAI-compatible) or Vercel AI Gateway are external services — both violate the "single docker compose up" portability requirement.
- The `Provider` interface can be implemented in the same `providers/` directory shape as already planned in PROJECT.md.

**Confidence:** HIGH for individual SDK picks; HIGH for the "no router" decision (router is over-engineering for 4 providers).

---

### 4. Docker Engine Access from Node

**Recommendation:** `dockerode` ^5 (currently 5.0.0, ~4M weekly downloads, 4.9k stars).

**Why:**
- De-facto standard. Wraps the full Engine REST API, supports streaming logs/stats out of the box.
- Backed by `docker-modem` (same author), so the IO layer is mature and shared with most JS Docker libs.

**Connection strategy (security-critical):**
- The **orchestrator backend** mounts `/var/run/docker.sock` read-only on Linux, or talks to the Windows named pipe. This is the unavoidable price of being able to spawn agent containers from the backend.
- **Agent containers MUST NOT have the Docker socket mounted.** They get `--read-only` rootfs, dropped Linux caps (`--cap-drop=ALL` with selective add-back), and an egress allow-list (api.anthropic.com, api.openai.com, generativelanguage.googleapis.com, plus their assigned git host).
- Consider an optional `--init` flag on agent containers (Docker's built-in `tini`) for zombie reaping when an agent spawns child processes.

**Do NOT use:**
- Shelling out to the `docker` CLI. Loses streaming events, harder to handle errors, and adds an extra binary dep inside the orchestrator image.
- `docker-modem` directly. Lower-level; you'd be re-implementing what dockerode already does.
- Privileged DinD inside agents. Critical security risk — the threat model in PROJECT.md (CVE-2025-59536, CVE-2026-21852) makes this non-negotiable.

**Future consideration (NOT MVP1):** Podman REST API is wire-compatible with Docker's. If we eventually want rootless on the host, dockerode-style code can point at the Podman socket with minimal change. Don't switch in MVP1 — Docker is already the requirement.

**Confidence:** HIGH

---

### 5. Database & ORM

#### 5a. Embedded DB: better-sqlite3 ^12

**Why:**
- Fastest SQLite driver in Node, synchronous (which is what you want for SQLite's single-writer model — async wrapping just adds queueing overhead).
- Single `.db` file fits the "single mountable volume" portability story perfectly.
- It is Drizzle's default SQLite driver, so the path forward is paved.

**Alternatives:**
- `libsql` (Turso fork): Adds remote/edge mode and `BEGIN CONCURRENT`. Use this **only** if you later need multi-writer (you won't in MVP1). The async API is a drop-in *replacement* but every call site changes — pick the synchronous version now.
- `node:sqlite` (built-in): Promising, but ecosystem support (Drizzle compat, migrations) is still maturing. Re-evaluate at MVP2.
- PGlite: Postgres-in-WASM is amazing for prototyping but not the right call when better-sqlite3 → real Postgres is the migration path.

**Do NOT use:**
- `sqlite3` (the original `node-sqlite3` callback-based lib). Async API forces you to serialize writes anyway, and it's ~2x slower than better-sqlite3.

#### 5b. ORM: Drizzle ORM (^0.40 / track 1.0)

**Why:**
- TS-native schema (no `.prisma` codegen step → faster CI, no schema-out-of-sync bugs).
- **Dual-dialect:** one schema file targets both SQLite (MVP1) and Postgres (post-MVP). Migrations are written once; only the driver swaps.
- Smallest bundle; cleanest with edge runtimes if a hosted variant ever appears.

**Alternatives:**
| Option | When | Notes |
|--------|------|-------|
| Kysely | If you want pure query-builder, no ORM | Lower-level; reasonable second pick, fewer migration tools |
| Prisma | If you have a team of 5+ and value `prisma studio` GUI | Generator step + binary engine + larger footprint hurt "one command up" |

**Do NOT use:**
- TypeORM (decorators, slower release cadence, larger surface area).
- Sequelize (legacy patterns, no first-class TS).

**Confidence:** HIGH — both picks are validated by 2026 ecosystem reports and match the migration story explicitly stated in PROJECT.md constraints.

---

### 6. WebSocket Transport

**Recommendation:**
- **Server:** `@fastify/websocket` 11.2.x (wraps `ws@8`).
- **Wire format:** JSON messages with a `type` discriminator (e.g. `agent.status`, `agent.log`, `chat.token`). Compress with `permessage-deflate` only for log floods; leave off for low-latency chat tokens.
- **Heartbeat:** Custom ping/pong every 25s. On the server, close stale connections at 60s of no pong (idle phone in pocket).
- **Client reconnection:** Hand-rolled exponential backoff (250ms → 8s, jittered), starting from network/visibility-change events. Use the browser's `online`/`offline` and `visibilitychange` events as triggers.

**Why:**
- Plain `ws` + a tiny client wrapper is **simpler** than Socket.IO for the 1-admin + N-agents topology and avoids Socket.IO's auto-reconnect quirks on iOS Safari PWA backgrounding.
- The PWA backgrounded on iOS will pause the socket; explicit visibility-change handling fixes the "I came back to the office tab and it's stale" UX.

**Do NOT use:**
- Socket.IO. Heavier, owns its own protocol (no plain `wscat` debugging), and the reconnection logic still doesn't perfectly handle iOS PWA suspension — you still have to wire visibility events. Costs more than it gives.
- Server-Sent Events (SSE). One-way; you need bidirectional for chat with agents.

**Confidence:** MEDIUM-HIGH — the heartbeat / visibility-change pattern is the standard fix for the mobile PWA case, but it does need testing on real iOS Safari and Chrome Android.

---

### 7. Frontend: React 19 + Vite 6 + PixiJS 8

**Recommendation:**
- React 19.x (latest stable)
- Vite 6.x
- PixiJS 8.17.x+
- `@pixi/react` 8.x

**Why:**
- `@pixi/react` v8 was rewritten "exclusively for React 19" — using React 18 means you can't use the new pixi-react JSX pragma cleanly.
- PixiJS 8 is the right choice over Phaser 4 for this use case: the office is **mostly a static desk grid with occasional sprite animations and React UI overlays**, not a game. Phaser's built-in physics, scene system, and game-loop opinions are dead weight here. PixiJS's "do what you want with sprites" philosophy fits perfectly.
- PixiJS 8 ships WebGPU support — future-proofing without forcing a migration.

**Phaser 4 — NOT recommended:**
- Built around scenes/game-loops/physics that don't model a dashboard well.
- React integration is via third-party glue; less mature than `@pixi/react` v8.
- Larger bundle.

**Confidence:** HIGH — directly validated by the existing project constraint, and reinforced by PixiJS React v8 design intent.

---

### 8. Client State: Zustand 5

**Recommendation:** Zustand 5.x.

**Why:**
- The agent registry is a **single, shared, frequently-streamed store** — one source of truth that every desk-sprite and every chat panel subscribes to. That's exactly the Zustand sweet spot (single store, selector-based subscriptions, minimal re-render cost at ~3KB gzipped).
- Tiny mental model — perfect for a one-developer codebase.
- `subscribe` with a selector lets PixiJS render code consume state outside React's reconciliation when you need that escape hatch.

**Alternatives:**
| Option | When better |
|--------|-------------|
| Jotai | Many *independent* atoms (per-feature flags, derived computed state). Overkill for 1 big stream |
| Valtio | Mutating-style API; great DX but harder to debug at scale |
| TanStack Query | For HTTP fetch caching alongside Zustand — **add this for REST endpoints** (it's a complement, not a substitute) |

**Do NOT use:**
- Redux Toolkit. ~15KB + boilerplate for the same outcome.
- Recoil. Effectively unmaintained.

**Confidence:** HIGH

---

### 9. Secrets Vault Encryption

**Recommendation:**
- **Cipher:** AES-256-GCM via Node's built-in `node:crypto` (`crypto.createCipheriv('aes-256-gcm', key, iv)`).
- **Key derivation:** Master key from env var `OFFICE_MASTER_KEY` (32 random bytes, base64). Use HKDF-SHA256 (`crypto.hkdfSync`) to derive per-purpose subkeys (`provider-keys`, `telegram-tokens`, `session-cookies`).
- **Master key storage:** Print on first-run bootstrap **once** to stdout, instruct the operator to save it. Also write to `<volume>/master.key` mode 0600 owned by the orchestrator user. Auto-load from file on subsequent starts so `docker compose up -d` continues to work.
- **Per-secret IV:** Random 12 bytes per encryption. Store as `iv:authtag:ciphertext` (all base64) in the SQLite vault row.

**Why:**
- AES-256-GCM is the IETF AEAD standard, hardware-accelerated on every modern x86/ARM CPU.
- Node 22's `node:crypto` is mature, KeyObject-friendly, and zero extra dependencies — matches the "minimal moving parts" portability goal.
- `libsodium-wrappers-sumo` does **not** expose AES-GCM (only ChaCha20-Poly1305). Picking libsodium would either force ChaCha20 (fine, but non-standard for "API key vault" wording) or force you to fall back to Web Crypto for AES — extra complexity for no win.

**Do NOT:**
- Use `crypto.createCipher` (deprecated, no IV).
- Store the master key in the same SQLite DB as the encrypted secrets (defeats the purpose).
- Use bcrypt for *encryption* (it's a one-way hash; some people confuse the two).

**Ordering implication for the roadmap:** The vault and its master-key bootstrap must precede any phase that adds provider API keys to the UI. Roadmap Phase 1 should establish the vault before Phase 2 wires up the providers.

**Confidence:** HIGH

---

### 10. Reverse Proxy: Caddy 2 (with caddy-docker-proxy)

**Recommendation:** Caddy v2 + `lucaslorentz/caddy-docker-proxy` image (Caddyfile generated from Docker labels).

**Why:**
- Caddy issues Let's Encrypt certificates *automatically with zero config* — exactly the "one command up" requirement. Just set `caddy: office.example.com` as a label.
- `caddy-docker-proxy` watches the Docker socket, generates a Caddyfile in memory, and reloads on container changes. No separate Caddyfile to maintain in version control.
- Smallest operational surface — single binary, single image.

**Traefik v3 — NOT picked, but reasonable second:**
- Equally good at label-based routing. Slightly faster in benchmarks. Larger configuration surface and more options.
- Caddy wins specifically on "automatic HTTPS with literally zero config" — that's the differentiator for first-run UX.

**Do NOT use:**
- Nginx + Certbot. More moving parts, manual renewal scripts, doesn't fit "trivial first-run."
- Nginx Proxy Manager. Web-GUI proxy management duplicates what the office UI is for; one more thing to expose.

**HTTPS toggle:** The compose file should accept `OFFICE_DOMAIN` env var. If set, Caddy issues a Let's Encrypt cert; if blank, Caddy serves on `:443` with a self-signed cert (or HTTP-only on `:80` for LAN-only deploys).

**Confidence:** HIGH

---

### 11. Python Telegram Sidecar & Node↔Python IPC

**Recommendation:**
- **Sidecar:** Existing `claude-telegram-agent` code, containerized. Python 3.12 base image (or distroless if size matters).
- **Library:** `python-telegram-bot` 22.7+ (latest in the 22.x line, async-first API).
- **IPC mechanism:** **Redis Pub/Sub** (already needed for the inter-agent message bus per PROJECT.md decision).

**Why Redis over alternatives:**
| Option | Why not chosen |
|--------|---------------|
| HTTP REST (Node hosts an internal API, Python calls it) | Simple, but bidirectional events (Telegram → Node) need long-polling or WS-over-HTTP — uglier than pub/sub |
| gRPC | Strong typing, but adds proto build step + Python grpcio compilation pain in containers. Overkill for two services |
| Unix domain sockets | Tight coupling between containers; doesn't extend to multi-host later |
| Redis Streams | Better for replay, but the bot doesn't need replay; pub/sub is simpler |

**Redis wins because:**
1. The inter-agent bus (MVP1 scaffold, MVP2 feature) already justifies including Redis in the compose stack — adding Telegram as a second consumer is free.
2. Bidirectional naturally: Node publishes `agent.<id>.telegram.outgoing`, Python publishes `agent.<id>.telegram.incoming`. Channel names are the contract.
3. Tiny in memory (alpine image ~7 MB), zero config, fits the single-volume portability story (Redis can be configured as ephemeral — telegram messages aren't durable state).

**Node client:** `ioredis` 5.x (despite the recent "node-redis is now official" Redis announcement, ioredis is still the more mature pub/sub client and the Bull/BullMQ ecosystem depends on it — choose stability for MVP1).

**Python client:** `redis-py` (aka `redis` on PyPI) 5.x.

**Confidence:** MEDIUM — the IPC choice has tradeoffs; Redis is the strongest pick given the inter-agent bus is also coming. If that requirement goes away, HTTP would be marginally simpler.

---

### 12. PWA Tooling

**Recommendation:** `vite-plugin-pwa` 0.21.x+ (with Workbox `generateSW` strategy).

**Why:**
- Zero-config for Vite + React projects.
- Auto-injects the manifest, generates the service worker, and handles asset precaching out of the box.
- The most documented option — high probability of finding answers when something breaks on iOS Safari.

**Serwist — Considered, NOT picked:**
- A modern Workbox fork. Gaining traction, especially in Next.js circles. The Vite plugin `@serwist/vite` exists but is less battle-tested than `vite-plugin-pwa`. Re-evaluate in 12 months.

**Do NOT use:**
- Raw Workbox CLI. More config, you re-derive what vite-plugin-pwa already wraps.
- Custom service worker from scratch. Time sink that adds zero unique value for this domain.

**Phone-first config:**
- `registerType: 'autoUpdate'` so phone clients pick up new app versions automatically when reconnected.
- Precache the React bundle + Pixi assets; **runtime-cache** the WS endpoint URL is meaningless (WS bypasses SW), but cache the REST `/api/agents` snapshot so the UI renders something instantly while WS reconnects.

**Confidence:** HIGH

---

### 13. Auth (MVP1: Single Admin)

**Recommendation:**
- **Hashing:** Argon2id via `argon2` npm package (NOT bcrypt — see below).
- **Tokens:** `@fastify/jwt` 9.x for issuing access tokens.
- **Session:** httpOnly, Secure, SameSite=Lax cookie containing a short-lived (15min) JWT + a long-lived refresh token (30 days) in `auth_sessions` table.
- **Schema design:** model `users` table from day one (id, email, hashed_password, role, created_at) **even though only one row exists in MVP1**, so the post-MVP migration is purely additive.

**Why Argon2 over bcrypt:**
- Argon2id is the password-hashing winner (PHC), recommended by OWASP. bcrypt is fine in 2026 but Argon2 is strictly better against GPU attacks.
- Both are still widely supported; using Argon2 sets the right default for the multi-user evolution.

**Why NOT Better Auth in MVP1:**
- Better Auth is the right answer when you have email/password + OAuth + magic-links + 2FA. For one admin user, it's 10x more surface area than needed.
- **Schema-compatible migration path:** design the `users` and `sessions` tables to match Better Auth's expectations (camelCase column names, exported types) so swapping in Better Auth in MVP2 is a drop-in.

**Lucia Auth — Do NOT use:** Officially deprecated as of March 2025. Library is now a tutorial only.

**bcrypt vs Argon2 decision is reversible.** Both produce a string field in `users.password_hash`; the hash format identifies the algorithm.

**Confidence:** MEDIUM-HIGH — picking JWT for a self-hosted single-admin product is mainstream; "design for Better Auth later" is the small bet that keeps the door open.

---

### 14. Logging & Observability

**Recommendation:**
- **Logger:** Pino 9.x. Pretty-print in dev via `pino-pretty`. JSON in prod (single line per event, ingestion-friendly).
- **Agent log aggregation:** Each agent container streams stdout/stderr → orchestrator collects via dockerode's `container.logs({ follow: true })` → Pino re-emits with `agentId` field → also pushed onto the agent's WS channel for the live status card.
- **OpenTelemetry:** Wire `@opentelemetry/instrumentation-pino` *now* so logs carry `traceId`/`spanId` even if there's no collector. Defer the OTLP collector (Jaeger/SigNoz/Grafana) to post-MVP1.

**Why Pino:**
- 7x faster than Winston, structured JSON by default, OTel-aware via the official transport.
- Pino's `bindings` lets you tag every log line in a request scope with `agentId`, `userId`, `requestId` — perfect for filtering one agent's chaos out of the noise.

**Do NOT use:**
- Winston. Slower, less structured-first, dated API.
- `console.log` straight to stdout for production. Loses structured context, harder to grep, no levels.

**Confidence:** HIGH

---

### 15. Process Init in Agent Containers

**Recommendation:** Use Docker's built-in `--init` flag (which is `tini` under the hood) on every agent container spawn.

**Why:**
- Zero extra package to install. Just pass `HostConfig.Init: true` to `dockerode.createContainer()`.
- Solves zombie process reaping when agents spawn child processes (Claude Code may exec git, ripgrep, etc).
- Forwards SIGTERM properly so `docker stop <agent>` actually shuts down cleanly within the grace period — critical for the lifecycle UI (pause / archive).

**dumb-init — equivalent functionally, but requires an explicit `apt-get install` and ENTRYPOINT change.** `--init` requires nothing. Pick `--init`.

**Confidence:** HIGH

---

### 16. Monorepo Tooling

**Recommendation:**
- **Package manager:** pnpm 9.x (`pnpm-workspace.yaml`).
- **Task runner:** Turborepo 2.x.
- **Layout:**
  ```
  packages/
    backend/         (Fastify + dockerode)
    frontend/        (Vite + React + Pixi)
    shared/          (TS types: AgentStatus, ProviderConfig, etc — used by both)
  services/
    telegram/        (Python sidecar; Dockerfile + uv/poetry project)
  infra/
    compose/         (docker-compose.yml, Caddyfile templates)
  ```

**Why this combo:**
- pnpm's symlinked node_modules + workspace protocol gives clean local linking with zero patching.
- Turborepo caches builds across `backend` and `frontend` independently — typecheck/build/test only what changed. CI gets meaningfully faster as the repo grows.
- The Python sidecar stays out of the JS workspace graph; it's just another Docker context built from `services/telegram/Dockerfile`. Turborepo can still orchestrate it via a `task` that calls `docker build`.

**Nx — NOT picked, but reasonable for later:**
- Nx is better for true polyglot graph awareness (knowing the Python sidecar imports types from `shared`). At MVP1's repo size this is over-engineering.

**Do NOT use:**
- npm workspaces alone. No caching, slower over time.
- Lerna. Maintenance status is unclear; Turborepo covers everything Lerna did.

**Confidence:** HIGH

---

### 17. CI/CD

**Recommendation:**
- GitHub Actions on push to main.
- Build multi-arch (amd64, arm64) images for `office-backend`, `office-frontend` (Caddy-served static), and `office-telegram` (Python sidecar).
- Push to GHCR (`ghcr.io/<owner>/office-backend:<git-sha>` and `:latest`).
- `docker-compose.yml` in the repo references `ghcr.io/...` images so end users `docker compose up -d` and get prebuilt images, no build on their box.

**Why:**
- This is the standard OSS-self-hosted pattern (Plausible, Outline, Penpot, Gitea all do this). Zero surprise to ops folks.
- Multi-arch matters: laptops are arm64 (Apple Silicon, ARM ThinkPads); VPSes are usually amd64. Both must work for "any host."
- GHCR is free for public images, integrates with GitHub's existing auth, no third-party container registry account needed.

**Workflow:**
- `.github/workflows/build.yml`: on push to main → matrix build → `docker buildx build --platform linux/amd64,linux/arm64` → push.
- `.github/workflows/release.yml`: on tag (`v*`) → same build, but also tag images with semver and create GitHub Release.

**Confidence:** HIGH

---

### 18. Styling: Tailwind CSS v4

**Recommendation:** Tailwind CSS v4 (Oxide engine). Add `shadcn/ui` for the UI primitives (modals, drawers, command palette).

**Why:**
- v4's Rust-based Oxide engine compiles in <500ms even on medium projects.
- Phone-first responsive design lives natively in Tailwind's `sm:` / `md:` prefixes — matches the "mobile PWA primary" constraint.
- shadcn/ui copy-paste components fit perfectly with a one-developer project: own the code, no version-bump churn.

**Alternatives:**
- **UnoCSS** — Faster still, with the killer `icon` preset for embedding 200k+ icons as CSS. Strong contender; pick this if Tailwind's build perf is ever a real bottleneck. Smaller ecosystem.
- **Panda CSS** — Build-time CSS-in-JS with full TS typing. Beautiful, but smaller community and overkill for this scope.

**Do NOT use:**
- Pure CSS modules. You will spend a lot of time naming things.
- Emotion / Styled-Components / runtime CSS-in-JS. The state-of-CSS-in-JS in 2026 has moved decisively toward zero-runtime.

**Confidence:** HIGH

---

## Stack Patterns by Variant

**If single-host LAN deploy (no public DNS):**
- Skip Let's Encrypt; have Caddy issue a self-signed cert.
- Bind to `0.0.0.0:443` and accept browser warnings, OR use Caddy's `internal` issuer for a local CA.

**If multi-host (post-MVP1):**
- Migrate SQLite → Postgres via the Drizzle dual-dialect schema (one config change).
- Move Redis from in-compose to a dedicated host or managed service.
- Run the orchestrator behind a load balancer (sticky sessions for WebSockets).

**If running on Apple Silicon dev box:**
- All chosen images publish arm64 tags; no emulation needed.
- `better-sqlite3` ships arm64 prebuilt binaries.

---

## Version Compatibility Matrix

| Package | Pinned As | Compatible With | Note |
|---------|-----------|-----------------|------|
| Node | 22.x | better-sqlite3 ≥ 11, dockerode ≥ 5 | Use `.nvmrc` |
| Fastify | ^5 | @fastify/websocket ^11, @fastify/jwt ^9 | Major 5 is the current line |
| @fastify/websocket | ^11 | ws ^8 | ws 9 not yet supported by the plugin |
| React | 19.x | @pixi/react ^8, vite ^6 | @pixi/react v8 explicitly requires React 19 |
| PixiJS | ^8.2.6 | @pixi/react ^8 | Per official install command |
| better-sqlite3 | ^12 | drizzle-orm ≥ 0.30 (sqlite driver) | Native; rebuilds per Node major |
| drizzle-orm | ^0.40 / 1.0 | drizzle-kit (matching minor) | Schema syntax stable since ~0.30 |
| Tailwind | v4.x | shadcn/ui (latest) | v4 has different config syntax than v3 |
| @anthropic-ai/claude-agent-sdk | ^0.2 | Node ≥ 20 | Pre-1.0; pin tightly |
| openai | ^6 | Node ≥ 18 | Responses API stable |
| @google/genai | ^2 | Node ≥ 18 | Use this, not @google/generative-ai |
| ollama (npm) | latest | Ollama server ≥ 0.4 | API-versioned wire protocol |
| python-telegram-bot | ^22.7 | Python ≥ 3.10 | Async-first |
| ioredis | ^5 | Redis 7.x | Pub/Sub uses dedicated connection |
| redis-py | ^5 | Redis 7.x | Async API for sidecar |

---

## What NOT to Use — Consolidated Hit List

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| Express | Maintenance mode, weaker WS story, no built-in schema validation | Fastify 5 |
| Socket.IO (server) | Heavy, owns protocol, mobile reconnect still needs help | Raw `@fastify/websocket` + custom heartbeat |
| uWebSockets.js | Tiny maintainer base, C-like API | `@fastify/websocket` |
| node-sqlite3 (`sqlite3` package) | Async API with single-writer DB = pointless serialization | better-sqlite3 |
| Prisma | Codegen step, larger surface, slower for a tiny single-binary project | Drizzle |
| TypeORM / Sequelize | Legacy patterns, slower release cadence | Drizzle |
| Lucia Auth | DEPRECATED March 2025 | `@fastify/jwt` + Argon2 now; Better Auth at MVP2 |
| @google/generative-ai | Legacy, sunset 2025-08-31 | @google/genai |
| LiteLLM (Python) | Extra Python service, doesn't fit single-stack portability | In-house TS `Provider` interface |
| Bun (as Fastify runtime) | Not 1:1 compatible with native addons (better-sqlite3, dockerode IO) | Node 22 LTS |
| Phaser 4 | Game-loop / scene model unfit for a dashboard | PixiJS 8 |
| Redux Toolkit | Boilerplate for the value | Zustand 5 |
| Recoil | Unmaintained | Zustand 5 |
| Winston | Slower, less structured | Pino 9 |
| Nginx + Certbot | More moving parts than needed for first-run UX | Caddy 2 |
| Nginx Proxy Manager | Web GUI duplicates the office UI's purpose | Caddy 2 with labels |
| Privileged DinD in agents | Container escape = host compromise | Mount socket only in orchestrator; agents get `--read-only --cap-drop ALL` |
| Mounting Docker socket into agents | Same — full root on host | Same |
| `crypto.createCipher` (no IV) | Deprecated, insecure | `crypto.createCipheriv` with random IV |
| Storing master key in same DB as encrypted secrets | Defeats encryption | File on disk + env var fallback |
| TypeScript decorators-heavy frameworks (NestJS) for this scope | DI overhead unjustified for single-binary app | Fastify with plain handlers |
| Lerna | Maintenance uncertain | Turborepo |

---

## Roadmap Ordering Implications (for the Roadmapper)

These dependencies are **load-bearing for phase ordering:**

1. **Vault before providers.** AES-256-GCM key store + master-key bootstrap must exist before the UI can accept any API keys. → Vault is an early Phase 1 deliverable; provider integrations follow.
2. **Auth (JWT + Argon2) before any UI route.** Even with single-admin, the login flow gates everything. → Phase 0 / Phase 1.
3. **Dockerode + container spawn skeleton before any real agent.** Need a working "spawn container, stream stdout, kill container" loop before Claude/OpenAI SDK integration. → Phase 1.
4. **Redis Pub/Sub bus before Telegram sidecar.** Telegram is just a consumer of the bus → Phase 2+ (after the bus interface lands in Phase 1 scaffold).
5. **WebSocket + heartbeat before PixiJS canvas.** The Pixi desks render data that the WS stream delivers — define the wire format first, render to it second. → Phase 1 establishes wire format; Phase 2 builds the canvas.
6. **Caddy + first-run bootstrap last to integrate, first to design.** The "docker compose up" UX is the headline feature; design it in Phase 0, validate it works end-to-end at every phase boundary.
7. **PWA service worker AFTER core WS-driven UI works.** Don't ship a service worker that caches a broken app. → Late Phase 2 or Phase 3.

---

## Confidence Assessment Summary

| Area | Confidence | Why |
|------|-----------|-----|
| Backend (Fastify 5 + ws + dockerode) | HIGH | Validated across multiple 2026 ecosystem reports; matches existing constraints |
| AI SDKs (claude-agent-sdk, openai, @google/genai, ollama) | HIGH | All four are official, current, and version-confirmed |
| Database (better-sqlite3 + Drizzle) | HIGH | The dual-dialect migration path is a stated requirement and Drizzle delivers it cleanly |
| Frontend (React 19 + Vite 6 + PixiJS 8) | HIGH | Constraint already pinned this; pixi-react v8 confirmed React-19-only |
| State (Zustand 5) | HIGH | Right tool for "one big stream of state"; constraint already picked this |
| Crypto (node:crypto AES-256-GCM) | HIGH | Native, audited, zero dep |
| Proxy (Caddy 2 + caddy-docker-proxy) | HIGH | Best fit for "automatic HTTPS, one command up" |
| Telegram IPC (Redis Pub/Sub) | MEDIUM | Right answer GIVEN inter-agent bus is also coming; if bus disappears, HTTP would be simpler |
| Auth (JWT + Argon2 now, Better Auth later) | MEDIUM-HIGH | Lucia is dead; Better Auth is heavy for one user; the migration plan is sound |
| PWA (vite-plugin-pwa) | HIGH | Mature, well-documented, fits Vite stack |
| Init (Docker --init / tini) | HIGH | Built-in, zero install |
| Monorepo (pnpm + Turborepo) | HIGH | Standard 2026 combo for sub-large repos |
| CI/CD (GitHub Actions → GHCR multi-arch) | HIGH | OSS-self-hosted standard |
| Styling (Tailwind v4 + shadcn) | HIGH | Phone-first responsive + Oxide perf + shadcn ecosystem |

---

## Open Questions for Phase-Specific Research

- **WebSocket reconnection on iOS PWA backgrounding** — needs real-device testing in Phase 2. The visibility-change + exponential-backoff pattern is theoretically correct but iOS Safari edge cases are notorious.
- **Egress allow-list enforcement** — best implementation (iptables in container? Docker network with policy? proxy sidecar?) needs a security-focused mini-research in Phase 1.
- **First-run bootstrap UX** — the exact wording / format of the printed access URL and master key needs a small UX iteration; defer until Phase 0 implementation.
- **Better Auth migration cost** — verify schema column-naming compatibility before MVP2; spec it now while choices are still open.

---

## Sources

- npm: [@anthropic-ai/claude-agent-sdk](https://www.npmjs.com/package/@anthropic-ai/claude-agent-sdk) — version 0.2.139, latest
- npm: [openai](https://www.npmjs.com/package/openai) — version 6.37.0, Responses API
- npm: [@google/genai](https://www.npmjs.com/package/@google/genai) — version 2.0.1; @google/generative-ai sunset 2025-08-31
- npm: [ollama](https://www.npmjs.com/package/ollama) — official Ollama JS client
- npm: [@fastify/websocket](https://www.npmjs.com/package/@fastify/websocket) — version 11.2.0, ws@8 base
- npm: [dockerode](https://www.npmjs.com/package/dockerode) — version 5.x, ~4M weekly
- npm: [better-sqlite3](https://www.npmjs.com/package/better-sqlite3) — fastest Node SQLite driver
- GitHub: [pixi-react v8 announcement](https://pixijs.com/blog/pixi-react-v8-live) — React 19 only
- GitHub: [krallin/tini](https://github.com/krallin/tini) — built into Docker via `--init`
- Caddy docs: [Automatic HTTPS](https://caddyserver.com/docs/automatic-https) — Let's Encrypt zero-config
- GitHub: [lucaslorentz/caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy) — label-based routing
- GitHub: [lucia-auth deprecation discussion](https://github.com/lucia-auth/lucia/discussions/1707) — deprecated March 2025
- Article: [Node.js 22 vs 24 LTS — 2026](https://www.pkgpulse.com/guides/nodejs-22-vs-nodejs-24-2026)
- Article: [Drizzle vs Prisma vs Kysely — 2026](https://www.pkgpulse.com/guides/drizzle-orm-v1-vs-prisma-6-vs-kysely-2026)
- Article: [Caddy vs Traefik vs Nginx — 2026](https://ossalt.com/guides/traefik-vs-caddy-vs-nginx-reverse-proxy-self-hosting-2026)
- Article: [Pino 9 + OpenTelemetry — 2026 guide](https://dev.to/1xapi/how-to-add-structured-logging-to-nodejs-apis-with-pino-9-opentelemetry-2026-guide-3jd2)
- Article: [State management 2026 — Zustand vs Jotai vs RTK vs Signals](https://dev.to/jsgurujobs/state-management-in-2026-zustand-vs-jotai-vs-redux-toolkit-vs-signals-2gge)
- Article: [Better LiteLLM alternatives — 2026](https://www.edenai.co/post/best-alternatives-to-litellm)
- Article: [Vite PWA vs Serwist — Workbox fork landscape](https://github.com/serwist/serwist/discussions/120)
- Article: [Tailwind vs UnoCSS vs Panda — 2026](https://www.pkgpulse.com/guides/tailwind-vs-unocss-2026)
- Article: [Turborepo vs Nx vs pnpm — 2026](https://daily.dev/blog/monorepo-turborepo-vs-nx-vs-bazel-modern-development-teams/)
- Article: [python-telegram-bot v22.7 docs](https://docs.python-telegram-bot.org/) — current release
- Article: [Lucia deprecation → Better Auth migration](https://www.nodejs-security.com/blog/nodejs-authentication-migration-from-lucia-to-better-auth)

---
*Stack research for: AI Agent Office (multi-AI agent orchestration dashboard)*
*Researched: 2026-05-13*
