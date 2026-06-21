# RESEARCH.md — Tradewind

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "fintech"

## Domain Signals

```yaml
domain_signals: ["has_webhooks", "has_payments", "has_dual_write"]
```

## Product Vision
**Problem:** When clients and independent freelancers transact directly, neither side is protected: clients pay up
  front and risk getting nothing; freelancers deliver work and risk not getting paid. There's no
  neutral party holding the money against agreed milestones, no structured way to dispute a bad
  outcome, and no transparent record of where funds are. For anyone building a platform to intermediate
  these deals, the hard, high-stakes part is getting the money handling exactly right — escrow
  accounting, fee splits, payouts, refunds, and a ledger that always balances — because mistakes there
  mean lost or double-paid funds.

**Who it affects:** - Clients (buyers): post projects and milestones, fund escrow, approve delivered work, open disputes.
  Can only view and act on their own projects and transactions.
  - Freelancers (sellers): accept projects, deliver work, request payouts of their available balance to
  a connected account. Can only view their own projects, balance, and payouts.
  - Platform admins: resolve disputes (refund, release, or split) and review the full ledger. Cannot
  move funds without producing an audit entry; every privileged action is logged.

**Why existing solutions fall short:** Closed marketplaces (Upwork/Fiverr-style) bundle escrow inside a platform you can't control, with
  opaque fees and no access to the underlying ledger. Raw payment tools (Stripe, PayPal, invoicing
  apps) move money but give you no escrow state machine, no milestone-gated release, no dispute
  resolution, and no double-entry ledger — leaving you to build the financial correctness yourself,
  which is exactly where teams introduce double-charges, negative balances, and reconciliation drift.
  Tradewind is the missing middle: a transparent, auditable escrow-and-payout engine with the
  milestone, dispute, and ledger logic built in and provably correct.

**Solution:** Tradewind is a multi-vendor services marketplace where clients hire freelancers and money is held in
  milestone-based escrow. A client funds a milestone up front; the platform holds the funds; when the
  freelancer delivers and the client approves, the platform releases the money to the freelancer minus
  a platform fee and schedules a payout to their connected account. Disputes escalate to an admin who
  can refund the client, release to the freelancer, or split the funds. Every movement of money is
  recorded in an immutable double-entry ledger with a full audit trail, and balances must always
  reconcile. Payments run through Stripe (Checkout + Connect) in test mode — this is a complete,
  working reference build, not a live money-transmission service.


## Users & Outcomes
**Key Workflows:**
W1. A client signs up, creates a project, and posts a milestone with a fixed amount.
  W2. A freelancer accepts the project and its milestone.
  W3. The client funds the milestone via Stripe Checkout (test mode); on payment success the funds move
  into escrow (held) and the ledger records the entry.
  W4. The freelancer marks the milestone delivered.
  W5. The client approves; the platform releases escrow — the freelancer is credited (amount minus
  platform fee) and the platform fee is credited — with debits equal to credits.
  W6. The freelancer requests a payout of their available balance to a connected account (Stripe
  Connect, test mode); the payout moves through a state machine (requested → paid or failed).
  W7. The client opens a dispute on a delivered milestone; an admin resolves it by refunding the
  client, releasing to the freelancer, or splitting the funds; the ledger and audit trail reflect the
  outcome.
  W8. Stripe webhooks (payment_intent.succeeded, payout.paid, payout.failed) are received and applied
  idempotently, so retried deliveries never double-credit or double-apply.

**Success Metrics:**
- The double-entry ledger always balances — total debits equal total credits across every
  transaction, verified by an automated reconciliation check.
  - No funds lost or double-paid: escrow balance never goes negative, each milestone releases at most
  once, and a payout can never exceed available balance — proven under concurrent requests by a race
  test.
  - Webhook idempotency holds: replaying any Stripe webhook produces no duplicate ledger entries or
  state changes.
  - Isolation holds: a client or freelancer cannot read or act on another user's projects, milestones,
  balances, payouts, or disputes (cross-tenant and same-tenant cross-user).
  - Every fund-moving and privileged admin action is captured in the audit trail.
  - Build gates pass: typecheck, lint, tests, security invariants, and a functional smoke covering
  W1–W8 through the UI.

