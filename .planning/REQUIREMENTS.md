# Requirements: Office - Local Agent Office

**Defined:** 2026-05-13
**Pivoted:** 2026-05-13
**Core Value:** One local command opens a visual office where the owner can see which agents are working, stuck, done, or close to context limits.

## v1 Requirements

Requirements for the local MVP. Each maps to exactly one roadmap phase.

### Local Foundation

- [ ] **LOCAL-01**: User can start the app locally with one documented command.
- [ ] **LOCAL-02**: Backend binds to localhost by default and does not expose a public network service in MVP.
- [ ] **LOCAL-03**: Web UI opens to the office view without login.
- [ ] **LOCAL-04**: Local persistence remembers configured agents and recent sessions across restart.
- [ ] **LOCAL-05**: Health/status endpoint reports backend readiness.

### Agent Runner

- [ ] **AGENT-01**: User can register a local workspace path.
- [ ] **AGENT-02**: User can create a Claude Code agent session for a registered workspace.
- [ ] **AGENT-03**: System starts and tracks the underlying local agent process.
- [ ] **AGENT-04**: System records lifecycle state: idle, starting, working, needs-input, completed, failed, stopped.
- [ ] **AGENT-05**: System captures recent stdout/stderr or SDK stream output for each session.
- [ ] **AGENT-06**: User can stop a running agent from the UI.
- [ ] **AGENT-07**: Session metadata includes provider, model when available, workspace, task, start time, last activity, and exit status.

### Live Status

- [ ] **STATUS-01**: Dashboard shows each agent's current state in real time.
- [ ] **STATUS-02**: Dashboard shows model and provider when available.
- [ ] **STATUS-03**: Dashboard shows context usage when available from the provider/status stream.
- [ ] **STATUS-04**: Dashboard shows turn count and cost when available; unavailable fields degrade gracefully.
- [ ] **STATUS-05**: Completed, failed, and needs-input states are visually obvious without opening the detail panel.

### 2D Office

- [ ] **OFFICE-01**: Main screen renders a 2D office with desks/avatars for agents.
- [ ] **OFFICE-02**: Each agent has a stable desk position.
- [ ] **OFFICE-03**: Agent color/animation reflects state: idle, working, needs-input, completed, failed.
- [ ] **OFFICE-04**: Each desk exposes a compact status overlay with project, model, context %, and current task.
- [ ] **OFFICE-05**: Adding/removing agents updates the office without full page reload.
- [ ] **OFFICE-06**: The office remains usable on a normal laptop screen.

### Agent Detail

- [ ] **DETAIL-01**: Clicking an agent opens a side panel.
- [ ] **DETAIL-02**: Side panel shows full status fields and recent timeline/log output.
- [ ] **DETAIL-03**: User can send a message/input to the agent when the provider supports it.
- [ ] **DETAIL-04**: User can stop the active run from the side panel.
- [ ] **DETAIL-05**: Detail panel clearly shows whether the agent is done or waiting for attention.

### Real-Time Transport

- [ ] **RT-01**: Backend broadcasts agent state changes to the frontend over WebSocket or SSE.
- [ ] **RT-02**: Frontend reconnects after refresh and reloads current known state.
- [ ] **RT-03**: Log/status updates stream without manual page refresh.

### Provider Boundary

- [ ] **PROV-01**: Code uses a minimal provider interface for status, start, stop, send input, and parse usage.
- [ ] **PROV-02**: Claude Code is the first implemented provider.
- [ ] **PROV-03**: Interface does not force OpenAI/Gemini work in MVP.

### Local Polish

- [ ] **POLISH-01**: Local start/stop/reload flow is documented and reliable enough for daily use.
- [ ] **POLISH-02**: Unavailable model/context/cost fields degrade gracefully without broken UI.
- [ ] **POLISH-03**: Empty, loading, failed, and disconnected states are visually clear.
- [ ] **POLISH-04**: UI copy and labels match the local visual-office concept rather than server/admin terminology.

## v2 / Later

- OpenAI/Codex provider.
- Remote/VPS deployment.
- Dockerized agent sandboxing.
- API key vault and encrypted secret storage.
- Telegram sidecar.
- PWA/mobile install.
- Auth for non-local access.
- Backup/restore/update CLI.
- Agent-to-agent delegation.
- Multi-room layouts, layout editor, visual themes.

## Out of Scope (MVP1)

| Feature | Reason |
|---------|--------|
| Public URL / VPS deployment | The MVP is local; remote access adds security and deployment work before the visual loop is proven. |
| Docker Compose production stack | Not needed for local visual monitoring. |
| Login/auth | Localhost single-owner app; login slows MVP. |
| Encrypted vault | API keys can stay in the user's existing local agent configuration for MVP. |
| Telegram | Useful later, not part of the local visual-office thesis. |
| Multiple providers | Claude-first keeps the implementation narrow. |
| Full repository sandboxing | Important for remote/untrusted use, but out of scope for local MVP. |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| LOCAL-01 | Phase 1 | Pending |
| LOCAL-02 | Phase 1 | Pending |
| LOCAL-03 | Phase 1 | Pending |
| LOCAL-04 | Phase 1 | Pending |
| LOCAL-05 | Phase 1 | Pending |
| AGENT-01 | Phase 2 | Pending |
| AGENT-02 | Phase 2 | Pending |
| AGENT-03 | Phase 2 | Pending |
| AGENT-04 | Phase 2 | Pending |
| AGENT-05 | Phase 2 | Pending |
| AGENT-06 | Phase 2 | Pending |
| AGENT-07 | Phase 2 | Pending |
| STATUS-01 | Phase 3 | Pending |
| STATUS-02 | Phase 3 | Pending |
| STATUS-03 | Phase 3 | Pending |
| STATUS-04 | Phase 3 | Pending |
| STATUS-05 | Phase 3 | Pending |
| OFFICE-01 | Phase 3 | Pending |
| OFFICE-02 | Phase 3 | Pending |
| OFFICE-03 | Phase 3 | Pending |
| OFFICE-04 | Phase 3 | Pending |
| OFFICE-05 | Phase 3 | Pending |
| OFFICE-06 | Phase 3 | Pending |
| DETAIL-01 | Phase 4 | Pending |
| DETAIL-02 | Phase 4 | Pending |
| DETAIL-03 | Phase 4 | Pending |
| DETAIL-04 | Phase 4 | Pending |
| DETAIL-05 | Phase 4 | Pending |
| RT-01 | Phase 4 | Pending |
| RT-02 | Phase 4 | Pending |
| RT-03 | Phase 4 | Pending |
| PROV-01 | Phase 2 | Pending |
| PROV-02 | Phase 2 | Pending |
| PROV-03 | Phase 2 | Pending |
| POLISH-01 | Phase 5 | Pending |
| POLISH-02 | Phase 5 | Pending |
| POLISH-03 | Phase 5 | Pending |
| POLISH-04 | Phase 5 | Pending |

**Coverage:**
- v1 requirements: 37 total
- Mapped to phases: 37
- Unmapped: 0

---
*Requirements updated: 2026-05-13 after local-first pivot*
