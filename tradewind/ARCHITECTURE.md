# ARCHITECTURE.md — Tradewind

> Source of truth for **architecture**. Behavior lives in `SPEC.md`. Precedence: RESEARCH > ARCHITECTURE > SPEC.
> Build depth: **comprehensive** → threat model + alternatives-considered included per major decision.

## 1. System Overview

Tradewind is a milestone-escrow services marketplace. Money is held in escrow on the Stripe **platform
balance** and mirrored in an immutable, always-balanced **double-entry ledger** in PostgreSQL. Three
roles — **client**, **freelancer**, **admin** — operate over a strict escrow + dispute + payout state
machine. Stripe (Checkout + Connect Express, **test mode**) is the payment rail.

**Topology (single Docker Compose, `scale: small`):**

```
            HTTPS (reverse proxy / TLS)
                     │
        ┌────────────┴────────────┐
        │                         │
   ┌────▼─────┐             ┌─────▼──────┐        ┌──────────────┐
   │  web     │  REST /v1   │   api      │  SQL   │  postgres    │
   │ Next.js  │────────────▶│  Express 5 │───────▶│  (ledger,    │
   │ 16 (RSC) │◀────────────│  + Prisma 7│◀───────│  business)   │
   └────┬─────┘   JSON      └─────┬──────┘        └──────▲───────┘
        │                         │  Stripe SDK          │ poll
        │ browser → Stripe        │  (idempotent)        │
        ▼ hosted Checkout    ┌────▼─────┐          ┌─────┴──────┐
   ┌──────────┐  webhooks    │  Stripe  │          │  worker    │
   │  Stripe  │─────────────▶│  (test)  │          │ outbox +   │
   │  Checkout│  (signed)    └──────────┘          │ reconcile  │
   └──────────┘                                    └────────────┘
```

- **web** — Next.js 16 App Router (RSC + Route Handlers proxy where needed); Tailwind; React Context
  for client session/UI state. Renders all screens; never holds Stripe secret keys; payment entry is
  Stripe **hosted Checkout** (card data bypasses Tradewind entirely → PCI SAQ-A).
- **api** — Express 5 REST under `/v1/`. Owns all money logic, auth, authorization, the ledger, Stripe
  server calls, and webhook ingestion. Structured-JSON logging (pino). Sentry for errors.
- **worker** — single Node process: drains `pending_emails` (Resend) and runs the periodic
  **reconciliation** job (ledger Σ=0 + balance derivation vs Stripe). Polls Postgres; no broker.
- **postgres** — business + accounting data. Serializable transactions, CHECK constraints, row locks.

## 2. Data Flow (the money path)

1. **Fund (W3):** client clicks *Fund milestone* → api creates a Stripe Checkout Session
   (`mode:'payment'`, idempotency key = `fund:<milestoneId>`) → browser redirects to Stripe → on
   success Stripe sends `checkout.session.completed` / `payment_intent.succeeded` webhook → api
   **verifies signature**, dedupes `event.id`, then in one serializable tx: milestone → `funded`, and
   posts ledger legs **EXTERNAL → ESCROW_HELD** (Σ=0). The browser success redirect is **not** proof of
   payment (RESEARCH §3.9.3).
2. **Deliver (W4):** freelancer marks `delivered` (state only, no money).
3. **Approve / release (W5):** client approves → api in one serializable tx: lock the escrow account,
   assert held ≥ amount, post **ESCROW_HELD → FREELANCER_PAYABLE (amount−fee)** + **ESCROW_HELD →
   PLATFORM_FEE (fee)** (Σ=0), milestone → `released`. UNIQUE idempotency key `release:<milestoneId>`
   makes double-release impossible.
4. **Payout (W6):** freelancer requests payout → api locks `FREELANCER_PAYABLE`, asserts available ≥
   amount, posts **FREELANCER_PAYABLE → PAYOUT_CLEARING**, creates a Stripe **Transfer** to the
   connected account (idempotency key `payout:<payoutId>`), payout row → `requested`. On `payout.paid`
   webhook → post **PAYOUT_CLEARING → EXTERNAL**, payout → `paid`. On `payout.failed` → reverse
   **PAYOUT_CLEARING → FREELANCER_PAYABLE**, payout → `failed`.