**Non-Goals:**
No raw card handling — Stripe owns PCI card scope; the app never touches card numbers. No
  crypto/blockchain. No real-time chat or messaging. No native mobile apps. No ML fraud scoring. No
  real money movement beyond Stripe test mode — Tradewind is a complete, working reference build, not a
  live money-transmission service (which would require money-transmitter licensing, KYC/AML, and real
  Connect onboarding).


## Build Constraints

```yaml
# Infrastructure & Ops
hosting_environment: "cloud (AWS/GCP/Azure)"
protocol_support: "HTTPS only"
ci_cd_required: true
monitoring: "metrics + alerting"
backup_strategy: "automated daily"
container_strategy: "single container (Docker Compose)"
error_reporting: "Sentry"

# Data & Storage
database_preference: "PostgreSQL"
pii_handling: ["PII in user profiles", "financial transaction data", "payment method tokens"]

# Notifications & Messaging
email_service: "Resend"
notification_urgency: "guaranteed delivery"
webhook_targets: ["Stripe"]

# Security & Compliance
security_baseline: ["PCI-DSS", "SOC2"]
rate_limiting: true
audit_logging: true
secrets_management: "environment variables"

# Frontend
frontend_framework: "Next.js"
css_approach: "Tailwind CSS"
state_management: "React Context"

# Backend
backend_framework: "Express"
api_style: "REST"
api_versioning: "URL path (/v1/)"
orm_preference: "Prisma"

# Performance & Quality
testing_strategy: "test-after"
logging_format: "structured JSON"

# Scope & Platform
scale: "small — under 1k concurrent"
multi_tenant: true
target_platforms: ["web"]
report_formats: ["PDF", "CSV"]
```

## Design Language

### Archetype
Archetype: saas

This product's design archetype is **saas**. Read `DESIGN.md` Part II §`saas` and treat its Layout
Doctrine, Density, Type System, Color & Atmosphere, Motion Budget, and Signature Components as binding
requirements, and its Good-vs-Avoid list as the acceptance rubric. The Universal Excellence floor
(`DESIGN.md` Part I) applies on top regardless of archetype.

### Brand Voice
Professional, confident, efficient. Composed information hierarchy; tables and forms done well; full dark mode.

