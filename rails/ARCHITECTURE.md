# ARCHITECTURE.md — RΛILS by MDLC

> ONE API. EVERY PAYMENT RAIL. — "The intelligence layer for money movement."
> Project `cd32b70e-34d4-468e-8d26-6afe7aab6492` · build_depth `comprehensive` · review_gates `auto` · scale tier **large**

## 1. Problem, users, scope

**Problem.** Merchants and platforms integrate each processor separately. Payment state, idempotency and financial records end up scattered across vendors, so duplicate charges, untraceable states and reconciliation gaps are routine.

**Solution.** RΛILS v1 is a multi-tenant payment-orchestration control plane:
- one REST API (`/api/v1`) for payment intents across processor adapters;
- a deterministic payment state machine with exactly-once idempotency;
- a double-entry ledger as the financial system of record;
- signed inbound and outbound webhooks with a developer sandbox;
- a realtime merchant console;
- a tamper-evident governance evidence trail.

**Users.**
- Merchant owners/admins (onboard, configure, monitor).
- Developers (API keys, sandbox, webhooks).
- Finance analysts (ledger, reports, export).
- Viewers (read-only).
- RΛILS platform operators (cross-tenant ops console).

**In scope (PRIMARY MVP WORKFLOWS + comprehensive expansion):**
- onboarding and tenant provisioning;
- payment intents;
- a Stripe adapter behind a reusable interface, plus a Sandbox rail;
- deterministic state and idempotency;
- double-entry ledger and reporting;
- refunds and disputes;
- signed webhooks (in and out) and the developer sandbox;
- governance and evidence;
- team/RBAC;
- realtime feed;
- analytics/CSV export;
- the platform operator console;
- scheduled operations;
- the public site.

**Non-Goals (from RESEARCH "PRIMARY MVP WORKFLOWS"; deferred, not built):**
- multi-processor *intelligent* routing (v1 ships a static, merchant-chosen processor plus failover from Stripe to the Sandbox rail only in test mode);
- advanced/ML fraud intelligence (v1 has basic rule checks only);
- ACH;
- Bitcoin/Lightning/digital assets;
- sophisticated settlement reconciliation (v1 runs ledger-integrity verification only);
- subscription billing;
- AI-assisted operations;
- Fiserv/Worldpay adapters;
- mobile apps;
- MFA (TOTP, deferred — ADR-005).

## 2. System architecture

```
                Browser (console + public site)            Merchant servers (sk_ keys)
                         │ HTTPS :${HTTPS_PORT:-9443}                │
                         ▼                                           ▼
                 ┌──────────────── Caddy 2.11 edge (tls internal, :80→:443 redirect) ─────────────┐
                 │  /api/*, /cable  ─────────────►  api (Rails 8.1 Puma, stateless, N replicas)    │
                 │  everything else ────────────►  web (Next.js 16 standalone, node server.js)    │
                 └────────────────────────────────────────────────────────────────────────────────┘
                                         │                     │
                     ┌───────────────────┼─────────────┐       │ Action Cable (Redis pub/sub)
                     ▼                   ▼             ▼       ▼
             Postgres 18 primary ──► Postgres 18 replica    Redis 7 (rack-attack counters, cache, cable)
                     ▲  (writes, queue)   (reports/reads)
                     │
             worker (Solid Queue supervisor: jobs + recurring.yml) ──► Stripe API / merchant webhook URLs / SMTP (Mailpit)
```

**Tiers:**

