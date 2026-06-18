# Preflight Checker

The preflight script (`scripts/preflight.js`) validates the deployment environment before the server starts. It catches misconfiguration early — before a request is served — rather than at the point of failure.

---

## What It Checks

### 1. Node.js version

Requires Node.js ≥ 18. Exits with code 1 if the running version is lower.

```
✓ Node.js v20.11.0 (required ≥ 18)
```

### 2. Required environment variables

Checks that all of these are set and non-empty:

- `DATABASE_URL`
- `JWT_SECRET`
- `JWT_ACCESS_SECRET`
- `STELLAR_NETWORK`
- `SOROBAN_RPC_URL`
- `CONTRACT_ID`

```
✓ DATABASE_URL is set
✓ JWT_SECRET is set
✗ JWT_ACCESS_SECRET is not set  ← exits here
```

### 3. Dangerous default values

Checks that JWT secrets do not contain known-weak values:
`secret`, `changeme`, `development`, `test`

```
✗ JWT_SECRET contains a known-weak default value ("secret")
```

### 4. DATABASE_URL format

Verifies the connection string begins with `postgresql://` or `postgres://`.

```
✓ DATABASE_URL format is valid (postgresql://)
```

### 5. Git working tree (production only)

In `NODE_ENV=production`, checks that the git working tree is clean. A dirty working tree in production means local changes that are not version-controlled, which is a deployment anti-pattern.

```
✓ Git working tree is clean
```

---

## Running the Preflight Check

```bash
node scripts/preflight.js
```

Exit code 0 means all checks passed. Exit code 1 means at least one check failed.

Integrate it into your server startup:

```json
// package.json
{
  "scripts": {
    "start": "node scripts/preflight.js && node backend/server.js"
  }
}
```

Or in your Dockerfile:

```dockerfile
CMD ["sh", "-c", "node scripts/preflight.js && node backend/server.js"]
```

---

## Output

All output goes to `stdout`. On failure, the failing check is printed and the process exits immediately.

```
Stellar Crowd Fund Escrow — preflight check
─────────────────────────────────────────
✓ Node.js v20.11.0 (required ≥ 18)
✓ DATABASE_URL is set
✓ JWT_SECRET is set
✓ JWT_ACCESS_SECRET is set
✓ STELLAR_NETWORK is set
✓ SOROBAN_RPC_URL is set
✓ CONTRACT_ID is set
✓ JWT_SECRET: no known-weak defaults detected
✓ JWT_ACCESS_SECRET: no known-weak defaults detected
✓ DATABASE_URL format: postgresql://
✓ Git working tree is clean
─────────────────────────────────────────
All checks passed. Starting server.
```

---

## Adding Custom Checks

Edit `scripts/preflight.js` to add project-specific checks. The pattern is:

```javascript
function check(label, condition, hint) {
  if (!condition) {
    console.error(`✗ ${label}`);
    if (hint) console.error(`  Hint: ${hint}`);
    process.exit(1);
  }
  console.log(`✓ ${label}`);
}

check(
  'STRIPE_SECRET_KEY is set',
  !!process.env.STRIPE_SECRET_KEY,
  'Set STRIPE_SECRET_KEY in your .env file'
);
```

---

## Related

- [Environment Variables](environment-variables.md) — full reference for all variables the preflight checks
- [Production Checklist](production-checklist.md) — pre-deploy validation beyond preflight
