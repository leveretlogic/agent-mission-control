# Agent Mission Control

Operations dashboard for a 5-agent AI orchestration platform (OpenClaw), built with Next.js 15 and SQLite — featuring real-time KPIs, agent session tracing, cron/coordinator monitoring, approval queues, and budget controls.

**Status:** Active
**Live:** Not public (local network on port 3001) · **Docs:** [/docs](/docs) · **Demo:** [/docs/assets/demo.gif](/docs/assets/demo.gif)

## Demo

![Demo](/docs/assets/demo.gif)

## Why this exists

- Running 5 AI agents (4 cloud GPT-5.2 + 1 local Qwen 2.5 7B) across Telegram and Discord with no visibility into cost, failures, or session state — decisions were made blind
- The coordinator loop runs every 5 minutes and cron triggers 8 scheduled jobs daily — incidents, stale approvals, and budget overruns needed a single pane of glass
- Wanted a real operations dashboard (not just logs) that tracks KPIs, surfaces anomalies, and provides actionable controls like pause/resume/rerun — all from a browser

## Key features

- Executive KPI panel with real-time cost tracking, health score, anomaly detection, and sparkline trends
- Agent Office: visual presence map showing active sessions per agent with live polling
- Coordinator & cron monitoring: job status, last run duration, consecutive errors, and gateway health
- Approval queue with risk-level badges, expiration timers, and master authorization workflow
- Per-agent 24h metrics, live SSE session tailing, session timeline visualization, and MEMORY.md viewer
- Budget controls: daily/monthly limits in USD, auto-pause on breach, per-agent cost breakdown
- Agent topology graph: delegation relationships and subagent run visualization
- Customizable dashboard: drag-and-drop widget configuration persisted in SQLite

## Tech stack

- Frontend: Next.js 15 (App Router), React 18, TypeScript 5.6, Tailwind CSS 3.4, Recharts, Lucide icons
- Backend: Next.js API routes (25+ endpoints), Server-Sent Events (SSE) for live streaming
- Data: SQLite (better-sqlite3, WAL mode), 14 tables, Zod schema validation
- Scripts: Node.js coordinator loop (20 kB), usage sync, retention rotation, backup/restore
- Infra: macOS LaunchAgents (auto-start), Docker Compose (app + worker), Sonner toast notifications

## Architecture (quick view)

- Server components query SQLite directly for most pages; client components poll API routes (10-30s intervals) for real-time data like agent sessions and system status
- A coordinator loop runs every 5 minutes via LaunchAgent, orchestrating agent turns, emitting events to SQLite, and enforcing budget guardrails
- 25+ API routes power the dashboard — from `/api/openclaw/status` (live gateway/cron/session state) to `/api/sessions/stream` (SSE agent session tailing)

See: [/docs/architecture.md](/docs/architecture.md)

## Getting started

```bash
# Clone the repository
git clone https://github.com/leveretlogic/agent-mission-control.git

# Install dependencies
cd agent-mission-control && npm install

# Configure environment
cp .env.example .env.local
# Set OPENCLAW_ROOT, MISSION_CONTROL_API_KEY, SQLITE_PATH

# Run development server (hot-reload)
./start.sh dev
# Dashboard available at http://localhost:3001

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

MIT - see [LICENSE](LICENSE)
