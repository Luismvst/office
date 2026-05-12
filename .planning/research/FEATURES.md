# Feature Research — AI Agent Office

**Domain:** Self-hostable multi-AI-agent management dashboard (phone-first, 2D office visualization, single admin)
**Researched:** 2026-05-13
**Confidence:** HIGH for what competitors ship; MEDIUM for what the user will tolerate as anti-features (no direct user research yet)

---

## Reference Projects Surveyed

| Project | Signal | Role for Us |
|---------|--------|-------------|
| `paulrobello/claude-office` | PixiJS + FastAPI, pixel-art office, real-time WebSocket | Direct visualization reference |
| `harishkotra/agent-office` | Phaser + Colyseus, agents walk/talk/hire interns, persistent memory (SQLite + embeddings), TTS/STT, multi-floor, plugin system | Aspirational ceiling — too heavy for v1, but a roadmap menu |
| `hesamsheikh/octogent` | Orchestration dashboard, "tentacles" job containers (CONTEXT.md + todo.md), inter-agent messaging, git-worktree isolation, local API + web UI | Closest functional sibling; validates worktree-per-agent and messaging scaffold |
| `smtg-ai/claude-squad` (5.6K stars) | tmux + git-worktree TUI, supports Claude/Codex/OpenCode/Amp, isolated branches, persistent sessions across TUI restarts | Validates multi-provider + worktree pattern; we replace TUI with web/PWA |
| `hoangsonww/Claude-Code-Agent-Monitor` | Express + WebSocket + React + SQLite, hook-driven event capture, Kanban board, live char-by-char streaming, JSON export | Reference for the live-status pipeline (hooks → WS → React) |
| `jakemor/kanna` | Beautiful Web UI for Claude Code + Codex, per-provider model selection, project-first sidebar with idle/running/waiting/failed indicators, embedded terminal (Bun PTY), branch switch + PR-from-app | Closest UX-quality benchmark; competitor on the "pretty single-user web UI" axis |
| Anthropic Agent View (May 2026, official) | TUI-only dashboard, rows = sessions with state (working/waiting/completed/failed/idle/stopped), `claude --bg`, peek with spacebar, attach with Enter | Confirms feature parity table stakes — every coding agent dashboard now has this |
| Cursor 2.0 + Mission Control | Up to 8 parallel agents, Background Agents in isolated VMs, agent-centric IDE shell | Competitor, but local-only and IDE-coupled |
| Windsurf Cascade Wave 13 + Arena | Parallel Cascade sessions with terminal profiles, Arena (multi-model race on same task, Jan 2026) | Hints at a "model bake-off" differentiator |
| Devin (Cognition) | Web dashboard, browser favicon status dot, sessions API filtering, PWA install on desktop+mobile, Slack-like UX, GitHub/Jira integration | Validates phone-first + PWA + favicon status dot pattern |
| OpenHands (All-Hands) | Open SDK + CLI + GUI, embedded shell/browser/editor/planner, self-hosted via Helm chart, Planning Mode beta, "thousands of parallel runs" | Heavy enterprise self-host; we are the lightweight single-admin counterpart |
| Aider | Terminal-only, `/architect` + `/ask` modes, repo-map context injection, auto-lint + auto-test loop | Validates `/ask` + `/architect` UX, repo-map mental model |
| Manus | Live "Computer" stream (browser + VS Code view), multi-model routing (Claude/Qwen), persistent cloud Ubuntu, scheduling/triggers | Validates "watch the agent work" visualization, persistent compute angle |
| LangGraph Studio / AutoGen Studio / CrewAI | Graph visualization + time-travel debugging (LG), no-code conversation builder (AutoGen), role assignment + enterprise observability (CrewAI Mar 2026) | Framework-level dashboards, not product-level — we are the product-level UX |
| `claude-telegram-agent` (existing asset) | Python `python-telegram-bot`, whitelist auth (single user ID), `/projects`, `/cd`, `/new`, `/stop`, `/status`, NSSM service installer, daily-rotated logs | Containerize as-is; the bot's command vocab becomes our Telegram MVP feature list |

---

## Feature Landscape

Legend for tags: **[TS]** = table stakes · **[D]** = differentiator · **[A]** = anti-feature for v1
Complexity: **S** (≤2 days) · **M** (3–10 days) · **L** (>10 days)

### 1. Agent Lifecycle

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Create agent from remote git URL (clone on create) | **TS** | M | Every reference (Squad, Octogent, Kanna) supports this; users will reject if missing. Depends on Docker workspace + secrets vault. |
| Create agent from local workspace path | **TS** | S | Trivial bind-mount; users with monorepos demand this. |
| Create agent from scratch dir (no repo) | **TS** | S | "Help me prototype" use case; no clone, just an empty volume. |
| Start / pause / resume | **TS** | M | Squad, Octogent, Agent View all support it. "Pause" = stop container but keep state; "resume" = restart with the same `CLAUDE_CONFIG_DIR`. |
| Archive (soft delete, retain workspace + transcript) | **TS** | S | Devin and Octogent both retain history. Users fear losing the conversation. |
| Hard delete (with confirmation) | **TS** | S | GDPR-ish reflex; must be explicit + double-confirm. |
| Clone agent (copy config, fresh workspace) | **D** | S | Octogent's "tentacle spawn" pattern. Lets the user iterate without losing the parent's transcript. Differentiator because most tools don't. |
| Templated agents (pre-canned roles: "frontend dev", "DBA") | **A** for v1 | L | Marketplace territory — explicitly out of scope in PROJECT.md. Don't ship until v2. Warning: leads to template sprawl and prompt-decay debugging. |
| Auto-restart on crash | **TS** | S | Container `restart: unless-stopped` covers this. Users assume "set and forget." |
| Bulk operations (pause-all, archive-all) | **D** | S | Power-user QoL the office metaphor invites ("end of day, send everyone home"). |

