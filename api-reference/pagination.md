# Pagination Reference

The API supports two pagination strategies: offset-based and cursor-based. Both are available on most list endpoints; cursor-based is preferred for large or frequently-updated datasets.

---

## Offset Pagination

The default strategy. Use `page` and `limit` query parameters.

```http
GET /api/escrows?page=2&limit=20
```

### Parameters

| Parameter | Default | Max | Description |
|---|---|---|---|
| `page` | `1` | — | 1-based page number |
| `limit` | `20` | `100` | Items per page |

### Response shape

```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 143,
    "totalPages": 8
  }
}
```

### When to use

- Low to medium result sets (under ~10,000 records)
- When you need to jump to a specific page number
- When you need to display total page count

### Limitation

Offset pagination can return duplicate or skipped items if records are inserted or deleted between requests. For stable, consistent pagination on large datasets, use cursor pagination.

---

## Cursor Pagination

An alternative to offset pagination for endpoints that support it. Use `cursor` and `limit` parameters instead of `page`.

```http
GET /api/escrows/:id/milestones?limit=10
```

To fetch the next page, pass the `nextCursor` from the previous response:

```http
GET /api/escrows/:id/milestones?cursor=clmil42&limit=10
```

### Parameters

| Parameter | Default | Description |
|---|---|---|
| `cursor` | — | Opaque cursor value from the previous response's `nextCursor` |
| `limit` | `20` | Items per page (max varies by endpoint — see endpoint docs) |

### Response shape

```json
{
  "data": [...],
  "nextCursor": "clmil52",
  "hasNextPage": true
}
```

When there are no more pages, `nextCursor` is `null` and `hasNextPage` is `false`:

```json
{
  "data": [...],
  "nextCursor": null,
  "hasNextPage": false
}
```

### When to use

- Infinite scroll or "load more" UI patterns
- Large, frequently-updated datasets
- When consistent pagination under concurrent writes is required

---

## Choosing a Strategy

| Scenario | Use |
|---|---|
| Displaying page X of Y | Offset |
| Infinite scroll / load more | Cursor |
| Real-time feed that updates frequently | Cursor |
| Admin table with jump-to-page | Offset |
| High-volume datasets (10k+ records) | Cursor |

---

## Endpoint Support

| Endpoint | Offset | Cursor |
|---|---|---|
| `GET /api/escrows` | Yes | No |
| `GET /api/escrows/:id/milestones` | Yes | Yes |
| `GET /api/disputes` | Yes | No |
| `GET /api/reputation/leaderboard` | Yes | No |
| `GET /api/reputation/:address/events` | Yes | No |
| `GET /api/webhooks/deliveries` | Yes | No |

---

## Related

- [Escrows API](escrows.md) — offset pagination on escrow list
- [Architecture → Database Schema](../architecture/database-schema.md) — indexes that back efficient pagination
