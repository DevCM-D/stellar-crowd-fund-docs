# Security Overview

Security is applied in layers — no single control is relied on exclusively. The principle is defence-in-depth: if one layer fails, others catch the issue.

---

## Threat Model

The primary threats addressed by the platform:

| Threat | Mitigation |
|---|---|
| Stolen JWT used after expiry | Short-lived access tokens (15 min) |
| Weak JWT secrets enabling token forgery | Secrets validated at startup; weak defaults rejected |
| Unauthenticated access to tenant data | Auth middleware on all routes except `/health` |
| Cross-tenant data leakage | All queries scoped by `tenantId` at the Prisma layer |
| SSRF via webhook URLs | HTTPS-only webhook URLs; no private IP ranges accepted |
| Webhook spoofing | HMAC-SHA256 signature on every delivery |
| Input injection (SQL, XSS, command) | Validation at route layer; Prisma parameterized queries |
| API abuse / DoS via endpoint flooding | Sliding-window rate limiter per user address |
| Weak contract security | Soroban enforces on-chain auth for every state change |
| Evidence tampering | IPFS content addressing — hash mismatch = different file |
| Malware in evidence files | Virus scan on upload before IPFS storage |
| Exposed secrets in deployments | Preflight checker rejects known-weak default values |

---

## Authentication and Authorisation

### Authentication

Users authenticate by signing a challenge with their Stellar private key. This proves ownership of the address without transmitting the private key. The signed challenge is exchanged for a JWT access token and a refresh token.

- Access tokens expire after 15 minutes
- Refresh tokens rotate on use
- MFA tokens are signed with a separate secret and expire in 5 minutes

### Authorisation

Route-level authorisation checks that the authenticated user is permitted to perform the requested action:

- Only the client or freelancer on an escrow can raise a dispute on it
- Only the client can approve milestones
- Only the freelancer can submit milestones
- Only an admin/arbiter can resolve disputes

These checks live in controllers, close to the data access, so they cannot be accidentally bypassed by middleware changes.

---

## Tenant Isolation

Every database table that holds tenant data has a `tenantId` column. Every query that accesses tenant data includes `WHERE tenantId = :id` scoped to the authenticated tenant.

This is enforced at the Prisma query level — not just application logic — so a missing condition in a controller is caught by the database layer. Bypassing tenant scope is an explicit, searchable code pattern (`withTenantScopeBypassed`) that is flagged in code review.

---

## Input Validation

All user input is validated and sanitized before it reaches controllers or the database:

- `express-validator` rules on every route that accepts input
- Query string parameters are typed and range-checked
- Body fields have format, type, and length constraints
- Search queries are sanitized: control characters stripped, length capped

Input validation errors return `422 Unprocessable Entity` with field-level error details.

---

## Rate Limiting

Sliding-window rate limiting is applied per user address. Endpoints with abuse potential have tighter limits:

- Webhook subscribe: 10 requests per 10 minutes per address
- Auth challenge / login: standard per-user limit
- General API: per-user limit configurable per tier

Rate limit state is stored in Redis (not in-memory) so limits are enforced across multiple server instances.

---

## Webhook Security

- Webhook subscription URLs must use HTTPS — no HTTP, no `localhost`, no private IP ranges
- Every webhook delivery is signed with HMAC-SHA256 using a per-subscription secret
- Subscribers should verify the `X-Webhook-Signature` header before processing payloads
- The subscription secret is shown once at subscribe time and never returned again

---

## Smart Contract Security

The Soroban contracts enforce authorisation on every state-changing function:

- `create_escrow` — requires `client` signature
- `submit_milestone` — requires `freelancer` signature
- `approve_milestone` — requires `client` signature
- `raise_dispute` — requires `client` or `freelancer` signature
- `resolve_dispute` — requires `arbiter` signature
- `cancel_escrow` — requires both `client` and `freelancer` signatures

No amount split, fund release, or state transition can happen without the cryptographically verified signature of the required party. The API layer cannot bypass this — it can only construct and submit transactions; the contract rejects invalid auth.

---

## Secrets Management

- JWT secrets are required and must not contain known-weak values (`secret`, `changeme`, etc.)
- Three separate secrets for access, refresh, and MFA tokens — no shared secret
- The preflight checker validates secrets at startup; the server refuses to start without them
- Secrets are passed via environment variables; never hardcoded in source

See [Security → Secrets](secrets.md) for rotation and management guidance.

---

## Audit Logging

Security-relevant actions are logged with structured fields:

- All HTTP requests: method, path, status, duration, tenant, user, correlation ID
- Admin actions (rate limit changes): action type, previous value, new value, performed-by address
- Auth events: login, token refresh, MFA verification (with success/failure)

Logs are structured JSON, suitable for ingestion by any log aggregation system.

---

## Related

- [Security → Secrets](secrets.md) — secret requirements and rotation
- [Security → Rate Limiting](rate-limiting.md) — rate limiter configuration
- [Authentication API](../api-reference/authentication.md) — token flow
- [Multi-Tenancy](../concepts/multi-tenancy.md) — tenant isolation details
