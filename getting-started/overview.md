# Overview

## What is Stellar Crowd Fund Escrow?

Stellar Crowd Fund Escrow is a decentralised platform that enables **trustless, milestone-based payment agreements** between clients and contractors, secured by Soroban smart contracts on the Stellar blockchain.

Instead of trusting a company to hold and release funds fairly, both parties trust a contract — a piece of open-source code that executes exactly as written, cannot be altered mid-agreement, and is auditable by anyone.

---

## The Problem It Solves

Traditional freelance and crowd-funding platforms act as centralised custodians:

- Funds sit in a company's bank account, not in a neutral escrow
- Dispute resolution is handled internally — decisions are opaque and non-appealable
- Reputation scores belong to the platform, not the individual. Switch platforms and your history is gone
- Platform shutdowns or policy changes can freeze or forfeit your funds

**Stellar Crowd Fund Escrow removes the intermediary entirely:**

| Traditional platform | Stellar Crowd Fund Escrow |
|---|---|
| Company holds your funds | Soroban contract holds your funds |
| Dispute decided by a support team | Dispute resolved by an on-chain arbiter |
| Reputation stored in a private database | Reputation stored as contract state — portable and permanent |
| Rules can change without warning | Contract logic is immutable once deployed |
| Funds can be frozen by the platform | Funds can only move according to contract rules |

---

## Key Capabilities

### Milestone-Based Escrow
Funds are divided into milestones before the agreement starts. Each milestone has an agreed amount and a description (stored as an IPFS content hash). Funds for each milestone are released only when the client approves that milestone — not all at once.

### On-Chain Reputation
Every outcome — milestone approved, dispute won, dispute lost — writes a `ReputationEvent` to the Soroban contract. These events accumulate into a score that is public, immutable, and tied to the participant's Stellar address (not our database). Your reputation travels with your wallet.

### Multi-Tenant Architecture
The platform can host multiple isolated organisations (tenants) under one deployment. Each tenant's data, cache, rate limits, and analytics are fully scoped by `tenantId` — a data leak between tenants would require bypassing every DB query's `WHERE` clause simultaneously.

### Dispute Resolution
Either party can raise a dispute at any time. Evidence (files, screenshots, documents) is uploaded to IPFS and linked on-chain. An authorised arbiter reviews the evidence and resolves the dispute by specifying how the remaining locked funds should be split.

### Webhook Notifications
Subscribers can register HTTPS endpoints to receive real-time event notifications (escrow created, milestone approved, dispute raised, etc.). Deliveries are signed with HMAC-SHA256 and retried with exponential backoff if your endpoint is temporarily unavailable.

---

## Who Uses This

| Role | What they do |
|---|---|
| **Client / Funder** | Creates escrows, locks funds, approves milestones, raises disputes |
| **Contractor / Freelancer** | Accepts work, submits completed milestones, builds on-chain reputation |
| **DAO / Community** | Pools funds from multiple contributors into a single escrow; approval can be gated to governance |
| **Arbiter** | Resolves disputes with on-chain authority; decisions are recorded permanently |
| **Developer / Integrator** | Self-hosts a tenant, integrates via the REST API, or extends the Soroban contract |

---

## How It Fits Into the Stellar Ecosystem

Stellar is a public blockchain optimised for fast, low-cost asset transfers. Soroban is Stellar's smart contract platform — Rust programs compiled to Wasm that run deterministically on the network.

This platform uses Stellar and Soroban for:
- Locking funds in the escrow contract (any Stellar asset — USDC, XLM, etc.)
- Releasing funds on milestone approval
- Writing immutable reputation events
- Dispute resolution with on-chain state

The backend API, database, and frontend are off-chain infrastructure that index on-chain events, provide a user interface, and handle non-financial operations (search, notifications, evidence storage).

---

## Next Steps

- [Quickstart](quickstart.md) — get the full stack running locally
- [Escrow Lifecycle](../concepts/escrow-lifecycle.md) — understand how an escrow moves from creation to completion
- [Architecture Overview](../architecture/overview.md) — see how all the layers connect
