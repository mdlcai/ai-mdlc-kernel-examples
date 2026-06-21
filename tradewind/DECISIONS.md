# DECISIONS.md — Tradewind

Architecture Decision Records (ADRs), assumptions, deviations, and technical debt. Precedence:
RESEARCH > ARCHITECTURE > SPEC.

---

## Review gate outcomes

- **Stage 0 (Research):** `force_research: false`, §3 absent → research performed; Go/No-Go = **GO**. Logged.
- **Stage 1 (Architecture):** `review_gates: auto` → self-verify and proceed. Logged.
- **Stage 2 (Spec):** `review_gates: auto` → self-verify and proceed.
- **Multi-Agent Plan Gate:** `review_gates: auto` → summarize, log, auto-dispatch.

---

## ADR-001 — Single platform currency (USD), currency column retained
**Status:** Accepted. **Context:** RESEARCH §6 notes multi-currency/FX is unspecified and a research gap.
**Decision:** All amounts are integer **minor units (cents)** with a `currency CHAR(3)` column fixed to
`USD` for this build. The column exists so future multi-currency is additive, but no FX conversion is
implemented. **Consequence:** ledger math is single-currency; mixing currencies in one transaction is
rejected at the service layer.

## ADR-002 — Auth model: server session cookies (auth_model was blank)
**Status:** Accepted (key-decision default, blank field). **Context:** `auth_model` not specified.
**Decision:** httpOnly + SameSite=Lax + Secure session cookie backed by a `sessions` table; password
hashing with **bcrypt cost 12**; **HIBP k-anonymity** breach screening + **15-char minimum** per NIST
800-63B (stricter than PCI v4's 12). No JWT (avoids client token storage / revocation complexity at
`scale: small`). **Consequence:** state-mutating routes need a CSRF/Origin guard (SameSite=Lax + Origin
check). Logged per Blank Field Policy.

## ADR-003 — Escrow via Stripe "separate charges and transfers", test mode
**Status:** Accepted. **Context:** Stripe has no escrow accounts; destination charges transfer funds
immediately, defeating milestone-gated release (RESEARCH §3.1, §3.8). **Decision:** Funds are collected
to the **platform balance** via Checkout (`mode: 'payment'`) at funding time (escrow held); on client
approval the platform creates a **Transfer** to the freelancer's connected account for (amount − fee)
with an idempotency key. **Consequence:** the platform balance is the real custodian; the internal
double-entry ledger mirrors it; reconciliation compares the two.

## ADR-004 — Ledger balances are derived, never stored
**Status:** Accepted. **Context:** Stored balances drift (RESEARCH §3.9, R1). **Decision:** `ledger_entries`
is append-only and immutable; an account's balance is `SUM(amount)` over its entries. Corrections are
reversing entries, never updates/deletes. **Consequence:** balance reads aggregate; acceptable at
`scale: small`. A materialized `account_balances` view MAY cache sums but is never the source of truth.

## ADR-005 — Closed-system chart of accounts
**Status:** Accepted. **Decision:** Accounts: `EXTERNAL` (the outside world / Stripe), `ESCROW_HELD`
(per project/milestone funds in custody), `FREELANCER_PAYABLE` (per freelancer available balance),
`PLATFORM_FEE` (platform revenue), `PAYOUT_CLEARING` (funds moving out to a connected account). Every
movement nets to zero across these. **Consequence:** invariant (d) — the sum of all account balances is
always zero (a closed system); any nonzero total is a reconciliation failure.

## ADR-006 — Idempotency at the database layer for webhooks and money mutations
**Status:** Accepted. **Decision:** `webhook_events(stripe_event_id UNIQUE)` inserted **before**
processing; money-mutation transactions carry a UNIQUE `idempotency_key` (derived from the business
action). Signature verification runs **before** any DB read on the webhook route. **Consequence:**
at-least-once Stripe delivery yields exactly-once business effect (R3/R4).

## ADR-007 — Outbox pattern for email + dual-write (has_dual_write, has_email)
**Status:** Accepted. **Decision:** Notification emails are inserted into a `pending_emails` outbox in
the **same transaction** as the triggering business write; a drain worker delivers via Resend with
capped backoff (1m/10m/1h → dlq). No synchronous provider calls in request handlers. **Consequence:**
guaranteed-delivery semantics; email loss decoupled from request latency (`notification_urgency:
guaranteed delivery`).

## ADR-008 — Local/dev at-rest encryption delegated to deploy infrastructure
**Status:** Accepted (dev-only gate per Compliance Baseline). **Context:** PCI-DSS/GDPR at-rest
encryption. **Decision:** App stores only Stripe tokens + last4/brand (no PAN/CVV ever). Column-level
encryption is **not** applied to profile PII in the dev stack; production at-rest encryption is
delegated to the cloud volume/RDS (KMS/TDE). **Consequence:** COMPLIANCE.md marks at-rest encryption
**⚠ (delegated)**, never ✓; QUICKSTART documents the production requirement.

