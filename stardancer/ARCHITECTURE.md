# ARCHITECTURE.md — STΛRDANCΞR

Version 0.1.0 · scale tier **large** · build_depth comprehensive · deploy: local compose behind Caddy TLS (ADR-008)

STARDANCER keeps a live, multi-tenant **System Graph** (entities, relationships, observations, events, vulnerabilities, changes) and runs a deterministic reasoning layer, the **Mirror Brain Ψ**, over it. That layer produces predictions, investigations, attack paths and governed action proposals. Every conclusion and decision is written to a hash-chained **evidence ledger**.

## 1. Layers

| Layer | Package / path | Responsibility |
|---|---|---|
| Edge | `deploy/Caddyfile` (caddy:2) | TLS termination (`tls internal`), `:80`→`:443` redirect, HTTP/2, `/api/*` → api (round-robin, 2 replicas), else → web, request-size cap, header buffers |
| UI | `web/` Next.js 16 app router + Redux Toolkit / RTK Query | Landing (template port), auth screens, 11-destination app shell. It renders only; there is no business logic and no DB access |
| API | `server/src/app.ts` Express 5 | `/api/v1/*` REST, `/api/health`, `/metrics`, `/api/v1/stream` SSE, `/api/v1/openapi.json`. Its layers are routes → middleware (requestId, auth, csrf, rate limit, zod validate, tenant scope) → services |
| Services | `server/src/services/**` | All business rules and **all** Prisma access. Every query is org-scoped through `scoped(orgId)` helpers |
| Domain engines | `server/src/engine/**` | Pure functions: impact propagation, attack-path search (bounded BFS), risk scoring, Mirror Brain rules, drift diff, policy evaluation, hash chain. They do no I/O, so they are unit-testable |
| Workers | `server/src/worker.ts` (BullMQ) | Queues `ingest`, `proposals`, plus job schedulers (see §5). Same image as the API, different entrypoint |
| Data | PostgreSQL (Neon in prod, `postgres:17` local), Redis 7 | System of record; Redis holds the queues, pub/sub, rate-limit buckets and cache-aside |
| Observability | prom-client, pino, Prometheus, Grafana | RED metrics per route, queue depth/lag, ledger verify status, SSE connections |
| Shared | `shared/` | zod schemas + enums used by server and web (one contract) |

## 2. Data flow

1. **Ingest (W1):** a source holding an `ingest` API key sends `POST /api/v1/ingest` (batch ≤ 500, body ≤ 1 MB).
   - The request passes key auth, then the per-key rate limit, then the zod check.
   - The batch is dedup-claimed on the `observations` UNIQUE(org, source, idempotency_key) key, and the rows are persisted as `observations(status=received)`.
   - The batch is enqueued as an `ingest` job and the API returns `202 {accepted, duplicates}`.
   - **Backpressure:** if the queue is deeper than `INGEST_MAX_DEPTH`, the API returns 429 with `Retry-After`.
2. **Processing:** the worker upserts entities and relationships, then appends events, vulnerabilities and changes.
   - It marks the observation `processed` (or `rejected` with a reason).
   - It publishes `graph.changed` to the `org:<id>` Redis channel and invalidates the cache keys.
3. **Read:** the UI calls REST through RTK Query. The SSE stream pushes invalidation hints (`feed`, `graph`, `health`, `proposals`, `predictions`), and RTK refetches the affected tags. One exception is `/overview`: its first `GET /v1/overview` snapshot is read server-side, with the caller's cookies, over `API_INTERNAL_URL` and rendered in the HTML. RTK Query owns every later read and refresh (ADR-066). Server-side reads forward the caller's address as `X-Forwarded-For`, so per-IP limits key on the browser (ADR-070).
4. **Reason:** scheduled jobs update the graph:
   - `mirror-brain` creates and expires predictions.
   - `drift-sweep` marks stale entities and produces drift findings.
   - `health-rollup` writes `health_snapshots`.
   - `compliance-eval` writes `control_evaluations`.
