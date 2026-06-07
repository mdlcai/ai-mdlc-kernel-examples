# RESEARCH.md — Ember

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "e-commerce"

## Domain Signals

```yaml
domain_signals: ["has_image_uploads", "has_webhooks", "has_payments"]
```

## Product Vision
**Problem:** Independent roasters sell exceptional product through
    clunky, generic storefronts that load slowly, look identical to
    everyone else, and bury the story behind the beans. They lose
    direct sales to marketplaces that take 15–30% and own the customer
     relationship.

**Who it affects:** Specialty-coffee drinkers buying 1–4 bags/month and
  gift shoppers; plus the roaster's own staff managing catalog,
    inventory, and orders. Not wholesale/B2B buyers.

**Why existing solutions fall short:** Off-the-shelf themes look the same and nickel-and-dime
    through paid apps; marketplaces take a large cut and hide the
    customer; generic builders can't express a roaster's brand or
    surface the tasting/brew content that actually drives conversion.

**Solution:** Ember is a direct-to-consumer online store for a
    specialty coffee roaster. Customers browse a catalog of
    single-origin and blend coffees, read tasting notes and brew
    guides, add bags to a cart, and check out securely with Stripe —
    with an optional subscription that ships their favorite roast on
  a schedule. It replaces the slow, generic template storefronts
  most small roasters run with a fast, beautiful, conversion-focused
    shop they fully own.


## Users & Outcomes
**Key Workflows:**
As a shopper, I browse the catalog and open a product
  to read tasting notes, then add a bag to my cart and check out via
    Stripe in under 2 minutes. As a shopper, I subscribe to a
    recurring monthly shipment of a roast. As a returning customer, I
    view my order history and reorder in one click. As staff, I add a
    product with photos, price, and stock level. As a shopper, I
    receive an order-confirmation email after a successful payment.

**Success Metrics:**
LCP < 1.5s on product pages; Lighthouse accessibility ≥ 95; checkout completes in < 3s; stock never goes negative under concurrent purchase; supports 500 concurrent shoppers; 99.9% uptime.

**Non-Goals:**
No wholesale/B2B portal; USD only (no multi-currency in
    v1); single-vendor (not a multi-roaster marketplace); no native
    mobile app; no in-house card processing (Stripe only); product
    reviews and gift cards deferred to v2.


## Build Constraints

```yaml
# Infrastructure & Ops
protocol_support: "HTTPS only"
ci_cd_required: true

# Data & Storage
database_preference: "PostgreSQL"
pii_handling: ["customer_email", "customer_name", "shipping_address", "payment_method_token"]

# Notifications & Messaging
email_service: "Resend"
notification_urgency: "guaranteed delivery"

# Security & Compliance
security_baseline: ["PCI-DSS"]
rate_limiting: true
audit_logging: true

# Frontend
frontend_framework: "Next.js"

# Backend
backend_framework: "Express"
api_style: "REST"
api_versioning: "none for v1"
orm_preference: "Prisma"

# Performance & Quality
performance_requirements: ["LCP < 1.5s", "checkout < 3s", "API response < 200ms"]
testing_strategy: "test-after"
logging_format: "structured JSON"

# Scope & Platform
scale: "medium — 1k-50k"
target_platforms: ["web"]
```

## Design Language

### Archetype
Archetype: ecommerce

This product's design archetype is **ecommerce**. Read `DESIGN.md` Part II §`ecommerce` (fetched alongside BUILD.md from the MDLC kernel) and treat its Layout Doctrine, Density, Type System, Color & Atmosphere, Motion Budget, and Signature Components as binding requirements, and its Good-vs-Avoid list as the acceptance rubric. The token tables below are the resolved starting palette; an explicit brand override outranks them per the `DESIGN.md` precedence list. The Universal Excellence floor (`DESIGN.md` Part I) applies on top regardless of archetype.

