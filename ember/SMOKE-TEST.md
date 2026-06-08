# SMOKE-TEST.md — Ember manual smoke checklist

Human walkthrough to confirm a running stack behaves. For the scripted version see
`bash smoke-test.sh` (and the `/smoke` command). Money is integer cents, USD.

> **Externals note.** Flows that touch Stripe or Resend need real keys in `.env`. Without them
> the app stays contract-correct: the dependent endpoints return **`503 service_unavailable`**
> (RFC 9457) rather than failing unsafely. Those steps are marked **(needs Stripe)** /
> **(needs Resend)** below. A run with externals unprovisioned is `-contract-verified`; a run
> with real keys is `-functional-verified`.

## 0. Bring up the stack

- [ ] `docker compose up -d db redis`
- [ ] `npm --prefix apps/api run db:migrate` then `npm --prefix apps/api run db:seed`
- [ ] `docker compose up --build` (proxy, web, api, worker, db, redis)
- [ ] Visit **https://localhost** — accept the local Caddy cert (self-signed internal CA).
- [ ] Confirm `http://localhost` **301-redirects** to `https://localhost` (no plaintext serving).
- [ ] `GET https://localhost/api/health` (use `curl -k`) returns `{status, version, services}`.

## 1. Browse the catalog (W1 — browse)

- [ ] Landing `/` renders: hero, featured coffees, subscribe band, brew teaser, about, footer.
- [ ] `/coffee` shows the product grid; roast filter and sort work; empty filter state reads
      "no matches — adjust filters".
- [ ] Open a product `/coffee/[slug]`: gallery, tasting notes, price, and an **unmistakable
      Add to cart** buy box.

## 2. Cart (W1 — cart)

- [ ] Add a bag to the cart; the cart drawer opens with the line item, qty stepper, and subtotal.
- [ ] Update quantity and remove an item; subtotal updates.
- [ ] Empty cart shows "Your cart is empty — browse coffees".

## 3. Register / login (W6)

- [ ] `/register` creates an account (field-level RFC 9457 errors on bad input; breached-password
      message when applicable).
- [ ] `/login` signs in; `/account` shows profile + addresses + theme toggle.
- [ ] Log in as the seed customer `hello@ember.test` / `EmberCustomer2026!`.

## 4. Reorder (W3)

- [ ] As a returning customer, `/account/orders` lists past orders (empty state: "No orders yet
      — start shopping").
- [ ] One-click **Reorder** repopulates the cart from a prior order.

## 5. Staff add product (W4)

- [ ] Log in as staff `staff@ember.test` / `EmberStaff2026!`.
- [ ] `/admin/products` → create a product (name, slug, origin, roast, tasting notes, price,
      stock) and upload an image.
- [ ] The new product appears in the catalog with its image.

## 6. Checkout (W1 — payment) **(needs Stripe)**

- [ ] From the cart, go to `/checkout`, pick/add an address, press pay.
- [ ] With real Stripe keys: redirected to the Stripe-hosted Checkout URL; after paying, the
      webhook flips the order to **paid** and `/order/[id]` shows it.
- [ ] **Without Stripe keys:** `POST /api/checkout/session` returns `503 service_unavailable`
      (contract-correct).

## 7. Subscription (W2) **(needs Stripe + `STRIPE_PRICE_SUBSCRIPTION`)**

- [ ] Subscribe to a roast from the product page → subscription Checkout session.
- [ ] After the webhook, `/account/subscriptions` shows an **active** subscription; cancel works.
- [ ] Without keys: `503 service_unavailable`.

## 8. Confirmation email (W5) **(needs Resend)**

- [ ] After a paid order, a `pending_emails` row is enqueued in the same txn and the worker drains
      it → status `sent` via Resend.
- [ ] **Without Resend keys:** the order still completes; the email stays queued in the outbox and
      sends once a key is provided (no hard failure).

## 9. Negative paths (no externals needed)

- [ ] **Invalid login** — wrong password, unknown email, and rate-limited all return the **same
      uniform** "invalid email or password" envelope (no account enumeration, R3).
- [ ] **Validation error** — submit a bad register/admin form; response is RFC 9457
      `validation_error` (400) with a per-field `errors{}` map attached to the offending control.
- [ ] **Out of stock** — add more than available / race a low-stock item; `POST /api/cart/items`
      or `POST /api/checkout/session` returns `409 out_of_stock`; stock never goes negative.
- [ ] **Cross-tenant** — request another user's `/api/orders/:id`; returns **404** (not 403).
- [ ] **Large cookie** — a ~16 KB cookie replays fine (32 KB header buffer,
      `--max-http-header-size=32768`).

## Verdict

- [ ] All non-external checks pass → at minimum **`-contract-verified`**.
- [ ] With real Stripe + Resend keys, steps 6–8 pass → **`-functional-verified`**.