5. **Govern (W9/W10/W15):** an AI agent (agent key) or an analyst creates a proposal. It is checked against policy, then validated by simulation (blast radius), then auto-approved when low-risk and policy allows it, otherwise sent to human approval. The executor then runs it (`assertExecutable` → `executeAction`) and records the outcome.
6. **Evidence (W11):** every state-changing service call appends an `evidence_records` row whose hash chains to the previous one. The rows are written in the same transaction under an org-level advisory lock. The hourly `ledger-verify` job recomputes the chain and writes a HMAC-signed `evidence_checkpoint`.
7. **Billing:** checkout runs `POST /api/v1/billing/checkout` (with an Idempotency-Key) through Stripe or the simulator, then the webhook.
   - The webhook path is `POST /api/v1/billing/webhook` (raw body), which calls `verifyWebhook` then `claimStripeEvent` (UNIQUE event_id).
   - The handler then updates the subscription, payment or refund in a transaction.

## 3. Agent / tool orchestration

STARDANCER **governs** external AI agents; it does not run an LLM itself (ADR-010).

- **Identity:** each agent has an org-scoped API key (`agent` kind), declared `capabilities[]` (action types it may propose) and `maxRisk`.
- **Context API:** `GET /api/v1/agent/context?entityId=` returns the entity, its 2-hop neighbourhood, criticality, open risks and applicable policies (W9). Reads are logged to the ledger.
- **Proposal API:** `POST /api/v1/agent/proposals` with an Idempotency-Key. The proposal moves through these states (§4 of SPEC):

  ```
  pending_policy → policy_rejected | pending_validation → validation_failed | pending_approval
  pending_approval → approved | rejected
  approved → executing → succeeded | failed
  ```

  Auto-approval applies only when policy `autoApproveMaxRisk ≥ risk` **and** the action is reversible.
- **Separation of duties:** an approver can't approve their own proposal, and agents can't approve at all. High-risk actions require an `owner` or `admin`.
- **Executor:** this is the only code path that changes graph state on an action's behalf. `assertExecutable(proposal)` re-checks the state, the approval and that the policy hash is unchanged before `executeAction()` runs.

## 4. Patterns

- **Tenant isolation:** `orgId` comes from the session or key, never from the request body. Services take `ctx {orgId, actor}` and every Prisma `where` includes `orgId`. A foreign id returns 404.
- **Idempotency:**
  - Write endpoints accept `Idempotency-Key`. Checkout, ingest and proposals persist it under UNIQUE constraints.
  - Webhooks dedup on `event_id`, and scheduled jobs claim `job_claims.key = <job>:<window>`.
- **Transactions:** a state change and its evidence append share one transaction.
- **Error contract:** RFC 9457 `application/problem+json` with `type, title, status, detail, code, requestId, errors[]`.
- **Pagination:** opaque base64url cursor `(createdAt,id)`, `limit` ≤ 100.
- **Caching:** Redis cache-aside for `overview:<org>` (30 s) and `graph:<org>:<view>` (60 s), invalidated on `graph.changed`.
- **Read split:** list and detail reads use `dbRead` (`DATABASE_REPLICA_URL` or primary); writes use `db`.
- **Config:** a zod env schema `server/src/config/env.ts` fails fast. Secrets come only from env (`.env` is gitignored, `.env.example` is committed).
- **Shutdown:** tini is PID 1. On SIGTERM the process stops accepting connections, drains SSE and HTTP for 10 s, closes workers, then closes Prisma and Redis.

## 5. Scheduled work (has_scheduled_work)

BullMQ job schedulers (`upsertJobScheduler`) run in the worker. Each run first claims `job_claims(key)` with key `<name>:<window-start-iso>`, so a duplicate tick or a second worker is a no-op.

| Trigger | Cadence | Effect | Effect test |
|---|---|---|---|
| `health-rollup` | every 1 min | writes `health_snapshots` (overall, infra/apps/security/network scores, counters, top risks) per org | `server/test/jobs.health-rollup.test.ts` |
| `drift-sweep` | every 5 min | marks entities not seen for `staleAfterMinutes` as `stale`, emits drift findings + feed items | `server/test/jobs.drift-sweep.test.ts` |
| `mirror-brain` | every 15 min | generates/refreshes predictions, expires predictions past `horizon` | `server/test/jobs.mirror-brain.test.ts` |
| `compliance-eval` | every 15 min | evaluates NIST CSF 2.0 controls into `control_evaluations` | `server/test/jobs.compliance-eval.test.ts` |
| `ledger-verify` | hourly | recomputes the hash chain since the last checkpoint and writes an HMAC checkpoint, or raises a critical integrity incident | `server/test/jobs.ledger-verify.test.ts` |

