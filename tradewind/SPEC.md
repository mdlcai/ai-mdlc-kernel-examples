# SPEC.md — Tradewind

> Source of truth for **behavior**. Build depth: **comprehensive**. Another AI could recreate the system
> from this document alone. Precedence: RESEARCH > ARCHITECTURE > SPEC. Non-Goals (RESEARCH) are excluded.

## 1. Scope

A milestone-escrow services marketplace with three roles (CLIENT, FREELANCER, ADMIN), a double-entry
ledger that always balances, Stripe Connect (test mode) for funding + payouts, dispute resolution, an
audit trail, and CSV/PDF reporting. Money is integer minor units (USD cents).

## 2. Roles & Permissions (RBAC)

| Capability | CLIENT | FREELANCER | ADMIN |
|------------|:------:|:----------:|:-----:|
| Register / login / logout | ✓ | ✓ | ✓ (seeded) |
| Create project + milestone | ✓ (own) | — | — |
| Accept project/milestone | — | ✓ (assigned) | — |
| Fund milestone (Checkout) | ✓ (own) | — | — |
| Mark delivered | — | ✓ (own) | — |
| Approve / release | ✓ (own) | — | — |
| Open dispute | ✓ (own milestone) | ✓ (own milestone) | — |
| Resolve dispute (refund/release/split) | — | — | ✓ |
| Connect onboarding | — | ✓ (own) | — |
| Request payout | — | ✓ (own balance) | — |
| View own ledger / statement | ✓ (own) | ✓ (own) | ✓ (all) |
| View full ledger / reconcile | — | — | ✓ |
| Export CSV / PDF | ✓ (own) | ✓ (own) | ✓ (all) |

Authorization: tenant/owner scope **plus** object-level `assertResourceAccess` on every owned route and
sub-resource (INV-8). Cross-owner reads return `404 NOT_FOUND` (never reveal existence). Admin actions
require explicit `requireRole('ADMIN')`.

## 3. Data Model (Prisma 7 / PostgreSQL)

All money: `amountMinor BigInt` + `currency CHAR(3)` (USD). Timestamps `TIMESTAMPTZ`. IDs `cuid()`.

### users
`id, email UNIQUE (citext), passwordHash, role (CLIENT|FREELANCER|ADMIN), displayName,
stripeConnectAccountId NULL, connectChargesEnabled BOOL default false, createdAt, updatedAt`

### sessions
`id, userId FK, tokenHash UNIQUE, expiresAt, createdAt` — token in cookie is random 32-byte; only its
SHA-256 hash is stored.

### projects
`id, clientId FK(users), title, description, status (OPEN|IN_PROGRESS|COMPLETED|CANCELLED), createdAt`

### milestones
`id, projectId FK, freelancerId FK(users) NULL, title, amountMinor BigInt, currency,
status (PENDING_ACCEPTANCE|ACCEPTED|FUNDED|DELIVERED|RELEASED|DISPUTED|REFUNDED|SPLIT|CANCELLED),
stripeCheckoutSessionId NULL, stripePaymentIntentId NULL, fundedAt NULL, releasedAt NULL, createdAt`
Indexes: `(projectId)`, `(freelancerId)`, `(status)`.

### accounts  (chart of accounts; ADR-005)
`id, type (EXTERNAL|ESCROW_HELD|FREELANCER_PAYABLE|PLATFORM_FEE|PAYOUT_CLEARING), ownerUserId NULL,
milestoneId NULL, currency, createdAt` — UNIQUE `(type, ownerUserId, milestoneId)`. System accounts
(EXTERNAL, PLATFORM_FEE) have null owner+milestone. ESCROW_HELD is per-milestone. FREELANCER_PAYABLE and
PAYOUT_CLEARING are per freelancer user.

### ledger_transactions  (immutable)
`id, kind (FUND|RELEASE|REFUND|SPLIT|PAYOUT_INITIATE|PAYOUT_PAID|PAYOUT_FAILED), idempotencyKey UNIQUE,
memo, createdAt`

### ledger_entries  (immutable, append-only)
`id, transactionId FK, accountId FK, amountMinor BigInt (signed), currency, createdAt`
**Invariant (INV-7):** for each `transactionId`, `SUM(amountMinor) = 0`. Balance of an account =
`SUM(amountMinor)` over its entries. **No UPDATE/DELETE** — corrections are reversing transactions.

