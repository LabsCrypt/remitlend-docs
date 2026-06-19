# Postgres Data Model

This document outlines the core tables, views, and indexes used in the Remitlend PostgreSQL database, as well as how they map to on-chain Soroban events.

## Core Tables

| Table Name | Purpose | Key Columns |
| :--- | :--- | :--- |
| **`loans`** | Stores the state and history of active/past loans. | `id`, `borrower_address`, `amount`, `status`, `created_at` |
| **`contract_events`** | Stores raw decoded events directly from the Soroban contract. | `event_id`, `tx_hash`, `topic`, `value`, `created_at` |
| **`scores`** | Maintains the current credit score for each user. | `borrower`, `score`, `updated_at`, `created_at` |
| **`webhook_subscriptions`** | User-configured webhooks for external integrations. | `id`, `user_address`, `callback_url`, `event_types` |
| **`webhook_deliveries`** | Logs of webhook payload delivery attempts. | `id`, `subscription_id`, `last_status_code` |
| **`notifications`** | In-app alerts and notifications for users. | `id`, `user_address`, `message`, `read`, `status` |
| **`audit_logs`** | Immutable log of critical off-chain system actions. | `id`, `action`, `actor`, `created_at` |
| **`indexer_state`** | Tracks the current ledger synchronization point. | `last_indexed_ledger`, `last_indexed_cursor` |

## Important Indexes & Views

### Indexes
To support high-performance queries and real-time indexing, several important indexes are maintained:
- **`scores(borrower)`**: Enables fast O(1) lookups of a user's credit score during API requests and dashboard loads.
- **`contract_events(event_id)`** & **`(loan_id, event_type, ledger)`**: Unique constraints that act as idempotency constraints (`ON CONFLICT DO NOTHING`), preventing duplicate event processing. (Note: API request idempotency is handled via Redis cacheService).
- **`webhook_subscriptions(event_types)`**: Allows efficient fan-out of webhook payloads when specific events occur.

### The Unified `loan_events` View
The **`loan_events`** view acts as a unified abstraction over the raw `contract_events` table. Instead of requiring contributors to parse raw XDR `topic` arrays manually, this view formats the events specifically related to the loan lifecycle (e.g., `LoanRequested`, `LoanRepaid`). It presents a clean, typed interface for querying historical loan activity.

## Mapping to On-Chain Events via the Indexer

The Postgres schema maps directly to on-chain Soroban events via the `EventIndexer` service:
1. The indexer polls the Stellar RPC and decodes the XDR base64 event payloads.
2. It parses the `topic` array (e.g., Event Type like `LoanRequested`, Borrower Address, Loan ID) and the `value`.
3. The decoded event is transactionally inserted into the `contract_events` table.
4. Concurrently, the indexer updates downstream tables derived from this event. For example, a `LoanRepaid` event will trigger an `UPDATE` on the `scores` table to boost the borrower's credit score.

For a detailed walkthrough of this synchronization flow, see the [Indexer Sync Flow](wiki/indexer-sync-flow.md).
