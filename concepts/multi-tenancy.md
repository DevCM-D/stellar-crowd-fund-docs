# Multi-Tenancy

The platform is designed from the ground up to support multiple isolated organisations (tenants) running on a single deployment. Each tenant operates as if they have their own private instance — their data, cache, rate limits, and analytics are completely separated.

---

## What a Tenant Is

A tenant represents an organisation or community using the platform. Examples:

- A freelance marketplace that embeds escrow functionality
- A DAO that runs its own contributor payment system
- A company that self-hosts the platform for internal use

Each tenant has its own:
- Escrows, milestones, disputes, and reputation records
- Cache namespace (keys are prefixed with the tenant slug)
- Rate limit counters
- Analytics metrics
- Webhook subscriptions and deliveries

---

## How Tenant Isolation Works

Every database table that contains tenant-specific data has a `tenantId` column. Every query that reads or writes that table includes `WHERE tenant_id = :tenantId` — enforced at the Prisma query level, not just at the application level.

The tenant is identified from the incoming request via middleware, before any controller logic runs:

```
Request → tenantMiddleware → req.tenant = { id, slug, name }
              │
              ▼
         All controllers read req.tenant.id
         All Prisma queries scope WHERE clause to this id
```

A controller that forgets to include `req.tenant.id` in its query would return data from all tenants — this is tested and caught during review.

---

## Tenant Identification

Tenants are typically identified by one of:

- A **subdomain**: `acme.platform.io` → tenant slug `acme`
- A **request header**: `X-Tenant-Id: acme`
- An **API key** prefix that maps to a tenant

The specific identification strategy depends on how the platform is deployed. See the deployment documentation for configuration options.

---

## Cache Scoping

Cache keys are scoped by tenant slug to prevent data from one tenant appearing in another tenant's responses.

Cache key format:
```
http:<tenantSlug>:<method>:<path>[:<queryString>]
```

Example:
```
http:acme:GET:/api/escrows:page=1&limit=20
http:beta:GET:/api/escrows:page=1&limit=20
```

These are separate cache entries even though they hit the same endpoint. Invalidating `acme`'s cache does not affect `beta`.

---

## Analytics Scoping

Route-level analytics metrics are prefixed with the tenant slug:

```
[acme] GET /api/escrows
[beta] GET /api/escrows
```

This means per-tenant dashboards can be built without post-processing — filter by prefix to see only one tenant's metrics.

---

## Rate Limiting

Each tenant's users have separate rate limit counters. A burst from tenant A's users cannot consume tenant B's quota.

Rate limit keys:
```
sliding:user:<address>           (per-user sliding window)
webhook-subscribe:addr:<address> (webhook subscribe endpoint)
```

These keys are not tenant-scoped at the rate-limiter level, but since user addresses are globally unique on Stellar, there is no cross-tenant collision. Tenant-scoped keys can be added if isolation requires it.

---

## Logging and Tracing

Every HTTP request log line includes the tenant slug:

```json
{
  "type": "http_request",
  "requestId": "abc-123",
  "correlationId": "xyz-456",
  "tenantSlug": "acme",
  "method": "GET",
  "path": "/api/escrows",
  "statusCode": 200,
  "durationMs": 12.3
}
```

This allows log queries to filter by tenant without joining on request path.

---

## Database Indexes

All frequently queried tables have composite indexes on `(tenantId, ...)` to ensure that tenant-scoped queries use an index rather than a full table scan:

```
@@index([tenantId, status])
@@index([tenantId, status, createdAt(sort: Desc)])
@@index([tenantId, clientAddress])
@@index([tenantId, raisedByAddress, raisedAt(sort: Desc)])
```

As tenant data volume grows, queries stay fast because the index narrows the scan to only that tenant's rows.

---

## Bypassing Tenant Scope

Some operations — like admin dispute resolution or platform-level statistics — need to read data across all tenants. These use a deliberate `withTenantScopeBypassed()` utility that makes the bypass explicit and searchable in code review:

```javascript
import { withTenantScopeBypassed } from '../lib/tenantContext.js';

const globalStats = await withTenantScopeBypassed(async () => {
  return prisma.escrow.count();
});
```

Any use of this utility is a code review flag — it should be justified with a comment.

---

## Related

- [Architecture Overview](../architecture/overview.md) — where tenant middleware fits in the request chain
- [Database Schema](../architecture/database-schema.md) — tenant columns and indexes
- [Security Overview](../security/overview.md) — tenant isolation as a security boundary
