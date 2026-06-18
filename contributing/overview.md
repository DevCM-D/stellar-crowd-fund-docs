# Contributing Overview

Stellar Crowd Fund Escrow is open to contributions. This section explains the contribution workflow, the standards expected, and how to get started.

---

## What We're Looking For

The most valuable contributions are:

- **Bug fixes** — especially regressions in the backend API or contract behaviour
- **Test coverage** — increasing coverage of edge cases in the 425+ existing backend tests
- **Documentation improvements** — corrections, examples, clarifications
- **Performance improvements** — query optimisations, cache improvements, mobile bundle size
- **Security improvements** — new input validation, rate limiting, or auth hardening

Feature additions are welcome if they align with the project roadmap. Open an issue to discuss before implementing a large feature — it avoids wasted effort.

---

## Contribution Workflow

1. **Fork** the repository and create a branch from `develop`
2. **Read** the [Development Setup](development-setup.md) guide
3. **Make** your changes
4. **Test** — run the full test suite before opening a PR
5. **Commit** — follow the [Commit Conventions](commit-conventions.md)
6. **Open a pull request** targeting `develop`

All pull requests are reviewed before merging. Reviews focus on correctness, test coverage, and whether the change matches the stated intent.

---

## Issue Labels

| Label | Meaning |
|---|---|
| `good first issue` | Suitable for contributors new to the codebase |
| `bug` | Confirmed defect |
| `enhancement` | Agreed addition or improvement |
| `needs-investigation` | Reported problem, root cause unclear |
| `security` | Security-related issue (please report privately first) |
| `contracts` | Changes to Soroban Rust contracts |
| `api` | Changes to the Express.js backend |
| `mobile` | Changes to the Expo React Native app |
| `docs` | Documentation-only changes |

---

## Reporting Security Vulnerabilities

Do not open a public issue for security vulnerabilities. Report them privately by emailing the maintainers. Include:

- Description of the vulnerability
- Steps to reproduce
- Potential impact
- (Optional) suggested fix

Responsible disclosure is appreciated and acknowledged in the changelog.

---

## Code of Conduct

Be direct, constructive, and respectful. Debate ideas; do not attack people. If you see behaviour that falls below this standard, report it to the maintainers.

---

## Related

- [Development Setup](development-setup.md) — getting the local environment running
- [Testing Guide](testing-guide.md) — how to write and run tests
- [Commit Conventions](commit-conventions.md) — how to write commit messages
