# SPEC.md — Ember

> Stage 2 behavioral contract (build_depth: **comprehensive**). Derived from RESEARCH.md +
> ARCHITECTURE.md. Precedence: RESEARCH > ARCHITECTURE > SPEC. Another engineer (or AI) should be
> able to recreate Ember from this document alone. Money is always **integer cents**, currency USD.

## Table of contents
1. Feature inventory (vertical slices)
2. Data model
3. REST API surface
4. Security requirements
5. Key Workflows (→ e2e tests W1–W6) & domain checklists
6. UI Surface (Design System + screen inventory)
7. Error contract (RFC 9457)
8. Environment & Configuration
9. Performance targets & test plan

---

## 1. Feature inventory (vertical slices)
Each slice = endpoint(s) + screen(s) + e2e test, ≤500 LOC impl+test. Build order = dependency order.

| F | Feature | Surface | Design pass | Builds workflow |
|---|---------|---------|-------------|-----------------|
| F-01 | Foundation: monorepo, compose, Prisma schema+migrations+seed, env, `/api/health`, Caddy, token/theme layer, security middleware skeleton | infra + backend | N/A | (enables all) |
| F-02 | Catalog browse: landing (ported template), catalog grid + filter/sort, product detail, brew guides | web + GET endpoints | ✅ | W1 (browse) |
| F-03 | Auth: register/login/logout, session, Argon2id, HIBP screen, rate limit, account page | web + api | ✅ | W6 |
| F-04 | Cart: add/update/remove, cart drawer, totals | web + api | ✅ | W1 (cart) |
| F-05 | Checkout: stock reservation + Stripe Checkout Session (payment mode) | web + api | ✅ | W1 (checkout) |
| F-06 | Stripe webhooks: signature verify → idempotency → fulfill order + enqueue email | api (internal) | N/A | W1, W5 |
| F-07 | Orders: history list, order detail, one-click reorder | web + api | ✅ | W3 |
| F-08 | Subscriptions: subscribe (subscription mode), manage/cancel, subscription webhooks | web + api | ✅ | W2 |
| F-09 | Email outbox: `pending_emails` + drain worker + Resend + Resend webhooks | worker (internal) | N/A | W5 |
| F-10 | Staff admin: product CRUD with image upload (EXIF strip, derivatives, content-type allow-list, quota) | web + api | ✅ | W4 |
| F-11 | Cross-cutting hardening: CSRF, Helmet/CSP, audit log, RFC 9457 error mapper, structured logging | api | N/A | (all) |
| F-12 | Invariant-lint runner, smoke tests (`smoke-test.sh`, `SMOKE-TEST.md`), CI workflow | tooling | N/A | (verification) |

---

## 2. Data model
PostgreSQL 18 via Prisma 7. IDs are **ULID** strings (sortable, unguessable — IDOR defense). All
`*_cents` are `Int`. Timestamps `timestamptz`.

