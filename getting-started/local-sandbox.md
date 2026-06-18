# Local Stellar Sandbox

Run a full local Stellar network — instead of using the public testnet — so you can develop and test contract interactions without network latency or testnet outages.

---

## What the Sandbox Provides

| Component | Port | Purpose |
|---|---|---|
| Stellar Quickstart node | 8000 | Local Stellar network in standalone mode |
| Soroban RPC endpoint | 8000/soroban/rpc | JSON-RPC for contract simulation and submission |
| Horizon API | 8000 | Stellar Horizon REST API for account and transaction data |
| PostgreSQL | 5432 | Persistent storage for the API |
| Redis | 6379 | Cache and job queues |

---

## Requirements

- Docker 24+
- Soroban CLI 21.0.0+
- Rust 1.74+ (for compiling the contract)

See [Prerequisites](prerequisites.md) for installation instructions.

---

## Start the Sandbox

```bash
./scripts/start-sandbox.sh
```

The script does the following automatically:

1. Pulls and starts the `stellar/quickstart` Docker image in Soroban standalone mode
2. Waits for the node to be ready (polls `/soroban/rpc` health endpoint)
3. Compiles the Rust escrow contract to Wasm: `cargo build --release --target wasm32-unknown-unknown`
4. Deploys the compiled contract to the local network using Soroban CLI
5. Funds a developer wallet with testnet XLM using the built-in friendbot
6. Writes the deployed contract ID and RPC URL into `frontend/.env.local`

The script is **idempotent** — running it again rebuilds and redeploys the contract without tearing down the Stellar node or losing ledger state.

---

## Verify the Sandbox is Running

```bash
# Check Docker container
docker ps --filter name=stellar-sandbox

# Confirm the RPC endpoint responds
curl -sf http://localhost:8000/soroban/rpc \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getHealth"}' | jq .

# Expected output:
# { "jsonrpc": "2.0", "id": 1, "result": { "status": "healthy" } }

# Check the latest ledger
curl -sf http://localhost:8000/soroban/rpc \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getLatestLedger"}' | jq .result.sequence
```

---

## Point the API at the Sandbox

After running `start-sandbox.sh`, update `backend/.env`:

```env
STELLAR_NETWORK="standalone"
SOROBAN_RPC_URL="http://localhost:8000/soroban/rpc"
CONTRACT_ID="<contract_id_written_by_start-sandbox.sh>"
```

The script prints the contract ID when it finishes. You can also find it in `frontend/.env.local`.

---

## Rebuild and Redeploy a Contract Change

After editing Rust contract source files:

```bash
./scripts/start-sandbox.sh
```

The script detects the existing node, skips the startup wait, recompiles the Wasm, and redeploys — giving you a fresh contract instance without restarting the network. Ledger history and any manually submitted transactions are preserved.

---

## Manual Contract Deployment

If you prefer to deploy manually (for example, to test different contract versions):

```bash
# 1. Build
cd contracts/escrow_contract
cargo build --release --target wasm32-unknown-unknown

# 2. Upload the Wasm to the network
soroban contract upload \
  --wasm target/wasm32-unknown-unknown/release/escrow_contract.wasm \
  --rpc-url http://localhost:8000/soroban/rpc \
  --network-passphrase "Standalone Network ; February 2017"

# 3. Deploy (instantiate) the contract
soroban contract deploy \
  --wasm-hash <wasm-hash-from-step-2> \
  --rpc-url http://localhost:8000/soroban/rpc \
  --network-passphrase "Standalone Network ; February 2017" \
  --source <your-secret-key>
```

Copy the printed contract ID into `backend/.env` as `CONTRACT_ID`.

---

## Fund a Test Wallet

The sandbox includes a friendbot that mints XLM for any account:

```bash
curl "http://localhost:8000/friendbot?addr=<your-stellar-address>"
```

---

## Teardown

```bash
docker compose down
```

This stops and removes the PostgreSQL, Redis, and Stellar containers. All ledger state is lost. Run `start-sandbox.sh` again to rebuild from scratch.

To stop containers without removing them (preserving state):

```bash
docker compose stop
docker compose start   # restart later
```

---

## Switching Back to Testnet

To go back to the public Stellar testnet, update `backend/.env`:

```env
STELLAR_NETWORK="testnet"
SOROBAN_RPC_URL="https://soroban-testnet.stellar.org"
CONTRACT_ID="<your-testnet-contract-id>"
```

Restart the API:

```bash
cd backend && npm run dev
```
</content>
</invoke>