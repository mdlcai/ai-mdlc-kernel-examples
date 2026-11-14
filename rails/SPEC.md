# SPEC.md — RΛILS by MDLC (v1.0.0)

> The behavioral contract. Canonical sections §1–§10. Every gate references these numbers.

## §1 Overview & scope

**Problem.** Payment operations are fragmented across processors, so merchants suffer duplicate charges, untraceable states and reconciliation gaps (RESEARCH "PROBLEM STATEMENT").

**Product.** RΛILS is authorization and settlement for agentic commerce, on a multi-tenant payment orchestration control plane. A merchant issues an AI agent a spend-capped, expiring **agent mandate** (F-20); the agent authorizes payments with it, funds are held, and the agent or merchant settles by capture into the double-entry ledger. Underneath: one API (`/api/v1`), processor adapters (Stripe + Sandbox), a deterministic payment state machine, exactly-once idempotency, signed webhooks, a realtime console and a tamper-evident governance trail (ADR-057).

**Personas & roles.**
- **owner** — merchant founder: everything, including deleting members and transferring ownership.
- **admin** — merchant operations lead: everything except ownership.
- **developer** — API keys, webhooks, sandbox, events; read payments.
- **analyst** — finance: read payments, ledger, reports, export, governance.
- **viewer** — read-only dashboard and payments.
- **platform operator** — RΛILS staff (`users.platform_role = 'operator'`): the `/ops` console, cross-tenant read, suspend merchants, DLQ.

**Modes.** Every merchant has `test` and `live` modes. API keys and agent mandates are mode-bound (`sk_test_…`, `sk_live_…`, `ak_test_…`, `ak_live_…`). The console has a mode toggle. All payment data is partitioned by `mode`.

**Non-Goals (deferred per RESEARCH "PRIMARY MVP WORKFLOWS"):**
- intelligent multi-processor routing / ML routing;
- advanced fraud intelligence;
- ACH;
- Bitcoin/Lightning/digital assets;
- sophisticated settlement reconciliation;
- subscription billing;
- AI-assisted operations;
- Fiserv/Worldpay adapters;
- QR / scan-to-pay and real-time account rails;
- third-party agent protocols and network agent tokens (mandates are merchant-issued, ADR-057);
- native mobile apps;
- MFA/TOTP;
- SSO.

**Scope expansion (comprehensive), each logged in DECISIONS.md:**
- platform operator console (F-18);
- analytics + CSV export (F-14);
- audit trail + evidence export (F-15);
- team management (F-04);
- own-session management (F-02);
- agent mandates (F-20, ADR-057).

## §2 Data model

Conventions:
- Every table has `id BIGINT` (identity), `created_at` and `updated_at`.
- Public ids are prefixed strings (`pi_`, `cus_`, `re_`, `dp_`, `evt_`, `we_`, `key_`, `je_`) stored in `public_id` (UNIQUE) and generated from 24 random base58 chars. The API exposes only `public_id` as `id`.
- Money is `amount_minor BIGINT` + `currency CHAR(3)` (ISO 4217, lowercase). The CHECK constraint is `amount_minor > 0` unless noted.
- "Tenant" = `merchant_id NOT NULL REFERENCES merchants`. "Mode" = `mode TEXT CHECK (mode IN ('test','live'))`.
- PII columns are encrypted with Active Record Encryption (AES-256-GCM) and are listed in the logger redaction filter.

| Table | Key columns & constraints | Owner |
|---|---|---|
| `merchants` | `public_id`, `name` (1–120), `slug` UNIQUE, `country CHAR(2)`, `default_currency CHAR(3)`, `status` CHECK (`active`,`suspended`), `support_email` (enc), `statement_descriptor` (≤22), `risk_max_amount_minor BIGINT` default 1_000_000, `logo_asset_id` | tenant root |
| `users` | `email` (enc, deterministic, UNIQUE via `email_digest`), `name` (enc), `password_digest`, `platform_role` CHECK (`none`,`operator`), `breach_check_pending BOOL`, `failed_logins INT`, `locked_until` | global identity |
| `memberships` | `user_id`, `merchant_id`, `role` CHECK (`owner`,`admin`,`developer`,`analyst`,`viewer`); UNIQUE(user_id, merchant_id) | tenant |
| `sessions` | `user_id`, `token_digest` UNIQUE, `active_merchant_id`, `ip`, `user_agent`, `last_seen_at`, `expires_at`, `revoked_at` | user |
| `invitations` | `merchant_id`, `email` (enc) + `email_digest`, `role`, `token_digest` UNIQUE, `invited_by_id`, `expires_at` (7d), `accepted_at`, `revoked_at` | tenant |
| `password_resets` | `user_id`, `token_digest` UNIQUE, `expires_at` (30 min), `used_at` | user |
| `api_keys` | `merchant_id`, `mode`, `name`, `prefix` (first 12 chars), `token_digest` UNIQUE (SHA-256), `last_used_at`, `revoked_at`, `created_by_id` | tenant |
| `agent_mandates` | `merchant_id`, `mode`, `public_id` (`mdt_`), `customer_id?` (composite FK, same tenant), `name` (1–80), `prefix` (first 12 chars, `ak_<mode>_…`), `token_digest` UNIQUE (SHA-256), `currency`, `per_transaction_limit_minor` BIGINT (1–99,999,999), `total_limit_minor` BIGINT (≥ per-transaction, ≤ 9,999,999,999), `capture_method` (default `manual`), `expires_at` (≤ 365 days), `last_used_at`, `revoked_at`, `created_by_id`. Spend is derived from its intents, never stored | tenant |
| `processor_accounts` | `merchant_id`, `mode`, `processor` CHECK (`sandbox`,`stripe`), `enabled`, `priority INT`, `is_default`; UNIQUE(merchant_id, mode, processor) | tenant |
| `customers` | `merchant_id`, `mode`, `public_id`, `email` (enc) + `email_digest`, `name` (enc), `phone` (enc), `billing_country CHAR(2)`, `billing_address` (enc JSON), `redacted_at` | tenant |
| `payment_methods` | `merchant_id`, `mode`, `customer_id`, `processor`, `processor_token` (enc), `brand`, `last4 CHAR(4)`, `exp_month SMALLINT` (1–12), `exp_year SMALLINT`, `fingerprint`, `detached_at` | tenant |
| `payment_intents` | `merchant_id`, `mode`, `public_id`, `customer_id?`, `payment_method_id?`, `agent_mandate_id?` (composite FK, same tenant), `amount_minor`, `currency`, `amount_capturable_minor`, `amount_captured_minor`, `amount_refunded_minor` (CHECK ≤ captured), `fee_minor`, `capture_method` CHECK (`automatic`,`manual`), `status` CHECK (`requires_payment_method`,`requires_confirmation`,`processing`,`requires_capture`,`succeeded`,`failed`,`canceled`), `processor`, `processor_ref`, `failure_code`, `cancellation_reason`, `description` (≤500), `metadata JSONB` (≤20 keys), `risk_score SMALLINT` (0–100), `lock_version INT`, `canceled_at`, `succeeded_at` | tenant |
| `payment_attempts` | `payment_intent_id`, `merchant_id`, `processor`, `attempt_no`, `outcome` CHECK (`succeeded`,`requires_capture`,`declined`,`error`,`unavailable`), `processor_ref`, `error_code`, `latency_ms`; UNIQUE(payment_intent_id, attempt_no) | tenant |
| `refunds` | `merchant_id`, `mode`, `public_id`, `payment_intent_id`, `amount_minor`, `currency`, `reason` CHECK (`requested_by_customer`,`duplicate`,`fraudulent`), `status` CHECK (`pending`,`succeeded`,`failed`), `processor_ref`, `failure_code` | tenant |
| `disputes` | `merchant_id`, `mode`, `public_id`, `payment_intent_id`, `amount_minor`, `currency`, `reason`, `status` CHECK (`needs_response`,`under_review`,`won`,`lost`), `evidence JSONB` (text ≤ 5000), `evidence_submitted_at`, `due_by`, `processor_ref` | tenant |
| `ledger_accounts` | `merchant_id`, `mode`, `code` CHECK (`processor_receivable`,`merchant_payable`,`platform_fees`,`refunds_clearing`,`dispute_losses`), `currency`, `normal_balance` CHECK (`debit`,`credit`), `balance_minor BIGINT` default 0 (signed, projection), `lock_version`; UNIQUE(merchant_id, mode, code, currency) | tenant |
| `journal_entries` | `merchant_id`, `mode`, `public_id`, `kind` CHECK (`capture`,`refund`,`dispute_open`,`dispute_won`,`dispute_lost`; `reversal` removed, ADR-050), `source_type`, `source_id`, `currency`, `memo`; UNIQUE(source_type, source_id, kind) | tenant, immutable |
| `ledger_entries` | `journal_entry_id`, `ledger_account_id`, `merchant_id`, `direction` CHECK (`debit`,`credit`), `amount_minor > 0`, `currency` | tenant, immutable |
| `idempotency_keys` | `merchant_id`, `mode`, `key` (1–255), `request_method`, `request_path`, `request_fingerprint` (SHA-256 of the canonical body), `locked_at`, `response_status`, `response_body JSONB`, `completed_at`; UNIQUE(merchant_id, mode, key) | tenant |
| `webhook_events` | `source` CHECK (`stripe`,`rails`; `sandbox` removed, ADR-050), `event_id`, `merchant_id?`, `mode`, `type`, `payload JSONB`, `signature_valid`, `anomaly` (text null), `processed_at`, `error`; UNIQUE(source, event_id) | tenant (nullable for unmatched) |
| `webhook_endpoints` | `merchant_id`, `mode`, `public_id`, `url` (https, ≤2048), `description`, `enabled_events TEXT[]`, `secret` (enc), `status` CHECK (`enabled`,`disabled`) | tenant |
| `pending_webhooks` | `merchant_id`, `webhook_endpoint_id`, `event_public_id`, `event_type`, `payload JSONB`, `signature` (HMAC-SHA256 computed at insert), `idempotency_key` UNIQUE, `status` CHECK (`pending`,`delivering`,`delivered`,`retrying`,`dlq`), `attempts SMALLINT`, `next_attempt_at`, `claimed_at`, `last_response_code`, `last_error`, `delivered_at` | tenant |
| `pending_emails` | `merchant_id?`, `to_address` (enc), `template` CHECK (`password_reset`,`invitation`,`welcome`,`breach_notice`), `params JSONB`, `status` CHECK (`pending`,`sending`,`sent`,`retrying`,`dlq`), `attempts`, `next_attempt_at`, `claimed_at`, `last_error`, `sent_at` | tenant/global |
| `audit_events` | PARTITION BY RANGE(`created_at`) monthly; PK(`id`,`created_at`); `merchant_id?`, `actor_type` (`user`,`api_key`,`agent`,`system`,`processor`), `actor_id`, `action`, `target_type`, `target_id`, `ip`, `request_id`, `data JSONB` (redacted), `prev_hash`, `hash` (sha256) | tenant, immutable |
| `scheduled_runs` | `job_name`, `run_key`, `merchant_id?`, `claimed_at`, `finished_at`, `status` CHECK (`running`,`succeeded`,`failed`), `counts JSONB`; UNIQUE(job_name, run_key) | system |
| `risk_signals` | `payment_intent_id`, `merchant_id`, `rule` CHECK (`amount_ceiling`,`velocity`,`country_mismatch`), `score SMALLINT`, `ip_country CHAR(2)`, `approx_lat NUMERIC(4,1)` CHECK −90..90, `approx_lng NUMERIC(5,1)` CHECK −180..180 | tenant |
| `merchant_assets` | `merchant_id`, `kind` (`logo`), `content_type` CHECK (`image/png`,`image/webp`), `byte_size` (≤1MB original), `variant` (`64`,`256`), `data BYTEA`, `sha256` | tenant |
| `evidence_exports` | `merchant_id`, `public_id`, `requested_by_id`, `from`, `to`, `status` (`ready`), `bundle JSONB`, `chain_head` | tenant |

