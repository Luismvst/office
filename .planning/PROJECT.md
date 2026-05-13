# Office - Local Agent Office

## What This Is

Office is a local web app for watching AI coding agents work as a visual 2D office instead of juggling several terminal windows. Each running agent sits at a desk, shows what it is doing, and exposes the useful operational signals: provider, model, context usage, state, last activity, task text, and recent output.

The first product is local-only. It runs on the developer machine, opens in the browser, and focuses on visibility and control for one owner. Remote VPS deployment, encrypted vaults, Telegram, production hardening, and multi-user access are explicitly later work.

## Core Value

**One local command opens a visual office where the owner can see which agents are working, stuck, done, or close to context limits.**

If everything else is removed, this must work:

1. Start Office locally.
2. Add or launch one or more local agent sessions.
3. See each agent represented visually in a 2D office.
4. Know at a glance: agent state, model, context, current task, and whether attention is needed.
5. Open an agent to view logs/chat and send a message or stop/resume it.

## Requirements

### Validated

(None yet - ship to validate)

### Active

#### Local App Shell
- [ ] One local dev command starts backend + web UI.
- [ ] Browser opens to the office view without login for MVP.
- [ ] App state persists locally enough to remember configured agents between restarts.

#### Agent Sessions
- [ ] User can register a local project/workspace and create an agent session for it.
- [ ] MVP supports Claude Code first.
- [ ] The system captures process lifecycle: starting, working, needs input, completed, failed, stopped.
- [ ] The system captures recent output and session metadata.

#### Visual Office
- [ ] The main screen is a 2D office/game-like view, not a generic dashboard.
- [ ] Each agent is shown as a desk/avatar with visual state color.
- [ ] Status overlay shows project, provider, model, context %, turns/cost when available, current task, and last activity.
- [ ] Completed or blocked agents are visually obvious without opening a terminal.

#### Agent Detail Panel
- [ ] Clicking an agent opens a side panel.
- [ ] Panel shows timeline/log output and key status fields.
- [ ] Owner can send a message to the agent when supported.
- [ ] Owner can stop the current run.

#### Real-Time Updates
- [ ] UI updates live while agents work.
- [ ] Reconnect after browser refresh without losing known agent state.

### Out of Scope (MVP1)

- VPS deployment, Docker Compose production setup, Caddy, TLS, public URL management.
- Encrypted API key vault and provider-key management UI.
- Telegram sidecar or mobile messenger control.
- Multi-user auth, roles, invitations, SSO.
- Full sandboxing of untrusted repositories.
- Built-in backup/restore/update CLI.
- Agent-to-agent autonomous delegation.
- Polished PWA/mobile install experience.
- OpenAI/Gemini/Ollama providers, unless Claude-first MVP is already solid.

## Context

### Existing Assets to Reuse

- The original research remains useful as background for Claude Code SDK details, context usage, and visual office references, but its Docker/VPS/security assumptions are superseded by this local-first pivot.

### Product Direction

The product should feel closer to a lightweight management game than a server admin panel. It exists because multiple agent terminals are hard to track: you forget which one finished, which one needs input, and which model/context is being used.

The visual layer is not decoration. It is the primary interface.

## Constraints

- **Runtime:** local machine first.
- **Backend:** Node.js + TypeScript is acceptable unless implementation research finds a simpler fit.
- **Frontend:** React + Vite + PixiJS or Canvas-based 2D rendering.
- **Persistence:** local SQLite or a small local file DB is enough for MVP.
- **Auth:** no login in local MVP.
- **Security:** do not expose the app publicly in MVP; bind to localhost by default.
- **Provider:** Claude Code first. Keep a provider boundary only where it costs little.
- **UX priority:** glanceable state beats feature depth.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Pivot to local-first visual monitor | The real pain is seeing agents work without many terminal windows | Locked |
| No VPS/Docker production target in MVP1 | Deployment complexity was dominating the first milestone before the core product was visible | Locked |
| No auth for MVP1 | Localhost-only single-owner tool; login slows the first usable slice | Locked |
| Claude Code first | Fastest route to real local agent visibility | Locked |
| 2D office is the primary screen | The product thesis is visual awareness, not a conventional dashboard | Locked |

## Evolution

This document should stay biased toward the smallest local product that proves the visual office loop. Production, vaults, remote access, and extra providers can return after the owner is using the local office day to day.

---
*Last updated: 2026-05-13 after local-first pivot*
