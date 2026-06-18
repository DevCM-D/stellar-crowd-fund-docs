# Health API

The health endpoint provides a quick way to verify that the API server and its dependencies are reachable.

No authentication required.

---

## Health Check

```http
GET /api/health
```

### Response (200 — healthy)

```json
{
  "status": "ok",
  "timestamp": "2025-06-15T14:00:00Z",
  "uptime": 86400,
  "version": "1.2.0",
  "dependencies": {
    "database": "ok",
    "redis": "ok",
    "elasticsearch": "ok"
  }
}
```

### Response (503 — degraded)

```json
{
  "status": "degraded",
  "timestamp": "2025-06-15T14:00:00Z",
  "uptime": 86400,
  "version": "1.2.0",
  "dependencies": {
    "database": "ok",
    "redis": "ok",
    "elasticsearch": "unreachable"
  }
}
```

The server returns `503` when any critical dependency is unreachable. Elasticsearch is optional — if it's unreachable, the status is `degraded` (503) but the platform continues to function in fallback mode (Prisma-backed search and leaderboard).

If the database or Redis is unreachable, the response is also `503` and the platform will not accept requests correctly.

---

## Dependency Statuses

| Dependency | Critical | Notes |
|---|---|---|
| `database` | Yes | PostgreSQL via Prisma |
| `redis` | Yes | Cache, sessions, rate limiting, job queue |
| `elasticsearch` | No | Search and leaderboard fallback to Prisma if unreachable |

---

## Usage in Monitoring

Point your uptime monitor at `GET /api/health` and alert on any non-`200` response. For detailed metrics, use the structured logs emitted by the request logger.

---

## Related

- [Deployment → Production Checklist](../deployment/production-checklist.md) — monitoring recommendations
- [Architecture Overview](../architecture/overview.md) — dependency relationships