| Tier | Image | Notes |
|---|---|---|
| `edge` | `caddy:2.11-alpine` | Terminates TLS (`tls internal`), redirects HTTP→HTTPS, security headers. HSTS is off on the internal CA (it would pin a cert the browser does not trust) and on only with `TRUSTED_CERT=true`. Header buffers are generous by default (ADR-012). |
| `api` | `ruby:3.4-slim` multi-stage, non-root, `tini` PID 1 | Rails 8.1 API-only on Puma. Stateless: sessions live in an encrypted cookie keyed by `SECRET_KEY_BASE` with a server-side session row for revocation. |
| `worker` | same image as `api` | `bin/jobs` (Solid Queue supervisor). Runs jobs plus `config/recurring.yml`. |
| `web` | `node:22-alpine` multi-stage | Next.js 16 `output: 'standalone'`, `node --max-http-header-size=32768 server.js`. |
| `db` | `postgres:18-alpine` | Primary, with WAL configured for streaming replication. |
| `db-replica` | `postgres:18-alpine` | Bootstrapped from `pg_basebackup -R`; read-only, used for reporting reads. |
| `redis` | `redis:7-alpine` | AOF off; ephemeral counters, cache and cable. |
| `mail` | `axllent/mailpit` | Local SMTP sink. Its UI is not published, so it is not exposed publicly. |

**Scale tier `large` (50k+ concurrent), resolved per kernel table:**

| Subsystem | Resolution |
|---|---|
| DB & pooling | Primary + streaming read replica. Rails multi-DB `connects_to` with writing/reading roles; reports and dashboards read from the replica via `ApplicationRecord.connected_to(role: :reading)`. Pool sized `RAILS_MAX_THREADS`. Hot append-only table `audit_events` is partitioned by month (declarative range partitioning). |
| Caching | Redis cache store; dashboard metrics cache-aside with a 15s TTL plus explicit key bust on payment state change. Static assets are immutable-cached at the edge; a CDN is delegated to production. |
| Async / jobs | Solid Queue with separate queues (`critical`, `webhooks`, `mailers`, `default`), `limits_concurrency` per endpoint, and a dead-letter state on outbox rows. Workers are horizontally scalable (`docker compose up --scale worker=N`). |
| Compute & deploy | Stateless `api`/`web` (`--scale api=N`); no sticky state (cable via Redis). Multi-AZ is delegated to the production orchestrator. |
| Rate limiting | rack-attack on Redis: per IP, per API key, per login email, plus cost-weighted throttles on money-moving POSTs. Edge enforcement is delegated (ADR-011). |
| Perf verification | Sustained load test at a stated concurrency, with headroom reported (Stage 4). |

Signal bumps: `has_payments` adds row locks (`SELECT … FOR UPDATE`) on payment intents and ledger balances; `has_websocket` adds a Redis cable adapter and a per-connection budget.

## 3. Modules (api)

