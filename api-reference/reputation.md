# Reputation API

Endpoints for querying on-chain reputation records, events, and the leaderboard.

Base path: `/api/reputation`

These endpoints are read-only. No write endpoints exist — reputation records are written by on-chain contract events and mirrored to the database automatically.

Authentication is required for leaderboard and search endpoints. Address lookup (`/api/reputation/:address`) is public.

---

## Get Reputation for an Address

```http
GET /api/reputation/:address
```

Returns the aggregated reputation record for a Stellar address.

No authentication required.

### Path Parameters

| Parameter | Description |
|---|---|
| `address` | A valid Stellar public key (G...) |

### Response (200)

```json
{
  "address": "GABCDEFG...",
  "tenantId": "cltn123",
  "totalScore": 1450,
  "completedEscrows": 12,
  "disputedEscrows": 1,
  "disputesWon": 1,
  "totalVolume": "24000000000",
  "lastUpdated": "2025-06-15T14:32:00Z"
}
```

If the address has no reputation history, a zero-score record is returned (not 404):

```json
{
  "address": "GZERO...",
  "totalScore": 0,
  "completedEscrows": 0,
  "disputedEscrows": 0,
  "disputesWon": 0,
  "totalVolume": "0",
  "lastUpdated": null
}
```

---

## List Reputation Events for an Address

```http
GET /api/reputation/:address/events
```

Returns the individual events that make up an address's reputation score, newest first.

### Query Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `page` | integer | `1` | Page number |
| `limit` | integer | `20` | Results per page (max 100) |

### Response (200)

```json
{
  "address": "GABCDEFG...",
  "data": [
    {
      "id": 101,
      "eventType": "MILESTONE_APPROVED",
      "escrowId": "clxyz123",
      "disputeId": null,
      "scoreDelta": 10,
      "createdAt": "2025-06-15T14:32:00Z"
    },
    {
      "id": 98,
      "eventType": "ESCROW_COMPLETED_AS_FREELANCER",
      "escrowId": "clxyz120",
      "disputeId": null,
      "scoreDelta": 50,
      "createdAt": "2025-06-10T09:15:00Z"
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 34, "totalPages": 2 }
}
```

---

## Leaderboard

```http
GET /api/reputation/leaderboard
```

Returns addresses ranked by `totalScore` for the current tenant, highest first.

Requires authentication.

### Query Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `page` | integer | `1` | Page number |
| `limit` | integer | `20` | Results per page (max 50) |

The `limit` is capped at 50. Leaderboard queries sort the full tenant table by score, which is expensive; the cap bounds the cost per request.

### Response (200)

```json
{
  "data": [
    {
      "rank": 1,
      "address": "GABCDEFG...",
      "totalScore": 2850,
      "completedEscrows": 28,
      "disputesWon": 2
    },
    {
      "rank": 2,
      "address": "GXYZ...",
      "totalScore": 1920,
      "completedEscrows": 19,
      "disputesWon": 0
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 150, "totalPages": 8 }
}
```

---

## Search Reputation Records

```http
GET /api/reputation/search?q=GABCD&limit=10
```

Search for reputation records by Stellar address prefix or full address. Backed by Elasticsearch when available; falls back to a PostgreSQL `LIKE` query.

Requires authentication.

### Query Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `q` | string | — | Address prefix or full address (max 200 chars) |
| `limit` | integer | `10` | Max results (max 100) |
| `from` | integer | `0` | Offset for pagination |

### Response (200)

```json
{
  "data": [
    {
      "address": "GABCDEFG...",
      "totalScore": 2850,
      "completedEscrows": 28
    }
  ],
  "total": 1,
  "from": 0,
  "limit": 10
}
```

---

## Score Weights

Score deltas are determined by the Soroban contract. Representative values:

| Event | Score delta |
|---|---|
| `ESCROW_COMPLETED_AS_CLIENT` | +50 |
| `ESCROW_COMPLETED_AS_FREELANCER` | +50 |
| `MILESTONE_APPROVED` | +10 |
| `DISPUTE_WON` | +30 |
| `DISPUTE_RAISED` | −5 |
| `DISPUTE_LOST` | −20 |

These values are defined in the contract source and may be adjusted via contract upgrade. Read the contract to get the canonical current values.

---

## Related

- [Reputation System Concept](../concepts/reputation-system.md) — architecture and motivation
- [Disputes API](disputes.md) — how dispute resolution creates reputation events
- [Leaderboard rate limiting](../security/rate-limiting.md) — the leaderboard cap and why
