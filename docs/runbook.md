# Runbook — {{PROJECT_NAME}}

## Run locally (Docker)
```bash
cp .env.example .env
docker compose up -d --wait
```
URLs: {{LIST}}

## Run locally (dev mode)
```bash
corepack enable && pnpm i
docker compose up -d postgres {{OTHER_DEPS}}
pnpm db:migrate && pnpm db:seed
pnpm dev
```

## Environment variables
| Variable | Required | Example | Notes |
|----------|----------|---------|-------|
| DATABASE_URL | yes | postgresql://app:app@localhost:5432/app | |
| {{VAR}} | | | |

## Seed and reset
```bash
pnpm db:seed          # deterministic, seed "nimbus-2026"
pnpm db:reset         # drop, migrate, seed
```

## Common failures
| Symptom | Cause | Fix |
|---------|-------|-----|
| `P1001: Can't reach database` | postgres not up | `docker compose up -d postgres` |
| {{SYMPTOM}} | | |

## Deploy
1. {{STEP}}
2. Verify: `curl {{URL}}/health`

## Rollback
- Vercel: promote previous deployment. Render: redeploy previous commit. DB: migrations are forward-only; write a compensating migration.

## Operational checks
- `GET /ready` returns 200 with all dependencies `up`.
- {{DASHBOARD_OR_LOGS}}
