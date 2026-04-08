# Security

## Threat model

### What we protect
- SQLite database containing agent events, approval decisions, and cost data
- Credential files (OAuth tokens for Gmail, OpenAI, Google Calendar)
- Agent session logs (JSONL files with conversation content)
- System configuration (`openclaw.json` - API keys, bot tokens, channel bindings)
- Dashboard API endpoints from unauthorized access

### From whom
- Unauthorized network access (other devices on the LAN)
- Accidental exposure of secrets via git or logs
- Malformed or malicious event payloads injected into the ingest endpoint

### Assumptions
- The dashboard runs on a local machine behind a home network - not exposed to the public internet
- Tailscale VPN provides encrypted access when remote
- The macOS user account is the sole operator (single-tenant)
- Agents communicate via the gateway on localhost:18789

## Controls

### API key authentication
- The ingest endpoint (`POST /api/events/ingest`) requires `x-api-key` header matching `MISSION_CONTROL_API_KEY` environment variable
- QA endpoint (`POST /api/qa/run`) requires the same API key
- All mutation endpoints (approvals, workflow actions, settings) validate the key

### Network binding
- Gateway binds to loopback only (`127.0.0.1:18789`) - never exposed externally
- Dashboard binds to `0.0.0.0:3001` for local network access (needed for mobile/tablet viewing via Tailscale)
- No public DNS, no port forwarding, no Cloudflare Tunnel to the dashboard

### Zod validation
- All inbound events are validated against a Zod schema before insertion
- Invalid payloads are routed to a `dead_letter_events` table with the raw payload and failure reason
- API inputs (approval actions, workflow controls, budget updates) are schema-validated

### Gateway authentication
- Gateway uses token-based auth: `Authorization: Bearer <token>` required for all gateway API calls
- Token stored in `openclaw.json` - not committed to any repository

### Coordinator guardrails
- PAUSE flag (`~/.openclaw/.claw/PAUSE`) immediately halts all agent turns
- Budget limits can trigger auto-pause when daily or monthly USD thresholds are breached
- Approval queue forces master authorization for high-risk agent actions

## Secrets and access

| Secret | Location | Exposure |
|--------|----------|----------|
| MISSION_CONTROL_API_KEY | `.env.local` | Server-side only (never in client bundle) |
| Gateway auth token | `openclaw.json` | Local filesystem only |
| Telegram bot token | `openclaw.json` | Local filesystem only |
| Discord bot token | `openclaw.json` | Local filesystem only |
| Notion API key | `openclaw.json` | Local filesystem only |
| OAuth tokens (Gmail, Google Calendar) | `~/.openclaw/credentials/` | Encrypted at rest by provider |
| OpenAI API credentials | `~/.openclaw/credentials/` | OAuth flow managed by OpenClaw CLI |

**Git safety:** `.env.local`, `.data/`, `credentials/`, `openclaw.json` are all gitignored. Only `.env.example` is committed.

## Limitations

### No authentication on the dashboard UI
The dashboard itself has no login page or session management. Access control relies entirely on network-level protection (local network + Tailscale VPN). Anyone on the same network can view all dashboard pages.

### API key is shared across all endpoints
A single `MISSION_CONTROL_API_KEY` is used for all authenticated endpoints. There is no per-agent or per-role access control.

### SQLite database is not encrypted
The SQLite file (`.data/mc.db`) is stored in plaintext on disk. It contains event payloads, cost data, and approval decisions. Protection relies on macOS filesystem permissions.

### No rate limiting on dashboard endpoints
Dashboard API routes used for polling (status, events, agents) have no rate limiting. A misconfigured client could hammer the server. Mitigated by the adaptive polling system that reduces frequency when idle.