### Entities
| Table | Key columns | Constraints / notes |
|-------|-------------|---------------------|
| **users** | id, email, password_hash, name, role(`customer`\|`staff`), created_at | **UNIQUE(email)** (INV-5); role default `customer` |
| **sessions** | id, user_id→users, expires_at, created_at, user_agent, ip | revocable; cached in Redis, source-of-truth in PG |
| **addresses** | id, user_id→users, line1, line2?, city, region, postal_code, country, is_default | scoped by user_id |
| **products** | id, slug, name, origin, roast_level(`light`\|`medium`\|`dark`), tasting_notes(text[]), description, price_cents, stock, weight_grams, image_url, active, subscription_price_id?, created_at | **UNIQUE(slug)**; **CHECK (stock >= 0)** (INV-15); **CHECK (price_cents > 0)** |
| **product_images** | id, product_id→products, url, variant(`thumb`\|`card`\|`detail`\|`orig`), width, height, bytes | EXIF stripped on upload; ≤5/product quota |
| **carts** | id, user_id→users?, session_token, created_at, updated_at | one active cart per user/session |
| **cart_items** | id, cart_id→carts, product_id→products, quantity | UNIQUE(cart_id, product_id); references **live** price |
| **orders** | id, user_id→users, status(`pending`\|`paid`\|`fulfilled`\|`cancelled`\|`refunded`), subtotal_cents, shipping_cents, total_cents, currency, stripe_session_id?, stripe_payment_intent_id?, shipping_address(jsonb snapshot), created_at | scoped by user_id; UNIQUE(stripe_session_id) |
| **order_items** | id, order_id→orders, product_id→products?, product_name, product_sku, unit_price_cents, quantity, line_total_cents | **immutable snapshot** (§3.9-E) |
| **payments** | id, order_id→orders, stripe_payment_intent_id, amount_cents, status, created_at | UNIQUE(stripe_payment_intent_id) |
| **subscriptions** | id, user_id→users, product_id→products, stripe_subscription_id, status(`active`\|`past_due`\|`canceled`), interval_months, current_period_end, created_at | **UNIQUE(stripe_subscription_id)** |
| **processed_webhook_events** | id, event_id, source(`stripe`\|`resend`), type, received_at | **UNIQUE(event_id)** (INV-4) |
| **pending_emails** | id, to_email, template, payload(jsonb), status(`pending`\|`sent`\|`failed`\|`dead`), attempts, last_error?, idempotency_key, created_at, sent_at? | outbox; **UNIQUE(idempotency_key)**; backoff cap 5 → `dead` (DLQ) |
| **audit_logs** | id, actor_user_id?, action, target_type, target_id?, ip, meta(jsonb), created_at | no PII/secret payloads |
| **brew_guides** | id, slug, title, body(md), hero_url, created_at | UNIQUE(slug) |

### State machines
- **order.status:** `pending` →(webhook paid)→ `paid` →(staff/auto)→ `fulfilled`; `pending`→`cancelled`
  (checkout abandoned / payment failed → stock released); `paid|fulfilled`→`refunded` (Stripe refund webhook).
  *Pre:* stock reserved at `pending`. *Post-paid:* reservation finalized, email enqueued (same txn).
- **subscription.status:** `active` ⇄ `past_due` (invoice.payment_failed/paid) → `canceled`
  (customer.subscription.deleted).
- **pending_email.status:** `pending` →(worker send ok)→ `sent`; →(error, attempts<5)→ `pending`
  (retry w/ capped backoff); →(attempts≥5)→ `dead`.

---

## 3. REST API surface
Base `/api`, no version prefix (`api_versioning: none for v1`). Auth via session cookie
`ember_sid` (HttpOnly, Secure, SameSite=Lax). All mutating non-GET require CSRF token. All
list/detail of customer-owned resources scoped by session user. Errors: RFC 9457 (§7). Target
**API p95 < 200 ms** (excludes Stripe round-trips).

### Public / catalog
| Method | Path | Auth | Input | Output | Errors |
|--------|------|------|-------|--------|--------|
| GET | `/api/health` | none | — | `{status, version, services:{db,redis,stripe,resend}}` | 503 if a dep down |
| GET | `/api/products` | none | `?q&roast&sort&page` | `{items:[ProductCard], page, total}` | 400 invalid query |
| GET | `/api/products/:slug` | none | — | `ProductDetail` | 404 |
| GET | `/api/brew-guides` | none | — | `{items:[BrewGuideCard]}` | — |
| GET | `/api/brew-guides/:slug` | none | — | `BrewGuide` | 404 |

### Auth
| POST | `/api/auth/register` | none | `{email,password,name}` | `201 {user}` + sets cookie | 400 validation; **200-shape uniform** on existing email (no enumeration, R3); 422 breached pw |
| POST | `/api/auth/login` | none | `{email,password}` | `200 {user}` + cookie | 401 uniform envelope (unknown/wrong/locked identical); 429 rate-limited |
| POST | `/api/auth/logout` | session | — | `204` | — |
| GET | `/api/auth/me` | session | — | `{user}` | 401 |
| GET | `/api/auth/csrf` | none | — | `{csrfToken}` | — |

