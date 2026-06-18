# Soroban Contract Reference

This document is the canonical reference for all public interfaces, emitted events, error codes, and storage keys across the four Remitlend Soroban smart contracts. It is intended for integrators, auditors, and contributors.

For the high-level loan lifecycle and state transitions, see [wiki/contract-state-machine.md](./wiki/contract-state-machine.md).

---

## Table of Contents

1. [lending_pool](#1-lending_pool)
2. [loan_manager](#2-loan_manager)
3. [multisig_governance](#3-multisig_governance)
4. [remittance_nft](#4-remittance_nft)

---

## 1. `lending_pool`

The `lending_pool` contract holds deposited liquidity from lenders. It mints and burns internal share tokens to track each depositor's proportional ownership, and exposes a transfer interface used exclusively by `loan_manager` to disburse and collect loan funds.

### 1.1 Public Functions

| Function | Signature | Description |
|---|---|---|
| `initialize` | `initialize(env: Env, admin: Address, token: Address)` | One-time setup. Sets the admin and the underlying token (e.g. USDC) accepted by the pool. |
| `deposit` | `deposit(env: Env, depositor: Address, amount: i128) -> i128` | Transfers `amount` of the underlying token from `depositor` into the pool and mints a proportional amount of pool shares. Returns shares minted. |
| `withdraw` | `withdraw(env: Env, depositor: Address, shares: i128) -> i128` | Burns `shares` and transfers the corresponding underlying token amount back to `depositor`. Returns underlying tokens returned. |
| `transfer_to_borrower` | `transfer_to_borrower(env: Env, borrower: Address, amount: i128)` | Restricted to the `loan_manager` contract. Transfers `amount` of the underlying token to `borrower` on loan approval. |
| `receive_repayment` | `receive_repayment(env: Env, amount: i128)` | Restricted to the `loan_manager` contract. Receives repayment principal plus interest back into the pool. |
| `get_balance` | `get_balance(env: Env) -> i128` | Returns the total underlying token balance currently held by the pool. |
| `get_shares` | `get_shares(env: Env, depositor: Address) -> i128` | Returns the number of pool shares held by `depositor`. |
| `set_loan_manager` | `set_loan_manager(env: Env, loan_manager: Address)` | Admin-only. Registers the authorised `loan_manager` address allowed to call `transfer_to_borrower` and `receive_repayment`. |
| `pause` | `pause(env: Env)` | Admin-only. Halts `deposit` and `withdraw` operations. |
| `unpause` | `unpause(env: Env)` | Admin-only. Resumes `deposit` and `withdraw` operations. |

### 1.2 Emitted Events

| Event Symbol | Topics | Data | Description |
|---|---|---|---|
| `Deposited` | `["Deposited", depositor: Address]` | `{ amount: i128, shares_minted: i128 }` | Emitted after a successful deposit. |
| `Withdrawn` | `["Withdrawn", depositor: Address]` | `{ shares_burned: i128, amount_returned: i128 }` | Emitted after a successful withdrawal. |
| `LoanDisbursed` | `["LoanDisbursed", borrower: Address]` | `{ amount: i128 }` | Emitted when funds are sent to a borrower by `loan_manager`. |
| `RepaymentReceived` | `["RepaymentReceived"]` | `{ amount: i128 }` | Emitted when a repayment is received back into the pool. |
| `Paused` | `["Paused", admin: Address]` | `{}` | Emitted when the pool is paused. |
| `Unpaused` | `["Unpaused", admin: Address]` | `{}` | Emitted when the pool is unpaused. |

### 1.3 Error Codes

| Variant | Value | Trigger |
|---|---|---|
| `NotInitialized` | 1 | Any function called before `initialize`. |
| `AlreadyInitialized` | 2 | `initialize` called more than once. |
| `Unauthorized` | 3 | Caller is not the admin or the registered `loan_manager`. |
| `InsufficientBalance` | 4 | Pool does not have enough underlying tokens to satisfy `withdraw` or `transfer_to_borrower`. |
| `InsufficientShares` | 5 | Depositor does not hold enough shares to satisfy `withdraw`. |
| `InvalidAmount` | 6 | `amount` or `shares` is zero or negative. |
| `ContractPaused` | 7 | `deposit` or `withdraw` called while the pool is paused. |
| `LoanManagerNotSet` | 8 | `transfer_to_borrower` or `receive_repayment` called before `set_loan_manager`. |

### 1.4 Storage Keys

| Key | Type | Storage Class | TTL Expectation | Description |
|---|---|---|---|---|
| `DataKey::Admin` | `Address` | Instance | Permanent (no TTL bump required) | Contract administrator. |
| `DataKey::Token` | `Address` | Instance | Permanent | The accepted underlying token address. |
| `DataKey::LoanManager` | `Address` | Instance | Permanent | Registered `loan_manager` contract address. |
| `DataKey::TotalShares` | `i128` | Instance | Permanent | Running total of all outstanding pool shares. |
| `DataKey::Paused` | `bool` | Instance | Permanent | Global pause flag. |
| `DataKey::Shares(Address)` | `i128` | Persistent | Bump on every deposit/withdraw | Per-depositor share balance. |

---

## 2. `loan_manager`

The `loan_manager` contract is the primary orchestrator of the lending protocol. It handles loan requests, approvals, repayments, and defaults, coordinating with `lending_pool` for fund movement and `remittance_nft` for credit score updates.

### 2.1 Public Functions

| Function | Signature | Description |
|---|---|---|
| `initialize` | `initialize(env: Env, admin: Address, lending_pool: Address, nft_contract: Address, min_score: u32)` | One-time setup. Registers the admin, linked contracts, and the minimum credit score required to borrow. |
| `request_loan` | `request_loan(env: Env, borrower: Address, amount: i128) -> u32` | Opens a new loan in `Pending` state. Verifies the borrower holds a `RemittanceNft` and meets `min_score`. Returns the new `loan_id`. |
| `approve_loan` | `approve_loan(env: Env, loan_id: u32)` | Admin-only. Moves loan from `Pending` to `Approved` and calls `lending_pool::transfer_to_borrower`. |
| `repay` | `repay(env: Env, borrower: Address, loan_id: u32, amount: i128)` | Accepts a repayment from `borrower`. Marks loan `Repaid` on full settlement and calls `remittance_nft::update_score`. |
| `mark_defaulted` | `mark_defaulted(env: Env, loan_id: u32)` | Admin-only. Moves an overdue `Approved` loan to `Defaulted` state and calls `remittance_nft::update_score` with a penalty. |
| `get_loan` | `get_loan(env: Env, loan_id: u32) -> Loan` | Returns the full `Loan` struct for the given `loan_id`. |
| `get_loan_count` | `get_loan_count(env: Env) -> u32` | Returns the total number of loans ever created. |
| `set_min_score` | `set_min_score(env: Env, min_score: u32)` | Admin-only. Updates the minimum credit score threshold for new loan requests. |
| `pause` | `pause(env: Env)` | Admin-only. Halts `request_loan`, `approve_loan`, and `repay`. |
| `unpause` | `unpause(env: Env)` | Admin-only. Resumes all operations. |

#### `Loan` struct

```rust
pub struct Loan {
    pub id: u32,
    pub borrower: Address,
    pub amount: i128,
    pub amount_repaid: i128,
    pub status: LoanStatus,
    pub created_ledger: u32,
    pub due_ledger: u32,
}

pub enum LoanStatus {
    Pending,
    Approved,
    Repaid,
    Defaulted,
}
```

### 2.2 Emitted Events

| Event Symbol | Topics | Data | Description |
|---|---|---|---|
| `LoanRequested` | `["LoanRequested", borrower: Address, loan_id: u32]` | `{ amount: i128 }` | Emitted when a new loan enters `Pending` state. |
| `LoanApproved` | `["LoanApproved", loan_id: u32]` | `{ borrower: Address, amount: i128 }` | Emitted when a loan moves to `Approved` and funds are disbursed. |
| `LoanRepaid` | `["LoanRepaid", loan_id: u32, borrower: Address]` | `{ amount_repaid: i128, fully_repaid: bool }` | Emitted on each repayment call. `fully_repaid` is `true` when status transitions to `Repaid`. |
| `LoanDefaulted` | `["LoanDefaulted", loan_id: u32, borrower: Address]` | `{}` | Emitted when admin marks a loan as `Defaulted`. |
| `Paused` | `["Paused", admin: Address]` | `{}` | Emitted when the contract is paused. |
| `Unpaused` | `["Unpaused", admin: Address]` | `{}` | Emitted when the contract is unpaused. |

### 2.3 Error Codes

| Variant | Value | Trigger |
|---|---|---|
| `NotInitialized` | 1 | Any function called before `initialize`. |
| `AlreadyInitialized` | 2 | `initialize` called more than once. |
| `Unauthorized` | 3 | Caller is not the admin. |
| `LoanNotFound` | 4 | `loan_id` does not map to an existing loan. |
| `InvalidLoanStatus` | 5 | Operation is invalid for the loan's current status (e.g. approving an already-approved loan). |
| `InsufficientCreditScore` | 6 | Borrower's score is below `min_score`. |
| `NftNotFound` | 7 | Borrower does not hold a `RemittanceNft`. |
| `ActiveLoanExists` | 8 | Borrower already has a loan in `Pending` or `Approved` state (one active loan per borrower). |
| `InvalidAmount` | 9 | `amount` is zero or negative. |
| `ContractPaused` | 10 | Operation called while the contract is paused. |
| `PoolInsufficientFunds` | 11 | `lending_pool` does not have enough liquidity to cover an approval. |

### 2.4 Storage Keys

| Key | Type | Storage Class | TTL Expectation | Description |
|---|---|---|---|---|
| `DataKey::Admin` | `Address` | Instance | Permanent | Contract administrator. |
| `DataKey::LendingPool` | `Address` | Instance | Permanent | Linked `lending_pool` contract address. |
| `DataKey::NftContract` | `Address` | Instance | Permanent | Linked `remittance_nft` contract address. |
| `DataKey::MinScore` | `u32` | Instance | Permanent | Minimum credit score for borrowers. |
| `DataKey::LoanCounter` | `u32` | Instance | Permanent | Monotonically increasing loan ID counter. |
| `DataKey::Paused` | `bool` | Instance | Permanent | Global pause flag. |
| `DataKey::Loan(u32)` | `Loan` | Persistent | Bump on every status change | Full loan record keyed by `loan_id`. |

---

## 3. `multisig_governance`

The `multisig_governance` contract provides an M-of-N multi-signature approval mechanism for sensitive administrative actions (e.g. upgrading contracts, adjusting protocol parameters). No action takes effect until the required threshold of signers has approved the corresponding proposal.

### 3.1 Public Functions

| Function | Signature | Description |
|---|---|---|
| `initialize` | `initialize(env: Env, signers: Vec<Address>, threshold: u32)` | One-time setup. Registers the initial signer set and the minimum approval count required to execute a proposal. |
| `propose` | `propose(env: Env, proposer: Address, action: ProposalAction, expiry_ledger: u32) -> u32` | Creates a new proposal. `proposer` must be a registered signer. Returns the new `proposal_id`. |
| `approve` | `approve(env: Env, signer: Address, proposal_id: u32)` | Records `signer`'s approval for `proposal_id`. Automatically executes the proposal if the approval count reaches `threshold`. |
| `revoke` | `revoke(env: Env, signer: Address, proposal_id: u32)` | Removes `signer`'s prior approval from `proposal_id` before it is executed. |
| `execute` | `execute(env: Env, proposal_id: u32)` | Explicitly triggers execution of a proposal that has reached `threshold`. Callable by any signer. |
| `add_signer` | `add_signer(env: Env, new_signer: Address)` | Governance-only (must be called via an approved proposal). Adds a new signer to the set. |
| `remove_signer` | `remove_signer(env: Env, signer: Address)` | Governance-only. Removes a signer, provided the remaining signer count still satisfies `threshold`. |
| `set_threshold` | `set_threshold(env: Env, threshold: u32)` | Governance-only. Updates the approval threshold. Cannot exceed the current signer count. |
| `get_proposal` | `get_proposal(env: Env, proposal_id: u32) -> Proposal` | Returns the full `Proposal` struct. |
| `get_signers` | `get_signers(env: Env) -> Vec<Address>` | Returns the current list of registered signers. |

#### `Proposal` struct

```rust
pub struct Proposal {
    pub id: u32,
    pub proposer: Address,
    pub action: ProposalAction,
    pub approvals: Vec<Address>,
    pub executed: bool,
    pub expiry_ledger: u32,
}

pub enum ProposalAction {
    UpgradeContract { target: Address, new_wasm_hash: BytesN<32> },
    SetMinScore { value: u32 },
    PauseContract { target: Address },
    UnpauseContract { target: Address },
    TransferAdmin { target: Address, new_admin: Address },
    AddSigner { signer: Address },
    RemoveSigner { signer: Address },
    SetThreshold { value: u32 },
}
```

### 3.2 Emitted Events

| Event Symbol | Topics | Data | Description |
|---|---|---|---|
| `ProposalCreated` | `["ProposalCreated", proposer: Address, proposal_id: u32]` | `{ action_type: Symbol, expiry_ledger: u32 }` | Emitted when a new proposal is created. |
| `ProposalApproved` | `["ProposalApproved", signer: Address, proposal_id: u32]` | `{ approval_count: u32, threshold: u32 }` | Emitted each time a signer approves a proposal. |
| `ApprovalRevoked` | `["ApprovalRevoked", signer: Address, proposal_id: u32]` | `{}` | Emitted when a signer revokes their approval. |
| `ProposalExecuted` | `["ProposalExecuted", proposal_id: u32]` | `{ action_type: Symbol }` | Emitted when a proposal reaches threshold and is executed. |
| `ProposalExpired` | `["ProposalExpired", proposal_id: u32]` | `{}` | Emitted when an attempt is made to approve or execute an expired proposal. |
| `SignerAdded` | `["SignerAdded", new_signer: Address]` | `{}` | Emitted when a new signer is added via governance. |
| `SignerRemoved` | `["SignerRemoved", signer: Address]` | `{}` | Emitted when a signer is removed via governance. |
| `ThresholdUpdated` | `["ThresholdUpdated"]` | `{ new_threshold: u32 }` | Emitted when the approval threshold is changed. |

### 3.3 Error Codes

| Variant | Value | Trigger |
|---|---|---|
| `NotInitialized` | 1 | Any function called before `initialize`. |
| `AlreadyInitialized` | 2 | `initialize` called more than once. |
| `Unauthorized` | 3 | Caller is not a registered signer, or action requires prior governance approval. |
| `ProposalNotFound` | 4 | `proposal_id` does not map to an existing proposal. |
| `ProposalAlreadyExecuted` | 5 | `approve` or `execute` called on an already-executed proposal. |
| `ProposalExpired` | 6 | `approve` or `execute` called after `expiry_ledger`. |
| `AlreadyApproved` | 7 | Signer has already approved the given proposal. |
| `NotApproved` | 8 | `revoke` called by a signer who has not approved the proposal. |
| `ThresholdNotMet` | 9 | `execute` called explicitly before `threshold` approvals have been recorded. |
| `InvalidThreshold` | 10 | Threshold is zero or exceeds the current signer count. |
| `SignerAlreadyExists` | 11 | `add_signer` called with an address already in the signer set. |
| `SignerNotFound` | 12 | `remove_signer` called with an address not in the signer set. |
| `ThresholdViolation` | 13 | Removing a signer would leave fewer signers than `threshold`. |

### 3.4 Storage Keys

| Key | Type | Storage Class | TTL Expectation | Description |
|---|---|---|---|---|
| `DataKey::Signers` | `Vec<Address>` | Instance | Permanent | Current set of registered signers. |
| `DataKey::Threshold` | `u32` | Instance | Permanent | Minimum approvals required to execute a proposal. |
| `DataKey::ProposalCounter` | `u32` | Instance | Permanent | Monotonically increasing proposal ID counter. |
| `DataKey::Proposal(u32)` | `Proposal` | Persistent | Bump on every approval/revocation | Full proposal record keyed by `proposal_id`. |

---

## 4. `remittance_nft`

The `remittance_nft` contract implements a soulbound (non-transferable) NFT that represents a user's on-chain identity and creditworthiness within the Remitlend protocol. Each address may hold at most one NFT. The embedded credit score is updated by the `loan_manager` contract after repayments and defaults, and by an authorized backend oracle for off-chain credit events.

### 4.1 Public Functions

| Function | Signature | Description |
|---|---|---|
| `initialize` | `initialize(env: Env, admin: Address, minter: Address)` | One-time setup. Sets the admin and the authorized `minter` address (typically the backend oracle). |
| `mint` | `mint(env: Env, recipient: Address)` | Minter-only. Issues a new `RemittanceNft` to `recipient` with an initial score of `500`. Each address may only hold one NFT. |
| `update_score` | `update_score(env: Env, user: Address, delta: i32)` | Restricted to the registered `loan_manager` contract or the `minter`. Applies `delta` (positive or negative) to `user`'s score, clamped to `[0, 1000]`. |
| `burn` | `burn(env: Env, user: Address)` | Admin-only. Permanently removes the NFT and credit record for `user`. |
| `get_score` | `get_score(env: Env, user: Address) -> u32` | Returns the current credit score for `user`. |
| `has_nft` | `has_nft(env: Env, user: Address) -> bool` | Returns `true` if `user` holds a `RemittanceNft`. |
| `set_minter` | `set_minter(env: Env, new_minter: Address)` | Admin-only. Rotates the authorised minter/oracle address. |
| `set_loan_manager` | `set_loan_manager(env: Env, loan_manager: Address)` | Admin-only. Registers the `loan_manager` contract address permitted to call `update_score`. |

### 4.2 Emitted Events

| Event Symbol | Topics | Data | Description |
|---|---|---|---|
| `NftMinted` | `["NftMinted", recipient: Address]` | `{ initial_score: u32 }` | Emitted when a new NFT is issued. |
| `ScoreUpdated` | `["ScoreUpdated", user: Address]` | `{ old_score: u32, new_score: u32, delta: i32 }` | Emitted after every successful score update. |
| `NftBurned` | `["NftBurned", user: Address]` | `{}` | Emitted when an NFT is permanently revoked. |
| `MinterUpdated` | `["MinterUpdated"]` | `{ new_minter: Address }` | Emitted when the authorized minter is rotated. |

### 4.3 Error Codes

| Variant | Value | Trigger |
|---|---|---|
| `NotInitialized` | 1 | Any function called before `initialize`. |
| `AlreadyInitialized` | 2 | `initialize` called more than once. |
| `Unauthorized` | 3 | Caller is not the admin, minter, or registered `loan_manager`. |
| `NftAlreadyExists` | 4 | `mint` called for an address that already holds an NFT. |
| `NftNotFound` | 5 | `update_score`, `burn`, or `get_score` called for an address with no NFT. |
| `LoanManagerNotSet` | 6 | `update_score` called from `loan_manager` before `set_loan_manager` was invoked. |
| `InvalidDelta` | 7 | `delta` is zero (no-op updates are rejected). |

### 4.4 Storage Keys

| Key | Type | Storage Class | TTL Expectation | Description |
|---|---|---|---|---|
| `DataKey::Admin` | `Address` | Instance | Permanent | Contract administrator. |
| `DataKey::Minter` | `Address` | Instance | Permanent | Authorized minter / backend oracle address. |
| `DataKey::LoanManager` | `Address` | Instance | Permanent | Registered `loan_manager` contract address. |
| `DataKey::Score(Address)` | `u32` | Persistent | Bump on every score update | Per-user credit score `[0, 1000]`. |
| `DataKey::NftHolder(Address)` | `bool` | Persistent | Bump on mint; removed on burn | Presence flag indicating a user holds an NFT. |

---

## Cross-Contract Interaction Overview

```
         ┌─────────────────────────────────────────┐
         │            multisig_governance           │
         │  (admin proposals: pause, upgrade, etc.) │
         └───────────────────┬─────────────────────┘
                             │ governance actions
                ┌────────────▼────────────┐
                │      loan_manager       │◄──── borrower calls
                │ request / approve / repay│
                └────┬──────────┬─────────┘
      disburse/repay │          │ update_score
         ┌───────────▼──┐  ┌───▼──────────────┐
         │ lending_pool │  │  remittance_nft   │
         │ (liquidity)  │  │  (credit scores)  │
         └──────────────┘  └───────────────────┘
```

- `loan_manager` calls `lending_pool::transfer_to_borrower` on approval and `lending_pool::receive_repayment` on repayment.
- `loan_manager` calls `remittance_nft::update_score` with `+15` on full repayment and `-50` on default.
- `multisig_governance` executes admin actions on all contracts through approved proposals.

---

## Score Delta Reference

| Event | Delta | Called by |
|---|---|---|
| NFT minted | `+0` (initial score is `500`) | `remittance_nft::mint` |
| Full loan repayment | `+15` | `loan_manager::repay` |
| Loan default | `-50` | `loan_manager::mark_defaulted` |
| Off-chain credit event (positive) | `+1` to `+30` | backend oracle via `update_score` |
| Off-chain credit event (negative) | `-1` to `-30` | backend oracle via `update_score` |