### Brand Voice
Warm, crafted, quietly expert — a roaster who loves the beans but never lectures. Tasting
  ▎ notes read like an invitation, not a spec sheet; headlines are short and sensory. Lead
  ▎ with origin and flavor, then make the price and Add to cart unmistakable. Imagery carries
  ▎ the mood; copy stays unfussy — no hype, no jargon walls. Confident, calm, and a little
  ▎ romantic about good coffee.

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #2d1e09 | Buttons, links, active states |
| Secondary | #182b0b | Accents, badges, highlights |
| Accent | #0e1b2e | Callouts, hover states |
| Background | #fafafa | Page background |
| Surface | #f5f5f4 | Cards, elevated containers |
| Text | #1d1a16 | Headings, body text |
| Text Secondary | #70685c | Captions, muted text |
| Success | #1fad53 | Success states, confirmations |
| Warning | #ec9c13 | Warnings, pending states |
| Error | #df2020 | Errors, destructive actions |

### Color System — Dark Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #2d1e09 | Buttons, links, active states |
| Secondary | #81d147 | Accents, badges, highlights |
| Accent | #6390cf | Callouts, hover states |
| Background | #0c0a09 | Page background |
| Surface | #171512 | Cards, elevated containers |
| Text | #ecebea | Headings, body text |
| Text Secondary | #958e83 | Captions, muted text |
| Success | #33cc6b | Success states, confirmations |
| Warning | #e2a336 | Warnings, pending states |
| Error | #d74242 | Errors, destructive actions |

### Typography
- Heading: Outfit (600/700 weight)
- Body: Inter (400/500 weight)
- Mono: JetBrains Mono (code, pre, kbd)
- Base size: 16px, scale ratio: 1.25
- Scale: 10.2 / 12.8 / 16 / 20 / 25 / 31.3 / 39.1px

### Layout
- Pattern: Topnav + Content
- Max width: 1280px
- Spacing: Comfortable (12/16/24/32px)
- Breakpoints: 640 / 768 / 1024 / 1280px

### Component Style
- Variant: Minimal
- Border radius: 4px (sm: 0px, lg: 8px, xl: 12px)
- Shadows: None — `none`
- Theme: Light + Dark — ship both palettes with a runtime theme toggle that follows the user's system preference (`prefers-color-scheme`) and persists their explicit choice

### Accessibility
- WCAG AA compliance
- Lighthouse target: 90+
- Responsive breakpoints: 640 / 768 / 1024 / 1280px
- Reduced motion: Standard animations

---

# §3 — Source Categories

> Stage 0 research performed 2026-06-07 (build_depth: comprehensive). Every version,
> API signature, and standard below was verified against the vendor's / standards
> body's own current documentation on that date, not from training memory. Items that
> could not be vendor-confirmed are flagged inline.

## §3.1 Official / Vendor Documentation

| Tech | Canonical docs | Confirmed current version (2026-06-07) | License | Notes for Ember |
|------|----------------|----------------------------------------|---------|-----------------|
| **Next.js (App Router)** | https://nextjs.org/docs/app · Route Handlers: https://nextjs.org/docs/app/getting-started/route-handlers | **16.x** (16 GA 2025-10-21; 16.2 2026-03-18). React 19.2, Turbopack default. | MIT | App Router default & stable. Storefront UI + thin BFF/Stripe redirect pages live here; canonical REST stays on Express. ⚠️ Not 13/14. |
| **Express** | https://expressjs.com · v5 migration: https://expressjs.com/en/guide/migrating-5.html | **5.2.x** stable (5.2.0 2025-12-01). | MIT | v5 auto-forwards rejected async-middleware promises to the error handler; stricter path matching (no bare `*`). ⚠️ Use 5, not 4. |
| **Prisma ORM** | https://www.prisma.io/docs · Transactions: https://www.prisma.io/docs/orm/prisma-client/queries/transactions | **7.x** (7.8.0 reported on npm — patch not vendor-pinned, verify at install). | Apache-2.0 | Interactive txns `prisma.$transaction(async tx => …)` with `isolationLevel`. ⚠️ **No first-class `SELECT … FOR UPDATE`/pessimistic-lock API** — must issue it via `$queryRaw` *inside* an interactive transaction, or use conditional atomic decrement + isolation level. |
| **PostgreSQL** | https://www.postgresql.org/docs/ · versioning: https://www.postgresql.org/support/versioning/ | **18.x** stable (18 GA 2025-09-25, supported to 2030). 19 is Beta only (2026-06-04) — **do not use in prod**. | PostgreSQL License | Row locks (`FOR UPDATE`/`FOR NO KEY UPDATE`), advisory locks, `UNIQUE`/`CHECK` constraints — Ember's concurrency + idempotency primitives. |
| **Stripe** | https://docs.stripe.com · webhooks: https://docs.stripe.com/webhooks/signature · idempotency: https://docs.stripe.com/api/idempotent_requests | **API `2026-05-27.dahlia`**; **stripe-node v22.2.0** (pins that API version). | MIT | Pin explicitly: `new Stripe(key, { apiVersion: '2026-05-27.dahlia' })`. Hosted Checkout Sessions = least code, strongest PCI posture. ⚠️ Not API `2024-*`/SDK v16. |
| **Resend** | https://resend.com/docs · Node: https://resend.com/docs/send-with-nodejs · webhooks: https://resend.com/docs/dashboard/webhooks/event-types | **resend npm v6.12.x** (patch reported, not vendor-pinned). | MIT | `resend.emails.send({from,to,subject,html})` → `{data:{id}}`. Supports idempotency keys. Webhooks via Svix (`svix-id` header, at-least-once → dedupe). |
| **Docker Compose** | https://docs.docker.com/reference/compose-file/ · https://www.compose-spec.io/ | Compose Specification (v2/v3 schemas merged); Compose V2 plugin (`docker compose`). | Apache-2.0 | ⚠️ Top-level `version:` field is **obsolete/ignored** — omit it. Canonical filename `compose.yaml`. |

