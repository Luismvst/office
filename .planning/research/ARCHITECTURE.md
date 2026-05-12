# Architecture Research

**Domain:** Multi-AI agent orchestration dashboard ("AI Agent Office") — self-hostable, portable, Docker-native, phone-controllable
**Researched:** 2026-05-13
**Confidence:** HIGH on container orchestration / Caddy / Claude Code session model. MEDIUM on egress filtering / Tailscale integration. LOW on a few inter-agent comms schema details that will be confirmed during MVP2 planning.

---

## 1. System Overview

The Office is a **single-host Docker Compose stack** whose master containers supervise an arbitrary set of worker (agent) containers spawned on demand. Everything writes to one named volume so the whole office is `tar`-relocatable.

```
                                INTERNET / LAN
                                      │
                  ┌───────────────────┴────────────────────┐
                  │             [reverse-proxy]            │
                  │            Caddy (TLS termination)     │
                  │         Auto-HTTPS · WebSocket proxy   │
                  └───────────────────┬────────────────────┘
                                      │
   ┌───────────────────┬──────────────┴──────────────┬──────────────────┐
   │                   │                             │                  │
┌──┴──────────┐  ┌─────┴─────────┐         ┌─────────┴──────┐   ┌───────┴───────┐
│  [web-ui]   │  │  [backend]    │         │ [telegram]     │   │   [redis]     │
│  React 19   │◄►│  Node 22      │◄───────►│  Python sidecar│◄─►│ pub/sub +     │
│  PixiJS v8  │WS│  Fastify      │ HTTP+WS │  python-tg-bot │   │ streams + KV  │
│  PWA        │  │  Orchestrator │  Redis  │                │   │               │
└─────────────┘  └─┬───────────┬─┘         └────────────────┘   └───────────────┘
                   │ Docker     │ better-sqlite3
                   │ Engine API │ (file in volume)
                   ▼            ▼
              ┌────────────────────────────────┐
              │   DOCKER ENGINE (host socket)  │
              │   /var/run/docker.sock         │
              │   Mounted ONLY into backend    │
              └─┬──────────────┬───────────────┘
                │              │
   ┌────────────▼──┐    ┌──────▼──────────────┐         ┌─────────────────────┐
   │ [agent-N]     │    │ [agent-M]           │   ...   │ [agent-K]           │
   │ Claude Code   │    │ OpenAI (Codex/GPT)  │         │ Ollama (local LLM)  │
   │ Node SDK      │    │ Node SDK            │         │ HTTP client         │
   │ --read-only   │    │ --read-only         │         │ --read-only         │
   │ egress-fenced │    │ egress-fenced       │         │ egress: ollama only │
   └───────────────┘    └─────────────────────┘         └─────────────────────┘

                   ┌─ ALL STATE ─────────────────────────────┐
                   │ Named volume: office_data               │
                   │  ├── db/office.sqlite                   │
                   │  ├── vault/master.key.enc               │
                   │  ├── caddy/data, caddy/config           │
                   │  ├── claude/                            │
                   │  │    └── projects/<encoded-cwd>/*.jsonl│
                   │  ├── workspaces/<agent-id>/<repo>/      │
                   │  ├── logs/<agent-id>/...                │
                   │  └── FIRST-RUN-CREDENTIALS.txt          │
                   └─────────────────────────────────────────┘
```

### Component Boundaries

| # | Component        | Container    | Responsibility                                                                                          | Talks To                                              | Never Touches            |
|---|------------------|--------------|---------------------------------------------------------------------------------------------------------|-------------------------------------------------------|--------------------------|
| 1 | reverse-proxy    | `caddy`      | TLS termination, HTTP/2, WS upgrade, host-name routing, optional Let's Encrypt                          | backend (HTTP+WS), web-ui (static)                    | DB, vault, agent secrets |
| 2 | web-ui           | served by Caddy or backend | React 19 + PixiJS 2D office, chat panels, secrets UI, PWA service worker, WS client       | backend over `/api`, `/ws`                            | Docker socket, vault, agents |
| 3 | backend          | `backend`    | API, auth (JWT), orchestrator, provider abstraction, vault, message-bus producer, Docker spawn/supervise | web-ui (HTTP+WS), Redis, Docker Engine, SQLite, telegram | Provider APIs directly (only via agents) |
| 4 | redis            | `redis`      | Pub/sub for live events, streams for durable agent inboxes, ephemeral state (locks, presence)            | backend, telegram, eventually agents                  | Persistent business data |
| 5 | telegram-sidecar | `telegram`   | Python python-telegram-bot polling; bridges Telegram chats ↔ a specific agent via backend HTTP+WS       | backend HTTP+WS, Redis                                | Docker socket, DB, vault |
| 6 | agent-N (workers)| `agent-<id>` | Long-running provider-specific runner (Claude Code SDK / OpenAI / Gemini / Ollama)                       | provider API (egress-fenced), backend HTTP+WS (status), Redis (optional) | Docker socket, DB, other agents' files |
| 7 | named volume     | `office_data`| Single source of truth on disk                                                                          | mounted by backend, caddy, redis (rdb), agents (scoped subpaths) | — |

**Security boundary:** Only the backend container holds the Docker socket. Agents never see the Docker socket, never see other agents' workspaces, never see the vault, never see other agents' provider keys.

---

## 2. Container & Network Architecture

### Compose Topology

```
networks:
  office-internal:     # everything except agents
    internal: false    # backend needs to reach Docker socket (host) and DNS
  office-agents:       # one-per-agent (or shared with egress allowlist)
    internal: false    # agents need outbound to provider hosts, blocked otherwise
```

