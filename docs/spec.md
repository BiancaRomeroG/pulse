# Spec 04 — `pulse` (Worker W3)

Event-driven notification platform of Nimbus Pay. Tenants send CloudEvents; Pulse evaluates rules, renders templates and delivers through email, Web Push, in-app (WebSocket) and outbound webhooks, with outbox, retries, dead-letter queues, replay, full OpenTelemetry tracing, Kubernetes deployment and Terraform-managed AWS resources (LocalStack). No paid services.

Pitch: *"A multi-tenant notification platform built the way payment companies need it: Kafka-backed, exactly-once-effect delivery with outbox and idempotent consumers, retry topics and DLQ replay, traced end to end with OpenTelemetry, deployed to Kubernetes with Helm and provisioned with Terraform."*

## 1. Scope

In: tenant onboarding and API keys with HMAC request signing; `POST /v1/events` ingest with schema validation (CloudEvents + catalogue); rules engine (match on `type`, JSONPath conditions on `data`, throttling windows, dedup keys, quiet hours); templates (Handlebars-style with preview); channels: email (SMTP to Mailpit locally; Resend optional in hosted), Web Push (VAPID), in-app (Socket.io gateway + persisted inbox), outbound webhook (signed, retried); delivery pipeline with outbox, retry topics, DLQ, replay; tenant dashboard; demo event generator that emits LedgerFlow-shaped events; observability stack; Helm chart + kind; Terraform + LocalStack; k6 load test.

Out: SMS/WhatsApp, mobile SDKs, billing.

## 2. Architecture

```
apps/web          Next.js dashboard (Auth.js GitHub) + public demo generator + in-app inbox widget demo
apps/api          NestJS: tenants, api keys, events ingest (validates, writes outbox), rules, templates, notifications query, DLQ replay, WebSocket gateway
apps/relay        Outbox relay: polls outbox, publishes to EventBus with traceparent, marks sent (at-least-once)
apps/router       Consumer of events.raw: loads tenant rules, dedups, throttles, renders template, emits notifications.dispatch.<channel>
apps/dispatcher   Consumers per channel (email, push, inapp, webhook) with idempotency table, retry topics, DLQ
packages/shared   zod schemas, CloudEvents envelope + catalogue, EventBus port + adapters (kafka, redis-streams, memory), OTel bootstrap, env
infra/compose     redpanda, redpanda-console, postgres, mailpit, otel-collector, tempo, prometheus, loki, grafana, localstack, all apps
infra/helm/pulse  chart with values for kind
infra/k8s         kind config + scripts
infra/terraform   LocalStack provider: S3 (exports), SQS (DLQ archive), Secrets Manager (tenant secrets)
infra/grafana     dashboards + datasources provisioning
load/k6           scenarios
```

Topics: `events.raw`, `notifications.dispatch.email|push|inapp|webhook`, `*.retry.1m`, `*.retry.5m`, `*.retry.30m`, `*.dlq`, `notifications.status` (delivered/failed events, also written to outbox for external consumers). Partition key: `tenantId`.

Delivery semantics: producer side transactional outbox (ingest writes `Event` + `Outbox` in one transaction); relay at-least-once; every consumer has an `Inbox(eventId, consumer)` idempotency table checked within the same transaction as its side effect; retries go to delayed retry topics with a consumer that waits until `notBefore`; after 3 retries to DLQ; DLQ replay re-publishes with a `replayOf` extension attribute.

Tracing: `traceparent` created at ingest (or propagated from the caller), carried in Kafka headers, continued in every consumer, so one trace shows ingest → relay → router → dispatcher → SMTP/push/webhook.

## 3. Data model (Prisma)

- `Tenant(id, name, slug, createdAt)`, `ApiKey(id, tenantId, prefix, hash, scopes[], lastUsedAt, revokedAt?)`, `TenantSecret(tenantId, hmacSecret)` (mirrored to Secrets Manager via Terraform-provisioned store in local), `User(id, githubId, email, role)`, `Membership(userId, tenantId, role)`
- `Event(id ULID, tenantId, type, source, subject?, time, data json, traceparent, receivedAt, dedupKey?)`, `Outbox(id, aggregate, payload json, topic, key, createdAt, publishedAt?, attempts)`
- `Rule(id, tenantId, name, eventType, conditions json, channels[], templateId, throttle json {windowSec, max}, dedupKeyPath?, quietHours json?, enabled)`
- `Template(id, tenantId, name, channel, subject?, body, variablesSchema json)`
- `Subscriber(id, tenantId, externalId, email?, pushSubscriptions json[], webhookUrl?, webhookSecret?, preferences json)`
- `Notification(id, tenantId, eventId, ruleId, subscriberId, channel, status: QUEUED|SENT|DELIVERED|FAILED|DEAD|REPLAYED, attempts, lastError?, renderedSubject?, renderedBody, traceId, createdAt, updatedAt)`
- `DeliveryAttempt(id, notificationId, at, status, latencyMs, responseCode?, error?)`
- `Inbox(eventId, consumer, processedAt)` (idempotency), `InAppMessage(id, subscriberId, notificationId, readAt?)`
- `DlqEntry(id, topic, key, payload json, error, failedAt, replayedAt?)`