### payouts
`id, freelancerId FK, amountMinor BigInt, currency, status (REQUESTED|PAID|FAILED),
stripeTransferId NULL, failureCode NULL, createdAt, settledAt NULL` — UNIQUE on the initiating ledger
transaction idempotencyKey `payout:<id>`.

### disputes
`id, milestoneId FK, openedByUserId FK, reason, status (OPEN|RESOLVED),
resolution (REFUND|RELEASE|SPLIT) NULL, splitClientBps INT NULL, resolvedByUserId FK NULL,
resolvedAt NULL, createdAt` — at most one OPEN dispute per milestone (partial UNIQUE).

### webhook_events  (idempotency)
`id, stripeEventId UNIQUE, type, payload JSONB, processedAt NULL, createdAt` — retained ≥30 days.

### pending_emails  (outbox; has_email)
`id, toAddress, templateKey, payload JSONB, status (pending|sent|failed|dlq), attempts INT default 0,
nextRetryAt, createdAt, sentAt NULL, errorText NULL` — index `(status, nextRetryAt)`;
UNIQUE `(toAddress, templateKey, businessRefId)` to prevent duplicate enqueues.

### audit_logs  (append-only)
`id, actorUserId FK NULL, action, targetType, targetId, metadata JSONB, ip, createdAt`

**Migrations:** every table has a forward migration; reversible (down) paths provided. Seed data:
3 demo users (client@demo.test, freelancer@demo.test, admin@demo.test — password `Tradewind-demo-2026!`),
system accounts (EXTERNAL, PLATFORM_FEE).

## 4. Security Requirements (PCI-DSS + SOC2 baseline) — `compliance:` tagged

| ID | Requirement | Control |
|----|-------------|---------|
| SEC-1 | Passwords hashed | **scrypt** (N=16384, r=8, p=1, 64-byte key, per-user salt) via Node's built-in crypto — a memory-hard OWASP-recommended KDF; constant-time compare; never plaintext/reversible. (Chosen over bcrypt per ADR-012 — avoids native-module build friction; equal/stronger posture.) |
| SEC-2 | Breached-password screening (NIST 800-63B SHALL) | HIBP range/k-anonymity (SHA-1 first 5 hex) on register + change; reject compromised. `compliance:NIST` |
| SEC-3 | Password length ≥ 15, accept ≤ 64, no composition rules, no forced rotation | NIST 800-63B; stricter than PCI v4's 12. |
| SEC-4 | Rate limit auth before hash compare | per-(ip,email) limiter fires inside authorize() before bcrypt. |
| SEC-5 | Anti-enumeration (shape + timing) | identical response shape/timing for unknown-email vs wrong-password vs rate-limited; register-existing-email same shape. |
| SEC-6 | Session security | httpOnly + Secure + SameSite=Lax cookie; server-side session table; logout invalidates. |
| SEC-7 | CSRF/Origin guard | Origin check on all state-mutating routes. |
| SEC-8 | Tenant + object-level authorization | scoped services + `assertResourceAccess` on owned + sub-resources (INV-8). |
| SEC-9 | Webhook signature before DB read | `stripe.webhooks.constructEvent` first; dedupe by event id (INV-4/INV-5). `compliance:` |
| SEC-10 | Idempotent money mutations | UNIQUE idempotency key per ledger transaction (INV-3); Stripe Idempotency-Key on charges/transfers/payouts. |
| SEC-11 | No raw card data (PCI SAQ-A) | Stripe hosted Checkout; store only token + last4/brand; PAN/CVV never in DB or logs (INV-9). `compliance:PCI-DSS` |
| SEC-12 | At-rest encryption | **⚠ delegated** to prod volume/RDS (KMS/TDE); dev unencrypted (ADR-008). `compliance:PCI-DSS,GDPR` |
| SEC-13 | Audit logging | every fund-moving + privileged admin action writes audit_logs (INV; SOC2). `compliance:SOC2` |
| SEC-14 | Rate limiting on public endpoints | token-bucket per route dimension. |
| SEC-15 | Secrets in env only, zod-validated, fail-fast | config/env.ts; never logged. |
| SEC-16 | Processing integrity (SOC2) | reconciliation: Σ legs = 0 per tx + closed-system Σ = 0; alert on imbalance (INV-7/INV-12). `compliance:SOC2` |
| SEC-17 | TLS in transit | HTTPS only; HTTP→HTTPS redirect + HSTS in prod (RESEARCH protocol_support). |

