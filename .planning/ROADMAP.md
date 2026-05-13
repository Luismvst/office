# Roadmap: Office - Local Agent Office

## Overview

MVP1 is now local-first. The goal is to prove the actual product loop quickly: replace multiple opaque terminal windows with a visual 2D office where the owner can see agents working, finished, blocked, or near context limits.

The previous VPS/Docker/vault/Telegram roadmap is superseded. Those ideas are still valid later, but they should not block the local visual product.

## Phases

- [ ] **Phase 1: local-app-shell** - Local backend + web app skeleton, localhost-only, persisted app state, empty 2D office screen
- [ ] **Phase 2: claude-agent-runner** - Register local workspaces, start/stop Claude Code sessions, capture lifecycle and output
- [ ] **Phase 3: visual-office-status** - Render agents as desks/avatars with live model/context/status overlays
- [ ] **Phase 4: agent-detail-and-control** - Click an agent for logs/timeline, send input when supported, stop active run, reconnect safely
- [ ] **Phase 5: local-polish-and-provider-boundary** - Smooth local UX, provider interface cleanup, graceful unavailable metrics, prepare OpenAI later

## Phase Details

### Phase 1: local-app-shell
**Goal**: Owner runs one local command and sees the Office web UI: an empty 2D office shell served from localhost with local persistence ready.
**Mode:** mvp
**Depends on**: Nothing
**Requirements**: LOCAL-01, LOCAL-02, LOCAL-03, LOCAL-04, LOCAL-05
**Success Criteria**:
  1. A documented local command starts backend and frontend.
  2. The app binds to localhost by default.
  3. Browser shows the office screen without login.
  4. A local persistence layer exists and can store app settings/configured agents.
  5. A health/status endpoint reports backend readiness.
**Plans**: TBD
**UI hint**: yes

### Phase 2: claude-agent-runner
**Goal**: Owner can register a local workspace, start a Claude Code agent session, and stop it from the app while Office tracks process state and output.
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: AGENT-01, AGENT-02, AGENT-03, AGENT-04, AGENT-05, AGENT-06, AGENT-07, PROV-01, PROV-02, PROV-03
**Success Criteria**:
  1. User registers a local workspace path.
  2. User starts a Claude Code agent session for that workspace.
  3. Backend tracks lifecycle state and recent output.
  4. UI can stop a running session.
  5. Metadata includes provider, model when available, workspace, task, start time, last activity, and exit status.
**Plans**: TBD
**UI hint**: yes

### Phase 3: visual-office-status
**Goal**: The main screen becomes the product: each live agent appears as a desk/avatar with glanceable visual status, model, context, task, and attention state.
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: STATUS-01, STATUS-02, STATUS-03, STATUS-04, STATUS-05, OFFICE-01, OFFICE-02, OFFICE-03, OFFICE-04, OFFICE-05, OFFICE-06
**Success Criteria**:
  1. Agents render as desks/avatars in a 2D office.
  2. Agent state is visible through color/animation.
  3. Status overlay shows project, provider/model, context %, task, and last activity where available.
  4. Completed, failed, and needs-input states are obvious at a glance.
  5. Adding/removing agents updates without a full reload.
**Plans**: TBD
**UI hint**: yes

### Phase 4: agent-detail-and-control
**Goal**: Owner clicks any agent to inspect logs/timeline, send input where supported, and stop active work without returning to terminal windows.
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: DETAIL-01, DETAIL-02, DETAIL-03, DETAIL-04, DETAIL-05, RT-01, RT-02, RT-03
**Success Criteria**:
  1. Clicking an agent opens a side panel.
  2. Panel shows full status and recent output.
  3. User can send input/message when provider support exists.
  4. User can stop the active run.
  5. UI receives live updates and reconnects after refresh.
**Plans**: TBD
**UI hint**: yes

### Phase 5: local-polish-and-provider-boundary
**Goal**: Make the local MVP comfortable enough for daily use and leave a clean path for OpenAI/Codex later without implementing it yet.
**Mode:** mvp
**Depends on**: Phase 4
**Requirements**: POLISH-01, POLISH-02, POLISH-03, POLISH-04
**Success Criteria**:
  1. Unavailable model/context/cost fields degrade gracefully.
  2. Empty, loading, failed, and disconnected states are visually clear.
  3. UI labels match the local visual-office concept rather than server/admin terminology.
  4. Local start/stop/reload flow is reliable enough for daily use.
**Plans**: TBD
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4 -> 5

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. local-app-shell | 0/TBD | Not started | - |
| 2. claude-agent-runner | 0/TBD | Not started | - |
| 3. visual-office-status | 0/TBD | Not started | - |
| 4. agent-detail-and-control | 0/TBD | Not started | - |
| 5. local-polish-and-provider-boundary | 0/TBD | Not started | - |

## Notes for Plan-Phase

- Phase 1 should avoid auth, Docker, Caddy, public URL, vault, and Telegram.
- Phase 2 should prove one real Claude Code session before expanding abstractions.
- Phase 3 is the key product phase; spend design effort there because the visual office is the thesis.
- Keep provider abstraction minimal. Do not let future OpenAI/Gemini needs slow the Claude-first local MVP.
- Treat prior Docker/VPS research as background only, not binding plan input.

---
*Roadmap updated: 2026-05-13 after local-first pivot*
