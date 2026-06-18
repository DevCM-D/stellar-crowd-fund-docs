# Smart Contracts

The Soroban contracts are written in Rust and compiled to WebAssembly. They run on the Stellar network and hold the financial source of truth for every escrow on the platform.

---

## Contract Modules

The contract codebase is structured as a single deployable WASM binary with several Rust modules:

```
contracts/escrow/
├── src/
│   ├── lib.rs           — contract entry point; #[contract] trait impl
│   ├── escrow.rs        — create, cancel, status queries
│   ├── milestone.rs     — submit, approve, reject milestone
│   ├── dispute.rs       — raise, resolve dispute
│   ├── reputation.rs    — emit and query ReputationEvent
│   ├── types.rs         — shared structs and enums (EscrowStatus, Milestone, etc.)
│   └── errors.rs        — ContractError enum
├── Cargo.toml
└── Makefile             — build, test, deploy targets
```

---

## Data Structures

### Escrow

```rust
pub struct Escrow {
    pub id: u64,
    pub client: Address,
    pub freelancer: Address,
    pub arbiter: Address,
    pub token: Address,
    pub total_amount: i128,
    pub remaining_balance: i128,
    pub status: EscrowStatus,
    pub milestones: Vec<Milestone>,
    pub created_at: u64,   // ledger timestamp
}
```

### EscrowStatus

```rust
pub enum EscrowStatus {
    Active,
    Disputed,
    Completed,
    Cancelled,
}
```

### Milestone

```rust
pub struct Milestone {
    pub index: u32,
    pub amount: i128,
    pub description_hash: String,   // IPFS CID
    pub status: MilestoneStatus,
    pub submission_hash: Option<String>,  // IPFS CID of deliverable
    pub submitted_at: Option<u64>,
    pub resolved_at: Option<u64>,
}
```

### MilestoneStatus

```rust
pub enum MilestoneStatus {
    Pending,
    Submitted,
    Approved,
    Rejected,
}
```

### ReputationEvent

```rust
pub struct ReputationEvent {
    pub address: Address,
    pub event_type: ReputationEventType,
    pub escrow_id: u64,
    pub score_delta: i64,
    pub timestamp: u64,
}

pub enum ReputationEventType {
    EscrowCompletedAsClient,
    EscrowCompletedAsFreelancer,
    MilestoneApproved,
    DisputeWon,
    DisputeLost,
    DisputeRaised,
}
```

---

## Contract Interface

### `create_escrow`

```rust
fn create_escrow(
    env: Env,
    client: Address,
    freelancer: Address,
    arbiter: Address,
    token: Address,
    amount: i128,
    milestones: Vec<MilestoneInput>,
) -> Result<u64, ContractError>
```

- Validates that milestone amounts sum to `amount`
- Transfers `amount` tokens from `client` to the contract's escrow account
- Emits a storage write for the new escrow
- Returns the new escrow ID

**Auth:** Requires `client` to have signed the transaction.

---

### `submit_milestone`

```rust
fn submit_milestone(
    env: Env,
    escrow_id: u64,
    milestone_index: u32,
    ipfs_hash: String,
) -> Result<(), ContractError>
```

- Validates escrow is `Active`
- Validates milestone is `Pending` or `Rejected`
- Records the IPFS hash and transitions milestone to `Submitted`

**Auth:** Requires `freelancer` to have signed.

---

### `approve_milestone`

```rust
fn approve_milestone(
    env: Env,
    escrow_id: u64,
    milestone_index: u32,
) -> Result<(), ContractError>
```

- Validates escrow is `Active`
- Validates milestone is `Submitted`
- Transfers milestone amount to `freelancer`
- Emits `ReputationEvent` for both parties
- If all milestones are approved, transitions escrow to `Completed`

**Auth:** Requires `client` to have signed.

---

### `raise_dispute`

```rust
fn raise_dispute(
    env: Env,
    escrow_id: u64,
    reason: String,
) -> Result<(), ContractError>
```

- Validates escrow is `Active`
- Stores `reason` on-chain
- Transitions escrow to `Disputed`

**Auth:** Requires either `client` or `freelancer` to have signed.

---

### `resolve_dispute`

```rust
fn resolve_dispute(
    env: Env,
    escrow_id: u64,
    client_amount: i128,
    freelancer_amount: i128,
) -> Result<(), ContractError>
```

- Validates escrow is `Disputed`
- Validates `client_amount + freelancer_amount == remaining_balance`
- Transfers funds according to split
- Emits `ReputationEvent` for both parties (win/loss based on who receives more)
- Transitions escrow to `Completed`

**Auth:** Requires `arbiter` to have signed.

---

### `cancel_escrow`

```rust
fn cancel_escrow(
    env: Env,
    escrow_id: u64,
) -> Result<(), ContractError>
```

- Validates escrow is `Active`
- Enforces two-step consent: first call records consent; second call (by the other party) completes cancellation
- On completion, transfers remaining balance back to `client`
- Transitions escrow to `Cancelled`

**Auth:** Requires both `client` and `freelancer` to have signed (two separate transactions).

---

### Reputation queries

```rust
fn get_reputation(env: Env, address: Address) -> ReputationRecord
fn get_reputation_events(env: Env, address: Address) -> Vec<ReputationEvent>
fn get_leaderboard(env: Env, limit: u32) -> Vec<ReputationRecord>
```

Read-only queries — no auth required.

---

## Errors

```rust
pub enum ContractError {
    EscrowNotFound,
    MilestoneNotFound,
    InvalidStatus,
    InvalidMilestoneStatus,
    AmountMismatch,
    Unauthorized,
    InvalidMilestoneAmounts,
    DisputeNotActive,
    InsufficientBalance,
    AlreadyConsented,
    ConsentRequired,
}
```

All errors are returned as a `Result<_, ContractError>` and surface as XDR error codes that the API server maps to HTTP error responses.

---

## Building and Deploying

```bash
# Build
cd contracts/escrow
make build

# Run unit tests
make test

# Deploy to testnet (requires Soroban CLI configured)
make deploy-testnet

# Deploy to mainnet
make deploy-mainnet
```

Set `CONTRACT_ID` in your environment to the deployed contract address. The API server reads this at startup.

---

## Contract Upgrade Path

Soroban contracts are upgradeable via a WASM hash swap. The upgrade flow:

1. Build new WASM binary
2. Upload WASM to Stellar network: `soroban contract upload --wasm <binary>`
3. Call `contract.upgrade(new_wasm_hash)` from the admin address
4. Update `CONTRACT_ID` in deployment config if the contract ID changed

The upgrade is atomic at the ledger level. In-flight escrows are not affected — their data storage persists across upgrades.

---

## Related

- [Architecture Overview](overview.md) — where contracts fit in the full stack
- [Escrow Lifecycle](../concepts/escrow-lifecycle.md) — how contract functions map to lifecycle transitions
- [Disputes](../concepts/disputes.md) — dispute resolution flow in detail
- [Reputation System](../concepts/reputation-system.md) — on-chain reputation events
