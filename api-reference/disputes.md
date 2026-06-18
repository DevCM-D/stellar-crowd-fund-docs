# Disputes API

Endpoints for raising disputes, uploading evidence, and listing dispute records.

Base path: `/api/disputes`

All endpoints require `Authorization: Bearer <token>`.

---

## Raise a Dispute

```http
POST /api/disputes/:escrowId
Content-Type: application/json
```

Either the client or the freelancer can raise a dispute. Transitions the escrow to `Disputed` state on-chain.

### Request Body

```json
{
  "reason": "Deliverable does not match the acceptance criteria defined in milestone 1"
}
```

### Response (201)

```json
{
  "id": 42,
  "escrowId": "clxyz123",
  "raisedByAddress": "GXYZ...",
  "raisedAt": "2025-06-15T14:00:00Z",
  "resolvedAt": null,
  "evidence": [],
  "_count": { "evidence": 0, "appeals": 0 }
}
```

### Errors

| Status | Meaning |
|---|---|
| `400` | Escrow not in Active state |
| `403` | Caller is not the client or freelancer on this escrow |
| `404` | Escrow not found |
| `409` | Dispute already exists for this escrow |

---

## Upload Evidence

```http
POST /api/disputes/:escrowId/evidence
Content-Type: multipart/form-data
```

Upload an evidence file for an active dispute. Either party can upload evidence.

### Form Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | binary | Yes | The evidence file (image, PDF, video, archive, etc.) |
| `evidenceType` | string | Yes | `screenshot`, `document`, `video`, `code`, `other` |
| `description` | string | No | Brief explanation of what this evidence demonstrates |

### Processing

1. File is scanned for malware
2. Thumbnail generated for images and video (stored on IPFS)
3. File uploaded to IPFS; CID recorded
4. Evidence record created with scan status `pending` → updated to `clean` or `flagged`

### Response (201)

```json
{
  "id": 7,
  "disputeId": 42,
  "submittedBy": "GXYZ...",
  "submittedAt": "2025-06-15T14:30:00Z",
  "filename": "milestone-2-output.png",
  "evidenceType": "screenshot",
  "ipfsCid": "QmABC...",
  "thumbnailCid": "QmDEF...",
  "scanStatus": "pending"
}
```

### Errors

| Status | Meaning |
|---|---|
| `400` | Missing required fields or invalid `evidenceType` |
| `403` | Caller is not a party to this escrow |
| `404` | No active dispute for this escrow |
| `415` | Unsupported file type |

---

## List Disputes

```http
GET /api/disputes
```

Returns a paginated list of disputes for the authenticated tenant.

### Query Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `status` | string | — | `resolved` or `unresolved` |
| `raisedBy` | string | — | Stellar address of the party who raised the dispute |
| `dateFrom` | ISO date | — | Disputes raised on or after this date |
| `dateTo` | ISO date | — | Disputes raised on or before this date |
| `sortBy` | string | `raisedAt` | `raisedAt`, `resolvedAt`, or `id` |
| `sortOrder` | string | `desc` | `asc` or `desc` |
| `page` | integer | `1` | Page number |
| `limit` | integer | `20` | Results per page (max 50) |

### Response (200)

```json
{
  "data": [
    {
      "id": 42,
      "escrowId": "clxyz123",
      "raisedByAddress": "GXYZ...",
      "raisedAt": "2025-06-15T14:00:00Z",
      "resolvedAt": null,
      "resolutionType": null,
      "_count": { "evidence": 3, "appeals": 0 }
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 5, "totalPages": 1 }
}
```

---

## Get Dispute

```http
GET /api/disputes/:id
```

Returns a single dispute with full detail including all evidence records.

### Response (200)

```json
{
  "id": 42,
  "escrowId": "clxyz123",
  "raisedByAddress": "GXYZ...",
  "raisedAt": "2025-06-15T14:00:00Z",
  "resolvedAt": null,
  "clientAmount": null,
  "freelancerAmount": null,
  "resolvedBy": null,
  "resolution": null,
  "resolutionType": null,
  "autoResolved": false,
  "evidence": [
    {
      "id": 7,
      "evidenceType": "document",
      "submittedBy": "GXYZ...",
      "submittedAt": "2025-06-15T14:30:00Z",
      "filename": "acceptance-criteria.pdf",
      "ipfsCid": "QmABC...",
      "thumbnailCid": null,
      "scanStatus": "clean"
    }
  ],
  "_count": { "evidence": 1, "appeals": 0 }
}
```

---

## File an Appeal

```http
POST /api/disputes/:escrowId/appeals
Content-Type: application/json
```

File an appeal after a dispute has been resolved. Only the party who lost the dispute (received less than half the funds) may appeal.

### Request Body

```json
{
  "reason": "The arbiter did not review evidence submitted on June 17th — see file QmABC..."
}
```

### Response (201)

```json
{
  "id": 3,
  "disputeId": 42,
  "filedBy": "GXYZ...",
  "filedAt": "2025-06-20T10:00:00Z",
  "reason": "..."
}
```

---

## Admin: Resolve Dispute

```http
PATCH /api/admin/disputes/:id/resolve
Authorization: Bearer <admin-token>
Content-Type: application/json
```

Resolves an active dispute. Arbiter-only endpoint. Calls `resolve_dispute` on the Soroban contract.

### Request Body

```json
{
  "clientAmount": "1500000000",
  "freelancerAmount": "500000000",
  "notes": "Client provided clear evidence that milestone 2 deliverable did not meet criteria"
}
```

Constraint: `clientAmount + freelancerAmount` must equal the escrow's remaining balance.

### Response (200)

```json
{
  "dispute": {
    "id": 42,
    "resolvedAt": "2025-06-20T11:00:00Z",
    "resolutionType": "MANUAL",
    "clientAmount": "1500000000",
    "freelancerAmount": "500000000"
  },
  "escrow": {
    "status": "Completed"
  }
}
```

---

## Related

- [Disputes Concept](../concepts/disputes.md) — how disputes work end-to-end
- [Reputation System](../concepts/reputation-system.md) — how dispute outcomes affect scores
- [Escrows API](escrows.md) — the escrow context disputes live within
