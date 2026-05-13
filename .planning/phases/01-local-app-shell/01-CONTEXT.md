# Phase 1: local-app-shell - Context

**Gathered:** 2026-05-13T21:30:53+02:00
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 1 delivers the local app shell: a localhost-only Node/Fastify backend, React/Vite frontend, local SQLite persistence, WebSocket plumbing, and an empty PixiJS 2D office screen. It does not launch real agents yet; that starts in Phase 2.

</domain>

<decisions>
## Implementation Decisions

### App Structure
- **D-01:** Use a monorepo with `apps/web` for React/Vite and `apps/server` for Node/Fastify.
- **D-02:** Keep the app local-first and bind to localhost by default. No login/auth in Phase 1.

### Persistence
- **D-03:** Use local SQLite with Drizzle. Avoid JSON/localStorage because sessions, status, and logs will need structured persistence soon.

### Real-Time Transport
- **D-04:** Use WebSocket from the start. The visual office needs live status and later bidirectional commands/logs.

### Visual Office
- **D-05:** Use PixiJS from Phase 1 so the real visual engine is present immediately.
- **D-06:** Initial style is clean functional pixel art: desks, simple avatars, readable labels, and clear status colors. It should feel like a lightweight management game, not a conventional dashboard.
- **D-07:** Before creating visuals from scratch, search GitHub and permissive OSS references. Prefer cloning/reviewing existing projects and reusing compatible patterns/assets over reinventing the wheel.
- **D-08:** Do not blindly copy code with incompatible licenses. Reuse MIT/permissive code directly when appropriate; use GPL or unclear-license projects only as inspiration unless the whole project intentionally adopts that license.

### the agent's Discretion
- Choose exact package manager and monorepo tooling unless existing repo constraints appear.
- Choose whether Phase 1 includes mocked agent fixtures in the empty office, as long as it does not imply real agent launching before Phase 2.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Planning
- `.planning/PROJECT.md` - local-first product direction and locked pivot decisions.
- `.planning/REQUIREMENTS.md` - Phase 1 requirements: LOCAL-01 through LOCAL-05.
- `.planning/ROADMAP.md` - Phase 1 scope and phase boundaries.
- `.planning/STATE.md` - current project state and superseded old roadmap notes.
- `.planning/phases/01-local-app-shell/01-PIVOT.md` - warning that the old Docker/VPS Phase 1 research is superseded.

### External References To Review
- `https://github.com/paulrobello/claude-office` - MIT reference implementation: real-time pixel-art Claude Code office using Next.js, PixiJS, FastAPI, Zustand, WebSocket, hooks, and persistent settings.
- `https://github.com/pixijs/pixijs` - MIT 2D rendering engine; use current PixiJS v8 patterns.
- `https://pixijs.com/llms` - official PixiJS AI/LLM guidance and skills; use for v8 implementation patterns.
- `https://github.com/phaserjs/phaser` - MIT alternative game framework; review only if PixiJS proves awkward.
- `https://github.com/harishkotra/agent-office` - pixel-art AI agent office reference; inspect license before copying anything.
- `https://github.com/topics/phaser3` - search surface for virtual-office and pixel-art patterns such as SkyOffice.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- No product code exists yet. The repo currently contains planning artifacts and `CLAUDE.md`.

### Established Patterns
- Planning is GSD-driven under `.planning/`.
- Phase 1 should create the initial app structure rather than adapt existing code.

### Integration Points
- New implementation should start at repo root with a monorepo structure.
- README already describes the local-first product direction and should stay consistent with the generated app commands.

</code_context>

<specifics>
## Specific Ideas

- The product exists because multiple terminal windows make it hard to know which agent has finished, which one is stuck, and which one needs attention.
- The office/game metaphor is central. The user explicitly wants the agents visible as if watching a small videogame office.
- Avoid overengineering. Start local and visual; remote deployment and security hardening come later.
- Strong preference: research existing GitHub repos and clone/reference them before building core visual patterns from scratch.

</specifics>

<deferred>
## Deferred Ideas

- Real Claude Code session launching belongs to Phase 2.
- Live agent status overlays belong to Phase 3.
- Agent detail/chat/control belongs to Phase 4.
- Remote deployment, auth, vault, Telegram, Docker production, and extra providers remain later work.

</deferred>

---

*Phase: 1-local-app-shell*
*Context gathered: 2026-05-13T21:30:53+02:00*
