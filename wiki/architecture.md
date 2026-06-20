# System Architecture Overview

RemitLend is a DeFi gamification platform built on Stellar's Soroban smart contracts. This document provides a high-level overview of how all components interact to deliver loan management and credit scoring functionality.

## Architecture Diagram

```mermaid
graph TB
    User[("👤 User<br/>(Browser)")]

    subgraph Frontend["🎮 Frontend Layer"]
        NextJS["Next.js App<br/>(React)"]
        RedisCache["Redis Cache<br/>(Session & Queries)"]
    end

    subgraph Backend["🔧 Backend Layer"]
        API["REST API<br/>(Node.js)"]
        AuthService["Auth Service<br/>(JWT)"]
        EventIndexer["Event Indexer<br/>(Polling)"]
    end

    subgraph Data["💾 Data Layer"]
        PostgreSQL["PostgreSQL<br/>(User, Scores,<br/>Loan Events)"]
    end

    subgraph Blockchain["⛓️ Blockchain Layer"]
        Contracts["Soroban Contracts<br/>(LoanManager,<br/>RemittanceNFT)"]
        StellarRPC["Stellar RPC Node<br/>(Event Stream)"]
    end

    subgraph External["🌐 External Services"]
        Sentry["Sentry<br/>(Error Tracking)"]
    end

    User -->|1. Interact| NextJS
    NextJS -->|2. API Calls| API
    API -->|3. Read/Write| PostgreSQL
    API -->|4. Check Cache| RedisCache
    RedisCache -->|5. Response| API
    API -->|6. Auth Check| AuthService

    NextJS -->|Display Data| User

    User -->|8. Submit Loan Request| API
    API -->|9. Invoke| Contracts
    Contracts -->|10. Emit Events| StellarRPC

    EventIndexer -->|11. Poll Events| StellarRPC
    EventIndexer -->|12. Decode & Process| EventIndexer
    EventIndexer -->|13. Update State| PostgreSQL
    EventIndexer -->|14. Notify Backend| API
    PostgreSQL -->|15. Query Results| NextJS

    API -->|Error Logs| Sentry
    EventIndexer -->|Error Logs| Sentry
    NextJS -->|Error Logs| Sentry
```

## Component Descriptions

### Frontend Layer

- **Next.js App**: React-based UI that displays loan status, credit scores, quest progress, and kingdom tier information
- **Redis Cache**: Temporary storage for session data and frequently-queried results to reduce database load

### Backend Layer

- **REST API**: Node.js server handling user requests, authentication, loan operations, and real-time data queries
- **Auth Service**: JWT-based authentication for secure API access
- **Event Indexer**: Background service that continuously polls Stellar RPC for on-chain events and synchronizes them with the database

### Data Layer

- **PostgreSQL**: Primary database storing user profiles, credit scores, loan history, and indexed events

### Blockchain Layer

- **Soroban Contracts**: Smart contracts on Stellar that manage loan lifecycle (`LoanManager`) and user credit tracking (`RemittanceNFT`)
- **Stellar RPC Node**: Provides access to contract events and allows transaction submission

### External Services

- **Sentry**: Error tracking and monitoring for all layers

## Data Flow Diagrams

### Request Flow (User Action → Contract)

```mermaid
sequenceDiagram
    participant User as 👤 User
    participant Frontend as 🎮 Next.js
    participant Backend as 🔧 Backend API
    participant Contract as ⛓️ Soroban Contract
    participant DB as 💾 PostgreSQL

    User->>Frontend: Click "Request Loan"
    Frontend->>Backend: POST /loans/request (amount, borrower)
    Backend->>Backend: Validate request
    Backend->>Contract: invoke request_loan()
    Contract->>Contract: Check eligibility, create loan
    Contract-->>Backend: ✓ Transaction ID
    Backend->>DB: Record loan request
    DB-->>Backend: ✓ Stored
    Backend-->>Frontend: { status: "pending", txId: "..." }
    Frontend-->>User: Show "Loan requested, awaiting approval"
```

### Event Processing Flow (On-Chain → Database)