Auth order (INV-8): `loginRateLimit` (key `(ip,email)`) → lookup → `verifyPassword` (Argon2id).
Rate limiter fires **before** the hash.

### Cart
| GET | `/api/cart` | session/guest | — | `{items:[{product,quantity,lineTotalCents}], subtotalCents}` | — |
| POST | `/api/cart/items` | session/guest | `{productId,quantity}` | `200 cart` | 400; 409 out_of_stock |
| PATCH | `/api/cart/items/:productId` | session/guest | `{quantity}` | `200 cart` | 404; 409 stock |
| DELETE | `/api/cart/items/:productId` | session/guest | — | `200 cart` | 404 |

### Checkout & orders
| POST | `/api/checkout/session` | session | `{mode:'payment', addressId}` | `200 {url}` (Stripe redirect) | 409 out_of_stock (atomic reserve failed); 400 |
| GET | `/api/orders` | session | — | `{items:[OrderSummary]}` (scoped) | 401 |
| GET | `/api/orders/:id` | session | — | `OrderDetail` (scoped) | 401; **404** if not owner (not 403, R4) |
| POST | `/api/orders/:id/reorder` | session | — | `200 {cart}` (W3 one-click) | 401; 404; 409 stock |

### Subscriptions
| POST | `/api/checkout/session` | session | `{mode:'subscription', productId, addressId}` | `200 {url}` | 400; 409 |
| GET | `/api/subscriptions` | session | — | `{items:[Subscription]}` (scoped) | 401 |
| POST | `/api/subscriptions/:id/cancel` | session | — | `200 {subscription}` | 401; 404 |

### Staff admin (role=staff)
| GET | `/api/admin/products` | staff | — | `{items}` | 401; 403 if not staff |
| POST | `/api/admin/products` | staff | `{name,slug,origin,roastLevel,tastingNotes,description,priceCents,stock,weightGrams}` | `201 {product}` | 400; 409 slug taken |
| PATCH | `/api/admin/products/:id` | staff | partial | `200 {product}` | 400; 404 |
| POST | `/api/admin/products/:id/image` | staff | multipart (`image`) | `201 {images}` | 400 bad type; 413 too large; 429 quota |

### Webhooks (internal; raw body)
| POST | `/api/webhooks/stripe` | signature | raw | `200 {received:true}` | 400 invalid signature |
| POST | `/api/webhooks/resend` | svix sig | raw | `200` | 400 |

Webhook order (INV-6): `constructEvent(raw, sig, secret)` → `processStripeEvent` (idempotency insert
→ handle). Unhandled types: 200 ack + log. Handled Stripe types: `checkout.session.completed`,
`checkout.session.async_payment_succeeded`, `payment_intent.succeeded`, `invoice.paid`,
`invoice.payment_failed`, `customer.subscription.created|updated|deleted`,
`charge.refunded`.

---

## 4. Security requirements
| ID | Requirement | Implementation | Verifies |
|----|-------------|----------------|----------|
| SEC-1 | Webhook authenticity | `stripe.webhooks.constructEvent` on raw body; per-endpoint signing secret | R1, INV-6 |
| SEC-2 | Webhook/payment idempotency | UNIQUE `processed_webhook_events.event_id`; insert-first-in-txn | R1, INV-4 |
| SEC-3 | No oversell | atomic `UPDATE … WHERE stock>=qty` + `CHECK(stock>=0)`; deterministic lock order | R2, INV-15 |
| SEC-4 | Anti-enumeration | uniform response shape + timing on register/login/reset | R3 |
| SEC-5 | Tenant isolation / no IDOR | `scopedRepo(userId)`; cross-user → 404 | R4, INV-13 |
| SEC-6 | PII protection | TLS 1.2+; AES-256 at rest (⚠️ delegated infra, gated in COMPLIANCE); pino redaction; audit log PII reads | R5, INV-14 |
| SEC-7 | Brute-force resistance | rate-limit before Argon2id; per-(ip,email); HIBP k-anonymity screen on register | R6, INV-8 |
| SEC-8 | CSRF | double-submit token on non-GET; SameSite=Lax + Secure + HttpOnly cookies; Origin check | R7 |
| SEC-9 | Secrets hygiene | env vars; `.env` gitignored, `.env.example` placeholders; no key literals in source | R8, INV-7 |
| SEC-10 | Supply chain / script integrity | CI `npm audit`; Helmet CSP (Stripe domains allow-listed); SRI on first-party JS (SAQ A) | R9 |
| SEC-11 | Card data scope | Stripe-hosted Checkout only; no PAN/CVV fields | PCI SAQ A, INV-1 |
| SEC-12 | Rate limiting (global) | every public endpoint has limiter w/ SPEC key dims; upload freq cap | R6, constraint |
| SEC-13 | Audit logging | auth events, payment/webhook (by event id), PII access, authz failures — no PII/secret in log body | constraint, R5/A09 |