The `ingest` queue has concurrency 8, 5 attempts with exponential backoff, and a DLQ (`ingest-dlq`). The `proposals` queue has concurrency 4 and 3 attempts.
Each process (api-1, api-2, worker) opens a `pg` pool of 20 connections with error listeners on the pool and every client, so a database restart fails queries rather than the process (ADR-064).

## 6. Dependencies (pinned, ADR-022)

| Package | Version | Use |
|---|---|---|
| next | 16.4.0 | UI framework |
| react / react-dom | 19.3.0 | UI |
| @reduxjs/toolkit / react-redux | 2.13.0 / 9.3.0 | state + RTK Query |
| tailwindcss | 4.3.3 | utility CSS on top of tokens |
| express | 5.3.0 | API |
| prisma / @prisma/client / @prisma/adapter-pg / pg | 7.10.0 / 7.10.0 / 7.10.0 / 8.23.1 | ORM |
| bullmq | 6.3.12 | queues + schedulers |
| ioredis | 6.0.0 | Redis client |
| zod | 4.6.6 | validation (shared) |
| pino | 10.4.0 | JSON logging |
| prom-client | 15.1.3 | metrics |
| stripe | 23.0.0 | billing |
| argon2 | 0.45.1 | password hashing (argon2id) |
| helmet | 8.3.0 | security headers (api) |
| rate-limiter-flexible | 11.2.1 | Redis rate limiting |
| typescript | 6.0.3 | language |
| vitest / @playwright/test | 5.0.3 / 1.64.0 | tests |
| eslint / typescript-eslint | 10.12.0 / 8.71.1 | lint |
| autocannon | 8.0.0 | load test |

Runtime: Node 22 (`node:22-alpine`), PostgreSQL 17, Redis 7, Caddy 2, Prometheus, Grafana.

## 7. Decisions (with alternatives)

| Decision | Chosen | Alternatives rejected |
|---|---|---|
| Realtime | SSE + Redis pub/sub (ADR-009) | Socket.IO (sticky sessions), polling |
| Graph store | Postgres adjacency tables + bounded BFS in app (depth ≤ 6, frontier ≤ 5 000) | Neo4j (an extra datastore; Neon is mandated) |
| Graph rendering | bespoke SVG force layout, precomputed server-side positions, LOD | cytoscape.js (bundle size, theming against the template) |
| Reasoning | deterministic rules (ADR-010) | LLM (no provider, AI independence) |
| Audit log | hash-chained evidence ledger + HMAC checkpoints | plain audit table (not tamper-evident) |
| API placement | Express service behind Caddy (ADR-021) | Next.js route handlers |
| Auth | DB sessions + argon2id (ADR-007) | JWT (no revocation), third-party IdP (no credentials) |

## 8. Risks & threat model

The risk register (R1–R8) and the STRIDE threat model with trust boundaries B1–B7 are in RESEARCH.md §5. The architecture answers them like this:

- **B1 browser→edge:** TLS only, CSP with a nonce, `frame-ancestors 'none'`, and HSTS when `HSTS_ENABLED` (1 day on loopback, 1 year elsewhere; ADR-065, superseding ADR-023 for local deploys).
- **B2 edge→api:** the request body is capped, the API trusts only the Caddy hop's `X-Forwarded-For` (`trust proxy` = 1), and a per-IP limiter applies.
- **B3 api→db:**
  - Prisma uses parameterised queries; `$queryRawUnsafe` is banned (INV).
  - Org scoping lives in the services, which are the only Prisma importers (INV).
- **B4 machine keys → ingest/agent API:** keys are hashed and scoped by kind, with per-key limits, batch caps and queue backpressure.
- **B5 Stripe → webhook:** the signature is verified before the event is claimed (INV boundary-order), events are deduplicated, and the raw body is preserved.
- **B6 worker → graph:** the executor is the only actuator, `assertExecutable` runs before `executeAction` (INV), and agents are not approvers.
- **B7 ledger:** the ledger is append-only (no UPDATE/DELETE in the service), uses a SHA-256 chain with UNIQUE (org_id, seq), and is HMAC-checkpointed. The verify job alerts on any break.
- **Auth abuse:** login has per-account plus per-IP limiters ahead of `verifyPassword`, gives anti-enumeration responses (constant work), rotates the session on login, and checks a CSRF token on cookie-auth writes.
- **Data exposure:** pino redacts `authorization`, `cookie`, `password`, `token`, `apiKey`, `secret`, `email`, `lat`, `lng`. Viewers do not see API key metadata.

