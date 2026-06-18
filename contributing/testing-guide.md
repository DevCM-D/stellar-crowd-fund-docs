# Testing Guide

The platform has 425+ backend tests covering controllers, middleware, queue behaviour, and utility functions. All tests must pass before merging any pull request.

---

## Test Stack

| Layer | Framework |
|---|---|
| Backend unit/integration tests | Jest + Supertest |
| Soroban contract tests | Rust `#[test]` with `soroban-sdk/testutils` |
| Mobile component tests | Jest + React Native Testing Library |

---

## Running Backend Tests

```bash
cd backend
npm test
```

Runs the full suite. Output shows pass/fail per test file with coverage summary.

### Run a single file

```bash
npm test -- --testPathPattern=webhookController
```

### Run with coverage report

```bash
npm test -- --coverage
```

Coverage report appears in `backend/coverage/`. The HTML report at `coverage/lcov-report/index.html` shows line-level coverage.

---

## Test Organization

Tests live alongside the source files they test, in `__tests__` directories:

```
backend/
├── api/
│   ├── controllers/
│   │   ├── webhookController.js
│   │   └── __tests__/
│   │       └── webhookController.test.js
│   ├── middleware/
│   │   ├── validation.js
│   │   └── __tests__/
│   │       └── validation.test.js
├── lib/
│   ├── pagination.js
│   └── __tests__/
│       └── pagination.test.js
```

---

## Writing a New Test

New tests should follow these conventions:

### Controller test

```javascript
import request from 'supertest';
import app from '../../app.js';
import { prismaMock } from '../../__mocks__/prisma.js';

describe('GET /api/escrows', () => {
  it('returns 401 when no auth token provided', async () => {
    const res = await request(app).get('/api/escrows');
    expect(res.status).toBe(401);
  });

  it('returns escrows scoped to tenant', async () => {
    prismaMock.escrow.findMany.mockResolvedValue([{ id: '1', status: 'Active' }]);
    const res = await request(app)
      .get('/api/escrows')
      .set('Authorization', 'Bearer valid-test-token');
    expect(res.status).toBe(200);
    expect(res.body.data).toHaveLength(1);
  });
});
```

### Middleware test

```javascript
import { validateRequest } from '../validation.js';

describe('webhookSubscribeRules', () => {
  it('rejects HTTP URLs', async () => {
    // ...
  });

  it('rejects empty eventTypes array', async () => {
    // ...
  });
});
```

### Utility test

```javascript
import { parseCursorPagination, buildCursorResponse } from '../pagination.js';

describe('parseCursorPagination', () => {
  it('defaults to limit 20 when none provided', () => {
    expect(parseCursorPagination({}).take).toBe(20);
  });

  it('caps limit at MAX_LIMIT', () => {
    expect(parseCursorPagination({ limit: '9999' }).take).toBeLessThanOrEqual(100);
  });
});
```

---

## What to Test

For each new controller:

- [ ] `401` when no auth token
- [ ] `403` when auth token is for the wrong role
- [ ] `400` for each invalid input case (missing fields, wrong types, out-of-range values)
- [ ] `404` when the resource doesn't exist
- [ ] `200`/`201` happy path with correct response shape
- [ ] Tenant scoping (results from tenant A do not appear for tenant B)

For each new middleware rule:

- [ ] Accepts valid input
- [ ] Rejects each class of invalid input with the correct error
- [ ] Sanitizes input correctly (control characters stripped, values clamped)

For each new utility function:

- [ ] Default values when no input provided
- [ ] Edge cases: zero, negative, null, undefined, max boundary
- [ ] Correct output shape

---

## Contract Tests (Rust)

```bash
cd contracts/escrow
cargo test
```

Test files live in `src/tests/` and use `soroban-sdk/testutils` to set up a test ledger:

```rust
#[test]
fn test_create_escrow_locks_funds() {
    let env = Env::default();
    let contract_id = env.register_contract(None, EscrowContract);
    let client = EscrowContractClient::new(&env, &contract_id);

    // ... set up addresses, token, milestones
    let escrow_id = client.create_escrow(&client_addr, &freelancer_addr, &token, &amount, &milestones);

    assert_eq!(escrow_id, 1u64);
    // ... assert token balances
}
```

---

## Pre-Push Hook

The Husky pre-push hook runs `npm test` in the `backend/` directory before every push. If any test fails, the push is blocked. This prevents broken code from reaching the remote.

Do not bypass the hook with `--no-verify`. Fix the test instead.

---

## Related

- [Development Setup](development-setup.md) — getting the test environment running
- [Commit Conventions](commit-conventions.md) — when to include tests in a commit
