# ARCHITECTURE.md — Ember

> Stage 1 architecture. Derived from RESEARCH.md (precedence: RESEARCH > ARCHITECTURE > SPEC).
> Build version 0.1.0 · 2026-06-07. Design archetype: **ecommerce** (DESIGN.md Part II).

## 1. System Architecture

### 1.1 Topology
Ember is a containerized multi-service monorepo. A reverse proxy terminates TLS (HTTPS-only)
and routes to two app surfaces; a background worker drains the email outbox; Postgres and Redis
are the stateful backing services.

```
                         Internet (HTTPS only)
                                │
                    ┌───────────▼───────────┐
                    │  Caddy reverse proxy   │  TLS 1.2/1.3, HSTS, HTTP→HTTPS 301
                    │  (auto-HTTPS / mkcert) │  security headers, CSP
                    └─────┬───────────┬──────┘
              /  (storefront)         /api/*  (REST)
                    │                     │
        ┌───────────▼──────┐   ┌──────────▼───────────┐      ┌────────────────────┐
        │  web  (Next.js16)│   │  api  (Express 5/TS) │◀────▶│ worker (outbox      │
        │  App Router, RSC │   │  REST + Stripe/Resend│      │ drain, same image)  │
        │  next/image, SSR │   │  webhooks, auth, cart│      └─────────┬──────────┘
        └───────┬──────────┘   └──────┬────────┬──────┘                │
                │ fetch /api          │        │                       │
                │                     │        └───────────────────────┤
                ▼                     ▼                                 ▼
          (browser)            ┌──────────────┐                 ┌──────────────┐
                               │ PostgreSQL18 │                 │   Redis      │
                               │ (Prisma 7)   │                 │ rate-limit,  │
                               └──────┬───────┘                 │ sessions     │
                                      │                         └──────────────┘
                              ┌───────▼────────┐
                              │ Stripe / Resend│ (external; webhooks inbound to api)
                              └────────────────┘
```

### 1.2 Layers & modules
- **Presentation (`apps/web`, Next.js 16 App Router):** server-rendered storefront. Routes:
  catalog (`/`, `/coffee`, `/coffee/[slug]`), cart drawer, checkout entry, account
  (`/account`, `/account/orders`, `/account/subscriptions`), auth (`/login`, `/register`),
  staff admin (`/admin/*`), brew guides (`/brew/[slug]`), order confirmation
  (`/order/[id]`). UI consumes the Express REST API. The landing/home surface ports the
  uploaded **DESIGN-TEMPLATE.html** scaffold; all screens use the copied token layer.
- **API (`apps/api`, Express 5 + TypeScript):** the canonical REST surface and trust boundary.
  Modular by domain: `auth`, `catalog`, `cart`, `checkout`, `orders`, `subscriptions`,
  `webhooks` (stripe, resend), `admin`, `health`. Cross-cutting middleware: session auth,
  rate limiter, CSRF, request validation (Zod), structured logging (pino-http), error mapper
  (RFC 9457).
- **Worker (`apps/api/src/worker`, same image):** polls `pending_emails` outbox, sends via
  Resend with idempotency key, capped backoff → DLQ. Runs as a separate compose service
  (`command: node dist/worker.js`).
- **Data (`apps/api/prisma`):** Prisma 7 schema + migrations against PostgreSQL 18. Single
  source of schema truth; client generated into the api service.
- **Shared (`packages/shared`):** Zod DTO schemas, error codes, money helpers (integer cents),
  shared TypeScript types consumed by both `web` and `api`.

### 1.3 Request / data flow (browse → buy)
1. Shopper hits `/coffee/[slug]` (RSC) → `web` server-fetches `GET /api/products/:slug` → renders.
2. Add to cart → `POST /api/cart/items` (session-scoped cart in Postgres) → optimistic UI.
3. Checkout → `POST /api/checkout/session` → api **reserves stock** (conditional atomic
   decrement in an interactive transaction), creates a pending `order`, creates a Stripe
   Checkout Session (`mode: payment|subscription`, idempotency key) → returns redirect URL.
4. Shopper pays on **Stripe-hosted** page (no card data touches Ember → SAQ A).
5. Stripe → `POST /api/webhooks/stripe` → **verify signature on raw body** → **idempotency
   insert** (`processed_webhook_events.event_id` UNIQUE) → on `checkout.session.completed`
   with `payment_status==='paid'`: mark order paid, finalize stock, **insert order-confirmation
   row into `pending_emails` outbox in the same transaction**.
6. Worker drains outbox → Resend sends confirmation email.
7. Success page `/order/[id]` reads Ember's own DB (never trusts the redirect as proof of payment).

## 2. Agent & Tool Orchestration Design
Ember is a conventional web application, not an AI-agent system — there is no LLM agent or
tool-calling runtime in the product. "Orchestration" here = service orchestration (compose) +
the async outbox worker + Stripe/Resend webhook choreography. This section is intentionally
minimal per the product's nature (logged: ADR-009). The MDLC build pipeline's own agents are
out of product scope.

