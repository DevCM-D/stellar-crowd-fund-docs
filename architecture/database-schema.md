# Database Schema

The platform uses PostgreSQL with Prisma ORM. The schema mirrors on-chain escrow and reputation state for fast querying, stores off-chain data (evidence metadata, webhook subscriptions), and supports full multi-tenancy.

---

## Core Tables

### Tenant

Every piece of data is owned by a tenant. The `Tenant` table is the root of the multi-tenancy tree.

```prisma
model Tenant {
  id        String   @id @default(cuid())
  slug      String   @unique            // used in cache keys and log prefixes
  name      String
  createdAt DateTime @default(now())

  escrows      Escrow[]
  disputes     Dispute[]
  webhookSubs  WebhookSubscription[]
  reputation   ReputationRecord[]
}
```

---

### Escrow

Off-chain mirror of on-chain escrow state. Kept in sync by the API server on every contract interaction.

```prisma
model Escrow {
  id                String     @id @default(cuid())
  onChainId         BigInt?                        // contract escrow_id once deployed
  tenantId          String
  clientAddress     String
  freelancerAddress String
  arbiterAddress    String?
  tokenAddress      String
  totalAmount       Decimal    @db.Decimal(38, 0)
  remainingBalance  Decimal    @db.Decimal(38, 0)
  status            EscrowStatus  @default(Active)
  briefHash         String?                        // IPFS CID of project brief
  deadline          DateTime?
  createdAt         DateTime   @default(now())
  updatedAt         DateTime   @updatedAt

  tenant     Tenant      @relation(fields: [tenantId], references: [id])
  milestones Milestone[]
  dispute    Dispute?
  webhooks   WebhookDelivery[]

  @@index([tenantId, status])
  @@index([tenantId, status, createdAt(sort: Desc)])
  @@index([tenantId, clientAddress])
  @@index([tenantId, freelancerAddress])
}

enum EscrowStatus {
  Active
  Disputed
  Completed
  Cancelled
}
```

---

### Milestone

Each row represents one milestone within an escrow.

```prisma
model Milestone {
  id               String          @id @default(cuid())
  escrowId         String
  milestoneIndex   Int                                   // 0-based
  title            String
  amount           Decimal         @db.Decimal(38, 0)
  descriptionHash  String?                               // IPFS CID
  status           MilestoneStatus @default(Pending)
  submissionHash   String?                               // IPFS CID of deliverable
  submittedAt      DateTime?
  resolvedAt       DateTime?

  escrow Escrow @relation(fields: [escrowId], references: [id])

  @@unique([escrowId, milestoneIndex])
  @@index([escrowId, status])
}

enum MilestoneStatus {
  Pending
  Submitted
  Approved
  Rejected
}
```

---

### Dispute

One dispute per escrow. Created when `raise_dispute` is called on-chain; updated when resolved.

```prisma
model Dispute {
  id                 Int      @id @default(autoincrement())
  escrowId           String   @unique
  tenantId           String
  raisedByAddress    String
  raisedAt           DateTime @default(now())
  resolvedAt         DateTime?
  clientAmount       Decimal? @db.Decimal(38, 0)
  freelancerAmount   Decimal? @db.Decimal(38, 0)
  resolvedBy         String?
  resolution         String?  @db.Text
  resolutionType     ResolutionType?
  autoResolved       Boolean  @default(false)

  escrow   Escrow          @relation(fields: [escrowId], references: [id])
  tenant   Tenant          @relation(fields: [tenantId], references: [id])
  evidence DisputeEvidence[]
  appeals  DisputeAppeal[]

  @@index([tenantId, resolvedAt, raisedAt(sort: Desc)])
  @@index([tenantId, raisedByAddress, raisedAt(sort: Desc)])
}

enum ResolutionType {
  MANUAL
  AUTO
  ESCALATED
}
```

### DisputeEvidence

Evidence files uploaded by either party during an active dispute.

```prisma
model DisputeEvidence {
  id           Int      @id @default(autoincrement())
  disputeId    Int
  submittedBy  String                       // Stellar address
  submittedAt  DateTime @default(now())
  filename     String
  evidenceType String                       // screenshot, document, video, code, other
  ipfsCid      String                       // IPFS content identifier
  thumbnailCid String?                      // IPFS CID of generated thumbnail
  scanStatus   String   @default("pending") // pending, clean, flagged

  dispute Dispute @relation(fields: [disputeId], references: [id])

  @@index([disputeId])
}
```

