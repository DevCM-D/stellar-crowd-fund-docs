# Secrets Management

The platform uses secrets for JWT signing and webhook delivery HMAC. These secrets must be treated with the same care as private keys — exposure allows token forgery and unverifiable webhook deliveries.

---

## JWT Secrets

Three separate secrets are required:

| Variable | Used for |
|---|---|
| `JWT_ACCESS_SECRET` | Signs 15-minute access tokens |
| `JWT_REFRESH_SECRET` | Signs 7-day refresh tokens |
| `JWT_MFA_SECRET` | Signs 5-minute MFA challenge tokens |
| `JWT_SECRET` | Fallback / legacy validation (also required) |

### Why separate secrets?

If a single secret is used for all token types, a stolen refresh token could be crafted into an access token of any scope. Separate secrets ensure that a token signed for one purpose is cryptographically invalid for another.

### Generating secrets

```bash
# Generate one secret
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"

# Generate four at once
for i in 1 2 3 4; do
  node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
done
```

Use 64 bytes (128 hex characters) as a minimum. Shorter secrets reduce the entropy of the HMAC.

---

## Weak Secret Detection

The preflight checker (`scripts/preflight.js`) rejects the following known-weak values in any JWT secret:

- `secret`
- `changeme`
- `development`
- `test`

The server will not start if any of these appear in a JWT secret variable. This prevents a common misconfiguration where a dev environment secret is accidentally used in production.

---

## Webhook Secrets

Each webhook subscription has its own HMAC-SHA256 secret, generated at subscribe time. These are stored in the `WebhookSubscription` table and used to sign delivery payloads.

Webhook secrets are never returned by the API after the initial subscribe response. If a secret is lost, the subscription must be deleted and recreated.

---

## Secret Rotation

### JWT secrets

JWT secrets can be rotated without logging users out, using a rolling rotation:

1. Add a new secret as `JWT_ACCESS_SECRET_NEW`
2. Update the token verification to accept signatures from both old and new secrets
3. Deploy; new tokens are signed with the new secret
4. After one access token TTL (15 min), all valid tokens use the new secret
5. Remove the old secret from the validator and environment

For refresh tokens (7-day TTL), repeat with `JWT_REFRESH_SECRET` but wait 7 days between steps 2 and 5.

For a forced logout of all users (e.g. security incident):

1. Replace the secret immediately
2. All existing tokens are immediately invalid
3. All users must re-authenticate

### Webhook secrets

Webhook secrets are per-subscription. To rotate:

1. Delete the existing subscription (`DELETE /api/webhooks/subscriptions/:id`)
2. Create a new subscription (`POST /api/webhooks/subscribe`)
3. Distribute the new secret to the subscriber

---

## Environment Variable Security

- Never put secrets in `.env` files that are committed to version control
- Use a secrets manager (AWS Secrets Manager, HashiCorp Vault, Doppler) for production
- Restrict access to the secrets manager to only the roles/services that need it
- Audit secret access logs periodically

For CI/CD pipelines, use repository secret storage (GitHub Actions Secrets, GitLab CI variables, etc.) rather than `.env` files.

---

## Related

- [Environment Variables](../deployment/environment-variables.md) — full list of secrets required
- [Preflight](../deployment/preflight.md) — weak secret detection at startup
- [Security Overview](overview.md) — how secrets fit in the security model
