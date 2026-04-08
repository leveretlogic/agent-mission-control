# Testing

## Strategy

Automated QA via scheduled health checks with manual build verification before deploys. The testing approach prioritizes runtime validation (real system health) over isolated unit tests, fitting the single-operator dashboard context.

| Layer | Tool | Scope | Trigger |
|-------|------|-------|---------|
| QA health checks | `/api/qa/run` | API contracts, database integrity, service health | Cron (daily 06:00 + 09:01 Lisbon) |
| Build verification | `npm run build` | TypeScript compilation, import resolution, page generation | Before every production restart |
| Manual smoke tests | Browser | Page loads, navigation, data rendering, real-time updates | After major changes |

## How to run

```bash
# QA health check (all API contracts)
curl -s -X POST http://localhost:3001/api/qa/run \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: <MISSION_CONTROL_API_KEY>'

# Build verification (TypeScript + Next.js)
npm run build

# Safe build + restart (production)
scripts/safe-build-restart.sh
```

## Automated QA system

Two cron jobs trigger QA checks daily:

### 06:00 — Silent QA (Jarbas agent)
- Runs `POST /api/qa/run` via agent turn
- If `ok: true` → no notification (silent pass)
- If `ok: false` → sends Telegram alert to Master with the failing check

### 09:01 — Verbose QA (Jarbas agent)
- Same endpoint, but always reports to Master
- GREEN: summary + 1 suggested improvement (for approval)
- RED: alert with evidence + proposed fix (without applying)
- Fix proposals require explicit Master approval before execution

### QA checks include
- API endpoint reachability (health, status, events, approvals)
- Database query execution (events, agents, workflows tables)
- Coordinator state integrity
- Budget configuration validity
- Response shape validation (JSON structure contracts)

## Coverage

No formal code coverage tooling. Coverage is achieved through:
- **API contracts:** QA system validates response shapes for critical endpoints
- **Build-time checks:** TypeScript strict mode catches type errors across ~1,300 lines of repository code
- **Runtime validation:** Zod schemas validate all inbound events and API inputs

## Notes

- QA results are persisted as `qa_run` events in the SQLite events table
- Failed QA checks automatically open incidents (`incident_opened` events) with deduplication
- The dashboard's `/api/qa/run` endpoint returns structured JSON with per-check results
- Build verification is the primary gate before production restarts — TypeScript errors block deployment
- No E2E browser tests (Playwright) yet — visual validation is manual
