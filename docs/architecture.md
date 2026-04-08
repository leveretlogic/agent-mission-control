# Architecture

## Overview

A Next.js 15 App Router application running on port 3001 (macOS, managed by LaunchAgents). The dashboard provides real-time observability for a 5-agent AI orchestration platform (OpenClaw). Most pages are server components that query SQLite directly; client components poll API routes at 10-30s intervals for live data (agent sessions, system status, cost updates). A coordinator loop (`coordinator-loop.mjs`) runs every 5 minutes via a separate LaunchAgent, orchestrating agent turns, emitting structured events, and enforcing budget guardrails.

## Project structure

```
workspace/mission-control/
├── app/
│   ├── (dashboard)/              # All page routes (grouped layout)
│   │   ├── overview/page.tsx     # Executive KPIs, system hero, sparklines
│   │   ├── office/page.tsx       # Agent presence map, live session counts
│   │   ├── workflows/page.tsx    # Cron jobs, coordinator status, gateway health
│   │   ├── costs/page.tsx        # Cost breakdown by agent, daily/monthly charts
│   │   ├── logs-events/page.tsx  # Filterable event timeline with payload preview
│   │   ├── projects/page.tsx     # System operations tracking, efficiency scores
│   │   ├── notificacoes/page.tsx # Approval queue (master authorization)
│   │   ├── calendario/page.tsx   # Today/tomorrow timeline + cron schedule
│   │   ├── agents/
│   │   │   ├── page.tsx          # Per-agent 24h metrics
│   │   │   ├── stream/          # Live SSE session tailing
│   │   │   ├── trace/           # Session timeline visualization
│   │   │   ├── memory/          # Agent MEMORY.md viewer
│   │   │   ├── budget/          # Budget limits + cost breakdown
│   │   │   ├── topology/        # Delegation graph visualization
│   │   │   └── corrections/     # Mid-flight agent redirect
│   │   ├── settings/page.tsx    # System configuration
│   │   ├── prompts/             # System prompt management
│   │   └── secrets/             # Credential viewer
│   │   └── layout.tsx           # Sidebar navigation (10 items, 2 groups)
│   ├── api/                      # 25+ API routes
│   │   ├── openclaw/status/     # Live system status: gateway, cron, sessions
│   │   ├── events/ingest/       # POST endpoint for structured event ingestion
│   │   ├── health/              # Simple health check
│   │   ├── stats/budget/        # Budget status + configuration
│   │   ├── stats/costs/breakdown/ # Per-agent/per-type cost breakdown
│   │   ├── sessions/stream/     # SSE stream of agent session lines
│   │   ├── approvals/           # Approval queue management
│   │   ├── qa/run/              # QA health check trigger
│   │   └── ...                  # 17 more endpoints
│   ├── globals.css              # Base styles (dark theme)
│   └── layout.tsx               # Root layout
├── components/
│   ├── dashboard/               # Domain-specific components
│   │   ├── system-hero.tsx      # Architecture visualization (SVG)
│   │   ├── openclaw-status-panel.tsx # Live gateway/cron status
│   │   ├── topology-graph.tsx   # Agent delegation graph
│   │   ├── approvals-panel.tsx  # Approval queue with risk badges
│   │   ├── customize-dashboard.tsx  # Widget configuration
│   │   ├── memory-search-panel.tsx  # Agent memory search
│   │   └── mid-flight-correction-panel.tsx # Agent redirect
│   ├── charts/                  # Recharts-based visualizations
│   │   ├── sparkline.tsx        # Inline mini-charts for KPI cards
│   │   ├── area-chart.tsx       # Time series charts
│   │   ├── bar-chart.tsx        # Agent comparison charts
│   │   └── activity-chart-panel.tsx / cost-chart-panel.tsx / ...
│   ├── tables/                  # Data tables
│   └── ui/                      # Reusable primitives (8 components)
│       ├── Sidebar.tsx          # Collapsible sidebar with icon navigation
│       ├── Card.tsx / Button.tsx / Badge.tsx / ...
│       └── PageHeader.tsx / StatusDot.tsx / Table.tsx / Tabs.tsx
├── lib/
│   ├── db/
│   │   ├── schema.sql           # 14 tables, 12 indexes (WAL mode)
│   │   ├── client.ts            # SQLite connection (better-sqlite3)
│   │   └── repository.ts        # All queries (~1300 lines, 30+ functions)
│   ├── agents/config.ts         # Agent registry (5 agents, positions, colors)
│   ├── events/schema.ts         # Zod-validated event schema
│   ├── events/mapper.ts         # Event → DB row transformation
│   ├── plugins/registry.ts      # Recommendation plugins
│   ├── settings/repository.ts   # Feature settings persistence
│   ├── dashboard/config.ts      # Widget configuration
│   └── ...                      # 20+ library modules
├── scripts/
│   ├── coordinator-loop.mjs     # Main coordinator (20 kB, runs every 5 min)
│   ├── usage-sync.mjs           # Token/cost synchronization
│   ├── retention-rotate.mjs     # Event retention management
│   ├── backup-sqlite.mjs        # Database backup
│   └── run-agent-turn.mjs       # Single agent turn execution
├── infra/
│   ├── docker-compose.yml       # App + Worker containers
│   ├── Dockerfile.app           # Next.js production image
│   └── Dockerfile.worker        # Event ingestion worker
├── start.sh                     # Startup script (status/dev/stop/restart)
└── .data/                       # SQLite DB + config files (gitignored)
```

