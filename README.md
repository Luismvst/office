# Office

Office is a local visual workspace for AI coding agents.

The goal is simple: instead of keeping several terminal windows open and losing track of which agent is working, blocked, finished, or near its context limit, Office shows them as a small 2D office. Each agent gets a desk/avatar with live status, model, context usage, current task, and recent activity.

## Current Direction

MVP1 is local-first.

- Runs on the developer machine.
- Opens in the browser on localhost.
- Focuses on one owner, no login in the first MVP.
- Starts with Claude Code support.
- Prioritizes visual awareness over production deployment.

Remote VPS deployment, Docker production setup, encrypted vaults, Telegram control, multi-user auth, and additional providers are later work.

## MVP Phases

1. **local-app-shell** - local backend + web UI skeleton, localhost-only, persisted app state, empty 2D office screen.
2. **claude-agent-runner** - register local workspaces, start/stop Claude Code sessions, capture lifecycle and output.
3. **visual-office-status** - render agents as desks/avatars with live model/context/status overlays.
4. **agent-detail-and-control** - click an agent for logs/timeline, send input when supported, stop active runs, reconnect safely.
5. **local-polish-and-provider-boundary** - smooth local UX, graceful unavailable metrics, small provider boundary for future providers.

## Product Thesis

The 2D office is the product, not decoration around a dashboard.

Office should make it obvious at a glance:

- Which agents are working.
- Which agents are done.
- Which agents need attention.
- Which model/provider each agent is using.
- Which agents are close to context limits.
- What each agent is currently doing.

## Planning

Planning artifacts live in `.planning/`:

- `.planning/PROJECT.md` - product direction and decisions.
- `.planning/REQUIREMENTS.md` - MVP requirements and traceability.
- `.planning/ROADMAP.md` - phased execution plan.
- `.planning/STATE.md` - current project state.

## Status

The project has been scoped and pivoted to the local-first MVP. Implementation has not started yet.