### Compliance baseline (PCI-DSS) & NIST password (RESEARCH §3.5)
- Password verifier per **NIST 800-63B-4**: min length **≥12** (covers PCI v4), accept ≤64+ (we
  cap 128), all printable chars, **no composition rules, no forced rotation**, breached-password
  screen via **HIBP k-anonymity** (send first 5 SHA-1 chars only), rate-limit before slow hash.
- **At-rest encryption** (PCI/GDPR): Postgres volume + backups AES-256. In delegated/managed
  Postgres this is provider-enforced → COMPLIANCE marks ⚠️ delegated honestly (never rolled up to ✅).
- **TLS:** HTTPS-only, HSTS preload, TLS 1.2/1.3 (Mozilla Intermediate).

---

## 5. Key Workflows (→ e2e tests) & domain checklists

| W | Workflow (RESEARCH §"Key Workflows") | Terminal step | e2e test name |
|---|--------------------------------------|---------------|---------------|
| **W1** | Browse catalog → open product → read tasting notes → add to cart → checkout via Stripe | redirect to Stripe Checkout URL returned (paid state set via webhook in W5) | `test('W1 browse to checkout')` |
| **W2** | Subscribe to a recurring monthly shipment of a roast | subscription Checkout session URL + active subscription on webhook | `test('W2 subscribe monthly')` |
| **W3** | Returning customer views order history and reorders in one click | reorder repopulates cart | `test('W3 reorder one click')` |
| **W4** | Staff adds a product with photos, price, stock level | product visible in catalog with image | `test('W4 staff add product')` |
| **W5** | Shopper receives order-confirmation email after successful payment | `pending_emails` row → worker → `sent` (Resend) | `test('W5 order confirmation email')` |
| **W6** | Account bootstrap: register → login → view account (auth foundation) | authed `/api/auth/me` 200 | `test('W6 auth bootstrap')` |

Each workflow drives end-to-end **through the UI** to its terminal step (Playwright), satisfying
the `ui-coverage` invariant (INV-12), 1:1 with e2e specs.

### Domain checklists (domain_signals → SPEC §5 requirements)
- **has_payments:** integer minor units everywhere; idempotent charge (Stripe idempotency key +
  event-id dedupe); provider-webhook is payment source-of-truth; refund/dispute lifecycle
  (`charge.refunded` → order `refunded`); **never persist PAN/CVV**.
- **has_webhooks:** verify-signature-before-DB-read; idempotency UNIQUE; retain processed events
  ≥72h; anomaly flag on signature failures (audit log + metric).
- **has_image_uploads:** **EXIF strip**, multi-size derivatives (`thumb`/`card`/`detail`),
  content-type allow-list (`image/jpeg|png|webp`; HEIC transcoded), per-product quota (≤5),
  max bytes 8 MB (413 over).

---

## 6. UI Surface

