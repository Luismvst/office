---
phase: 1
slug: local-app-shell
status: supersedes-old-phase-1
created: 2026-05-13
---

# Phase 1 Pivot Note

The previous Phase 1 research and validation files targeted a Docker/VPS first-run product:

- portable Docker Compose stack
- admin login
- Caddy/public URL
- Docker socket proxy
- filesystem and clock guards

That direction is superseded.

Current Phase 1 is `local-app-shell`: a localhost-only app shell for the local visual agent office. Planning should use `.planning/PROJECT.md`, `.planning/REQUIREMENTS.md`, and `.planning/ROADMAP.md` as the source of truth.

Do not treat old Docker/VPS research as binding. At most, reuse background details about Claude Code status/context handling and visual office references when relevant to later phases.