## 4. API (`/v1`)

Tenant (API key + HMAC):
- `POST /events` (single or batch up to 100; returns accepted ids; `Idempotency-Key` honoured), `GET /events/:id` (with notifications and trace id)
- `GET /notifications?status&channel&cursor`, `GET /notifications/:id`
- `POST /subscribers`, `PATCH /subscribers/:id`, `POST /subscribers/:id/push-subscriptions`
Dashboard (Auth.js session):
- `GET/POST/PATCH /tenants/:id/rules`, `/templates` (+ `POST /templates/:id/preview`), `/api-keys` (create returns plaintext once; revoke)
- `GET /dlq?topic`, `POST /dlq/:id/replay`, `POST /dlq/replay-all?topic`
- `GET /metrics/summary?window` (funnel: received → matched → queued → delivered/failed per channel)
- `WS /inapp` (Socket.io, subscriber token) for live in-app notifications
- `GET /health`, `GET /ready` (DB + broker)
Signing: `X-Pulse-Timestamp`, `X-Pulse-Signature: v1=hmac_sha256(secret, ts + "." + body)`, 5-minute window, nonce cache.

## 5. Web

`/login` (GitHub); `/` overview (funnel chart, per-channel success rate, live event stream via WS, consumer lag from broker admin API); `/events` (table, detail with trace id link to Grafana Tempo when local, notification fan-out tree); `/rules` (builder: event type picker from catalogue, condition rows with JSONPath + operator + value, channels, template, throttle, test against a sample event); `/templates` (editor with live preview and variables from the catalogue schema); `/subscribers`; `/notifications` (filters, attempts timeline, resend); `/dlq` (entries, error, replay buttons); `/api-keys`; `/demo` (public: generator that emits LedgerFlow-shaped events at a chosen rate with a chaos slider, and an in-app inbox widget showing arrivals in real time; opens Mailpit link locally, shows "email preview" in hosted mode).

## 6. Infra deliverables

- `docker compose up`: everything with seeded tenant `nimbus-ops`, 3 rules, 4 templates, 10 subscribers, and a generator producing 1 event/s by default (toggle). Grafana at :3001 with provisioned dashboards: "Delivery funnel", "Consumer lag & throughput", "Traces by tenant".
- Helm chart: Deployments (api, relay, router, dispatcher x4 as one image with `ROLE` env), Services, Ingress (nginx), ConfigMap, Secret, HPA on dispatcher (CPU), PodDisruptionBudget, readiness/liveness probes; `values-kind.yaml`; `make kind-up` script installs Redpanda operator or a single-node Redpanda chart, Postgres (bitnami), Mailpit, the OTel stack (kube-prometheus-stack light), and the Pulse chart.
- Terraform: `provider "aws"` with LocalStack endpoints; modules `storage` (S3 bucket `pulse-exports` with lifecycle), `dlq-archive` (SQS queue + policy; an exporter job in `apps/dispatcher` ships DLQ entries older than 24h), `secrets` (Secrets Manager entries per tenant). `terraform fmt -check`, `validate`, `tflint` in CI; `make tf-apply-local`.
- k6: `ingest.js` (ramp to 200 rps for 2 min with signatures), `fanout.js`; results table in `docs/performance.md`.
- Hosted demo (Render + Neon + Upstash): `EVENT_BUS=redis-streams`, relay/router/dispatcher run as in-process workers inside the api service to fit one free instance; clearly labelled "lite transport" in the UI footer.

## 7. Tasks