## Data flow

### Event ingestion pipeline
```
Agent runs task
    ↓
Gateway emits structured event (JSON)
    ↓ POST /api/events/ingest
Zod validation → Dead letter queue (if invalid)
    ↓
SQLite INSERT (events table, WAL mode)
    ↓
Dashboard reads via server component or API poll
```

### Coordinator cycle (every 5 minutes)
```
LaunchAgent triggers coordinator-loop.mjs
    ↓
1. Check PAUSE flag (~/.openclaw/.claw/PAUSE)
2. Check budget limits (daily/monthly USD)
3. Read coordinator state (cycle count, dedup)
4. Execute pending agent turns
5. Emit job_started/job_finished/job_failed events
6. Update coordinator state
    ↓
Dashboard auto-refreshes via adaptive polling
```

### Real-time data flow
```
Client component mounts
    ↓
Poll API route (10-30s interval, adaptive)
    ↓
API reads: SQLite (events/approvals)
           Filesystem (cron/jobs.json, agent sessions, gateway health)
           Process state (launchctl, port probes)
    ↓
JSON response → React state update → UI render
```

## Key decisions

- **SQLite over PostgreSQL** - Single-node deployment on macOS; WAL mode provides concurrent reads without a database server. 14 tables with 12 indexes handle the event volume (hundreds/day, not thousands/second).
- **Server components by default** - Most dashboard pages query SQLite directly in server components, avoiding client-side fetch waterfalls and reducing JavaScript bundle. Only real-time panels (status, sessions) use client-side polling.
- **LaunchAgents over Docker (production)** - macOS LaunchAgents provide auto-start on login, crash recovery, and native process management. Docker Compose exists for portability but isn't the primary deployment mode.
- **Portuguese UI, English code** - Dashboard labels and navigation are in Portuguese (pt-PT) to match the owner's preference. All code, comments, and API responses use English.

## Diagrams

### Database schema (core tables)

```
events (id, timestamp, type, source, severity, actor, riskLevel, payload, ...)
    ↓ actor
agents (id, name, status)
    ↓ workflowId
workflows (id, name, status, retries)
    ↓ id
workflow_actions_audit (workflow_id, action, actor, risk_level, ...)

approvals (id, action, status, riskLevel, actor, target, expires_at, ...)
incidents (id, title, status, severity, opened_at, closed_at)
daily_rollups (day, total_events, success_rate, mttr_minutes, ...)

usage_cycles (cycle_id, start_datetime, total_quota, total_used, ...)
usage_events (event_id, model_name, tokens_input, tokens_output, ...)

project_jobs (id, project_slug, status, kind, quota_impact_estimate, ...)
    ↓ job_id
job_outputs (id, job_id, agent_id, model, text, raw_json, ...)

project_activity (id, project_id, type, actor, message, ...)
settings (key, value)
```