## §3.2 GitHub Repositories (reference implementations & libraries)
*(stars/last-active verified via GitHub on 2026-06-07)*

| Repo | URL | Stars | Last active | License | Relevance |
|------|-----|-------|-------------|---------|-----------|
| **yournextstore/yournextstore** | https://github.com/yournextstore/yournextstore | ~5.4k | 2026-06-06 | MIT | **Closest match** — Next.js App Router DTC storefront with payments/checkout **directly on Stripe** (not Shopify). Reference for Stripe Checkout wiring in App Router. |
| **vercel/commerce** | https://github.com/vercel/commerce | ~14.1k | 2026-02-02 | MIT | Next.js commerce **UX/cart architecture** reference. ⚠️ Now **Shopify-only** for payments — use for storefront patterns, *not* Stripe. |
| **vercel/nextjs-subscription-payments** | https://github.com/vercel/nextjs-subscription-payments | ~7.7k | 2025-01-23 | MIT | End-to-end **Stripe Subscriptions** + webhook→DB sync pattern. (Supabase-based; adapt data layer to Prisma.) |
| **stripe-samples/subscription-use-cases** | https://github.com/stripe-samples/subscription-use-cases | ~0.9k | 2026-05-01 | MIT | Stripe-authored **Node/Express** samples — authoritative raw-body + signature-verification webhook handler. |
| **antonio-lazaro/prisma-express-typescript-boilerplate** | https://github.com/antonio-lazaro/prisma-express-typescript-boilerplate | ~0.3k | 2024-06 | MIT | **Express + TS + Prisma** REST baseline (JWT, validation, logging, Docker). Port of `hagopj13/node-express-boilerplate` (~7.6k). |
| **prisma/prisma-examples** | https://github.com/prisma/prisma-examples | ~6.6k | 2026-06-07 | Apache-2.0 | Official REST+Express+TS schema/migration/transaction reference for atomic stock decrements. |
| **express-rate-limit/express-rate-limit** | https://github.com/express-rate-limit/express-rate-limit | ~3.3k | 2026-06-05 | MIT | Standard Express rate-limiter; pair with `rate-limit-redis` for multi-replica correctness. |
| **pinojs/pino** | https://github.com/pinojs/pino | ~17.9k | 2026-06-02 | MIT | De-facto JSON structured logger; `pino-http` for request logs with PII redaction. |

