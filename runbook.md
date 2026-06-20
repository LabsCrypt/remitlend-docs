# RemitLend Operations Runbook

This runbook is the single source of truth for operators running or deploying RemitLend. It covers environment variables, staging deployments, local development, and common troubleshooting.

---

## 1. Environment Variables

Environment variables are organized by service. All URLs and secrets must be provided as strings.

### 1.1 API / Backend Service

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PORT` | No | `3000` | HTTP listen port for the API server. |
| `DATABASE_URL` | Yes | — | PostgreSQL connection string (`postgres://user:pass@host:5432/dbname`). |
| `REDIS_URL` | No | `redis://localhost:6379` | Redis connection string for caching and rate limiting. |
| `STELLAR_RPC_URL` | Yes | — | Stellar Soroban RPC endpoint (e.g., `https://soroban-rpc.stellar.org`). |
| `STELLAR_NETWORK_PASSPHRASE` | Yes | — | Stellar network passphrase (`Test SDF Network ; September 2015` or `Public Global Stellar Network ; September 2015`). |
| `CONTRACT_ID` | Yes | — | Deployed Soroban `loan_manager` contract ID. |
| `INTERNAL_API_KEY` | Yes | — | Shared secret for internal service-to-service authentication. Set a strong random 32+ char string. |
| `WEBHOOK_SECRET` | No | — | HMAC secret for signing outgoing webhook payloads. |
| `JWT_SECRET` | Yes | — | Secret used to sign and verify JWTs for user sessions. |
| `WEBHOOK_TIMEOUT_MS` | No | `5000` | Timeout for outbound webhook HTTP calls. |

### 1.2 Event Indexer Service

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DATABASE_URL` | Yes | — | Alias into the same PostgreSQL instance managed by the backend API. |
| `STELLAR_RPC_URL` | Yes | — | Soroban RPC endpoint for ledger polling. |
| `STELLAR_NETWORK_PASSPHRASE` | Yes | — | Must match the backend service. |
| `INDEXER_POLL_INTERVAL_MS` | No | `30000` | Milliseconds between RPC poll cycles. Tune based on ledger throughput (every 5–30s is typical). |
| `MAX_EVENTS_PER_POLL` | No | `200` | Upper bound on events fetched per RPC call to avoid timeouts. |

### 1.3 Notification Service

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PORT` | No | `4000` | HTTP listen port. |
| `DATABASE_URL` | Yes | — | PostgreSQL connection string. |
| `REDIS_URL` | No | `redis://localhost:6379` | Redis connection for queueing. |
| `SMTP_HOST` | Yes | — | Outbound mail server hostname. |
| `SMTP_PORT` | Yes | — | Outbound mail server port (usually `587` for TLS, `465` for SSL). |
| `SMTP_USER` | Yes | — | SMTP authentication username. |
| `SMTP_PASS` | Yes | — | SMTP authentication password or app-specific token. |
| `FROM_EMAIL` | Yes | — | Sender address seen by end users (e.g., `noreply@remitlend.com`). |

### 1.4 Frontend (Next.js)

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `NEXT_PUBLIC_API_URL` | Yes | — | Public-facing API base URL (e.g., `https://api-staging.remitlend.com`). |
| `NEXT_PUBLIC_STELLAR_NETWORK` | Yes | — | `"testnet"` or `"public"` — controls which Horizon/RPC URLs the wallet uses. |

### 1.5 Startup Validation

Services should fail fast on missing mandatory variables. If you see a startup log like:

```
FATAL: DATABASE_URL is required
```

set the variable before restarting.

---

## 2. Local Development Setup

### 2.1 Prerequisites

- Node.js 20+ and pnpm
- PostgreSQL 15+
- Redis 7+
- Docker and Docker Compose
- Stellar CLI (optional, for contract inspection)

### 2.2 Database and Redis

```bash
# Start PostgreSQL (Homebrew example; use your OS package manager)
brew services start postgresql@15

# Start Redis
brew services start redis

# Create the application database
createdb remitlend_dev
```

### 2.3 Running Migrations

Migrations live in `backend/src/db/migrations/` and use a migration runner script.

```bash
cd backend

# Install dependencies
pnpm install

# Run pending migrations
pnpm migrate

# Verify by connecting to the DB
psql $DATABASE_URL -c "SELECT * FROM schema_migrations LIMIT 5;"
```

If the `schema_migrations` table does not exist, the migration runner will create it on first run.

### 2.4 Environment File

Create a `.env` in the backend root by copying `.env.example`:

```bash
cp backend/.env.example backend/.env
# Fill in STELLAR_RPC_URL, CONTRACT_ID, JWT_SECRET, etc.
```

Common pitfalls:
- `STELLAR_RPC_URL` must point to a Soroban-capable RPC node (plain Horizon endpoints will not work for contract events).
- `CONTRACT_ID` should be the **Soroban contract ID** (hex + base32), not the classic operation hash.

### 2.5 Starting Services

```bash
# Terminal 1 — API server
cd backend && pnpm dev

# Terminal 2 — Event indexer
cd backend && pnpm dev:indexer

# Terminal 3 — Notification service
cd notifier && pnpm dev

# Terminal 4 — Frontend
cd frontend && pnpm dev
```

---

## 3. Staging Deployment

### 3.1 Overview

Staging deploys are triggered by merging to `main`. A GitHub Actions workflow builds Docker images, scans them with Trivy, pushes to the staging container registry, deploys, performs healthchecks, and can roll back automatically on failure.

### 3.2 Image Build and Registry

Images are built with multi-stage Dockerfiles and pushed to:

