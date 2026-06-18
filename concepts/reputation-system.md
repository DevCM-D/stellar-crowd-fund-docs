# Reputation System

The on-chain reputation system gives both clients and contractors a verifiable, portable track record that is impossible to fake, delete, or transfer to another address.

---

## The Core Idea

Every significant outcome in the platform emits a `ReputationEvent` to the Soroban contract. Events accumulate into a score. Both the raw events and the aggregate score are readable by anyone — no account login required, no API key needed.

Your reputation is tied to your Stellar address. It travels with you regardless of which frontend, tenant, or platform you use, as long as it supports reading Soroban contract state.

---

## What Generates Reputation Events

| Event type | Who it affects | Positive or negative |
|---|---|---|
| Milestone approved | Freelancer | Positive (`+score`) |
| Milestone approved | Client | Positive (for being a payer who follows through) |
| Escrow completed | Both parties | Positive |
| Dispute raised against you | The accused party | Slight negative |
| Dispute resolved in your favour | The winning party | Positive |
| Dispute resolved against you | The losing party | Negative |

The exact scoring weights are defined in the Soroban contract's `reputation.rs` module and are visible to anyone reviewing the source code.

---

## ReputationRecord Fields

The aggregated reputation record for an address contains:

| Field | Type | Description |
|---|---|---|
| `address` | string | Stellar address this record belongs to |
| `totalScore` | integer | Aggregate score (higher is better) |
| `completedEscrows` | integer | Number of escrows completed as either party |
| `disputedEscrows` | integer | Number of escrows that reached a dispute state |
| `disputesWon` | integer | Number of disputes resolved in this address's favour |
| `totalVolume` | string | Cumulative funds transacted (base units) |
| `lastUpdated` | datetime | Timestamp of the most recent event |

---

## ReputationEvent Fields

Each individual event that contributes to the score:

| Field | Type | Description |
|---|---|---|
| `address` | string | The address this event affects |
| `eventType` | enum | Type of event (see below) |
| `escrowId` | integer | The escrow this event relates to |
| `disputeId` | integer | The dispute this event relates to (if applicable) |
| `scoreDelta` | integer | How much this event changed the score |
| `tenantId` | string | Which tenant's escrow generated this event |
| `createdAt` | datetime | When the event was recorded |

### Event Types

| `eventType` | Trigger |
|---|---|
| `ESCROW_COMPLETED_AS_CLIENT` | Client completes an escrow |
| `ESCROW_COMPLETED_AS_FREELANCER` | Freelancer completes an escrow |
| `MILESTONE_APPROVED` | A milestone was approved |
| `DISPUTE_WON` | Dispute resolved in this address's favour |
| `DISPUTE_LOST` | Dispute resolved against this address |
| `DISPUTE_RAISED` | This address raised a dispute |

---

## Leaderboard

The leaderboard ranks addresses by `totalScore` within a tenant. It is served from Elasticsearch when available (fast, full-text capable) and falls back to a Prisma database query when Elasticsearch is unavailable.

```
GET /api/reputation/leaderboard?page=1&limit=20
```

The leaderboard is capped at 50 results per page to bound the query cost — leaderboard queries sort and score the full tenant table, so large pages are disproportionately expensive.

---

## Address Lookup

Any address can be queried directly:

```
GET /api/reputation/:address
```

If the address has no recorded events, the API returns a zero-score record rather than 404 — every address starts with a clean slate.

---

## Search

Addresses can be searched by prefix or full address:

```
GET /api/reputation/search?q=GABCD&limit=10
```

Useful for autocomplete when adding a freelancer or client to a new escrow.

---

## Why On-Chain?

Storing reputation in a database (as traditional platforms do) means:
- The platform owner can alter or delete reputation data
- Data is lost if the platform shuts down
- Records cannot be used by other applications without the platform's permission

Storing reputation in the Soroban contract means:
- No one — not even the contract deployer — can alter past events
- The data persists as long as the Stellar network exists
- Any application that can read Soroban state can use the reputation system
- Users can prove their reputation to any counterparty by pointing to a public contract address

The off-chain database (`ReputationRecord` and `ReputationEvent` tables in PostgreSQL) is an indexed mirror of on-chain state — used for fast queries, but ultimately derived from the contract and rebuildable from it.

---

## Related

- [Escrow Lifecycle](escrow-lifecycle.md) — when reputation events are emitted
- [Disputes](disputes.md) — how dispute outcomes affect reputation
- [API Reference → Reputation](../api-reference/reputation.md) — REST endpoints