## ADR-009 — Container & worker topology
**Status:** Accepted. **Decision:** Docker Compose with services: `postgres`, `api` (Express),
`web` (Next.js), `worker` (outbox/reconciliation cron). `scale: small` → no Redis, no queue broker;
the worker polls Postgres. **Consequence:** within-tier; documented in ARCHITECTURE §6.

## ADR-010 — Node http header buffer sizing
**Status:** Accepted. **Decision:** Both `api` and `web` start scripts pass
`--max-http-header-size=32768` (per BUILD.md Environment & Startup Artifacts) to survive accumulated
foreign cookies on a shared host. **Consequence:** no 431 on first request.

---

## Assumptions summary (blank fields resolved by the agent — Stage 3 pre-build review)
- `auth_model` → server session cookies (ADR-002)
- Platform currency → USD (ADR-001)
- Platform fee → **10%** of milestone amount (configurable via `PLATFORM_FEE_BPS=1000`), rounded to
  integer cents with remainder favoring the platform fee leg (documented in SPEC §5 W5).
- Connect account type → **Express** (test mode).

---

## Multi-Agent Plan Gate outcome (review_gates: auto)
Dispatched automatically. Plan: 15 features in 5 dependency-ordered waves (foundation → auth+ledger →
escrow → payout+reporting → frontend). Waves are dependency-coupled (shared Prisma client, services,
error contract) → built sequentially in one orchestrating context; a fresh-context sub-agent is reserved
for the Reviewer Gate (mandated independence). Per-feature bundle obligations enumerated in SPEC §5/§6
(endpoints), §3 (tables), §8 (UI). No bundled-ADR feature-dropping risk — each feature maps 1:1 to
SPEC entries.

## ADR-011 — Pinned dependency versions reflect the build environment registry
**Status:** Accepted (dependency reconciliation). **Context:** RESEARCH §3 projected June-2026 majors
(Prisma 7, stripe-node 22, Next 16). The build environment's npm registry resolves the current
published majors: **Express 5.2.1, stripe 18.5.0, Prisma 6.19.3, zod 3.25.76, pino 9.14, @sentry/node
8.55, Next 15.x**. **Decision:** pin to the actually-available versions — a build that installs and runs
outranks an aspirational version number; the ORM/framework/payment-rail choices (Prisma/Express/Stripe/
Next) are unchanged, only the patch/major numbers. No security primitive swapped. **Consequence:** code
uses Prisma 6 (`prisma-client-js` generator) and stripe v18 API surface, both API-compatible with the
patterns in ARCHITECTURE.

## ADR-012 — Password verifier: Node scrypt (not bcrypt native)
**Status:** Accepted. **Context:** native `bcrypt` needs node-gyp compilation that is unreliable on
Windows/CI; `bcryptjs` would be a forbidden security-primitive downgrade. **Decision:** use Node's
built-in `crypto.scrypt` (N=16384, r=8, p=1, 64-byte key, per-user 16-byte salt) — a memory-hard,
OWASP-recommended KDF in the standard library, a deliberate first choice rather than a friction
substitution. Constant-time compare via `crypto.timingSafeEqual`. **Consequence:** SPEC SEC-1 reads
"scrypt" not bcrypt; no native module; same security posture. NIST 800-63B controls (HIBP screen,
15-char min, no composition/rotation) unchanged.

## ADR-013 — INV-1 exclusion globs refined for nested prisma + test infra
**Status:** Accepted (invariant amendment). **Context:** the initial INV-1 `outside` globs used root
`prisma/**`, but the schema/seed live at `apps/api/prisma/**`, and test utilities (`apps/api/src/test/**`)
legitimately use the raw client. **Decision:** broaden `outside` to `apps/api/prisma/**` +
`apps/api/src/test/**`. This is a correctness fix to the exclusion list, NOT a weakening — the production
service boundary (routes/middleware never call scoped `prisma.<model>`) is still fully enforced.
ARCHITECTURE §9 INV-1 and invariants.json updated together.

## ADR-014 — Security Audit residuals dispositioned
**Status:** Accepted. **Decision:** Auto-remediated all CRITICAL/HIGH/MEDIUM dependency findings:
@sentry/node 8→10.59 (OTel CVEs), vitest 2→3 + `npm audit fix` (dev-toolchain vite/esbuild/vitest CVEs,
including the dev-only CRITICAL vitest-UI + HIGH vite path-traversal). API now 0 vulns. **Two MEDIUM
residuals DEFERRED:** postcss <8.5.10 bundled inside Next 15 — build-time-only XSS in CSS stringify, not
a runtime path; the only fix is a breaking Next→v9 downgrade. Tracked in SECURITY-AUDIT.md F-A3; remove
when Next bumps its bundled postcss. No security primitive swapped; scrypt unchanged. See SECURITY-AUDIT.md.
