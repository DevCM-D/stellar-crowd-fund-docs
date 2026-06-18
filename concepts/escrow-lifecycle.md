# Escrow Lifecycle

An escrow on Stellar Crowd Fund Escrow is a Soroban smart contract instance that holds funds and enforces the rules of a work agreement. Understanding its lifecycle is essential to working with the platform.

---

## States

Every escrow exists in exactly one state at any given time:

| State | Meaning |
|---|---|
| `Active` | Funds are locked; milestones are pending or in progress |
| `Disputed` | Either party has raised a dispute; funds are frozen pending resolution |
| `Completed` | All milestones approved; funds fully disbursed; escrow closed |
| `Cancelled` | Both parties agreed to cancel; remaining funds refunded to client |

---

## State Transitions

```
                    ┌──────────────────────────────┐
                    │                              │
                    ▼                              │
              ┌──────────┐                         │
  create ──► │  Active  │                         │
              └──────────┘                         │
                  │   │                            │
     dispute ─────┘   └──── all milestones ────► Completed
        │                    approved
        ▼
   ┌──────────┐
   │ Disputed │
   └──────────┘
        │
   arbiter resolves
        │
        ▼
    Completed

              Active ──── mutual consent ────► Cancelled
```

### Transitions table

| From | To | Who triggers | Contract function |
|---|---|---|---|
| (new) | `Active` | Client | `create_escrow` |
| `Active` | `Completed` | System (last milestone approved) | `approve_milestone` |
| `Active` | `Disputed` | Client or freelancer | `raise_dispute` |
| `Active` | `Cancelled` | Both parties (mutual consent) | `cancel_escrow` |
| `Disputed` | `Completed` | Arbiter | `resolve_dispute` |

---

## Creating an Escrow

The client calls `create_escrow` with:
- The freelancer's Stellar address
- The token contract address (e.g. USDC on Stellar)
- Total amount in base units (7 decimal places for most Stellar assets)
- An array of milestones, each with an amount and description IPFS hash

The contract locks the full amount immediately. The client cannot withdraw the funds unilaterally after this point.

```
create_escrow(
  freelancer: Address,
  token: Address,
  amount: i128,
  milestones: Vec<MilestoneInput>
) → escrow_id: u64
```

The returned `escrow_id` is the on-chain identifier for all future interactions with this escrow.

---

## Milestone Submission

The freelancer signals that a milestone is ready for review:

```
submit_milestone(
  escrow_id: u64,
  milestone_index: u32,
  ipfs_hash: String
)
```

The `ipfs_hash` is a content-addressed pointer to the deliverable (a document, design files, code repository snapshot, etc.). This hash is stored on-chain and cannot be altered, giving the client an immutable record of exactly what was delivered for review.

---

## Milestone Approval

The client reviews the submitted work and approves the milestone:

```
approve_milestone(
  escrow_id: u64,
  milestone_index: u32
)
```

On approval:
1. The milestone's funds are transferred to the freelancer's address
2. A `ReputationEvent` is written for both client and freelancer
3. If this was the last milestone, the escrow transitions to `Completed`

The client cannot un-approve a milestone once confirmed — the transaction is on-chain.

---

## Disputes

Either party can raise a dispute at any time while the escrow is `Active`:

```
raise_dispute(
  escrow_id: u64,
  reason: String
)
```

After a dispute is raised:
- The escrow transitions to `Disputed`
- No further milestones can be submitted or approved
- Both parties can upload evidence files to IPFS via the REST API
- The evidence hashes are recorded in the off-chain database and referenced in resolution

An authorised arbiter (set at escrow creation or by platform governance) resolves the dispute:

```
resolve_dispute(
  escrow_id: u64,
  client_amount: i128,
  freelancer_amount: i128
)
```

The arbiter specifies how the remaining locked funds are split. The sum of `client_amount + freelancer_amount` must equal the remaining balance. The contract enforces this.

After resolution:
- Funds are transferred according to the split
- Reputation events are written for both parties, reflecting the dispute outcome
- The escrow transitions to `Completed`

---

## Cancellation

Both parties can agree to cancel:

```
cancel_escrow(escrow_id: u64)
```

This requires both the client's and freelancer's signed authorisation (implemented as a two-step confirmation: one party initiates, the other confirms). On cancellation:
- All remaining locked funds are refunded to the client
- No reputation events are written (cancellation is neutral)
- The escrow transitions to `Cancelled`

---

## Escrow in the API

The REST API mirrors the contract state. Escrow objects returned from the API include:

```json
{
  "id": "12345",
  "clientAddress": "GABCD...",
  "freelancerAddress": "GXYZ...",
  "tokenAddress": "USDC_CONTRACT_ID",
  "totalAmount": "2000000000",
  "remainingBalance": "1000000000",
  "status": "Active",
  "briefHash": "QmIPFSHash...",
  "deadline": "2025-12-31T00:00:00Z",
  "createdAt": "2025-03-01T10:00:00Z",
  "milestones": [...]
}
```

`totalAmount` and `remainingBalance` are strings representing the token amount in base units (multiply by 10^-7 for the human-readable value for most Stellar assets).

---

## Related

- [Milestones](milestones.md) — detailed milestone states and transitions
- [Disputes](disputes.md) — evidence submission, resolution, and appeals
- [Reputation System](reputation-system.md) — how outcomes affect on-chain scores
- [API Reference → Escrows](../api-reference/escrows.md) — REST endpoints