| Module | Path | Responsibility |
|---|---|---|
| Config / env schema | `api/config/initializers/00_env_schema.rb` → `Rails::EnvSchema` (`api/lib/env_schema.rb`) | Validates every env var at boot. A missing or malformed value aborts with exit 1, naming the variable. |
| Request context | `api/app/models/current.rb`, `api/app/controllers/concerns/authentication.rb` | `Current.user / merchant / membership / api_key / mode / request_id` |
| Tenant scope | `api/app/controllers/concerns/tenant_scoped.rb` | The only path to tenant data is `current_merchant.<association>`. Unscoped finds are banned in controllers (INV). |
| Authorization | `api/app/controllers/concerns/authorization.rb`, `api/app/policies/team_policy.rb` | Role→permission matrix (SPEC §4.2). Membership role checked per action. |
| Errors | `api/app/controllers/concerns/problem_responses.rb`, `api/app/services/problem.rb` | RFC 7807 envelope with `code` and `requestId`. Cross-tenant access → 404. |
| Payments domain | `api/app/services/payments/*` | `CreateIntent`, `ConfirmIntent`, `CaptureIntent`, `CancelIntent`, `StateMachine`, `RiskCheck` |
| Processors | `api/app/services/processors/*` | `ProcessorAdapter` (interface), `SandboxAdapter`, `StripeAdapter`, `Registry` |
| Idempotency | `api/app/services/idempotency/*`, `idempotency_keys` table | Claims a UNIQUE (merchant_id, key) row in its own transaction; fingerprints the request; replays the stored response. |
| Ledger | `api/app/services/ledger/*` | `Post` (balanced journal), `Accounts`; DB triggers enforce immutability and balance. |
| Refunds/Disputes | `api/app/services/refunds/*`, `api/app/services/disputes/*` | Lifecycle state machines, with ledger postings |
| Webhooks in | `api/app/controllers/api/v1/webhooks/*`, `api/app/services/inbound_webhooks/*` | Verify signature on the raw body → dedupe (`webhook_events` UNIQUE (source, event_id)) → apply |
| Webhooks out | `api/app/services/outbound_webhooks/*`, `pending_webhooks` table | Outbox written in the same txn; HMAC at insert; dispatcher with `SsrfGuard.assert_safe_url!` before `Net::HTTP`; backoff 1m/5m/30m/2h/6h → dlq |
| Email | `api/app/services/emails/*`, `pending_emails` table | Outbox; retries 1m/10m/1h → dlq; Action Mailer SMTP |
| Realtime | `api/app/channels/*` | `MerchantEventsChannel` streams `merchant:<id>:<mode>`; Origin allow-list; message budget |
| Audit | `api/app/services/audit.rb` | `Audit.record!` writes a hash-chained `audit_events` row (sha256 of prev_hash ‖ canonical JSON) |
| Governance evidence | `api/app/services/governance/*` | Chain verification; evidence bundle export (JSON) |
| Scheduled ops | `api/app/jobs/*`, `config/recurring.yml` | Triggers listed in §6 |
| Uploads | `api/app/services/uploads/logo_processor.rb` | Magic-byte sniff, EXIF strip via re-encode (`ruby-vips`), derivatives 64/256 px, 1MB quota |
| Passwords | `api/app/services/passwords/*` | Length policy, HIBP k-anonymity client (fail-open-with-flag, ADR-006), bcrypt cost 12 |
| Observability | `api/config/initializers/logging.rb` | JSON lines via lograge plus a custom formatter. `request_id` is echoed on `X-Request-Id`. Redaction filter covers params plus PII columns. |

**Web (`web/`):** Next.js App Router.
- `app/(marketing)/` holds the public site (template port).
- `app/(auth)/` holds sign-in, sign-up and password reset.
- `app/(console)/` holds the authenticated app shell.
- `lib/api.ts` is the single fetch client (base `/api/v1`, never double-prefixed). It sends the CSRF header and parses the problem+json envelope.
- React Context providers: Session (with the test/live mode) in `components/session-provider.tsx`, Toast in `components/toast.tsx`, Realtime in `context/realtime.tsx`. The theme is a pre-paint script (`components/theme-script.tsx`), not a provider.
- `components/ui.tsx` holds the shared primitives (buttons, badges, tables, empty states).
- `proxy.ts` (Next 16 middleware) issues the CSP nonce.

## 4. Data layer

PostgreSQL 18. Money is `BIGINT amount_minor` + `CHAR(3) currency` (ISO 4217), with a currency exponent table in code. No float types anywhere in migrations except geo coarse coordinates, which are `NUMERIC(4,1)`/`NUMERIC(5,1)` (not float either).

Key tables, each tenant-owned unless noted. Full columns are in SPEC §2.
- Identity and tenancy: `merchants` (tenant root), `users` (global identity), `memberships` (user↔merchant + role), `sessions`, `invitations`, `password_resets`, `api_keys`.
- Payments: `customers` (PII encrypted), `payment_methods` (token + brand/last4/exp only), `processor_accounts`, `payment_intents`, `payment_attempts`, `refunds`, `disputes`.
- Ledger: `ledger_accounts`, `journal_entries`, `ledger_entries`.
- Idempotency and webhooks: `idempotency_keys` (UNIQUE merchant_id+mode+key), `webhook_events` (inbound; UNIQUE source+event_id), `webhook_endpoints`, `pending_webhooks` (outbox), `pending_emails` (outbox).
- Evidence and ops: `audit_events` (partitioned, hash chain), `scheduled_runs` (claim table, UNIQUE job_name+run_key), `risk_signals`, `merchant_assets`.
- Solid Queue's own tables.