### 2. Agent Visualization (the 2D Office)

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Top-down 2D canvas (PixiJS) with desk grid | **TS for this product** | L | Core identity. Without it, the product is "yet another Kanna fork." |
| Per-agent sprite at a desk | **TS for this product** | M | Anchor of the metaphor. |
| Status color (green idle / yellow working / red error or context>70%) | **TS** | S | Already in PROJECT.md; matches Agent View states and Devin's favicon dot. |
| Card overlay per desk (project, model, ctx%, turns, cost, current task) | **TS** | M | Information density a glance gives — table-stakes for the metaphor. |
| Click desk → open chat panel | **TS** | S | The interaction primitive. |
| Sprite animations during tool calls (typing, walking to printer, etc.) | **D** | M | claude-office's "printer station," agent-office's walking. Wins the demo video. **Caution:** must degrade gracefully on low-end phones. |
| Free-form layout editor (drag desks) | **D** | M | agent-office has it; matches the "office layout" mental model. |
| Room-based grouping (frontend room, backend room) | **D** | M | Future-proofs the floor-plan UX; aligns with how owners mentally group work. |
| Multi-floor / multi-office (zoom out) | **A** for v1 | L | agent-office ships this; over-built for a single admin with <20 agents. Defer until users actually run that many agents. |
| Multiplayer view (other humans see same office) | **A** for v1 | L | agent-office uses Colyseus for this; PROJECT.md explicitly scopes to single admin. Don't pay the multiplayer-state-sync cost. |
| Voice-bubble emotes ("I'm done", "I need input") | **D** | S | agent-office uses these; cheap to add, huge personality boost. |
| Day/night mode tied to system clock | **D** | S | Office metaphor flourish — desks dim at night. |
| 3D / isometric view | **A** | L | Burns budget on rendering, doesn't help comprehension. 2D top-down is the spec. |

### 3. Live Monitoring

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Context window % | **TS** | M | All references show this. Use status-line JSON `context_window.used_percentage` (Claude SDK) or compute from `usage.input_tokens` / model window. |
| Total input/output token count (cumulative) | **TS** | S | Trivial accumulation off SDK result messages. |
| Cumulative cost (USD) | **TS** | S | `total_cost_usd` is in `SDKResultMessage`. Devin, Squad, Kanna all show it. |
| Current model | **TS** | S | Even more useful when fallback routing kicks in. |
| Current task / last user message preview | **TS** | S | "What is it doing right now?" — primary user question. |
| Turn count | **TS** | S | Cheap counter; users use it as a "is this loop stuck?" signal. |
| Last activity timestamp | **TS** | S | Anthropic Agent View shows this; users sort by it. |
| Error state surface (last error, retry counter) | **TS** | M | Without this, "red light" is unactionable. |
| Live char-by-char streaming of agent output | **D** | M | hoangsonww does it with `--include-partial-messages` + RAF smoothing. High wow factor for a phone screen. **Caution:** WebSocket back-pressure on slow networks. |
| Auto-compaction event banner | **D** | S | SDK emits `compact_boundary` system message; surface it so users understand context resets. |
| Per-tool-call timeline (Bash, Edit, Read, etc.) | **D** | M | Kanban-style or vertical timeline (claude-code-agent-monitor pattern). Helps debugging. |
| Subagent / Task tool tree | **D** | M | When Claude spawns subagents (Task tool), show them as nested rows. Anthropic Agent View does this. |
| Animated "thinking" cursor while turn is in flight | **TS** | S | Affordance users assume. |

### 4. Provider Management

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Multiple providers (Claude Code SDK + OpenAI Responses) | **TS for this product** | L | Kanna, Squad, Manus all support it. Single-provider products lose the bake-off conversation. |
| Pluggable provider interface (Gemini, Ollama, DeepSeek added without core changes) | **D** | M | PROJECT.md calls it out. Most competitors hard-code 1–2 providers. |
| Model switching mid-session | **D** | M | Kanna does it on the same message input. Powerful for "let me try Opus on this one". |
| Fallback chain (Opus → Sonnet → Haiku on overload/rate-limit) | **D** | M | Resilience differentiator; Anthropic's own SDK has overload errors regularly. |
| Cost-aware routing (cheap model for trivial turns, escalate on complex) | **A** for v1 | L | Sounds great, classification is fragile, users get burned when it picks wrong. Keep model choice explicit. |
| Per-agent provider/model lock | **TS** | S | "This agent is always GPT-5"; aligned with persona. |
| Local model support (Ollama, LM Studio) | **D** | M | agent-office uses Ollama; differentiates from cloud-only Devin/Cursor. Privacy story matters. |
| Arena mode (run same task on N models, compare) | **A** for v1 | L | Windsurf shipped this Jan 2026; super flashy but tangential to the "office" thesis. Maybe v2. |

### 5. Secrets Management

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Vault UI to add/rotate/revoke provider keys | **TS** | M | PROJECT.md mandates this. Without it, the "portable office" story falls apart on host migration. |
| Encrypted at rest (AES-256-GCM, master key from env or KMS) | **TS** | M | PROJECT.md mandates. Use libsodium/sealed-box pattern. |
| Per-agent key assignment | **TS** | S | So an agent that talks to GPT can't also see your Claude key. Real attack-surface reduction. |
| Keys never sent to the browser | **TS** | S | Standard practice; show masked previews only. |
| Telegram bot tokens in same vault | **TS** | S | Single source of truth. |
| Key-usage audit log (which agent used which key, when, succeeded/failed) | **D** | M | Lifts product above "encrypted env file" — real ops hygiene. Required if/when multi-user lands. |
| Scoped keys (e.g., "this OpenAI key only for project X") | **D** | M | Nice when you have separate billing accounts. |
| Auto-rotate on schedule | **A** for v1 | M | Most key APIs don't support programmatic rotation; users will misconfigure and lock themselves out. Defer. |
| External KMS integration (AWS KMS, Vault) | **A** for v1 | L | Enterprise-grade, opposite of single-admin home-lab. Document the env-var master-key path instead. |

