# RemitLend Operations Runbook

Single source of truth for running and deploying RemitLend. Accuracy is prioritized over breadth; everything below is grounded against the actual `remitlend-backend`, `remitlend-frontend`, and `remitlend-contracts` repositories.

---

## 1. Environment Variables

Copy `remitlend-backend/.env.example` to `.env` and fill in values. All secrets must be strings.

### 1.1 remitlend-backend

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DATABASE_URL` | Yes | | PostgreSQL connection string (`postgres://user:pass@host:5432/dbname`). |
| `REDIS_URL` | No | `redis://localhost:6379` | Redis connection for caching and rate limiting. |
| `STELLAR_RPC_URL` | Yes | | Soroban RPC endpoint. |
| `STELLAR_NETWORK` | Yes | | Network identifier (`testnet` or `public`). |
| `LOAN_MANAGER_CONTRACT_ID` | Yes | | `loan_manager` contract ID. |
| `LENDING_POOL_CONTRACT_ID` | Yes | | `lending_pool` contract ID. |
| `REMITTANCE_NFT_CONTRACT_ID` | Yes | | `remittance_nft` contract ID. |
| `MULTISIG_GOVERNANCE_CONTRACT_ID` | Yes | | `multisig_governance` contract ID. |
| `INTERNAL_API_KEY` | Yes | | Shared secret for internal service-to-service auth; passed via `x-api-key` header. |
| `JWT_SECRET` | Yes | | Secret used to sign/verify JWTs. |
| `WEBHOOK_REQUEST_TIMEOUT_MS` | No | `5000` | Timeout for outbound webhook HTTP calls. |
| `SENDGRID_API_KEY` | Yes | | SendGrid API key for email. |
| `FROM_EMAIL` | Yes | | Sender address for user-facing email. |
| `ADMIN_EMAIL` | Yes | | Admin address for operational alerts. |
| `TWILIO_ACCOUNT_SID` | Yes | | Twilio account SID for SMS. |
| `TWILIO_AUTH_TOKEN` | Yes | | Twilio auth token. |
| `TWILIO_PHONE_NUMBER` | Yes | | Outbound Twilio phone number. |
| `PORT` | No | `3000` | HTTP listen port. |
| `INDEXER_POLL_INTERVAL_MS` | No | `30000` | Milliseconds between RPC poll cycles. |
| `INDEXER_BATCH_SIZE` | No | `200` | Upper bound on events fetched per RPC call. |

### 1.2 remitlend-frontend

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `NEXT_PUBLIC_API_URL` | Yes | | Public-facing API base URL. |
| `NEXT_PUBLIC_STELLAR_NETWORK` | Yes | | `"testnet"` or `"public"`; controls wallet/Horizon URLs. |

### 1.3 Startup Validation

Services fail fast on missing mandatory variables. If you see:

```
Missing required env: DATABASE_URL
```

set the variable before restarting.

---

## 2. Local Development Setup

### 2.1 Prerequisites

- Node.js 20+
- PostgreSQL 15+
- Redis 7+
- npm (project uses npm, not pnpm)

### 2.2 Database and Redis

```bash
# Start PostgreSQL
brew services start postgresql@15

# Start Redis
brew services start redis

# Create the application database
createdb remitlend_dev
```

### 2.3 Running Migrations

Migrations live in `migrations/` at the `remitlend-backend` repo root. State is tracked by `node-pg-migrate` in a `pgmigrations` table.

```bash
cd remitlend-backend

# Install dependencies
npm install

# Run pending migrations
npm run migrate

# Verify
psql $DATABASE_URL -c "SELECT * FROM pgmigrations LIMIT 5;"
```

`node-pg-migrate` creates the `pgmigrations` table on first run if it does not exist.

### 2.4 Environment File

```bash
cp .env.example .env
# Fill in STELLAR_RPC_URL, all four *_CONTRACT_ID values, JWT_SECRET, etc.
```

Common pitfalls:
- `STELLAR_RPC_URL` must be a Soroban-capable RPC node; plain Horizon endpoints will not work.
- Each `*_CONTRACT_ID` must be the Soroban contract ID (hex + base32), not a classic operation hash.

### 2.5 Starting Services

The backend repo contains the API server, event indexer, and notification logic. There are no separate `notifier/` or `indexer/` directories or repos.

```bash
# Terminal 1 — Backend (API + indexer + notifications)
cd remitlend-backend && npm run dev

# Terminal 2 — Frontend
cd remitlend-frontend && npm run dev
```