### 6.1 Design System (peer to screen inventory; binding)
- **Tokens:** copied verbatim from `DESIGN-TEMPLATE.html` `:root` (ADR-008). Palette: bg `#f6efe6`,
  surface `#fffaf3`, surface-2 `#efe4d4`, fg `#2a2018`, muted `#6f6155`, border `#e4d8c7`,
  accent `#2d1e09`, accent-soft `#b98a4e`, cream `#f3e7d3`, success `#3f7d4e`, warning `#b5781f`,
  error `#b4452f`. Dark via `[data-theme="dark"]` (bg `#16110b`, accent `#e7b97e`, …).
- **Type:** display **Outfit** (600/700), body **Inter** (400/500), mono **JetBrains Mono**.
  Scale 10.2/12.8/16/20/25/31.3/39.1px (ratio 1.25). Body measure 60–80ch.
- **Spacing:** 4-based scale `--space-1..9` (4/8/12/16/24/32/48/64/96). **Radius** sm 4 / md 8 /
  lg 18. **Shadow** warm-tinted sm/md/lg. **Max width** 1200px.
- **Theme:** light + dark; follows `prefers-color-scheme`, persists explicit choice (localStorage
  `ember-theme`), runtime toggle.
- **Archetype = ecommerce** (DESIGN.md Part II): merchandising-led; product imagery hero; price &
  buy action unmistakable; consistent product-card shape; checkout funnel with visible progress &
  persistent order summary; moderate conversion-safe motion (120–320ms, reduced-motion safe).
- **Motion:** add-to-cart feedback, cart-drawer slide, gallery transition; honor `prefers-reduced-motion`.

### 6.2 Screen inventory
Every screen lists route · purpose · states (empty/loading/error/success) · API binding · workflow
· RBAC.

| Screen | Route | Purpose | States | Binds | Workflow | RBAC |
|--------|-------|---------|--------|-------|----------|------|
| **Landing/Home** | `/` | Ported template scaffold: hero, featured coffees, subscribe band, brew teaser, about, footer | loading skeleton; success | `GET /products?featured` | W1 | public |
| **Catalog** | `/coffee` | Grid + filter (roast) + sort; product cards | empty ("no matches — adjust filters"); loading skeleton; error; success | `GET /products` | W1 | public |
| **Product detail** | `/coffee/[slug]` | Gallery, tasting notes, buy box (qty, add-to-cart, subscribe), trust signals | loading; error; 404; success | `GET /products/:slug`, `POST /cart/items` | W1, W2 | public |
| **Cart drawer** | (overlay) | Line items, qty steppers, subtotal, checkout CTA | **empty** ("Your cart is empty — browse coffees"); loading; error; success | `GET/POST/PATCH/DELETE /cart` | W1 | public |
| **Checkout entry** | `/checkout` | Address select/add, order summary, pay button → Stripe | loading; error (out_of_stock surfaced inline); success→redirect | `POST /checkout/session` | W1 | session |
| **Order confirmation** | `/order/[id]` | Reads own DB; "paid" or "processing" if webhook pending | loading (poll); processing; success; 404 | `GET /orders/:id` | W1, W5 | owner |
| **Brew guides** | `/brew`, `/brew/[slug]` | Editorial brew content | loading; 404; success | `GET /brew-guides` | — | public |
| **Register** | `/register` | Create account | error (field-level RFC 9457; breached-pw message); loading; success | `POST /auth/register` | W6 | public |
| **Login** | `/login` | Sign in | error (uniform "invalid email or password"); rate-limited notice; loading; success | `POST /auth/login` | W6 | public |
| **Account** | `/account` | Profile + addresses + theme toggle | loading; error; success | `GET /auth/me` | W6 | session |
| **Orders history** | `/account/orders` | Past orders, reorder button | **empty** ("No orders yet — start shopping"); loading; error; success | `GET /orders`, `POST /orders/:id/reorder` | W3 | owner |
| **Subscriptions** | `/account/subscriptions` | Manage/cancel | empty; loading; error; success | `GET /subscriptions`, cancel | W2 | owner |
| **Admin products** | `/admin/products` | Staff list/create/edit + image upload | empty; loading; error (upload type/size inline); success | `GET/POST/PATCH /admin/products`, image | W4 | staff |

