# Crowd-Funding Pools

A **crowd-funding pool** replaces the single client in a standard escrow with a collective of contributors. Anyone can put XLM or USDC into the pool; when the target is reached, the funds are locked in the escrow contract and work begins under the normal milestone workflow.

---

## Why Pools Exist

Traditional freelance escrow assumes one payer. Many valuable open-source projects, community tools, and public-good initiatives have no single funder — they need a mechanism for many stakeholders to share the cost and governance of getting work done.

Crowd-funding pools solve this without a trusted intermediary:

- **No custodian.** Funds move from contributor wallets directly into the Soroban contract. No platform wallet holds contributor money.
- **Automatic refund on miss.** If the target is not reached by the deadline the contract refunds every contributor on-chain. No support ticket required.
- **Shared governance.** Contributors vote on milestone approval proportional to their contribution weight. No single party can release funds unilaterally.
- **Portable reputation.** Both the contractor and contributors earn on-chain reputation entries that travel with their Stellar address across platforms.

---

## Pool Lifecycle at a Glance

```
OPEN → FUNDING → FUNDED → ACTIVE → COMPLETED
                    │
                    └─── CANCELLED → REFUNDING → REFUNDED
```

See the [Pool Lifecycle reference](https://github.com/DevCM-D/Stellar-Crowd-Fund-Escrow/blob/develop/docs/POOL_LIFECYCLE.md) in the main repo for full state transition details.

---

## Creating a Pool

Pools are created through the web dashboard or directly via the REST API.

**Required fields:**

| Field | Description |
|-------|-------------|
| `title` | Human-readable name shown in the discovery feed |
| `description` | What work will be done and why it matters |
| `target_amount` | Total XLM or USDC required to start the escrow |
| `token` | `XLM` or `USDC` |
| `deadline` | ISO-8601 datetime — pool auto-cancels if target not reached by this time |
| `freelancer_address` | Stellar public key of the contractor who will do the work |
| `milestones` | Array of milestone objects (title, description, amount) |

**Optional fields:**

| Field | Default | Description |
|-------|---------|-------------|
| `min_contribution` | `1 XLM` / `1 USDC` | Minimum per-wallet contribution |
| `max_contribution_per_wallet` | None | Cap per wallet to prevent concentration |
| `max_contributors` | `500` | Maximum number of unique contributor wallets |

### API example

```bash
curl -X POST https://api.stellar-crowd-fund.example/api/pools \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Open-source Stellar SDK for Python",
    "description": "Fund a production-ready Python SDK for the Soroban RPC.",
    "target_amount": "5000",
    "token": "XLM",
    "deadline": "2026-09-30T00:00:00Z",
    "freelancer_address": "GXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "milestones": [
      { "title": "Core RPC client", "description": "Async HTTP client for all RPC methods", "amount": "2000" },
      { "title": "Contract bindings", "description": "Type-safe bindings for common contract types", "amount": "2000" },
      { "title": "Documentation & examples", "description": "Full API docs and 10 worked examples", "amount": "1000" }
    ]
  }'
```

---

## Contributing to a Pool

Any Stellar wallet can contribute while the pool is in `FUNDING` state.

```bash
curl -X POST https://api.stellar-crowd-fund.example/api/pools/<pool_id>/contribute \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{ "amount": "100", "signed_xdr": "<wallet-signed XDR>" }'
```

The contribution is recorded in the contract and the contributor is added to the governance roster. Contributions are final — they cannot be withdrawn while the pool is in `FUNDING` state, only refunded if the pool is cancelled.

---

## Governance — Milestone Approval Voting

Once the pool reaches `FUNDED`, contributors vote on whether to release each milestone payment to the freelancer. Votes are weighted by contribution amount.

**Default quorum rules:**

| Decision | Required approval weight |
|----------|--------------------------|
| Approve milestone | ≥ 51% of total contributed |
| Cancel active pool | ≥ 67% of total contributed |

Votes are cast via signed Stellar transactions and recorded as Soroban contract data entries. The quorum thresholds are set at pool creation and cannot be changed after the first contribution.

---

## Refunds

If a pool is cancelled — either because the deadline was missed or because a governance cancellation vote passed — every contributor is automatically refunded:

1. Anyone calls `refund_all()` on the contract (permissionless).
2. The contract iterates the contributor list and issues one Stellar payment per wallet.
3. Each successful refund emits a `ContributorRefunded` event.
4. When the last refund clears, the pool moves to `REFUNDED` state.

Failed refunds (e.g. account not accepting the token) are queued in `pending_refunds` and can be retried by calling `retry_refund(<contributor_address>)`.

---

## Reputation Effects

| Outcome | Contractor effect | Contributor effect |
|---------|-------------------|-------------------|
| Pool completed successfully | +reputation, completion recorded | +small bonus for backing a completed project |
| Pool cancelled (deadline miss) | No change | No change (only refunded) |
| Pool cancelled (dispute) | Dispute recorded, may reduce score | No change |

---

## Related Topics

- [Escrow Lifecycle](escrow-lifecycle.md) — how the milestone phase works after a pool is funded
- [Milestones](milestones.md) — milestone submission, review, and approval
- [Disputes](disputes.md) — what happens when a contributor or contractor raises a dispute
- [Reputation System](reputation-system.md) — how scores are computed and stored on-chain
