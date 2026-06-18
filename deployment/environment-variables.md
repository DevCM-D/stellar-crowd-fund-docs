# Environment Variables

All configuration is driven by environment variables. Never hardcode secrets or environment-specific values in the codebase.

---

## Required Variables

The platform will not start without these. The preflight checker (`node scripts/preflight.js`) verifies them at startup.

### Database

| Variable | Description | Example |
|---|---|---|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@localhost:5432/stellar_escrow` |

The URL must begin with `postgresql://` or `postgres://`. Other formats are rejected by the preflight check.

### JWT Secrets

All three secrets must be set and must not be default/guessable values (`secret`, `changeme`, `development`, `test`).

| Variable | Description |
|---|---|
| `JWT_SECRET` | General-purpose fallback secret |
| `JWT_ACCESS_SECRET` | Signs access tokens |
| `JWT_REFRESH_SECRET` | Signs refresh tokens |
| `JWT_MFA_SECRET` | Signs MFA challenge tokens |

Use a strong random generator:

```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

Generate a separate value for each variable.

### Stellar / Soroban

| Variable | Description | Example |
|---|---|---|
| `STELLAR_NETWORK` | `testnet` or `mainnet` | `testnet` |
| `SOROBAN_RPC_URL` | Soroban RPC endpoint | `https://soroban-testnet.stellar.org` |
| `CONTRACT_ID` | Deployed escrow contract address | `CAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA...` |

---

## Optional Variables

These have safe defaults, but should be configured for production.

### Server

| Variable | Default | Description |
|---|---|---|
| `PORT` | `3000` | HTTP server port |
| `NODE_ENV` | `development` | Set to `production` in production |
| `LOG_LEVEL` | `info` | `debug`, `info`, `warn`, `error` |

### Redis

| Variable | Default | Description |
|---|---|---|
| `REDIS_URL` | `redis://localhost:6379` | Redis connection string |
| `CACHE_TTL_SECONDS` | `300` | Default HTTP cache TTL |

### Elasticsearch (Optional)

Leave unset to disable Elasticsearch and use Prisma fallback for search and leaderboard.

| Variable | Description |
|---|---|
| `ELASTICSEARCH_URL` | Elasticsearch cluster URL |
| `ELASTICSEARCH_USERNAME` | Elasticsearch auth username |
| `ELASTICSEARCH_PASSWORD` | Elasticsearch auth password |

### IPFS

| Variable | Default | Description |
|---|---|---|
| `IPFS_GATEWAY_URL` | `https://ipfs.io` | Gateway for reading IPFS content |
| `IPFS_API_URL` | `http://localhost:5001` | Kubo API for uploading content |

### Webhook Queue

| Variable | Default | Description |
|---|---|---|
| `WEBHOOK_MAX_RETRY_ATTEMPTS` | `5` | Max delivery retries before marking failed |
| `WEBHOOK_BACKOFF_BASE_MS` | `5000` | Base delay for exponential backoff (ms) |
| `WEBHOOK_KEEP_FAILED_JOBS` | `100` | Number of failed jobs to retain in the queue |

### JWT Expiry

| Variable | Default | Description |
|---|---|---|
| `JWT_ACCESS_EXPIRY` | `15m` | Access token lifetime |
| `JWT_REFRESH_EXPIRY` | `7d` | Refresh token lifetime |
| `JWT_MFA_EXPIRY` | `5m` | MFA challenge token lifetime |

---

## `.env` File

For local development, create a `.env` file at the project root:

```env
# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/stellar_escrow_dev

# JWT
JWT_SECRET=<generate with: node -e "console.log(require('crypto').randomBytes(64).toString('hex'))">
JWT_ACCESS_SECRET=<separate random value>
JWT_REFRESH_SECRET=<separate random value>
JWT_MFA_SECRET=<separate random value>

# Stellar
STELLAR_NETWORK=testnet
SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
CONTRACT_ID=<your testnet contract id>

# Optional
REDIS_URL=redis://localhost:6379
NODE_ENV=development
LOG_LEVEL=debug
```

Never commit `.env` to version control. It is in `.gitignore`.

---

## Validation at Startup

Run the preflight checker before deploying:

```bash
node scripts/preflight.js
```

It checks:
- Node.js version (≥ 18 required)
- All required env vars are set
- No required var contains a known-dangerous default value
- `DATABASE_URL` has the correct format
- In production, the git working tree is clean

See [Deployment → Preflight](preflight.md) for full details.

---

## Related

- [Deployment → Preflight](preflight.md) — startup validation script
- [Security → Secrets](../security/secrets.md) — secret rotation and management
- [Getting Started → Quickstart](../getting-started/quickstart.md) — minimal local setup
