# Contributing

This is a personal portfolio project, but it is run like a team repo: issues, branches, PRs, CI, changelog.

## Prerequisites
- Node 20.x (`.nvmrc`), pnpm 9 via corepack (`corepack enable`), Docker Desktop, Git.

## Setup
```bash
pnpm i
cp .env.example .env
docker compose up -d
pnpm db:migrate && pnpm db:seed
pnpm dev
```

## Branches and commits
- Branch from `main`: `feat/<issue>-<slug>`, `fix/…`, `docs/…`, `chore/…`.
- Conventional Commits (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`, `perf:`, `ci:`). Enforced by commitlint.
- Commit body explains the why. Reference issues (`Refs #12`).

## Pull requests
- One issue per PR when possible. Fill the PR template completely.
- CI must be green: lint, typecheck, unit, API e2e, build, docker build, OpenAPI staleness.
- Squash merge. The PR title becomes the changelog entry (release-please), so write it for a reader.
- Docs are part of the change: README, `docs/architecture.md`, `docs/runbook.md`, `docs/data-model.md`, ADRs.

## Architecture decisions
- Any non-obvious choice gets an ADR in `docs/adr/` using `docs/adr/0000-template.md`. Add a line to `docs/decisions-log.md`.

## Tests
```bash
pnpm test            # vitest
pnpm test:api        # supertest against a disposable DB
pnpm test:e2e        # playwright (requires the stack running)
pnpm test:coverage   # thresholds enforced
```

## Releases
- release-please opens a release PR; merging it tags `vX.Y.Z` and updates `CHANGELOG.md`.
