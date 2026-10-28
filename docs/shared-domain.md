# Spec 00 — Shared domain, contracts and design language

Applies to all repos. Workers copy what they need; nothing is imported across repos (ADR-0001, ADR-0005).

## 1. Nimbus Pay (fictional company)

Nimbus Pay is a LATAM payments company. Merchants accept card and QR payments; Nimbus settles funds to merchant balances, charges fees, and pays out. Internal teams rely on notifications and on-call tooling.

Fixed vocabulary (use these exact names in code, docs and UI):

| Term | Meaning |
|------|---------|
| Merchant | A business that accepts payments. Has one or more Accounts in the ledger. |
| Customer | End user paying a merchant. Only an opaque id in our systems. |
| Payment Intent | A request to collect an amount from a customer through a Provider. State machine below. |
| Provider | External acquirer/gateway. We have two simulated ones: `cardsim` (card-like) and `qrsim` (QR-like). |
| Settlement Batch | Daily set of captured payments a Provider pays to Nimbus. Source of reconciliation. |
| Reconciliation | Matching provider batch lines to ledger entries; discrepancies produce cases. |
| Tenant (pulse) | An internal team or merchant that sends events and receives notifications. |
| Incident | An operational problem tracked by on-call, with timeline and remediation. |
| Runbook | A document describing how to diagnose/fix a class of problems. |

Currencies: `BOB`, `USD`. Money is always integer minor units + ISO code. Never floats.

Payment Intent state machine (shared by ledgerflow and pulse event types):

```
created -> requires_action -> processing -> captured -> settled
   |             |               |            |-> refunded (partial/full)
   |             |               |-> failed
   |-> canceled  |-> expired
```

## 2. Event contracts (CloudEvents 1.0)

Envelope (zod, duplicated in each repo at `packages/shared/src/events/envelope.ts`):

```ts
{
  specversion: "1.0",
  id: string,            // ULID
  source: string,        // e.g. "nimbuspay/ledgerflow"
  type: string,          // reverse-DNS + version suffix, see catalogue
  time: string,          // ISO-8601
  subject?: string,      // aggregate id
  datacontenttype: "application/json",
  traceparent?: string,  // W3C trace context
  data: object           // typed per event
}
```

Catalogue v1 (emitted by ledgerflow, consumed by pulse, referenced by incident-copilot):

| type | data |
|------|------|
| `com.nimbuspay.ledger.payment.created.v1` | paymentIntentId, merchantId, amount{minor,currency}, provider |
| `com.nimbuspay.ledger.payment.captured.v1` | paymentIntentId, merchantId, amount, provider, providerRef |
| `com.nimbuspay.ledger.payment.failed.v1` | paymentIntentId, merchantId, reason, provider |
| `com.nimbuspay.ledger.payment.refunded.v1` | paymentIntentId, refundId, amount |
| `com.nimbuspay.ledger.settlement.batch.received.v1` | batchId, provider, lineCount, total |
| `com.nimbuspay.ledger.reconciliation.discrepancy.v1` | caseId, batchId, kind, amountDelta |
| `com.nimbuspay.pulse.notification.delivered.v1` | notificationId, channel, tenantId |
| `com.nimbuspay.pulse.notification.failed.v1` | notificationId, channel, tenantId, error, attempts |
| `com.nimbuspay.ops.incident.opened.v1` | incidentId, severity, service, title |

Each repo ships `packages/shared/src/events/catalogue.ts` with zod schemas and a `fixtures/` folder with one valid JSON sample per type (used by tests and simulators).

## 3. Seed data (identical across repos)

