# Quickstart

Get the full stack — API, web dashboard, and mobile app — running locally in a single session.

**Time estimate:** 15–25 minutes (longer if compiling the Rust contract for the first time).

---

## Before You Start

Make sure you have the [required dependencies](prerequisites.md) installed. At minimum:
- Node.js 18+
- PostgreSQL 14+
- Redis 7+

---

## Step 1 — Clone the repository

```bash
git clone https://github.com/DevCM-D/Stellar-Crowd-Fund-Escrow.git
cd Stellar-Crowd-Fund-Escrow
```

---

## Step 2 — Run the preflight checker

```bash
node scripts/preflight.js
```

The preflight script checks:
- Node.js version (must be 18+)
- Required environment variables are present and not set to dangerous defaults
- `DATABASE_URL` format is valid
- Working tree is clean (in production mode)

It exits with a clear error message if anything is wrong. Fix the reported issues before continuing.

> On a fresh clone the script will tell you `JWT_SECRET is not set` — that is expected. Set it in the next step.

---

## Step 3 — Install dependencies

```bash
# Root workspace (linting, Git hooks)
npm install

# Backend API
cd backend && npm install && cd ..

# Web dashboard
cd frontend && npm install && cd ..
```

---

## Step 4 — Configure environment variables

```bash
cp backend/.env.example backend/.env
```

Open `backend/.env` and fill in the values. For local development, the minimum viable configuration is:

```env
# Database
DATABASE_URL="postgresql://escrow:escrow@localhost:5432/stellar_escrow"

# Redis (optional — falls back to in-memory if unset)
REDIS_URL="redis://localhost:6379"

# JWT secrets — generate each with: openssl rand -hex 64
JWT_SECRET="paste-64-byte-hex-here"
JWT_ACCESS_SECRET="paste-64-byte-hex-here"
JWT_REFRESH_SECRET="paste-64-byte-hex-here"

# Stellar
STELLAR_NETWORK="testnet"
SOROBAN_RPC_URL="https://soroban-testnet.stellar.org"
CONTRACT_ID="your-deployed-contract-address"

# Runtime
NODE_ENV="development"
PORT=4000
LOG_LEVEL="info"
```

Generate secrets:

```bash
openssl rand -hex 64   # run once per secret
```

> Never commit `.env` files. Never reuse secrets between environments.

For the frontend:

```bash
cp frontend/.env.example frontend/.env.local
```

Set `NEXT_PUBLIC_API_URL=http://localhost:4000` in `frontend/.env.local`.

---

## Step 5 — Set up the database

Create the database if it doesn't exist:

```bash
createdb stellar_escrow   # or use psql / pgAdmin
```

Run migrations:

```bash
cd backend
npx prisma migrate dev --name init
npx prisma generate
cd ..
```

Seed with sample data (optional but recommended for development):

```bash
cd backend

# Preview what will be seeded without writing
node ../scripts/seed.js --dry-run

# Write the seed data
node ../scripts/seed.js
```

The seed script is idempotent — running it twice will not create duplicates.

---

## Step 6 — Run the preflight check again

Now that the environment is configured:

```bash
node scripts/preflight.js
```

All checks should pass this time. If any still fail, the output will tell you exactly what is wrong and where.

---

## Step 7 — Start the development servers

Open three terminal windows:

**Terminal 1 — Backend API**
```bash
cd backend && npm run dev
```
The API starts on `http://localhost:4000`. You should see:
```
[info] Server listening on port 4000
[info] Database connection established
```

**Terminal 2 — Web Dashboard**
```bash
cd frontend && npm run dev
```
The dashboard starts on `http://localhost:3000`.

**Terminal 3 — Mobile App (optional)**
```bash
cd mobile && npx expo start
```
Scan the QR code with the Expo Go app on your phone, or press `i` for iOS simulator / `a` for Android emulator.

---

## Step 8 — Verify everything is working

**API health check:**
```bash
curl http://localhost:4000/health | jq .
```

Expected response (abbreviated):
```json
{
  "status": "ok",
  "components": {
    "db": { "status": "ok", "latencyMs": 2.1 },
    "cache": { "status": "ok" },
    "stellar": { "status": "ok", "ledger": 1234567 }
  }
}
```

If `stellar.status` is `"error"`, the testnet RPC is unreachable. This does not affect the API's core functionality for local testing.

**Web dashboard:** Open `http://localhost:3000` in your browser. Install Freighter if you haven't already and switch it to Testnet.

---

## Using Docker for the Data Layer

If you prefer not to install PostgreSQL and Redis locally, use Docker Compose:

```bash
docker compose up -d
```

This starts PostgreSQL on port 5432 and Redis on port 6379. Your `DATABASE_URL` and `REDIS_URL` in `.env` should point to `localhost` on those ports.

Then run the API normally:

```bash
cd backend && npm run dev
```

---

## Running the Test Suite

```bash
cd backend && npm test
```

All 39 test suites (425 tests) should pass. The pre-push Git hook runs these automatically before any push — a failed test blocks the push.

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| `Error: JWT_SECRET environment variable is required` | `.env` not created or secret missing | Run `cp backend/.env.example backend/.env` and fill in secrets |
| `Can't reach database server` | PostgreSQL not running | Start PostgreSQL or `docker compose up -d` |
| `ECONNREFUSED 127.0.0.1:6379` | Redis not running | Start Redis or `docker compose up -d` |
| `P1001: Can't reach database` | Wrong `DATABASE_URL` | Check the connection string matches your PostgreSQL setup |
| Freighter shows "No connection" | Wrong network selected | Switch Freighter to Testnet in its settings |
| Contract calls fail | `CONTRACT_ID` not set | Deploy a contract or use a known testnet contract ID |

---

## Next Steps

- [Local Sandbox](local-sandbox.md) — run a full local Stellar node instead of using testnet
- [Escrow Lifecycle](../concepts/escrow-lifecycle.md) — understand the states and transitions
- [API Reference](../api-reference/authentication.md) — integrate with the REST API
