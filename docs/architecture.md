# Architecture — {{PROJECT_NAME}}

## 1. Context (C4 level 1)

```mermaid
C4Context
  title {{PROJECT_NAME}} in Nimbus Pay
  Person(op, "Operator", "Internal user")
  System(sys, "{{PROJECT_NAME}}", "{{ONE_LINE}}")
  System_Ext(ext1, "{{EXTERNAL_1}}", "{{WHAT}}")
  Rel(op, sys, "Uses", "HTTPS")
  Rel(sys, ext1, "{{HOW}}")
```

## 2. Containers (C4 level 2)

```mermaid
C4Container
  title Containers
  Container(web, "web", "Next.js 16", "Dashboard")
  Container(api, "api", "NestJS 11", "REST + {{SSE/WS}}")
  ContainerDb(db, "postgres", "PostgreSQL 16", "{{SCHEMAS}}")
  Rel(web, api, "JSON/HTTPS")
  Rel(api, db, "Prisma")
```

## 3. Key flows

### 3.1 {{FLOW_NAME}}
```mermaid
sequenceDiagram
  participant W as web
  participant A as api
  participant D as db
  W->>A: POST /v1/...
  A->>D: tx: ...
  A-->>W: 201
```

## 4. Where things live

| Concept | Path |
|---------|------|
| Domain rules | `apps/api/src/<module>/domain/` |
| HTTP layer | `apps/api/src/<module>/http/` |
| Persistence | `apps/api/prisma/`, `apps/api/src/<module>/infra/` |
| Contracts | `packages/shared/src/` |
| UI components | `apps/web/src/components/ui/` |

## 5. Cross-cutting
- Validation, errors, pagination, idempotency: per Spec 00 section 5.
- Logging and tracing: {{DETAILS}}
- Security: {{AUTH + SIGNATURES}}
- Configuration: `packages/shared/src/env.ts`

## 6. Deployment

```mermaid
flowchart LR
  V[Vercel: web] --> R[Render: api]
  R --> S[(Supabase/Neon: Postgres)]
```

## 7. Known limitations
- {{LIMITATION}}
