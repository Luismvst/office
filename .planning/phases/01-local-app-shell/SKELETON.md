# Walking Skeleton - Office

**Phase:** 1
**Generated:** 2026-05-13

## Capability Proven End-to-End

The owner can run one local command, open Office in the browser, see a PixiJS office shell, edit the office name, and see that value persist through the Node/Fastify API into local SQLite.

## Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Monorepo | npm workspaces with `apps/web` and `apps/server` | Uses tooling that ships with Node, keeps web/server separate without extra workspace dependency. |
| Web framework | React + Vite | Fast local iteration and matches the chosen web app direction. |
| 2D renderer | PixiJS v8, preferably via `@pixi/react` if clean | The visual office is the product; Phase 1 should not use a throwaway mock. |
| API framework | Fastify + TypeScript | Small, fast local backend with clean route/plugin structure. |
| Real-time transport | `@fastify/websocket` on `/ws` | Needed for live agent state in later phases; cheap to wire early. |
| Data layer | SQLite file under `.office/office.sqlite` with Drizzle | Local, portable, structured persistence for settings and later agents/sessions. |
| Auth | None in MVP1 | Localhost-only single-owner tool. |
| Network binding | `127.0.0.1` by default | Prevents accidental public exposure in local MVP. |
| Local command | `npm run dev` | One command from repo root starts server and web. |

## Stack Touched in Phase 1

- [ ] Project scaffold: root npm workspace, TypeScript configs, web app, server app.
- [ ] Routing: `/health`, `/api/settings`, `/ws`, and root web route.
- [ ] Database: one real SQLite read and one real write.
- [ ] UI: one interactive office setting wired to the API.
- [ ] Visual shell: PixiJS office canvas renders nonblank, stable, and responsive.
- [ ] Local run: documented `npm install` and `npm run dev` commands.

## Out of Scope (Deferred to Later Slices)

- Launching real Claude Code sessions.
- Agent lifecycle state from real processes.
- Model/context/cost extraction.
- Agent detail/chat/control panel.
- Authentication.
- Remote/VPS deployment.
- Docker production setup.
- Telegram.
- Encrypted vault.

## Subsequent Slice Plan

- Phase 2: register local workspaces and start/stop Claude Code sessions.
- Phase 3: show live visual status, model, context, and current task in the office.
- Phase 4: open agent detail panels with logs/input/stop controls.
- Phase 5: polish local UX and tighten the provider boundary for future providers.