### 6. Workspace Management

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Clone from git URL (HTTPS + SSH) | **TS** | M | Squad, Octogent, Kanna all do this. SSH key handling is the gotcha. |
| Browse repo tree in UI | **TS** | M | Kanna ships it; without it, "what is the agent looking at?" is unanswerable on mobile. |
| Show live diff of agent's pending changes | **TS** | M | Squad's preview pane, Kanna's branch view. Approving without seeing the diff is unacceptable. |
| Persist edits across container restarts | **TS** | S | Bind-mounted volume per agent; non-negotiable. |
| Git operations: branch, commit, push from UI | **TS** | M | Kanna ships it. On mobile, terminal access is painful. |
| Open PR from UI | **D** | M | Kanna and Devin do it. Closes the loop without leaving the app. |
| Worktree-per-agent (one repo, many agent branches, no conflicts) | **D** | M | Squad and Octogent's killer feature. Highly differentiating vs. naive "clone-per-agent" disk waste. |
| Pull/sync remote in UI | **TS** | S | Kanna ships sidebar sync. |
| In-browser code editor (Monaco) | **A** for v1 | L | OpenHands ships it; reproducing it well is L-effort and competes with VS Code. The agent edits files — the user reviews diffs. |
| Embedded terminal (PTY) | **A** for v1 | L | Kanna ships it with Bun PTY (macOS/Linux only); cross-platform PTY in browser is hard, and the use case ("just let me run a command") is better served by a one-shot exec dialog. |
| One-shot "run command in agent's container" dialog | **D** | S | Lighter-weight alternative to embedded terminal. Pipe output back to chat. |
| Workspace size / disk-usage indicator | **D** | S | Single owner on a VPS notices when one agent's `node_modules` eats 30GB. |