5. **Dispute (W7):** client opens dispute on a `delivered`/`funded` milestone → admin resolves:
   **refund** (ESCROW_HELD → EXTERNAL via Stripe Refund), **release** (as W5), or **split**
   (ESCROW_HELD → FREELANCER_PAYABLE + PLATFORM_FEE + EXTERNAL refund portion). Every resolution writes
   an audit entry; admins cannot move funds without one.

Every state-mutating money action emits: a balanced ledger transaction, an `audit_logs` row, and
(where a party must be notified) a `pending_emails` outbox row — all in the **same DB transaction**.

## 3. Modules / Layers (api)

```
apps/api/src/
  index.ts                 # app bootstrap, middleware order, route mounting, graceful shutdown
  config/env.ts            # zod-validated env (fail fast on missing required vars)
  lib/
    prisma.ts              # singleton PrismaClient
    logger.ts              # pino structured JSON
    errors.ts              # AppError + E.* error codes (typed contract)
    money.ts               # integer-minor-unit helpers, fee split math
    stripe.ts              # Stripe SDK singleton (apiVersion pinned)
    sentry.ts              # Sentry init (no-op without DSN)
  middleware/
    auth.ts                # requireAuth (session lookup), attaches principal
    authorize.ts           # assertResourceAccess(row, principal) chokepoint; requireRole
    rateLimit.ts           # per-(ip,email)/per-user limiter (in-memory token bucket, small tier)
    csrf.ts                # Origin/CSRF guard for state-mutating routes
    error.ts               # central error handler → typed envelope
    requestId.ts           # request id + access log
  services/                # ALL scoped DB writes live here (INV-1)
    auth.service.ts        # register/login/logout/session, bcrypt, HIBP screen
    ledger.service.ts      # postTransaction(legs) — asserts Σ=0; balance(accountId)
    account.service.ts     # account resolution per user/escrow
    project.service.ts     # projects + milestones CRUD, state transitions
    escrow.service.ts      # fund/release/dispute orchestration (uses ledger + stripe)
    payout.service.ts      # payout request + state machine
    dispute.service.ts     # dispute open + admin resolve
    webhook.service.ts     # verify→dedupe→apply
    audit.service.ts       # append-only audit log writer
    outbox.service.ts      # enqueue pending_emails
    report.service.ts      # CSV/PDF ledger + statement export
  routes/                  # thin controllers → services; zod input validation
    auth.routes.ts  projects.routes.ts  milestones.routes.ts
    payouts.routes.ts  disputes.routes.ts  ledger.routes.ts
    connect.routes.ts  webhooks.routes.ts  health.routes.ts  reports.routes.ts
  validation/              # zod schemas (request bodies, params)
prisma/
  schema.prisma            # data model
  migrations/              # SQL migrations (reversible)
  seed.ts                  # reference data + demo client/freelancer/admin
scripts/
  invariant-lint.mjs       # invariants.json runner (all check types)
```

**Boundary rule (INV-1):** No route handler calls Prisma on a scoped table directly; every scoped read
/write goes through a `services/*` function. This centralizes tenant + object-level authorization and
makes a newly-added sub-resource unable to silently inherit only a tenant filter.

## 4. Data Layer (Prisma 7 + PostgreSQL)

**Money:** every amount is `BigInt amountMinor` + `currency CHAR(3)` (USD). Never `Float`/`Decimal`-without-scale.

Core tables (full field list in SPEC §3):
- `users` (role: CLIENT | FREELANCER | ADMIN; passwordHash; stripeConnectAccountId nullable)
- `sessions` (token hash, userId, expiresAt)
- `projects` (clientId owner)
- `milestones` (projectId; freelancerId; amountMinor; currency; status enum; idempotency anchors)
- `accounts` (type enum: EXTERNAL | ESCROW_HELD | FREELANCER_PAYABLE | PLATFORM_FEE | PAYOUT_CLEARING;
  ownerUserId nullable for system accounts; UNIQUE(type, ownerUserId, milestoneId))