---

## 3. Staging Deployment

### 3.1 Current State

Staging is deployed manually today. The pipeline below is a **proposed target setup** and is not yet implemented.

### 3.2 Proposed Target Setup

1. **Push** — CI builds Docker images and pushes to a container registry.
2. **Scan** — Trivy scans images for critical/high CVEs; pipeline fails above policy threshold.
3. **Deploy** — Deploy manifests reference the newly pushed image tags.
4. **Healthcheck** — Post-deploy script hits each service `/health` endpoint.
5. **Rollback** — If healthchecks or smoke tests fail, roll back to the previous known-good tag.

---

## 4. Healthchecks

| Service | Endpoint | Checks |
|---------|----------|--------|
| API | `/health` | DB connectivity, RPC connectivity, contract reachability |
| Indexer (in-process) | `/health` | DB connectivity, `last_indexed_ledger` recency (lag < 2 `INDEXER_POLL_INTERVAL_MS` cycles) |
| Frontend | `/api/health` | Build version, feature flags loaded |

Example response:

```json
{
  "status": "healthy",
  "timestamp": "2026-06-20T12:00:00Z",
  "checks": {
    "database": { "status": "up" },
    "stellar_rpc": { "status": "up", "latency_ms": 120 }
  }
}
```

---

## 5. Troubleshooting

### 5.1 Service crashes immediately with a missing env var

**Symptom:** `Missing required env: ...` or `ECONNREFUSED` to PostgreSQL.

**Fix:**
- Confirm the variable is set (local `.env` or runtime config).
- Verify PostgreSQL is running: `pg_isready -h <host>`.

### 5.2 Indexer not ingesting events (lag grows)

**Symptom:** `last_indexed_ledger` stops advancing; `indexer_state` shows stale `updated_at`.

**Fix:**
- Check `STELLAR_RPC_URL` is reachable and not rate-limiting.
- Look for RPC downtime or `getEvents` errors in backend logs.
- Verify the correct `*_CONTRACT_ID` values for the target network.
- Increase `INDEXER_POLL_INTERVAL_MS` if the RPC node is timing out.

### 5.3 500 errors on API after deploy

**Symptom:** `5xx` rate spike post-deploy.

**Fix:**
- Check logs for missing env vars or unhandled exceptions.
- Verify schema matches code version: `npm run migrate`.
- If recent, check for breaking contract ID mismatches between frontend and backend.

### 5.4 Outbound webhooks / email / SMS failing

**Symptom:** Webhook delivery failures, or SendGrid / Twilio errors in logs.

**Fix:**
- Verify outbound egress is not blocked.
- Increase `WEBHOOK_REQUEST_TIMEOUT_MS` if the downstream service is slow.
- Confirm `SENDGRID_API_KEY`, `TWILIO_ACCOUNT_SID`, and `TWILIO_AUTH_TOKEN` are current.

### 5.5 Frontend shows stale data or wallet errors

**Symptom:** Old balances, or wallet connector says "network mismatch."

**Fix:**
- Ensure `NEXT_PUBLIC_STELLAR_NETWORK` matches the backend network.
- Clear browser cache and local storage (wallet keypairs cache network config).
- Verify the RPC node is not returning stale/cached ledgers.

### 5.6 PostgreSQL connection pool exhaustion

**Symptom:** `too many connections for role "remitlend"`.

**Fix:**
- Check backend DB connection pool settings.
- Enable PgBouncer if running many replicas.
- Reduce `INDEXER_BATCH_SIZE` or `INDEXER_POLL_INTERVAL_MS` to lower indexer query volume.

---

## 6. Useful Commands

```bash
# Install backend deps
cd remitlend-backend && npm install

# Run migrations
cd remitlend-backend && npm run migrate

# Start backend dev server
cd remitlend-backend && npm run dev

# Start frontend dev server
cd remitlend-frontend && npm run dev

# Check PostgreSQL
pg_isready -h <host>

# Check Redis
redis-cli ping

# Inspect indexer events
psql $DATABASE_URL -c "SELECT * FROM contract_events ORDER BY created_at DESC LIMIT 20;"

# Inspect failed webhooks
psql $DATABASE_URL -c "SELECT * FROM webhook_deliveries WHERE last_status_code >= 400 ORDER BY created_at DESC LIMIT 20;"

# View migration history
psql $DATABASE_URL -c "SELECT * FROM pgmigrations ORDER BY run_on DESC LIMIT 10;"
```