### 7. Chat / Task Interface

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Message history (persisted, scrollback) | **TS** | M | SDK writes to `~/.claude/projects/<encoded>/<session>.jsonl`; tail and render. |
| File attachment upload (images, PDFs, code) | **TS** | M | Devin, Manus, Kanna support. Mobile users especially need this (screenshots). |
| Code diff view inside chat | **TS** | M | Kanna's hydrated tool calls. Without it, mobile users can't review. |
| Per-tool approval gating (acceptEdits / bypassPermissions / plan-only) | **TS** | M | Aider's `/architect`, Squad's diff approval. Single admin with `bypassPermissions` is what claude-telegram-agent uses today; expose toggle per agent. |
| Plan mode toggle (show plan, don't execute) | **D** | S | Aider `/architect`, Claude Code plan mode. Mobile-friendly: read plan, tap approve. |
| Stop / interrupt turn | **TS** | S | claude-telegram-agent has `/stop`; users hit this constantly when the agent goes off-rails. |
| Re-roll last response | **D** | S | Cheap; users expect ChatGPT-style. |
| Undo last agent action (git checkout the change) | **D** | M | Powerful only if it actually works — needs the worktree commit-per-turn pattern. Trumpet only if confidence is high. |
| Inline approval prompts (tap to allow tool call) | **TS** | M | Kanna's interactive prompts. On mobile this beats keyboard input. |
| `/ask` mode (read-only Q&A about the repo) | **D** | S | Aider pattern; lighter than full agent mode. |
| Slash commands menu | **D** | S | Discoverability — claude-telegram-agent already has 6 commands worth exposing. |
| Voice input | **D** | S | Web Speech API is free; phone-first UX wants it. Differentiator because no competitor surveyed has it on web. |
| Code snippet syntax highlighting (mobile-friendly, wraps not scrolls) | **TS** | S | Mobile-readability table stakes. |
| Export chat to markdown | **D** | S | Cheap; users archive long sessions. |
| Search across all agents' chat histories | **D** | M | Cross-cutting power-user feature; SQLite FTS5 ~free. |

### 8. Telegram Integration

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Attach Telegram bot to any agent without restart | **TS for this product** | M | PROJECT.md mandates. Reuse Python sidecar from `claude-telegram-agent`. |
| One bot can switch which agent it controls (slash command) | **TS** | S | PROJECT.md mandates. `/cd` already exists in the Python bot. |
| Per-agent bot vs shared bot (user picks) | **D** | S | Per-agent = louder/clear; shared = fewer tokens to manage. Let the user choose. |
| Whitelist auth (single allowed user ID) | **TS** | S | Already in the Python sidecar; non-negotiable. |
| `/status`, `/stop`, `/new`, `/projects`, `/cd` parity with claude-telegram-agent | **TS** | S | Migration story — existing user shouldn't lose functionality. |
| File upload from Telegram (photos, docs → agent) | **D** | S | python-telegram-bot handles natively; piped to the agent's workspace. Big win for "snap a screenshot, ask the agent." |
| Voice messages from Telegram | **D** | M | python-telegram-bot supports; whisper transcription on the sidecar. Phone-first UX shines. |
| Inline keyboard for approval prompts | **D** | S | Cleaner than typing "yes/no" on phone. |
| Per-user bot multiplexing | **A** for v1 | M | Single admin scope; defer. |
| Bot-to-bot agent coordination via Telegram | **A** | L | Convoluted; the in-app message bus is the right channel, not Telegram. |

### 9. Cross-Agent Coordination

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Internal message-bus interface (scaffold only, not enabled) | **TS for this product** | S | PROJECT.md mandates the scaffold. Use EventEmitter behind an interface; later swap to Redis pub/sub. |
| Manual task handoff (user moves task from agent A to agent B) | **TS** | M | The "humans-only in MVP1" workflow. UI: pick task, pick target agent, send. |
| Shared scratchpad / whiteboard (read-write file all agents see) | **D** | S | claude-office has a whiteboard. Cheap and a strong demo. |
| Agent-to-agent direct messaging (autonomous) | **A** for v1 | L | PROJECT.md explicitly scopes this to MVP2. Only the interface ships now. |
| Task queue with priority + dependencies | **A** for v1 | L | Octogent has this with the tentacle/todo.md pattern; significant complexity for a feature solving a problem v1 users won't have (one human, few agents). |
| Shared memory / RAG across agents | **A** for v1 | L | agent-office uses SQLite + embeddings; too heavy for v1. Defer. |
| Delegation visualization (parent → child agent arrows on the canvas) | **D** | M | When MVP2 lands, this is what makes the office metaphor pay off. Build the data model in v1 so visualization is easy later. |

### 10. Deployment / Portability

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| `docker compose up -d` brings entire stack online | **TS for this product** | M | PROJECT.md's core promise. Tested clean-host bring-up <2 min. |
| First-run auto-bootstrap (master key, admin user, access URL) | **TS** | M | PROJECT.md mandates. No manual config files for the user. |
| All state in single named volume | **TS** | S | PROJECT.md mandates. |
| Backup = `tar` the volume | **TS** | S | Documented runbook; nothing fancy. |
| Restore = `tar -x` + `docker compose up` on any host | **TS** | M | The whole portability story tested end-to-end. |
| Healthchecks for every container | **TS** | S | Compose-native `healthcheck:`. Users notice when things silently die. |
| Bundled Caddy reverse proxy with Let's Encrypt (toggle) | **TS** | M | PROJECT.md spec. Phone-from-anywhere needs HTTPS. |
| Auto-update channel (`docker compose pull && up`) | **D** | S | Watchtower-style or a simple "check for new release" badge in UI. |
| Pre-flight system check (Docker version, disk, ports) on first run | **D** | S | Saves a class of support requests. |
| Multi-arch images (amd64, arm64) | **TS** | S | Owner might run on a Raspberry Pi or M-series Mac mini. |
| Cluster / Kubernetes deployment | **A** for v1 | L | OpenHands does it via Helm. Out of scope: this is a single-host product. |
| Cloud-hosted SaaS tier | **A** | L | Opposite of self-host thesis. Maybe a separate product later. |

### 11. Auth & Multi-User

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Single admin user, bcrypt + JWT | **TS** | S | PROJECT.md mandates. |
| Auto-generated admin password on first run | **TS** | S | Printed once to compose logs. |
| Password change UI | **TS** | S | Trivial; users will rotate. |
| Session timeout / "remember this device" | **TS** | S | PWA on phone needs long-lived session or it's painful. |
| API tokens for programmatic access (read-only + read-write) | **D** | M | Scripting the office (CI, cron job to summarize) becomes possible. |
| RBAC / multi-user / team accounts | **A** for v1 | L | PROJECT.md explicitly out of scope. |
| SSO (Google/GitHub OAuth) | **A** for v1 | M | Single admin doesn't need it. Defer. |
| 2FA / TOTP | **D** | M | Internet-exposed admin panel + Telegram tokens = juicy target. Strongly recommended even at single-user. |

### 12. Mobile / Phone UX

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Responsive layout (portrait phone primary) | **TS for this product** | M | PROJECT.md mandate; this is the differentiation axis. |
| PWA installable (manifest, service worker, offline shell) | **TS** | M | Devin shipped PWA-install in 2026; expected. |
| Web push notifications (turn complete, error, awaiting approval) | **D** | M | VAPID + service worker. Most competitors ship this only in Telegram, not web. |
| Telegram-native fallback when web push blocked | **TS** | S | Already covered by Telegram integration. |
| Haptic feedback on key actions | **D** | S | `navigator.vibrate()` — cheap personality, feels native. |
| Tap-to-approve large buttons in tool prompts | **TS** | S | Mobile-thumb-friendly approval is the difference between using on phone and giving up. |
| Bottom-sheet chat panel (iOS-style) | **D** | S | UX polish; reads as native, not "shrunk desktop." |
| Voice dictation for chat input | **D** | S | Web Speech API; phone-first edge. |
| Reachability: thumb-zone primary nav | **TS** | S | Material/HIG guidance; affects layout choices. |
| Native iOS/Android wrappers | **A** for v1 | L | PROJECT.md explicitly out of scope; PWA suffices. |
| Background sync (offline-queue user messages) | **A** for v1 | M | Service workers can do it, but reasoning about "queued for an agent that may have moved" is gnarly. Defer. |

### 13. Observability

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Per-agent log tail (live) | **TS** | M | claude-telegram-agent already logs to file; surface in UI. |
| Structured error display ("rate-limit", "tool failed", with link to retry) | **TS** | M | Anthropic Agent View shows failed state; users need the why. |
| Cost roll-up (daily / weekly / per-agent / per-model) | **D** | M | Devin and Cursor both ship this. Owners watching API spend = repeat-engagement hook. |
| Performance dashboard (avg turn duration, tokens/sec, success rate) | **D** | M | Visible value over time; sets up for "which agent is best" hypotheses. |
| OpenTelemetry traces / metrics export | **A** for v1 | M | OpenHands has it; over-built for single admin. Add when multi-user lands. |
| Webhook on lifecycle events (turn started, ended, error) | **D** | S | Integrators want this; cheap to add since SDK already emits events. |
| In-UI alerting (toast on red state, sound, vibrate) | **TS** | S | Cheap and high-value. |
| External alerting (email, Slack, Discord, Telegram on critical errors) | **D** | S | Telegram path already exists; route critical errors there. |
| Prompt-decay detector (warn when ctx >70%, suggest /compact) | **D** | S | Cheap heuristic; aligns with PROJECT.md context color thresholds. Differentiator because most tools just show the % without prescribing action. |

### 14. Plugins / Extensibility

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Per-agent MCP server configuration | **D** | M | MCP is now table stakes in the broader ecosystem (Claude Code, Codex, Cursor all support it). For a "manage all agents" product, exposing per-agent MCP server lists is high-value. Reading: confirms HIGH user expectation as of 2026. |
| Per-agent custom system prompt / persona | **TS** | S | Without this, every agent is a generic Claude — no "frontend dev" vs "DBA" distinction. |
| Per-agent hook scripts (pre-turn, post-turn) | **D** | M | Claude Code SDK has HTTP hooks; surface them in UI. |
| User-defined agent skills (drop a folder, agent gets a skill) | **D** | M | VoltAgent's `awesome-agent-skills` ecosystem; we mount and expose. |
| Plugin marketplace | **A** for v1 | L | agent-office's plugin system is the only one shipping; marketplaces require curation and brand. Defer. |
| Webhook receivers (external system POSTs a task into the office) | **D** | S | Closes the loop with Jira/Linear/etc. |

### 15. Security

| Feature | Tag | Complexity | Rationale / Notes |
|---|---|---|---|
| Sandboxed Docker container per agent (read-only root, dropped caps) | **TS for this product** | M | PROJECT.md mandates after CVE-2025-59536 + CVE-2026-21852. Non-negotiable. |
| Network egress allowlist per agent (only `api.anthropic.com`, repo's git host, etc.) | **TS** | M | Same CVEs demonstrated key exfiltration via repo hooks. Use a per-container network namespace with iptables or a sidecar proxy. |
| No host Docker socket access from inside agent containers | **TS** | S | PROJECT.md mandates. Compose enforces. |
| Disabled-by-default `bypassPermissions` for non-admin agents | **D** | S | claude-telegram-agent defaults to `bypassPermissions` because single-user; surface a per-agent toggle. |
| Content scanning on uploaded files (e.g. block .env, .ssh paste-in) | **D** | M | Cheap regex + size check; saves users from themselves. |
| Supply-chain pin (image SHAs, not `:latest`) | **TS** | S | Watchtower without pinning is itself a CVE. |
| Audit log of every secret-vault access | **D** | M | Already listed in §5; cross-cutting with security. |
| Bring-your-own TLS cert (Caddy supports) | **D** | S | Some users hate Let's Encrypt; cheap to expose. |
| Per-agent CPU / memory quota | **TS** | S | Compose `deploy.resources` or `mem_limit`. One runaway agent should not OOM the host. |
| Egress firewall in admin UI ("add allowed host for agent X") | **D** | M | Saves users from editing iptables. Powerful. |
| WAF / DDoS protection in front of admin UI | **A** for v1 | L | Self-host audience puts their own Cloudflare in front. Don't bake it in. |
| Network isolation testing in CI | **TS** for the project (not the user) | M | Internal: every build verifies an agent container *cannot* reach `metadata.google.internal` or `169.254.169.254`. |

---

## Five+ Differentiators Specific to the AI Agent Office Vision

These are the features that make the product not-just-another-Kanna-fork. They directly serve the "phone-first, portable, multi-model office" thesis.

1. **2D top-down PixiJS office canvas with sprite animations and status colors** — visual identity nothing else in the surveyed set has at the level of polish + utility (closest is paulrobello/claude-office, which is more demo than product).
2. **Phone-first PWA with web push + voice input + haptics** — Devin has PWA install, but no surveyed product is *designed* for the phone as the primary screen.
3. **Telegram bot per agent, hot-attachable, with shared secrets vault** — reusing the proven `claude-telegram-agent` code as a containerized sidecar is uniquely possible because the asset exists; no competitor ships first-class Telegram control.
4. **`docker compose up` portability with `tar`-the-volume migration** — most competitors are either local-CLI (Squad, Aider, Kanna) or cloud-only (Devin, Cursor BG, Manus). The "move my whole office to another VPS in 5 minutes" story is unique.
5. **Pluggable multi-provider backend (Claude SDK + OpenAI Responses + Gemini/Ollama hooks)** — Kanna and Squad have 2–4 providers; the office's provider interface is designed for swap-in/swap-out without core changes.
6. **Sandboxed Docker per agent with egress allowlist** — direct response to CVE-2025-59536 / CVE-2026-21852. No surveyed open-source competitor enforces this by default; even claudebox is a template, not a product.
7. **Scaffolded inter-agent message bus** (humans-route-tasks today, agents-talk-tomorrow) — Octogent has the pattern but not the swap-in interface. Sets up MVP2 without rework.
8. **Per-agent egress firewall + key-usage audit log in the admin UI** — turns the secrets vault from "encrypted at rest" to "observably governed" without enterprise tooling.

---

## Anti-Features (Warnings)

These keep coming up in adjacent products. Resist them in v1.

| Anti-Feature | Why It Hurts | Alternative |
|---|---|---|
| Cost-aware auto-routing (model picker) | Misclassifies, surprises users with wrong-model output, opaque debugging | Make model selection explicit per-agent + per-message override |
| Agent marketplace / templates | Sprawl, prompt-decay, support burden, no curation team in v1 | Allow user-saved presets in the database (free, no UI for "share") |
| Real-time multiplayer (other humans see same office) | State-sync complexity, no use case for single admin | Keep single-user; revisit only if RBAC ships |
| Full in-browser code editor (Monaco) | L-effort, competes with VS Code, the agent is the editor in this product | Diff view + one-shot exec dialog; user edits in their real IDE |
| Embedded PTY in browser | Cross-platform PTY is hard; usage is rare once the agent + exec dialog work | One-shot exec command; punt to SSH for power users |
| Native iOS/Android wrapper apps | PROJECT.md scopes them out; PWA covers it | Polished PWA + Add-to-Home-Screen |
| Multi-floor / multi-office | Solving a problem v1 users (<20 agents) don't have | One floor, room-based grouping ships later |
| Autonomous agent-to-agent messaging | Loops, runaway cost, debugging nightmare | Interface scaffolded only; humans route in v1 |
| Auto-rotate API keys | Most providers don't support programmatic rotation; users lock themselves out | Manual rotate-and-revoke flow, audit log surfaces stale keys |
| Cloud-hosted SaaS tier | Opposite of self-host thesis, splits product focus | Stay self-host; users who want SaaS use Devin |
| Shared RAG across agents | Token cost, indexing complexity, marginal value | Per-agent workspace + optional shared scratchpad file |
| Real-time everything (every state change broadcast) | Back-pressure on slow networks, battery drain on phone | Coalesce updates, throttle UI to ~5 Hz, send full snapshots on reconnect |
| 3D / isometric office view | Burns rendering budget, doesn't add comprehension | 2D top-down per PROJECT.md |
| Cluster / Kubernetes deployment | OpenHands' territory; opposite of "one-VPS-one-owner" | `docker compose` only |

---

## Feature Dependencies

```
[Secrets Vault] ── required by ──> [Agent Lifecycle (create)]
                                       │
                                       ├─ required by ──> [Provider Routing]
                                       └─ required by ──> [Telegram Bot Attach]

[Sandboxed Docker per agent] ── required by ──> [Agent Lifecycle (start)]
        │
        └─ required by ──> [Workspace Management] ── required by ──> [Git Operations]

[Live Monitoring pipeline (SDK → WS)]
        │
        ├─ required by ──> [2D Office card overlay]
        ├─ required by ──> [Status colors / sprite states]
        ├─ required by ──> [Web Push notifications]
        └─ required by ──> [Telegram /status]

[Internal Message Bus (scaffold)] ── required by ──> [Manual Task Handoff]
                                                          │
                                                          └─ enables ──> [Autonomous A2A (MVP2)]

[PWA shell + service worker] ── required by ──> [Web Push] ── required by ──> [Mobile alerting]

[Worktree-per-agent] ── required by ──> [Undo last action], [Open PR from UI]

[Backup/Restore tar] ── requires ──> [All state in single named volume]
                                            │
                                            └─ enforced by ──> [Container volume conventions]
```

### Key Cross-Cutting Dependencies

- **Telegram integration depends on Secrets Vault** — bot tokens have to live in the same encrypted store, otherwise the portability story breaks.
- **Sandboxed Docker per agent depends on the workspace abstraction** — every persistent path is a bind mount; the egress allowlist needs to know the git host.
- **The 2D office canvas card overlay depends on the live monitoring pipeline** — without WS-pushed `context_window.used_percentage`, the green/yellow/red colors are static.
- **Web Push depends on PWA shell with valid service worker** — needs HTTPS, which makes Caddy + Let's Encrypt a hard prerequisite for mobile alerting.
- **"Phone-first" is a constraint on every other category** — every UI feature must be tested in a 390×844 portrait viewport before it's done.

---

## MVP Definition

### Launch With (v1 — MVP1)

The minimum to validate "one-command portable office of multi-model agents, controlled from a phone."

**Lifecycle & Workspace**
- [ ] Create agent from git URL / local path / scratch dir
- [ ] Start / pause / resume / archive / delete
- [ ] Each agent in a sandboxed Docker container, read-only root, egress allowlist, no host docker socket
- [ ] Worktree-per-agent (or clone-per-agent if worktree too risky at v1)
- [ ] Persist all state in a single named volume
- [ ] Auto-restart on crash; per-container memory/CPU quota

**Provider & Secrets**
- [ ] Claude Code SDK provider (full)
- [ ] OpenAI Responses provider (full)
- [ ] Pluggable provider interface in code
- [ ] Per-agent provider/model lock
- [ ] Secrets vault: add/rotate/revoke, AES-256-GCM at rest, per-agent assignment
- [ ] Bot tokens in same vault

**2D Office Frontend**
- [ ] PixiJS canvas with desk grid, one sprite per agent
- [ ] Status colors (green/yellow/red) per PROJECT.md thresholds
- [ ] Card overlay (project, model, ctx%, turns, cost, current task, last activity)
- [ ] Click desk → chat panel
- [ ] Animated "thinking" cursor + auto-compaction banner
- [ ] Live char-by-char streaming over WebSocket

**Chat**
- [ ] Message history (SDK JSONL backed)
- [ ] File attachment upload
- [ ] Diff view inside chat for agent edits
- [ ] Per-tool approval gating with mobile-friendly tap-to-approve
- [ ] Stop / interrupt turn
- [ ] Slash commands menu mirroring claude-telegram-agent (/new, /status, /projects, /cd, /stop)
- [ ] Syntax-highlighted, mobile-wrapping code blocks
- [ ] One-shot "run command in agent's container" dialog

**Telegram**
- [ ] Containerized Python sidecar reusing claude-telegram-agent
- [ ] Attach bot to any agent without restart
- [ ] One bot can switch agents via slash command
- [ ] `/status`, `/stop`, `/new`, `/projects`, `/cd` parity
- [ ] File upload from Telegram into agent workspace
- [ ] Inline keyboard for tool approval prompts

**Coordination (scaffold only)**
- [ ] Internal message-bus interface (EventEmitter implementation behind an interface)
- [ ] Manual task handoff UI (pick task → pick target agent)
- [ ] No autonomous A2A — humans route only

**Deployment**
- [ ] `docker compose up -d` → working stack in <2 min on clean host
- [ ] First-run auto-bootstrap (master key, admin user, access URL printed once)
- [ ] All state in a single named volume; documented backup/restore runbook
- [ ] Bundled Caddy reverse proxy with optional Let's Encrypt
- [ ] Healthchecks on every container; multi-arch images (amd64, arm64)

**Auth**
- [ ] Single admin, bcrypt + JWT
- [ ] Auto-generated password on first run
- [ ] Password change UI
- [ ] Long-lived session for PWA
- [ ] 2FA / TOTP

**Mobile**
- [ ] Responsive layout, portrait-phone primary
- [ ] PWA installable (manifest + service worker)
- [ ] Web push notifications (turn complete, error, awaiting approval)
- [ ] Haptic feedback on approvals
- [ ] Voice dictation for chat input

**Observability**
- [ ] Per-agent live log tail
- [ ] Structured error display with retry
- [ ] In-UI alerting (toast + sound + vibrate)
- [ ] Critical-error escalation via Telegram

**Security**
- [ ] Per-agent sandboxed Docker (already listed) — including drop caps, no-new-privileges
- [ ] Network egress allowlist per agent
- [ ] CI integration test that verifies network isolation
- [ ] Image-SHA pinning in compose
- [ ] Audit log of secret-vault access

### Add After Validation (v1.x)

Trigger to add: first user reports of the missing feature, or measurable usage of the precursor feature.

- [ ] Free-form layout editor (drag desks) — trigger: users ask "can I rearrange?"
- [ ] Sprite animations during tool calls — trigger: demo / share-on-social pressure
- [ ] Per-tool timeline visualization — trigger: debugging session length grows
- [ ] Subagent / Task-tool tree visualization — trigger: users start using subagents heavily
- [ ] Cost roll-up dashboards (daily/weekly/per-agent) — trigger: user's monthly bill conversation
- [ ] Open PR from UI — trigger: git-push usage rate hits a threshold
- [ ] Repo tree browser — trigger: requests for "what files does this agent see?"
- [ ] Search across all chat histories (SQLite FTS5) — trigger: history gets too long to scroll
- [ ] Webhook receivers (external task injection) — trigger: integration requests
- [ ] Per-agent MCP server configuration — trigger: MCP server adoption among users
- [ ] API tokens for programmatic access (R/RW) — trigger: power-user automation requests
- [ ] Per-agent custom hook scripts — trigger: advanced workflows surface
- [ ] Content scanning on uploads — trigger: any incident of accidental secret paste
- [ ] Egress firewall in admin UI — trigger: users hitting allowlist denial
- [ ] Re-roll last response, export chat to markdown — opportunistic / cheap wins

### Future Consideration (v2 — MVP2 and beyond)

Defer until the v1 thesis is validated and the audience is bigger than the single owner.

- [ ] Autonomous agent-to-agent messaging (enabling the scaffolded bus) — the headline MVP2 feature
- [ ] Delegation visualization on the canvas (arrows, hand-off animations) — MVP2 dependency
- [ ] Cost-aware routing — only after Arena-mode-style human comparison runs build trust in the routing logic
- [ ] Agent skills / templates / marketplace — needs curation strategy first
- [ ] Multi-floor / multi-office canvas — only when agent count routinely exceeds ~20
- [ ] Multi-user / team accounts / RBAC / SSO — when the product moves past single owner
- [ ] OpenTelemetry export — when multi-user lands and observability becomes ops
- [ ] Arena mode (multi-model bake-off on the same task) — flashy but tangential
- [ ] Shared RAG / cross-agent memory — heavy; only if real coordination need surfaces

---

## Feature Prioritization Matrix (top items)

| Feature | User Value | Implementation Cost | Priority |
|---|---|---|---|
| 2D office canvas + status colors + card overlay | HIGH | HIGH | P1 |
| Sandboxed Docker + egress allowlist per agent | HIGH | MEDIUM | P1 |
| Secrets vault (encrypted, per-agent assignment) | HIGH | MEDIUM | P1 |
| Multi-provider (Claude SDK + OpenAI Responses) | HIGH | HIGH | P1 |
| Telegram sidecar attach/switch | HIGH | MEDIUM | P1 |
| `docker compose up` portability + Caddy + bootstrap | HIGH | MEDIUM | P1 |
| PWA + web push | HIGH | MEDIUM | P1 |
| Diff view + tap-to-approve | HIGH | MEDIUM | P1 |
| Live monitoring (ctx%, cost, tokens, current task) | HIGH | MEDIUM | P1 |
| Worktree-per-agent | HIGH | MEDIUM | P1 |
| Message-bus interface (scaffold) | MEDIUM | LOW | P1 |
| Manual task handoff UI | MEDIUM | MEDIUM | P1 |
| Voice dictation + haptics | MEDIUM | LOW | P1 |
| 2FA / TOTP | MEDIUM | MEDIUM | P1 |
| Free-form layout editor | MEDIUM | MEDIUM | P2 |
| Sprite animations on tool calls | MEDIUM | MEDIUM | P2 |
| Cost roll-up dashboards | MEDIUM | MEDIUM | P2 |
| Open PR from UI | MEDIUM | MEDIUM | P2 |
| Per-agent MCP server config | MEDIUM | MEDIUM | P2 |
| Repo tree browser | MEDIUM | MEDIUM | P2 |
| Search across chats (FTS5) | MEDIUM | LOW | P2 |
| API tokens / webhook receivers | MEDIUM | LOW | P2 |
| Autonomous A2A messaging | HIGH (long term) | HIGH | P3 |
| Multi-floor / room-based grouping | LOW (now) | MEDIUM | P3 |
| Arena mode | LOW | HIGH | P3 |
| Cluster/K8s deployment | LOW | HIGH | P3 (anti for v1) |
| RBAC / SSO / multi-user | LOW (now) | HIGH | P3 |
| Marketplace | LOW | HIGH | P3 (anti for v1) |

**Priority key:**
- **P1** — Must ship in MVP1 (v1). Required to validate the core thesis.
- **P2** — Add after MVP1 ships and users adopt. Most are explicit ROI bets.
- **P3** — Defer; many are MVP2 or anti-features for v1.

---

## Competitor Feature Analysis (selected)

| Feature | Claude Squad | Octogent | Kanna | Devin | claude-office | Anthropic Agent View | **Our Approach** |
|---|---|---|---|---|---|---|---|
| Multi-provider | Yes (4) | Claude only | Claude+Codex | Devin runtime | Claude only | Claude only | Claude + OpenAI + pluggable |
| UI | TUI | Web | Web | Web + PWA | Web (pixel) | TUI | Web/PWA (pixel-art, phone-first) |
| Visualization | Tab preview | Tentacle tree | Sidebar | Sessions list | Pixel office | Sessions list | 2D top-down office canvas |
| Workspace isolation | tmux + worktree | worktree | Per-project | VM | n/a | n/a | Docker container + worktree |
| Secrets management | Env vars | Env vars | Env vars | Cloud-managed | n/a | API key | Encrypted vault + per-agent + audit |
| Phone support | None | None | None | PWA | None | None | PWA + push + Telegram (primary) |
| Telegram | None | None | None | None | None | None | First-class sidecar |
| Inter-agent | None | Messaging | None | None | "Subagents" UI | Subagent rows | Scaffolded bus (humans route in v1) |
| Self-host | Yes | Yes | Yes | Cloud only | Yes | Local | Yes (the whole point) |
| One-command bring-up | `brew install` | `npm` | `bun` | n/a | `make install-all` | `claude agents` | `docker compose up -d` |
| Network sandboxing | None | None | None | Cloud-managed | None | None | Egress allowlist (default-deny) |
| MCP per-agent | Partial | n/a | Partial | n/a | n/a | Yes | Planned P2 |

The pattern: every competitor wins on one or two axes (Squad on workspace isolation, Kanna on web UX, Devin on cloud + PWA, Octogent on orchestration, Anthropic Agent View on first-party SDK integration). **No one combines all of: 2D office + phone-first PWA + Telegram + portable Docker + multi-provider + hardened sandbox.** That combination is the office.

---

## Open Questions for Phase-Level Research

1. **Worktree vs clone per agent at v1** — worktree is differentiator-strong but adds git-state complexity. Recommend revisiting in Phase 2 (workspace).
2. **Egress allowlist implementation** — per-container netns + iptables vs sidecar proxy (mitmproxy, gVisor). Affects security & ops cost; Phase 2 or 3.
3. **WebSocket back-pressure strategy on slow phone networks** — coalesce vs drop vs snapshot-on-reconnect. Phase 2 (real-time).
4. **PixiJS performance budget on low-end Android** — sprite count ceiling, texture atlas strategy. Phase 3 (visualization).
5. **Python sidecar ↔ Node backend transport** — HTTP REST vs Redis pub/sub. PROJECT.md says either; pick during Phase 2.

---

## Sources

- [paulrobello/claude-office (GitHub)](https://github.com/paulrobello/claude-office)
- [paulrobello/claude-office WHITEBOARD docs](https://github.com/paulrobello/claude-office/blob/main/docs/WHITEBOARD.md)
- [harishkotra/agent-office (GitHub)](https://github.com/harishkotra/agent-office)
- [Dev.to: How I Built AgentOffice](https://dev.to/harishkotra/how-i-built-agentoffice-self-growing-ai-teams-in-a-pixel-art-virtual-office-4o0p)
- [hesamsheikh/octogent (GitHub)](https://github.com/hesamsheikh/octogent)
- [Octogent docs index](https://github.com/hesamsheikh/octogent/blob/main/docs/index.md)
- [Octogent experimental features](https://github.com/hesamsheikh/octogent/blob/main/docs/reference/experimental-features.md)
- [Magicshot: Octogent dashboard summary](https://magicshot.ai/news/octogent-claude-code-multi-agent-dashboard/)
- [smtg-ai/claude-squad (GitHub)](https://github.com/smtg-ai/claude-squad)
- [Claude Squad website](https://smtg-ai.github.io/claude-squad/)
- [Claude Squad README](https://github.com/smtg-ai/claude-squad/blob/main/README.md)
- [hoangsonww/Claude-Code-Agent-Monitor (GitHub)](https://github.com/hoangsonww/Claude-Code-Agent-Monitor)
- [Claude Code Agent Monitor wiki](https://hoangsonww.github.io/Claude-Code-Agent-Monitor/)
- [disler/claude-code-hooks-multi-agent-observability](https://github.com/disler/claude-code-hooks-multi-agent-observability)
- [jakemor/kanna (GitHub)](https://github.com/jakemor/kanna)
- [Kanna on HN](https://news.ycombinator.com/item?id=47501122)
- [Anthropic Agent View: Code with Claude 2026 recap](https://www.mindstudio.ai/blog/code-with-claude-2026-new-agent-features)
- [TestingCatalog: Agent View](https://www.testingcatalog.com/anthropic-adds-agent-view-for-claude-code-for-parralel-work/)
- [Claude Code docs](https://code.claude.com/docs/en/overview)
- [Simon Willison live blog: Code w/ Claude 2026](https://simonwillison.net/2026/May/6/code-w-claude-2026/)
- [Cursor 2.0 + Windsurf Wave 13 comparison](https://www.codecademy.com/article/agentic-ide-comparison-cursor-vs-windsurf-vs-antigravity)
- [Verdent: Windsurf vs Cursor 2026](https://www.verdent.ai/guides/windsurf-vs-cursor-2026)
- [Devin 2026 release notes](https://docs.devin.ai/release-notes/2026)
- [Cognition: Introducing Devin 2.2](https://cognition.ai/blog/introducing-devin-2-2)
- [OpenHands website](https://www.openhands.dev/)
- [All-Hands-AI/OpenHands-Cloud](https://github.com/All-Hands-AI/OpenHands-Cloud)
- [Aider docs](https://aider.chat/docs/)
- [Aider 2026 review](https://aiagentslist.com/agents/aider)
- [Manus product page](https://manus.im/)
- [Manus AI Cloud Computer](https://aiautomationglobal.com/blog/manus-cloud-computer-always-on-ai-automation-2026)
- [CrewAI vs LangGraph vs AutoGen 2026](https://dev.to/emperorakashi20/crewai-vs-langgraph-vs-autogen-which-multi-agent-framework-should-you-use-in-2026-5h2f)
- [Stack Overflow: bugs and incidents with AI coding agents](https://stackoverflow.blog/2026/01/28/are-bugs-and-incidents-inevitable-with-ai-coding-agents/)
- [The Gnar: Why your AI coding agent keeps making bad decisions](https://www.thegnar.com/blog/why-your-ai-coding-agent-keeps-making-bad-decisions-and-how-to-fix-it)
- [Infisical: Best secret management tools 2026](https://infisical.com/blog/best-secret-management-tools)
- [MDN: PWA push notifications](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Tutorials/js13kGames/Re-engageable_Notifications_Push)
- [VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills)
- Internal: `C:\Users\luism\Documents\Luis\PROYECTOS\CURSOR\office\.planning\PROJECT.md`
- Internal: `C:\Users\luism\Documents\Luis\PROYECTOS\CURSOR\claude-telegram-agent\README.md`

---

*Feature research for: AI Agent Office (self-hostable multi-AI dashboard)*
*Researched: 2026-05-13*
