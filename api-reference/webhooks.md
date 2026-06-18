# Webhooks API

Endpoints for managing webhook subscriptions and inspecting delivery history.

Base path: `/api/webhooks`

All endpoints require `Authorization: Bearer <token>`.

---

## Subscribe

```http
POST /api/webhooks/subscribe
Content-Type: application/json
```

Create a new webhook subscription for the authenticated user.

**Rate limit:** 10 subscription requests per 10 minutes per user address.

### Request Body

```json
{
  "url": "https://your-server.example.com/webhook",
  "eventTypes": ["milestone.approved", "dispute.raised", "escrow.completed"]
}
```

Constraints:
- `url` must use HTTPS
- `eventTypes` must be a non-empty array with at most 20 entries

### Response (201)

```json
{
  "id": "sub-clxyz123",
  "url": "https://your-server.example.com/webhook",
  "eventTypes": ["milestone.approved", "dispute.raised", "escrow.completed"],
  "secret": "whsec_abc123...",
  "active": true,
  "createdAt": "2025-06-15T14:00:00Z"
}
```

**Save the `secret` immediately.** It is shown once and cannot be retrieved again. You'll use it to verify incoming webhook signatures via `X-Webhook-Signature`.

### Errors

| Status | Meaning |
|---|---|
| `400` | Invalid URL (not HTTPS) or invalid eventTypes |
| `429` | Rate limit exceeded |

---

## List Subscriptions

```http
GET /api/webhooks/subscriptions
```

Returns all active webhook subscriptions for the authenticated user.

### Response (200)

```json
{
  "data": [
    {
      "id": "sub-clxyz123",
      "url": "https://your-server.example.com/webhook",
      "eventTypes": ["milestone.approved"],
      "active": true,
      "createdAt": "2025-06-15T14:00:00Z"
    }
  ]
}
```

Note: `secret` is never returned after the initial subscribe response.

---

## Delete Subscription

```http
DELETE /api/webhooks/subscriptions/:id
```

Deactivates a webhook subscription. In-flight deliveries will still complete, but no new events will be queued.

### Response (204)

No body.

---

## List Deliveries

```http
GET /api/webhooks/deliveries?subscriptionId=<id>&limit=30
```

Returns recent delivery attempts for a subscription, newest first.

### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `subscriptionId` | string | No | Filter by subscription |
| `status` | string | No | `pending`, `delivered`, `failed` |
| `limit` | integer | No | Results per page (default 30, max 100) |

### Response (200)

```json
{
  "data": [
    {
      "id": "del-abc123",
      "subscriptionId": "sub-clxyz123",
      "eventType": "milestone.approved",
      "status": "delivered",
      "attemptCount": 1,
      "lastStatusCode": 200,
      "createdAt": "2025-06-15T14:32:00Z",
      "lastAttemptAt": "2025-06-15T14:32:05Z"
    },
    {
      "id": "del-abc124",
      "subscriptionId": "sub-clxyz123",
      "eventType": "dispute.raised",
      "status": "failed",
      "attemptCount": 5,
      "lastStatusCode": 503,
      "lastResponse": "Service Unavailable",
      "createdAt": "2025-06-15T15:00:00Z",
      "lastAttemptAt": "2025-06-15T15:02:20Z"
    }
  ],
  "pagination": { "page": 1, "limit": 30, "total": 84, "totalPages": 3 }
}
```

---

## Webhook Payload Format

Every delivery sends a `POST` to your endpoint with this body:

```json
{
  "id": "del-abc123",
  "subscriptionId": "sub-clxyz123",
  "eventType": "milestone.approved",
  "timestamp": "2025-06-15T14:32:00Z",
  "data": {
    "escrowId": "clxyz123",
    "milestoneIndex": 1,
    "clientAddress": "GABCD...",
    "freelancerAddress": "GXYZ...",
    "amountReleased": "500000000"
  }
}
```

---

## Signature Verification

Each delivery includes a signature header:

```
X-Webhook-Signature: sha256=<hmac-hex>
```

Verify it before processing:

```javascript
import { createHmac, timingSafeEqual } from 'crypto';

function verify(rawBody, secret, signatureHeader) {
  const expected = createHmac('sha256', secret)
    .update(rawBody)
    .digest('hex');
  const received = (signatureHeader ?? '').replace('sha256=', '');
  try {
    return timingSafeEqual(
      Buffer.from(expected, 'hex'),
      Buffer.from(received, 'hex'),
    );
  } catch {
    return false;
  }
}
```

Always use `timingSafeEqual` to prevent timing attacks. Never use `===` for signature comparison.

---

## Retry and Backoff

| Attempt | Delay |
|---|---|
| 1st | 5 seconds |
| 2nd | 10 seconds |
| 3rd | 20 seconds |
| 4th | 40 seconds |
| 5th | 80 seconds |

After 5 failures the delivery is marked `failed`. Failed deliveries are retained for 100 records (configurable).

Return a `2xx` status code quickly — if your handler takes longer than the timeout window, the delivery is treated as failed and retried.

---

## Related

- [Webhooks Concept](../concepts/webhooks.md) — event types, payload schemas, and security guidance
- [Security → Rate Limiting](../security/rate-limiting.md) — subscription rate limiter details