## 5. Features & Key Workflows (§5)

Money math: `fee = floor(amountMinor * PLATFORM_FEE_BPS / 10000)` (default 1000 bps = 10%);
`freelancerNet = amountMinor - fee`. Rounding remainder stays in the fee leg. All legs in one tx; Σ=0.

### W1 — Client creates project + milestone  *(F-04, F-05)*
`POST /v1/projects` then `POST /v1/projects/:id/milestones {title, amountMinor, currency}`.
Validation: amountMinor ≥ 100 (≥ $1), integer, currency=USD. Milestone → `PENDING_ACCEPTANCE`.
Errors: 400 invalid amount; 403/404 not project owner.

### W2 — Freelancer accepts milestone  *(F-06)*
`POST /v1/milestones/:id/accept`. Pre: status=PENDING_ACCEPTANCE, freelancer not the client.
Sets `freelancerId`, status → `ACCEPTED`. Race: optimistic guard on status; second accept → 409.

### W3 — Client funds milestone via Stripe Checkout  *(F-07, has_payments)*
`POST /v1/milestones/:id/fund` → creates Stripe Checkout Session (`mode:'payment'`, line item =
amountMinor, idempotencyKey `fund:<milestoneId>`, success/cancel URLs to web). Returns `{ checkoutUrl }`.
Pre: status=ACCEPTED, caller=client owner. The browser redirect is **not** trusted; funding is confirmed
**only** by webhook (W8). On `payment_intent.succeeded`: in one serializable tx → milestone `FUNDED`,
ledger tx `FUND`: **EXTERNAL −amount / ESCROW_HELD(milestone) +amount** (Σ=0), audit + outbox email to
freelancer. Idempotent on event id and `fund:<milestoneId>`.
Errors: 409 already funded; 402 on Stripe error; state transition guard.

### W4 — Freelancer marks delivered  *(F-08)*
`POST /v1/milestones/:id/deliver`. Pre: status=FUNDED, caller=assigned freelancer. → `DELIVERED`.
No money movement. Audit + outbox email to client. Race: status guard → 409.

### W5 — Client approves → release  *(F-09, money-correctness)*
`POST /v1/milestones/:id/approve` (idempotencyKey header optional; server derives `release:<id>`).
Pre: status=DELIVERED (or FUNDED if client approves early — allowed), caller=client owner, no OPEN
dispute. In one serializable tx, `SELECT FOR UPDATE` the ESCROW_HELD account, assert held ≥ amount, post
ledger tx `RELEASE`: **ESCROW_HELD −amount / FREELANCER_PAYABLE(freelancer) +net / PLATFORM_FEE +fee**
(Σ=0), milestone → `RELEASED`, `releasedAt`. Audit + outbox. **Double-release impossible:** UNIQUE
`release:<milestoneId>` + one-way state. Errors: 409 already released; 409 open dispute; 422 if held <
amount (should never happen — reconciliation alert).

### W6 — Freelancer payout  *(F-11, money-correctness)*
Pre: freelancer has a Connect account with `connectChargesEnabled`/transfers (W10).
`POST /v1/payouts {amountMinor}`. In one serializable tx: `SELECT FOR UPDATE` FREELANCER_PAYABLE, assert
**available ≥ amountMinor** (INV-11) else `422 INSUFFICIENT_FUNDS`; post ledger tx `PAYOUT_INITIATE`:
**FREELANCER_PAYABLE −amount / PAYOUT_CLEARING +amount** (Σ=0); create payout row `REQUESTED`; create
Stripe Transfer to connected account (idempotencyKey `payout:<payoutId>`). Audit.
- On `payout.paid` webhook → tx `PAYOUT_PAID`: **PAYOUT_CLEARING −amount / EXTERNAL +amount**, payout →
  `PAID`, `settledAt`. Outbox email.
- On `payout.failed` → tx `PAYOUT_FAILED` (reverse): **PAYOUT_CLEARING −amount / FREELANCER_PAYABLE
  +amount**, payout → `FAILED`, `failureCode`. Funds returned to available balance. Outbox email.
