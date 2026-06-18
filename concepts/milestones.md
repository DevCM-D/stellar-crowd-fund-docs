# Milestones

Milestones are the core mechanism that makes this escrow system trustworthy for both parties. Instead of releasing all funds at the end of a project, work is broken into discrete checkpoints — each with its own amount and verifiable deliverable.

---

## Why Milestones Matter

**For clients:** You never release more than one milestone's worth of funds at a time. If the freelancer fails to deliver on a milestone, you stop payment there — the remaining milestones stay locked. Your risk exposure is bounded.

**For contractors:** Once a milestone is approved on-chain, the funds transfer immediately and irrevocably. You do not have to chase payment. Approved work cannot be clawed back.

---

## Milestone Structure

Each milestone is defined at escrow creation and contains:

| Field | Type | Description |
|---|---|---|
| `milestoneIndex` | integer | Zero-based position in the milestone sequence |
| `title` | string | Short human-readable label |
| `amount` | string | Funds allocated to this milestone (base units) |
| `descriptionHash` | string | IPFS content hash of the milestone description |
| `status` | enum | Current state (see below) |
| `submittedAt` | datetime | When the freelancer marked it as submitted |
| `resolvedAt` | datetime | When the client approved or the dispute was resolved |

---

## Milestone States

```
Pending → Submitted → Approved
                   ↘
                   (dispute raised on escrow) → frozen until resolved
```

| State | Meaning |
|---|---|
| `Pending` | Work has not started or is in progress; no submission yet |
| `Submitted` | Freelancer has submitted the milestone for review |
| `Approved` | Client has approved; funds have been released for this milestone |
| `Rejected` | Client rejected the submission; freelancer can resubmit (not a dispute) |

Rejected milestones return to `Pending` — the freelancer can address the feedback and resubmit.

---

## Milestone Amounts

The amount for each milestone is set when the escrow is created and cannot be changed afterwards. The sum of all milestone amounts must equal the total escrow amount. The contract enforces this during `create_escrow`.

Amounts are stored as strings representing the asset in base units:
- USDC and most Stellar assets use 7 decimal places
- `1000000000` = 100 USDC (100 × 10^7)
- `500000000` = 50 USDC

---

## IPFS-Backed Descriptions and Deliverables

Both milestone descriptions (set at creation) and deliverable submissions use IPFS content hashes:

**Description hash** (`descriptionHash`): Set at creation. Points to a document describing what constitutes successful completion of this milestone — the acceptance criteria. Both parties agree to this before funds are locked.

**Submission hash** (`ipfs_hash` in `submit_milestone`): Set when the freelancer submits. Points to the actual deliverable — code, designs, documents, etc. Stored on-chain and cannot be altered after submission.

Because IPFS addresses are content-derived, altering the file would produce a different hash. The on-chain record is therefore tamper-evident.

---

## Milestone Sequencing

Milestones are indexed starting at 0 and are intended to be completed in order, but the contract does not technically enforce sequencing. The client and freelancer agree on the intended order in the milestone descriptions.

In practice, sequential completion is almost always the right approach:
- Each milestone builds on the previous one
- The client can verify progress incrementally
- The freelancer receives partial payment at each checkpoint

---

## Interaction Flow

```
Freelancer                    Client
    │                            │
    │── submit_milestone(0) ────►│
    │   (IPFS hash of work)      │
    │                            │── approves?
    │                            │     Yes: approve_milestone(0)
    │◄── funds released ─────────│         funds transfer
    │                            │
    │── submit_milestone(1) ────►│
    │                            │── approves?
    │                            │     No: reject
    │◄── feedback ───────────────│
    │                            │
    │── resubmit milestone(1) ──►│
    │                            │── approve_milestone(1)
    │◄── funds released ─────────│
    │                            │
    ...repeat for each milestone...
    │                            │
    │── submit_milestone(n) ────►│
    │                            │── approve_milestone(n)  ← last one
    │◄── final funds released ───│
    │                       escrow status → Completed
```

---

## Querying Milestones

Via the REST API:

```
GET /api/escrows/:id/milestones
```

Returns a paginated list of milestones for the given escrow.

```
GET /api/escrows/:id/milestones/:milestoneIndex
```

Returns a single milestone with full detail.

See [API Reference → Escrows](../api-reference/escrows.md) for full parameter documentation.

---

## Related

- [Escrow Lifecycle](escrow-lifecycle.md) — how escrow state connects to milestone completion
- [Disputes](disputes.md) — what happens when milestone approval breaks down
