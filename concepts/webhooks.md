# Webhooks

Webhooks let external services receive real-time notifications when events happen on the platform — escrow status changes, milestone approvals, dispute events, and more — without polling the API.

---

## How Webhooks Work

When an event occurs, the platform queues a delivery job that sends an HTTP POST request to the subscribed URL. The request body is a JSON payload describing the event. The receiving server processes the event and returns a 2xx response.

If the delivery fails (network error, timeout, non-2xx response), the platform retries with exponential backoff. Failed deliveries are logged and inspectable via the API.

---

## Subscribing

```http
POST /api/webhooks/subscribe
Authorization: Bearer <token>
Content-Type: application/json

{
  "url": "https://your-server.example.com/webhook",
  "eventTypes": ["milestone.approved", "dispute.raised", "escrow.completed"]
}
```

Constraints enforced at subscription time:
- `url` must use HTTPS (not HTTP)
- `eventTypes` must be a non-empty array with at most 20 entries
- Rate limit: 10 subscription requests per 10 minutes per user address

The API returns a subscription object with an `id` and a `secret`. Save the secret — you'll use it to verify incoming webhook signatures.

---

## Event Types

| Event type | When it fires |
|---|---|
| `escrow.created` | A new escrow is created |
| `escrow.completed` | All milestones approved; escrow closed |
| `escrow.cancelled` | Escrow cancelled by mutual consent |
| `milestone.submitted` | Freelancer submits a milestone |
| `milestone.approved` | Client approves a milestone; funds released |
| `milestone.rejected` | Client rejects a milestone submission |
| `dispute.raised` | A dispute is raised on an escrow |
| `dispute.evidence_uploaded` | New evidence added to a dispute |
| `dispute.resolved` | Arbiter resolves the dispute |
| `reputation.updated` | A reputation event is written for an address |

---

## Payload Format

All webhook deliveries share the same outer envelope:

```json
{
  "id": "delivery-uuid-here",
  "subscriptionId": "sub-uuid-here",
  "eventType": "milestone.approved",
  "timestamp": "2025-06-15T14:32:00Z",
  "data": {
    ...event-specific fields...
  }
}
```

### Example: `milestone.approved`

```json
{
  "id": "d3b7a2c1-...",
  "subscriptionId": "s1a2b3c4-...",
  "eventType": "milestone.approved",
  "timestamp": "2025-06-15T14:32:00Z",
  "data": {
    "escrowId": "12345",
    "milestoneIndex": 1,
    "clientAddress": "GABCD...",
    "freelancerAddress": "GXYZ...",
    "amountReleased": "500000000",
    "tenantSlug": "acme"
  }
}
```

### Example: `dispute.raised`

```json
{
  "id": "a1b2c3d4-...",
  "subscriptionId": "s1a2b3c4-...",
  "eventType": "dispute.raised",
  "timestamp": "2025-06-15T14:35:00Z",
  "data": {
    "escrowId": "12345",
    "disputeId": 42,
    "raisedByAddress": "GXYZ...",
    "reason": "Deliverable does not match acceptance criteria"
  }
}
```

---

## Verifying Signatures

Every delivery includes an `X-Webhook-Signature` header. This is an HMAC-SHA256 of the raw request body, signed with the `secret` returned when you subscribed.

Verify it before processing the payload:

```javascript
import { createHmac } from 'crypto';

function verifyWebhookSignature(rawBody, secret, signatureHeader) {
  const expected = createHmac('sha256', secret)
    .update(rawBody)
    .digest('hex');
  const received = signatureHeader?.replace('sha256=', '') ?? '';
  return timingSafeEqual(
    Buffer.from(expected, 'hex'),
    Buffer.from(received, 'hex')
  );
}
```

**Never skip signature verification.** Without it, an attacker who knows your webhook URL can send you fake events.

---

## Retry Behaviour

The delivery queue uses exponential backoff:

| Attempt | Delay before retry |
|---|---|
| 1st retry | ~5 seconds |
| 2nd retry | ~10 seconds |
| 3rd retry | ~20 seconds |
| 4th retry | ~40 seconds |
| 5th retry | ~80 seconds |

After 5 failed attempts, the delivery is marked `failed` and stored for inspection. Retry counts and delays are configurable via environment variables:

```
WEBHOOK_MAX_RETRY_ATTEMPTS=5
WEBHOOK_BACKOFF_BASE_MS=5000
WEBHOOK_KEEP_FAILED_JOBS=100
```

---

## Listing Subscriptions

```http
GET /api/webhooks/subscriptions
```

Returns all active webhook subscriptions for the authenticated user.

---

## Listing Deliveries

```http
GET /api/webhooks/deliveries?subscriptionId=<id>&limit=30
```

Returns recent delivery attempts. Each delivery record includes:
- Status (pending, delivered, failed)
- Attempt count
- Last response status and body
- Timestamps for each attempt

The `limit` parameter is capped at 100.

---

## Deleting a Subscription

```http
DELETE /api/webhooks/subscriptions/:id
```

Removes the subscription. In-flight deliveries will still complete, but no new events will be queued for this subscription.

---

## Security Considerations

- Only HTTPS URLs are accepted for subscriptions — no HTTP, no localhost
- Signatures prevent forged deliveries
- Rate limiting prevents subscription flooding (10 per 10 minutes per user)
- Delivery logs let you audit what was sent and when

---

## Related

- [API Reference → Webhooks](../api-reference/webhooks.md) — endpoint reference with full request/response schemas
- [Security → Rate Limiting](../security/rate-limiting.md) — how the subscription rate limiter works
