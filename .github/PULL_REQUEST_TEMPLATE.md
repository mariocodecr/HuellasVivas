# Pull Request

## Description

<!-- Clear and concise description of what this PR does and why. -->

**What does this PR do?**

**Why is this change needed?**

Closes #<!-- issue number -->

---

## Type of Change

- [ ] `feat` — New feature
- [ ] `fix` — Bug fix
- [ ] `refactor` — Code restructuring (no behavior change)
- [ ] `docs` — Documentation only
- [ ] `test` — Tests only
- [ ] `chore` — Tooling, dependencies, config

---

## How Has This Been Tested?

- [ ] Unit tests added / updated
- [ ] Tested locally with `npm run start:dev`
- [ ] Tested endpoint manually via Swagger (`/api/docs`) or Postman

Steps to reproduce / test:

1.
2.
3.

---

## CI Checks

All of the following run automatically on every PR. Do not merge if any are failing.

| Check | What it validates |
|---|---|
| 🔐 Secret Scan | No API keys, tokens, or private keys in the diff |
| 🎨 Lint — Frontend | ESLint + TypeScript type check |
| 🎨 Lint — Backend | ESLint + TypeScript type check |
| 🧪 Tests — Backend | All unit tests pass, coverage thresholds met |
| 🏗️ Build — Frontend | Next.js compiles without errors |
| 🏗️ Build — Backend | NestJS compiles without errors |
| 📝 Commit Messages | Conventional Commits format enforced |

---

## Author Checklist

Before requesting a review, confirm:

- [ ] Commits follow [Conventional Commits](../CONTRIBUTING.md#commit-messages) (`feat(scope): description`)
- [ ] No `.env` files or real credentials committed
- [ ] All new endpoints have Swagger decorators (`@ApiTags`, `@ApiOperation`, `@ApiResponse`)
- [ ] New business logic has unit tests in a `.spec.ts` file
- [ ] Ran `npm run lint` and `npm run test` locally — both pass
- [ ] Response DTOs do not expose `passwordHash` or `encryptedSecretKey`
- [ ] Code follows the standards in [`backend/docs/standards/`](../backend/docs/standards/)

---

## Screenshots (if applicable)

<!-- Add screenshots for UI changes, or Swagger screenshots for new endpoints. -->