- `ledger_transactions` (id, kind, idempotencyKey **UNIQUE**, createdAt) — immutable
- `ledger_entries` (transactionId, accountId, amountMinor signed, currency) — immutable, append-only;
  **per-transaction Σ(amountMinor) = 0** enforced by `ledger.service.postTransaction` + reconciliation
- `payouts` (freelancerId; amountMinor; status enum requested|paid|failed; stripeTransferId)
- `disputes` (milestoneId; openedBy; status open|resolved; resolution refund|release|split; resolvedBy)
- `webhook_events` (stripeEventId **UNIQUE**, type, payload, processedAt) — idempotency dedupe
- `pending_emails` (outbox: to, templateKey, payload, status, attempts, nextRetryAt)
- `audit_logs` (actorUserId, action, targetType, targetId, metadata, createdAt) — append-only

**Transactions:** money mutations use `prisma.$transaction(fn, { isolationLevel: 'Serializable' })`
with explicit `SELECT … FOR UPDATE` (raw) on the locked account rows for release/payout.

## 5. Auth, Authorization & Tenancy

- **AuthN:** session cookie (httpOnly, Secure, SameSite=Lax). `requireAuth` resolves the principal.
- **AuthZ (two layers):**
  1. **Tenant/ownership scope** — services filter by owner; cross-owner reads return `E.NOT_FOUND`.
  2. **Object-level** — `assertResourceAccess(row, principal)` chokepoint on every owned route **and
     every sub-resource / child-by-id** (milestones, payouts, disputes, ledger of a milestone). Admins
     bypass via explicit `requireRole('ADMIN')`, never implicit.
- **Anti-enumeration:** register with an existing email returns the same shape as a new email; login
  failure returns one envelope regardless of unknown-email / wrong-password / rate-limited; timing
  padded via per-(ip,email) rate-limit before the bcrypt compare.

## 6. Cross-Cutting Infrastructure

- **Logging:** pino structured JSON; request id; never logs secrets, tokens, or PAN-like values.
- **Metrics + alerting:** `/metrics` (Prometheus text) on api exposing counters (funds_held,
  releases, payouts, reconciliation_imbalance_cents) + a reconciliation alert when Σ≠0. (`monitoring:
  metrics + alerting`.)
- **Errors:** Sentry (`@sentry/node`, `@sentry/nextjs`), no-op without DSN.
- **Rate limiting:** in-memory token bucket keyed per route dimension (`scale: small` → no Redis).
- **Backups:** `backup_strategy: automated daily` → documented `pg_dump` cron in QUICKSTART + compose
  `backup` note; not a runtime service at small tier (ADR if added).
- **CI/CD:** `.github/workflows/ci.yml` runs install → typecheck → lint → test → build → invariant lint.
- **Header buffers:** `node --max-http-header-size=32768` on api + web start (ADR-010).
- **TLS:** `protocol_support: HTTPS only` → production behind a reverse proxy terminating TLS with an
  HTTP→HTTPS redirect + HSTS; local dev via mkcert/self-signed (QUICKSTART §Protocol & TLS).

## 7. Key Technical Decisions & Alternatives

| Decision | Chosen | Alternatives | Why |
|----------|--------|--------------|-----|
| Escrow mechanism | Separate charges & transfers (hold on platform, Transfer on approval) | Destination charges (`transfer_data`) | Destination charges move funds immediately — no milestone-gated hold. (ADR-003) |
| Ledger balances | Derived (SUM of entries) | Stored mutable balance column | Stored balances drift; derived is provably correct at small scale. (ADR-004) |
| Idempotency | DB UNIQUE constraint (event id + business key) | App-level read-then-check | TOCTOU race; UNIQUE closes it at the DB. (ADR-006) |
| Auth | Server session cookie | JWT, Auth.js | Simplest correct multi-role + revocation; no client token storage. (ADR-002) |
| Email/dual-write | Outbox + drain worker | Synchronous Resend call | Guaranteed delivery; decouples request latency. (ADR-007) |
| Money type | `BigInt` minor units | Float / Decimal | Float corrupts totals; integer cents match Stripe. (ADR-001) |
| Concurrency | Serializable tx + `FOR UPDATE` | Optimistic version only | `has_payments` bumps concurrency-safety above small-tier baseline. |