Race: two concurrent payout requests over the same balance — serializable + row lock ⇒ exactly one
succeeds; the other gets 422 (covered by race test).

### W7 — Dispute + admin resolution  *(F-10, F-12)*
`POST /v1/milestones/:id/disputes {reason}` by client or freelancer on a FUNDED/DELIVERED milestone →
milestone `DISPUTED`, dispute `OPEN` (at most one open). Blocks release.
Admin: `POST /v1/disputes/:id/resolve {resolution, splitClientBps?}` (requireRole ADMIN):
- **REFUND:** Stripe Refund of the PaymentIntent; ledger tx `REFUND`: **ESCROW_HELD −amount / EXTERNAL
  +amount**; milestone → `REFUNDED`.
- **RELEASE:** as W5 (`RELEASE` legs); milestone → `RELEASED`.
- **SPLIT:** `clientPart = floor(amount*splitClientBps/10000)`, remainder to freelancer minus fee on the
  freelancer part. Ledger tx `SPLIT`: **ESCROW_HELD −amount / EXTERNAL +clientPart /
  FREELANCER_PAYABLE +(freelancerPart−feeOnFreelancerPart) / PLATFORM_FEE +feeOnFreelancerPart** (Σ=0);
  milestone → `SPLIT`. Stripe partial Refund for clientPart.
Every resolution writes audit (actor=admin) + outbox emails to both parties. Dispute → `RESOLVED`.
Errors: 403 non-admin; 409 dispute already resolved; 422 invalid split bps.

### W8 — Stripe webhooks (idempotent)  *(F-03, has_webhooks)*
`POST /v1/webhooks/stripe` mounted with `express.raw`. Order (INV-5): **verify signature
(`constructEvent`, 5-min tolerance) → insert webhook_events(stripeEventId) [UNIQUE; on conflict, 200
no-op] → process in serializable tx**. Handles `checkout.session.completed`/`payment_intent.succeeded`
(W3), `payout.paid`/`payout.failed` (W6). Unknown events: stored + 200. Anomaly (e.g. paid event for a
non-REQUESTED payout) → audit + metric, no double-apply. Replay of any event ⇒ no duplicate ledger
entries (Success Metric).

### W9 — View ledger / statement  *(F-13)*
`GET /v1/ledger?scope=me` (own entries) / `GET /v1/ledger/all` (admin). Returns transactions + entries +
derived balances. `GET /v1/accounts/me/balance` → available FREELANCER_PAYABLE for the freelancer.

### W10 — Connect onboarding  *(F-02)*
`POST /v1/connect/onboard` (freelancer) → create Express account if absent + Account Link; returns
`{ onboardingUrl }`. `GET /v1/connect/status` reflects `charges_enabled`/`payouts_enabled` (refreshed
from Stripe or `account.updated` webhook). Payout (W6) blocked until enabled.

### W11 — Reports (CSV/PDF)  *(F-14, report_formats)*
`GET /v1/reports/ledger.csv` and `.pdf` (own scope; admin = all). CSV = ledger entries; PDF = formatted
statement (date, description, amount, status). Streamed; respects scope.

### W12 — Reconciliation  *(F-15, SEC-16)*
Worker every 5 min + admin `GET /v1/ledger/reconcile`: assert every tx Σ=0 and closed-system Σ=0; on
imbalance → metric `reconciliation_imbalance_cents` + audit + alert. Always returns the posture.

## 6. API Surface (REST `/v1/`)