Agents are placed on `office-agents` with an **iptables `DOCKER-USER` egress allowlist** (sources: [Docker iptables](https://docs.docker.com/engine/network/firewall-iptables/), [egress allowlist patterns](https://fruty.io/2021/02/15/how-to-restrict-outbound-traffic-on-a-docker-infrastructure/)):

- `api.anthropic.com:443`
- `api.openai.com:443`
- `generativelanguage.googleapis.com:443`
- `github.com:443`, `gitlab.com:443`, configured git remotes
- Optional Ollama on a `host.docker.internal` exception

Everything else egress = DROP. CVE-2025-59536 / CVE-2026-21852 mandate this.

### Per-Agent Container Layout

```
┌─────────────────────────────── agent-<uuid> ─────────────────────────────┐
│                                                                            │
│  PID 1:  runner.js  (Node 22)  — long-lived event loop                    │
│   ├─ provider client (Claude SDK | OpenAI SDK | Gemini | Ollama HTTP)     │
│   ├─ heartbeat poster ➜ POST backend://agents/<id>/heartbeat (10s)        │
│   ├─ status reporter  ➜ POST backend://agents/<id>/status (on event)      │
│   ├─ inbox consumer   ➜ HTTP long-poll OR Redis BLPOP (MVP1: long-poll)   │
│   └─ stdout streamer  ➜ tail to backend via WS (`/agents/<id>/stream`)    │
│                                                                            │
│  Mounts (read-write but namespaced):                                       │
│   /work/repo        ← bind from /data/workspaces/<id>/repo  (rw)          │
│   /home/agent/.claude  ← bind from /data/claude/<id>        (rw, JSONL)   │
│   /tmp                ← tmpfs (ephemeral)                                  │
│                                                                            │
│  Env at spawn time (NEVER written to disk inside container):               │
│   ANTHROPIC_API_KEY=...   ← decrypted by backend, injected as Env array   │
│   AGENT_ID=...            ← so it knows itself                            │
│   BACKEND_URL=http://backend:3000                                          │
│   AGENT_TOKEN=...         ← short-lived JWT, regenerated on respawn        │
│   CLAUDE_CONFIG_DIR=/home/agent/.claude                                    │
│                                                                            │
│  Hardening:                                                                │
│   --read-only            (root FS immutable)                              │
│   --cap-drop ALL --cap-add NET_BIND_SERVICE                               │
│   --security-opt no-new-privileges                                        │
│   --pids-limit 256                                                        │
│   --memory 4g  --cpus 2.0                                                 │
│   user: "10001:10001"  (non-root)                                         │
│   network: office-agents (egress-fenced)                                  │
└───────────────────────────────────────────────────────────────────────────┘
```

### Spawn & Supervise Loop (Backend → Docker Engine)

Uses `dockerode` against `/var/run/docker.sock` (sources: [dockerode README](https://github.com/apocas/dockerode), [@types/dockerode](https://www.npmjs.com/package/@types/dockerode)).

```
1. POST /api/agents  { provider, repoUrl, model, keyId, ... }
2. backend.AgentManager.create():
   a. INSERT agent row (status=provisioning, id=uuid)
   b. git clone repoUrl → /data/workspaces/<id>/repo  (inside backend container)
   c. mkdir /data/claude/<id>
   d. vault.unwrap(keyId)  → providerKey (in memory only)
   e. agentToken = signJwt({ sub: id, scope: ['heartbeat','status'] }, ttl=24h)
   f. docker.createContainer({
        Image: 'office/agent-claude:vX',
        Env:   [`ANTHROPIC_API_KEY=${providerKey}`, ... ],
        HostConfig: { Binds, ReadonlyRootfs, CapDrop, NetworkMode },
        Labels: { 'office.agent-id': id, 'office.provider': 'claude' }
      })
   g. providerKey reference → cleared from memory
   h. container.start()
   i. UPDATE agent SET status=starting, container_id=...
3. Heartbeat loop in agent (POST /agents/:id/heartbeat) is the liveness probe.
4. Backend runs an event listener on `docker.getEvents()` for die/oom/oomkill
   → re-spawn or mark crashed depending on policy.
5. Crash-loop backoff: 1s, 5s, 15s, 60s, give up.
6. On backend restart: docker.listContainers({ filter: 'office.agent-id' })
   → re-attach to running agents; respawn missing ones.
```

The backend never `exec`s into agents at runtime. All control is via HTTP/WS to the agent's runner.

---

## 3. Provider Abstraction (Plug-in Mechanism)

The whole point of MVP1 is to ship Claude + OpenAI behind an interface so Gemini and Ollama (and DeepSeek, etc.) plug in without backend changes.

### Where the Abstraction Lives

```
backend/src/providers/
├── types.ts                ← interface ProviderRunner, ProviderDescriptor
├── registry.ts             ← discover plugins from disk + config
├── claude/
│   ├── descriptor.ts       ← name, models, image, env keys, capabilities
│   ├── runner.ts           ← runs INSIDE the agent container (compiled into image)
│   └── Dockerfile          ← FROM office/agent-base; npm i @anthropic-ai/claude-agent-sdk
├── openai/
│   ├── descriptor.ts
│   ├── runner.ts
│   └── Dockerfile
├── gemini/   ← added post-MVP1, zero backend changes
└── ollama/   ← added post-MVP1, zero backend changes
```

### Interface

```typescript
// backend/src/providers/types.ts
export interface ProviderDescriptor {
  id: 'claude' | 'openai' | 'gemini' | 'ollama' | string;
  displayName: string;
  image: string;                         // docker image tag the backend will spawn
  models: { id: string; alias: string; contextWindow: number }[];
  envKeys: string[];                     // e.g. ['ANTHROPIC_API_KEY']
  supportsStreaming: boolean;
  supportsHooks: boolean;                // Claude yes, OpenAI partial, Ollama no
  estimateContextPct(usage: Usage, model: string): number;
}

export interface ProviderRunnerProtocol {
  // What the agent runner posts back to backend, normalized across providers:
  heartbeat: { agentId; ts; uptimeMs; status: 'idle'|'busy'|'error' };
  status:    { agentId; ts; model; contextPct; turns; costUsd;
               currentTask?: string; lastToolUse?: string };
  message:   { agentId; ts; role: 'user'|'assistant'|'tool'; content: string };
  result:    { agentId; ts; sessionId; tokensIn; tokensOut; costUsd };
}
```

**Adding a new provider** = drop a `descriptor.ts` + `runner.ts` + Dockerfile in `providers/<name>/`, build the image, restart backend. Registry picks it up. No core backend changes.

---

## 4. Status Reporting Pipeline (Agent → Backend → Web)

For Claude Code specifically, **three signal sources** combine into a single normalized status stream. Sources: [Claude Code statusLine docs](https://code.claude.com/docs/en/statusline), [hooks reference](https://code.claude.com/docs/en/hooks).

| Signal Source                       | Push Mechanism                          | When                          | Latency |
|-------------------------------------|-----------------------------------------|-------------------------------|---------|
| **Status-line script** (Claude)     | bash script → curl POST → backend HTTP  | Every status update from CC   | ~1s     |
| **HTTP hooks** (Claude)             | Native `{"type":"http","url":...}`      | PreToolUse, PostToolUse, Stop, SessionStart, compact_boundary | event-driven |
| **SDK result message**              | runner.js posts after each `query()`    | After every turn              | end-of-turn |

For OpenAI / Gemini / Ollama (no status-line / hooks): `runner.js` is the **only** signal source; it computes `contextPct` from `usage.input_tokens / model.contextWindow`.

### Status Flow Diagram

```
   [Claude Code process inside agent]
            │
   ┌────────┼──────────┬─────────────────┐
   │        │          │                 │
status-   hooks      result          stdout
 line    (HTTP)     message         (log tail)
script   POST       (SDK)
   │        │          │                 │
   └────────┼──────────┘                 │
            ▼                            │
   POST /agents/:id/status               │
   (auth: Bearer AGENT_TOKEN)            │
            │                            │
            ▼                            ▼
  backend.MessageBus.publish('agent.<id>.status', {...})
            │
            ├──► Redis pub/sub channel  (telegram subscribes here)
            │
            ▼
   WS broadcast to all clients subscribed to agent <id>
            │
            ▼
   PixiJS sprite color + card overlay updates
```

**Decision (HIGH confidence):** Use **status-line for steady cadence** (1Hz heartbeat-ish) **+ hooks for state transitions** (tool use, stop, compact) **+ SDK result for cost/tokens.** Don't tail logs — too noisy, parser-fragile, and stdout already streams over the runner WS for the chat UI.

---

## 5. Session Persistence & Resume Across Restarts

Claude Code stores transcripts at `${CLAUDE_CONFIG_DIR}/projects/<encoded-cwd>/<session-id>.jsonl`. (Sources: [Persist sessions docs](https://code.claude.com/docs/en/agent-sdk/session-storage), [Work with sessions](https://platform.claude.com/docs/en/agent-sdk/sessions).)

**Decision:** `CLAUDE_CONFIG_DIR=/home/agent/.claude`, bind-mounted from `/data/claude/<agent-id>/` on the host (inside the `office_data` named volume). This means:

- Backend restart → agent containers keep running (Docker doesn't kill them), session JSONLs untouched.
- Agent container restart (image upgrade, crash) → backend respawns container with same bind mount → JSONLs still there → runner.js calls `claude --resume <last-session-id>` or `ClaudeSDKClient({ resume: lastSessionId })`.
- Host restart → compose comes back up → agents restart from named volume → resume works.
- Host move (tar+scp) → restore on new host → resume works **only if cwd is identical** (encoded into the path). Pin cwd to `/work/repo` in every agent image; never `/Users/...`.

**Where the last session id lives:** Two places, dual-write:
1. `/data/claude/<agent-id>/last-session.txt` (so the agent itself can find it on respawn)
2. SQLite `agents.last_session_id` column (so the backend can query it without exec'ing)

**Gotcha (MEDIUM confidence, from [issue #53417](https://github.com/anthropics/claude-code/issues/53417)):** Some CC versions silently stop writing JSONLs after resume on upgrade. Mitigation: pin the CC version per agent image; warn on upgrade.

---

## 6. Message Bus: MVP1 Scaffold for MVP2

**The MVP1 scaffold is mandatory** — refactoring a humans-only event channel into an agent-to-agent one later is the single biggest source of rewrite pain in this domain.

### The Interface (define once, swap implementations)

```typescript
// backend/src/bus/types.ts
export interface MessageBus {
  publish<T extends Event>(topic: Topic, evt: T): Promise<void>;
  subscribe<T extends Event>(topic: Topic, handler: (evt: T) => Promise<void>): Unsub;
  // For durable per-agent inboxes (MVP2 will lean on this for handoff):
  enqueue(agentId: string, task: Task): Promise<TaskId>;
  consume(agentId: string, handler: (t: Task) => Promise<TaskResult>): Unsub;
}

export type Topic =
  | `agent.${string}.status`
  | `agent.${string}.message`
  | `agent.${string}.task`     // ← MVP1: human publishes. MVP2: agents publish too.
  | `system.audit`;

// Event schema is fixed today even though only MVP1 producers exist.
export interface Task {
  id: string;
  fromAgentId: string | 'human:<userId>';   // ← discriminator lets MVP2 expand
  toAgentId: string;
  intent: 'do' | 'review' | 'handoff' | 'ask';
  payload: { text: string; context?: TaskContext };
  ts: number;
}
```

### Implementation Strategy

**MVP1:** `RedisMessageBus` implementation backed by Redis pub/sub for `publish/subscribe` and Redis Streams (`XADD`/`XREADGROUP`) for `enqueue/consume`. (Sources: [Redis pub/sub patterns](https://blog.logrocket.com/using-redis-pub-sub-node-js/).)

**Why not in-process EventEmitter first?** Telegram sidecar is a *separate process*, so an in-process bus would force a duplicate HTTP bridge anyway. Just ship Redis on day one — it's already in the compose file and the cost is one tiny container.

**MVP2 transition:** Zero code changes in the bus interface. Agents start calling `bus.publish('agent.X.task', ...)` instead of just humans doing it. The `fromAgentId` discriminator handles routing/auth.

### Bus Layout

```
Redis topics in use:
  agent.<id>.status        pub/sub      live UI updates
  agent.<id>.message       pub/sub      chat stream
  agent.<id>.task          stream       durable inbox per agent
  system.audit             stream       audit log
  presence:agent:<id>      key TTL=15s  liveness derived from heartbeat
```

---

## 7. Telegram Sidecar Integration

Reuses `claude-telegram-agent/` Python code (`bot.py`, `agent.py`, `handlers.py`). The existing single-agent design (whitelist user, single global `SESSION`, slash commands) maps cleanly onto the office model with one change: the sidecar **does not run the Claude SDK itself**; it routes to whichever office-agent it's bound to.

### Discovery & Auth

```
┌──── telegram container ────┐                ┌──── backend container ────┐
│                            │  HTTP+WS       │                           │
│  startup:                  │ ───────────►   │  POST /api/sidecar/login  │
│   read /run/secrets/        │  { sharedSecret, hostname }                │
│     TELEGRAM_BOT_TOKEN     │ ◄───────────   │  → { sidecarToken }       │
│     SIDECAR_SHARED_SECRET  │                │                           │
│   BACKEND_URL=http://      │                │  emits 'sidecar.online'   │
│     backend:3000           │                │                           │
│                            │                │                           │
│  runtime:                  │ WS ──────────► │  /ws/sidecar              │
│   subscribes to            │                │  forwards bus events for  │
│   "agent.<bound>.message"  │                │  bound agent              │
│                            │                │                           │
│  command from Telegram:    │ HTTP POST ───► │  /api/agents/<id>/message │
│   /attach <agent-id>       │                │                           │
│   "fix the bug"            │                │                           │
└────────────────────────────┘                └───────────────────────────┘
```

- **Discovery** = compose-time DNS (`backend` hostname inside `office-internal` network). No mDNS, no service registry needed.
- **Auth** = `SIDECAR_SHARED_SECRET` env var (generated at first-run, written to `office_data/secrets/sidecar.env`, mounted into both containers). Sidecar exchanges it for a JWT on startup.
- **Bot token** is *not* in the sidecar env directly — it's pulled from the vault at startup via an HTTP call to the backend with the sidecar JWT.

### Existing Code Adaptation

| Existing                     | Office adaptation                                                |
|------------------------------|------------------------------------------------------------------|
| `AgentSession` in `agent.py` | Replaced by `RemoteAgentClient` that proxies to backend HTTP+WS |
| `handlers.py` slash commands | Mostly preserved (`/projects`, `/status`, `/new`, `/stop`) but `/cd` becomes `/attach <agent-id>` |
| `is_authorized` whitelist    | Kept — Telegram-level allowlist still applies                   |
| Single global `SESSION`      | Becomes a dict `{ chatId → agentId }` (one user, but allows /attach to switch) |

---

## 8. Secrets Vault & Key Injection

### Vault Internals

```
SQLite table: secrets
  id            TEXT PRIMARY KEY  (uuid)
  kind          TEXT              ('anthropic'|'openai'|'gemini'|'telegram-bot'|'github-pat')
  label         TEXT
  ciphertext    BLOB              AES-256-GCM(plaintext, masterKey, nonce)
  nonce         BLOB              12 bytes
  auth_tag      BLOB
  created_at    INTEGER
  last_used_at  INTEGER

Master key storage:
  Primary:  $OFFICE_MASTER_KEY env var (set by operator OR generated first-run)
  Fallback: /data/vault/master.key.enc
            encrypted-at-rest with a *hardware-derived* key:
              HKDF(machine-id || cpu-info || NIC-mac, salt=...)
            This means a stolen volume alone cannot decrypt unless attacker
            also has the host hardware fingerprint OR the env var.
```

### Injection Flow (the security-critical path)

```
┌──────────────┐    ┌────────────────────────────┐    ┌────────────────┐
│  Browser     │    │  Backend (Node)            │    │ Docker Engine  │
│              │    │                            │    │                │
│  POST        │───►│ 1. Validate JWT (admin)    │    │                │
│  /agents     │    │ 2. Load secret row by id   │    │                │
│  { keyId }   │    │ 3. masterKey ← env         │    │                │
│              │    │ 4. plaintext = AES-GCM     │    │                │
│              │    │      decrypt(ciphertext)   │    │                │
│              │    │    (kept ONLY in a single  │    │                │
│              │    │     local string, never    │    │                │
│              │    │     logged)                │    │                │
│              │    │ 5. dockerode.create({      │    │                │
│              │    │      Env: [`KEY=${plain}`] │───►│  createContainer│
│              │    │    })                      │    │  (Env -> kernel │
│              │    │ 6. plaintext = null        │    │   /proc/<pid>/  │
│              │    │ 7. start()                 │    │   environ)      │
│              │    │ 8. UPDATE secrets          │    │                │
│              │    │      SET last_used_at=now  │    │                │
└──────────────┘    └────────────────────────────┘    └────────────────┘
                                                              │
                              ┌───────────────────────────────┘
                              ▼
                   ┌─────────────────────────────────────┐
                   │  Agent container                    │
                   │  process.env.ANTHROPIC_API_KEY      │
                   │  → never written to disk            │
                   │  → not in any mount                 │
                   │  → removed from /proc on respawn    │
                   └─────────────────────────────────────┘
```

**Three things this prevents:**
- Browser leaks: key never crosses the wire to the frontend.
- Agent-filesystem leaks (CVE-2025-59536 class): no `.env`, no config file, env-only.
- Backup leaks: only the encrypted ciphertext sits in the volume.

**What this does NOT prevent:** A malicious Claude Code tool running `printenv` inside the agent. That's the residual risk; mitigated by egress allowlist (key has nowhere to be exfiltrated to).

---

## 9. Reverse Proxy & Auto-HTTPS

Caddy is bundled. (Sources: [Caddy automatic HTTPS](https://caddyserver.com/docs/automatic-https), [Caddy tls directive](https://caddyserver.com/docs/caddyfile/directives/tls), [Caddy + Tailscale](https://tailscale.com/blog/caddy).)

### Three Modes (auto-detected on first run)

| Mode               | Trigger                                  | Cert source                       | UX                                                                |
|--------------------|------------------------------------------|-----------------------------------|-------------------------------------------------------------------|
| **A. Local/LAN**   | `OFFICE_DOMAIN` unset, IP-only access    | Caddy internal CA (self-signed)   | First-run prints "trust this cert" + downloadable root cert + QR  |
| **B. Public**      | `OFFICE_DOMAIN=office.example.com` set   | Let's Encrypt HTTP-01 or DNS-01   | Caddy provisions automatically                                    |
| **C. Tailscale**   | `OFFICE_TAILSCALE=1` + tailscaled socket | Tailscale `*.ts.net` cert         | Zero-config remote access without exposing ports                  |

Mode A is the default for portability — most users won't have a domain. Caddy's internal CA can be installed into the user's trust store via a script we ship (`office trust-cert`). On a phone, the user accepts the warning once (PWA install proceeds anyway).

Mode C is the **recommended remote-access path**: no domain, no port-forward, just `tailscale up` and the office is reachable at `https://office.<tailnet>.ts.net`. We bundle a `tailscale/tailscale` sidecar option in compose, commented out by default.

### Caddyfile Sketch

```
{
    auto_https off
    local_certs   # mode A default; overridden by mode B/C below
}

# Mode A: bind to any host on local network
:443 {
    tls internal
    reverse_proxy /api/*  backend:3000
    reverse_proxy /ws/*   backend:3000
    reverse_proxy /*      web-ui:80
}

# Mode B (if OFFICE_DOMAIN set, generated dynamically):
# {$OFFICE_DOMAIN} {
#    tls admin@{$OFFICE_DOMAIN}
#    reverse_proxy /api/* backend:3000
#    ...
# }
```

---

## 10. Single-Volume State Model

```
named volume: office_data            (declared in compose as a top-level volume)
mount point:  /data inside backend, caddy, redis (rdb only), agents (scoped subpaths)

/data/
├── db/
│   └── office.sqlite               better-sqlite3 file
├── vault/
│   └── master.key.enc              fallback master key (encrypted)
├── caddy/
│   ├── data/                       Caddy data dir (Let's Encrypt certs)
│   └── config/                     Caddy config
├── redis/
│   ├── dump.rdb
│   └── appendonly.aof
├── claude/
│   └── <agent-id>/                 CLAUDE_CONFIG_DIR per agent
│       └── projects/<encoded-cwd>/<session-id>.jsonl
├── workspaces/
│   └── <agent-id>/
│       └── repo/                   git clone of the agent's repo
├── logs/
│   └── <agent-id>/
│       ├── runner.log
│       └── stdout-<date>.log       rotated
├── secrets/
│   └── sidecar.env                 sidecar shared secret (mode 0600)
└── FIRST-RUN-CREDENTIALS.txt       mode 0600, printed once
```

### `down` vs `down -v` Semantics (CRITICAL)

| Command                          | Effect                                                       |
|----------------------------------|--------------------------------------------------------------|
| `docker compose down`            | Containers removed. **Volume `office_data` survives.** All state preserved. |
| `docker compose down -v`         | **Volume deleted. All state lost.** Tantamount to factory reset. |
| `docker compose stop`            | Containers stopped (not removed). Volume untouched.          |
| `docker compose restart backend` | Single service. Volume untouched.                            |

**Decision:** Use a **named volume**, not a bind mount to a host path.

| Choice         | Pros                                                                   | Cons                                                       |
|----------------|------------------------------------------------------------------------|------------------------------------------------------------|
| **Named volume** ✅ | Portable across hosts (just tar & ship). Docker manages permissions. No host-path coupling. | `down -v` foot-gun. Cannot edit files from host without an exec. |
| Bind mount     | Easy to inspect from host. Survives even `down -v`. Familiar.          | Path is host-specific (breaks portability). Permissions mismatch between host UID and container UID. |

**Mitigation for the foot-gun:** Ship an `office` CLI wrapper that aliases `office reset` (with confirmation prompt) instead of teaching users `docker compose down -v`.

---

## 11. Backup / Restore Flow

```
┌──── BACKUP ────────────────────────────────────────────────────────────┐
│                                                                         │
│  office backup [--output backup-YYYYMMDD.tgz]                          │
│      │                                                                  │
│      1. (optional) docker compose pause backend  ← avoid mid-write     │
│         OR: redis BGSAVE + sqlite "PRAGMA wal_checkpoint(TRUNCATE)"     │
│      2. docker run --rm                                                 │
│           --volumes-from backend                                        │
│           -v $(pwd):/backup                                             │
│           alpine tar czf /backup/backup.tgz /data                       │
│      3. docker compose unpause backend                                  │
│      4. Print SHA256 of archive                                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼  scp / rsync / S3 / cold storage
                              │
┌──── RESTORE ───────────────────────────────────────────────────────────┐
│                                                                         │
│  office restore backup-YYYYMMDD.tgz                                    │
│      │                                                                  │
│      1. docker compose down  (NOT down -v)                              │
│      2. docker volume create office_data                                │
│      3. docker run --rm                                                 │
│           -v office_data:/data                                          │
│           -v $(pwd):/backup                                             │
│           alpine sh -c "cd / && tar xzf /backup/backup.tgz"             │
│      4. docker compose up -d                                            │
│      5. Health check: poll /api/health for 60s                          │
│      6. Verify: count agents in DB matches snapshot manifest            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Backup Gotchas (sources: [Docker volume backup guide](https://www.w3tutorials.net/blog/how-should-i-backup-restore-docker-named-volumes/), [volume backup gotchas](https://github.com/loomchild/volume-backup))

- **SQLite WAL files** — must checkpoint or pause writes; otherwise a partial WAL leads to corruption on restore.
- **Volume labels** — not in the tar; recreate via `docker inspect office_data -f '{{json .Labels}}' > labels.json` and include in the manifest.
- **UID/GID mismatch** — if the new host's Docker daemon runs as a different storage UID, ownership in the volume can break. Mitigation: use a fixed UID (`10001:10001`) inside agent images so files in the volume are always owned by `10001`.
- **Caddy certs** — Let's Encrypt account keys are in `/data/caddy/`; portable. Internal CA is in `/data/caddy/pki/`; portable, but clients must re-trust if the host changes (mode A only).
- **Running agents at backup time** — pause backend first to quiesce; agent JSONLs are append-only so worst case is the in-flight turn is lost, not the file.

---

## 12. First-Run Auto-Bootstrap

```
┌─ docker compose up -d (first time) ────────────────────────────────────┐
│                                                                         │
│  backend entrypoint:                                                    │
│    1. Detect first-run: ls /data → empty?                              │
│    2. mkdir /data/{db,vault,workspaces,claude,logs,secrets}             │
│    3. masterKey ← randomBytes(32)                                       │
│       if $OFFICE_MASTER_KEY set: use that                              │
│       else: hwKey ← deriveFromHardware()                               │
│              write /data/vault/master.key.enc (chmod 0600)             │
│    4. adminPassword ← randomWords(4)  (e.g. "tiger-cloud-orbit-jazz")  │
│       bcrypt → INSERT INTO users                                       │
│    5. sidecarSecret ← randomBytes(32)                                   │
│       write /data/secrets/sidecar.env (chmod 0600)                     │
│    6. Detect access URL:                                                │
│         if OFFICE_DOMAIN set → https://OFFICE_DOMAIN                   │
│         elif TAILSCALE → https://<host>.<tailnet>.ts.net               │
│         else → https://<lan-ip>                                        │
│    7. Render QR code for the URL using qrcode-terminal                  │
│    8. Write /data/FIRST-RUN-CREDENTIALS.txt (chmod 0600):              │
│         Access URL: ...                                                 │
│         Admin user: admin                                               │
│         Admin password: tiger-cloud-orbit-jazz                          │
│         Master key (if generated): export OFFICE_MASTER_KEY=...        │
│    9. PRINT all of the above to stdout (one time) so `docker logs`     │
│       captures it.                                                      │
│   10. Continue normal boot.                                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 13. PWA + WebSocket Reliability

Phone backgrounding kills WS connections. The web client needs resume logic.

```
client                                backend
  │                                      │
  │  WS open /ws?lastEventId=<id>        │
  │ ───────────────────────────────────► │
  │                                      │  Look up agent.last_event_id_<conn>
  │                                      │  Redis XREAD from lastEventId onward
  │  catch-up: e1, e2, e3, ...           │
  │ ◄─────────────────────────────────── │
  │  live: e4, e5, ...                   │
  │ ◄─────────────────────────────────── │
  │  every event carries `id`            │
  │                                      │
  ┌─────── BACKGROUND ─────────┐         │
  │  client persists lastId    │         │
  │  in IndexedDB              │         │
  └────────────────────────────┘         │
  │  WS close (idle / OS kill)           │
  │                                      │
  │  WS reopen with lastEventId          │
  │ ───────────────────────────────────► │
  │  replay gap + resume live            │
```

Backed by Redis Streams (`XADD` with auto-id, `XREAD BLOCK`). Streams are trimmed to last 1000 events per agent (`MAXLEN ~ 1000`) — enough for reconnect, bounded memory.

---

## 14. Project Structure

```
office/
├── docker-compose.yml
├── docker-compose.tailscale.yml      # optional overlay
├── Caddyfile                         # baseline; backend can write a Caddyfile.gen
├── .env.example                      # OFFICE_DOMAIN, OFFICE_MASTER_KEY, ...
├── packages/
│   ├── backend/
│   │   ├── src/
│   │   │   ├── server.ts             # Fastify bootstrap
│   │   │   ├── auth/                 # bcrypt + JWT
│   │   │   ├── vault/                # AES-GCM, key derivation
│   │   │   ├── orchestrator/         # AgentManager, dockerode wrapper, supervise loop
│   │   │   ├── providers/            # ← plug-in dir (see §3)
│   │   │   ├── bus/                  # MessageBus interface + Redis impl
│   │   │   ├── ws/                   # WS gateway, resume logic
│   │   │   ├── routes/               # REST endpoints
│   │   │   ├── db/                   # better-sqlite3 + migrations
│   │   │   └── bootstrap/            # first-run detection + setup
│   │   ├── docker/
│   │   │   └── agent-base/Dockerfile # shared base for all provider agents
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── web/
│   │   ├── src/
│   │   │   ├── office/               # PixiJS scene
│   │   │   ├── chat/                 # chat panel
│   │   │   ├── admin/                # secrets UI, agent CRUD
│   │   │   ├── ws/                   # WS client with resume
│   │   │   └── pwa/                  # service worker, manifest
│   │   └── vite.config.ts
│   ├── telegram/
│   │   ├── bot.py                    # reused (lightly adapted)
│   │   ├── agent_remote.py           # NEW: replaces local AgentSession
│   │   ├── handlers.py               # reused
│   │   └── Dockerfile
│   └── cli/
│       └── office                    # bash/Node wrapper: backup, restore, reset, ...
├── images/
│   ├── agent-claude/Dockerfile       # FROM agent-base; installs claude-agent-sdk
│   ├── agent-openai/Dockerfile
│   ├── agent-gemini/Dockerfile       # (post-MVP1 ready)
│   └── agent-ollama/Dockerfile       # (post-MVP1 ready)
└── docs/
    ├── ARCHITECTURE.md
    └── OPERATIONS.md                 # backup, restore, trust-cert, ...
```

### Rationale

- **Monorepo** keeps the contract between backend ↔ web ↔ telegram in one PR.
- **`providers/`** as a folder, not a separate repo, so plug-in discovery is a filesystem read at boot — adding Gemini = `git add packages/backend/src/providers/gemini/` + new image, no version bump.
- **`images/`** separate from backend source because they ship as Docker images, not npm packages.

---

## 15. Auto-Update Strategy

Three options surveyed:

| Option            | Pros                                              | Cons                                                       | Verdict        |
|-------------------|---------------------------------------------------|------------------------------------------------------------|----------------|
| Watchtower        | Drop-in, polls registry                           | Auto-pulls breaking changes; no migration awareness        | ❌ Too risky    |
| `office update` CLI | Explicit; can run migrations between pull & up   | Manual                                                     | ✅ Default      |
| Webhook self-update | Push-driven from CI                              | Complex; harder to debug; not portable                     | ❌ Post-MVP    |

**Decision:** Ship an `office update` CLI that runs `docker compose pull && docker compose up -d` after backing up the volume automatically. No Watchtower.

---

## 16. Build-Order Implications → Suggested Phases

The roadmap can use these as a starting point. Each phase ends in a demonstrable artifact.

### Phase 1 — Foundations & First Run
**Builds:** monorepo skeleton, compose stack with backend + caddy + redis + sqlite, first-run bootstrap, basic admin login, named-volume layout, `office backup` / `office restore` CLI.
**Demo:** `docker compose up` on a clean VM → access via printed URL, log in, see empty dashboard. Tar volume, scp to second VM, untar, compose up, same login works.
**Why first:** Portability promise is THE core value. If this isn't solid, nothing else matters.
**Dependencies:** none.

### Phase 2 — Vault + Provider Abstraction
**Builds:** AES-GCM vault, secrets UI, `ProviderDescriptor` registry, hardcoded Claude descriptor (no spawning yet).
**Demo:** Add an Anthropic key in the UI, see it appear in the DB encrypted, view list of available providers/models.
**Why second:** Every later phase needs to inject keys into containers. Get the security model right before scaling out.
**Dependencies:** Phase 1.

### Phase 3 — Single-Agent Spawn (Claude only)
**Builds:** `AgentManager`, `dockerode` integration, `agent-claude` image, runner.js with status-line + hooks reporting, agent lifecycle (create/start/pause/resume/archive/delete), egress allowlist iptables rules, `CLAUDE_CONFIG_DIR` persistence, session-resume across restart.
**Demo:** Create one agent against a repo. Chat with it via a minimal REST endpoint. Watch context %, cost, model in API responses. Restart the container, resume session.
**Why third:** Riskiest piece technically. De-risk the whole "spawn-and-supervise" loop with one provider before adding the second.
**Dependencies:** Phase 2.

### Phase 4 — WebSocket Gateway + Message Bus + 2D Frontend
**Builds:** Redis-backed `MessageBus` (full interface, even though only humans publish), WS gateway with resume (lastEventId), PixiJS 2D office, chat panel, color-coded sprite status, mobile PWA shell + service worker.
**Demo:** Phone scans QR, installs PWA, sees a desk with a colored avatar, clicks the desk, chats with the agent live with status updates streaming in.
**Why fourth:** This is the "wow" moment of the product — but only meaningful once one agent works end-to-end (Phase 3).
**Dependencies:** Phase 3.

### Phase 5 — Second Provider + Plug-in Validation
**Builds:** `agent-openai` image, OpenAI provider descriptor, runner.js with normalized status reporting (no hooks, no status-line — `usage`-based contextPct).
**Demo:** Add an OpenAI key, spawn a Codex/GPT agent next to a Claude agent, both visible in the office.
**Why fifth:** This is where the provider abstraction is *proven*. If adding the second provider required backend changes, the abstraction failed and must be redesigned before MVP ships. Catching this in Phase 5 (not Phase 7) saves a rewrite.
**Dependencies:** Phase 4.

### Phase 6 — Telegram Sidecar
**Builds:** Containerize `claude-telegram-agent`, `RemoteAgentClient` replacing `AgentSession`, `/attach` command, sidecar JWT auth flow, vault-backed bot token retrieval.
**Demo:** From a phone with no internet at office (just cellular), Telegram → /attach <agent-id> → "fix the bug in foo.ts" → agent works, replies stream back.
**Why sixth:** Optional but signature feature. Independent of the web UI, so can be done in parallel with later polish.
**Dependencies:** Phase 4 (needs the bus and agent runtime).

### Phase 7 — Production Hardening: Caddy Modes + Backup Robustness + Auto-Update
**Builds:** Mode B (Let's Encrypt) and Mode C (Tailscale) Caddy configs, `office update` CLI with pre-update backup, SQLite WAL checkpoint on backup, audit log via `system.audit` stream.
**Demo:** Bring office up on a public VPS with a real domain; `office update` to a newer version; `office backup` mid-conversation; verify zero data loss.
**Why seventh:** Hardening once the happy path is end-to-end verified.
**Dependencies:** Phase 1 (for backup), Phase 6 (so all features under test).

### Phase 8 (Optional, MVP1.5) — Inter-Agent Comms Activation
**Builds:** Flip the `fromAgentId` guard in the bus to allow agent→agent publishes; add an "assign to" action in the chat UI; agent runner.js gains `bus.publish('agent.X.task', ...)` tool.
**Demo:** Agent A finishes a refactor, hands off to Agent B saying "please review", B picks it up from its inbox.
**Why optional:** This is MVP2. But because the scaffold lives in Phase 4, the actual activation is a 1-2 day phase, not a rewrite.
**Dependencies:** Phase 5 (need ≥2 agents to make handoff meaningful).

---

## 17. Data Flows (Summary)

### Spawn flow

```
User (browser) ─POST /api/agents─► Backend ─git clone─► /data/workspaces/<id>
                                       │
                                       ├─ vault.decrypt(keyId) [in-memory]
                                       │
                                       └─ dockerode.create({Env:[KEY=...]}) ─► Docker Engine
                                                                                    │
                                                                                    ▼
                                                                              Agent container
                                                                              (runner.js boots,
                                                                               registers heartbeat,
                                                                               resumes session if any)
```

### Live status flow

```
Claude in agent ─status-line/hooks─► runner.js ─POST /agents/<id>/status─► Backend
                                                                              │
                                                                              ├─► Redis pub:agent.<id>.status
                                                                              │              │
                                                                              │              └─► Telegram sidecar (subscribed)
                                                                              │
                                                                              └─► WS gateway ─► All web clients on agent <id>
                                                                                                       │
                                                                                                       ▼
                                                                                                  PixiJS sprite redraw
```

### Backup flow

```
Operator ─office backup─► ephemeral alpine container ─tar /data─► host filesystem
                                       │                                  │
                                       └─ before: PRAGMA WAL checkpoint    └─ scp to remote
                                            redis BGSAVE
```

---

## 18. Anti-Patterns

### Anti-Pattern 1: Mounting the Docker socket into agents
**What people do:** Give agents `/var/run/docker.sock` so they can spawn sub-agents.
**Why it's wrong:** Container escape → host root. With CVE-2025-59536 class threats, an agent processing a malicious repo can take over the host.
**Do instead:** Only the backend gets the socket. Agents request sub-agents via the backend API.

### Anti-Pattern 2: Storing API keys in agent volumes
**What people do:** Write `.env` files into the agent's bind-mounted workspace.
**Why it's wrong:** Keys end up in backups, in `git status`, in the volume the user might share. Any tool inside the agent can `cat .env`.
**Do instead:** Env-var-only injection at spawn time. Vault decrypts in backend RAM.

### Anti-Pattern 3: In-process EventEmitter for "we'll add Redis later"
**What people do:** Use Node EventEmitter for events, plan to "abstract" later.
**Why it's wrong:** Telegram sidecar is already a separate process; you'll need cross-process IPC on day one. The "later" never comes without a rewrite.
**Do instead:** Redis on day one behind a `MessageBus` interface.

### Anti-Pattern 4: Bind-mounting host paths instead of named volumes
**What people do:** `volumes: ./data:/data` because it's easier to inspect from the host.
**Why it's wrong:** Breaks portability (host-specific paths), UID/GID mismatch, can't `tar` cleanly across hosts.
**Do instead:** Named volume. Provide an `office shell` command for inspection.

### Anti-Pattern 5: Tailing agent log files for status
**What people do:** Parse stdout for "[INFO] context: 45%".
**Why it's wrong:** Brittle across CC version upgrades; high latency; mixes display logs with state.
**Do instead:** Status-line script + HTTP hooks (Claude); `usage` field from SDK result (all providers).

### Anti-Pattern 6: Letting Caddy provision Let's Encrypt before the user has DNS
**What people do:** Default the compose to Let's Encrypt mode.
**Why it's wrong:** Rate-limit lockout from failed ACME challenges. Bad first-run UX.
**Do instead:** Default to mode A (internal CA). Mode B requires explicit `OFFICE_DOMAIN`.

---

## 19. Scaling Considerations

| Scale                | Adjustments                                                                                       |
|----------------------|---------------------------------------------------------------------------------------------------|
| 1 user, 1-5 agents   | MVP shape exactly as above. SQLite, single backend, single Redis.                                |
| 1 user, 5-20 agents  | Add per-agent CPU/memory limits in compose. Pre-pull all provider images at boot. No code change.|
| 1 user, 20-50 agents | Move to PostgreSQL (DB abstraction was preserved for this). Consider Docker resource quotas.     |
| Multi-user (post-MVP)| Per-user `CLAUDE_CONFIG_DIR`, RBAC on bus topics, separate vault namespaces. Significant rework. |

**First bottleneck:** SQLite write contention if status-line is pushing at 1Hz × N agents. Mitigation: batch status writes via a 200ms debouncer in the backend; only the Redis stream gets every event, the DB stores last-known-state.

**Second bottleneck:** Docker Engine API rate. Listing 50+ containers is slow. Mitigation: cache `docker.listContainers()` results in Redis for 5s; rely on `docker.getEvents()` push instead of polling.

---

## 20. Integration Points

### External Services

| Service              | Pattern                                            | Gotchas                                                                  |
|----------------------|----------------------------------------------------|--------------------------------------------------------------------------|
| Anthropic API        | `@anthropic-ai/claude-agent-sdk` inside agent      | Rate limits per key; CC version pinning per image                        |
| OpenAI API           | `openai` SDK Responses API inside agent            | Tokenizer per model for contextPct; `usage` shape differs between models |
| Telegram API         | python-telegram-bot polling                        | Bot token rotation = restart sidecar; 4096 msg limit (existing code handles)|
| Tailscale            | `tailscale/tailscale` sidecar (optional)           | Requires `--privileged` or specific caps; documented as optional         |
| Let's Encrypt        | Caddy ACME (built-in)                              | Rate limits (5 certs/week/domain); use staging during testing            |
| Docker Engine        | `dockerode` over Unix socket                       | Socket path differs on Windows hosts (named pipe); MVP1 Linux/macOS only |

### Internal Boundaries

| Boundary                     | Communication                          | Notes                                                  |
|------------------------------|----------------------------------------|--------------------------------------------------------|
| backend ↔ web-ui             | HTTPS REST + WSS                       | Both behind Caddy on same origin → no CORS pain        |
| backend ↔ agent (control)    | HTTPS-on-internal-network REST         | Mutual auth: backend has admin JWT, agent has AGENT_TOKEN |
| backend ↔ agent (stream)     | WS                                     | Bi-directional for stdout, hooks responses             |
| backend ↔ redis              | TCP, password-auth, network-internal   | Password lives in compose env, not in volume           |
| backend ↔ telegram           | HTTPS-on-internal-network REST + WSS   | Shared-secret JWT exchange                             |
| agent ↔ agent                | **Not allowed in MVP1.** All routed via backend. | MVP2 may relax to direct Redis pub if needed         |

---

## 21. Quality Gate Self-Check

- [x] Component boundaries clearly defined (§1, §2 tables)
- [x] Data flow explicit with directionality (§17 diagrams)
- [x] Build order suggests 5-8 phases with rationale (§16: 7 mandatory + 1 optional)
- [x] ASCII diagrams included (overall, per-agent, status flow, secrets injection, backup, first-run, PWA reconnect)
- [x] Security boundaries called out (§2 "Security boundary" callout, §8 full injection flow, §18 anti-patterns)
- [x] "MVP1 scaffold for MVP2 inter-agent comms" pattern is concretely specified (§6 with full interface, §16 Phase 8)

---

## Sources

- [Customize your status line — Claude Code Docs](https://code.claude.com/docs/en/statusline)
- [Hooks reference — Claude Code Docs](https://code.claude.com/docs/en/hooks)
- [Persist sessions to external storage — Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/session-storage)
- [Work with sessions — Claude API Docs](https://platform.claude.com/docs/en/agent-sdk/sessions)
- [dockerode GitHub](https://github.com/apocas/dockerode)
- [@types/dockerode](https://www.npmjs.com/package/@types/dockerode)
- [Automatic HTTPS — Caddy Documentation](https://caddyserver.com/docs/automatic-https)
- [tls Caddyfile directive](https://caddyserver.com/docs/caddyfile/directives/tls)
- [Use Caddy to Manage Tailscale HTTPS Certificates](https://tailscale.com/blog/caddy)
- [caddy-tailscale GitHub](https://github.com/tailscale/caddy-tailscale)
- [Docker iptables docs](https://docs.docker.com/engine/network/firewall-iptables/)
- [Packet filtering and firewalls — Docker Docs](https://docs.docker.com/engine/network/packet-filtering-firewalls/)
- [How to Restrict Outbound Traffic on a Docker Infrastructure](https://fruty.io/2021/02/15/how-to-restrict-outbound-traffic-on-a-docker-infrastructure/)
- [How to Backup and Restore Docker Named Volumes](https://www.w3tutorials.net/blog/how-should-i-backup-restore-docker-named-volumes/)
- [loomchild/volume-backup GitHub](https://github.com/loomchild/volume-backup)
- [Docker Volumes Documentation](https://docs.docker.com/engine/storage/volumes/)
- [Using Redis pub/sub with Node.js — LogRocket](https://blog.logrocket.com/using-redis-pub-sub-node-js/)
- [Redis Pub/Sub for microservices (Node.js)](https://medium.com/@alextkd/redis-pub-sub-communication-between-microservices-in-node-js-ff91fb308996)
- [redis-eventemitter GitHub](https://github.com/freeall/redis-eventemitter)
- [CVE-2025-59536 / CVE-2026-21852 context](https://www.anthropic.com/security) (from project context, not freshly verified)

---
*Architecture research for: multi-AI agent orchestration dashboard*
*Researched: 2026-05-13*
