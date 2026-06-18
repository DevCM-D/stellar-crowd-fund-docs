# Stellar Crowd Fund Escrow — Documentation

Welcome to the official documentation for **Stellar Crowd Fund Escrow**, a decentralised milestone-based escrow platform built on the Stellar blockchain using Soroban smart contracts.

> **Source code:** [DevCM-D/Stellar-Crowd-Fund-Escrow](https://github.com/DevCM-D/Stellar-Crowd-Fund-Escrow)

---

## Where to Start

| I want to… | Go to |
|---|---|
| Understand what this is and why it exists | [Getting Started → Overview](getting-started/overview.md) |
| Run the project locally in under 10 minutes | [Getting Started → Quickstart](getting-started/quickstart.md) |
| Understand how escrows and milestones work | [Concepts → Escrow Lifecycle](concepts/escrow-lifecycle.md) |
| See how the platform is structured | [Architecture → Overview](architecture/overview.md) |
| Integrate via the REST API | [API Reference → Authentication](api-reference/authentication.md) |
| Deploy to a server or cloud | [Deployment → Environment Variables](deployment/environment-variables.md) |
| Report a security issue or review security practices | [Security → Overview](security/overview.md) |
| Contribute code, tests, or docs | [Contributing → Overview](contributing/overview.md) |

---

## Documentation Map

```
docs/
├── getting-started/
│   ├── overview.md              What the project is and the problem it solves
│   ├── quickstart.md            Run the full stack locally in one session
│   ├── prerequisites.md         Everything you need installed before starting
│   └── local-sandbox.md         Run a local Stellar node with Docker
│
├── concepts/
│   ├── escrow-lifecycle.md      States, transitions, and who triggers them
│   ├── milestones.md            How milestone-based fund release works
│   ├── reputation-system.md     On-chain reputation events and scoring
│   ├── disputes.md              Dispute flow, evidence submission, and resolution
│   ├── multi-tenancy.md         How tenant isolation works across the stack
│   └── webhooks.md              Event subscription, delivery, and retry behaviour
│
├── architecture/
│   ├── overview.md              Four-layer architecture with decision rationale
│   ├── smart-contracts.md       Soroban contract internals and entry points
│   └── database-schema.md       Prisma models, relations, and index strategy
│
├── api-reference/
│   ├── authentication.md        Auth flow, token lifecycle, MFA
│   ├── escrows.md               Escrow list, get, broadcast endpoints
│   ├── disputes.md              Dispute list, detail, evidence upload
│   ├── reputation.md            Reputation lookup and leaderboard
│   ├── search.md                Full-text search and autocomplete
│   ├── webhooks.md              Subscription management and delivery log
│   ├── health.md                Liveness, readiness, and full health probes
│   └── pagination.md            Offset and cursor pagination reference
│
├── deployment/
│   ├── environment-variables.md Full environment variable reference
│   ├── preflight.md             Pre-deployment checker usage
│   ├── database-migrations.md   Prisma migration workflow
│   └── production-checklist.md  Steps to verify before going live
│
├── security/
│   ├── overview.md              Security model and defence-in-depth summary
│   ├── secrets.md               Secret generation, rotation, and storage
│   └── rate-limiting.md         Rate limit tiers and configuration
│
└── contributing/
    ├── overview.md              How to contribute — issues, PRs, reviews
    ├── development-setup.md     Full local dev environment from scratch
    ├── testing-guide.md         How to write and run backend tests
    └── commit-conventions.md    Branch naming, commit format, PR checklist
```

---

## Quick Navigation

**Getting Started**
- [Overview](getting-started/overview.md)
- [Quickstart](getting-started/quickstart.md)
- [Prerequisites](getting-started/prerequisites.md)
- [Local Sandbox](getting-started/local-sandbox.md)

**Concepts**
- [Escrow Lifecycle](concepts/escrow-lifecycle.md)
- [Milestones](concepts/milestones.md)
- [Reputation System](concepts/reputation-system.md)
- [Disputes](concepts/disputes.md)
- [Multi-Tenancy](concepts/multi-tenancy.md)
- [Webhooks](concepts/webhooks.md)

**Architecture**
- [Overview](architecture/overview.md)
- [Smart Contracts](architecture/smart-contracts.md)
- [Database Schema](architecture/database-schema.md)

**API Reference**
- [Authentication](api-reference/authentication.md)
- [Escrows](api-reference/escrows.md)
- [Disputes](api-reference/disputes.md)
- [Reputation](api-reference/reputation.md)
- [Search](api-reference/search.md)
- [Webhooks](api-reference/webhooks.md)
- [Health](api-reference/health.md)
- [Pagination](api-reference/pagination.md)

**Deployment**
- [Environment Variables](deployment/environment-variables.md)
- [Preflight Checker](deployment/preflight.md)
- [Database Migrations](deployment/database-migrations.md)
- [Production Checklist](deployment/production-checklist.md)

**Security**
- [Overview](security/overview.md)
- [Secrets Management](security/secrets.md)
- [Rate Limiting](security/rate-limiting.md)

**Contributing**
- [Overview](contributing/overview.md)
- [Development Setup](contributing/development-setup.md)
- [Testing Guide](contributing/testing-guide.md)
- [Commit Conventions](contributing/commit-conventions.md)

---

*This documentation is maintained in [DevCM-D/stellar-crowd-fund-docs](https://github.com/DevCM-D/stellar-crowd-fund-docs). To report a documentation issue or suggest an improvement, open an issue in that repository.*
