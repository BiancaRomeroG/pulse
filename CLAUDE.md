# CLAUDE.md — rules for agents working in `pulse`

You are a worker on the Bianca Romero portfolio program. The program plan, program-level ADRs and this repo's spec live in `D:\Repositorio\portfolio-program` (also summarised in `docs/spec.md` here). Read them before changing anything.

## Non-negotiables
1. Never ask the owner. Decide, write an ADR in `docs/adr/`, continue.
2. Everything in TypeScript. No .NET, Java, Python. Shell scripts must run in Git Bash and Linux.
3. No paid services, no credit card, no secrets in git. `.env.example` is always complete.
4. English for code, commits, docs, default UI locale. Spanish only in README "Español".
5. Every change goes through a PR with the template fully filled, CI green, squash merge, branch deleted.
6. Docs change in the same PR as code. New decision = new ADR + line in `docs/decisions-log.md`.
7. OpenAPI (`docs/api/openapi.json`) is regenerated on build and committed; CI fails if stale.
8. Verify before claiming: run the tests, open the URL, paste the evidence in the PR.

## Workflow per task
1. Issue exists with the spec task id in the title (`[P1-08] Ledger module`). If not, create it with `gh issue create` and labels `phase:N`, `area:*`.
2. `git switch -c feat/<issue>-<slug>` from `main`.
3. Implement with tests. Conventional commits; the body says why.
4. `pnpm turbo lint typecheck test build` locally.
5. `gh pr create --fill` then edit the body with the template. `Closes #<issue>`.
6. Wait for CI (`gh pr checks --watch`). Fix until green.
7. `gh pr merge --squash --delete-branch`.
8. `git switch main && git pull`.

## Conventions
- Layout per ADR-0003: `apps/*`, `packages/*`, `infra/*`, `docs/*`.
- Validation with zod at every boundary. Env parsed once; crash on invalid.
- Errors are RFC 9457 Problem Details. Pagination is cursor-based. Money is integer minor units.
- Tests: Vitest unit (>= 70% on domain), Supertest API e2e, Playwright smoke. Property tests with fast-check where the spec lists invariants.
- Logging: pino, structured, request id. No `console.log` in app code.
- Comments explain why. Public functions in `packages/shared` have TSDoc.
- Design tokens and components per `portfolio-program/specs/00-shared-domain.md` section 4.

## Definition of done for a PR
- [ ] Issue linked and closed by the PR
- [ ] Tests added or updated; summary pasted in PR
- [ ] Docs updated (README / architecture / runbook / data-model / ADR as applicable)
- [ ] OpenAPI regenerated if API changed
- [ ] Screenshots or GIF for UI changes
- [ ] No secrets, no `any`, no skipped tests without an issue link

## End of phase
Open an issue `Phase N report` with shipped PRs, deferred items, ADRs, metrics (coverage, test counts, Lighthouse), known issues, and the next-phase plan. Update `docs/BLOCKED.md` if anything needs the owner.

## Commands
```bash
pnpm i                      # install
docker compose up -d        # full local stack
pnpm dev                    # all apps in dev mode
pnpm turbo lint typecheck test build
pnpm test:e2e               # playwright
pnpm db:migrate && pnpm db:seed
pnpm openapi:export         # regenerate docs/api/openapi.json
```