| Method + Path | Auth | Role | Body / Query | Success | Errors |
|---------------|------|------|--------------|---------|--------|
| POST /v1/auth/register | — | — | {email,password,role,displayName} | 201 {user} | 400 validation; 200-shape for existing email (SEC-5) |
| POST /v1/auth/login | — | — | {email,password} | 200 + Set-Cookie | 401 generic (SEC-5); 429 |
| POST /v1/auth/logout | ✓ | any | — | 204 | 401 |
| GET /v1/auth/me | ✓ | any | — | 200 {user} | 401 |
| POST /v1/projects | ✓ | CLIENT | {title,description} | 201 {project} | 400; 403 |
| GET /v1/projects | ✓ | any | — | 200 [own/all] | 401 |
| GET /v1/projects/:id | ✓ | owner/admin | — | 200 | 404 (scope) |
| POST /v1/projects/:id/milestones | ✓ | CLIENT owner | {title,amountMinor,currency} | 201 | 400; 404 |
| GET /v1/milestones/:id | ✓ | party/admin | — | 200 | 404 |
| POST /v1/milestones/:id/accept | ✓ | FREELANCER | — | 200 | 409; 404 |
| POST /v1/milestones/:id/fund | ✓ | CLIENT owner | — | 200 {checkoutUrl} | 409; 402; 404 |
| POST /v1/milestones/:id/deliver | ✓ | FREELANCER assigned | — | 200 | 409; 404 |
| POST /v1/milestones/:id/approve | ✓ | CLIENT owner | — | 200 | 409; 422; 404 |
| GET /v1/milestones/:id/ledger | ✓ | party/admin | — | 200 [entries] | 404 (sub-resource authz) |
| POST /v1/milestones/:id/disputes | ✓ | party | {reason} | 201 | 409; 404 |
| POST /v1/disputes/:id/resolve | ✓ | ADMIN | {resolution,splitClientBps?} | 200 | 403; 409; 422 |
| GET /v1/disputes | ✓ | admin/party | — | 200 | 401 |
| POST /v1/connect/onboard | ✓ | FREELANCER | — | 200 {onboardingUrl} | 402 |
| GET /v1/connect/status | ✓ | FREELANCER | — | 200 {enabled} | 401 |
| POST /v1/payouts | ✓ | FREELANCER | {amountMinor} | 201 {payout} | 422 insufficient; 409 |
| GET /v1/payouts | ✓ | FREELANCER/admin | — | 200 [own] | 401 |
| GET /v1/ledger | ✓ | any (own) | ?scope | 200 | 401 |
| GET /v1/ledger/all | ✓ | ADMIN | — | 200 | 403 |
| GET /v1/ledger/reconcile | ✓ | ADMIN | — | 200 {balanced,imbalanceCents} | 403 |
| GET /v1/accounts/me/balance | ✓ | FREELANCER | — | 200 {availableMinor} | 401 |
| GET /v1/reports/ledger.csv | ✓ | any (own) | — | 200 text/csv | 401 |
| GET /v1/reports/ledger.pdf | ✓ | any (own) | — | 200 application/pdf | 401 |
| POST /v1/webhooks/stripe | sig | — | raw | 200 | 400 bad sig |
| GET /v1/health | — | — | — | 200 {status,services} | — |
| GET /metrics | — | — | — | 200 text/plain | — |

## 7. Error Contract

Typed envelope: `{ error: { code, message, fields?: {field: message} } }`. Codes (`E.*`):
`VALIDATION` (400, with field map), `UNAUTHENTICATED` (401), `FORBIDDEN` (403), `NOT_FOUND` (404),
`CONFLICT` (409, state), `INSUFFICIENT_FUNDS` (422), `LEDGER_IMBALANCE` (500, alert), `RATE_LIMITED`
(429), `PAYMENT_ERROR` (402), `INTERNAL` (500). Validation errors map field-level issues so the UI can
surface them on the offending control (BUILD per-feature design pass). Auth errors are deliberately
generic (SEC-5). Never leak stack traces or raw DB errors.

## 8. UI Surface (screens)

Sidebar + content layout (saas archetype). Each screen lists route · purpose · states · endpoints · KW · RBAC.