## §3.3 Videos / Tutorials
- **"I Mastered Stripe Subscriptions in Next.js 15 and You Can Too!"** — https://www.youtube.com/watch?v=NfeFXQwmDXA — App Router subscription flow: Checkout Sessions, DB sync, access gating. Maps to Ember's subscription tier.
- **Vercel `nextjs-subscription-payments` walkthrough** — https://github.com/vercel/nextjs-subscription-payments — production-grade webhook-as-source-of-truth reference (adapt off Supabase).
- **Pedro Alonso — "Stripe Subscriptions in a Next.js Application"** — https://www.pedroalonso.net/blog/stripe-subscriptions-nextjs/ — concise lifecycle-webhook + billing-portal checklist.

## §3.4 Articles / Blog Posts
- **"95+ Lighthouse Performance in a Next.js E-commerce Site"** — https://dev.to/seyedahmaddv/how-i-achieved-a-95-lighthouse-performance-score-in-a-nextjs-e-commerce-site-and-how-you-can-2pe5 — priority/`sizes` hero image, font strategy, JS deferral → Ember's LCP<1.5s.
- **"Next.js Image Optimization — Responsive Images for Ecommerce"** — https://prateeksha.com/blog/nextjs-image-optimization-responsive-ecommerce-portfolio — per-use-case quality tuning; product image is the LCP element.
- **Stigg — "Best practices … integrating Stripe webhooks"** — https://www.stigg.io/blog-posts/best-practices-i-wish-we-knew-when-integrating-stripe-webhooks — respond-fast, dedupe-by-event-id, monitor. Reliability backbone.
- **Checkly — "the one webhook to rule them all"** — https://www.checklyhq.com/blog/our-stripe-billing-implementation-the-one-webhook-to-rule-them-all/ — pragmatic webhook scoping for subscription billing.

## §3.5 Standards / RFCs / Specifications
- **PCI-DSS v4.0.1** — https://blog.pcisecuritystandards.org/important-updates-announced-for-merchants-validating-to-self-assessment-questionnaire-a · SAQ A PDF: https://listings.pcisecuritystandards.org/documents/PCI-DSS-v4-0-SAQ-A.pdf — Ember is **SAQ A eligible** (Stripe-hosted card capture; no PAN/CVV on Ember servers). v4.x future-dated reqs mandatory since 2025-03-31. SAQ A (Jan 2025) **removed 6.4.3/11.6.1/12.3.1**, replaced with a broader **whole-site script-attack eligibility attestation** → Ember needs CSP + script hygiene sitewide. Still required: TLS, no card data on servers/logs, strong admin auth.
- **NIST SP 800-63B-4** (final 2025-07-31) — https://csrc.nist.gov/pubs/sp/800/63/b/4/final · PDF https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-63b-4.pdf — min length ≥8 (≥12 for PCI-adjacent), accept ≥64, all printable chars, **breached-password screening via HIBP k-anonymity** (https://haveibeenpwned.com/API/v3#PwnedPasswords), no composition rules, no forced rotation, rate-limit before hash.
- **RFC 9457 — Problem Details for HTTP APIs** (obsoletes RFC 7807) — https://www.rfc-editor.org/rfc/rfc9457.html — `application/problem+json` error contract for all 4xx/5xx; no internals/PII in `detail`.
- **WCAG 2.2 AA** — https://www.w3.org/TR/WCAG22/ — checkout-critical SC: 1.4.3 contrast 4.5:1, 1.4.11 non-text 3:1, 2.4.7/2.4.11 focus visible+appearance, 2.4.12 focus-not-obscured, **2.5.8 target size ≥24px**, 3.3.2 labels, 1.3.5 autocomplete, 3.3.7 redundant entry, 3.3.8 accessible auth.
- **OWASP Top 10:2025** — https://owasp.org/Top10/2025/ · **ASVS 5.0** — https://owasp.org/www-project-application-security-verification-standard/ — drive engineering from ASVS L2 (PII + payments).
- **HSTS RFC 6797** — https://www.rfc-editor.org/rfc/rfc6797.html · Mozilla TLS (Intermediate, TLS 1.2+1.3) https://ssl-config.mozilla.org/ — `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`.

