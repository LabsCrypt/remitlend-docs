# API Reference

This document is the canonical reference for the RemitLend REST API. It covers every public endpoint, schemas, auth requirements, error shapes, and cross-cutting concerns like idempotency and rate limiting.

---

## 1. Base URLs

| Environment | Base URL |
|-------------|----------|
| Production | `https://api.remitlend.com` |
| Staging | `https://api-staging.remitlend.com` |

All endpoints below are relative to the base URL unless otherwise noted. Every response payload is JSON.

---

## 2. Conventions

- **Paths** are prefixed with `/v1`.
- **Auth** requirements are listed per endpoint.
- **RBAC**: Protected endpoints require a JWT whose embedded role satisfies the minimum access level. See [2.2 RBAC Roles](#22-rbac-roles).
- **Idempotency**: Mutations that may be retried (payment submission, loan creation) accept an `Idempotency-Key` header. Replay of the same key returns the original response.
- **Rate limit**: A sliding-window counter keyed by caller IP or user ID is enforced. See [7. Rate Limiting](#7-rate-limiting).

---

## 3. Authentication

### 3.1 JWT Bearer Token

User-facing endpoints require a `Bearer` token issued by the challenge-verify flow described in `security.md`.

**Request header:**
```
Authorization: Bearer <jwt>
```

### 3.2 Internal API Key

Administrative, internal, or cron endpoints require the `x-api-key` header.

**Request header:**
```
x-api-key: <internal-api-key>
```

This secret must never be exposed to browser clients.

### 3.3 RBAC Roles

| Role | Capabilities |
|------|--------------|
| `borrower` | View own balances and loans, initiate transfers, manage profile |
| `lender` | Manage liquidity pools, view lending metrics |
| `admin` | Execute critical state mutations, access admin routes |

Roles are resolved at auth time and attached to request context. A `403` response means the authenticated identity lacks the required role.

---

## 4. Endpoints

### 4.1 Auth

#### `POST /v1/auth/challenge`

Issue a time-bound challenge for wallet signature.

| | |
|--|--|
| Auth | None (public) |
| Request body | `{ "address": "string" }` |
| Response `200` | `{ "challenge": "string", "expiresAt": "ISO8601" }` |

---

#### `POST /v1/auth/verify`

Verify a signed challenge and receive a JWT.

| | |
|--|--|
| Auth | None (public) |
| Request body | `{ "address": "string", "signature": "string" }` |
| Response `200` | `{ "accessToken": "string", "expiresAt": "ISO8601" }` |
| Error `401` | Invalid or expired signature |

---

### 4.2 Loans

#### `GET /v1/loans`

List loans visible to the authenticated caller.

| | |
|--|--|
| Auth | JWT — borrower, lender, or admin scopes |
| Query params | `status?: string`, `borrower?: string`, `cursor?: string`, `limit?: number` |
| Response `200` | `{ "items": Loan[], "nextCursor": "string?" }` |

---

#### `POST /v1/loans`

Create a new loan request.

| | |
|--|--|
| Auth | JWT — borrower |
| Headers | `Idempotency-Key: <uuid>` |
| Request body | `{ "amount": "number", "termDays": "number", "collateralAddress": "string" }` |
| Response `201` | `{ "loanId": "string", "status": "Pending", "createdAt": "ISO8601" }` |
| Error `409` | Idempotency key already used |
| Error `422` | Invalid amount or term |

---

#### `GET /v1/loans/:loanId`

Fetch a single loan by ID.

| | |
|--|--|
| Auth | JWT — borrower, lender, or admin |
| Response `200` | `Loan` object |

---

#### `POST /v1/loans/:loanId/repay`

Submit a repayment for an active loan.

| | |
|--|--|
| Auth | JWT — borrower |
| Headers | `Idempotency-Key: <uuid>` |
| Request body | `{ "txHash": "string" }` |
| Response `200` | `{ "status": "Repaid", "repaymentTx": "string" }` |
| Error `409` | Idempotency key already used |

---

#### `PATCH /v1/loans/:loanId/approve`

Approve a pending loan. Lenders or admin only.

| | |
|--|--|
| Auth | JWT — lender or admin |
| Response `200` | `{ "status": "Approved", "approvedAt": "ISO8601" }` |

---

### 4.3 Pool

#### `GET /v1/pool`

Get current liquidity pool metrics.

| | |
|--|--|
| Auth | JWT — lender or admin |
| Response `200` | `{ "totalLiquidity": "string", "availableLiquidity": "string", "utilizationRateBps": "number" }` |

---

#### `POST /v1/pool/deposit`

Deposit funds into the liquidity pool.

| | |
|--|--|
| Auth | JWT — lender |
| Headers | `Idempotency-Key: <uuid>` |
| Request body | `{ "amount": "string", "txHash": "string" }` |
| Response `200` | `{ "txHash": "string", "newLenderShare": "string" }` |

---

#### `POST /v1/pool/withdraw`

Withdraw from the liquidity pool.

| | |
|--|--|
| Auth | JWT — lender |
| Headers | `Idempotency-Key: <uuid>` |
| Request body | `{ "amount": "string", "txHash": "string" }` |
| Response `200` | `{ "txHash": "string", "remainingShare": "string" }` |

---

### 4.4 Remittances

#### `POST /v1/remittances`

Initiate a remittance transfer.

| | |
|--|--|
| Auth | JWT — borrower or user |
| Headers | `Idempotency-Key: <uuid>` |
| Request body | `{ "to": "string", "amount": "string", "currency": "string" }` |
| Response `201` | `{ "remittanceId": "string", "status": "Pending", "estimatedArrival": "ISO8601" }` |

---

#### `GET /v1/remittances`

List remittances.

| | |
|--|--|
| Auth | JWT |
| Query params | `status?: string`, `cursor?: string`, `limit?: number` |
| Response `200` | `{ "items": Remittance[], "nextCursor": "string?" }` |

---

#### `GET /v1/remittances/:remittanceId`

Fetch a single remittance.

| | |
|--|--|
| Auth | JWT |
| Response `200` | `Remittance` object |

---

### 4.5 Score

#### `GET /v1/score/:address`

Get the credit score for a Stellar address.

| | |
|--|--|
| Auth | JWT (self or admin only) |
| Response `200` | `{ "address": "string", "score": "number", "updatedAt": "ISO8601" }` |

---

### 4.6 Notifications

#### `GET /v1/notifications`

List notifications for the authenticated user.

| | |
|--|--|
| Auth | JWT |
| Query params | `read?: boolean`, `cursor?: string`, `limit?: number` |
| Response `200` | `{ "items": Notification[], "nextCursor": "string?" }` |

---

#### `PATCH /v1/notifications/:notificationId/read`

Mark a notification as read.

| | |
|--|--|
| Auth | JWT — owner only |
| Response `204` | — |

---

### 4.7 Admin

All admin endpoints require a JWT with the `admin` role **and** a valid internal API key unless specified otherwise.

#### `POST /v1/admin/webhooks`

Register or update a webhook subscription.

| | |
|--|--|
| Auth | JWT — admin + `x-api-key` |
| Request body | `{ "userId": "string", "callbackUrl": "string", "events": "string[]" }` |
| Response `200` | `{ "subscriptionId": "string" }` |

---

#### `GET /v1/admin/webhooks`

List webhook subscriptions.

| | |
|--|--|
| Auth | JWT — admin + `x-api-key` |
| Response `200` | `WebhookSubscription[]` |

---

#### `POST /v1/admin/contract/upgrade`

Queue a contract upgrade through the timelock flow.

| | |
|--|--|
| Auth | JWT — admin + `x-api-key` |
| Request body | `{ "wasmHash": "string", "notes": "string" }` |
| Response `200` | `{ "timelockTransactionId": "string", "unlockAt": "ISO8601" }` |

---

#### `GET /v1/admin/audit-logs`

Fetch audit log entries.

| | |
|--|--|
| Auth | JWT — admin + `x-api-key` |
| Query params | `actor?: string`, `action?: string`, `cursor?: string`, `limit?: number` |
| Response `200` | `{ "items": AuditLog[], "nextCursor": "string?" }` |

---

### 4.8 Indexer (Read-Only Internal)

The indexer exposes a minimal read surface for observability. All endpoints require the internal API key.

#### `GET /v1/indexer/health`

Healthcheck for the indexer service.

| | |
|--|--|
| Auth | `x-api-key` |
| Response `200` | `{ "status": "string", "lastIndexedLedger": "number", "lag": "number" }` |

---

## 5. Schemas

### 5.1 Loan

| Field | Type | Description |
|-------|------|-------------|
| `loanId` | string | Unique internal identifier |
| `borrowerAddress` | string | Stellar public key |
| `amount` | string | Loan amount in smallest unit |
| `termDays` | number | Repayment term |
| `status` | enum | `Pending`, `Approved`, `Repaid`, `Defaulted`, `Liquidated`, `Cancelled`, `Rejected` |
| `createdAt` | ISO8601 | Timestamp |
| `approvedAt` | ISO8601? | Nullable |

---

### 5.2 Remittance

| Field | Type | Description |
|-------|------|-------------|
| `remittanceId` | string | Unique internal identifier |
| `from` | string | Sender Stellar address |
| `to` | string | Recipient Stellar address |
| `amount` | string | Amount in smallest unit |
| `currency` | string | Asset code |
| `status` | enum | `Pending`, `InFlight`, `Completed`, `Failed` |
| `estimatedArrival` | ISO8601 | Estimated completion time |
| `createdAt` | ISO8601 | Timestamp |

---

### 5.3 Notification

| Field | Type | Description |
|-------|------|-------------|
| `notificationId` | string | Unique identifier |
| `userId` | string | Recipient user ID |
| `message` | string | Notification body |
| `type` | enum | `LoanApproved`, `LoanRepaid`, `LoanDefaulted`, `System` |
| `read` | boolean | Read status |
| `createdAt` | ISO8601 | Timestamp |

---

### 5.4 WebhookSubscription

| Field | Type | Description |
|-------|------|-------------|
| `subscriptionId` | string | Unique identifier |
| `userId` | string | Owning user ID |
| `callbackUrl` | string | Delivery endpoint |
| `events` | string[] | Subscribed event types |
| `secret` | string? | HMAC secret (returned on creation only) |
| `createdAt` | ISO8601 | Timestamp |

---

### 5.5 AuditLog

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier |
| `actor` | string | Wallet address or service name |
| `action` | string | Action performed (e.g., `loan.approve`) |
| `resourceId` | string? | Affected resource ID |
| `ipAddress` | string? | Caller IP (anonymized) |
| `userAgent` | string? | Caller user agent |
| `createdAt` | ISO8601 | Timestamp |

---

## 6. Standardized Error Response Shape

All errors return JSON with this structure.

```json
{
  "error": {
    "code": "string",
    "message": "string",
    "details": {},
    "requestId": "string"
  }
}
```

| Field | Description |
|-------|-------------|
| `code` | Machine-readable error identifier (e.g., `UNAUTHORIZED`) |
| `message` | Human-readable description |
| `details` | Optional map of field-level error descriptions (used with `422`) |
| `requestId` | Trace ID for support correlation |

### 6.1 Common Error Codes

| HTTP | Code | Meaning |
|------|------|---------|
| `400` | `BAD_REQUEST` | Malformed input or validation failure |
| `401` | `UNAUTHORIZED` | Missing or invalid JWT / API key |
| `403` | `FORBIDDEN` | JWT is valid but role is insufficient |
| `404` | `NOT_FOUND` | Resource does not exist |
| `409` | `IDEMPOTENCY_CONFLICT` | `Idempotency-Key` already used for a distinct request |
| `422` | `VALIDATION_ERROR` | Semantically invalid payload (e.g., amount below `MinLoanAmount`) |
| `429` | `RATE_LIMITED` | Rate limit exceeded |
| `500` | `INTERNAL` | Unhandled server error |

---

## 7. Rate Limiting

- Requests are throttled on a sliding window keyed by IP for unauthenticated routes and by user ID for authenticated routes.
- Limit headers are included in every response:

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 44
X-RateLimit-Reset: 1718882400
```

- On limit exhaustion, the API returns `429` with a `Retry-After` header indicating seconds to wait.
- Concurrent write bursts to the same loan or remittance are serialized; retries respect `Idempotency-Key`.

---

## 8. Idempotency

Mutating endpoints (`POST /v1/loans`, `POST /v1/loans/:id/repay`, `POST /v1/remittances`, `POST /v1/pool/deposit`, `POST /v1/pool/withdraw`) accept an `Idempotency-Key` header:

```
Idempotency-Key: <client-generated-uuid-v4>
```

Behavior:
- First valid request with a given key is processed normally.
- Replay within 24 hours returns the exact same response (status, body, headers).
- Distinct keys always process independently.

---

## 9. Versioning

The `/v1` prefix is pinned. Backward-breaking changes will be introduced under `/v2`. Deprecation announcements are surfaced via the `Sunset` response header and changelog.

---

## 10. Examples

### cURL — Create Loan

```bash
curl -X POST https://api.remitlend.com/v1/loans \
  -H "Authorization: Bearer $JWT" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{"amount": 1000, "termDays": 30, "collateralAddress": "G..."}'
```

### cURL — Verify Signature (Admin)

```bash
curl -X POST https://api.remitlend.com/v1/admin/webhooks \
  -H "Authorization: Bearer $ADMIN_JWT" \
  -H "x-api-key: $INTERNAL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"userId": "user-123", "callbackUrl": "https://example.com/hook", "events": ["LoanRepaid","LoanDefaulted"]}'
```