## 3. Design Patterns
- **Webhook-as-source-of-truth** (§3.9-D): payment/subscription state mutates only inside the
  verified webhook handler, never from the client redirect.
- **Transactional outbox** (§3.9-C): order/webhook txn inserts `pending_emails`; worker drains.
  Removes the dual-write hazard; guarantees at-least-once email (`notification_urgency: guaranteed`).
- **Idempotent consumer**: Stripe `event.id` and Resend `svix-id` deduped via UNIQUE constraint;
  webhook order is `verify_signature() → check_idempotency() → process_event()`.
- **Conditional atomic decrement + CHECK constraint** (§3.9-A): `UPDATE products SET stock =
  stock - :qty WHERE id = :id AND stock >= :qty` (reject on 0 rows) backstopped by
  `CHECK (stock >= 0)`; deterministic lock order (`ORDER BY id ASC`) for multi-row `FOR UPDATE`.
- **Immutable order snapshot** (§3.9-E): `order_item` copies name/sku/unit_price at purchase;
  money as integer cents.
- **Repository scoping / row-level tenancy**: all customer-owned reads/writes go through a
  `scopedRepo(userId)` wrapper; cross-customer access returns `NOT_FOUND`, never `FORBIDDEN`.
- **Problem Details (RFC 9457)**: all 4xx/5xx return `application/problem+json` with field-level
  validation detail; no stack traces / PII.
- **Token-driven design system**: one `:root` token layer (copied verbatim from DESIGN-TEMPLATE)
  consumed via Tailwind theme + CSS variables; no hard-coded colors in components.

## 4. Dependencies (libraries, APIs, versions)
| Dependency | Version (verified 2026-06-07) | Role |
|------------|------------------------------|------|
| next | 16.x | Storefront (App Router, RSC, next/image) |
| react / react-dom | 19.2.x | UI runtime |
| tailwindcss | 4.x | Token-driven styling |
| express | 5.2.x | REST API + webhooks |
| typescript | 5.x | Both apps |
| prisma / @prisma/client | 7.x | ORM + migrations (PostgreSQL 18) |
| stripe (stripe-node) | 22.2.x | API `2026-05-27.dahlia` (pinned) |
| resend | 6.x | Transactional email |
| svix | latest | Resend webhook verification |
| zod | 3.x | Validation / DTOs |
| argon2 | latest | Password hashing (Argon2id) |
| express-rate-limit + rate-limit-redis | latest | Rate limiting (Redis store) |
| ioredis | latest | Redis client (rate-limit, sessions) |
| pino / pino-http | latest | Structured JSON logging (PII-redacted) |
| helmet | latest | Security headers / CSP |
| vitest + supertest | latest | Unit/integration tests |
| @playwright/test | latest | e2e tests |
| **PostgreSQL** | 18 | Database |
| **Redis** | 7.x | Rate-limit/session store |
| **Caddy** | 2.x | Reverse proxy, auto-HTTPS, HSTS |
External APIs: Stripe (Checkout, Billing, Webhooks), Resend (email + webhooks), HIBP range API
(breached-password screening, k-anonymity).

## 5. Key Technical Decisions & Alternatives
| Decision | Choice | Alternatives considered | Why |
|----------|--------|------------------------|-----|
| Card capture | Stripe hosted Checkout | Embedded Elements; custom form | SAQ A posture (no PAN on servers); least code. Elements deferred (ADR-006). |
| Stock concurrency | Atomic conditional decrement + `CHECK(stock>=0)`; `FOR UPDATE` for hot reads | Optimistic version column; advisory locks | Prisma has no native pessimistic lock (§3.1); atomic decrement is structurally correct and simplest. |
| Email delivery | Transactional outbox + worker | Inline Resend call in request | `guaranteed delivery` constraint forbids best-effort inline send (dual-write hazard). |
| Auth | Session cookie + Argon2id (first-party email/pw) | OAuth/social; magic link; JWT | Blank `auth_model` → simplest safe default for a storefront (ADR-001); sessions revocable. |
| TLS | Caddy auto-HTTPS (prod) / mkcert (dev) | Nginx + Certbot | Caddy gives automatic certs + HSTS with least config (HTTPS-only constraint). ADR-005. |
| Monorepo | npm workspaces (`apps/*`, `packages/*`) | Polyrepo; Turborepo | Shared DTOs/types between web & api; single CI. |
| Design tokens | Copy DESIGN-TEMPLATE `:root` verbatim (overrides config palette) | Use project-config palette | Template is binding per DESIGN.md precedence (ADR-008). |

## 6. Implementation Notes & Risks
- **Stripe raw body**: webhook route must mount `express.raw({type:'application/json'})` *before*
  any global `express.json()`; mis-order silently breaks signature verification.