## §3.6 Competing Products / Prior Art
- **Shopify** — https://www.shopify.com — hosted SaaS + theme/app marketplace; ~2.9%+30¢ and a **0.2–2.0% surcharge for non-Shopify-Payments gateways**, stacking app fees, convergent generic themes. *Ember reaction:* own checkout (Stripe direct, no platform tax), win on speed + distinctive design.
- **Vercel/Next.js Commerce** — https://github.com/vercel/commerce — open-source Next.js storefront head (Shopify-backed default). *Ember:* borrow performance/component patterns, pair with Stripe + own backend.
- **Medusa** — https://medusajs.com — MIT self-hostable Node commerce engine. *Ember:* validates "own your stack, no platform tax"; reference for cart/order/fulfillment primitives; full weight unnecessary for a single-roaster catalog.
- **Trade Coffee** — https://www.drinktrade.com/ — taste-quiz personalization, flavor collections, flexible pause/skip, roasted-to-order, replacement guarantee. *Ember:* steal onboarding-by-taste & frictionless pause/skip, deliver with **one** roaster's cohesive brand story + tasting notes + brew guides. (Counterpoint: Atlas Coffee Club https://atlascoffeeclub.com/.)

## §3.7 Community Threads / Discussions
- **Medusa GitHub Discussion #5136** — https://github.com/medusajs/medusa/discussions/5136 — why go headless/custom vs Shopify; warning on rewriting the frontend repeatedly → lock Ember's architecture early.
- **HN — "the one webhook to rule them all"** — https://news.ycombinator.com/item?id=19608140 — Stripe doesn't guarantee event **order** or timing → design order-independent, idempotent handlers; webhooks (not redirects) are source of truth.
- **Duncan Mackenzie — "Handling duplicate Stripe events"** — https://www.duncanmackenzie.net/blog/handling-duplicate-stripe-events/ — persist `event.id` with UNIQUE, short-circuit (still 200) on repeat; dedupe edge cases on `data.object` id + `event.type`.

## §3.8 APIs / Integrations (verified surface)
**Stripe**
- One-time bag purchase → `stripe.checkout.sessions.create({ mode:'payment', line_items, success_url, cancel_url })` → PaymentIntent.
- Subscription → Checkout Session `mode:'subscription'` referencing recurring `Price` → `Subscription` + recurring `Invoice`.
- **Webhook events to handle:** `checkout.session.completed` (primary fulfillment trigger, check `payment_status==='paid'`), `checkout.session.async_payment_succeeded`, `payment_intent.succeeded`, `invoice.paid` / `invoice.payment_failed` (dunning), `customer.subscription.created|updated|deleted`.
- **Signature verification:** `stripe.webhooks.constructEvent(rawBody, 'Stripe-Signature', endpointSecret)` — **raw unparsed body** (`express.raw({type:'application/json'})` on the webhook route only).
- **Idempotency:** pass `{ idempotencyKey: <uuid> }` on POSTs; Stripe replays first response per key (~24h).
- **PCI:** Checkout collects card in a Stripe-domain iframe → SAQ A, *provided* Ember never adds custom PAN fields.

**Resend**
- `POST https://api.resend.com/emails` (SDK `resend.emails.send`), server-side bearer key only.
- Webhook events (prioritize): `email.bounced`, `email.complained`, `email.delivery_delayed` (suppression/deliverability). Svix `svix-id` dedupe; HMAC via `svix-signature`/`svix-timestamp` — confirm verifier snippet at impl time.