- 5 merchants: `mrc_cafe_sol`, `mrc_tienda_lu`, `mrc_farma_cruz`, `mrc_gym_andes`, `mrc_libros_rio`.
- 2 providers: `cardsim`, `qrsim`.
- 30 days of payment history, ~2,000 payment intents, realistic distribution (80% captured, 8% failed, 5% refunded, rest canceled/expired).
- Services for telemetry (pulse/incident-copilot): `ledger-api`, `provider-relay`, `pulse-ingest`, `pulse-dispatcher`, `copilot-api`.
- Seeds are deterministic (seeded PRNG, e.g. `seedrandom` with seed `nimbus-2026`).

## 4. Design language (web apps of P1–P3; the portfolio has its own spec but shares tokens)

The three project UIs share one look so they read as one company's internal tools, and they echo the portfolio's "Event Stream" identity.

- Theme: dark by default, light supported via `prefers-color-scheme` and a toggle.
- Tokens (Tailwind v4 `@theme`):
  - `--color-bg: #08090B`, `--color-surface: #0F1114`, `--color-surface-2: #15181D`, `--color-border: #23272E`
  - `--color-text: #E8EAED`, `--color-muted: #8A9099`
  - `--color-accent: #C6FF3F` (lime), `--color-accent-2: #8B7CFF` (violet, sparingly), `--color-ok: #3DDC97`, `--color-warn: #FFB347`, `--color-danger: #FF5C5C`
  - Light theme inverts bg/surface/text; accent stays lime but darker `#7FB800` for contrast.
- Type: `Geist Sans` body, `Geist Mono` for ids/amounts/logs, `Instrument Serif` italic only for display accents (portfolio) and empty-state headlines (apps).
- Radius 12px cards, 8px controls. 1px borders, no heavy shadows. Subtle grain overlay on page background (CSS, 3% opacity).
- Motion: Motion (`motion/react`). Page/section enter: fade + 12px rise, 400ms, `easeOut`. Lists stagger 40ms. Live data (events, ledger lines) slide in from the top with a lime flash that decays in 800ms. Respect `prefers-reduced-motion` (disable transforms, keep opacity).
- Components to build in each repo's `apps/web/src/components/ui` (copy, not share): `Button` (default, accent, ghost; magnetic hover on accent), `Card`, `Badge` (state colours), `DataTable` (sortable, sticky header, mono numerics), `JsonViewer` (collapsible, used by webhook/event inspectors), `Timeline`, `StateMachineViz` (small SVG showing current state highlighted), `LiveFeed` (auto-scrolling list for events/logs), `EmptyState`, `Toast`, `CommandPalette` (Cmd+K; nice to have).
- Each app has a top bar with the project name in mono, environment badge (`local` / `demo`), theme toggle, and a "View on GitHub" link.
- Accessibility: focus-visible rings in accent, AA contrast, all interactive elements keyboard reachable.

## 5. API conventions (NestJS)

- Prefix `/v1`. JSON only. Errors follow RFC 9457 Problem Details (`type`, `title`, `status`, `detail`, `instance`, `errors[]`).
- Pagination: cursor-based (`?cursor=&limit=`), response `{ data, nextCursor }`.
- Idempotency: `Idempotency-Key` header on all POSTs that create money movements or notifications; stored 24h; replay returns the original response with `Idempotent-Replayed: true`.
- Request id: `X-Request-Id` accepted or generated; echoed; in logs.
- Rate limit: `@nestjs/throttler`, 60 req/min default, 429 with `Retry-After`.
- Swagger: every DTO with examples; tags per module; served at `/docs`; exported to `docs/api/openapi.json` on build.
- Health: `/health` (liveness) and `/ready` (DB, broker, model loaded).

## 6. Quality bar (all repos)

- `pnpm turbo lint typecheck test build` green locally and on CI.
- Unit coverage >= 70% on `packages/shared` and domain modules; thresholds enforced in vitest config.
- Playwright smoke on CI (Chromium), full suite nightly.
- Lighthouse CI on the main page: performance >= 90, accessibility >= 95, best practices >= 95, SEO >= 90.
- `pnpm audit --prod` no high/critical.
- Docker images < 300 MB, non-root, `HEALTHCHECK`.
