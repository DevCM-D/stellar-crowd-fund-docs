# Commit Conventions

Commits follow the [Conventional Commits](https://www.conventionalcommits.org/) specification. This makes the changelog readable and enables automated tooling.

---

## Format

```
<type>(<scope>): <short summary>

[optional body]

[optional footer]
```

### Type

| Type | When to use |
|---|---|
| `feat` | New feature or capability |
| `fix` | Bug fix |
| `refactor` | Code change that neither adds a feature nor fixes a bug |
| `test` | Adding or updating tests |
| `docs` | Documentation only changes |
| `chore` | Tooling, build, dependency, or config changes |
| `perf` | Performance improvement |
| `security` | Security-related change (validation, rate limiting, auth hardening) |
| `ci` | CI/CD configuration changes |

### Scope

Optional. Names the area of the codebase the commit touches:

| Scope | Area |
|---|---|
| `api` | Express.js backend |
| `contracts` | Soroban Rust contracts |
| `mobile` | Expo React Native app |
| `frontend` | Next.js web app |
| `db` | Database schema or migrations |
| `queue` | BullMQ webhook queue |
| `auth` | Authentication or JWT handling |
| `docs` | Documentation (this repo) |

---

## Examples

Good commit messages:

```
feat(api): add cursor-based pagination to milestone list endpoint
fix(api): cap leaderboard results at 50 per page
security(api): reject HTTP webhook URLs at subscribe time
test(api): add validation tests for webhookSubscribeRules
refactor(mobile): replace console.error with structured logger in useEscrows
perf(db): add covering indexes on Dispute table for date-range queries
chore(contracts): bump soroban-sdk to 21.7.1
docs: add cursor pagination reference to api docs
```

Bad commit messages (avoid):

```
fix stuff
WIP
update
changes
```

---

## Summary Line Rules

- Maximum 72 characters
- Use the imperative mood: "add", not "added" or "adds"
- No period at the end
- Start with lowercase after the type prefix

---

## Body

Use the body to explain WHY the change was made if the summary line is not enough. The body is not a description of WHAT changed (that's visible in the diff).

```
fix(api): cap webhook delivery limit at 100

Default `limit` was unbounded — a client could request thousands of
delivery records in a single query. Capped at 100 to match the pattern
used by other list endpoints.
```

---

## Breaking Changes

Mark breaking changes in the footer:

```
feat(api): remove offset pagination from milestone list

BREAKING CHANGE: the milestone list endpoint now only accepts cursor
pagination parameters. Clients using `page` and `limit` must migrate
to `cursor` and `limit`.
```

---

## Commit Granularity

Prefer small, focused commits over large omnibus commits. One logical change per commit:

- A bug fix and its test in the same commit
- A new feature and its documentation in the same commit
- A refactor isolated from unrelated changes

If you find yourself writing "and" in the summary, consider splitting.

---

## Related

- [Contributing Overview](overview.md) — contribution workflow
- [Development Setup](development-setup.md) — local setup
- [Testing Guide](testing-guide.md) — what tests to include with your commit