## 8. Implementation Notes & Risks

- **Reconciliation** runs in the worker every N minutes and on demand via `GET /v1/ledger/reconcile`
  (admin): asserts every `ledger_transaction` legs Σ=0 and the closed-system total Σ=0; emits a metric
  and (on imbalance) an alert + audit entry. This operationalizes Success Metric #1.
- **Race test** (Success Metric #2) — an integration test fires two concurrent release/payout requests
  over the same milestone/balance and asserts exactly one succeeds, no negative balance, no double
  ledger transaction.
- **Webhook retry retention** — `webhook_events` rows kept ≥30 days (> Stripe's 3-day retry window).
- **Risk:** bleeding-edge stack (Next 16 / Prisma 7 / Express 5). Mitigation: conservative, well-trodden
  API usage; pinned versions; CI build gate.

## 9. Architectural Invariants

Each rule below has a machine-checkable or `manual` entry in `invariants.json` (same `id`). The Stage 4
build lint pass and the ship-time Drift Detection Gate evaluate this file.

| ID | Rule | File / section | Check type |
|----|------|----------------|-----------|
| **INV-1** | All scoped DB writes/reads go through `apps/api/src/services/*` — no direct `prisma.<model>` calls outside services/seed/lib. | §3 boundary rule | forbidden-pattern |
| **INV-2** | Money is never stored or computed as floating point — no `Float`/`Decimal`/`parseFloat` on money paths; amounts are `BigInt` minor units. | §4 Data Layer | forbidden-pattern |
| **INV-3** | The ledger transaction table enforces a UNIQUE idempotency key so a money mutation runs at most once. | §4 `ledger_transactions` | required-unique-constraint |
| **INV-4** | Webhook events are deduped by a UNIQUE `stripeEventId` (at-least-once → exactly-once). | §4 `webhook_events` | required-unique-constraint |
| **INV-5** | On the Stripe webhook route, signature verification precedes any DB read/processing. | §2 step 1; webhook.service | boundary-order |
| **INV-6** | An executable invariant-lint runner exists and is wired into CI/verification. | §3 scripts | required-file |
| **INV-7** | The double-entry ledger posting function asserts every transaction's legs sum to zero (Σ=0). | §2; ledger.service | manual |
| **INV-8** | Object-level authorization (`assertResourceAccess`) is enforced on owned resources and every sub-resource, in addition to tenant scope. | §5 AuthZ | manual |
| **INV-9** | No raw card data (PAN/CVV) is ever stored or logged; only Stripe tokens + last4/brand. | §1, §6; PCI SAQ-A | forbidden-pattern |
| **INV-10** | Every SPEC §UI Surface screen is routable; every non-internal endpoint is reachable from the UI; every Key Workflow has an e2e test to its terminal step. | SPEC §UI Surface | ui-coverage |
| **INV-11** | Payout amount can never exceed the freelancer's available (FREELANCER_PAYABLE) balance; enforced under a row lock. | §2 step 4; payout.service | manual |
| **INV-12** | The chart of accounts is a closed system — the sum of all account balances is always zero. | ADR-005; reconciliation | manual |

## 10. Threat Model (comprehensive)

See RESEARCH §5 for the full STRIDE assessment and risk register. Architecture-level mitigations:

- **Browser↔API:** session auth + `assertResourceAccess` (R6/R7); server-authoritative amounts; audit
  log; card data never enters Tradewind (R10 / SAQ-A).
- **API↔Stripe:** idempotency keys on every charge/transfer/payout (R3/R5); secrets server-side only,
  test-mode, never logged (R8); pre-transfer balance assertion under lock (R2/R11).
- **Webhook↔API:** HMAC verify before DB read (INV-5/R4); timestamp tolerance; `event.id` UNIQUE dedupe.
- **API↔DB:** least-privilege role; owner predicate (R7); serializable + locked money tx (R1/R5);
  append-only ledger; reconciliation alert (INV-7/INV-12).