## 9. Invariants

Machine-checked by `scripts/invariant-lint.mjs` against `invariants.json`; run in CI and at every gate.

| ID | Rule | Check |
|---|---|---|
| INV-01 | Prisma is reached only through the service layer | forbidden `prisma\.\w+\.(find|create|update|delete|upsert|count|aggregate|groupBy)` outside `server/src/services/**`, `server/src/db/**`, `server/prisma/**`, tests |
| INV-02 | One Prisma client construction site | forbidden `new PrismaClient\(` outside `server/src/db/client.ts` |
| INV-03 | No unsafe raw SQL | forbidden `\$(queryRawUnsafe|executeRawUnsafe)` |
| INV-04 | Money is never floating point | forbidden `\b(FLOAT|REAL|DOUBLE PRECISION)\b` in migrations; `Float` in schema.prisma |
| INV-05 | Stripe webhooks are verified before they are claimed | boundary-order `verifyWebhook\(` before `claimStripeEvent\(` in the webhook route |
| INV-06 | Stripe events are processed once | unique `stripe_events(event_id)` |
| INV-07 | Checkout is idempotent | unique `checkout_requests(org_id, idempotency_key)` |
| INV-08 | Scheduled work runs once per window | unique `job_claims(key)` |
| INV-09 | Ingest is idempotent | unique `observations(org_id, source_id, idempotency_key)` |
| INV-10 | The evidence ledger is a single chain per org | unique `evidence_records(org_id, seq)` |
| INV-11 | Proposals are idempotent | unique `proposals(org_id, idempotency_key)` |
| INV-12 | Login is rate-limited before password verification | boundary-order `consumeLoginLimit\(` before `verifyPassword\(` in the auth service |
| INV-13 | Actions run only after the execution gate | boundary-order `assertExecutable\(` before `executeAction\(` in the executor |
| INV-14 | CSP never allows inline script | forbidden `unsafe-inline` on `script-src` |
| INV-15 | No tokens in web storage | forbidden `localStorage\.setItem|sessionStorage\.setItem` in `web/**` |
| INV-16 | No dynamic code evaluation | forbidden `\beval\(|new Function\(` in source |
| INV-17 | Colours come from tokens | forbidden hex literals in `web/src/**/*.tsx` outside the token stylesheet |
| INV-18 | Template tokens are canonical | required `--amber:#ffb547` in `web/src/styles/tokens.css` |
| INV-19 | Passwords use argon2id, never bcrypt | forbidden `\bbcrypt` |
| INV-20…INV-46 | Every declared status is written | required-pattern `status: '<literal>'` per proposal (10), prediction (4), payment (5), observation (1), investigation (2), subscription (3) status and entity `state` (2) in `server/src/**` |
| INV-47…INV-53 | Service floor files | required-file health route, `.env.example`, env schema, `scripts/web-baseline.mjs`, ci.yml, compose.yaml, deploy/Caddyfile |
| INV-54…INV-56 | Governance, web baseline, UI coverage | `governance`; `web-baseline` (FRONTEND-AUDIT.json); `ui-coverage` (`web/src/app/**` vs SPEC.md) |
| INV-M1 | Each scheduled trigger has its effect test (§5) | manual |

## 10. Deployment topology

```
browser ──https:8643──▶ caddy ──▶ web:3000 (Next standalone)
              :8280 ─301─┘   └──▶ api-1:4000, api-2:4000 (Express) ──▶ postgres:5432, redis:6379
                                   worker (BullMQ) ───────────────────▶ postgres, redis
prometheus:9090 (127.0.0.1:9390) scrapes api-1, api-2, worker /metrics ; grafana (127.0.0.1:3390)
```

All app containers run as non-root under tini with multi-stage images. A one-shot `migrate` service runs `prisma migrate deploy` before the API and worker start.
