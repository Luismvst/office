# Phase 1: local-app-shell - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md - this log preserves the alternatives considered.

**Date:** 2026-05-13T21:30:53+02:00
**Phase:** 01-local-app-shell
**Areas discussed:** app structure, persistence, real-time transport, rendering engine, visual style, OSS reuse

---

## App Structure

| Option | Description | Selected |
|--------|-------------|----------|
| Monorepo Vite + Node API | `apps/web` React/Vite + `apps/server` Node/Fastify; ordered and ready to grow. | yes |
| Single Vite app with light API | Simpler at first, weaker fit for runner/WebSocket work. | |
| Electron/Tauri from the start | Desktop packaging before product validation. | |

**User's choice:** Monorepo Vite + Node API.
**Notes:** Keep it local-first, but avoid a structure that will collapse when agent runner work starts.

---

## Persistence

| Option | Description | Selected |
|--------|-------------|----------|
| SQLite local with Drizzle | Solid local persistence, useful for sessions/logs later. | yes |
| JSON file | Fast but fragile once sessions and logs appear. | |
| LocalStorage only | Minimal now, likely rework in Phase 2. | |

**User's choice:** SQLite local with Drizzle.
**Notes:** Phase 1 should establish the persistence layer even if the first schema is small.

---

## Real-Time Transport

| Option | Description | Selected |
|--------|-------------|----------|
| WebSocket from the start | Flexible for state, logs, and later bidirectional commands. | yes |
| SSE + REST | Good for one-way status, less natural for controls. | |
| Polling | Easy but mismatched with live visual product. | |

**User's choice:** WebSocket.
**Notes:** Live visual status is core to the product.

---

## Rendering Engine

| Option | Description | Selected |
|--------|-------------|----------|
| PixiJS from Phase 1 | Real visual engine present from the first slice. | yes |
| HTML/CSS mock | Fast scaffold but likely later migration. | |
| Canvas 2D custom | Fewer dependencies, more custom interaction work. | |

**User's choice:** PixiJS from Phase 1.
**Notes:** Avoid a throwaway mock if the visual office is the core product.

---

## Visual Style

| Option | Description | Selected |
|--------|-------------|----------|
| Pixel-art clean and functional | Game-like desks/avatars with readable state. | yes |
| Modern/isometric top-down | More polished, more asset/design cost. | |
| Dashboard with canvas central | More conventional, less game-like. | |

**User's choice:** Pixel-art clean and functional.
**Notes:** The user wants it to feel like watching agents work in a videogame-style office.

---

## OSS Reuse

| Option | Description | Selected |
|--------|-------------|----------|
| Search/clone existing GitHub references | Review existing projects before implementing visual patterns. | yes |
| Build from scratch | More control but wastes time and risks reinventing solved pieces. | |

**User's choice:** Look on GitHub, clone/reference, and do not reinvent the wheel.
**Notes:** Strong preference to reuse existing repositories and assets when licenses allow. `paulrobello/claude-office` is the strongest known reference so far.

---

## the agent's Discretion

- Exact package manager and monorepo tooling can be chosen during planning.
- Mocked office fixtures are acceptable in Phase 1 if they help validate the visual shell without pretending real agents exist yet.

## Deferred Ideas

- Real agent launching: Phase 2.
- Live agent status overlays: Phase 3.
- Detail/chat/control panel: Phase 4.
- Remote deployment/auth/vault/Telegram: later.