## §3.9 Design / Architecture Patterns
- **A. No-oversell stock:** conditional atomic decrement `UPDATE products SET stock = stock - :qty WHERE id=:id AND stock >= :qty` (reject if 0 rows) + DB `CHECK (stock >= 0)` backstop; use `SELECT … FOR UPDATE` (raw, inside interactive `$transaction`) for hot multi-row read-then-decide. Sources: https://leapcell.io/blog/implementing-concurrent-control-with-orm-a-deep-dive-into-pessimistic-and-optimistic-locking , https://learning-notes.mistermicheels.com/data/sql/optimistic-pessimistic-locking-sql/
- **B. Idempotent webhooks:** verify signature → `INSERT event.id INTO processed_webhook_events` (UNIQUE) in same txn as side-effects → on conflict short-circuit 200. Sources: https://hookdeck.com/webhooks/guides/implement-webhook-idempotency , https://docs.stripe.com/webhooks
- **C. Outbox for guaranteed email:** insert into `pending_emails` in the same DB txn as the order/webhook; a drain worker calls Resend (with idempotency key from outbox row id), capped backoff → DLQ. Never send inline. Sources: https://microservices.io/patterns/data/transactional-outbox.html
- **D. Webhook-as-source-of-truth:** success/redirect page is UX only; order flips to paid **only** in the verified webhook handler on `payment_status==='paid'`. Sources: https://docs.stripe.com/checkout/fulfillment
- **E. Order modeling w/ price snapshot:** mutable `cart`/`cart_item` → live products; immutable `order`/`order_item` snapshots `product_name`,`product_sku`,`unit_price`,`quantity`,`line_total` + shipping/billing address at purchase; money as integer **cents**; `payment` row holds Stripe `payment_intent`. Sources: https://erflow.io/en/blog/ecommerce-database-schema

---

# §4 — Stack Candidates & Decision

All architecture-defining constraints in §"Build Constraints" are **filled** (hard constraints), so the stack is largely determined; §4 records the resolved choices and the few open library decisions.

| Layer | Chosen | Rationale / alternatives considered |
|-------|--------|-------------------------------------|
| Frontend | **Next.js 16 (App Router, TypeScript)** | Constraint-fixed. RSC + `next/image` for LCP<1.5s; Tailwind for token system. Alt (Remix/Astro) rejected — constraint says Next.js. |
| Backend | **Express 5 (TypeScript)** | Constraint-fixed. v5 async error handling. Hosts canonical REST API + Stripe/Resend webhooks + outbox worker. |
| ORM / DB | **Prisma 7 + PostgreSQL 18** | Constraint-fixed. Interactive transactions + raw `FOR UPDATE` for stock; UNIQUE for webhook idempotency. |
| Payments | **Stripe (hosted Checkout + Billing), stripe-node v22, API `2026-05-27.dahlia`** | SAQ A posture; subscriptions native; webhook source-of-truth. |
| Email | **Resend (v6) via transactional outbox** | Constraint-fixed (`email_service: Resend`, `guaranteed delivery` → outbox + drain worker). |
| Auth | **Session cookie (HttpOnly, Secure, SameSite=Lax) + Argon2id password hashing** | Blank field (ADR) — first-party email/password accounts; no third-party IdP in v1. Argon2id per NIST 800-63B. |
| Rate limiting | **express-rate-limit + rate-limit-redis** | Constraint `rate_limiting: true`; Redis store for multi-replica correctness. |
| Logging | **pino / pino-http (structured JSON, PII-redacted)** | Constraint `logging_format: structured JSON`, `audit_logging: true`. |
| Cache/queue store | **Redis** | Rate-limit store + outbox/session support; runs in compose. |
| Container | **Docker Compose (web=Next.js, api=Express, worker=outbox drain, db=Postgres 18, redis)** | Constraint `ci_cd_required`, HTTPS-only via reverse proxy (Caddy/Nginx). |
| Validation | **Zod** | Shared request/response schemas → RFC 9457 field-level errors. |
| Testing | **Vitest + Supertest (api), Playwright (e2e), test-after** | Constraint `testing_strategy: test-after`. |

# §5 — Risk Register & Threat Model (STRIDE)

