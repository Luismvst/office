---
phase: 1
slug: local-app-shell
status: complete
created: 2026-05-13
sources_checked:
  - https://github.com/paulrobello/claude-office
  - https://react.pixijs.io/getting-started
  - https://pixijs.com/8.x/guides/getting-started/quick-start
  - https://fastify.dev/docs/latest/Guides/Getting-Started/
  - https://github.com/fastify/fastify-websocket
  - https://orm.drizzle.team/docs/get-started-sqlite
---

# Phase 1 Research - local-app-shell

## Research Complete

Phase 1 should create a local walking skeleton, not a production platform. The selected stack from CONTEXT.md is viable:

- `apps/web`: React + Vite + PixiJS.
- `apps/server`: Node + TypeScript + Fastify.
- Local SQLite via Drizzle.
- WebSocket from the backend.
- Localhost-only by default.

## OSS Reference Findings

### `paulrobello/claude-office`

`https://github.com/paulrobello/claude-office`

Best reference found. It is MIT licensed and directly matches the desired product direction: a real-time pixel-art office that visualizes Claude Code operations. It uses Next.js, PixiJS, FastAPI, Zustand, WebSocket, hooks, persistent settings, animated status indicators, context visualization, and office/game metaphors.

What to reuse:

- Visual vocabulary: office, desks, agents, speech/thought bubbles, state indicators, context visualization.
- Event-driven architecture idea: agent/tool events flow into backend, backend broadcasts to frontend.
- UX principle: the visual office is primary, not a decorative dashboard.
- Pixel-art style direction and state clarity.

What not to copy wholesale in Phase 1:

- Backend stack: it uses FastAPI, while this project has locked Node/Fastify.
- Advanced features: multi-floor navigation, whiteboard modes, hooks, OpenCode plugin, Docker, AI summary.
- Claude integration: belongs in Phase 2, not Phase 1.

Planner/executor should clone or inspect this repo during implementation if network is available, then selectively copy MIT-compatible patterns/assets only when they fit the local architecture.

### PixiJS / Pixi React

Official PixiJS docs recommend Node 20+ and provide `npm create pixi.js@latest` scaffolding. `@pixi/react` supports PixiJS v8 in React and exposes Pixi objects through JSX. The React v8 release is designed for React 19.

Recommendation:

- Use direct PixiJS or `@pixi/react`; prefer `@pixi/react` if it works cleanly with Vite and React in the scaffold.
- Keep the scene simple: grid floor, desks, empty state, optional demo desks. Avoid advanced camera/pathfinding in Phase 1.
- Use stable dimensions and explicit resize handling so the canvas does not collapse or go blank.

### Fastify + WebSocket

Fastify defaults examples to localhost unless host is changed. This supports LOCAL-02. `@fastify/websocket` is the official plugin and uses route options with `websocket: true`.

Recommendation:

- Server listens on `127.0.0.1` by default.
- Add `/health` returning JSON readiness.
- Add `/ws` even if it only sends connected/heartbeat/demo events in Phase 1.

### Drizzle + SQLite

Drizzle supports SQLite through both `libsql` and `better-sqlite3`. For a local-only desktop-style MVP, `better-sqlite3` keeps the DB file simple and synchronous.

Recommendation:

- Store DB under `.office/office.sqlite`.
- Add a first schema for `settings` and `agent_configs`, even if only settings are used in Phase 1.
- Include a real read/write path from UI -> API -> SQLite -> UI to satisfy walking skeleton requirements.

## Recommended Phase 1 Shape

One plan is enough. This is a walking skeleton:

1. Scaffold npm workspaces.
2. Create Fastify server, health endpoint, SQLite/Drizzle schema, and WebSocket route.
3. Create Vite React app with PixiJS office canvas.
4. Wire one real UI interaction to SQLite, such as editing the office name.
5. Add smoke tests and README command updates.

## Pitfalls

- Do not overbuild the office. Phase 1 is an empty shell, not live agent status.
- Do not import the whole `claude-office` architecture. Use it as a reference, not a dependency.
- Do not expose `0.0.0.0` in MVP.
- Do not add auth, Docker, vault, Telegram, or Claude launching in Phase 1.
- Do not create a throwaway HTML mock if PixiJS is the chosen renderer.

## Sources

- `https://github.com/paulrobello/claude-office` - MIT reference implementation for real-time pixel-art Claude Code office.
- `https://react.pixijs.io/getting-started` - PixiJS React setup and v8 compatibility.
- `https://pixijs.com/8.x/guides/getting-started/quick-start` - official PixiJS quick start and Node requirement.
- `https://fastify.dev/docs/latest/Guides/Getting-Started/` - Fastify server/listen behavior and localhost guidance.
- `https://github.com/fastify/fastify-websocket` - official WebSocket plugin usage.
- `https://orm.drizzle.team/docs/get-started-sqlite` - Drizzle SQLite setup with `better-sqlite3`.
