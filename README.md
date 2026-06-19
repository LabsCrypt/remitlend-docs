# RemitLend Documentation Repository

## Overview
This repository houses the **design and technical documentation** for the RemitLend platform. It provides a single source of truth for architecture diagrams, UI/UX designs, transaction flow specifications, and the in‑repo wiki that contains deeper technical references.

## Relationship to Other RemitLend Repositories
- **remitlend** – The main application codebase (frontend + backend). This docs repo explains the design decisions that drive that code.
- **remitlend-backend** – Backend services; see the design docs here for API contracts and data flow.
- **remitlend-contracts** – Soroban smart contracts; the wiki contains contract state‑machine diagrams and sync flows.
- **remitlend-frontend** – Front‑end UI; the UI/UX design docs describe the components and interactions implemented there.

## Documentation Map
| Document | Summary |
|----------|---------|
| [DESIGN.md](./DESIGN.md) | Product, UI/UX, and gamification design specifications |
| [LANDING_PAGE_DESIGN.md](./LANDING_PAGE_DESIGN.md) | Landing page layout, marketing copy, and visual assets |
| [TRANSACTION_PREVIEW.md](./TRANSACTION_PREVIEW.md) | Detailed design of the transaction preview interface |
| [wiki/README.md](./wiki/README.md) | Overview of the in‑repo wiki and how to contribute |
| [wiki/contract-state-machine.md](./wiki/contract-state-machine.md) | Soroban contract state‑machine diagram and description |
| [wiki/indexer-sync-flow.md](./wiki/indexer-sync-flow.md) | Flow of data syncing between the indexer and the database |
| [wiki/frontend-patterns.md](./wiki/frontend-patterns.md) | Common frontend patterns and "standard library" components |
| [runbook.md](./runbook.md) | Operations runbook: env vars, staging deploy, healthchecks, rollback, and troubleshooting |
| [api-reference.md](./api-reference.md) | REST API reference: endpoints, schemas, auth, errors, rate limits, and idempotency |

## Contributing
If you’d like to add or improve documentation, start by reading our **contribution guide**: [CONTRIBUTING.md](./CONTRIBUTING.md).

- Follow the markdown style guide (headings, code blocks, mermaid diagrams).
- Use the provided templates for reference, how‑to, and design pages.
- Submit changes via a PR; reviewers will ensure consistency and completeness.

---

Have questions? Open an issue or reach out to the maintainers.