### Art Direction
Tradewind's visual system is anchored on a confident indigo accent (#4f46e5) that signals trust and
financial authority without flash — every UI element earns its presence through clarity and precision,
never decoration. The palette sits on near-neutral surfaces (off-white #fafafa in light mode) with a
restrained secondary palette of cool grays for hierarchy and disabled states, allowing the indigo to
land with quiet authority on buttons, active states, and critical affordances like "Release Funds" or
"Approve Milestone." All monetary values render in a monospaced typeface (JetBrains Mono) at consistent
sizing to anchor them as immutable ledger entries, while body copy uses a neutral sans-serif (Inter) at
a generous baseline for legibility across dense transaction tables and forms. Layout is deliberately
utilitarian: tables are the primary information architecture, with right-aligned numerics, clear column
hierarchy (date, description, amount, status), subtle row dividers, and hover states that lift rows.
Spacing follows a 4px grid. Motion is minimal and purposeful (200ms ease-out). Accessibility is
non-negotiable: WCAG AA contrast, visible 2px focus outlines, always-visible form labels, error
messages above inputs with icon + text, monospaced numerics always paired with a currency symbol and
label. The system rejects rounded corners (max 2px on controls, 0px on tables) to reinforce precision
and fintech credibility, and every state — pending, approved, disputed, released, reconciled — has a
distinct visual marker (color + icon + status badge).

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #4f46e5 | Buttons, links, active states |
| Secondary | #dd4ed5 | Accents, badges, highlights |
| Accent | #d5db6e | Callouts, hover states |
| Background | #fafafa | Page background |
| Surface | #f4f4f5 | Cards, elevated containers |
| Text | #16161d | Headings, body text |
| Text Secondary | #5d5c70 | Captions, muted text |
| Success | #1fad53 | Success states, confirmations |
| Warning | #ec9c13 | Warnings, pending states |
| Error | #df2020 | Errors, destructive actions |

### Typography
- Heading: Space Grotesk (600/700 weight)
- Body: Inter (400/500 weight)
- Mono: JetBrains Mono (monetary values, code)
- Base size: 14px, scale ratio: 1.2

### Layout
- Pattern: Sidebar + Content
- Max width: 1440px, sidebar: 224px
- Spacing: Comfortable (12/16/24/32px)
- Breakpoints: 640 / 768 / 1024 / 1280px

### Component Style
- Variant: Sharp
- Border radius: 2px (sm: 0px, lg: 6px)
- Shadows: None
- Theme: Light only — single fixed palette, no theme switcher

## Design Template

No design template was uploaded for this project. Fall back to the archetype defaults from `DESIGN.md`
Part II plus the tokens above.

---

# §3 Source Categories (Stage 0 research — build_depth: comprehensive)

> Research conducted 2026-06-21 via web search + GitHub against vendor-of-truth references. All
> commands, flags, API signatures, and versions confirmed against the vendor's own documentation, not
> training memory. Resolved stack versions (June 2026): **stripe-node 22.2.2** (API `2026-05-27.dahlia`),
> **Prisma ORM 7.x**, **Next.js 16.2.9** (LTS, Node 20+), **Express 5.x**, **Sentry 10.59.0**, **Resend** current.

## §3.1 Official / Vendor Documentation

| Source | URL | Verified fact |
|--------|-----|---------------|
| Stripe Checkout — create session | https://docs.stripe.com/api/checkout/sessions/create | One-time payment uses `mode: 'payment'` + `line_items[]` + `success_url`. `payment_intent_data` accepts Connect fields `application_fee_amount` (integer minor units), `on_behalf_of`, and `transfer_data.destination`. |
| Stripe Connect — destination charges | https://docs.stripe.com/connect/destination-charges | Charge created on platform; fee via `application_fee_amount`; remainder transferred to connected account via `transfer_data[destination]`. Single-step. |
| Stripe Connect — separate charges & transfers | https://docs.stripe.com/connect/separate-charges-and-transfers | Two-step: PaymentIntent on platform, then a separate `Transfer` at release time; link via `transfer_group`. **This is the true escrow-timing model Tradewind uses** (hold on platform balance, release on approval). |
| Stripe Connect — Express accounts | https://docs.stripe.com/connect/express-accounts | `POST /v1/accounts type=express` with `capabilities[transfers][requested]`; onboard via `POST /v1/account_links` (`account`, `refresh_url`, `return_url`, `type=account_onboarding`). Returned URL single-use; check `charges_enabled`/`payouts_enabled`, don't assume completion on return. |
| Stripe Connect — testing | https://docs.stripe.com/connect/testing | Test payouts simulate but aren't bank-processed; test external bank numbers enable payouts with no real identity verification. Test keys only. |
| Stripe webhooks | https://docs.stripe.com/webhooks | `stripe.webhooks.constructEvent(rawBody, sigHeader, whsec)` requires the **raw unmodified body**, `Stripe-Signature` header, `whsec_…` secret. Default timestamp tolerance 5 min. **Live retries up to 3 days** → idempotency table must retain ≥72h. |
| Stripe Connect webhooks / event types | https://docs.stripe.com/connect/webhooks · https://docs.stripe.com/api/events/types | Connect events carry top-level `account`. `payment_intent.succeeded`, `checkout.session.completed`, `payout.paid`, `payout.failed` are valid v1 types; failed payout exposes `failure_code`. |
| Stripe Idempotency-Key | https://docs.stripe.com/api/idempotent_requests | Write requests accept client-supplied `Idempotency-Key` (e.g. UUID); Stripe caches+replays the first response. In stripe-node pass `{ idempotencyKey }` as the options arg. |
| Prisma ORM 7 | https://www.prisma.io/blog/announcing-prisma-orm-7-0-0 | Prisma 7.0.0 (2025-11-19), current major June 2026. Rust-free TS query compiler; generated client emitted into project source. |
| Prisma — transactions | https://www.prisma.io/docs/orm/prisma-client/queries/transactions | Sequential `$transaction([...])` (atomic) and interactive `$transaction(async (tx) => {...})` with `{ maxWait, timeout, isolationLevel }`. Use `Serializable` for money moves. |
| Prisma — BigInt | https://www.prisma.io/docs/orm/prisma-client/special-fields-and-types | `BigInt` maps to JS `BigInt`; correct type for integer-minor-unit money columns. |
| Next.js 16 | https://nextjs.org/blog/next-16-1 · https://endoflife.date/nextjs | Latest stable 16.2.9 (2026-06-09), major 16 LTS. Turbopack default; min Node 20. App Router with Route Handlers + Server Actions; `next start` runs prod server after `next build`. |
| Express 5 | https://expressjs.com/en/blog/2024-10-15-v5-release.html | Express 5.x (released 2024-10-15), Node 18+. Built-in `express.json()`, `express.raw()` — mount `express.raw({type:'application/json'})` on the webhook route only, before any global JSON parser. |
| Resend Node SDK | https://resend.com/docs/send-with-nodejs | `new Resend('re_…')`; `await resend.emails.send({from,to,subject,html})` → `{ data, error }`. |
| Sentry Next.js/Node | https://docs.sentry.io/platforms/javascript/guides/nextjs/ | sentry-javascript major 10 (10.59.0, 2026-06-19); `Sentry.init({ dsn, tracesSampleRate })`. |

## §3.2 GitHub Repositories

| Repo | URL | Stars | Last active | License | Relevance |
|------|-----|-------|-------------|---------|-----------|
| tigerbeetle/tigerbeetle | https://github.com/tigerbeetle/tigerbeetle | ~16.3k | 2026 (v0.17.4) | Apache-2.0 | Canonical double-entry (debit/credit) financial DB; reference for balancing-transfer + account-invariant semantics. |
| flash-oss/medici | https://github.com/flash-oss/medici | ~349 | 2025 (v7.2.0) | MIT | Node double-entry library: "every debit has a corresponding credit; everything balances to zero"; voiding via offsetting entries. |
| prisma/prisma-examples | https://github.com/prisma/prisma-examples | ~6.6k | 2026 | Apache-2.0 | Official Next.js (App Router)+Prisma and Express+Prisma examples — maps the exact stack. |
| stripe-archive/…kavholm-marketplace | https://github.com/stripe-archive/stripe-demo-connect-kavholm-marketplace | ~142 | archived 2022 | MIT | Stripe's two-sided Connect marketplace demo (funds routed to connected accounts). Reference-only (archived). |
| auchenberg-stripe/…connect-global-marketplace | https://github.com/auchenberg-stripe/stripe-sample-connect-global-marketplace | (Stripe sample) | — | MIT | Next.js + Node REST + Connect Express onboarding + PaymentIntents — closest front/back match. |
| AleRapchan/escrow-service | https://github.com/AleRapchan/escrow-service | ~22 | 2021 | MIT | Three-party escrow state machine (fund→hold→deliver→confirm/dispute→release/refund/arbitrate). |

## §3.3 Video / Tutorials

- **"How I Built a Marketplace with Stripe Connect"** — https://www.youtube.com/watch?v=A37tBAPpgY0 — onboarding connected accounts, payments, fund splits, platform application fees.
- **"Stripe Connect Escrow / Withhold Funds"** — https://www.youtube.com/watch?v=GiOJqFuNRZg — delayed-payout (separate charges + transfers) hold-then-release pattern; the mechanism behind "escrow" on Stripe.
- **"Designing a Payment System"** (Pragmatic Engineer) — https://newsletter.pragmaticengineer.com/p/designing-a-payment-system — pay-in/pay-out flows, double-entry ledger (Σ entries = 0), idempotency, reconciliation, dead-letter handling.

## §3.4 Articles

- Modern Treasury — *Accounting for Developers, Part I* — https://www.moderntreasury.com/journal/accounting-for-developers-part-i — debit-normal vs credit-normal accounts, balanced atomic transactions.
- Modern Treasury — *Enforcing Immutability in Your Double-Entry Ledger* — https://www.moderntreasury.com/journal/enforcing-immutability-in-your-double-entry-ledger — posted entries immutable; corrections via reversing entries; separate mutable business objects from immutable accounting objects.
- Stripe Engineering — *Online migrations at scale* — https://stripe.com/blog/online-migrations — four-phase dual-write migration for money-critical schemas.
- Anvil — *An Engineer's Guide to Double-Entry Bookkeeping* — https://anvil.works/blog/double-entry-accounting-for-engineers — Accounts/Transactions/JournalEntries schema with working code.
- Brandur Leach — *Implementing Stripe-like Idempotency Keys in Postgres* — https://brandur.org/idempotency-keys — atomic store + UNIQUE constraint; concurrent second request loses.

## §3.5 Standards / RFCs

- **PCI DSS v4.0 SAQ-A** — https://listings.pcisecuritystandards.org/documents/PCI-DSS-v4-0-SAQ-A.pdf — Tradewind's target scope: card-not-present, fully outsources account-data handling to Stripe, never stores/processes/transmits PAN. Tokens only ⇒ token store outside the CDE.
- **PCI DSS v4.0 Tokenization Guidelines** — https://www.pcisecuritystandards.org/documents/Tokenization_Guidelines_Info_Supplement.pdf — token-only systems fall outside the CDE; encrypted PANs in-scope, tokens not.
- **NIST SP 800-63B (Rev 4 / 800-63-4)** — https://pages.nist.gov/800-63-4/sp800-63b.html — single-factor min **15 chars**; **SHALL** screen against breached-password blocklist; **SHALL NOT** impose composition rules or force periodic rotation. (PCI v4 floor is ≥12; adopt the stricter 15.)
- **AICPA 2017 Trust Services Criteria (2022 PoF)** — https://www.aicpa-cima.com/resources/download/2017-trust-services-criteria-with-revised-points-of-focus-2022 — SOC 2 criteria; **Processing Integrity** most relevant for a ledger (completeness/accuracy/timeliness).
- **AICPA SOC Suite of Services** — https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services — SOC 1/2/3 scope, Type I vs II.

## §3.6 Competing Products

- **Upwork milestones** — https://support.upwork.com/hc/en-us/articles/211063718 — client funds milestone up front; 14-day review auto-release; 5-day security hold; opaque internal ledger.
- **Fiverr** — https://help.fiverr.com/hc/en-us/articles/34069565843985 — order-based; 80/20 split; 14-day clearing; Resolution Center disputes; closed accounting.
- **Escrow.com milestones** — https://www.escrow.com/milestones/how-it-works — licensed escrow agent; full pre-funding; formal arbitration. The regulated model Tradewind imitates economically, not legally.
- **Stripe Connect** — https://docs.stripe.com/connect/separate-charges-and-transfers — the payment rail (a dependency, not a competitor): no escrow accounts, only "escrow-like" holds via delayed transfers (≤~90 days). **Tradewind's own double-entry ledger differs by making every held/released/refunded/split cent provable in its own DB, independent of Stripe's balance reporting.**

## §3.7 Community Threads

- GitHub — *stripe-webhook-idempotency-guard* — https://github.com/primeautomation-dev/stripe-webhook-idempotency-guard — at-least-once webhooks; insert event id atomically before side effects for exactly-once business effect.
- Hookdeck — *How to Implement Webhook Idempotency* — https://hookdeck.com/webhooks/guides/implement-webhook-idempotency — dedupe on the constant `evt_…` id; atomic insert / unique-constraint guard over read-then-write.
- Bubble Forum — *Stripe for an Escrow-Style Marketplace* — https://forum.bubble.io/t/stripe-for-an-escrow-style-marketplace-app/348175 — hold-then-release, 90-day Connect hold limit, Transfers on milestone completion.

## §3.8 APIs / Integrations

- **stripe-node 22.2.2** — https://github.com/stripe/stripe-node/releases — `new Stripe(sk, { apiVersion: '2026-05-27.dahlia' })`; idempotency via `{ idempotencyKey }` options arg.
- **Stripe Checkout → release wiring** — escrow uses **separate charges & transfers**: Checkout `mode:'payment'` collects funds to the platform balance now (escrow held); on client approval, create a `Transfer` to the connected account for (amount − fee) with an idempotency key. Gives true milestone-release timing control vs destination charges.
- **Webhook ingestion (Express 5)** — mount `express.raw({type:'application/json'})` on `/v1/webhooks/stripe` only, before the global `express.json()`; verify signature, then process.
- **Resend** — transactional email via outbox drain worker (never synchronous in a request handler).
- **Sentry** — `@sentry/node` (API) + `@sentry/nextjs` (web) for error reporting per the `error_reporting: Sentry` constraint.

## §3.9 Implementation Patterns

1. **Double-entry bookkeeping in SQL** — immutable append-only `ledger_entries` (one signed row per leg), each transaction's legs sum to zero, balances derived as `SUM(amount)` over entries (never a mutable stored balance). Refs: Square *Books* (https://developer.squareup.com/blog/books-an-immutable-double-entry-accounting-database-service/), Modern Treasury *Accounting for Developers*.
2. **Outbox pattern for dual-write** — write business row + outbox row in one ACID transaction; a relay drains the outbox into the second system (email/webhook side-effects). Refs: https://microservices.io/patterns/data/transactional-outbox.html, https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html.
3. **Idempotency keys with UNIQUE constraints** — `UNIQUE(idempotency_key)`; store-and-check atomically so retried fund/release/payout calls execute once. Refs: Stripe idempotency, Brandur Leach.
4. **Escrow state machine** — enumerated states with guarded one-way transitions; each transition emits balanced ledger postings. `milestone: pending_acceptance → accepted → funded(held) → delivered → approved(released) | disputed → resolved(refunded|released|split)`.
5. **Money as integer minor units** — `BIGINT amount_minor` + `currency CHAR(3)`; never float/`NUMERIC`-without-scale. Matches Stripe's integer cents. Refs: Fowler *Money* pattern (https://martinfowler.com/eaaCatalog/money.html).

---

# §4 Stack Candidates & Decision

| Layer | Chosen | Alternatives considered | Rationale |
|-------|--------|-------------------------|-----------|
| Frontend | **Next.js 16 (App Router) + Tailwind + React Context** | Remix, plain React+Vite | Hard constraint (`frontend_framework: Next.js`, `css_approach: Tailwind`, `state_management: React Context`). |
| Backend | **Express 5 + REST + URL `/v1/` versioning** | Fastify, NestJS | Hard constraint (`backend_framework: Express`, `api_style: REST`, `api_versioning: URL path`). |
| ORM / DB | **Prisma 7 + PostgreSQL** | Drizzle, Knex; MySQL | Hard constraint. Postgres gives serializable transactions, CHECK constraints, row locks — essential for ledger correctness. |
| Payments | **Stripe (Checkout + Connect Express, separate charges & transfers), test mode** | Braintree, Adyen | Hard constraint (`webhook_targets: Stripe`). Separate-charges model gives milestone-release timing. |
| Email | **Resend via outbox + drain worker** | Postmark, SES | Hard constraint (`email_service: Resend`, `notification_urgency: guaranteed delivery` → outbox). |
| Errors | **Sentry** (`@sentry/node`, `@sentry/nextjs`) | Rollbar, Bugsnag | Hard constraint (`error_reporting: Sentry`). |
| Auth | **Session cookie (httpOnly, SameSite=Lax) + scrypt KDF + HIBP breach screen** | JWT, Auth.js | `auth_model` blank → chose server sessions: simplest correct multi-role model, easy CSRF posture, no client token storage. ADR-002. Password verifier = Node scrypt (ADR-012). |
| Container | **Single Docker Compose** (web, api, worker, postgres) | k8s | Hard constraint (`container_strategy: single container (Docker Compose)`), `scale: small`. |
| Scale tier | **small (<1k concurrent)** | — | Hard constraint. No Redis/replicas; framework default pool; inline + cron worker. `has_payments` signal bumps concurrency-safety (row locks) above tier baseline. |

# §5 Risk Register & Threat Model

### Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|-----------|
| R1 | Ledger imbalance (debits ≠ credits) | Med | Critical | Balanced postings in one serializable tx; app-level assertion Σ=0 + reconciliation job; never store mutable balances. |
| R2 | Negative / over-drawn balance; payout exceeds available | Med | Critical | Balances derived from ledger; pre-payout `SELECT … FOR UPDATE` asserting available ≥ amount; reject otherwise. |
| R3 | Double-release of a milestone | Med | Critical | One-way state machine + row lock + UNIQUE(milestone_id, kind) on release ledger transaction. |
| R4 | Webhook replay / spoofing | High | High | Verify `Stripe-Signature` before any DB read; 5-min tolerance; persist `event.id` UNIQUE to dedupe (≥72h retention). |
| R5 | Race on concurrent release / payout | Med | Critical | Serializable tx + pessimistic locks on account rows; Stripe idempotency keys on transfers/payouts. |
| R6 | Same-tenant IDOR (user A acts on B's resource) | High | High | `assertResourceAccess(row, principal)` chokepoint on every owned route + sub-resource; deny-by-default. |
| R7 | Cross-tenant leakage | Med | Critical | Scoped service wrappers; mandatory owner predicate; NOT_FOUND (not FORBIDDEN) on cross-tenant. |
| R8 | Secret leakage | Med | High | Env-only secrets; never logged/returned; gitleaks scan; test-mode keys only. |
| R9 | Dispute-resolution authz bypass | Med | High | Admin-only role gate; fund moves require audit entry; no unilateral buyer/seller fund move. |
| R10 | PCI scope creep | Low | Critical | PAN stays in Stripe (hosted Checkout); store token + last4/brand only; forbidden-pattern lint on PAN-like columns/logs. |

### STRIDE — Trust Boundaries

- **Browser ↔ API:** session auth + object-level authz on every request (R6/R7); server is sole authority on amounts/state (never trust client amounts); audit log per money action; **card data never crosses into Tradewind** — browser → Stripe hosted Checkout (keeps SAQ-A, R10).
- **API ↔ Stripe:** idempotency keys on all charges/transfers/payouts (R3/R5); secret keys server-side, test-mode, never logged (R8); pre-transfer balance assertion (R2).
- **Stripe webhook ↔ API:** mandatory HMAC verify **before** DB read (R4); timestamp tolerance + persisted `evt_id` dedupe (at-least-once → idempotent handlers); store raw event for audit; respond 2xx fast, money work in atomic idempotent tx.
- **API ↔ DB:** least-privilege role; tenant/owner predicate (R7); money moves in serializable/locked transactions with Σ=0 and available≥0 enforced in app + verified by reconciliation; append-only immutable ledger (corrections via reversing entries only).

**Cross-cutting invariants (encoded in invariants.json §9):** (a) every transaction's ledger legs sum to zero; (b) no account available balance goes negative; (c) a milestone releases at most once; (d) total of all balances is conserved by every operation (closed system: external + escrow + payable + fee + payout accounts net to zero); (e) no payout exceeds available balance; (f) webhook signature verified before DB read; (g) money never stored as float.

# §6 Research Gaps

- **Live Stripe Connect onboarding** (real identity verification, payouts to real banks) is out of scope — test mode only (Non-Goal). Real KYC/AML/money-transmitter licensing deferred (ADR in DECISIONS.md).
- **Multi-currency** — build assumes a single platform currency (USD) with `currency` column present for future expansion; FX not specified. Documented assumption.
- **Idempotency-Key retention exact window** on Stripe's reference page not re-quoted verbatim; we retain webhook event ids ≥30 days (well past the 3-day retry window) to be safe.

# §7 Summary & Go/No-Go

Tradewind is a well-scoped fintech reference build with hard constraints fully specified and a mature,
verified stack (Stripe Connect test mode, Prisma 7 + Postgres, Express 5, Next.js 16). The hard part —
money correctness — is addressable with established patterns (immutable double-entry ledger, separate
charges & transfers, idempotency keys, serializable transactions, outbox for email/dual-write). PCI
scope is contained at SAQ-A by never touching PANs. No blocking unknowns; the research gaps are scoped
deferrals, not blockers.

**Go/No-Go: GO.** Sources: 14 vendor/official · 6 GitHub · 15 other (articles, standards, threads, videos, patterns).
