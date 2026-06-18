# Rate Limiting

Rate limiting prevents any single user or address from overwhelming the API. The platform uses a sliding-window algorithm backed by Redis.

---

## Algorithm

**Sliding window** — more accurate than fixed-window counting. For a 10-minute window:

- A new request at `T=9:00` can fill the window quota
- Those requests remain in the window until `T=10:00` (10 minutes later), not until `T=9:10`
- The window slides forward with time; old requests expire naturally

Fixed-window algorithms allow a burst at the end of one window and the start of the next. Sliding-window prevents this by continuously accounting for the full prior window.

---

## Rate Limit Tiers

Each user tier has its own quota. Tier assignment is set by an admin:

| Tier | Requests per window | Window |
|---|---|---|
| Default | 100 | 1 minute |
| Elevated | 300 | 1 minute |
| Admin | 1000 | 1 minute |

Tier limits are configurable at runtime via the admin API (without restarting the server). Changes are audit-logged.

---

## Endpoint-Specific Limits

Some endpoints have stricter limits regardless of tier:

### Webhook Subscribe

```
10 requests per 10 minutes per user address
```

Subscription flooding could exhaust database and Redis storage. The tight limit prevents abuse without affecting legitimate use (users rarely create more than a few subscriptions).

Rate limit key: `webhook-subscribe:addr:<address>` (per authenticated address)

---

## Redis Storage

Rate limit counters live in Redis using sorted sets. Each request adds a timestamp to the set; expired timestamps are removed before checking the count. This pattern is safe under concurrent requests and does not require locking.

Key format:
```
sliding:<prefix>:<identifier>
```

Example:
```
sliding:user:GABCDEFG...    (per-user general rate limit)
sliding:webhook-subscribe:addr:GABCDEFG...
```

---

## Rate Limit Response

When a request exceeds the limit:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 42
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1718000060

{
  "error": "Rate limit exceeded — try again later",
  "retryAfter": 42
}
```

The `Retry-After` header tells the client how many seconds to wait before retrying.

---

## Admin: Update Rate Limits

Admins can adjust rate limits per tier at runtime:

```http
PATCH /api/admin/rate-limits
Authorization: Bearer <admin-token>
Content-Type: application/json

{
  "tier": "default",
  "limit": 200
}
```

Changes take effect immediately and are logged:

```json
{
  "type": "admin_action",
  "action": "UPDATE_RATE_LIMIT",
  "tier": "default",
  "previous": 100,
  "updated": 200,
  "performedBy": "GABCDEFG...",
  "requestId": "abc-123"
}
```

---

## Key Generation

The key used to identify a request for rate limiting:

- **Authenticated requests**: keyed by the user's Stellar address — limits apply per user regardless of IP
- **Unauthenticated requests** (where applicable): keyed by IP address — less reliable behind proxies

For webhook subscribe, the key prefers the address over the IP:

```javascript
keyGenerator: (req) =>
  req.user?.address
    ? `webhook-subscribe:addr:${req.user.address}`
    : `webhook-subscribe:ip:${req.ip ?? 'unknown'}`
```

---

## Disabling Rate Limiting

Rate limiting is always on in production. In local development, you can raise the limits to reduce friction, but do not set them to zero or disable them entirely — the middleware is part of the request pipeline and removing it can cause unexpected behaviour in tests.

---

## Related

- [Security Overview](overview.md) — rate limiting in context
- [Webhooks Concept](../concepts/webhooks.md) — subscription-level rate limiting
- [Webhooks API](../api-reference/webhooks.md) — subscribe endpoint rate limit details