```mermaid
sequenceDiagram
    participant Contract as ⛓️ Soroban Contract
    participant RPC as 🔌 Stellar RPC
    participant Indexer as 📡 Event Indexer
    participant DB as 💾 PostgreSQL
    participant Cache as 🔴 Redis

    Contract->>RPC: Emit LoanApproved event

    loop Every 30 seconds
        Indexer->>RPC: getEvents(startLedger, contractId)
        RPC-->>Indexer: [LoanApproved, ...]
        Indexer->>Indexer: Decode event XDR
        Indexer->>DB: UPDATE scores (+points)
        DB-->>Indexer: ✓ Updated
        Indexer->>Cache: Invalidate user score
        Cache-->>Indexer: ✓ Cleared
    end

    note over Indexer,DB: Real-time credit score updates
```

### User Dashboard Load (Frontend → Database)

```mermaid
sequenceDiagram
    participant User as 👤 User
    participant Frontend as 🎮 Next.js
    participant Backend as 🔧 Backend API
    participant Cache as 🔴 Redis
    participant DB as 💾 PostgreSQL

    User->>Frontend: Visit dashboard
    Frontend->>Backend: GET /dashboard

    Backend->>Cache: Check cached data
    alt Cache hit
        Cache-->>Backend: { score: 850, loans: [...] }
        Backend-->>Frontend: ✓ (fast)
    else Cache miss
        Backend->>DB: Query user scores, loans
        DB-->>Backend: [Data]
        Backend->>Cache: Store for 5 min
        Backend-->>Frontend: [Data]
    end

    Frontend-->>User: Render dashboard with real-time data
```

## Trust Boundaries

```mermaid
graph LR
    Internet["🌐 Internet<br/>(Untrusted)"]
    FrontendUser["🎮 Frontend +<br/>User Browser<br/>(Semi-Trusted)"]
    Backend["🔧 Backend<br/>(Trusted)"]
    Contracts["⛓️ Soroban<br/>Contracts<br/>(Immutable)"]

    Internet -->|HTTPS| FrontendUser
    FrontendUser -->|JWT Token| Backend
    Backend -->|Invoke only<br/>after checks| Contracts

    note over Internet,FrontendUser: Trust Boundary #1<br/>Validated by HTTPS & JWT
    note over Backend,Contracts: Trust Boundary #2<br/>Soroban is source of truth
```

### Trust Boundaries Explained

1. **Internet ↔ Frontend**: SSL/TLS encryption; all API calls authenticated with JWT
2. **Frontend ↔ Backend**: JWT validation; backend enforces permissions before querying DB or contracts
3. **Backend ↔ Contracts**: Contracts are immutable source of truth; backend reads events, never overwrites contract state

## External Dependencies

| Dependency              | Purpose                                  | Risk                                                   |
| ----------------------- | ---------------------------------------- | ------------------------------------------------------ |
| **Stellar RPC Node**    | Access to contract events and state      | High availability required; fallback nodes recommended |
| **PostgreSQL Database** | Persistent storage of scores and history | Data consistency critical; backups required            |
| **Redis**               | Session cache and query result caching   | Performance optimization; loss is recoverable          |
| **Sentry**              | Error tracking and monitoring            | Optional but recommended for production                |

## Key Files & Documentation

- **[Contract State Machine](./contract-state-machine.md)** – Loan lifecycle and contract state transitions
- **[Indexer Sync Flow](./indexer-sync-flow.md)** – Detailed explanation of event polling and database synchronization
- **[Frontend Patterns](./frontend-patterns.md)** – React component conventions and reusable UI patterns
- **[DESIGN.md](../DESIGN.md)** – Product vision, UI/UX decisions, and visual design language

## Running the System Locally

Typically:

1. Start PostgreSQL (or connect to remote instance)
2. Start Stellar RPC (or use testnet/mainnet)
3. Deploy Soroban contracts to testnet
4. Start backend: `npm start` (runs API + indexer)
5. Start frontend: `npm run dev` (runs Next.js)

See each service's README for specific setup instructions.

## See Also

- [Soroban Documentation](https://developers.stellar.org/docs/learn/storing-data)
- [Stellar RPC API](https://developers.stellar.org/docs/data/rpc/api-reference)
- [RemitLend Wiki](./README.md)
