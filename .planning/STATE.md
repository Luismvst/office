# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-05-13)

**Core value:** One command brings up an entire portable office of multi-model coding agents — anywhere — and the owner controls them all from a phone.
**Current focus:** Phase 1 — foundations-and-first-run

## Current Position

Phase: 1 of 7 (foundations-and-first-run)
Plan: 0 of TBD in current phase
Status: Ready to plan
Last activity: 2026-05-13 — Roadmap created, 90/90 v1 requirements mapped across 7 phases

Progress: [░░░░░░░░░░] 0%

## Performance Metrics

**Velocity:**
- Total plans completed: 0
- Average duration: —
- Total execution time: —

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| — | — | — | — |

**Recent Trend:**
- Last 5 plans: (none yet)
- Trend: (no data)

*Updated after each plan completion*

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Init: Docker required, no PTY fallback (security model relies on container isolation)
- Init: Argon2id (not bcrypt) for admin password hashing — overrides PROJECT.md "bcrypt" mention; aligns with Better-Auth-compatible schema
- Init: Redis in compose stack from day one — serves bus, WS replay, TG polling lock, TG IPC
- Init: `Tecnativa/docker-socket-proxy` from day one — backend never mounts `/var/run/docker.sock`
- Init: Two-layer Bitwarden-style master/data key model for vault; master key written to 3 sinks on first run

### Pending Todos

None yet.

### Blockers/Concerns

- Worktree-vs-clone-per-agent decision deferred to Phase 3 spike (research/SUMMARY.md item)
- Phase 1 egress allowlist mechanism (iptables `DOCKER-USER` vs proxy sidecar) needs research at plan-phase 1 time
- Phase 7 Caddy mode C (Tailscale) is "best-effort optional" — needs a 1-day spike before promising

## Deferred Items

Items acknowledged and carried forward — tracked in REQUIREMENTS.md v2 section:

| Category | Item | Status | Deferred At |
|----------|------|--------|-------------|
| Inter-agent comms | A2A-01..04 (agents publish to other agents) | v2 / MVP1.5 | 2026-05-13 (roadmap creation) |
| Providers | PROV-V2-01..03 (Gemini, Ollama, DeepSeek/Mistral routes) | v2 | 2026-05-13 |
| Multi-user | MU-01..04 (invites, RBAC, OIDC SSO) | v2 | 2026-05-13 |
| UX polish | UX-V2-01..04 (sprite animations, multi-room, layout editor, themes) | v2 | 2026-05-13 |
| Ops | OPS-V2-01..04 (Watchtower, Postgres migration, log retention, Prometheus) | v2 | 2026-05-13 |

## Session Continuity

Last session: 2026-05-13
Stopped at: Roadmap and STATE initialized, 7 phases defined, traceability written to REQUIREMENTS.md
Resume file: None