No screen invents its own tokens; all reference §6.1. Error states **consume** the RFC 9457
field-level detail and attach it to the offending control (verified by triggering, not assumed).

---

## 7. Error contract (RFC 9457)
All 4xx/5xx return `application/problem+json`:
```json
{ "type":"https://ember.app/problems/<code>", "title":"<human>", "status":<int>,
  "detail":"<safe message>", "instance":"<request-id>", "errors":{"<field>":"<message>"}? }
```
Codes: `validation_error` (400, `errors` map), `unauthorized` (401), `forbidden` (403),
`not_found` (404), `conflict`/`out_of_stock` (409), `breached_password` (422), `rate_limited` (429),
`service_unavailable` (503, `{dependency}`), `internal_error` (500). No stack traces / SQL / PII in
`detail`. Frontend maps `errors{}` onto form fields.

---

## 8. Environment & Configuration
All config via env vars; **`.env` gitignored, `.env.example` committed** (INV-2). Owning service noted.

| Var | Service | Required | Example | Description |
|-----|---------|----------|---------|-------------|
| `NODE_ENV` | all | yes | `production` | runtime mode |
| `DATABASE_URL` | api, worker | yes | `postgresql://ember:pw@db:5432/ember` | Postgres 18 DSN |
| `REDIS_URL` | api, worker | yes | `redis://redis:6379` | rate-limit/session store |
| `SESSION_SECRET` | api | yes | `<32+ random bytes>` | session cookie signing |
| `CSRF_SECRET` | api | yes | `<32+ random bytes>` | CSRF token signing |
| `STRIPE_SECRET_KEY` | api | yes | `sk_test_xxx` | Stripe API key (test/live) |
| `STRIPE_WEBHOOK_SECRET` | api | yes | `whsec_xxx` | Stripe webhook signing secret |
| `STRIPE_PRICE_SUBSCRIPTION` | api | optional | `price_xxx` | recurring price id |
| `RESEND_API_KEY` | worker | optional* | `re_xxx` | email send (*flows degrade gracefully if absent) |
| `RESEND_WEBHOOK_SECRET` | api | optional | `whsec_xxx` | Resend/Svix webhook verify |
| `RESEND_FROM` | worker | optional | `Ember <orders@ember.app>` | sender |
| `APP_URL` | web, api | yes | `https://localhost` | base URL (success/cancel) |
| `NEXT_PUBLIC_API_URL` | web | yes | `https://localhost/api` | client API base |
| `HIBP_ENABLED` | api | optional | `true` | toggle breach screen (default true) |

- **Header-buffer sizing** (BUILD.md): Caddy default header limits suffice; Node api started with
  `--max-http-header-size=32768`; documented in DECISIONS.md. Large-cookie (≥16KB) resilience
  smoke-tested.
- **Protocol & TLS:** `protocol_support: HTTPS only` → Caddy forces HTTP→HTTPS 301, HSTS preload;
  dev uses mkcert. No plaintext HTTP except the redirect. (Detailed in QUICKSTART.md.)

## 9. Performance targets & test plan
- **Targets (acceptance criteria):** product page **LCP < 1.5s** (next/image priority+sizes, RSC,
  self-hosted fonts); **checkout < 3s** to Stripe redirect; **API p95 < 200ms**; 500 concurrent
  shoppers; stock never negative under concurrency.
- **Test plan (test-after):** Vitest unit (money, validation, scopedRepo, decrementStock,
  idempotency); Supertest integration per endpoint incl. ≥1 negative path each (validation/auth/
  payment) asserting RFC 9457 contract + field message; **racing integration test** for stock
  decrement (concurrent buyers, asserts no oversell); Playwright e2e W1–W6 (1:1). Invariant lint
  (`scripts/invariant-lint.mjs`) in CI. Smoke: `smoke-test.sh` (health, auth flow, CRUD cycle,
  negative paths, large-cookie replay).
