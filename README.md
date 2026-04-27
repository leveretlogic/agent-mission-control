# Agent Mission Control

A reliability-first operations dashboard for multi-agent AI systems — approval gates, traceability, health checks, and budget guardrails for 5 agents running across Telegram and Discord. Built with Next.js 15 and SQLite.

**Status:** Active · **Stack:** Next.js 15 · TypeScript · SQLite (14 tables, WAL) · 25+ API routes · SSE · LaunchAgents
**Live:** Local network + Tailscale tunnel (private) · **Docs:** [/docs](/docs) · **Demo:** [/docs/assets/demo.gif](/docs/assets/demo.gif)
**Companion repo:** [ai-reliability-patterns](https://github.com/leveretlogic/ai-reliability-patterns) — the patterns extracted from this project, isolated and reusable.

## Demo

![Demo](/docs/assets/demo.gif)

## The problem

Running 5 AI agents (4 cloud GPT-5.2 + 1 local Qwen 2.5 7B) across Telegram and Discord, I had no real visibility into what they were doing. Logs scrolled past, costs accrued silently, approvals piled up with no SLA, and the most dangerous failure mode — *a green status with the wrong outcome* — was almost impossible to catch in time. The coordinator loop runs every 5 minutes; cron fires 8 scheduled jobs daily; the gateway can fail; an agent can drift. Without a single pane of glass, every operational decision was made blind.

## The solution

Mission Control treats agent operations as an ops problem first and a UI problem second. Every agent action is a structured, Zod-validated event written to SQLite (14 tables, WAL mode). Risky actions go through an approval queue with risk-tiered SLAs — high-risk approvals expire in 90 minutes and double-confirmation is required outside the 09:00–19:00 operational window. Health checks run continuously: a gateway watchdog escalates after 3 consecutive failures via a side-channel that bypasses the failed dependency, and a cron-freshness check fires when no scheduled job has run in over 2 hours during business hours.

The dashboard turns that event stream into the controls a human operator actually needs: real-time KPIs, per-agent 24h metrics, live SSE session tailing, a topology graph of delegations, daily/monthly budget limits with auto-pause on breach, and customizable widgets. Server components query SQLite directly for most pages; client components poll API routes at 10–30s intervals for the panels that need to be live. The whole thing runs on macOS LaunchAgents for native auto-start and crash recovery, with a Docker Compose option for portability.

## Numbers

- 5 agents under supervision (4 cloud, 1 local)
- 25+ API routes, 14 SQLite tables, 12 indexes
- Coordinator loop every 5 minutes; 8 cron jobs/day
- Approval SLAs: high = 90 min, medium = 20 min, low = no expiry
- Gateway watchdog: incident at 3 consecutive failures, side-channel alert at 3+
- Cron-freshness alarm: 2h stale (incident), 3h stale (direct alert)

## Reliability patterns inside

These are the load-bearing patterns — extracted as standalone snippets in [ai-reliability-patterns](https://github.com/leveretlogic/ai-reliability-patterns):

- **Approval gate with risk-tiered SLA** — risk levels drive expiration, double-confirmation, and audit events ([`lib/approvals/service.ts`](lib/approvals/service.ts), [`lib/policies/risk.ts`](lib/policies/risk.ts))
- **Health-watchdog with side-channel escalation** — consecutive-failure threshold + dedup + alert path that bypasses the failed dependency ([`scripts/coordinator-loop.mjs`](scripts/coordinator-loop.mjs))
- **Heartbeat / freshness check** — "is anything actually running?" detection based on last-run timestamps and business-hour windows ([`scripts/coordinator-loop.mjs`](scripts/coordinator-loop.mjs))
- **Stale-fallback / SLA expiry** — pending approvals auto-expire and open an incident instead of hanging forever ([`app/api/approvals/expire-check/route.ts`](app/api/approvals/expire-check/route.ts))

## Key features

- Executive KPI panel: real-time cost, health score, anomaly detection, sparkline trends
- Agent Office: presence map showing active sessions per agent, live polling
- Coordinator & cron monitoring: job status, last-run duration, consecutive errors, gateway health
- Approval queue: risk badges, expiration timers, master authorization workflow
- Per-agent 24h metrics, live SSE session tailing, session timeline, MEMORY.md viewer
- Budget controls: daily/monthly USD limits, auto-pause on breach, per-agent cost breakdown
- Agent topology graph: delegation relationships and subagent run visualization
- Customizable dashboard: drag-and-drop widget configuration persisted in SQLite

## Tech stack

- **Frontend:** Next.js 15 (App Router), React 18, TypeScript 5.6, Tailwind CSS 3.4, Recharts, Lucide
- **Backend:** Next.js API routes (25+ endpoints), Server-Sent Events for live streaming
- **Data:** SQLite (better-sqlite3, WAL mode), 14 tables, Zod schema validation
- **Scripts:** Node.js coordinator loop, usage sync, retention rotation, backup/restore
- **Infra:** macOS LaunchAgents (auto-start), Docker Compose (app + worker), Sonner toasts

## Architecture (quick view)

- Server components query SQLite directly for most pages; client components poll API routes (10–30s) for panels that need real-time data (sessions, status, costs)
- A coordinator loop runs every 5 minutes via LaunchAgent — it orchestrates agent turns, emits structured events to SQLite, runs health/freshness watchdogs, and enforces budget guardrails
- 25+ API routes power the dashboard, from `/api/openclaw/status` (live gateway/cron/session state) to `/api/sessions/stream` (SSE agent session tailing)

See: [/docs/architecture.md](/docs/architecture.md)

## Getting started

```bash
# Clone the repository
git clone https://github.com/leveretlogic/agent-mission-control.git
cd agent-mission-control

# Install dependencies
npm install

# Configure environment
cp .env.example .env.local
# Set OPENCLAW_ROOT, MISSION_CONTROL_API_KEY, SQLITE_PATH

# Run development server (hot reload)
./start.sh dev
# Dashboard at http://localhost:3001

# Or run with npm directly
npm run dev -- -p 3001
```

## Quality

- **Tests:** Automated QA health checks via `/api/qa/run` (daily cron at 06:00 + 09:01 Lisbon time)
- **CI:** Manual build verification (`npm run build`) before production restart
- **Known limits:**
  - Dashboard UI is in Portuguese (pt-PT) — no i18n toggle yet
  - SQLite is single-node (no replication or clustering)
  - SSE session stream reads JSONL files from disk — high session volume may lag
  - Budget cost estimation falls back to per-event approximation when agents don't emit `total_cost_usd`

## Screenshots

| Description | Screenshot |
|-------------|------------|
| Overview — executive KPIs and system status | ![Screenshot](/docs/assets/screenshot-01.png) |
| Agent Office — presence map and session counts | ![Screenshot](/docs/assets/screenshot-02.png) |
| Logs & Events — filterable timeline with payload preview | ![Screenshot](/docs/assets/screenshot-03.png) |

## License

MIT — see [LICENSE](LICENSE)