| # | Threat (STRIDE) | Severity | Description | Mitigation (binds to SPEC §4 / ARCH §9) | Source |
|---|-----------------|----------|-------------|------------------------------------------|--------|
| R1 | Webhook spoofing / payment fraud (S,T) | **Critical** | Forged `checkout.session.completed` marks unpaid orders paid. | `constructEvent` signature verify on raw body; per-endpoint signing secret in secrets; fulfill only on verified `payment_status==='paid'`; idempotent on `event.id`. | https://docs.stripe.com/webhooks/signature |
| R2 | Overselling / stock race (T, TOCTOU) | **High** | Concurrent checkouts oversell last unit. | Conditional atomic decrement in txn + `CHECK(stock>=0)`; `FOR UPDATE` for hot reads; deterministic lock order by `id ASC`; racing integration test. | OWASP A06 Insecure Design |
| R3 | Account enumeration (I) | **Medium** | Signup/login/reset reveal email existence by shape or timing. | Uniform response shape + timing (pad/per-IP limit); generic reset copy; same envelope for unknown-email/wrong-password/rate-limited. | OWASP Auth Cheat Sheet |
| R4 | Broken access control / IDOR (E,I) | **Critical** | `GET /orders/{id}` leaks another customer's order/PII. | Scope every query by session `user_id`; cross-tenant → `NOT_FOUND` not `FORBIDDEN`; ULID ids as defense-in-depth. | OWASP A01 |
| R5 | PII exposure / at-rest (I) | **High** | Email/name/address/token leak via breach, backup, logs. | TLS 1.2+ in transit; AES-256 at rest (DB/disk/backup, gate delegated-infra honestly); never log PII/tokens; audit-log PII reads. | GDPR Art.32 |
| R6 | Brute force / credential stuffing (S) | **High** | Automated login guessing. | Rate-limit **before** Argon2 hash; per-(ip,email) throttle/lockout; HIBP breach screen; optional MFA. | NIST 800-63B-4 |
| R7 | CSRF on state-changing endpoints (T) | **Medium** | Forged cross-site order/address/email change. | SameSite=Lax + Secure + HttpOnly cookies; double-submit CSRF token on non-GET; Origin check. | OWASP CSRF Cheat Sheet |
| R8 | Secrets exposure (I,E) | **Critical** | Stripe `sk_*`/signing secret/DB creds leak via repo, env dump, client bundle. | Secrets in env/secret manager, gitignored `.env`, committed `.env.example`; never in client bundle; least-privilege DB user; rotate on suspicion. | OWASP A02 |
| R9 | Supply-chain / Magecart (T) | **High** | Malicious dep or injected checkout script. | Lockfile + `npm audit`/SCA in CI; CSP + SRI; minimize third-party JS on checkout (also satisfies SAQ A script attestation). | OWASP A03 |
| R10 | Email outbox poisoning / dual-write loss (T,I) | **Medium** | Order commits but confirmation email lost, or duplicate sends. | Transactional outbox; idempotency key per email; capped backoff → DLQ; never synchronous Resend call in request path. | microservices.io outbox |

**Cross-cutting:** audit-log auth events, payment/webhook events (by Stripe event id), PII access, authz failures — **without** writing PII/secrets (OWASP A09). Threat-model assessment satisfies comprehensive-depth §5 requirement.

# §6 — Research Gaps / Open Items
1. **Prisma exact patch (7.8.0)** and **Resend SDK patch (6.12.x)** reported via aggregators, not vendor-pinned — confirm at `npm install` time and pin in lockfile. *(non-blocking)*
2. **Resend Svix HMAC verifier snippet** — confirm exact header names + verification library call against Svix docs before coding the inbound webhook verifier. *(non-blocking; Stage 3)*
3. **Reverse-proxy TLS choice** (Caddy auto-HTTPS vs Nginx + Certbot) — decide in ARCHITECTURE; both satisfy HTTPS-only. *(resolved in Stage 1)*
4. **HIBP API** is keyless for the range endpoint but rate-limited — confirm acceptable throughput for signup volume (medium scale 1k–50k). *(non-blocking)*
5. No raw-card or 3DS custom flow in scope (Stripe hosts) — keeps SAQ A; revisit only if embedded Elements is later required.

# §7 — Summary & GO / NO-GO
**Summary:** Ember is a well-bounded single-roaster DTC storefront on a fully constraint-specified, current stack (Next.js 16 · Express 5 · Prisma 7 · PostgreSQL 18 · Stripe hosted Checkout/Billing · Resend). All high-risk areas have established, verified mitigations: webhook-as-source-of-truth + signature verification (R1), atomic stock decrement + CHECK constraint (R2), session auth with NIST-aligned password handling (R3/R6), per-user query scoping (R4), transactional outbox for guaranteed email (R10), and SAQ A PCI posture by keeping all card capture in Stripe's hosted page. No unresolved blocking unknowns; §6 items are non-blocking and scheduled into later stages.

**Go/No-Go: GO** — research complete, stack verified current, threat model established, no blockers.