| Route | Purpose | States | Endpoints | KW | Roles |
|-------|---------|--------|-----------|----|-------|
| `/` (landing) | Product intro + CTA to login/register | static | — | — | public |
| `/register` | Create account (role select) | empty/loading/error(field)/success | POST /auth/register | — | public |
| `/login` | Sign in | empty/loading/error(generic)/success | POST /auth/login | — | public |
| `/dashboard` | Role-aware home (client: projects; freelancer: balance+payouts; admin: ledger summary) | loading/empty/error/success | GET /auth/me, /projects, /accounts/me/balance | W1–W12 entry | all |
| `/projects` | Client: list + create project | empty/loading/error/success | GET/POST /projects | W1 | CLIENT |
| `/projects/[id]` | Project detail + milestones + create milestone | empty/loading/error/success | GET /projects/:id, POST …/milestones | W1 | owner/admin |
| `/milestones/[id]` | Milestone detail + actions (accept/fund/deliver/approve/dispute) + ledger | loading/empty/error/success | milestone action endpoints, GET …/ledger | W2–W5,W7 | party/admin |
| `/funding/success` | Post-Checkout return (shows pending→confirmed via webhook poll) | loading/success/error | GET /milestones/:id | W3 | CLIENT |
| `/funding/cancel` | Checkout cancelled | static | — | W3 | CLIENT |
| `/payouts` | Freelancer: available balance, request payout, payout history + status | empty/loading/error/success | GET /accounts/me/balance, GET/POST /payouts | W6 | FREELANCER |
| `/connect` | Connect onboarding status + start onboarding | loading/error/success | POST /connect/onboard, GET /connect/status | W10 | FREELANCER |
| `/disputes` | List disputes; admin resolve (refund/release/split) | empty/loading/error/success | GET /disputes, POST /disputes/:id/resolve | W7 | admin/party |
| `/ledger` | Ledger table (own; admin all) + balances + reconcile + export CSV/PDF | empty/loading/error/success | GET /ledger, /ledger/reconcile, /reports/* | W9,W11,W12 | all |

**Every Key Workflow is completable end-to-end through the UI to its terminal step** (fund → Checkout →
confirmed; deliver; approve → released; payout request → status; dispute → admin resolve; export). Every
non-internal endpoint has a UI affordance. `/metrics` and `/webhooks/stripe` are internal (no screen).

## 9. Performance & Non-Functional

- Scale: small (<1k concurrent). Single Postgres, framework pool. P95 API < 300ms for reads at seed
  volume. Money mutations serializable — correctness over latency.
- Reconciliation job < 1s at seed volume. Smoke covers W1–W12.
- Lighthouse target 95+ (accessibility AA) on rendered screens.

## 10. Configuration / Environment

See `.env.example`. Required: `DATABASE_URL`, `SESSION_SECRET`, `STRIPE_SECRET_KEY`,
`STRIPE_WEBHOOK_SECRET`, `WEB_BASE_URL`, `API_BASE_URL`. Optional: `RESEND_API_KEY`, `SENTRY_DSN`,
`PLATFORM_FEE_BPS` (default 1000), `HIBP_ENABLED` (default true). All validated by zod at boot (fail-fast).

## 11. Domain Checklists (mapped)

- **has_payments:** money = BigInt minor units + currency (§3); idempotent charge/transfer creation
  (Stripe Idempotency-Key + UNIQUE ledger key, SEC-10); provider-is-truth via webhook not redirect
  (W3/W8); refund/dispute/split lifecycle modeled (W7); never persist PAN/CVV (SEC-11). ✓
- **has_webhooks:** signature verify before DB read (W8/INV-5); UNIQUE event id (INV-4); retention
  ≥30 days > 3-day retry window; anomaly-flag pattern (W8). ✓
- **has_dual_write:** outbox pattern for email — `pending_emails` inserted in the same tx as the business
  write; drain worker with capped backoff → dlq (ADR-007). ✓

## 12. Compliance Baseline (mapped)

- **PCI-DSS v4.0 (SAQ-A):** SEC-11 tokens-only, hosted Checkout, no PAN; SEC-3 length ≥ 12 (we use 15);
  SEC-12 at-rest ⚠ delegated.
- **NIST 800-63B verifier:** SEC-2 HIBP breach screen, SEC-3 length/no-composition/no-rotation, SEC-4
  rate-limit before hash.
- **SOC2:** SEC-13 audit logging, SEC-16 processing integrity (reconciliation), Security TSC across
  SEC-1…SEC-17.

## 13. Test Plan

- **Unit:** money math (fee split, rounding remainder), ledger Σ=0 assertion, HIBP screen, password rules.
- **Integration (supertest + test Postgres):** auth (register/login/logout/me, anti-enumeration shape),
  full escrow happy path W1→W5, payout W6 (paid + failed), dispute resolutions (refund/release/split),
  webhook idempotency (replay → no dup), authorization (cross-tenant + same-tenant IDOR on milestone +
  sub-resources), reconciliation closed-system Σ=0.
- **Race:** two concurrent approves → one release; two concurrent payouts → one succeeds, no negative
  balance.
- **e2e (Playwright):** each Key Workflow through the UI to its terminal step (INV-10 ui-coverage).
- **Negative paths:** invalid amount field-level message; payout over balance 422; non-admin resolve 403.
