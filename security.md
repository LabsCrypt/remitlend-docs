# RemitLend Security Model

This document outlines the security-critical mechanisms across the RemitLend codebase, including authentication flows, role-based access control, secure internal communication, and governance.

## 1. Authentication Flow

RemitLend uses a standard cryptographic challenge-verify authentication flow for users interacting via web3 wallets.

- **Challenge Issuance:** The client requests a nonce/challenge from the backend. The backend generates a secure, time-bound challenge string.
- **Verification:** The client signs the challenge using their private key and submits the signature. The backend verifies the signature against the provided public address.
- **JWT Issuance:** Upon successful verification, the backend issues a JSON Web Token (JWT). The JWT contains the user's public address and assigned roles.
- **JWT Validation:** All protected API endpoints validate the JWT signature using a secure secret. If the token is valid, the extracted address and roles are attached to the request context.

## 2. Role-Based Access Control (RBAC)

Access to sensitive API routes and contract functions is governed by RBAC. The actual role assignment is determined by the `resolveRoleForWallet` function (see `src/auth/rbac.ts`), which checks against configured wallet lists (e.g., `ADMIN_WALLETS` and `LENDER_WALLETS`).

- **`borrower` / `user`**: The default role assigned to all authenticated addresses. Can view their own balances, initiate transfers, and manage their profile.
- **`lender`**: A specialized role for liquidity providers to manage loan pools and view lending metrics.
- **`admin`**: Can execute critical state mutations. Admin operations are heavily audited and, where applicable, routed through the multisig timelock.

## 3. Internal API Key & Webhook HMAC Verification

To secure communication between RemitLend microservices and external webhook subscribers:

- **Internal API Key:** Administrative mutations and internal cron jobs utilize a strictly rotated internal API key. This key is passed via the `x-api-key` header (tied to the `INTERNAL_API_KEY` configuration) and validated by edge middleware before reaching the core services. It must never be exposed to the client.
- **Webhook HMAC Signatures:** RemitLend signs its OUTGOING webhooks so that third-party subscribers can verify the authenticity of the payloads.
  - The payload is hashed using HMAC-SHA256 with a pre-shared webhook secret.
  - The resulting signature is attached to the outbound request via the `x-remitlend-signature` header, allowing subscribers to securely verify that the event originated from RemitLend using a constant-time string comparison.

## 4. Multisig Governance & Timelock

Administrative governance is strictly managed via a multisig wallet and enforced by a timelock contract, specifically for the admin-transfer flow of a single target contract.

- **Multisig Threshold:** Administrative actions require a minimum threshold of signatures (e.g., 3-of-5) from trusted signers.
- **Timelock Guarantees:** Once a transaction is queued by the multisig, it enters a mandatory minimum 24-hour timelock delay, with a 7-day proposal TTL. This ensures that no sudden, unilateral changes can be made without giving the community time to review.
- **Admin-Transfer Flow:** The transfer of the "admin" role from one address to another is a two-step process: 
  1. The current admin proposes the transfer and queues it in the timelock.
  2. After the timelock expires, the new admin must explicitly call `finalize_admin_transfer` to complete the transfer, preventing accidental loss of control.