DB-enforced invariants:
- `ledger_entries` / `journal_entries` / `audit_events`: a `BEFORE UPDATE OR DELETE` trigger raises.
- Journal balance: a `DEFERRABLE INITIALLY DEFERRED` constraint trigger checks Σdebit = Σcredit per journal_entry_id and currency.
- Status CHECK constraints on every state column.
- `amount_minor > 0` CHECK.
- `refunded_amount_minor <= amount_captured_minor` CHECK.
- Lat/lng bounds CHECK.

Encryption at rest: Active Record Encryption (AES-256-GCM, non-deterministic) on customer email/name/phone/billing address, user name, and webhook endpoint secret. `email_digest` (HMAC-SHA256 with a separate key) serves as the lookup blind index. User login email uses **deterministic** encryption because unique lookup is required (ADR-013). Keys come from `AR_ENCRYPTION_*` env vars. Volume encryption is delegated to the host.

## 5. Payment flow (deterministic)

`POST /api/v1/payment_intents` (Idempotency-Key required):
1. rack-attack throttle.
2. Authenticate the key; resolve merchant + mode.
3. `Idempotency.claim!(merchant, mode, key, fingerprint)`. If the key exists with the same fingerprint and a completed response, the stored response is replayed (`Idempotent-Replayed: true`). With a different fingerprint → 422 `IDEMPOTENCY_KEY_REUSED`. In flight → 409 `IDEMPOTENCY_IN_PROGRESS`.
4. One transaction, intent row-locked:
   - create the intent (`requires_confirmation` when a payment method is given, else `requires_payment_method`);
   - if `confirm: true`, run `RiskCheck`;
   - call the processor via the adapter. The call sits outside the DB lock but inside the idempotency claim; the processor itself receives `idempotency_key = "rails_<intent_id>_<attempt>"`;
   - transition via `StateMachine.transition!(intent, to:, event:)`, which rejects illegal edges;
   - post ledger entries on capture;
   - insert `pending_webhooks` rows (HMAC computed) and the audit event;
   - commit.
5. Store the response on the idempotency row.
6. Broadcast on Action Cable after commit (`after_commit`).

The processor webhook is the source of truth. `payment_intent.succeeded` / `charge.refunded` / `charge.dispute.*` from Stripe reconcile the state (an idempotent transition: the same target state is a no-op).

State machine (`payment_intents.status`):
- `requires_payment_method → requires_confirmation → processing → (requires_capture → succeeded) | succeeded | failed`
- `canceled` is reachable from `requires_payment_method | requires_confirmation | requires_capture`
- terminal states: `succeeded`, `failed`, `canceled`
- refund status is derived from `refunded_amount_minor`

## 6. Scheduled work (has_scheduled_work)

| Trigger | Schedule | Job | Effect |
|---|---|---|---|
| T1 outbound webhook delivery | enqueued on insert + every 1 min sweep | `DeliverPendingWebhooksJob` / `run_for_merchant` | rows `pending→delivered | retrying | dlq` |
| T2 email delivery | enqueued + every 1 min sweep | `DeliverPendingEmailsJob` | rows `pending→sent | retrying | dlq` |
| T3 ledger integrity verification | hourly at :05 (UTC) | `LedgerIntegrityJob` | `scheduled_runs` row with `checked/imbalanced` counts; flags an imbalance as a dlq-severity audit event |
| T4 stale intent expiry | every 15 min | `ExpireStaleIntentsJob` | `requires_payment_method`/`requires_confirmation` older than 24h → `canceled` (reason `expired`) |
| T5 retention sweep | daily 03:00 UTC | `RetentionSweepJob` | deletes `idempotency_keys` >30d, `webhook_events` payloads >30d (keeps id rows ≥72h minimum), expired sessions/resets |
| T6 HIBP re-screen | every 30 min | `RescreenPasswordsJob` | users flagged `breach_check_pending` are re-checked |

