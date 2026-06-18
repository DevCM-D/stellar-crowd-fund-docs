# Development Setup

Getting a local development environment running end-to-end.

---

## Prerequisites

Before starting, ensure these are installed:

| Tool | Version | Check |
|---|---|---|
| Node.js | ≥ 18 | `node --version` |
| npm | ≥ 9 | `npm --version` |
| PostgreSQL | ≥ 14 | `psql --version` |
| Redis | ≥ 7 | `redis-server --version` |
| Rust | stable (≥ 1.79) | `rustc --version` |
| Soroban CLI | latest | `soroban --version` |
| Git | any recent | `git --version` |

See [Getting Started → Prerequisites](../getting-started/prerequisites.md) for installation instructions.

---

## Clone and Install

```bash
git clone git@github.com:DevCM-D/Stellar-Crowd-Fund-Escrow.git
cd Stellar-Crowd-Fund-Escrow

# Install backend dependencies
cd backend
npm install

# Install mobile dependencies
cd ../mobile
npm install

# Install frontend dependencies
cd ../frontend
npm install
```

---

## Environment Setup

Copy the example env file and fill in values:

```bash
cp backend/.env.example backend/.env
```

Minimum required values for local dev:

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/stellar_escrow_dev
JWT_SECRET=any-long-random-string-for-local-only
JWT_ACCESS_SECRET=another-long-random-string
JWT_REFRESH_SECRET=yet-another-long-random-string
JWT_MFA_SECRET=one-more-random-string
STELLAR_NETWORK=testnet
SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
CONTRACT_ID=<your testnet contract ID>
```

For local dev, weak secrets are acceptable — use strong values only for staging and production.

---

## Database Setup

```bash
# Create the database
createdb stellar_escrow_dev

# Apply migrations
cd backend
npx prisma migrate dev

# Seed with sample data
node scripts/seed.js
```

To reset and re-seed from scratch:

```bash
npx prisma migrate reset
node scripts/seed.js --force
```

---

## Start Services

```bash
# Start Redis (if not already running)
redis-server --daemonize yes

# Start the API server
cd backend
npm run dev

# Start the web frontend (separate terminal)
cd frontend
npm run dev

# Start the mobile app (separate terminal)
cd mobile
npx expo start
```

The API server runs on `http://localhost:3000` by default. The frontend runs on `http://localhost:3001`.

---

## Build and Test the Contract

```bash
cd contracts/escrow

# Build
make build

# Run contract unit tests
make test

# Deploy to testnet (requires Soroban CLI configured with a funded account)
make deploy-testnet
```

After deployment, update `CONTRACT_ID` in your `.env` file.

---

## Running the Full Test Suite

```bash
cd backend
npm test
```

This runs all 425+ backend tests. Tests must pass before opening a pull request.

For a single test file:

```bash
npm test -- --testPathPattern=webhookController
```

See [Testing Guide](testing-guide.md) for more detail.

---

## Git Hooks

The repository uses Husky pre-push hooks:

- **pre-push**: Runs the full backend test suite before allowing a push

If the test suite fails, the push is blocked. Fix the failing tests before pushing.

---

## Related

- [Prerequisites](../getting-started/prerequisites.md) — tool installation
- [Quickstart](../getting-started/quickstart.md) — end-to-end quickstart guide
- [Testing Guide](testing-guide.md) — writing and running tests
- [Commit Conventions](commit-conventions.md) — commit message format