## §3 API surface

All paths are relative to the API mount `https://<host>/api`. The web client's base is `/api`, and every path below begins with `/v1`, so it is never double-prefixed. Responses are `application/json`; errors use §7.

**Auth:**
- **S**: session cookie + CSRF header (console).
- **K**: API key (`Authorization: Bearer sk_…`).
- **S|K**: either.
- **A**: agent mandate (`Authorization: Bearer ak_…`, F-20). Accepted only where listed; every other endpoint answers 403 `FORBIDDEN`. Agents always send `Idempotency-Key` on POSTs.
- **P**: public.
- **O**: platform operator session.

**Lists:** every list returns `{data:[…], next_cursor}`, with `limit` 1–100 (default 25) and `starting_after` as the cursor. Side-effecting POSTs on payments, refunds and webhook test sends require `Idempotency-Key` for K and accept it for S (the console sends a UUIDv4).

| Endpoint | Auth / role | Input | Output | Internal |
|---|---|---|---|---|
| `GET /api/health` | P | — | `{status, version, commit, checks:{db,db_replica,redis,queue}}` 200/503 | yes (internal) |
| `POST /v1/auth/signup` | P (throttled) | `{name, email, password, merchant_name, country, default_currency}` | 201 `{user, merchant, membership, test_api_key}` + session cookie | no |
| `POST /v1/auth/login` | P (throttled) | `{email, password}` | 200 `{user, memberships}` + cookie | no |
| `GET /v1/auth/session` | S | — | `{user, active_merchant, role, memberships, csrf_token, mode}` | no |
| `DELETE /v1/auth/session` | S | — | 204, cookie cleared | no |
| `POST /v1/auth/password_resets` | P (throttled) | `{email}` | 202 always (no enumeration) | no |
| `PUT /v1/auth/password_resets/:token` | P | `{password}` | 204; revokes all sessions | no |
| `POST /v1/me/active_merchant` | S | `{merchant_id}` (must be a membership) | 200 session | no |
| `GET /v1/me/sessions` | S | — | own sessions only | no |
| `DELETE /v1/me/sessions/:id` | S | — | 204 (own only; another user's → 404) | no |
| `GET /v1/members` | S any | — | members list | no |
| `PATCH /v1/members/:id` | S owner/admin | `{role}` | member (cannot demote the last owner) | no |
| `DELETE /v1/members/:id` | S owner/admin | — | 204 | no |
| `GET /v1/invitations` | S owner/admin | — | pending invitations | no |
| `POST /v1/invitations` | S owner/admin | `{email, role}` | 201 (email queued) | no |
| `DELETE /v1/invitations/:id` | S owner/admin | — | 204 | no |
| `GET /v1/invitations/:token` | P | — | `{merchant_name, role, email_masked}` | no |
| `POST /v1/invitations/:token/accept` | P or S | `{name?, password?}` (new user) | 200 + session | no |
| `GET /v1/api_keys` | S owner/admin/developer | `mode` | keys (prefix only) | no |
| `POST /v1/api_keys` | S owner/admin/developer | `{name, mode}` | 201 with the plaintext `secret` exactly once | no |
| `DELETE /v1/api_keys/:id` | S owner/admin/developer | — | 204 revoked | no |
| `GET /v1/agent_mandates` | S owner/admin/developer | cursor (mode from the console toggle) | mandates (prefix only) with `spent_minor`, `remaining_minor`, `status` | no |
| `POST /v1/agent_mandates` | S owner/admin/developer | `{name, currency?, per_transaction_limit_minor, total_limit_minor, capture_method?, expires_at?, customer?}` | 201 with the plaintext `secret` exactly once | no |
| `GET /v1/agent_mandates/:id` | S owner/admin/developer | — | mandate | no |
| `DELETE /v1/agent_mandates/:id` | S owner/admin/developer | — | 204 revoked | no |
| `GET /v1/agent_mandates/current` | A | — | the calling mandate, with remaining spend | no |
| `GET /v1/merchant` | S any | — | merchant profile | no |
| `PATCH /v1/merchant` | S owner/admin | `{name, support_email, statement_descriptor, default_currency, risk_max_amount_minor}` | merchant | no |
| `POST /v1/merchant/logo` | S owner/admin | multipart `file` (png/jpeg/webp ≤1MB) | 201 `{logo_url}` | no |
| `GET /v1/merchant/logo/:variant` | S any | `variant` ∈ 64,256 | image/png bytes | no |
| `GET /v1/processor_accounts` | S any | `mode` | adapters with `available` flag | no |
| `PATCH /v1/processor_accounts/:id` | S owner/admin | `{enabled, priority, is_default}` | account | no |
| `GET /v1/customers` | S\|K | `email` (exact via digest), cursor | customers (email masked) | no |
| `POST /v1/customers` | S\|K owner/admin/developer | `{email, name, phone?, billing_country, billing_address?}` | 201 customer | no |
| `GET /v1/customers/:id` | S\|K | — | customer + payment methods | no |
| `PATCH /v1/customers/:id` | S\|K owner/admin/developer | same as create | customer | no |
| `DELETE /v1/customers/:id` | S owner/admin | — | 200 redacted (PII wiped, financial rows kept) | no |
| `POST /v1/customers/:id/payment_methods` | S\|K owner/admin/developer | `{token}` (sandbox `pm_card_*` or Stripe `pm_…`) | 201 payment method (brand/last4/exp) | no |
| `DELETE /v1/payment_methods/:id` | S\|K owner/admin/developer | — | 204 detached | no |
| `GET /v1/payment_intents` | S\|K\|A (A: own intents only) | `status`, `customer`, `created_gte`, `created_lte`, `currency`, `agent_mandate`, cursor | intents | no |
| `POST /v1/payment_intents` | S\|K owner/admin/developer, or A, + Idempotency-Key (A: mandate caps, currency and customer apply; 403 `MANDATE_LIMIT_EXCEEDED`) | `{amount_minor, currency, customer?, payment_method?, capture_method, confirm, description?, metadata?, processor?}` | 201 intent | no |
| `GET /v1/payment_intents/:id` | S\|K\|A (A: own intents only) | — | intent + attempts + refunds + disputes + risk_signals + ledger journal ids | no |
| `POST /v1/payment_intents/:id/confirm` | S\|K\|A + Idempotency-Key | `{payment_method?}` | intent | no |
| `POST /v1/payment_intents/:id/capture` | S\|K\|A + Idempotency-Key | `{amount_to_capture_minor?}` (≤ capturable) | intent | no |
| `POST /v1/payment_intents/:id/cancel` | S\|K\|A + Idempotency-Key | `{cancellation_reason}` ∈ `requested_by_customer`,`duplicate`,`fraudulent`,`abandoned` | intent | no |
| `GET /v1/refunds` | S\|K | `payment_intent`, `status`, cursor | refunds | no |
| `POST /v1/refunds` | S\|K owner/admin + Idempotency-Key | `{payment_intent, amount_minor?, reason}` | 201 refund | no |
| `GET /v1/refunds/:id` | S\|K | — | refund | no |
| `GET /v1/disputes` | S\|K | `status`, cursor | disputes | no |
| `GET /v1/disputes/:id` | S\|K | — | dispute | no |
| `POST /v1/disputes/:id/evidence` | S owner/admin | `{customer_communication, product_description, uncategorized_text}` (each ≤5000) | dispute → `under_review` | no |
| `POST /v1/sandbox/disputes` | S owner/admin/developer, test mode only | `{payment_intent}` (succeeded) | 201 dispute `needs_response` | no |
| `POST /v1/sandbox/disputes/:id/resolve` | S owner/admin/developer, test mode only | `{outcome}` ∈ `won`,`lost` | dispute | no |
| `GET /v1/ledger/accounts` | S owner/admin/analyst | `currency`, `mode` | accounts with balances | no |
| `GET /v1/ledger/journal_entries` | S owner/admin/analyst | `account`, `kind`, `created_gte`, `created_lte`, cursor | journal entries with lines | no |
| `GET /v1/ledger/journal_entries/:id` | S owner/admin/analyst | — | entry + lines + source | no |
| `GET /v1/reports/summary` | S any | `from`, `to`, `currency` | `{gross_minor, net_minor, fees_minor, refunds_minor, count, success_rate, by_status, by_processor}` | no |
| `GET /v1/reports/timeseries` | S any | `from`, `to`, `currency`, `granularity` ∈ hour/day | buckets | no |
| `GET /v1/reports/transactions.csv` | S owner/admin/analyst | `from`, `to`, `status`, `currency` | text/csv (formula-injection escaped) | no |
| `GET /v1/webhook_endpoints` | S owner/admin/developer | — | endpoints | no |
| `POST /v1/webhook_endpoints` | S owner/admin/developer | `{url, description, enabled_events[]}` | 201 + `secret` once | no |
| `PATCH /v1/webhook_endpoints/:id` | S owner/admin/developer | `{url, description, enabled_events, status}` | endpoint | no |
| `DELETE /v1/webhook_endpoints/:id` | S owner/admin/developer | — | 204 | no |
| `POST /v1/webhook_endpoints/:id/rotate_secret` | S owner/admin/developer | — | `{secret}` once | no |
| `POST /v1/webhook_endpoints/:id/test` | S owner/admin/developer | `{event_type}` | 202 delivery queued | no |
| `GET /v1/webhook_deliveries` | S owner/admin/developer | `endpoint`, `status`, cursor | deliveries | no |
| `POST /v1/webhook_deliveries/:id/retry` | S owner/admin/developer | — | 202 (dlq/retrying → pending) | no |
| `GET /v1/events` | S\|K | `type`, `source`, cursor | event log | no |
| `GET /v1/events/:id` | S\|K | — | event with payload | no |
| `POST /v1/webhooks/stripe` | Stripe-Signature | raw body | 200 / 400 | yes (internal, called by Stripe) |
| `GET /v1/audit_events` | S owner/admin/analyst | `action`, `actor`, `created_gte`, `created_lte`, cursor | audit events | no |
| `GET /v1/governance/verification` | S owner/admin/analyst | — | `{valid, checked, head_hash, first_broken_id?}` | no |
| `POST /v1/governance/evidence_exports` | S owner/admin/analyst | `{from, to}` | 201 export | no |
| `GET /v1/governance/evidence_exports/:id` | S owner/admin/analyst | — | JSON bundle (download) | no |
| `GET /v1/ops/merchants` | O | `status`, `q`, cursor | merchants (no PII) | no |
| `PATCH /v1/ops/merchants/:id` | O | `{status}` ∈ active/suspended, `reason` | merchant | no |
| `GET /v1/ops/dlq` | O | `kind` ∈ webhooks/emails | dead-letter rows | no |
| `GET /v1/ops/scheduled_runs` | O | `job_name`, cursor | runs with counts | no |
| `POST /v1/ops/scheduled_runs` | O, non-production only | `{job_name, merchant_id?}` | 202 run | yes (internal smoke trigger) |
| `GET /cable` | S (cookie) WebSocket | subscribe `MerchantEventsChannel` | events `payment_intent.*`, `refund.*`, `dispute.*` | no |

## §4 Security requirements

### §4.1 Threat-derived controls (RESEARCH §5 STRIDE)
| ID | Control | Applies to |
|---|---|---|
| SEC-1 | HTTPS only at the Caddy edge (`tls internal` locally); :80 → 308 to https; HSTS only when `TRUSTED_CERT=true` | all |
| SEC-2 | Session cookie `__Host-rails_session`: Secure, HttpOnly, SameSite=Lax; idle 30m, absolute 12h; rotated on login; revoked on logout/reset | F-02 |
| SEC-3 | CSRF: unsafe S requests need `X-CSRF-Token` == HMAC(session) and an allowed `Origin` | all S |
| SEC-4 | API keys: 32 random bytes; only the SHA-256 digest is stored; shown once; revocable; mode-bound | F-05 |
| SEC-5 | Tenant isolation: every lookup goes through `current_merchant`; another tenant's id → 404 | all |
| SEC-6 | Object-level authz by role (§4.2); personal resources scoped to `current_user` | all |
| SEC-7 | Rate limits (rack-attack, Redis): login 5/min/email + 20/min/IP; signup 5/h/IP; reset 3/h/email; API 100/s/key; money POST 25/s/key; invitation accept 10/h/IP. 429 uses the §7 envelope plus `Retry-After` | all |
| SEC-8 | Passwords: 15–128 chars, no composition rules, HIBP k-anonymity breach screen (5-char SHA-1 prefix, `Add-Padding: true`), bcrypt cost 12, no forced rotation, no KBA, email-link reset (30 min, single use) | F-02 |
| SEC-9 | Account lockout: 10 consecutive failures → 15 min lock; generic error text (no enumeration) | F-02 |
| SEC-10 | PAN guard: Luhn-valid 13–19 digit strings in card-like params → 422 `PAN_NOT_ACCEPTED`; never logged | F-06, F-08 |
| SEC-11 | Inbound webhook signature verified on the raw body before any DB write; 300s tolerance; dedupe UNIQUE(source,event_id) | F-11 |
| SEC-12 | Outbound: SSRF guard (https only, DNS resolve, block private/loopback/link-local/CGNAT/metadata ranges, pin the IP, no redirects, 5s timeout); HMAC-SHA256 `Rails-Signature: t=<ts>,v1=<hex>` | F-12 |
| SEC-13 | WebSocket: `allowed_request_origins`; session auth on connect; channel authz = membership + mode; 20 msgs/10s budget, then disconnect | F-13 |
| SEC-14 | PII encrypted at rest (AR Encryption); redacted from logs and audit data | F-02, F-06 |
| SEC-15 | Audit: every mutation writes a hash-chained `audit_events` row with actor, IP and request id; immutable via trigger | F-15 |
| SEC-16 | Upload hardening: magic-byte sniff (PNG/JPEG/WebP), declared type ignored, 1MB cap, re-encode to PNG via libvips (EXIF/metadata stripped), HEIC rejected with `UNSUPPORTED_MEDIA_TYPE`, 5 uploads/h per merchant | F-16 |
| SEC-17 | CSV export escapes cells starting with `= + - @ \t \r` (formula injection) | F-14 |
| SEC-18 | Security headers: web CSP Profile N (nonce + strict-dynamic); api `default-src 'none'`; nosniff; Referrer-Policy; Permissions-Policy; COOP; `frame-ancestors 'none'` | F-01 |
| SEC-19 | Suspended merchants: every S/K request → 403 `MERCHANT_SUSPENDED` (except session read/logout) | F-18 |
| SEC-20 | `RATE_LIMIT_SCALE` is ignored in production (boot refuses `ALLOW_RATE_LIMIT_SCALE=true` with `RAILS_ENV=production`) | F-01 |

### §4.2 RBAC matrix
| Capability | owner | admin | developer | analyst | viewer |
|---|---|---|---|---|---|
| View dashboard, payments, customers, disputes, events | ✓ | ✓ | ✓ | ✓ | ✓ |
| Create/confirm/capture/cancel payments; manage customers & payment methods | ✓ | ✓ | ✓ | – | – |
| Refund; submit dispute evidence; redact customer | ✓ | ✓ | – | – | – |
| Sandbox dispute simulate/resolve (test mode) | ✓ | ✓ | ✓ | – | – |
| API keys, agent mandates, webhook endpoints, deliveries | ✓ | ✓ | ✓ | – | – |
| Ledger, reports export, audit, governance evidence | ✓ | ✓ | – | ✓ | – |
| Merchant settings, logo, processors | ✓ | ✓ | – | – | – |
| Team: invite, change role, remove | ✓ | ✓ (cannot touch owners) | – | – | – |

An agent mandate (A) is not a role: it may create, read, confirm, capture and cancel only the payment intents it created, within its limits, and read itself. Every other capability is 403 `FORBIDDEN`.

### §4.3 Domain-signal checklists
- **has_webhooks:**
  - (a) signature verified before any DB write (INV-6);
  - (b) idempotency via UNIQUE(source,event_id) (INV-3);
  - (c) retention ≥72h (payload kept 30 days, id rows 90 days);
  - (d) anomaly flag: unknown event type, unmatched processor ref, or >100 events/min from one source → `webhook_events.anomaly` + audit event.
- **has_geo:**
  - (a) bounds CHECK on `approx_lat/lng`;
  - (b) map-provider rate limit is N/A (no map provider; ADR-009);
  - (c) PII: only 1-decimal coarse coordinates (~11 km) derived from the IP country centroid table; no precise location stored.
- **has_websocket:**
  - (a) Origin check (`allowed_request_origins`);
  - (b) per-connection message budget (20/10s);
  - (c) channel authorization (membership + mode).
- **has_scheduled_work** (ARCHITECTURE §6, T1–T6):
  - (1) effect assertion per job;
  - (2) every trigger tested;
  - (3) per-tenant entry point `run_for_merchant(merchant_id, now:)`;
  - (4) idempotent claim `UPDATE … WHERE claimed_at IS NULL` / `scheduled_runs` UNIQUE;
  - (5) injected clock `now:`, time zone UTC;
  - (6) observable outcome counts in `scheduled_runs.counts`;
  - (7) startup resumes unclaimed rows, SIGTERM drains;
  - plus `POST /v1/ops/scheduled_runs` (non-prod) for the smoke test.
- **has_dual_write:** outbox tables `pending_webhooks`/`pending_emails` are written in the domain transaction; Solid Queue lives in the same Postgres.
- **has_payments:**
  - (a) BIGINT `amount_minor` + CHAR(3) currency (INV-1, INV-14);
  - (b) idempotent charge creation, UNIQUE key (INV-2), processor idempotency key `rails_<pi>_<attempt>`;
  - (c) processor webhook is the source of truth;
  - (d) refund and dispute lifecycles (F-10);
  - (e) never store PAN/CVV (INV-11, SEC-10); tokens + brand/last4/exp only.
- **has_email:** `pending_emails` outbox; retries 1m / 10m / 1h, then `dlq`; templates are plain text + HTML with no tracking pixels.
- **has_webhook_send:**
  - `pending_webhooks` outbox;
  - HMAC computed at insert;
  - `Idempotency-Key` header = row key;
  - 5 attempts with backoff 1m / 5m / 30m / 2h / 6h, then `dlq`;
  - SSRF guard (INV-5).
- **has_image_uploads:**
  - EXIF strip (re-encode);
  - derivatives 64 / 256;
  - HEIC rejected with a clear error;
  - type allow-list by magic bytes;
  - 1MB quota per file, one logo per merchant;
  - 5 uploads/hour.

### §4.4 Compliance baseline (PCI-DSS 4.0.1)
| Item | Requirement | Implementation |
|---|---|---|
| PCI-3 | No PAN/SAD storage | Tokens only; PAN guard (SEC-10); INV-11 |
| PCI-4 | Strong crypto in transit | TLS at the Caddy edge (SEC-1) |
| PCI-8.3.6 | Password ≥12 | 15-char minimum (ADR-005); breach screen |
| PCI-8.3.4 | Lockout after ≤10 attempts | SEC-9 |
| PCI-8.2.8 | Idle timeout ≤15 min for CDE admin | 30 min console idle; documented compensating control ADR-017 (the console never displays PAN, so it is out of CDE scope) |
| PCI-10 | Audit trails | SEC-15, hash chain, partitioned, retention indefinite |
| PCI-6.4 | Public web app protection | rack-attack, CSP, input validation |
| AT-REST | Sensitive data at rest | AR Encryption (tests: round-trip, tamper fails, nonce uniqueness); volume encryption ⚠ delegated |

## §5 Features & Key Workflows

### Features
- **F-01 Platform foundation** (FOUNDATIONAL):
  - Rails API + Next web + compose (edge/api/worker/web/db/db-replica/redis/mail);
  - env schema fail-fast;
  - `/api/health`;
  - JSON logging + request id;
  - §7 envelope;
  - security headers;
  - rack-attack base;
  - launch scripts.
  - AC: health 200 with all checks; a missing `SECRET_KEY_BASE` exits 1 naming it; SIGTERM exits 0 within 10s.
- **F-02 Identity & sessions** (CRITICAL):
  - signup provisions merchant + owner membership + ledger accounts + sandbox processor accounts + a test API key, in one transaction;
  - login/logout, session read, own sessions list/revoke;
  - password reset; HIBP; lockout.
  - AC: signup → 201 with the cookie set; a breached password → 422 `PASSWORD_BREACHED`; a 14-char password → 422 `PASSWORD_TOO_SHORT`; a revoked session → 401.
  - States: user `active` / `locked` (locked_until).
- **F-03 Email outbox** (HIGH):
  - `pending_emails`; T2 delivery job; SMTP.
  - AC: reset email row `pending` → `sent`; an SMTP failure → `retrying` with next_attempt_at +1m, then +10m, +1h, then `dlq`.
- **F-04 Team & roles** (CRITICAL):
  - members, invitations (token emailed), accept (new or existing user), role change, removal;
  - the last owner is protected.
- **F-05 API keys & modes** (CRITICAL):
  - create (secret once), list, revoke; K auth; mode toggle in the console.
- **F-06 Customers & payment methods** (HIGH):
  - PII encrypted; email lookup via digest; tokens only; redaction.
- **F-07 Processor adapters** (CRITICAL):
  - `ProcessorAdapter` interface (`authorize`, `capture`, `cancel`, `refund`, `attach_payment_method`, `available?`);
  - `SandboxAdapter`: deterministic outcomes per token:
    - `pm_card_visa` → succeeded;
    - `pm_card_mastercard` → succeeded;
    - `pm_card_chargeDeclined` → declined `card_declined`;
    - `pm_card_insufficientFunds` → declined `insufficient_funds`;
    - `pm_card_riskHigh` → succeeded with risk score 90;
    - `pm_card_processorError` → error.
  - `StripeAdapter` (stripe gem, pinned api_version; `SERVICE_UNAVAILABLE` when keys are absent);
  - processor accounts settings;
  - failover: if the default processor returns `unavailable`/`error` and the next enabled account exists, try it (attempt rows record each).
- **F-08 Payment intents** (CRITICAL):
  - create/confirm/capture/cancel; state machine; idempotency; row locks; risk checks;
  - ledger posting on capture; outbox + realtime broadcast.
  - Transition table (pre → event → post):
    - `requires_payment_method` → attach PM / create with PM → `requires_confirmation`;
    - `requires_confirmation` → confirm → `processing`;
    - `processing` → processor succeeded with automatic capture → `succeeded` (ledger capture journal);
    - `processing` → authorized with manual capture → `requires_capture`;
    - `processing` → declined/error → `failed` (failure_code);
    - `requires_capture` → capture (≤ capturable) → `succeeded`;
    - `requires_payment_method` / `requires_confirmation` / `requires_capture` → cancel → `canceled`.
    - Any other edge → 409 `INVALID_STATE_TRANSITION`.
    - A same-state webhook replay is a no-op.
  - Risk rules:
    - amount > `risk_max_amount_minor` → blocked, `failed` with `risk_blocked`;
    - >10 intents per customer in 10 min → score +40;
    - billing country ≠ IP country → score +30.
    - Score ≥80 → blocked.
    - Signals are stored.
  - Concurrency:
    - `SELECT … FOR UPDATE` on the intent for every transition;
    - optimistic `lock_version` on ledger account balances;
    - a duplicate concurrent confirm → one processes, the other gets 409 `IDEMPOTENCY_IN_PROGRESS` (same key) or `INVALID_STATE_TRANSITION`.
- **F-09 Ledger & reporting** (CRITICAL):
  - accounts per merchant/mode/currency;
  - balanced journals:
    - capture: Dr processor_receivable amount / Cr merchant_payable (amount − fee) / Cr platform_fees fee, where fee = 2.9% + 30 minor units, rounded half-up in integer math;
    - refund: Dr merchant_payable / Cr processor_receivable;
    - dispute open: Dr dispute_losses / Cr processor_receivable;
    - dispute won: reversal;
    - dispute lost: Dr merchant_payable / Cr dispute_losses.
  - DB triggers enforce immutability and balance; the T3 integrity job; summary/timeseries reads go to the replica; cached 15s.
- **F-10 Refunds & disputes** (CRITICAL):
  - refund partial or full (Σ refunds ≤ captured); refund state `pending → succeeded | failed`;
  - dispute `needs_response → under_review → won | lost`; evidence submission; sandbox simulate/resolve.
- **F-11 Inbound processor webhooks** (CRITICAL):
  - `/v1/webhooks/stripe` verifies, dedupes and applies `payment_intent.succeeded|payment_failed|amount_capturable_updated`, `charge.refunded`, `charge.dispute.created|closed`;
  - the anomaly flag; the event log screen.
- **F-12 Outbound webhooks** (CRITICAL):
  - endpoints CRUD, secret rotation, test event, outbox delivery (T1), retries/dlq, manual retry, deliveries screen.
- **F-13 Realtime** (HIGH):
  - Action Cable `MerchantEventsChannel`; the console live feed on the dashboard and payments list.
- **F-14 Dashboard & analytics** (MEDIUM):
  - KPI tiles, timeseries chart, status/processor breakdown, CSV export.
- **F-15 Audit & governance evidence** (HIGH):
  - audit list with filters; chain verification; evidence export bundle (audit rows + chain head + invariants summary + version).
- **F-16 Merchant settings & logo** (HIGH):
  - profile, risk ceiling, statement descriptor; the logo pipeline (§4.3).
- **F-17 Scheduled operations** (HIGH):
  - T1–T6 jobs, `scheduled_runs` claim + counts, the ops trigger.
- **F-18 Platform operator console** (CRITICAL — cross-tenant):
  - merchants list, suspend/reactivate (audit), DLQ list, scheduled runs.
- **F-19 Public site & docs** (LOW):
  - landing (agentic commerce: mandates, authorize → settle), `/docs` API reference, SEO artifacts, designed 404.
- **F-20 Agent mandates — agentic authorization & settlement** (CRITICAL — money movement, ADR-057):
  - A developer issues a mandate to an AI agent: one currency, a per-transaction cap, a total cap, a capture method (default `manual`), an expiry (default 30 days, at most 365) and optionally one customer. The `ak_` secret is shown once; only its digest is stored. Issue and revoke are audited.
  - The agent authenticates with `Bearer ak_…`, in the mandate's mode. Revoked, expired and unknown mandates are 401 `UNAUTHENTICATED`.
  - On create: currency must be the mandate's; amount ≤ per-transaction cap; the mandate's committed spend (open intents at full amount, succeeded at the captured amount, failed/canceled at zero) plus the amount ≤ total cap, checked under the mandate's row lock; otherwise 403 `MANDATE_LIMIT_EXCEEDED` with `limit` (`per_transaction`|`total`). Customer and capture method come from the mandate; saved payment methods only through a mandate bound to their customer.
  - Settlement is capture (full or partial; the uncaptured remainder is released against the cap) by the agent or the merchant, posting the usual ledger journal and outbox events. Audit events carry `actor_type: agent`, `actor_id` the mandate's public id.
  - Revoking a mandate stops new requests at once; open authorizations remain for the merchant to capture or cancel.

### Error paths & recovery (comprehensive)
- **Processor timeout (>10s) or 5xx** → attempt `error`; failover per F-07; otherwise intent `failed` with `processor_unavailable`. The idempotency row is stored with the 402/502 response, so a client retry with the same key replays it and does not re-charge.
- **Crash between the processor call and commit** → the idempotency row stays `locked_at` without `completed_at`. A retry after 60s re-claims it and calls the processor with the same processor idempotency key, which yields the same processor result (no double charge).
- **Ledger posting failure** (constraint) → the whole transaction rolls back, the intent stays `processing`, and the T3 job reports it.
- **Webhook for an unknown intent** → stored with `anomaly: unmatched_reference`, 200 returned (no retry storm).
- **Outbound delivery non-2xx/timeout** → `retrying` with backoff; after 5 attempts `dlq` + audit; the operator or merchant can retry.

### Input validation boundaries
- `amount_minor`: 1 … 99_999_999 (≤ the merchant risk ceiling to pass risk).
- `currency` ∈ {usd, eur, gbp, cad, aud, jpy}. `jpy` has exponent 0.
- `description` ≤ 500.
- `metadata`: ≤20 keys, key ≤40, value ≤500.
- Email RFC-lite ≤254.
- `name` 1–120.
- `password` 15–128.
- Webhook URL: https, ≤2048, public host.
- `enabled_events`: 1–16 from the event catalog.
- Evidence fields ≤5000.
- `limit` 1–100.

### Key Workflows

RESEARCH Step 3 key workflows → W mapping. The deferred ones (KW3 intelligent routing, KW5 advanced fraud, KW7 ACH, KW8 BTC/Lightning, KW10 sophisticated reconciliation, KW12 subscriptions, KW15 AI operations) are Non-Goals. KW4 failover is covered within W3.

#### W1 — Merchant onboarding & tenant provisioning (KW1)
- W1.1 Visitor opens `/sign-up`, enters name, email, password (≥15), merchant name, country, currency.
- W1.2 The API provisions the merchant, owner membership, ledger accounts, sandbox processor account and test API key atomically.
- W1.3 The console lands on `/dashboard` in test mode with an onboarding checklist and the one-time test key reveal.
- W1.4 **Terminal:** the owner opens `/developers/api-keys` and sees the provisioned test key (prefix) listed.

#### W2 — Unified payment processing (KW2, KW5 basic risk, realtime)
- W2.1 The developer opens `/payments/new`, picks amount, currency, capture method, an optional customer and a sandbox card token.
- W2.2 Submit → `POST /v1/payment_intents` with `confirm: true` and an Idempotency-Key.
- W2.3 The intent reaches `succeeded` (or `requires_capture` for manual capture). The dashboard live feed shows it in realtime.
- W2.4 For manual capture, the user captures from `/payments/[id]`.
- W2.5 **Terminal:** `/payments/[id]` shows the status, the attempts timeline, risk signals and the linked journal entry.

#### W3 — Processor configuration & failover (KW3 static, KW4)
- W3.1 The admin opens `/settings/processors`, sees Sandbox (available) and Stripe (unavailable — keys not configured).
- W3.2 The admin sets Stripe as default with Sandbox enabled as fallback.
- W3.3 A test payment is attempted; the Stripe attempt records `unavailable` and Sandbox succeeds.
- W3.4 **Terminal:** the payment detail shows two attempts (stripe unavailable → sandbox succeeded).

#### W4 — Customers & tokenized payment methods (KW6)
- W4.1 The user creates a customer on `/customers`.
- W4.2 On `/customers/[id]` they attach a sandbox token. The response stores brand/last4/exp only.
- W4.3 **Terminal:** the customer detail lists the card as brand •••• last4; the user can detach it.

#### W5 — Ledger & financial reporting (KW9, KW13)
- W5.1 The analyst opens `/ledger` and sees account balances per currency (Σ balanced).
- W5.2 They open a journal entry `/ledger/entries/[id]` showing balanced debit/credit lines and the source payment.
- W5.3 They open `/reports`, choose a date range and currency, and view the KPIs and timeseries.
- W5.4 **Terminal:** export CSV of transactions for the range (download).

#### W6 — Refunds & disputes (KW11)
- W6.1 The admin opens a succeeded payment and issues a partial refund (amount, reason).
- W6.2 The refund becomes `succeeded`; the payment shows the refunded amount; a ledger refund journal is posted.
- W6.3 In test mode, the admin simulates a dispute (`/developers/sandbox`). The dispute is `needs_response`.
- W6.4 On `/disputes/[id]` the admin submits evidence → `under_review`.
- W6.5 **Terminal:** the admin resolves it via sandbox as won/lost; the dispute detail shows the outcome and the ledger journal.

#### W7 — Developer API, webhooks & sandbox (KW14)
- W7.1 The developer creates an API key on `/developers/api-keys` (secret shown once, copy).
- W7.2 The developer adds a webhook endpoint on `/developers/webhooks` (URL, events) and receives its signing secret once.
- W7.3 They send a test event from `/developers/webhooks/[id]`. The delivery appears with its status.
- W7.4 They inspect the event log `/developers/events`.
- W7.5 **Terminal:** they retry a failed/dlq delivery and see it re-queued.

#### W8 — Team access management (KW1 extension)
- W8.1 The owner invites a teammate by email with a role on `/settings/team`.
- W8.2 The invitee opens `/invite/[token]`, sets name + password and joins.
- W8.3 **Terminal:** the owner changes the member's role; the member list reflects it.

#### W9 — Governance & evidence (KW16)
- W9.1 The analyst opens `/governance` and sees the audit trail with filters (action, date).
- W9.2 They run chain verification and see `valid` with the head hash.
- W9.3 **Terminal:** they export an evidence bundle for a date range and download it.

#### W10 — Platform operations (comprehensive expansion)
- W10.1 The operator opens `/ops` and sees merchants with status.
- W10.2 The operator suspends a merchant with a reason (audited). That merchant's API calls return 403 `MERCHANT_SUSPENDED`.
- W10.3 The operator reviews the DLQ and scheduled-run outcomes.
- W10.4 **Terminal:** the operator reactivates the merchant.

#### W11 — Account security self-service (F-02)
- W11.1 The user requests a password reset at `/forgot-password`. The email is queued and delivered to Mailpit.
- W11.2 The user sets a new password at `/reset-password/[token]`. All sessions are revoked.
- W11.3 **Terminal:** the user signs in and reviews/revokes their sessions on `/settings/security`.

#### W12 — Agentic authorization & settlement (F-20)
- W12.1 A developer opens `/agents` and issues a mandate with caps and an expiry; the secret is shown once.
- W12.2 The agent creates and confirms a manual-capture intent with the mandate; the funds are held and the mandate's remaining spend drops.
- W12.3 An authorization that would cross the total cap is refused with 403 `MANDATE_LIMIT_EXCEEDED`.
- W12.4 The merchant opens the agent's payments and captures part of the hold; the remainder is released to the mandate.
- W12.5 **Terminal:** the developer revokes the mandate; the agent's next request is 401.

## §6 UI Surface

| Screen | Route | Example | Roles | Binds | Workflow steps | User controls | States |
|---|---|---|---|---|---|---|---|
| Landing | `/` | `/` | public | — | — | nav, CTA, routing demo segment buttons | success |
| API docs | `/docs` | `/docs` | public | — | W7 | section nav | success |
| Sign up | `/sign-up` | `/sign-up` | public | `POST /v1/auth/signup` | W1.1, W1.2 | name, email, password, merchant name, country, default currency | loading, error, success |
| Sign in | `/sign-in` | `/sign-in` | public | `POST /v1/auth/login` | W11.3 | email, password | loading, error, success |
| Forgot password | `/forgot-password` | `/forgot-password` | public | `POST /v1/auth/password_resets` | W11.1 | email | loading, error, success |
| Reset password | `/reset-password/[token]` | `/reset-password/abc` | public | `PUT /v1/auth/password_resets/:token` | W11.2 | password | loading, error, success |
| Accept invite | `/invite/[token]` | `/invite/abc` | public | `GET /v1/invitations/:token`, `POST /v1/invitations/:token/accept` | W8.2 | name, password | loading, error, success |
| Dashboard | `/dashboard` | `/dashboard` | all | `GET /v1/auth/session`, `GET /v1/reports/summary`, `GET /v1/reports/timeseries`, `GET /v1/payment_intents`, `GET /cable`, `POST /v1/me/active_merchant`, `DELETE /v1/auth/session` | W1.3, W2.3 | mode toggle, merchant switcher, range select | empty, loading, error, success |
| Payments | `/payments` | `/payments` | all | `GET /v1/payment_intents` | W2.3, W12.4 | status filter, currency filter, date from/to, customer filter, agent mandate filter (from `/agents`), pagination | empty, loading, error, success |
| New payment | `/payments/new` | `/payments/new` | owner, admin, developer | `POST /v1/payment_intents`, `GET /v1/customers` | W2.1, W2.2, W3.3 | amount, currency, capture method, customer, card token, processor override, description | loading, error, success |
| Payment detail | `/payments/[id]` | `/payments/pi_demo` | all | `GET /v1/payment_intents/:id`, `POST /v1/payment_intents/:id/confirm`, `POST /v1/payment_intents/:id/capture`, `POST /v1/payment_intents/:id/cancel`, `POST /v1/refunds` | W2.4, W2.5, W3.4, W6.1, W6.2 | capture amount, cancel reason, refund amount, refund reason | loading, error, success |
| Customers | `/customers` | `/customers` | all | `GET /v1/customers`, `POST /v1/customers` | W4.1 | email search, create form (email, name, phone, country), pagination | empty, loading, error, success |
| Customer detail | `/customers/[id]` | `/customers/cus_demo` | all | `GET /v1/customers/:id`, `PATCH /v1/customers/:id`, `DELETE /v1/customers/:id`, `POST /v1/customers/:id/payment_methods`, `DELETE /v1/payment_methods/:id` | W4.2, W4.3 | edit fields, card token select, detach, redact | loading, error, success |
| Refunds | `/refunds` | `/refunds` | all | `GET /v1/refunds`, `GET /v1/refunds/:id` | W6.2 | status filter, pagination | empty, loading, error, success |
| Disputes | `/disputes` | `/disputes` | all | `GET /v1/disputes` | W6.3 | status filter, pagination | empty, loading, error, success |
| Dispute detail | `/disputes/[id]` | `/disputes/dp_demo` | all | `GET /v1/disputes/:id`, `POST /v1/disputes/:id/evidence` | W6.4, W6.5 | evidence fields | loading, error, success |
| Ledger | `/ledger` | `/ledger` | owner, admin, analyst | `GET /v1/ledger/accounts`, `GET /v1/ledger/journal_entries` | W5.1 | currency select, kind filter, account filter, date from/to, pagination | empty, loading, error, success |
| Journal entry | `/ledger/entries/[id]` | `/ledger/entries/je_demo` | owner, admin, analyst | `GET /v1/ledger/journal_entries/:id` | W5.2 | — | loading, error, success |
| Reports | `/reports` | `/reports` | all (export: owner, admin, analyst) | `GET /v1/reports/summary`, `GET /v1/reports/timeseries`, `GET /v1/reports/transactions.csv` | W5.3, W5.4 | from, to, currency, granularity, export status filter | empty, loading, error, success |
| API keys | `/developers/api-keys` | `/developers/api-keys` | owner, admin, developer | `GET /v1/api_keys`, `POST /v1/api_keys`, `DELETE /v1/api_keys/:id` | W1.4, W7.1 | name, mode, revoke | empty, loading, error, success |
| Agent mandates | `/agents` | `/agents` | owner, admin, developer | `GET /v1/agent_mandates`, `POST /v1/agent_mandates`, `DELETE /v1/agent_mandates/:id` | W12.1, W12.4, W12.5 | name, currency, per-payment limit, total limit, capture method, expiry, customer, revoke | empty, loading, error, success |
| Webhooks | `/developers/webhooks` | `/developers/webhooks` | owner, admin, developer | `GET /v1/webhook_endpoints`, `POST /v1/webhook_endpoints`, `GET /v1/webhook_deliveries` | W7.2 | url, description, enabled events, delivery status filter | empty, loading, error, success |
| Webhook endpoint | `/developers/webhooks/[id]` | `/developers/webhooks/we_demo` | owner, admin, developer | `PATCH /v1/webhook_endpoints/:id`, `DELETE /v1/webhook_endpoints/:id`, `POST /v1/webhook_endpoints/:id/rotate_secret`, `POST /v1/webhook_endpoints/:id/test`, `GET /v1/webhook_deliveries`, `POST /v1/webhook_deliveries/:id/retry` | W7.3, W7.5 | url, events, status toggle, test event type, retry | loading, error, success |
| Events | `/developers/events` | `/developers/events` | all | `GET /v1/events`, `GET /v1/events/:id` | W7.4 | type filter, source filter, pagination | empty, loading, error, success |
| Sandbox | `/developers/sandbox` | `/developers/sandbox` | owner, admin, developer | `POST /v1/sandbox/disputes`, `POST /v1/sandbox/disputes/:id/resolve`, `GET /v1/payment_intents` | W6.3, W6.5 | payment select, outcome select | empty, loading, error, success |
| Settings | `/settings` | `/settings` | owner, admin (read: all) | `GET /v1/merchant`, `PATCH /v1/merchant`, `POST /v1/merchant/logo`, `GET /v1/merchant/logo/:variant` | — | name, support email, descriptor, default currency, risk ceiling, logo upload | loading, error, success |
| Processors | `/settings/processors` | `/settings/processors` | owner, admin | `GET /v1/processor_accounts`, `PATCH /v1/processor_accounts/:id` | W3.1, W3.2 | enabled toggle, priority, default | loading, error, success |
| Team | `/settings/team` | `/settings/team` | owner, admin (read: all) | `GET /v1/members`, `PATCH /v1/members/:id`, `DELETE /v1/members/:id`, `GET /v1/invitations`, `POST /v1/invitations`, `DELETE /v1/invitations/:id` | W8.1, W8.3 | invite email, invite role, role select, remove, revoke invite | empty, loading, error, success |
| Security | `/settings/security` | `/settings/security` | all | `GET /v1/me/sessions`, `DELETE /v1/me/sessions/:id` | W11.3 | revoke session | loading, error, success |
| Governance | `/governance` | `/governance` | owner, admin, analyst | `GET /v1/audit_events`, `GET /v1/governance/verification`, `POST /v1/governance/evidence_exports`, `GET /v1/governance/evidence_exports/:id` | W9.1, W9.2, W9.3 | action filter, actor filter, date from/to, export range | empty, loading, error, success |
| Operations | `/ops` | `/ops` | platform operator | `GET /v1/ops/merchants`, `PATCH /v1/ops/merchants/:id`, `GET /v1/ops/dlq`, `GET /v1/ops/scheduled_runs` | W10.1–W10.4 | status filter, search, suspend reason, dlq kind, job filter | empty, loading, error, success |

**Defaulted parameters (not user controls, with rationale):**
- `limit` is fixed at 25 per page (pagination cursor = "Load more").
- Payment `metadata` is API-only in v1, since the console is for test payments.
- `mode` on lists comes from the global mode toggle.

### Design System
- **Archetype:** saas (DESIGN.md Part II) for the console; the template drives the landing page.
- **Tokens (binding, verbatim from the template):** `--bg #050711`, `--bg2 #0A1020`, `--surf #10192D`, `--surf2 #141F38`, `--line #1C2742`, `--line2 #2A3A62`, `--cyan #00E5FF`, `--violet #8257FF` (primary), `--blue #287BFF`, `--green #00E6A7` (success), `--amber #FFB547` (warning/demo), `--coral #FF5470` (danger), `--text #F4F8FF`, `--muted #8D9AB5`, `--dim #6B7894`, rails `--c-stripe` etc., `--r 12px`, `--maxw 1240px`. A light theme is derived (DECISIONS).
- **Type:** Space Grotesk (display/headings, uppercase tracking on hero/section titles), Inter (UI), JetBrains Mono (ids, amounts, code, eyebrows). Scale: 12/14/16/20/24/32/48 plus a clamp hero.
- **Layout:**
  - Console: app shell with a 248px sidebar (collapses to a focus-trapped drawer below 768px); a 64px top bar with mode toggle, merchant switcher, theme toggle and user menu; content max 1240px.
  - Dashboard hierarchy: KPI row (4) → timeseries chart + live feed → status breakdown + processor mix.
- **Components:**
  - `.panel` surfaces;
  - buttons 44px (`data-primary` on primary CTAs);
  - status badges, color + text (never color alone);
  - tables: sticky header, mono numerics right-aligned, row hover, card layout below 640px;
  - forms: labels above, inline errors bound to fields via `aria-describedby`;
  - toasts (`role=status`);
  - dialogs (focus trap);
  - empty states with an action;
  - skeleton loaders.
- **Motion:** 150–250ms ease-out; hero canvas and marquee disabled under `prefers-reduced-motion`.

## §7 Error contract

Every non-2xx response is `application/problem+json`:

```json
{ "type": "https://rails.mdlc.ai/errors/validation-failed", "title": "Validation failed", "status": 422,
  "detail": "2 fields failed validation", "instance": "/api/v1/customers",
  "code": "VALIDATION_FAILED", "requestId": "<x-request-id>",
  "errors": [ { "field": "email", "message": "Email is invalid", "code": "INVALID" } ] }
```

| `code` | status | When |
|---|---|---|
| `VALIDATION_FAILED` | 422 | Schema/param validation; `errors[]` populated |
| `BAD_REQUEST` | 400 | Malformed JSON, missing Idempotency-Key where required (`IDEMPOTENCY_KEY_REQUIRED` 400) |
| `UNAUTHENTICATED` | 401 | No or invalid session/key |
| `FORBIDDEN` | 403 | Role lacks permission (own tenant); `CSRF_FAILED` 403; `MERCHANT_SUSPENDED` 403 |
| `MANDATE_LIMIT_EXCEEDED` | 403 | An agent mandate's per-transaction or total cap would be crossed (`limit`, `limit_minor`, `spent_minor?`) |
| `NOT_FOUND` | 404 | Missing, or belongs to another tenant/user |
| `INVALID_STATE_TRANSITION` | 409 | Illegal payment/refund/dispute transition |
| `IDEMPOTENCY_IN_PROGRESS` | 409 | Same key in flight |
| `IDEMPOTENCY_KEY_REUSED` | 422 | Same key, different body |
| `PASSWORD_TOO_SHORT` / `PASSWORD_BREACHED` | 422 | Password policy |
| `PAN_NOT_ACCEPTED` | 422 | PAN-like data submitted |
| `UNSUPPORTED_MEDIA_TYPE` | 415 | Upload type |
| `PAYLOAD_TOO_LARGE` | 413 | Upload/body > limit |
| `RATE_LIMITED` | 429 | Throttled (+ `Retry-After`) |
| `PAYMENT_DECLINED` | 402 | Processor decline (the intent body is included as `payment_intent`) |
| `RISK_BLOCKED` | 402 | Risk rules |
| `SERVICE_UNAVAILABLE` | 503 | Processor not configured (contract mode) / dependency down |
| `INTERNAL_ERROR` | 500 | Unhandled; no stack trace |

## §8 Configuration & environment

| Variable | Purpose | Required | Service |
|---|---|---|---|
| `RAILS_ENV` | Environment | yes | api, worker |
| `SECRET_KEY_BASE` | Cookie/session signing (≥64 hex) | yes | api, worker |
| `DATABASE_URL` | Primary Postgres | yes | api, worker |
| `DATABASE_REPLICA_URL` | Read replica (falls back to primary if empty) | optional | api, worker |
| `REDIS_URL` | rack-attack, cache, cable | yes | api, worker |
| `AR_ENCRYPTION_PRIMARY_KEY` / `AR_ENCRYPTION_DETERMINISTIC_KEY` / `AR_ENCRYPTION_KEY_DERIVATION_SALT` | Active Record Encryption | yes | api, worker |
| `BLIND_INDEX_KEY` | HMAC key for email digests | yes | api, worker |
| `APP_ORIGIN` | Public origin (`https://localhost:9443`) | yes | api, web |
| `ALLOWED_ORIGINS` | CSRF/cable origin allow-list (comma) | yes | api |
| `STRIPE_SECRET_KEY` | Stripe API key (contract mode when empty) | optional | api, worker |
| `STRIPE_WEBHOOK_SECRET` | Stripe signing secret | optional | api |
| `STRIPE_API_VERSION` | Pinned API version (default `2026-03-25.dahlia`) | optional | api |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_FROM` | Email delivery | yes (host/port), optional (from) | worker |
| `HIBP_ENABLED` | Enable breach screening (default true) | optional | api |
| `HIBP_BASE_URL` | Range API base (default `https://api.pwnedpasswords.com`) | optional | api |
| `LOG_LEVEL` | info/debug/warn | optional | api, worker, web |
| `RAILS_MAX_THREADS` / `WEB_CONCURRENCY` | Puma sizing | optional | api |
| `ALLOW_RATE_LIMIT_SCALE` / `RATE_LIMIT_SCALE` | Non-prod load-test multiplier | optional | api |
| `OPS_TRIGGER_ENABLED` | Enables `POST /v1/ops/scheduled_runs` (refused in production) | optional | api |
| `OPERATOR_EMAIL` / `OPERATOR_PASSWORD` | Seeds the platform operator on first boot | optional | api |
| `APP_VERSION` / `GIT_COMMIT` | Build provenance baked at image build | yes (baked) | api, web |
| `HTTPS_PORT` / `HTTP_PORT` | Edge host ports (default 9443/9080) | optional | edge |
| `TRUSTED_CERT` | Enable HSTS (only with a trusted cert) | optional | edge |
| `API_INTERNAL_URL` | Web → API (server-side only) | optional | web |

## §9 Performance targets

Derived from RESEARCH success metrics and the `large` tier.
- Platform-controlled operations (excluding processor latency) p95 < 250 ms.
- Read endpoints p95 < 50 ms.
- Error rate < 0.1%.
- Zero duplicate charges.

| Operation | Target (p95) | Load-test scenario |
|---|---|---|
| `GET /api/health` | < 50 ms | — |
| `GET /v1/payment_intents` (25 rows, 100k-row tenant) | < 50 ms | read mix |
| `GET /v1/payment_intents/:id` | < 50 ms | read mix |
| `GET /v1/reports/summary` (cached) | < 50 ms | read mix |
| `POST /v1/payment_intents` confirm (sandbox) | < 250 ms | write mix |
| `POST /v1/refunds` | < 250 ms | write mix |
| Idempotent replay | < 50 ms | replay |
| `POST /v1/auth/login` | < 400 ms (bcrypt cost 12 dominates; excluded from the 250ms platform target, documented) | auth |

**Load-test parameters:**
- Seed 2 merchants × 100,000 payment intents + ledger journals.
- 5 minutes sustained per mix at 200 virtual users (local single-node), plus a step-up to saturation.
- `RATE_LIMIT_SCALE=1000` in non-prod only, with the restore proven.

Throughput and headroom are reported per node. 50k concurrent is reached by horizontal scaling (N api replicas behind a load balancer + cable via Redis), and the per-node capacity is stated so N can be computed.

## §10 Test plan

| Feature / Workflow | Unit / service (RSpec) | Request (RSpec) | Web (Vitest) | E2E (Playwright) | Smoke |
|---|---|---|---|---|---|
| F-01 | env schema, problem envelope | health, 404 envelope, headers | api client | — | health, SIGTERM, fail-fast, provenance, 16KB cookie |
| F-02 | password policy, HIBP client | signup/login/logout/reset/lockout/rate limit | forms | W1, W11 | auth bootstrap |
| F-03 | outbox transitions | — | — | W11 | email delivered (Mailpit API) |
| F-04 | — | invite/accept/role/last-owner, cross-user | — | W8 | same-tenant cross-user |
| F-05 | key digest | create/revoke/K auth/mode | — | W7 | — |
| F-06 | PAN guard, encryption round-trip/tamper/nonce | CRUD, tenant 404 | — | W4 | cross-tenant |
| F-07 | sandbox outcomes, failover, stripe adapter (WebMock) | processor settings | — | W3 | [contract] stripe |
| F-08 | state machine, idempotency, risk | create/confirm/capture/cancel, replay, reuse, concurrency | — | W2 | primary CRUD, idempotent replay |
| F-09 | posting balance, trigger immutability | ledger/reports/csv | — | W5 | — |
| F-10 | refund caps, dispute lifecycle | refunds/disputes/sandbox | — | W6 | — |
| F-11 | signature verify, dedupe, anomaly | stripe webhook | — | — | webhook reception + replay |
| F-12 | SSRF guard, HMAC, backoff/dlq | endpoints/deliveries/retry | — | W7 | outbound delivery effect |
| F-13 | channel authz, origin | — | realtime context | W2 | — |
| F-14 | aggregates | summary/timeseries | chart wrapper | W5 | — |
| F-15 | hash chain, verify | audit/verify/export | — | W9 | — |
| F-16 | logo processor (EXIF, HEIC, magic bytes) | upload/serve | — | — | binary asset delivery |
| F-17 | T1–T6 effect specs (`api/spec/jobs/`) | ops trigger | — | — | background job effect + worker liveness |
| F-18 | — | ops authz (non-operator 404), suspend | — | W10 | — |
| F-19 | — | — | primitives | landing in baseline | web baseline |
| F-20 | spend derivation, caps under lock | issue/validate/caps/release/confinement/binding/revoke/expire/mode | nav | W12 | — |
