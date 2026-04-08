# Deployment

## Environments

| Environment | URL | Trigger | Status |
|-------------|-----|---------|--------|
| Production | http://localhost:3001 | LaunchAgent (auto-start on login) | Active |
| Development | http://localhost:3001 | `./start.sh dev` (hot-reload) | Manual |
| Docker | http://localhost:3000 | `docker compose up` | Available |

## Requirements

- macOS (darwin arm64) — LaunchAgents require macOS
- Node.js 18+ (LTS)
- npm or pnpm
- OpenClaw root directory (`~/.openclaw/`) with configured agents
- SQLite database created on first run (`.data/mc.db`)

## Production (LaunchAgents)

Two macOS LaunchAgents manage the services. They auto-start on login and restart on crash.

### Dashboard service
- **Plist:** `~/Library/LaunchAgents/com.openclaw.dashboard.plist`
- **Runs:** `npx next start -p 3001 -H 0.0.0.0`
- **Log:** `.data/app.log`

### Coordinator service
- **Plist:** `~/Library/LaunchAgents/com.openclaw.coordinator.plist`
- **Runs:** `node scripts/coordinator-loop.mjs`
- **Log:** `.data/coordinator/coordinator.log`
- **Cycle:** Every 5 minutes
- **State:** `.data/coordinator_state.json`

### Service management (via start.sh)

```bash
# Check current status
./start.sh               # Shows dashboard/coordinator status

# Restart production services
./start.sh restart        # Unloads + reloads both LaunchAgents

# Stop all services
./start.sh stop           # Unloads LaunchAgents, kills processes

# Development mode (hot-reload, stops LaunchAgents)
./start.sh dev            # Next.js dev server + coordinator
```

## Development

```bash
# Start dev mode (unloads LaunchAgents to avoid port conflict)
./start.sh dev

# Or manually:
npm run dev -- -p 3001    # Next.js dev server
node scripts/coordinator-loop.mjs  # Coordinator (separate terminal)
```

### Environment variables (`.env.local`)
```
OPENCLAW_ROOT=/Users/franciscocoelho/.openclaw
MISSION_CONTROL_API_KEY=<api-key>
SQLITE_PATH=.data/mc.db
PORT=3001
```

## Docker Compose

For portable deployment outside macOS:

```bash
cd infra/
MISSION_CONTROL_API_KEY=<key> docker compose up -d
# Dashboard: http://localhost:3000
# Worker polls events from inbox directory
```

Services:
- **app** — Next.js production server (port 3000)
- **worker** — Event ingestion worker (polls `.data/events/inbox/`)
- **volume** — `mission_data` for shared SQLite database

## Build and verify

```bash
# Production build (verify before restart)
npm run build

# Restart production safely
scripts/safe-build-restart.sh

# Or manual restart
kill $(lsof -ti:3001)
npx next start -p 3001 -H 0.0.0.0 &
```

## Rollback

1. The SQLite database has automated backups: `npm run backup:sqlite`
2. To restore: `npm run restore:test` (verifies backup integrity)
3. For code rollback: `git revert <commit>`, then `npm run build && ./start.sh restart`
4. Coordinator state can be reset by deleting `.data/coordinator_state.json` (restarts cycle count)
