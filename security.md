# RemitLend Security Model

This document outlines the security-critical mechanisms across the RemitLend codebase, including authentication flows, role-based access control, secure internal communication, and governance.

## 1. Authentication Flow

RemitLend uses a standard cryptographic challenge-verify authentication flow for users interacting via web3 wallets.

- **Challenge Issuance:** The client requests a nonce/challenge from the backend. The backend generates a secure, time-bound challenge string.
- **Verification:** The client signs the challenge using their private key and submits the signature. The backend verifies the signature against the provided public address.
- **JWT Issuance:** Upon successful verification, the backend issues a JSON Web Token (JWT). The JWT contains the user's public address and assigned roles.
- **JWT Validation:** All protected API endpoints validate the JWT signature using a secure secret. If the token is valid, the extracted address and roles are attached to the request context.

## 2. Role-Based Access Control (RBAC)

Access to sensitive API routes and contract functions is governed by RBAC.

- **`user`**: The default role assigned to all authenticated addresses. Can view their own balances, initiate transfers, and manage their profile.
- **`moderator`**: Can view platform-wide metrics, flag suspicious transactions for review, but cannot execute administrative mutations.
- **`admin`**: Can execute critical state mutations (e.g., pausing contracts, updating fees). Admin operations are heavily audited and, where applicable, routed through the multisig timelock.
- **`system`**: An internal role used exclusively by automated backend jobs and trusted microservices.

## 3. Internal API Key & Webhook HMAC Verification

To secure communication between RemitLend microservices and external webhook providers:

- **Internal API Key:** Administrative mutations and internal cron jobs utilize a strictly rotated internal API key. This key is passed via the `X-Internal-Secret` header and validated by edge middleware before reaching the core services. It must never be exposed to the client.
- **Webhook HMAC Signatures:** All incoming webhooks (e.g., from indexing services or payment processors) must include an `X-Signature` header. 
  - The payload is hashed using HMAC-SHA256 with a pre-shared webhook secret.
  - The backend recalculates the hash and compares it using a constant-time string comparison to prevent timing attacks. If the signatures do not match, the request is immediately dropped.

## 4. Multisig Governance & Timelock

All smart contract administrative actions (upgrades, fee adjustments, admin-transfer flows) are decentralized via a multisig wallet and enforced by a timelock contract.

- **Multisig Threshold:** Administrative actions require a minimum threshold of signatures (e.g., 3-of-5) from trusted signers.
- **Timelock Guarantees:** Once a transaction is queued by the multisig, it enters a mandatory timelock period (e.g., 48 hours). This ensures that no sudden, unilateral changes can be made without giving the community time to review.
- **Admin-Transfer Flow:** The transfer of the "admin" role from one address to another is a two-step process: 
  1. The current admin proposes the transfer and queues it in the timelock.
  2. After the timelock expires, the new admin must explicitly `acceptAdmin` to complete the transfer, preventing accidental loss of control.