### Phase 1
- [P3-01] Turborepo scaffold with all apps and packages; tooling; CI (includes `terraform validate`, `helm lint`); Dockerfiles (single worker image with `ROLE`); compose with every dependency.
- [P3-02] `packages/shared`: CloudEvents envelope + catalogue + fixtures; `EventBus` port; `MemoryEventBus`; `KafkaEventBus` (kafkajs) with headers; `RedisStreamsEventBus` (ioredis / Upstash REST); contract tests shared by all adapters (testcontainers for Redpanda).
- [P3-03] Prisma schema v1 + migration + seed (tenant, rules, templates, subscribers, 24h of events).
- [P3-04] NestJS base + HMAC API-key guard + Auth.js session guard; Swagger + export; health/ready.
- [P3-05] OTel bootstrap in `packages/shared/otel.ts` (auto-instrumentations, pino transport to Loki, Kafka header propagation helpers); otel-collector config; Grafana provisioning with datasources.
- [P3-06] Web shell (Auth.js GitHub), tokens, ui components, `/events` list.
- [P3-07] Docs skeleton, ADR-0001 versions, architecture.md v1 (C4 + delivery sequence), runbook v1.

### Phase 2
- [P3-08] Ingest endpoint: validation against catalogue, dedup, transactional outbox, batch.
- [P3-09] Relay app with at-least-once publish, backoff, metrics.
- [P3-10] Router: rules evaluation (JSONPath via `jsonpath-plus`), throttle windows (Postgres or Redis), dedup, quiet hours, template rendering (Handlebars), fan-out to channel topics. Unit tests with fast-check on throttle windows.
- [P3-11] Dispatcher framework: consumer base class with `Inbox` idempotency, retry topics with `notBefore`, DLQ, `DeliveryAttempt` logging.
- [P3-12] Email channel (nodemailer to Mailpit; Resend adapter behind interface).
- [P3-13] Web Push channel (`web-push`, VAPID keys generated by a script, subscription management).
- [P3-14] In-app channel: Socket.io gateway, rooms per subscriber, persisted inbox, read receipts.
- [P3-15] Outbound webhook channel: signed, retries, circuit breaker per endpoint.
- [P3-16] DLQ replay endpoints + `notifications.status` events to outbox.
- [P3-17] Broker admin: consumer lag query for the overview.
- [P3-18] End-to-end trace verified in Tempo (document a screenshot); Grafana dashboards JSON committed.
- [P3-19] API e2e: ingest → delivered on all four channels against compose; chaos test: kill dispatcher mid-batch → no duplicates, no loss (prove via `Inbox` + attempt counts). data-model.md; ADRs: outbox vs CDC, retry topics vs delayed queue, idempotency scope, transport abstraction.

### Phase 3
- [P3-20] Overview with funnel, lag, live stream.
- [P3-21] Events + notification fan-out tree + trace link.
- [P3-22] Rules builder with test-against-sample.
- [P3-23] Templates editor with preview.
- [P3-24] Subscribers, notifications with attempts timeline, resend.
- [P3-25] DLQ page with replay.
- [P3-26] API keys page (plaintext once, copy, revoke).
- [P3-27] Public `/demo` generator + in-app widget.
- [P3-28] Playwright: login, create rule, emit event from demo, see email in Mailpit API, see in-app arrival, force failure, replay from DLQ. Screenshots to `docs/demo.md`.

### Phase 4
- [P3-29] Helm chart + kind script; `docs/kubernetes.md` with commands and screenshots of pods and HPA.
- [P3-30] Terraform + LocalStack modules; CI validation; `docs/terraform.md`.
- [P3-31] Hosted deploy: Neon, Upstash, Render single service with in-process workers, Vercel web. Owner supplies Neon/Upstash/GitHub OAuth values (BLOCKED.md otherwise).
- [P3-32] k6 runs + `docs/performance.md`; README with GIF (emit → fan-out → fail → DLQ → replay → trace), badges, Spanish section; `docs/case-study.md`; v1.0.0.

### Phase 5
- [P3-33] Schema registry-lite: versioned catalogue with compatibility check in CI (breaking change detection).
- [P3-34] Multi-region style failover demo: two Redpanda brokers in kind, partition reassignment doc.
- [P3-35] Tenant-level rate limits and quotas; usage dashboard.
- [P3-36] Chaos toolkit script (kill pods randomly in kind) with a recorded run.

## 8. Definition of done (Gate 4)
`docker compose up` shows an event delivered on four channels with one trace in Tempo; DLQ replay works; kind deployment documented with screenshots; Terraform applies to LocalStack in CI; hosted demo works in lite mode; coverage >= 70% on `packages/shared` and router; docs complete; GIF in README.
