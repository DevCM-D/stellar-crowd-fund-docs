# Search API

Full-text search across escrows. Backed by Elasticsearch when available; falls back to a PostgreSQL full-text query.

Base path: `/api/search`

Requires `Authorization: Bearer <token>`.

---

## Search Escrows

```http
GET /api/search/escrows?q=frontend+redesign&limit=20&from=0
```

### Query Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `q` | string | — | Search query (max 200 characters) |
| `limit` | integer | `20` | Results per page (max 100) |
| `from` | integer | `0` | Offset for pagination |

### Input Sanitization

The `q` parameter is sanitized before it reaches Elasticsearch or the database:
- Truncated to 200 characters
- Control characters (ASCII 0x00–0x1F and 0x7F) stripped
- Leading and trailing whitespace trimmed

### Response (200)

```json
{
  "data": [
    {
      "id": "clxyz123",
      "clientAddress": "GABCD...",
      "freelancerAddress": "GXYZ...",
      "status": "Active",
      "totalAmount": "2000000000",
      "createdAt": "2025-03-01T10:00:00Z",
      "_score": 1.42
    }
  ],
  "total": 3,
  "from": 0,
  "limit": 20
}
```

The `_score` field is the Elasticsearch relevance score. It is only present when the Elasticsearch backend is active.

---

## Searchable Fields

When Elasticsearch is configured, the following escrow fields are indexed:

| Field | Indexed as |
|---|---|
| `briefHash` | keyword (exact match on CID) |
| `clientAddress` | keyword |
| `freelancerAddress` | keyword |
| `status` | keyword |
| Milestone `title` | text (full-text search) |
| Milestone `descriptionHash` | keyword |

The title field supports partial matches and fuzzy search (e.g. searching `"frontend"` matches escrows with milestones titled `"Frontend implementation"`).

---

## Fallback Behavior

When Elasticsearch is unavailable, the API falls back to a Prisma `fullTextSearch` query on the escrow and milestone tables. Relevance scoring and fuzzy matching are not available in fallback mode — results are filtered but not ranked by relevance.

The fallback is silent: the response format is the same; `_score` is omitted from results.

---

## Rate Limiting

Search is included in the standard per-user rate limit. There is no separate search-specific rate limit.

---

## Logging

Every search query is logged with structured fields:

```json
{
  "type": "search_query",
  "q": "frontend redesign",
  "tenantSlug": "acme",
  "requestId": "abc-123",
  "page": 1,
  "limit": 20
}
```

This allows audit and debugging of search usage without leaking PII from the request body.

---

## Related

- [Reputation Search](reputation.md) — searching reputation records by address
- [Escrows API](escrows.md) — fetching a specific escrow by ID
- [Pagination Reference](pagination.md) — offset vs cursor pagination