### DisputeAppeal

```prisma
model DisputeAppeal {
  id          Int      @id @default(autoincrement())
  disputeId   Int
  filedBy     String
  filedAt     DateTime @default(now())
  reason      String   @db.Text
  resolvedAt  DateTime?
  outcome     String?

  dispute Dispute @relation(fields: [disputeId], references: [id])
}
```

---

### ReputationRecord

Aggregated reputation for each Stellar address, per-tenant.

```prisma
model ReputationRecord {
  id               String   @id @default(cuid())
  tenantId         String
  address          String
  totalScore       Int      @default(0)
  completedEscrows Int      @default(0)
  disputedEscrows  Int      @default(0)
  disputesWon      Int      @default(0)
  totalVolume      Decimal  @db.Decimal(38, 0) @default(0)
  lastUpdated      DateTime @updatedAt

  tenant Tenant           @relation(fields: [tenantId], references: [id])
  events ReputationEvent[]

  @@unique([tenantId, address])
  @@index([tenantId, totalScore(sort: Desc)])
}
```

### ReputationEvent

Individual events that contribute to the reputation score.

```prisma
model ReputationEvent {
  id           Int      @id @default(autoincrement())
  tenantId     String
  address      String
  eventType    String
  escrowId     String?
  disputeId    Int?
  scoreDelta   Int
  createdAt    DateTime @default(now())

  record ReputationRecord @relation(fields: [tenantId, address], references: [tenantId, address])

  @@index([tenantId, address, createdAt(sort: Desc)])
}
```

---

### WebhookSubscription

```prisma
model WebhookSubscription {
  id         String   @id @default(cuid())
  tenantId   String
  address    String                  // subscribing user's Stellar address
  url        String                  // HTTPS endpoint
  secret     String                  // HMAC signing secret
  eventTypes String[]
  active     Boolean  @default(true)
  createdAt  DateTime @default(now())

  tenant    Tenant            @relation(fields: [tenantId], references: [id])
  deliveries WebhookDelivery[]

  @@index([tenantId, address])
}
```

### WebhookDelivery

```prisma
model WebhookDelivery {
  id             String   @id @default(cuid())
  subscriptionId String
  escrowId       String?
  eventType      String
  payload        Json
  status         String   @default("pending") // pending, delivered, failed
  attemptCount   Int      @default(0)
  lastStatusCode Int?
  lastResponse   String?  @db.Text
  createdAt      DateTime @default(now())
  lastAttemptAt  DateTime?

  subscription WebhookSubscription @relation(fields: [subscriptionId], references: [id])
  escrow       Escrow?             @relation(fields: [escrowId], references: [id])

  @@index([subscriptionId, createdAt(sort: Desc)])
  @@index([status, createdAt])
}
```

---

## Index Strategy

Indexes are designed around the query patterns the API actually uses:

| Table | Index | Query it serves |
|---|---|---|
| Escrow | `(tenantId, status)` | List escrows filtered by status |
| Escrow | `(tenantId, status, createdAt DESC)` | Sorted, filtered listing |
| Escrow | `(tenantId, clientAddress)` | Escrows for a specific client |
| Dispute | `(tenantId, resolvedAt, raisedAt DESC)` | Unresolved disputes sorted by age |
| Dispute | `(tenantId, raisedByAddress, raisedAt DESC)` | Disputes by a specific party |
| ReputationRecord | `(tenantId, totalScore DESC)` | Leaderboard ordering |
| ReputationEvent | `(tenantId, address, createdAt DESC)` | Event history for an address |

---

## Migrations

Prisma migrations live in `backend/database/migrations/`. Each migration is a timestamped SQL file generated by `prisma migrate dev`.

```bash
# Create a new migration
npx prisma migrate dev --name descriptive_name

# Apply to production
npx prisma migrate deploy

# View migration status
npx prisma migrate status
```

Never edit a migration file after it has been applied to any database. Create a new migration instead.

---

## Related

- [Architecture Overview](overview.md) — how the database fits in the full stack
- [Multi-Tenancy](../concepts/multi-tenancy.md) — how tenant scoping is enforced
- [Deployment → Database Migrations](../deployment/database-migrations.md) — applying migrations in production