```
registry.example.com/remitlend/<service>:<git-sha>-<semver>
```

Services:
- `api`
- `indexer`
- `notifier`
- `frontend`

### 3.3 Deployment Steps

1. **Push** — GitHub Actions builds images and pushes to the registry.
2. **Trivy Scan** — Each image is scanned for critical and high-severity CVEs. The pipeline fails if uncritical vulnerabilities are above the policy threshold.
3. **Deploy** — `kubectl apply` (or your orchestrator) deploys manifests that reference the newly pushed image tags.
4. **Healthcheck** — A post-deploy script hits each service health endpoint:
   - `GET https://api-staging.remitlend.com/health`
   - `GET https://indexer-staging.remitlend.com/health`
   - `GET https://notifier-staging.remitlend.com/health`
5. **Smoke Tests** — A canary test suite verifies basic contract invocation and event ingestion.

### 3.4 Rollback

Rollbacks are manual but straightforward:

```bash
# 1. Identify the previous working image tag from the GitHub Actions run history
# or the Kubernetes deployment history:
kubectl rollout history deployment/api -n staging

# 2. Roll back the deployment:
kubectl rollout undo deployment/api -n staging
kubectl rollout undo deployment/indexer -n staging
kubectl rollout undo deployment/notifier -n staging

# 3. Verify:
kubectl rollout status deployment/api -n staging
kubectl get pods -n staging -l app=api
```

If the database has a breaking migration, roll it back first:

```bash
kubectl exec -it <postgres-pod> -- psql $DATABASE_URL -c "SELECT * FROM schema_migrations;"
# Identify the bad migration and revert the schema change manually.
```

### 3.5 Rollback Triggers

Consider rolling back when:
- Healthcheck fails for more than two consecutive minutes.
- Outbox/webhook delivery failure rate exceeds 10%.
- Error rate (5xx) on API exceeds 5%.

---

## 4. Healthchecks

Each service exposes a lightweight health endpoint.

| Service | Endpoint | Checks |
|---------|----------|--------|
| API | `/health` | DB connectivity, RPC connectivity, contract reachability |
| Indexer | `/health` | DB connectivity, `last_indexed_ledger` recency (lag < 2 `POLL_INTERVAL` cycles) |
| Notifier | `/health` | DB connectivity, SMTP connectivity |
| Frontend | `/api/health` | Build version, feature flags loaded |

Example healthcheck response:

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

### 5.1 Service crashes immediately with a DATABASE_URL error

**Symptom:** `FATAL: DATABASE_URL is required` or `ECONNREFUSED` to PostgreSQL.

**Fix:**
- Confirm `DATABASE_URL` is set in the environment (local `.env` or K8s secret).
- Verify PostgreSQL is running: `pg_isready -h <host>`.
- Check firewall / VPC peering if DB is remote.

### 5.2 Indexer not ingesting events (lag grows)

**Symptom:** `last_indexed_ledger` stops advancing, `indexer_state` table shows stale `updated_at`.

**Fix:**
- Check `STELLAR_RPC_URL` is reachable and not rate-limiting.
- Look for Stellar RPC downtime or `getEvents` errors in indexer logs.
- Verify `CONTRACT_ID` is correct on the target network.
- Increase `INDEXER_POLL_INTERVAL_MS` if the RPC node is timing out under load.
- Check the `contract_events` table for constraint violations that might block inserts.

### 5.3 500 errors on API after deploy

**Symptom:** `5xx` rate spike post-deploy.

**Fix:**
- Check API logs for unhandled exceptions (missing env vars after a ConfigMap/secret change is a common cause).
- Verify the database schema matches the code version (run migrations).
- If the deploy is recent, check for breaking contract changes — the frontend and backend must target the same `CONTRACT_ID`.

### 5.4 Outbound webhooks failing

**Symptom:** `webhook_deliveries` table shows consecutive `5xx` or timeout status codes.

**Fix:**
- Verify outbound network egress is not blocked by a firewall or WAF.
- Increase `WEBHOOK_TIMEOUT_MS` if the downstream service is slow.
- Check that `WEBHOOK_SECRET` matches what the consumer expects (rotated secrets break HMAC verification).

### 5.5 Frontend shows stale data or wallet errors

**Symptom:** Old balances, or the wallet connector says "network mismatch."

**Fix:**
- Ensure `NEXT_PUBLIC_STELLAR_NETWORK` matches the deployed backend network.
- Clear the browser cache and local storage (wallet keypair caches network config).
- Verify the RPC URL is not returning cached ledgers (use a dedicated RPC node rather than a heavily cached public endpoint).

### 5.6 PostgreSQL connection pool exhaustion

**Symptom:** `too many connections for role "remitlend"` or `remaining connection slots are reserved`.

**Fix:**
- Check connection pool settings in `backend/src/db/connection.js` and the notifier config.
- Enable PgBouncer if running many service replicas.
- Reduce `MAX_EVENTS_PER_POLL` or `INDEXER_POLL_INTERVAL_MS` to lower query volume on the indexer.

---

## 6. Useful Commands

```bash
# Check PostgreSQL replication lag
psql $DATABASE_URL -c "SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;"

# Check Redis connectivity
redis-cli ping

# Inspect the last 20 indexer events
psql $DATABASE_URL -c "SELECT * FROM contract_events ORDER BY created_at DESC LIMIT 20;"

# Inspect failed webhook deliveries
psql $DATABASE_URL -c "SELECT * FROM webhook_deliveries WHERE last_status_code >= 400 ORDER BY created_at DESC LIMIT 20;"

# Roll back a Kubernetes deployment
kubectl rollout undo deployment/<service> -n staging
```
