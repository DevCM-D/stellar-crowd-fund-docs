# Production Checklist

Run through this checklist before every production deployment.

---

## Pre-Deploy

- [ ] All environment variables are set (run `node scripts/preflight.js` and confirm all checks pass)
- [ ] JWT secrets are strong random values, not default/dev placeholders
- [ ] `NODE_ENV` is set to `production`
- [ ] `DATABASE_URL` points to the production database, not a dev or staging instance
- [ ] Database backup completed (`pg_dump $DATABASE_URL > backup.sql`)
- [ ] Redis is running and `REDIS_URL` is correct
- [ ] Elasticsearch is reachable (if enabled) or confirmed as expected to be disabled
- [ ] Soroban contract is deployed and `CONTRACT_ID` matches the deployed address
- [ ] `STELLAR_NETWORK` matches the deployed contract (both `mainnet` or both `testnet`)
- [ ] Git working tree is clean — no uncommitted local changes

## Database

- [ ] `npx prisma migrate deploy` completed successfully
- [ ] Migration status shows all migrations applied: `npx prisma migrate status`
- [ ] No pending migrations remain

## Application

- [ ] `npm run build` (or Docker build) completed without errors
- [ ] `node scripts/preflight.js` exits with code 0
- [ ] Server starts and responds to `GET /api/health` with `"status": "ok"`
- [ ] All dependencies show `"ok"` in the health response

## Monitoring

- [ ] Uptime monitor is pointing to `GET /api/health`
- [ ] Structured logs are flowing to your log aggregator
- [ ] Alerts are configured for 5xx error rate spikes
- [ ] BullMQ dashboard (or equivalent) is monitoring webhook delivery queue

## Security

- [ ] HTTPS is enforced on all ingress paths (no plain HTTP)
- [ ] CORS is configured correctly — only expected origins are allowed
- [ ] Rate limiting is active (test: hammer `/api/webhooks/subscribe` 11 times and confirm 429)
- [ ] Webhook signature verification is documented and communicated to subscribers

## Post-Deploy

- [ ] Smoke test: create an escrow, submit a milestone, approve it — end to end
- [ ] Check structured logs for unexpected errors in the first 5 minutes
- [ ] Verify webhook deliveries are being processed in the queue
- [ ] Rollback plan confirmed: previous Docker image tag or git commit is noted

---

## Rollback Plan

If a deployment fails after the migration step:

1. Stop the new server
2. Restore the database from pre-deploy backup (`psql $DATABASE_URL < backup.sql`)
3. Start the previous version of the application
4. Verify `GET /api/health` returns ok

If the migration itself failed (partial apply):

1. `npx prisma migrate status` — identify the failed migration
2. Manually fix the migration SQL or add a compensating migration
3. Re-run `npx prisma migrate deploy`

---

## Related

- [Environment Variables](environment-variables.md) — full env var reference
- [Preflight](preflight.md) — what the preflight checker validates
- [Database Migrations](database-migrations.md) — migration commands and safety rules