- Every job has a per-tenant entry point `run_for_merchant(merchant_id, now:)`.
- Claims are idempotent: `UPDATE … SET claimed_at = now WHERE id = ? AND claimed_at IS NULL` on outbox rows, and an `INSERT … ON CONFLICT DO NOTHING` on `scheduled_runs (job_name, run_key)`.
- The clock is injected (`now:` keyword, default `Time.current`). The time zone is UTC, named in `config.time_zone = "UTC"`.
- Every run records counts in `scheduled_runs`.
- On startup the supervisor resumes unclaimed work. On SIGTERM, Solid Queue finishes in-flight jobs within the shutdown timeout and releases claims.
- Non-prod trigger: `POST /api/v1/ops/scheduled_runs` (platform operator, refuses in production) for the smoke test.

## 7. Security architecture (threat model in RESEARCH §5)

**Authentication.**
- The console uses an encrypted session cookie `__Host-rails_session` (Secure, HttpOnly, SameSite=Lax, Path=/) backed by `sessions` rows for revocation. Idle timeout 30 min; absolute timeout 12 h.
- Merchant servers use `Authorization: Bearer sk_{test|live}_<32 random>`. Only SHA-256 is stored, compared constant-time by digest lookup.
- Publishable keys `pk_` are allowed only on tokenization-safe endpoints (none in v1). ADR.

**CSRF.** Cookie-authenticated unsafe requests must carry `X-CSRF-Token` matching the `rails_csrf` cookie (double-submit, HMAC-bound to the session) AND an `Origin` in `ALLOWED_ORIGINS`. API-key requests are exempt (no ambient credential).

**Authorization.**
- Role matrix: owner > admin > developer > analyst > viewer.
- Object-level checks: every resource is looked up through `current_merchant`, so another tenant's id → 404.
- Same-tenant cross-user: personal resources (own sessions, own pending invitations) are scoped to `current_user`.

**Rate limiting.** rack-attack runs before authentication and before the bcrypt compare:
- login 5/min per email + 20/min per IP;
- signup 5/h per IP;
- reset 3/h per email;
- API 100/s per key;
- money POSTs 25/s per key.
`RATE_LIMIT_SCALE` (a multiplier) is honoured only when `RAILS_ENV != production` and `ALLOW_RATE_LIMIT_SCALE=true`.

**CSP.**
- web: Profile N (nonce + `strict-dynamic`), `frame-ancestors 'none'`.
- api: `default-src 'none'; frame-ancestors 'none'`.
- Both send nosniff, Referrer-Policy strict-origin-when-cross-origin, Permissions-Policy, and COOP same-origin.

**Other controls.**
- PAN guard: any request param whose digits pass Luhn with 13–19 length under keys like `number|card|pan` → 422 `PAN_NOT_ACCEPTED`, and the value is never logged.
- SSRF guard: https only in every environment (ADR-050), resolve DNS, reject private/loopback/link-local/CGNAT/metadata/multicast ranges, connect to the resolved IP with SNI/Host preserved, no redirects, 5s timeout.
- WebSocket: `allowed_request_origins` from `ALLOWED_ORIGINS`; connection auth via session cookie; channel authz checks membership + mode; budget 20 messages/10s per connection, then disconnect.

## 8. Dependencies (pinned)

**api Gemfile:**

| Gem | Version |
|---|---|
| rails | ~> 8.1.4 |
| pg | ~> 1.6 |
| puma | ~> 7 (or 8) |
| solid_queue | ~> 1.2 |
| redis | ~> 5.4 |
| rack-attack | ~> 6.8 |
| rack-cors | not used (same origin) |
| bcrypt | ~> 3.1.20 |
| stripe | ~> 15+ (resolved to the latest at bundle time, recorded in Gemfile.lock) |
| lograge | ~> 0.14 |
| ruby-vips | ~> 2.2 |
| image_processing | ~> 1.14 (not required) |
| bootsnap | (default) |
| rspec-rails | dev/test |
| factory_bot_rails | dev/test |
| webmock | dev/test |
| simplecov | dev/test |

