# Prerequisites

Everything you need installed and configured before running the project locally.

---

## Required

### Node.js 18+

The API and frontend both require Node.js 18 or later. Earlier versions are not supported.

```bash
# Check your version
node --version   # must be v18.x or higher

# Recommended: use nvm to manage versions
nvm install 18
nvm use 18
```

Install nvm: [nvm-sh/nvm](https://github.com/nvm-sh/nvm)

---

### PostgreSQL 14+

The API stores all off-chain data (escrows, milestones, disputes, reputation records) in PostgreSQL.

**Option A — Install locally**

```bash
# macOS
brew install postgresql@14
brew services start postgresql@14

# Ubuntu / Debian
sudo apt install postgresql-14
sudo service postgresql start
```

**Option B — Use Docker (recommended for development)**

```bash
docker run -d \
  --name stellar-pg \
  -e POSTGRES_USER=escrow \
  -e POSTGRES_PASSWORD=escrow \
  -e POSTGRES_DB=stellar_escrow \
  -p 5432:5432 \
  postgres:14
```

Your `DATABASE_URL` will then be:
```
postgresql://escrow:escrow@localhost:5432/stellar_escrow
```

---

### Redis 7+

Redis is used for HTTP response caching and the BullMQ job queue (webhook delivery, email dispatch).

The API falls back to in-memory caching if Redis is unavailable, but the job queue requires Redis for reliable webhook delivery.

**Option A — Install locally**

```bash
# macOS
brew install redis
brew services start redis

# Ubuntu / Debian
sudo apt install redis-server
sudo service redis-server start
```

**Option B — Use Docker**

```bash
docker run -d \
  --name stellar-redis \
  -p 6379:6379 \
  redis:7
```

Your `REDIS_URL` will then be `redis://localhost:6379`.

---

### Git

Required to clone the repository and run the pre-push hooks.

```bash
git --version   # must be 2.x or higher
```

---

## Required for Smart Contract Development

If you plan to modify or redeploy the Soroban escrow contract, you also need:

### Rust and the Wasm target

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Verify
rustc --version   # must be 1.74 or higher

# Add the Wasm compilation target
rustup target add wasm32-unknown-unknown
```

### Soroban CLI

```bash
cargo install --locked soroban-cli --version 21.0.0

# Verify
soroban --version
```

---

## Required for Web Interaction

### Freighter Wallet (browser extension)

Freighter is the Stellar browser wallet used for signing transactions on the web dashboard.

- Install: [freighter.app](https://www.freighter.app/)
- Available for: Chrome, Firefox, Brave

After installing, create or import a Stellar account and switch to **Testnet** for local development.

---

## Optional

### Docker Desktop

Used for running the full local Stellar sandbox (PostgreSQL + Redis + Stellar node in one command). Required only if you want a local blockchain instead of connecting to testnet.

- Install: [docker.com/get-started](https://www.docker.com/get-started/)
- Minimum version: 24

### Elasticsearch

Used to power full-text escrow search. The API falls back gracefully to Prisma `ILIKE` queries when Elasticsearch is not available. You do not need it for basic local development.

```bash
# Run with Docker
docker run -d \
  --name stellar-es \
  -e discovery.type=single-node \
  -p 9200:9200 \
  elasticsearch:8.11.0
```

---

## Environment Summary

Once all dependencies are installed, verify your environment:

```bash
node --version      # v18.x or higher
npm --version       # 9.x or higher
git --version       # 2.x or higher
psql --version      # 14.x or higher (if local)
redis-cli --version # 7.x or higher (if local)
docker --version    # 24.x or higher (if using Docker)
```

When everything checks out, proceed to the [Quickstart](quickstart.md).
