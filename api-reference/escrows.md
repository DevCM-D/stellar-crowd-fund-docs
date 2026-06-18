# Escrows API

Endpoints for creating, listing, and managing escrows and their milestones.

Base path: `/api/escrows`

All endpoints require a valid `Authorization: Bearer <token>` header.

---

## List Escrows

```http
GET /api/escrows
```

Returns a paginated list of escrows for the authenticated tenant.

### Query Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `status` | string (comma-separated) | — | Filter by status: `Active`, `Completed`, `Disputed`, `Cancelled` |
| `clientAddress` | string | — | Filter to escrows where this Stellar address is the client |
| `freelancerAddress` | string | — | Filter to escrows where this Stellar address is the freelancer |
| `page` | integer | `1` | Page number (1-based) |
| `limit` | integer | `20` | Results per page (max 100) |
| `sortBy` | string | `createdAt` | Field to sort by |
| `sortOrder` | string | `desc` | `asc` or `desc` |

### Response (200)

```json
{
  "data": [
    {
      "id": "clxyz123",
      "onChainId": "12345",
      "clientAddress": "GABCD...",
      "freelancerAddress": "GXYZ...",
      "tokenAddress": "USDC_CONTRACT_ID",
      "totalAmount": "2000000000",
      "remainingBalance": "1000000000",
      "status": "Active",
      "briefHash": "QmIPFS...",
      "deadline": "2025-12-31T00:00:00Z",
      "createdAt": "2025-03-01T10:00:00Z",
      "_count": { "milestones": 4 }
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 42,
    "totalPages": 3
  }
}
```

---

## Get Escrow

```http
GET /api/escrows/:id
```

Returns a single escrow with its milestones included.

### Response (200)

```json
{
  "id": "clxyz123",
  "onChainId": "12345",
  "clientAddress": "GABCD...",
  "freelancerAddress": "GXYZ...",
  "tokenAddress": "USDC_CONTRACT_ID",
  "totalAmount": "2000000000",
  "remainingBalance": "1000000000",
  "status": "Active",
  "briefHash": "QmIPFS...",
  "deadline": "2025-12-31T00:00:00Z",
  "createdAt": "2025-03-01T10:00:00Z",
  "milestones": [
    {
      "id": "clmil1",
      "milestoneIndex": 0,
      "title": "Initial design mockups",
      "amount": "500000000",
      "descriptionHash": "QmABC...",
      "status": "Approved",
      "submissionHash": "QmDEF...",
      "submittedAt": "2025-04-01T12:00:00Z",
      "resolvedAt": "2025-04-03T09:00:00Z"
    },
    {
      "id": "clmil2",
      "milestoneIndex": 1,
      "title": "Frontend implementation",
      "amount": "1500000000",
      "descriptionHash": "QmGHI...",
      "status": "Submitted",
      "submissionHash": "QmJKL...",
      "submittedAt": "2025-05-15T14:00:00Z",
      "resolvedAt": null
    }
  ]
}
```

---

## Create Escrow

```http
POST /api/escrows
Content-Type: application/json
```

Creates a new escrow. Calls `create_escrow` on the Soroban contract to lock funds.

### Request Body

```json
{
  "freelancerAddress": "GXYZ...",
  "tokenAddress": "USDC_CONTRACT_ID",
  "totalAmount": "2000000000",
  "briefHash": "QmIPFS...",
  "deadline": "2025-12-31T00:00:00Z",
  "milestones": [
    {
      "title": "Initial design mockups",
      "amount": "500000000",
      "descriptionHash": "QmABC..."
    },
    {
      "title": "Frontend implementation",
      "amount": "1500000000",
      "descriptionHash": "QmGHI..."
    }
  ]
}
```

Constraints:
- Milestone amounts must sum to `totalAmount`
- At least one milestone required
- `tokenAddress` must be a valid Stellar contract address
- Client address is taken from the authenticated user's JWT

### Response (201)

Returns the created escrow object (same shape as GET).

---

## Submit Milestone

```http
POST /api/escrows/:id/milestones/:milestoneIndex/submit
Content-Type: application/json
```

Freelancer marks a milestone as submitted for review.

### Request Body

```json
{
  "submissionHash": "QmIPFShashOfDeliverable..."
}
```

### Response (200)

Returns the updated milestone object.

### Errors

| Status | Meaning |
|---|---|
| `400` | Milestone not in Pending or Rejected state |
| `403` | Caller is not the escrow's freelancer |
| `404` | Escrow or milestone not found |

---

## Approve Milestone

```http
POST /api/escrows/:id/milestones/:milestoneIndex/approve
```

Client approves the milestone submission. Triggers on-chain fund release.

No request body required.

### Response (200)

```json
{
  "milestone": { ...updated milestone... },
  "escrow": { ...updated escrow — status Completed if last milestone... },
  "amountReleased": "500000000"
}
```

### Errors

| Status | Meaning |
|---|---|
| `400` | Milestone not in Submitted state |
| `403` | Caller is not the escrow's client |
| `404` | Escrow or milestone not found |

---

## Reject Milestone

```http
POST /api/escrows/:id/milestones/:milestoneIndex/reject
Content-Type: application/json
```

Client rejects the milestone. Freelancer can revise and resubmit.

### Request Body

```json
{
  "reason": "Does not match the acceptance criteria in the description"
}
```

### Response (200)

Returns the updated milestone with `status: "Rejected"`.

---

## Cancel Escrow

```http
POST /api/escrows/:id/cancel
```

Initiates or confirms escrow cancellation. Both parties must call this endpoint.

- First call records the calling party's consent
- Second call (by the other party) completes cancellation and refunds remaining balance to the client

### Response (200)

```json
{
  "status": "consent_recorded",
  "message": "Waiting for the other party to confirm cancellation"
}
```

or, when the second party confirms:

```json
{
  "status": "cancelled",
  "refundedAmount": "1500000000"
}
```

---

## List Milestones

```http
GET /api/escrows/:id/milestones
```

Returns all milestones for an escrow, in order by `milestoneIndex`.

---

## Pagination

All list endpoints support offset-based pagination via `page` and `limit`. Cursor-based pagination is available on milestone lists using `cursor` and `limit` parameters — see [Pagination Reference](pagination.md) for details.

---

## Related

- [Milestones](../concepts/milestones.md) — milestone states and sequencing
- [Escrow Lifecycle](../concepts/escrow-lifecycle.md) — full state machine
- [Disputes API](disputes.md) — raising a dispute on an escrow