- **Prisma + FOR UPDATE**: must run inside `prisma.$transaction(async tx => …)` via `tx.$queryRaw`;
  cannot send multi-statement raw strings.
- **next/image LCP**: hero/product images use `priority` + correct `sizes` to hit LCP<1.5s.
- **Redis required for correctness**: rate limits must share state across api replicas.
- **Secrets**: `.env` gitignored, `.env.example` committed with placeholders; no `sk_*`/`whsec_`
  literals in source (INV-7).
- Risks tracked in RESEARCH.md §5 (threat model) and §6 (open items).

## 7. Threat Model (comprehensive depth)
Full STRIDE register in RESEARCH.md §5 (R1–R10). Architectural placement of each control:
- R1 Webhook spoofing → `apps/api/src/webhooks/stripe.ts` signature verify (INV-6) + idempotency (INV-4).
- R2 Oversell → `apps/api/src/inventory/decrementStock.ts` atomic decrement + `CHECK(stock>=0)` (INV-9/INV-15).
- R3 Enumeration → `apps/api/src/auth/*` uniform shape+timing.
- R4 IDOR → `scopedRepo` wrapper (INV-13 manual).
- R5 PII at-rest → DB/disk encryption (⚠️ delegated-infra, gated honestly in COMPLIANCE).
- R6 Brute force → rate-limit before Argon2 (INV-10).
- R7 CSRF → SameSite + double-submit token middleware.
- R8 Secrets → env management (INV-7).
- R9 Supply chain → CI `npm audit` + CSP/SRI (INV-11 manual).
- R10 Outbox → worker + idempotency key.

## 8. Alternatives-Considered (comprehensive depth)
Recorded inline in §5 and as ADRs in DECISIONS.md (ADR-001…ADR-010). No RESEARCH-level
deliverable (framework, design system, a11y/perf target, Success Metric) is dropped or downgraded.

## 9. Architectural Invariants
Each rule has a machine-checkable (or `manual`) entry in **invariants.json** at project root,
cross-referenced by `id`. The Stage 4 build lint and the ship-time Drift Detection Gate consume it.

| ID | Rule | Source / file | Check type |
|----|------|---------------|-----------|
| **INV-1** | No raw card data (PAN/CVV) is ever captured, stored, or referenced in source — card entry lives only in Stripe-hosted Checkout (SAQ A). | RESEARCH §5 R1; §3.5 PCI | forbidden-pattern |
| **INV-2** | A committed `.env.example` enumerates every required env var (no secrets). | §3.9; Stage-3 env artifacts | required-file |
| **INV-3** | A `compose.yaml` defines the full local/deploy stack (web, api, worker, db, redis, proxy). | §1.1 | required-file |
| **INV-4** | Webhook idempotency is enforced by a UNIQUE constraint on `processed_webhook_events(event_id)`. | §3.9-B; R1 | required-unique-constraint |
| **INV-5** | Account identity is unique: UNIQUE on `users(email)`. | §3 auth; R3 | required-unique-constraint |
| **INV-6** | In the Stripe webhook handler, signature verification precedes event processing. | §3.9-B/D; R1 | boundary-order |
| **INV-7** | No live secrets in source: `sk_live_`/`sk_test_`/`whsec_`/`re_` key literals appear only in `.env.example`/docs as placeholders. | §6; R8 | forbidden-pattern |
| **INV-8** | In the auth login path, the rate limiter executes before the Argon2 password verify. | §3.5 NIST; R6 | boundary-order |
| **INV-9** | Design-token discipline: no hard-coded hex colors in component/source files outside the token layer. | §3 Design; DESIGN.md §8 | forbidden-pattern |
| **INV-10** | Transactional email is sent only from the worker/outbox path — no direct `resend.emails.send` in request handlers. | §3.9-C; R10 | forbidden-pattern |
| **INV-11** | A working invariant-lint runner exists and implements every check type present in this file. | BUILD.md Verification Gate | required-file |
| **INV-12** | UI coverage: every SPEC §UI-Surface screen route resolves to a rendered route, every non-internal endpoint is referenced by UI, every §5 workflow has an e2e test. | SPEC §UI Surface | ui-coverage |
| **INV-13** | Tenant isolation: every customer-owned DB access goes through the `scopedRepo(userId)` wrapper; cross-customer lookups return NOT_FOUND. | §3 patterns; R4 | manual |
| **INV-14** | All monetary values are stored and computed as integer minor units (cents); PII is never written to logs (pino redaction configured). | §3.9-E; R5 | manual |
| **INV-15** | Stock decrements use the conditional atomic `UPDATE … WHERE stock >= qty` helper; `products.stock` carries `CHECK (stock >= 0)`. | §3.9-A; R2 | manual |

Invariant count: **15** (11 machine-checkable: 2 required-file, 2 required-unique-constraint,
2 boundary-order, 3 forbidden-pattern, 1 ui-coverage, 1 required-file runner; 3 manual; note
INV-2/INV-3/INV-11 are required-file).