Exact versions are resolved by bundler into `Gemfile.lock` and recorded in VERSION.md (ADR-014: the gem constraints follow the bundler resolution on ruby 3.4).

**web package.json:**

| Package | Version |
|---|---|
| next | 16.x |
| react / react-dom | 19.x |
| typescript | ~5.9 |
| tailwindcss + @tailwindcss/postcss | 4.x |
| lucide-react | — |
| recharts | — |
| zod | 4.x |

Dev: eslint 9/10 + typescript-eslint 8 + eslint-plugin-jsx-a11y + eslint-plugin-react-hooks, prettier 3, vitest + @testing-library/react + jsdom, @playwright/test, @axe-core/playwright.

**Ops:** `postgres:18-alpine`, `redis:7-alpine`, `caddy:2.11-alpine`, `axllent/mailpit:v1.31.4`, `node:22-alpine`, `ruby:3.4-slim`.

## 9. Architectural Invariants

Every row is mirrored 1:1 in `invariants.json` (same id). Prose → JSON is the cross-reference. Manual entries: 1 of 40 (cap: min(3, 20%) = 3).

| ID | Rule | Check type |
|---|---|---|
| INV-1 | Money and amounts are never floating point in migrations (BIGINT minor units only) | forbidden-pattern |
| INV-2 | Idempotency keys are UNIQUE per merchant, mode and key | required-unique-constraint `idempotency_keys(merchant_id, mode, key)` |
| INV-3 | Inbound webhook events are deduplicated by (source, event_id) | required-unique-constraint `webhook_events(source, event_id)` |
| INV-4 | Scheduled-run claims are UNIQUE per (job_name, run_key) | required-unique-constraint `scheduled_runs(job_name, run_key)` |
| INV-5 | Outbound webhook dispatcher runs the SSRF guard before any Net::HTTP call | boundary-order in `api/app/services/outbound_webhooks/dispatcher.rb` |
| INV-6 | Stripe webhook signature is verified before the event is persisted | boundary-order in `api/app/controllers/api/v1/webhooks/stripe_controller.rb` |
| INV-7 | Health route exists | required-file `api/app/controllers/api/health_controller.rb` |
| INV-8 | .env.example documents every env var | required-file `.env.example` |
| INV-9 | Env schema module validates configuration at boot | required-file `api/lib/env_schema.rb` |
| INV-10 | Controllers and channels never query tenant-owned models unscoped (must go through current_merchant) | forbidden-pattern |
| INV-11 | No PAN/CVV storage columns exist | forbidden-pattern |
| INV-12 | dangerouslySetInnerHTML is only used by the JSON-LD component | forbidden-pattern |
| INV-13 | Clients never double-prefix the API path | forbidden-pattern |
| INV-14 | Money services never convert amounts to Float | forbidden-pattern |
| INV-15 | Payment state requires_confirmation is written by production code | required-pattern in `api/app/**` |
| INV-16 | Payment state processing is written by production code | required-pattern in `api/app/**` |
| INV-17 | Payment state requires_capture is written by production code | required-pattern in `api/app/**` |
| INV-18 | Payment state succeeded is written by production code | required-pattern in `api/app/**` |
| INV-19 | Payment state failed is written by production code | required-pattern in `api/app/**` |
| INV-20 | Payment state canceled is written by production code | required-pattern in `api/app/**` |
| INV-21 | Payment state requires_payment_method is written by production code | required-pattern in `api/app/**` |
| INV-22 | Refund states pending/succeeded/failed are written by the refund services | required-pattern in `api/app/services/refunds/**` |
| INV-23 | Dispute state under_review is written by production code | required-pattern in `api/app/services/disputes/**` |
| INV-24 | Dispute states won and lost are written by production code | required-pattern in `api/app/services/disputes/**` |
| INV-25 | Dispute state needs_response is written by production code | required-pattern in `api/app/services/disputes/**` |
| INV-26 | Outbound webhook outbox writes delivered | required-pattern in `api/app/services/outbound_webhooks/**` |
| INV-27 | Outbound webhook outbox writes retrying | required-pattern in `api/app/services/outbound_webhooks/**` |
| INV-28 | Outbound webhook outbox writes dlq after 5 attempts | required-pattern in `api/app/services/outbound_webhooks/**` |
| INV-29 | Email outbox writes sent, retrying and dlq | required-pattern in `api/app/services/emails/**` |
| INV-30 | Ledger rows are immutable at the database level | required-pattern in `api/db/migrations/**` |
| INV-31 | Journal entries must balance (deferred constraint trigger) | required-pattern in `api/db/migrations/**` |
| INV-32 | Rate limiting middleware is configured | required-pattern in `api/config/initializers/rack_attack.rb` |
| INV-33 | Action Cable origin allow-list is configured | required-pattern in `api/config/**` |
| INV-34 | PII columns use Active Record Encryption | required-pattern in `api/app/models/**` |
| INV-35 | Web Delivery Baseline report passes and is fresh | web-baseline |
| INV-36 | Web Delivery Baseline runner is pinned | required-file `scripts/web-baseline.mjs` |
| INV-37 | Every SPEC UI screen, endpoint, control and workflow is covered by the UI and e2e tests | ui-coverage |
| INV-38 | Governance ledger is append-only and every governed commit has a decision | governance |
| INV-39 | Release version agrees across the root manifest, web, compose and the API | version-agreement |
| INV-40 | Every scheduled trigger T1-T6 (webhook delivery, email delivery, ledger integrity, stale-intent expiry, retention sweep, HIBP re-screen) has a spec asserting its business effect | manual |

