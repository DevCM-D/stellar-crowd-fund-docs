# Authentication

All API endpoints (except the health check) require authentication via JSON Web Tokens (JWT).

---

## Token Types

The platform issues two types of tokens:

| Token | Lifetime | Purpose |
|---|---|---|
| Access token | 15 minutes | Authorize API requests |
| Refresh token | 7 days | Obtain new access tokens without re-login |

Access tokens are short-lived intentionally. Use the refresh token endpoint to renew them silently in the background.

---

## Authenticating

Include the access token in every request as a Bearer token:

```http
Authorization: Bearer <access_token>
```

---

## Obtaining Tokens

Tokens are issued after verifying a user's Stellar wallet signature. The flow:

### 1. Request a challenge

```http
GET /api/auth/challenge?address=G...
```

**Response:**
```json
{
  "challenge": "sign-this-string-abc123",
  "expiresIn": 300
}
```

The challenge is a short-lived nonce tied to the requesting address. It expires after 5 minutes.

### 2. Sign the challenge

Sign the challenge string with the user's Stellar private key using the Freighter wallet or a compatible Stellar SDK:

```javascript
// In the browser with Freighter
const { signedXDR } = await window.freighter.signTransaction(challenge, {
  networkPassphrase: Networks.TESTNET,
});
```

### 3. Exchange for tokens

```http
POST /api/auth/login
Content-Type: application/json

{
  "address": "G...",
  "signature": "<signed_challenge_xdr>"
}
```

**Response (200):**
```json
{
  "accessToken": "eyJ...",
  "refreshToken": "eyJ...",
  "expiresIn": 900
}
```

---

## Refreshing Tokens

```http
POST /api/auth/refresh
Content-Type: application/json

{
  "refreshToken": "eyJ..."
}
```

**Response (200):**
```json
{
  "accessToken": "eyJ...",
  "expiresIn": 900
}
```

The refresh token is rotated on use — you receive a new refresh token alongside the new access token. Store the new refresh token and discard the old one.

---

## MFA (Multi-Factor Authentication)

Accounts with MFA enabled must complete a second factor after the wallet signature step:

```http
POST /api/auth/mfa/verify
Authorization: Bearer <short-lived-mfa-token>
Content-Type: application/json

{
  "code": "123456"
}
```

On success, the MFA token is exchanged for a full access token and refresh token.

---

## Token Claims

The JWT payload contains:

```json
{
  "sub": "G...",           // Stellar address (user identifier)
  "tenantId": "...",       // Tenant CUID
  "iat": 1718000000,
  "exp": 1718000900
}
```

The API uses `sub` as the user's identifier on all write operations.

---

## Logout

```http
POST /api/auth/logout
Authorization: Bearer <access_token>
```

Invalidates the current refresh token server-side. Access tokens cannot be revoked (they expire naturally); ensure access token lifetime is short.

---

## Error Responses

| Status | Code | Meaning |
|---|---|---|
| `401` | `MISSING_TOKEN` | No Authorization header |
| `401` | `INVALID_TOKEN` | Malformed, expired, or tampered JWT |
| `401` | `TOKEN_REVOKED` | Refresh token has been invalidated |
| `403` | `INSUFFICIENT_SCOPE` | Token valid but lacks required permissions |

---

## Environment Variables

```env
JWT_ACCESS_SECRET=<strong-random-secret>
JWT_REFRESH_SECRET=<different-strong-random-secret>
JWT_MFA_SECRET=<third-distinct-secret>
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d
```

These must all be set and must not be default or guessable values. The preflight checker enforces this at startup.

---

## Related

- [Security → Secrets](../security/secrets.md) — JWT secret requirements and rotation
- [Deployment → Environment Variables](../deployment/environment-variables.md) — full env var reference
- [API Reference → Health](health.md) — the only unauthenticated endpoint
