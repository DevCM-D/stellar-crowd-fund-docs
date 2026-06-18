# Architecture Overview

Stellar Crowd Fund Escrow is structured as four cooperating layers: an on-chain contract layer, a backend API layer, a web frontend, and a mobile app. Each layer has a clearly defined role and communicates through well-specified interfaces.

---

## System Diagram

```
 ┌──────────────────────────────────────────────────────────────────┐
 │                         Clients                                   │
 │    Browser (Next.js)         Mobile App (Expo React Native)       │
 └──────────────┬───────────────────────────┬───────────────────────┘
                │ HTTPS REST                 │ HTTPS REST
                ▼                           ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │                     Express.js API Server                         │
 │                                                                   │
 │  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────────┐│
 │  │  Middleware  │  │  Controllers │  │   Background Queues      ││
 │  │  - Auth JWT  │  │  - Escrows   │  │   - Webhook delivery     ││
 │  │  - Tenant ID │  │  - Disputes  │  │     (BullMQ + Redis)     ││
 │  │  - Rate limit│  │  - Webhooks  │  │                          ││
 │  │  - Logging   │  │  - Reputation│  └──────────────────────────┘│
 │  │  - Analytics │  │  - Search    │                              │
 │  └─────────────┘  └──────────────┘                              │
 └──────────────────────────────────────────────────────────────────┘
         │ Prisma ORM       │ Redis client        │ Soroban SDK
         ▼                  ▼                     ▼
 ┌──────────────┐  ┌──────────────┐   ┌───────────────────────────┐
 │  PostgreSQL  │  │    Redis     │   │    Stellar Network        │
 │              │  │  - Cache     │   │    (Testnet / Mainnet)    │
 │  Prisma      │  │  - Sessions  │   │                           │
 │  migrations  │  │  - Rate limit│   │  Soroban Smart Contracts  │
 │  + indexes   │  │  - Job queue │   │  ┌─────────────────────┐  │
 └──────────────┘  └──────────────┘   │  │  escrow.wasm        │  │
                                       │  │  - create_escrow    │  │
 ┌──────────────┐                      │  │  - approve_milestone│  │
 │ Elasticsearch│                      │  │  - raise_dispute    │  │
 │  (optional)  │                      │  │  - resolve_dispute  │  │
 │  - Reputation│                      │  │  - reputation module│  │
 │    leaderboard                      │  └─────────────────────┘  │
 │  - Full-text │                      └───────────────────────────┘
 └──────────────┘
```

---

## Layer Responsibilities

### Soroban Smart Contracts (Rust)

The source of truth for all financial state. The contracts:

- Lock funds when an escrow is created
- Release milestone payments when approved
- Enforce the mutual-consent requirement for cancellation
- Write tamper-proof `ReputationEvent` records on-chain
- Arbitrate dispute resolution fund splits

The contracts do not know about tenants, users, or the database. They only know about Stellar addresses, token contracts, and amounts.

### Express.js API Server

Sits between clients and the contract. Responsibilities:

- Authenticate users via JWT
- Identify the current tenant from the request
- Validate and sanitize input before passing it on
- Mirror contract state to the PostgreSQL database for fast queries
- Serve REST endpoints for all platform features
- Queue webhook deliveries asynchronously
- Rate limit and audit log sensitive operations

The API is stateless — each request carries a JWT; session state lives in Redis (for rate limiting) and PostgreSQL (for data).

### PostgreSQL Database

The persistent store for the API layer. Contains:

- Mirror of on-chain escrow and milestone state (for fast queries and filtering)
- Off-chain reputation records rebuilt from on-chain events
- Dispute evidence metadata (IPFS CIDs, scan status, thumbnails)
- Tenant configuration
- Webhook subscriptions and delivery logs

If the database is ever lost, most state is rebuildable from the Stellar network. Evidence files live on IPFS and are referenced by their content hashes.

### Redis

Used for:

- HTTP response caching (tenant-scoped)
- Sliding-window rate limiting counters
- BullMQ job queue backend (webhook delivery)
- Session data (when applicable)

### BullMQ + Webhook Queue

Webhook deliveries run in a background worker process rather than blocking the HTTP response. The queue:

- Receives a delivery job when an event is emitted
- Assigns a deterministic job ID (`webhook:<deliveryId>`) to prevent duplicate deliveries on retry
- Retries with exponential backoff on failure
- Stores the last N failed jobs for inspection

### Elasticsearch (Optional)

An optional search backend for:

- Full-text reputation address search
- Leaderboard queries at scale

When Elasticsearch is unavailable, both features fall back to Prisma/PostgreSQL queries. The fallback is automatic and transparent to API consumers.

### Next.js Frontend

Web dashboard for tenants and their users. Provides:

- Escrow creation and management UI
- Milestone submission and approval flows
- Dispute filing and evidence upload
- Reputation leaderboard view

Built with React Server Components where possible; client components for interactive flows.

### Expo React Native Mobile App

Cross-platform mobile app for freelancers and clients who need on-the-go access. Features:

- Escrow listing with React Query caching and offline fallback
- Milestone notifications
- Biometric authentication (Touch ID / Face ID)
- Deep link support (open specific escrows from external links)

---

## Request Lifecycle (API)

A typical API request flows through these layers in order:

```
1. HTTP Request arrives
2. requestLogger middleware — assigns request ID, logs start
3. tenantMiddleware — identifies tenant from subdomain/header
4. auth middleware — validates JWT, attaches req.user
5. rateLimiter middleware — checks per-user/per-tenant quota
6. validation middleware — validates and sanitizes body/query
7. Controller — business logic, Prisma queries, contract calls
8. Response sent
9. analytics middleware — records route metric
10. requestLogger — logs completion with duration and status
```

---

## Data Flow: Escrow Creation

```
Client browser                 API Server              Stellar Network
      │                             │                        │
      │── POST /api/escrows ───────►│                        │
      │   { freelancer, amount,     │                        │
      │     milestones, ... }       │                        │
      │                             │── validate input        │
      │                             │── create DB record      │
      │                             │── call create_escrow() ►│
      │                             │                   ◄─── escrow_id
      │                             │── update DB with        │
      │                             │   on-chain escrow_id    │
      │                             │── emit escrow.created   │
      │                             │   webhook event         │
      │◄── 201 { escrow } ─────────│                        │
```

---

## Related

- [Smart Contracts](smart-contracts.md) — contract module breakdown
- [Database Schema](database-schema.md) — table and index design
- [Multi-Tenancy](../concepts/multi-tenancy.md) — how tenant isolation is enforced
- [Security Overview](../security/overview.md) — where security controls sit in the stack
