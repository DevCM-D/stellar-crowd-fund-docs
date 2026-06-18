# Disputes

Disputes are the safety valve of the platform. When a client and contractor cannot agree on whether work meets the milestone criteria, either party can raise a dispute. A neutral arbiter reviews the evidence and resolves it on-chain.

---

## When Disputes Happen

A dispute can be raised at any point while an escrow is `Active`. Common triggers:

- Freelancer submits a milestone the client believes is incomplete or incorrect
- Client refuses to approve a milestone the freelancer believes meets the criteria
- External circumstances (missed deadline, changed requirements) that need arbitration
- One party disappears or becomes unresponsive

---

## Raising a Dispute

```
POST /api/disputes/:escrowId
```

Or directly via the contract:

```
raise_dispute(escrow_id: u64, reason: String)
```

Either the client or the freelancer can raise a dispute. The `reason` field is stored on-chain as a brief explanation.

After a dispute is raised:
- The escrow transitions to `Disputed`
- No further milestones can be submitted or approved
- Both parties can upload evidence
- The escrow stays frozen until an arbiter resolves it

---

## Submitting Evidence

Evidence is the core of the dispute process. Either party can upload files to support their case:

```
POST /api/disputes/:escrowId/evidence
Content-Type: multipart/form-data

file: <file>
evidenceType: "screenshot" | "document" | "video" | "code" | "other"
description: "Brief explanation of what this evidence shows"
```

Evidence files are uploaded to IPFS. The IPFS content hash is stored in the database and on-chain, making the evidence tamper-evident — the same file always produces the same hash; a different hash means a different file.

### Evidence processing

When a file is uploaded:

1. The file is scanned for malware (virus scan middleware)
2. Image and video files receive a thumbnail generated for preview
3. The file is uploaded to IPFS and a `CID` (content identifier) is returned
4. A Merkle root of all evidence for the dispute is computed
5. The evidence record is stored with: `ipfsCid`, `thumbnailCid`, `filename`, `evidenceType`, `scanStatus`, `submittedBy`, `submittedAt`

---

## Resolution

An authorised arbiter reviews the evidence and resolves the dispute:

```
PATCH /api/admin/disputes/:id/resolve
{
  "clientAmount": "1000000000",
  "freelancerAmount": "500000000",
  "notes": "Client provided evidence of incomplete delivery for milestone 2"
}
```

Or via the contract directly:

```
resolve_dispute(
  escrow_id: u64,
  client_amount: i128,
  freelancer_amount: i128
)
```

Rules enforced by the contract:
- `clientAmount + freelancerAmount` must equal the remaining locked balance
- Only the designated arbiter can call `resolve_dispute`
- Resolution can only happen when the escrow is in `Disputed` state

After resolution:
- Funds are split and transferred as specified
- A `ReputationEvent` is written for both parties reflecting the outcome
- The escrow transitions to `Completed`

---

## Dispute Data Structure

```json
{
  "id": 42,
  "escrowId": "12345",
  "raisedByAddress": "GABCD...",
  "raisedAt": "2025-06-01T14:00:00Z",
  "resolvedAt": null,
  "clientAmount": null,
  "freelancerAmount": null,
  "resolvedBy": null,
  "resolution": null,
  "resolutionType": null,
  "autoResolved": false,
  "evidence": [
    {
      "id": 1,
      "evidenceType": "document",
      "submittedBy": "GABCD...",
      "submittedAt": "2025-06-02T09:00:00Z",
      "filename": "milestone-2-acceptance-criteria.pdf",
      "ipfsCid": "QmABC...",
      "thumbnailCid": "QmDEF...",
      "scanStatus": "clean"
    }
  ],
  "_count": {
    "evidence": 1,
    "appeals": 0
  }
}
```

---

## Resolution Types

| `resolutionType` | Meaning |
|---|---|
| `MANUAL` | Arbiter reviewed evidence and made a decision |
| `AUTO` | System resolved automatically (e.g. deadline passed with no response) |
| `ESCALATED` | Escalated to a higher-level arbiter or DAO governance |

---

## Appeals

If a party believes the arbiter's decision was incorrect, they can file an appeal:

```
POST /api/disputes/:escrowId/appeals
{
  "reason": "The arbiter did not consider evidence submitted on June 3rd"
}
```

Appeals are reviewed by a senior arbiter or a governance body (implementation depends on tenant configuration). The appeals process is recorded in the `DisputeAppeal` table.

---

## Listing Disputes

```
GET /api/disputes?status=unresolved&sortBy=raisedAt&sortOrder=desc
```

Filter parameters:

| Parameter | Values | Description |
|---|---|---|
| `status` | `resolved`, `unresolved` | Filter by resolution state |
| `raisedBy` | Stellar address | Filter by who raised the dispute |
| `dateFrom` | ISO date | Disputes raised on or after this date |
| `dateTo` | ISO date | Disputes raised on or before this date |
| `sortBy` | `raisedAt`, `resolvedAt`, `id` | Sort field |
| `sortOrder` | `asc`, `desc` | Sort direction |
| `page` | integer | Page number |
| `limit` | integer (max 50) | Results per page |

---

## Impact on Reputation

Dispute outcomes affect both parties' on-chain reputation:

| Outcome | Effect on winner | Effect on loser |
|---|---|---|
| Full funds to freelancer | Freelancer: `DISPUTE_WON` (+score) | Client: `DISPUTE_LOST` (−score) |
| Full funds to client | Client: `DISPUTE_WON` (+score) | Freelancer: `DISPUTE_LOST` (−score) |
| Split resolution | Both receive `DISPUTE_RAISED` (slight negative) | Neither wins or loses fully |

---

## Related

- [Escrow Lifecycle](escrow-lifecycle.md) — how disputes fit into the overall state machine
- [Reputation System](reputation-system.md) — how dispute outcomes affect scores
- [API Reference → Disputes](../api-reference/disputes.md) — full endpoint documentation
