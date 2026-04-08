# Decisions

## Decision log

- 2025-12-XX: **Next.js 15 (App Router) over alternatives** - Needed server components for direct SQLite access, API routes for the 25+ endpoints, and the App Router's layout system for the sidebar-based dashboard. Alternatives considered: Remix (better server-first model but smaller ecosystem for dashboards), plain Express+React (more setup, no SSR benefits), Grafana (too opinionated for custom agent workflows).

- 2025-12-XX: **SQLite over PostgreSQL** - Single-user, single-node deployment on macOS. WAL mode gives concurrent reads without a database server. Event volume is hundreds/day, not thousands/second. Tradeoff: no replication, no clustering, single-writer bottleneck. Acceptable for a personal operations dashboard.

- 2025-12-XX: **Server components for most pages** - Pages like `/overview`, `/costs`, `/logs-events`, and `/agents` query SQLite synchronously via `better-sqlite3` in server components. This avoids client-side fetch waterfalls and keeps the JS bundle small. Only real-time panels (system status, session stream) are client components with polling.

- 2025-12-XX: **Tailwind CSS for styling** - Dark theme with precise pixel control (#0a0a0a bg, #ededed text, 9-10px labels, 11px body). Utility classes match the minimal, high-contrast design language. No component library - 8 hand-built UI primitives (Card, Button, Badge, Sidebar, StatusDot, Table, Tabs, PageHeader).

- 2025-12-XX: **Recharts for data visualization** - Needed sparklines, area charts, bar charts, and pie charts. Recharts is React-native, composable, and SSR-compatible. Lighter than D3 for dashboard charts. Sparkline component is custom-built on top of Recharts SVG.

- 2025-12-XX: **LaunchAgents over Docker for production** - macOS LaunchAgents provide auto-start on login, crash recovery via `KeepAlive`, and zero-overhead process management. The dashboard and coordinator each have their own plist. Docker Compose exists for portability but isn't used in daily operation.

- 2025-12-XX: **Sonner for toast notifications** - Lightweight, accessible, and styled to match the dark theme. Used for workflow action confirmations, approval decisions, and error feedback.

- 2026-01-XX: **Adaptive polling over WebSocket** - Client components poll API routes at 10-30s intervals based on activity level (high/medium/low). This avoids WebSocket connection management complexity and works behind reverse proxies. The adaptive system reduces polls when the system is idle.

- 2026-01-XX: **SSE for session streaming** - Agent session files are JSONL on disk. SSE (`/api/sessions/stream`) tails these files and pushes new lines to the browser. Simpler than WebSocket for unidirectional data flow, natively supported by `EventSource`.

- 2026-02-XX: **Recommendation plugins** - The projects page generates AI-powered recommendations (failure rate, cost optimization, duration trends) via a plugin system. Each plugin calculates severity and message from project metrics. Extensible without modifying the repository layer.

- 2026-03-XX: **Portuguese UI with English code** - All dashboard labels, navigation, and user-facing text are in Portuguese (pt-PT). All code, comments, variable names, API responses, and documentation are in English. This matches the owner's daily workflow language.

## Alternatives considered

- **Grafana + Prometheus** - Rejected because the data model (structured events, approval queues, agent memory) doesn't map cleanly to time-series metrics. Would require a custom data source plugin and lose the agent-specific features.

- **Plain Express + React SPA** - Rejected in favor of Next.js for server component benefits (direct SQLite queries, no API layer for read-heavy pages). Express would add a separate server process to manage.

- **PostgreSQL** - Rejected for deployment complexity on a single Mac. Would need a running database server, migrations tool, and more infrastructure for a system that peaks at ~200 events/day.

- **WebSocket for real-time** - Rejected in favor of adaptive polling + SSE. WebSocket adds connection lifecycle complexity, doesn't provide meaningful latency improvement for a 10-30s refresh dashboard, and is harder to debug.

- **Prisma ORM** - Rejected in favor of raw `better-sqlite3` queries. The repository module (~1300 lines) uses prepared statements directly, avoiding an ORM abstraction layer and keeping SQLite specifics (json_extract, datetime arithmetic) explicit.

## Open questions

- Migrate to neon.tech PostgreSQL for remote access and multi-device support?
- Add i18n toggle (PT/EN) to the dashboard, or keep PT-only?
- Implement authentication (currently relies on local network / Tailscale access)?
