# COMPLIANCE.md — Ember

Maps every SPEC §4 security requirement + compliance-tagged clause to implementation status.
Status: ✅ implemented & audit-clean · ⚠️ implemented with open notes/limitations · ❌ gap.
Build 0.1.2 · audited 2026-06-07 (see SECURITY-AUDIT.md).

| Req | Description | Status | Evidence | Notes |
|-----|-------------|--------|----------|-------|
| SEC-1 | Webhook authenticity (Stripe signature verify on raw body) | ✅ | `apps/api/src/webhooks/stripe.ts` (INV-6); raw-body mount in `app.ts` | constructEvent before processing |
| SEC-2 | Webhook/payment idempotency (UNIQUE event_id) | ✅ | migration `processed_webhook_events_event_id_key` (INV-4); `webhooks/stripe.ts` | insert-first, 200 on conflict |
| SEC-3 | No oversell (atomic decrement + CHECK) | ✅ | `inventory/decrementStock.ts`; `products_stock_nonneg` CHECK (INV-15); `racing.test.ts` | racing test green |
| SEC-4 | Anti-enumeration (uniform shape+timing) | ✅ | `auth/*`; smoke W6 negative paths | unknown-email == wrong-password == 401 |
| SEC-5 | Tenant isolation / no IDOR (scopedRepo → 404) | ✅ | `lib/scopedRepo.ts` (INV-13); `orders.test.ts`; smoke W3 | cross-user → 404 not 403 |
| SEC-6 | PII protection (TLS, at-rest enc, no PII logs) | ⚠️ | TLS via Caddy ✅; pino redaction ✅ (INV-14); at-rest enc **delegated** | At-rest AES-256 is delegated to the managed/host Postgres volume; not app-layer. Marked ⚠️ delegated honestly per BUILD.md — verify provider encryption in production. |
| SEC-7 | Brute-force resistance (rate-limit before hash, HIBP) | ✅ | `middleware/rateLimit.ts` + `auth/login.controller.ts` (INV-8); `lib/password.ts` HIBP | IPv6 bypass fixed (audit F1) |
| SEC-8 | CSRF (double-submit + Origin) | ✅ | `middleware/csrf.ts`; SameSite=Lax+Secure+HttpOnly cookies | |
| SEC-9 | Secrets hygiene (env, no literals) | ✅ | `.env.example`; INV-7 grep clean; `.gitignore` | |
| SEC-10 | Supply chain / script integrity (CSP, SRI, audit) | ⚠️ | Helmet CSP (Stripe allow-listed) ✅; CI `npm audit` ✅; SRI not applied to first-party JS | 1 LOW residual (build-time postcss via next, deferred — SECURITY-AUDIT F4). SRI deferred — Next manages its own asset hashes; revisit for SAQ A script attestation. |
| SEC-11 | Card data scope (Stripe-hosted, no PAN) | ✅ | INV-1 (0 card identifiers); ADR-006 | SAQ A posture |
| SEC-12 | Rate limiting (global + per-endpoint key dims) | ✅ | `middleware/rateLimit.ts`; smoke (ratelimit-* headers) | Redis store, multi-replica safe |
| SEC-13 | Audit logging (auth/payment/PII/authz, no PII body) | ✅ | `middleware/audit.ts`; `audit_logs` table | structured JSON (pino) |
| PCI-DSS | SAQ A eligibility (outsourced card capture) | ⚠️ | Stripe hosted Checkout; TLS/HSTS; CSP | Code controls met; the **Attestation of Compliance + operational controls** (vendor mgmt, access reviews) are an org process outside the codebase. |
| NIST-PW | Password policy (≥12, HIBP, no rotation, slow hash) | ✅ | `dto.ts` passwordSchema; `lib/password.ts` (argon2id + HIBP) | NIST 800-63B-4 |
| AT-REST | At-rest encryption (PCI/GDPR) | ⚠️ | Delegated to managed Postgres / host volume | Same as SEC-6 — gate as ⚠️ delegated, do not roll up to ✅ until provider encryption is confirmed. |

## Posture
- **A (✅ clean): 12**
- **B (⚠️ open notes): 3** — SEC-6 / AT-REST (at-rest encryption delegated to managed DB), SEC-10 (SRI on first-party JS deferred + 1 LOW build-time advisory), PCI-DSS (AOC + operational controls are org process).
- **C (❌ gaps): 0**

The three ⚠️ items are **delegated/operational**, not code defects — they require infrastructure/process
decisions at deploy time, documented here as permanent posture and surfaced at Pipeline Complete.