Migration files live under `api/db/migrations/` (Rails `migrations_paths`) so the pinned runner's `**/migrations/**` glob inspects them; UNIQUE constraints are declared as SQL (`CREATE UNIQUE INDEX`) inside the migrations (ADR-015).

## 10. Key decisions & alternatives (comprehensive)

| Decision | Chosen | Alternatives considered | Why |
|---|---|---|---|
| Queue | Solid Queue (Postgres) | Sidekiq + Redis; GoodJob | Job enqueue is transactional with domain writes, so there is no dual-write; GoodJob is equivalent but less Rails-default |
| Session model | Encrypted cookie + server row | JWT in localStorage; Devise | Revocable, no XSS-readable token; a smaller dependency surface than Devise |
| Ledger enforcement | DB triggers + app service | App-only validation; separate ledger service (TigerBeetle) | DB is the last line of defence; TigerBeetle adds operational burden at v1 |
| Realtime | Action Cable + Redis | SSE from Next; Pusher | Native to Rails, self-hosted, scales horizontally with Redis |
| Multi-tenancy | Shared schema + merchant_id scoping | Schema-per-tenant; RLS-only | Simplest operationally at 50k merchants; INV-10 + request specs enforce it |
| Processor abstraction | Adapter interface + registry | Direct Stripe calls | Processor independence is the product |
| Read scaling | Streaming replica + role switch | Citus; caching only | Native Postgres, no extension lock-in |
| Frontend data | Client fetch + React Context | Server components with cookie forwarding | Constraint (React Context); a single origin keeps cookies simple |

## 11. Implementation notes & risks

- Stripe is in contract mode until `STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET` are real (ADR-008). `StripeAdapter` is unit-tested with WebMock against documented request shapes.
- The p95 targets (<250ms platform ops, <50ms read endpoints) are measured in the Stage 4 load test. The 99.99% availability target is delegated to production topology (multi-AZ, ≥3 api replicas).
- The local deploy uses a self-signed CA. The Deploy URL needs a one-time `caddy trust` (QUICKSTART).
- Risks R1–R11 are in RESEARCH §5.
