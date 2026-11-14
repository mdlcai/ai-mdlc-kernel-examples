# DECISIONS.md — RΛILS

Architecture Decision Records for project `cd32b70e-34d4-468e-8d26-6afe7aab6492`.
Precedence used under `review_gates: auto`: outer prompt > kernel; design template > RESEARCH > archetype; RESEARCH Non-Goals / PRIMARY MVP WORKFLOWS for scope.

---

## Design Template tokens (Step 2b — copied verbatim, binding)

Source: `DESIGN-TEMPLATE.html` (uploaded `rails_1.html`, 416,725 bytes), `:root` block copied exactly:

```css
:root{
  --bg:#050711;--bg2:#0A1020;--surf:#10192D;--surf2:#141F38;--line:#1C2742;--line2:#2A3A62;
  --cyan:#00E5FF;--violet:#8257FF;--blue:#287BFF;--green:#00E6A7;--amber:#FFB547;--coral:#FF5470;
  --text:#F4F8FF;--muted:#8D9AB5;--dim:#6B7894;
  --c-stripe:#5B8CFF;--c-fiserv:#D9772E;--c-worldpay:#A35BD6;--c-ach:#1797A8;--c-btc:#A08A22;
  --f-head:'Space Grotesk',system-ui,sans-serif;--f-ui:'Inter',system-ui,-apple-system,'Segoe UI',sans-serif;--f-mono:'JetBrains Mono',ui-monospace,SFMono-Regular,Menlo,monospace;
  --r:12px;--maxw:1240px;
}
```

Component patterns carried over from the template: `.panel` (gradient rgba(20,31,56,.78)→rgba(16,25,45,.78), 1px `--line`, radius `--r`, 8px backdrop blur); `.btn` (44px min-height, radius 10px, `--line2` border), `.btn.pri` (violet, glow, hover `#9170FF`), `.btn.sm` (36px); `.eyebrow` (mono 12px, .2em tracking, cyan); 2px cyan focus-visible outline with 2px offset; `.skip` link (violet); fixed 68px blurred header; 44px burger below 1080px; inputs on `--bg2` with `--line2` border, radius 8px, 44px tall; hero h1 gradient `#B6A2FF→#8257FF→#287BFF→#00E5FF`.

Ported surfaces: the landing page (`/`) ports the template's sections in order — hero `#top`, `#network`, `#routing` (routing simulator), `#command`, `#fraud`, `#api`, `#security`, `#kernel`, `#ecosystem`, `#access`. (Reorganized for agentic commerce by ADR-057: `#kernel` and `#ecosystem` removed, `#mandates` and `#flow` added.) The authenticated console ports the template's dashboard demo cards (KPI tiles, rail mix, live transaction feed) into the saas app shell.

Light mode: the template is dark-only. A light palette is derived per RESEARCH "Color System — Light Mode" (same accent hues, `#F6F8FC` background, `#0B1222` text) so the theme toggle can ship both; dark stays the default.

---

## ADR-001 — Archetype: saas
- **Context:** RESEARCH Design Language names the product a merchant/enterprise console with a marketing front door.
- **Decision:** Use the `saas` archetype for the authenticated app shell (sidebar that becomes a drawer below md, composed dashboard, designed tables and forms, full dark mode). The template drives the public landing page.
- **Consequence:** JS budget is 400KB. CSP Profile N applies.

## ADR-002 — Typography: Space Grotesk display + Inter UI + JetBrains Mono
- **Decision:** Keep the template's three-face system. Inter is never the sole voice: Space Grotesk carries every heading and JetBrains Mono carries ids, amounts and code (DESIGN anti-slop rule satisfied).

## ADR-003 — TypeScript pinned to ~5.9
- **Context:** `typescript-eslint@8.71.2` declares the peer `typescript >=4.8.4 <6.1.0`. npm `latest` is 7.0.2, and 6.0.3 also exists.
- **Decision:** Pin `typescript@~5.9` (the stable line supported by both Next 16 and typescript-eslint), so the `--max-warnings 0` type-checked lint can run without peer overrides.

## ADR-004 — Host port overrides (never stop foreign containers)
- **Context:** Host ports 80/443 (another stack's edge), 127.0.0.1:5432, 6379, 9090, 13001, 8280, 8643, 3390 and 9390 are taken by unrelated containers.
- **Decision:** Caddy publishes `${HTTPS_PORT:-9443}:443` and `${HTTP_PORT:-9080}:80`. Postgres, Redis, the API and the web tier publish no host ports and are reachable only on the compose network. Foreign processes are never stopped.

## ADR-005 — Password policy (PCI-DSS 4.0.1 8.3.6 vs NIST SP 800-63B-4)
- **Context:** PCI 8.3.6 requires a minimum of 12 characters ("alphabetic and numeric"). NIST 800-63B-4 says a verifier SHALL NOT impose composition rules, and sets a 15-character minimum when the password is the only authenticator.
- **Decision:** Minimum 15 characters (meets both), maximum ≥64 (we accept 128), no composition rules, HIBP breach screen, no forced rotation, no knowledge-based questions, email-link reset. Because length ≥15 is far stronger than 12 alphanumeric characters, the PCI composition clause is satisfied by equivalent strength. This is recorded in COMPLIANCE.md as a compensating rationale.

## ADR-006 — HIBP availability: fail open with a flag
- **Decision:** If the HIBP range API is unreachable (2s timeout), registration and password change proceed, but the user is flagged `breach_check_pending` and re-screened by a scheduled job. A hard outage of a third-party service must not block onboarding. The decision is audit-logged.

## ADR-007 — Ruby 3.4 / Rails 8.1
- **Decision:** `ruby:3.4-slim` with Rails 8.1.x (security support to Oct 2027). Ruby 4.0 is newer, but gem compatibility across stripe, rack-attack and solid_queue is proven on 3.4.

## ADR-008 — Stripe in sandbox/contract mode when keys are placeholders
- **Context:** Stripe is a hard dependency, but no live/test keys are provisioned in this environment.
- **Decision:** `StripeAdapter` sits behind the `ProcessorAdapter` interface. With `STRIPE_SECRET_KEY` unset or placeholder, the platform runs the built-in `SandboxAdapter` (deterministic test-card outcomes) and Stripe-specific endpoints return `SERVICE_UNAVAILABLE`. The smoke test records `[contract]` entries and VERSION carries `-contract-verified`.

## ADR-009 — Step 3b Pre-Build Review (findings applied)
1. Routing: RESEARCH defers multi-processor routing, so v1 ships a merchant-selected default processor plus test-mode failover to Sandbox. Not built: the "intelligent routing" ML.
2. Fraud: v1 ships basic rule checks (amount ceiling, velocity, billing-vs-IP country mismatch). Advanced fraud is deferred.
3. Reconciliation: hourly ledger-integrity verification only. Settlement-file reconciliation is deferred.
4. has_geo exists only as coarse risk-signal coordinates (NUMERIC(4,1)/(5,1), bounds CHECK). There is no map provider, so the provider rate-limit item is N/A.
5. has_image_uploads covers only the merchant logo upload.
6. Migrations go in `api/db/migrations` with SQL UNIQUE indexes so the pinned runner can see them (ADR-015).
7. No schema dump. Triggers and partitions are migration-defined, and tests run `db:migrate` (avoids pg_dump 18 client dependency).
8. The landing-page routing simulator from the template is kept as illustrative demo content, labelled `demo`.

## ADR-010 — Default build-strategy values
prompt_mode is blank, so it defaults to `direct`. confidence_level is blank, but `review_gates: auto` governs, so the Stage 1/2 gates self-verify.

## ADR-011 — Rate limiting enforced at the app tier (Redis-shared), edge enforcement delegated
Caddy has no built-in rate limiter. rack-attack with a Redis store gives a shared, multi-replica limit. Production edge/WAF limits are delegated.

## ADR-012 — Header buffers
Caddy's defaults are generous (no action needed). Next runs with `--max-http-header-size=32768`. Puma's default header limits accept the 16KB cookie probe; this is verified in the smoke test.

## ADR-013 — Login email is deterministically encrypted
A unique lookup on login requires deterministic encryption. All other PII is non-deterministic.

## ADR-014 — Gem versions follow bundler resolution on Ruby 3.4
The exact versions are locked in `api/Gemfile.lock` and listed in VERSION.md at Stage 4.

## ADR-015 — Rails migrations path `db/migrations`
`migrations_paths: db/migrations` in database.yml. UNIQUE constraints are written as `CREATE UNIQUE INDEX` SQL.

## ADR-016 — 99.99% availability delegated
Availability cannot be proven by a single-host compose deploy. Stateless tiers, health checks and graceful shutdown are built; multi-AZ is the production topology's responsibility.

## Stage 1 gate
`review_gates: auto` matched row 3 → self-verified (Build Input Reconciliation 34/34, §9 ↔ invariants.json 40/40) and proceeded.
Invariant runner pinned via `get_file reference/invariant-lint.mjs` (REST returned 401: no `MDLC_API_KEY` in the environment). `--self-test`: OK.
`--prove` closing line: `40 total / 39 machine (39 proven, 0 unproven) / 1 manual (skipped -- Reviewer Gate item 8)`

## ADR-017 — PCI 8.2.8 idle timeout: console out of CDE scope (scope expansion (comprehensive))
The console never displays or accepts PAN (tokens only, PAN guard rejects card data), so it is not a CDE admin interface. A 30-minute idle timeout applies; this is recorded in COMPLIANCE.md as a compensating rationale.

## ADR-018 — Scope expansion (comprehensive)
Additions beyond RESEARCH PRIMARY MVP WORKFLOWS, each derived from RESEARCH personas or compliance needs:
- F-18 platform operator console (W10);
- F-14 analytics + CSV export;
- F-15 audit verification + evidence export (W9);
- F-04 team management (W8);
- F-02 own-session management (W11).
Non-Goals stay deferred.

## Stage 2 gate
`review_gates: auto` → self-verified. SPEC §1–§10 are present. The pinned runner parses §6/§5/§3 into 30 screens, 11 workflows and 71 non-internal endpoints. Every §6 Binds entry exists in §3, and every W step maps to a §6 row.

## Multi-Agent Plan Gate (review_gates: auto → row 3, dispatch without halting)
- 8 waves, 19 features. CRITICAL/FOUNDATIONAL features (F-01, F-02, F-04, F-05, F-07–F-12, F-18) run sequentially in the orchestrator.
- Sub-agent waves: wave 4 (F-06, F-15, F-16, F-19) and wave 7 (F-13, F-14, F-17). Each is file-disjoint by manifest `files_write` (per-feature `config/routes/<f>.rb`, pre-declared nav, per-feature migration timestamp blocks).
- Waves 2 and 8 have a single feature, so they run in the orchestrator.
- Estimate ~14h; t_max 2026-10-19T09:43Z. Governance is pinned (govern v1.8, policy 1.4) before the plan commit.
- HUMAN_APPROVAL outcomes (payments/dependencies/infrastructure) are queued for the ship banner and never answered by the agent.

## ADR-019 — Audit hash column is `row_hash` (SPEC §4 `audit_events.hash` overridden)
- **Context:** `hash` collides with `Object#hash`, so Active Record raises `DangerousAttributeError`.
- **Decision:** The column is `row_hash`. The API keeps the SPEC field name: `AuditEvent#as_json` emits `hash:`, and chain verification is unchanged.
- **Consequence:** Only the DB column name differs from SPEC §4. API consumers see no change.

## ADR-020 — Health severity and container roles
- **Decision:**
  - `/api/health` returns 503 when any of db, db_replica, redis or queue fails.
  - The queue probe checks that the Solid Queue tables are readable; worker liveness is the worker container's own healthcheck (`pgrep` on the supervisor), not the api's.
  - Migrations run only in the `PROCESS_ROLE=server` container.
- **Why:** A probe that required worker heartbeats would deadlock compose: the worker depends on a healthy api, which already migrated.

## ADR-021 — Ruby style is rubocop-rails-omakase
`api/.rubocop.yml` inherits `rubocop-rails-omakase`. CI runs `rubocop`, `brakeman` and `bundle-audit`.

## ADR-022 — `pending_emails.params` stored as encrypted text (SPEC §2 `params JSONB` overridden)
- **Context:** Template params carry one-time links (reset and invite tokens).
- **Decision:** `params` is a `text` column, JSON-serialized and then Active Record–encrypted. `to_address` is encrypted too. Params are cleared to `{}` once a message is `sent`.
- **Trade-off:** The database cannot query params. Nothing in the product needs to.
- **Retry ladder:** 4 attempts in total, at +1m, +10m and +1h, then `dlq`, which records the platform audit event `email.dead_lettered`. A `sending` claim older than 10 minutes is reclaimable.

## ADR-023 — Signup provisioning steps are named constants resolved at call time
- **Context:** W1.2 provisions ledger accounts (F-09), sandbox processor accounts (F-07) and a test API key (F-05) in the signup transaction. Those features build after F-02.
- **Decision:** `Signup::PROVISIONING_STEPS` names `Ledger::Provision`, `Processors::Provision` and `ApiKeys::Provision`. Each lives in its owning feature's service directory and is resolved with `safe_constantize`. Each step runs inside the signup transaction and returns a hash; only `test_api_key` is surfaced in the 201 body, once.
- **Consequence:** Until a step's feature ships, signup simply skips that step. The W1 e2e terminal step (W1.4, the key listed on /developers/api-keys) passes from F-05 onwards. Any step that raises rolls the whole signup back (spec'd).

## ADR-024 — Signup reveals a taken email; login and reset do not
- **Context:** SEC-9 requires no account enumeration.
- **Decision:** Login and reset are fully generic. Login returns one 401 text for unknown email, wrong password or a locked account, and unknown emails still pay for a bcrypt compare. Reset always returns 202 with the same body. Signup returns 422 `TAKEN` on `email`, because an account cannot be created silently.
- **Mitigation:** Signup is rate-limited (5/h per IP) and Origin-checked.
- **Also:** Unauthenticated unsafe auth endpoints (signup, login, reset) require an allowed `Origin` (login-CSRF), and login rotates the session by revoking any token the browser already carried.

## ADR-025 — Invitations are bound to their email; accept resolves the joining user from it
- **Context:** W8.2 accepts "new or existing user". SPEC §3 gives `POST /v1/invitations/:token/accept` as P or S.
- **Decision:** The invitation's email decides who joins, not the caller.
  - If the caller's session email matches, they join with no credentials.
  - Otherwise, if an account exists for that email, its password is required. This reuses `Passwords::Login`, so the lockout applies.
  - Otherwise a new account is created with a name and a password that passes the policy.
  - A signed-in user with a different email cannot take the invitation.
- **Token lookup:** `GET /v1/invitations/:token` returns the SPEC fields plus `inviter_name`, `expires_at` and `requires` (`none`/`password`/`name_and_password`) so the form knows what to ask for. Revealing whether the invited address has an account is acceptable, because only the holder of the emailed 256-bit token sees it.
- **Lifecycle:**
  - The token is single use: it is claimed by a conditional UPDATE inside the membership transaction.
  - It expires after 7 days.
  - Re-inviting revokes the earlier open invitation (partial UNIQUE on open `(merchant_id, email_digest)`).
- **Rate limits and Origin:** Accept is limited to 10/h per IP and requires an allowed Origin.
- **Last-owner guard:** It runs under a merchant row lock. It returns 422 `VALIDATION_FAILED` with `errors[0].code = LAST_OWNER`, which stays within the §7 code table.
- **Admins and owners:** Admins can neither touch owners nor grant or revoke the owner role or owner invitations.

## ADR-026 — API key format, mode binding and console-only management (F-05)
- **Status:** Accepted (build, autonomous; SPEC §2 api_keys, SEC-4).
- **Format:** `sk_<mode>_` + 32 random bytes in base58, fixed at 44 characters. Only the SHA-256 digest (UNIQUE) and a 12-character display prefix are stored. A CHECK ties the prefix to the key's mode. The secret appears once: in the `POST /v1/api_keys` response, or in the signup response for the provisioned test key (ADR-023).
- **Mode binding:** A key's mode overrides the `Rails-Mode` header. The console lists, creates and revokes only keys in its current mode, so a key in the other mode returns 404.
- **Management:** Key management needs a session (owner/admin/developer). A request authenticated with an API key gets 403, so a leaked key can neither mint nor revoke keys.
- **Last used:** `last_used_at` is written at most once a minute, so the hot authentication path does not write on every request.
- **Live-stack e2e:** The journeys sign up several merchants per run, which exceeds signup's real 5/h/IP limit. Production has no override (SEC-20), so the Playwright global setup clears `rack-attack:*` keys in the local compose Redis, and only when BASE_URL is localhost. The limits themselves are unchanged.

## ADR-027 — Wave 4 dispatch (Plan Gate, review_gates: auto)
- **Status:** Accepted (build, autonomous).
- **Dispatch:** F-06, F-15, F-16 and F-19 have disjoint `files_write` sets, so each goes to its own sub-agent. Each agent is briefed only with its manifest and `BUILD-STATE.md`.
- **Agent limits:** Agents write code and tests. They do not commit, rebuild the live compose stack, or edit shared ledgers (DECISIONS, REPORT, CHANGELOG, COMPLIANCE, BUILD-PLAN, BUILD-STATE, SPEC). Instead, each returns its ADR text and manifest amendments, and the orchestrator applies them, commits each feature, and runs the wave-end checkpoint.
- **Test isolation:** `scripts/test-api.sh` takes `TEST_DB=<name>`, so parallel agents do not drop each other's test database.

## ADR-028 — Customer redaction, card tokens and the PAN guard (F-06)
- **Status:** Accepted (build, autonomous).
- **Redaction:** `DELETE /v1/customers/:id` is a session-only owner/admin action (SPEC §4.2 "redact customer"). It clears email, email digest, name, phone and address, keeps the row and `billing_country`, detaches every card, and is idempotent. A DB CHECK requires every PII column to be NULL once `redacted_at` is set. PATCH or attach on a redacted customer → 409 `INVALID_STATE_TRANSITION`.
- **Cards:** detaching wipes `processor_token`, so a detached card can never be charged; brand, last4 and expiry are kept for history. A composite FK `(customer_id, merchant_id, mode)` stops attaching across tenants or modes. The fingerprint is an HMAC of merchant, brand, last4 and expiry, keyed with `BLIND_INDEX_KEY`.
- **Tokens:** sandbox `pm_card_*` tokens (SPEC §5 F-07 names) are accepted in test mode only. Until F-07 exists, any other `pm_…` token returns 503 `SERVICE_UNAVAILABLE`; F-07 moves token resolution into the processor registry.
- **PAN guard (SEC-10):** `Security::PanGuard.check!(params, also:)` rejects Luhn-valid 13–19 digit values in card-like keys at any depth (422 `PAN_NOT_ACCEPTED`, field path only, value never logged). Customers also scan `name` and `billing_address`. Phone is capped at 15 digits (E.164) instead of being scanned, so long international numbers don't false-positive. F-08 reuses the guard.
- **Logs (SEC-14):** `LogRedaction.path` now keeps query parameter names but replaces their values, so `GET /v1/customers?email=…` no longer writes an email into the request log.

## ADR-029 — Merchant logo storage, limits and delivery (F-16)
- **Status:** Accepted (build, autonomous).
- **Storage:** logos live in `merchant_assets.data` (BYTEA, SPEC §2) as two PNG rows, 64 and 256 px, under UNIQUE (merchant_id, kind, variant). `merchants.logo_asset_id` points at the 256 row (FK, ON DELETE SET NULL). A replace runs in one transaction under a merchant row lock. The original upload is never kept.
- **Hardening (SEC-16):** the type is decided by magic bytes only; only the matching libvips loader runs (fail_on: error). HEIC/AVIF → 415 `UNSUPPORTED_MEDIA_TYPE` with error code `HEIC_NOT_SUPPORTED`; >1 MB → 413 before decoding; images over 8192 px per side or 25 MP are rejected from the header (decompression-bomb guard). Output is auto-rotated, converted to sRGB and saved as PNG with all metadata stripped.
- **Rate limit:** 5 uploads per hour per merchant, counted in the controller through `Rack::Attack.cache.count` (same Redis store, honours `RATE_LIMIT_SCALE`). Middleware can't resolve the merchant, so this isn't a rack_attack.rb rule. Every authorised attempt counts, including rejected files.
- **Delivery:** `GET /v1/merchant/logo/:variant` returns image/png with `nosniff`, `CSP: default-src 'none'; sandbox`, `CORP: same-origin`, `Cache-Control: private, max-age=3600` and an ETag (304 on match). URLs carry `?v=<sha256[0,12]>` for cache-busting.
- **Descriptor:** `statement_descriptor` may not contain `< > \ ' " *` or control characters and must contain a letter, because card networks reject it otherwise.

## ADR-030 — Evidence exports are synchronous, immutable and self-verifying (F-15)
- **Status:** Accepted (build, autonomous).
- **No job:** `POST /v1/governance/evidence_exports` builds the bundle in the request, so the row is born `ready`. Ranges are capped at 366 days and 10,000 events (422 `OUT_OF_RANGE` beyond). Because the build is synchronous, rack-attack limits each session to 10 exports per minute (`evidence-export/session`).
- **Bundle:** the range's audit rows (created_at to the microsecond), chain head, the merchant chain key, hash algorithm, four checks (chain valid, immutability trigger enabled, range contiguous, overall) and the app version and commit. `bundle_sha256` is SHA-256 over recursively key-sorted JSON; the download adds a `manifest` carrying that digest, so the bundle can be re-verified offline. A trigger on `evidence_exports` refuses UPDATE.
- **Verification:** `Governance::ChainVerifier` walks the chain 1,000 rows at a time (plucked columns, by id) and reuses `Audit.digest`, so memory stays flat as the trail grows. Tamper tests (edit, delete, forged hash) each report the first broken event id.
- **PII:** bundles hold public ids and audit data already filtered by `Audit`. IP addresses stay in rows because they are part of the hashed record (SEC-15, PCI-10); dropping them would break verification.
- **Login events:** `session.created` is recorded with no merchant (F-02), since a user may belong to several merchants, so it appears in the platform trail rather than a merchant's.
- **Filters:** `action` (exact, or a `family.*` prefix), `actor` (a type or a public id), `created_gte`/`created_lte`; `action` is read from `request.query_parameters` because Rails reserves `params[:action]`.

## ADR-031 — Marketing site, docs and SEO (F-19)
- **Status:** Accepted (build, autonomous).
- The landing (`/`) and `/docs` live in the `(marketing)` route group; the root `web/app/page.tsx` moved there. Styles are scoped under `.mk` and reuse the global tokens.
- `/docs` is generated from typed data mirroring SPEC §3 and §7. A Vitest contract test parses SPEC.md and fails if the docs drift from it in either direction.
- `robots.ts` derives its disallow list from `lib/nav` plus the auth routes; `robots.ts` and `sitemap.ts` are force-dynamic so `APP_ORIGIN` is read at runtime. `public/robots.txt` was removed so it can't shadow `robots.ts`. The sitemap lists only `/` and `/docs`.
- JSON-LD is the only inline script and carries the request CSP nonce. Rails outside v1 (Fiserv, Worldpay, ACH, Bitcoin) are labelled "Deferred · not in v1"; the routing simulator is labelled `demo` (ADR-009 item 8).
- The footer carries the structural "Built with MDLC.ai" attribution linking https://mdlc.ai.

## ADR-032 — Outbound webhook signature scheme (fixes the signed string for F-12)
- **Status:** Accepted (build, autonomous).
- SPEC names the `Rails-Signature` header but never defines the signed string. It is fixed as: `Rails-Signature: t=<unix_ts>,v1=hex(HMAC-SHA256(endpoint_secret, "<t>.<raw_body>"))`.
- Receivers compare in constant time and should reject timestamps more than 300 s old. `/docs` documents this, and F-12 must sign exactly this way.

## ADR-033 — Processor routing, failover and Stripe contract mode (F-07)
- **Status:** Accepted (build, autonomous).
- **Accounts:** each merchant has one `processor_accounts` row per processor and mode, provisioned at signup (and lazily for older merchants). Test mode: Sandbox enabled and default, Stripe disabled behind it. Live mode: both disabled, with Stripe first. A partial UNIQUE index allows one default per merchant and mode, and a CHECK requires the default to be enabled. The default can only be replaced, never disabled or cleared.
- **Routing:** `Processors::Router` tries the default first, then the other enabled accounts by ascending priority, skipping processors that cannot hold the token. Only `unavailable` and `error` fail over. A decline is the processor's answer and stops the chain. Each try is an attempt with its own processor idempotency key, `<intent key>_<attempt_no>`, so a replay cannot double-charge. If no account can charge, the call returns SERVICE_UNAVAILABLE.
- **Stripe:** stripe gem 20.0 with `STRIPE_API_VERSION` pinned (default `2026-03-25.dahlia`) and a client built per call. With the key unset or `placeholder`, the adapter is unavailable (`keys_not_configured`) and makes no network call (ADR-008, [contract]). A key works only in its own mode (`key_mode_mismatch`). Mapping: card errors decline; 5xx and timeouts are errors; auth errors are unavailable.
- **Token resolution:** this moved from F-06 into `Processors::Registry.resolve_payment_method!`. Behaviour is unchanged: sandbox tokens work in test mode only, and other tokens need an available Stripe.
- **W3.3/W3.4** need payment intents, so the W3 e2e covers W3.1–W3.2 now, and F-08 extends it with the failover payment and its two attempts.
- **Sessions only:** routing is a console setting. API keys cannot read or change it (SPEC §4 "S").

## ADR-034 — Payment intents, idempotency and risk (F-08)
- **Status:** Accepted (build, autonomous).
- **State machine:** `Payments::StateMachine` holds SPEC §5 F-08's nine edges. Any other edge returns 409 `INVALID_STATE_TRANSITION`, with the current intent in the body. A `processor_*` event that repeats the current state is a no-op, so a replayed processor answer is safe. Every write takes `intent.lock!`.
- **Idempotency (INV-2):** `idempotency_keys` is UNIQUE on (merchant, mode, key). The fingerprint is SHA-256 of the method, the path and canonical JSON of the body. A different body with the same key → 422 `IDEMPOTENCY_KEY_REUSED`. A completed key replays the stored status and body with `Idempotent-Replayed: true`. An in-flight key → 409 `IDEMPOTENCY_IN_PROGRESS`.
  - A claim older than 60 s is taken over. A create that already has a `resource_id` resumes that intent, never a second one. A `processing` intent re-runs routing with the same attempt numbers, so the processor keys are unchanged and nothing can be charged twice.
  - 4xx answers are stored. 5xx answers and crashes release the claim, so the client can retry.
  - API keys must send `Idempotency-Key`. Console sessions may omit it; the console sends one per form submission anyway.
- **Fee:** `fee = min(amount, round_half_up(amount × 2.9%) + 30)`, in integer minor units only (INV-14). It is charged on the captured amount.
- **Failures:** a decline → `failed` with the processor's code. If every processor is unavailable or errors → `failed` with `processor_unavailable`. Both answer 402 `PAYMENT_DECLINED` with the failed intent in the body. A risk block → `failed`/`risk_blocked`, 402 `RISK_BLOCKED`, and no processor is called.
- **Processor override:** `processor` on create restricts routing to that one processor, with no failover. This is for explicit merchant choice and testing.
- **Risk:**
  - Rules and scores: amount above the ceiling (+100), more than 10 intents for the customer in 10 min (+40), billing country ≠ IP country (+30). The score is capped at 100; ≥ 80 blocks.
  - IP country comes from a static table: the RFC 5737 documentation ranges, plus a centroid per supported country. An unknown IP skips the country rule.
  - Coordinates are stored only as coarse NUMERIC(4,1)/(5,1) centroids, never as raw IPs or precise locations (has_geo).
  - A processor's own risk score is recorded (the max is kept) but never reverses a charge that already succeeded.
- **Extra columns** beyond SPEC §3: `payment_method_token` (encrypted, never serialised) for raw-token charges, `processor_override`, and `idempotency_keys.resource_id` for resume.
- **Plug-in hooks:** `Payments::Hooks` calls `Ledger::PostPayment` (F-09), `OutboundWebhooks::Enqueue` (F-12; renamed from `Webhooks::Enqueue`, ADR-038) and `Realtime::Broadcast` (F-13), resolving each with `safe_constantize`. Each one is a no-op until its feature lands. The broadcast runs after commit.
  - The detail view's `refunds`, `disputes` and `ledger_journal_ids` are empty until F-09/F-10 provide them.

## ADR-035 — Ledger posting and enforcement (F-09)
- **Status:** Accepted (build, autonomous).
- **Accounts:** five per merchant, mode and currency (`processor_receivable`, `refunds_clearing` and `dispute_losses` are debit-normal; `merchant_payable` and `platform_fees` are credit-normal). A CHECK pins each code's normal balance. Signup opens the default currency in both modes (`Ledger::Provision`). Other currencies open lazily on their first posting, or on a ledger read, through an idempotent `insert_all` keyed on the unique index.
- **Balances:** `balance_minor` is a signed projection in the account's normal direction. It is updated in the posting transaction under `lock_version`, with up to three retries on `StaleObjectError`. The journal lines stay the source of truth.
- **Capture journal:** Dr processor_receivable for the captured amount; Cr merchant_payable for amount − fee; Cr platform_fees for the fee. A zero line is dropped (the fee can take the whole amount), so the journal keeps ≥ 2 positive lines.
- **Idempotency:** one journal per (source_type, source_id, kind) by UNIQUE index. Re-posting returns the existing journal, so a replayed capture hook is safe.
- **Enforcement (INV-30/31), in depth:**
  - The app rejects unbalanced, non-Integer or single-line journals before writing.
  - `ledger_entries_immutable()` blocks UPDATE/DELETE on both tables; corrections are new `reversal` journals.
  - `ledger_journal_balanced()` is a DEFERRABLE INITIALLY DEFERRED constraint trigger that checks each journal at commit (≥ 2 lines and Σ debits = Σ credits).
  - Composite FKs keep every line in its journal's merchant, mode and currency, on an account with the same merchant, mode and currency.
- **Schema extras beyond SPEC §2:** `ledger_accounts.public_id` (`la_…`), used by the journal `account` filter. Merchant FKs on the ledger tables do not cascade: books outlive any cleanup path.
- **Reads:** finance roles only (owner, admin, analyst), console sessions only. They run on the primary so a balance always matches its journals. Replica and cache reads are reserved for F-14 summaries. `GET /v1/ledger/accounts` takes an optional `mode` override, so finance can compare both modes; the default is the session mode.
- **Payment detail:** `ledger_journal_ids` comes from `Ledger::Journals.ids_for`, covering the intent's own journals plus refund and dispute journals once F-10 provides those models. The console links each one to `/ledger/entries/[id]`.

## ADR-036 — Refunds and disputes (F-10)
- **Status:** Accepted (build, autonomous).
- **Who can refund:** owners and admins in the console. API keys may also `POST /v1/refunds`, as SPEC's "S|K owner/admin" row allows, overriding the developer mapping for this one route. As with other money POSTs, API keys must send an Idempotency-Key; the console always sends one. Evidence submission is console-only (owner/admin). The sandbox is owner/admin/developer, test mode only (403 in live).
- **Refundable amount:** captured − Σ(pending + succeeded refunds), computed with the payment row locked. Omitting `amount_minor` refunds what is left. A refund over the remainder is 422 `OUT_OF_RANGE`. A payment that is not succeeded, has nothing left, or is disputed is 409 `INVALID_STATE_TRANSITION`: a disputed payment's money is already withheld, so refunding it would return it twice.
- **Settlement:** the refund row is created `pending` in the request transaction (so the idempotency claim can point at it). The processor is called after commit with the key `rails_<re_id>`.
  - succeeded → `succeeded`, with a `refund` journal (Dr merchant_payable / Cr processor_receivable).
  - declined → `failed`, with its failure code, which frees the amount.
  - transient → stays `pending`, 503 `SERVICE_UNAVAILABLE` with `retry_after`. The 5xx releases the idempotency key, so a retry with the same key resumes the same refund using the same processor key.
  - Leaving `pending` is a single compare-and-swap (`UPDATE … WHERE status = 'pending'`), so the synchronous response and F-11's webhook cannot settle it twice.
- **Disputes:** one per payment (UNIQUE), for captured − refunded, due 7 days after opening.
  - `needs_response` → (evidence) `under_review` → `won` | `lost`. Evidence is three text fields, each ≤ 5000 chars, PAN-guarded, submitted once. A repeat of the same outcome is a no-op (webhook redelivery).
  - Journals: open = Dr dispute_losses / Cr processor_receivable; won = Dr processor_receivable / Cr dispute_losses; lost = Dr merchant_payable / Cr dispute_losses.
  - `Disputes::Lifecycle.open!/close!` and `Refunds::Settle.succeed!/fail!` are the entry points F-11's processor webhooks will call.
- **Schema extras:** a composite FK `(payment_intent_id, merchant_id, mode, currency)` keeps refunds and disputes in their payment's tenant, mode and currency. It is backed by a new unique index on payment_intents. CHECKs require `succeeded_at`/`failed_at`/`closed_at`/`evidence_submitted_at` for their states. Dispute evidence is a jsonb object capped at 20000 octets.
- **Events:** `refund.created|succeeded|failed`, `dispute.created|updated|closed`. Each writes an audit row, goes to the outbox hook (F-12) in the same transaction, and is broadcast after commit (F-13).

## ADR-037 — Inbound processor webhooks and the event log (F-11)
- **Status:** Accepted (build, autonomous).
- **Verification (SEC-11, INV-6):** `POST /v1/webhooks/stripe` is unauthenticated. It calls `Stripe::Webhook.construct_event` on the raw body with `STRIPE_WEBHOOK_SECRET` and a 300 s tolerance before anything is written. A bad, stale or missing signature, or a body that is not JSON, gets 400 `BAD_REQUEST`. If the secret is unset the endpoint answers 503, so Stripe keeps the events and retries them once it is set. Every verified event gets 200, duplicates and anomalies included, so Stripe stops redelivering.
- **Dedupe and retries (INV-3):** `INSERT … ON CONFLICT (source, event_id) DO NOTHING`, then the event row is locked. A processed row answers `duplicate: true`. A domain refusal (409/422, e.g. no edge from the intent's state) is stored as `error` and acknowledged, because a redelivery would only repeat it. Any other exception rolls the apply back and returns 500: the row stays unprocessed and Stripe's redelivery applies it.
- **Mapping:**
  - `payment_intent.succeeded` → `processor_succeeded`, or `capture` from requires_capture, which covers a capture done in the Stripe dashboard. It posts the capture journal through the hooks.
  - `payment_failed` → failed.
  - `amount_capturable_updated` → requires_capture.
  - `charge.refunded` (embedded refunds), `refund.updated` and `refund.failed` → `Refunds::Settle.succeed!`/`fail!`. The refund is found by processor_ref or by the `rails_idempotency_key` metadata the adapter now sends, so a webhook that lands before the API response still settles it.
  - `charge.dispute.created`/`closed` → `Disputes::Lifecycle.open!` (with Stripe's `evidence_details.due_by`) and `close!`.
  - Replays are no-ops in those entry points. Webhook changes are audited with actor `processor`/`stripe` (new `Current.processor`).
- **Anomalies:** an unknown type, an unmatched processor reference, a livemode/mode mismatch, a failure reported after success, an unsupported dispute outcome, or more than 100 events per minute from one source. Each is written to `webhook_events.anomaly` (several are joined with "; ") and audited as `webhook.anomaly`. Unmatched rows keep `merchant_id` NULL and are not visible through `/v1/events`.
- **PII:** before storage the payload has `billing_details`, `shipping`, `receipt_email`, `email`, `phone`, `address` and `name` replaced with `[redacted]` at any depth. The 30-day payload / 90-day row retention sweep is scheduled by F-17.
- **Sources:** `stripe` is live. `sandbox` is reserved, because the sandbox adapter answers synchronously and sends no webhooks. `rails` rows are written by F-12's outbox and are exempt from the signature CHECK.
- **Event log:** `GET /v1/events` (any role or API key, tenant + mode scoped) filters by `source` and by exact `type` or a `family.*` prefix, with cursor pagination. `GET /v1/events/:id` adds the payload. The console's `/developers/events` lists the events and opens the payload in a dialog. Its e2e (W7.4) runs in F-12's W7 spec: locally the signing secret is unset and sandbox payments send no webhooks, so F-12 is the first feature to put events in a tenant's log.

## ADR-038 — Outbound merchant webhooks (F-12)
- **Status:** Accepted (build, autonomous).
- **Outbox (has_dual_write):** `Payments::Hooks.publish!` calls `OutboundWebhooks::Enqueue` inside the domain transaction. It writes one `webhook_events` row (source `rails`, the payload `{id, object: "event", type, livemode, created, data: {object}}`) and one `pending_webhooks` row per enabled endpoint of that merchant and mode that subscribes to the type (or `*`), so a rollback writes nothing. Each row's idempotency key is `<evt>:<we>` (UNIQUE), and the job is enqueued after commit. The event is logged even with no endpoints, so `/developers/events` shows what was emitted. The plan's constant name `Webhooks::Enqueue` became `OutboundWebhooks::Enqueue` to match the module (ADR-034's hooks line updated).
- **Delivery (T1):** `DeliverPendingWebhooksJob` (ARCHITECTURE §6's name; the plan's `webhook_delivery_job.rb` is amended) runs per row on insert and as an every-minute sweep (`config/recurring.yml`) over due `pending`/`retrying` rows and claims older than 10 minutes. `run_for_merchant` scopes the sweep to one merchant. A claim is a conditional `UPDATE … SET status = 'delivering', attempts = attempts + 1`, so two workers never send the same attempt.
- **SSRF (SEC-12, INV-5):** endpoint URLs pass `Security::SsrfGuard.check_url!` when saved (https only, no credentials, no internal names such as localhost/.local/.internal, no private, loopback, link-local, CGNAT or metadata literals, IPv4-mapped IPv6 included), so a bad URL gets 422. Before every attempt `assert_safe_url!` resolves the host (DNS bounded to 2 s) and refuses the attempt if any answer is non-public, then the connection is pinned to the checked address (`ipaddr`) with SNI and certificate checks on the host name, which closes DNS rebinding. Redirects are not followed (a 3xx is a failed attempt), open/read/write timeouts are 5 s, and a response body over 64 KB fails the attempt.
- **Signature:** `Rails-Signature: t=<unix>,v1=<hex HMAC-SHA256(secret, "<t>.<body>")>` over the canonical JSON body (keys sorted). Every attempt is re-signed with the current time and secret, so retries stay inside a receiver's timestamp tolerance and a rotation takes effect at once. Headers also carry `Idempotency-Key` (the row key), `Rails-Event-Id` and `Rails-Delivery-Id`. Secrets are `whsec_` + 32 random bytes, encrypted at rest, and returned only by create and rotate.
- **Retries (INV-26..28):** 5 attempts in all. Failures 1–4 go to `retrying` with backoff 1m, 5m, 30m, 2h, and the 5th goes to `dlq`, audited as `webhook_delivery.dead_lettered`. SPEC's sixth step (6h) would be a 6th attempt and contradict "dlq after 5 attempts", so it is dropped. A disabled endpoint fails attempts rather than skipping them, so nothing is silently lost. `POST /v1/webhook_deliveries/:id/retry` moves a `retrying` or `dlq` row to `pending` for an immediate attempt (409 otherwise). If that attempt fails it goes straight back to `dlq`, because the attempt count is kept.
- **No single-endpoint read:** SPEC §3 has no `GET /v1/webhook_endpoints/:id`, so the wave 6 checkpoint removed it. The detail screen finds its endpoint in the list (at most 16 per mode, so one page of 100 is complete) and shows a not-found state otherwise.
- **API and roles:** endpoints and deliveries are console-only (`session_only`) for owner/admin/developer (`developer_settings`), scoped to the console mode, with at most 16 endpoints per mode. `test` sends a sample `{id: "<object>_test", object, test: true}` of any listed type to that one endpoint. The actions are named `send_test`/`requeue` because `test` and `retry` clash with Ruby. Every change is audited.
- **Console:** `/developers/webhooks` (list, add with event checkboxes, secret shown once, recent deliveries with a status filter and refresh) and `/developers/webhooks/[id]` (edit, enable/disable, rotate with confirm, delete with confirm, send test event, deliveries with Retry). The W7 e2e targets `https://hooks.invalid/rails`, which never resolves, so the real worker's attempt lands in `retrying` and W7.5's retry runs on a genuine failure.

## ADR-057 — Agentic commerce focus: agent mandates (F-20), landing reorganized
- **Status:** Accepted (build, at the human's request, 2026-11-11). This departs from SPEC (§1 product, §2, §3, §4.2, §5 F-20/W12, §6, §7 and §10 were amended with it) and from the template-port landing order recorded under "Ported surfaces".
- **Context:** the human asked to make agentic authorization and settlement the focal point of RΛILS and to drop the landing features that are far from shipping. The deferred rail cards (Fiserv/Worldpay, ACH/Bitcoin/Lightning, scan to pay), the Ψ kernel section and the ecosystem grid all promised or described things RΛILS does not do.
- **Decision: agent mandates, built on the existing payment lifecycle rather than beside it.**
  - New table `agent_mandates` (migration `20261111000100_agent_mandates`): tenant-scoped like `api_keys`; the secret `ak_<mode>_` + 32 random bytes (base58) is returned once and stored as a SHA-256 digest with a 12-character prefix. Limits are BIGINT minor units in one currency, with CHECKs mirroring the model validations. `payment_intents.agent_mandate_id` cites the mandate through a composite `(id, merchant_id, mode)` FK, so an intent can never point at another tenant's mandate.
  - **Authorization is the intent, settlement is the capture.** Mandates default to `capture_method: manual`. Spend is derived, never stored: open intents reserve their full amount, succeeded ones count `amount_captured_minor`, failed/canceled count zero. A partial capture therefore releases the remainder, and cancel releases all of it.
  - **Caps under a lock.** `Payments::AgentGuard` checks currency, the per-transaction cap and customer binding before anything is written, then `reserve!` takes the mandate row lock inside the create transaction and checks the total. Two concurrent agent requests serialize on the mandate and cannot both fit.
  - **Lock order (extends ADR-045):** mandate row → merchant audit chain → ledger accounts. The mandate is locked only at intent creation, before the audit chain; nothing holding the audit chain or a ledger lock locks a mandate.
  - **403 `MANDATE_LIMIT_EXCEEDED`**, not 402: the payment was never attempted, the agent lacks authority for it. The body carries `limit` (`per_transaction`|`total`), `limit_minor` and, for the total, `spent_minor`.
  - **Confinement:** an `ak_` bearer token reaches only payment intents index/show/create/confirm/capture/cancel and `GET /v1/agent_mandates/current` (an action allowlist in `Authentication`); everything else is 403. Within those, `TenantScoped` narrows intents to the mandate's own, customers to its bound customer, and every other model to none, so another agent's or the merchant's intent is a 404. Agents always need `Idempotency-Key`. A saved payment method is usable only through a mandate bound to its customer.
  - **Audit:** `audit_events.actor_type` gains `agent` (CHECK re-created), `actor_id` is the mandate's public id. Issue and revoke are audited as `agent_mandate.created` / `agent_mandate.revoked`.
  - **Review hardening (invariant-check):** an agent's idempotency fingerprint includes its mandate, so the same key and body from another agent or the merchant is 422 `IDEMPOTENCY_KEY_REUSED`, never a replay of a response it may not see. A mandate bound to a redacted customer cannot create payments. `reserve!` re-checks the mandate's status under the lock and `revoke!` takes the same lock, so a revoke that commits mid-request still stops the payment. Issue and revoke commit with their audit rows in one transaction (mandate row, then audit chain), and revoking twice audits once.
  - **Not done, deliberately:** revoking does not void open authorizations (the merchant captures or cancels them); there is no agent-protocol integration (AP2/ACP) or network agent token; mandates are issued only by a console session (owner/admin/developer), never by an API key or another agent.
- **Decision: landing reorganized.** Sections are now hero (agentic) `#top`, `#mandates`, `#flow` (authorize → settle), `#routing`, `#api`, `#network` (processors and networks, v1 and via-processor nodes only), `#command`, `#fraud`, `#security` (with a one-line `#governance` note) and `#access`. `#kernel` and `#ecosystem` are removed with their CSS; tokens are untouched (ADR-044). The hero and footer drop the Ψ signature line. SEO title and description follow.
- **Supersedes:** ADR-051 (the scan-to-pay card is removed; `docs/proposals/QR-PAYMENTS.md` stays internal and unlinked, and QR/scan-to-pay is listed as a SPEC §1 non-goal).
- **Consequences:** a new console screen `/agents` (W12) and a payments-list `agent_mandate` filter; the `/docs` page gains an Agent mandates section and auth mode `A`. Specs: `api/spec/requests/agent_mandates_spec.rb`; e2e: `web/e2e/w12-agent-mandates.spec.ts`.

## ADR-056 — Public hostname rails.mdlc.ai via Cloudflare Tunnel (test mode)
- **Status:** Accepted (human: host = Cloudflare Tunnel, payments = test mode), 2026-10-28. Moves the build from `deploy: local` (Outcome C) to `deploy: remote: cloudflare-tunnel (rails.mdlc.ai)`. Amends ADR-048 and SEC-1 for the public host only.
- **Topology:** a `tunnel` service (`cloudflare/cloudflared:2026.9.1`, compose profile `public`, `TUNNEL_TOKEN` from `.env`) dials out to Cloudflare, so no inbound host port opens. The tunnel's public hostname routes to `http://edge:8080`, a Caddy listener that compose never publishes. It imports the same `(app)` routes as the localhost site.
- **SEC-1 departure:** on the public host, TLS terminates at Cloudflare with a publicly trusted certificate, and the hop from cloudflared to the edge is plain HTTP inside the compose network. A request that reached Cloudflare over `http` (`X-Forwarded-Proto: http`) gets a 308 to `https`, and Cloudflare "Always Use HTTPS" should be on for the zone as well.
- **HSTS (amends ADR-048):** the `:8080` site sets `Strict-Transport-Security: max-age=63072000; includeSubDomains`. There is no `preload`, because the host is not an apex domain. `TRUSTED_CERT` stays `false`, because `web/proxy.ts` would also send HSTS from the localhost `tls internal` origin, which is exactly the harm ADR-048 prevents. The localhost baseline therefore keeps INV-35's `site/headers` carry.
- **Client IP (SEC-7):** the `:8080` server trusts `private_ranges`, so Caddy keeps Cloudflare's `X-Forwarded-For` and Rack::Attack / `remote_ip` key on the visitor, not on the cloudflared container. Both sites strip `Forwarded`, which Rack 3 and Rails read before `X-Forwarded-For`, so a client could otherwise set its own IP and defeat every per-IP throttle. They also strip `Client-Ip`, which trips Rails' spoofing check and would cause 500s, and overwrite `X-Forwarded-Host` with the real host. Found by the invariant-check review of this change. The localhost site had the same `Forwarded` gap before.
- **Origins:** `APP_ORIGIN=https://rails.mdlc.ai`, and `ALLOWED_ORIGINS` lists the public origin and `https://localhost:9443`, so the local edge keeps working. Mailer links, canonical and og URLs follow `APP_ORIGIN`.
- **Payments:** `STRIPE_SECRET_KEY` stays empty (contract mode), so no real money can move. Live keys need a new ADR and the COMPLIANCE ⚠ rows (ADR-053) reviewed first.
- **Known limits:** SMTP still goes to the local Mailpit, so public visitors don't receive emails. Availability depends on this host being up (single host, as before).

## ADR-055 — Design Quality Gate close: 27 ✓ / 3 ⚠ / 0 ✗, residuals carried, not fixed
- **Status:** Accepted (auto, `review_gates: auto`), 2026-10-21.
- **Context:** The closing independent re-score (`artifacts/design/score-4.md`, c89d74e; RB-24 had no visual effect) moved landing and payments from mechanical ⚠ to ✓. No mechanical design ⚠ is left. Reviewer round 8 passed RB-23 with three advisories.
- **Decision:** Close the gate at 27 ✓ / 3 ⚠ / 0 ✗ over 30 screens and ship without further visual change. Carried, each with its reason:
  - **docs@1280** (both palettes) ⚠ artifact. The capture opens scrolled by about 86 px, so the eyebrow sits under the sticky nav. The page itself has 96 px of top padding. The pinned runner causes it, and the runner is not edited.
  - **ledger@375 and governance@375** (both palettes) ⚠ artifact. The pinned runner's focus probe focuses date inputs below the fold, so the sticky header is captured mid-page.
  - **security:** bulk sign-out of other sessions is a spec-class note, not a ⚠. SPEC lists per-session revoke only.
  - **Reviewer r8 advisories:**
    - A1: the light `--eco-stardancer` `#7b5ce0` is about 4.60:1 where the name sits, and 4.44:1 at the card's bottom edge, where no text is. A darker shade would add margin. Not changed: it passes where the text is, and the token stays as added under ADR-044.
    - A2: below lg the payments header can shift sideways when the live status text changes. This is the deliberate trade made in RB-23, and CLS on the baseline is 0.
    - A3: no unit test pins the 7-card landing grids (3n+1) or the light ecosystem colours.
  - **Optional cosmetic:** `text-wrap: balance` on `.mk-final h2` for the 375 "MONEY" line.
- **Consequence:** These are recorded as residuals in REPORT.md. None of them blocks Stage 4.

## ADR-054 — Landing Lighthouse performance 84 < 85 on the definitive run: gate halted, not accepted
- **Status:** Resolved, 2026-10-21. The definitive run on a quiet host passed with no source change (see **Resolution**).
- **Context:** The definitive `web:baseline --full` on c89d74e (RB-24) failed one check: landing Lighthouse performance 84 (runs 84/84/83), below the pinned target − 10 = 85. Site HSTS is also failing but is carried under ADR-048. Dashboard measures 87. The landing results on the definitive runs so far:

  | Commit | Score | Runs |
  |---|---|---|
  | b638db5 (RB-22) | 85 | 82/85/86 |
  | be7df80 (RB-23) | 83 | 83/84/82 |
  | c89d74e (RB-24) | 84 | 84/84/83 |

  RB-23 changed only CSS and tokens, so the drop from 85 to 83 is within the spread. Twelve isolated runs on c89d74e scored 81–86, median about 84. Within those runs, Lighthouse's own host CPU benchmark (`benchmarkIndex`) ranged from 1,384 to 6,089 under the same 4× applied throttle.
- **Fixed:**
  - RB-22: marketing links no longer prefetch.
  - RB-24: the routing demo formats money without `Intl`, which saved about 75 ms of hydration.
- **Measured and ruled out:**

  | Lever | Result |
  |---|---|
  | No preload for the heading font | First paint unchanged at about 2.0 s, scores 76–84 |
  | Plain `<a>` instead of `next/link` for marketing links | Scores 82–86, median 83.5 |
  | `content-visibility` on landing sections | Not allowed: blanks full-page captures |
  | `font-display: optional` | Saved about 20 ms |
  | Non-latin glyph fallback | About 30–40 ms |

  - **Remaining blocking time:** total blocking time is 450–650 ms and comes from two sources.
    - The first full layout of 906 objects takes about 300 ms at 4×, plus about 117 ms to lay out again after the font swap. A warm relayout of the same page takes 1 ms, and hiding any one section changes it by 4 ms or less. So this is first-use font fallback and text shaping in the Lighthouse container, not anything in the page's CSS.
    - One hydration task of 250–430 ms: Next and React module instantiation plus parsing of the 48 KB RSC flight payload. The page has only two client components (site header and routing demo) and 578 elements.
- **Not accepted:** the `web-baseline` invariant's `accepted` list is only for a failure proven unsatisfiable under a kernel rule, or one that measures a simulator rather than the page. This is neither: b638db5 measured 85, and the throttle is applied rather than simulated. INV-35 therefore stays red, which blocks Stage 4 and `ship`. Per the Web Delivery Baseline Gate, a failure still present on the final full run halts the gate with the failure list.
- **Run after stopping threatwatch-mdlc (2026-10-21T00:58Z, c84c320, rebuilt from HEAD):**
  - Results: landing 83 (83/85/71) and dashboard 84 (87/78/84).
  - Dashboard code has not changed since it measured 88, so the drop and the 71 and 78 outliers come from the host, not the page.
  - With no build work running, the host idles at 35–49% total CPU. node.exe PID 11452 (started Oct 18, 3.9 GB, 33.5 CPU-hours) holds about 85% of one core, and Defender (MsMpEng) takes about 8%.
  - An earlier run at 00:47Z is void, because it measured a stale build (`buildFreshness` FAIL).
- **Halt (`environment`):** the definitive run needs a quiet host. Lighthouse's CPU benchmark varied 4× between back-to-back runs, so the 4× throttle measures this machine's contention as much as the page. Before resuming, stop other workloads (other Compose stacks, heavy apps). On resume the build re-runs `web:baseline --full` once, with no source change, and proceeds if landing reaches 85 or more. If a quiet-host run still measures below 85, the shortfall belongs to the page. The remaining remedy is then structural (for example a static export of the landing page, or a smaller DOM and RSC payload), and it is recorded here before any work starts.
- **Resolution (2026-10-21T19:19Z, c84c320, no source change since RB-24):**
  - The human ended node.exe PID 11452, and the host settled at about 12–20% total CPU.
  - The re-run scored landing **93** (92/95/93, LCP 1781 ms, CLS 0.002) and dashboard **93** (93/96/93, LCP 1555 ms, CLS 0).
  - The only baseline failure left is HSTS, carried under ADR-048. `npm run lint:invariants` is 39/39 machine checks passing, so INV-35 is green.
  - The 83–84 scores were therefore host contention, not the page, and no structural landing change is needed. The landing performance margin remains a residual, because on a loaded host it measures below 85.

## ADR-053 — COMPLIANCE posture at the Stage 3 wide close: 62 ✓ / 7 ⚠ / 0 ✗
- **Status:** Accepted (auto, `review_gates: auto`), 2026-10-20.
- **Context:** The COMPLIANCE.md Gate re-checked every row against the code, the specs and the definitive `web:baseline --full` (`FRONTEND-AUDIT.summary.json`, gitSha 1f22df8, 30 screens / 150 cells / 0 failing). It also added the WEB.md §2/§3/§6 rows the gate requires.
- **Decision:** Proceed to the Design Quality Gate with posture B (⚠). There is no ✗ row. The seven ⚠ rows are recorded residuals, not gaps:
  - HSTS is absent on the localhost edge (SEC-1, SEC-18, PCI-4, WEB.md §6), carried under ADR-048.
  - Three LOW audit residuals: SA-4 (bcrypt reads only the first 72 bytes), N3 (webhook sweep fairness) and N4 (idempotent-resume edges).
- **Consequences:** REPORT.md carries the posture and the ⚠ list. A production deploy must add `Strict-Transport-Security` at the edge before PCI-4 and SEC-1 can move to ✓.

## ADR-052 — The human re-pin of the baseline runner (77028f0) ships as a recorded INV-38 exception
- **Status:** Superseded, 2026-10-20. The govern hook decided 77028f0 itself as C-5 = BLOCK (GD-0043), so it could not stay. The human reverted it with the hook's command (fcd30bd, "Revert blocked C-5"), and INV-38 passes again. v1.8 returns through option 2: a kernel fix that lets a verbatim re-pin be decided against a manifest of published kernel hashes, followed by a normal build re-pin. The text below is kept as recorded.
- **Originally:** Accepted by the human, 2026-10-20 (reply "1,2,3" to the governance halt; read as options 1 and 2, because option 3, reverting the re-pin, undoes what option 1 keeps).
- **Context:** web-baseline v1.7 cannot start Lighthouse inside the Playwright container, so INV-35 can never pass on it. Kernel v1.8 (4a30730, PR #30) fixes that. The build's own re-pin (RB-10) was BLOCKed by policy as a kernel-path modify (GD-0040) and reverted (cee1053). The human then committed the verbatim re-pin as 77028f0. Under the pinned `govern.mjs` v1.4, `classify 77028f0` returns `build scope kernel -> minimum BLOCK`, so any decision would force a Revert, and with no decision INV-38 reports the commit as ungoverned. The kernel has no path through which a verbatim re-pin can be decided.
- **Decision:**
  - 77028f0 stays. It is a human commit (signed off by the human), verified byte-equal to kernel 4a30730 `reference/web-baseline.mjs`, and INV-36 passes.
  - No governance decision is recorded for it. The agent never runs `govern.mjs decide`, and a decision would be a BLOCK.
  - The failing INV-38 line on 77028f0 is reported verbatim in REPORT as a governance residual. It is not carried silently, and the pinned lint is not edited.
  - The kernel fix (option 2) is raised as a kernel PR, so a later verbatim re-pin is decidable: an install-from-manifest rule that accepts a pinned file only when its content hash matches a published kernel reference.
- **Consequences:** v1.0.1 ships with INV-38 failing on exactly one commit, for the reason above. Any other INV-38 failure is still a blocker. Once a fixed kernel is installed, the exception can be retired by recording a decision on 77028f0.

## ADR-051 — A scan-to-pay roadmap card on the landing page, kept generic
- **Status:** Superseded by ADR-057 (2026-11-11): the card was removed with the other deferred rail cards. Originally accepted (build, at the human's request, 2026-10-18). This departed from SPEC: the landing page is a port of the design template, and this added one card the template does not have.
- **Decision:** the "Every rail" network grid (section 02) gets a seventh node, "Scan to pay · Real-time account rails", with the existing `Deferred · not in v1` status. Its copy says only that instant, QR-initiated bank payments are on the roadmap, through the same API and via licensed partners. It names no market, scheme, provider, data model or flow. The human asked that no IP be given away.
- **Not public:** the design is in `docs/proposals/QR-PAYMENTS.md`, marked internal. It is not linked from the site, sitemap or docs page, and it is not part of the SPEC. It becomes a SPEC addendum and its own ADR only when scheduled, after 1.0.1.
- **Consequences:** none in the API or console. The card reuses the node markup and tokens (ADR-044) and is labelled as deferred, like the other roadmap nodes, so the page still claims nothing it doesn't ship.

## ADR-050 — Drop declared values and switches that nothing uses (Reviewer round 1, item 9)
- **Status:** Accepted (build, autonomous; Reviewer FAIL on 03a93c8). Precedence: the kernel's "no unreachable declarations" Reviewer item over the SPEC §2 CHECK lists and the SPEC §8 env table. This is a departure from SPEC.
- **Context:** SPEC §2 allows `journal_entries.kind = 'reversal'` (with `source_type = 'JournalEntry'`) and `webhook_events.source = 'sandbox'`. SPEC §8 lists `ALLOW_HTTP_WEBHOOKS` and `SSRF_ALLOW_HOSTS` for a non-production smoke receiver. Nothing writes either value. Ledger journals are immutable, and no manual-correction flow is in scope. Only Stripe events are received, and sandbox payments emit nothing inbound. The smoke test passes 53/53 against a public https receiver, so it needs neither switch. `SSRF_ALLOW_HOSTS` was never read, and `ALLOW_HTTP_WEBHOOKS` was only validated.
- **Decision:** remove them rather than wire them in.
  - Migration `20261018001800_rb3_narrow_unreachable_values` narrows the `journal_entries.kind`, `journal_entries.source_type` and `webhook_events.source` CHECKs, and the model constants and the console's kind and source vocabularies follow.
  - The two env vars leave `.env.example`, `EnvSchema` and the load-test overlay. The SSRF guard is https-only, with no host exemptions, in every environment.
  - Wiring the switches would have added a non-production SSRF bypass that no flow needs.
- **Consequences:**
  - A later correction feature restores `reversal` with its own migration, writer and tests.
  - A deployment that sets either variable sees no effect. Unknown variables are ignored, not rejected.

## ADR-049 — An unseen id in a request body is a 404, like one in the path
- **Status:** Accepted (build, autonomous; found by the smoke test). Precedence: SPEC SEC-5 and the error table (`NOT_FOUND`: "missing, or belongs to another tenant/user") over the F-10 implementation.
- **Context:** `POST /v1/refunds` and `POST /v1/sandbox/disputes` take the payment id in the body. When the tenant could not see that payment, they answered `422 VALIDATION_FAILED` with a field error on `payment_intent`. SEC-5 says another tenant's id is a 404. The 422 leaked nothing, since the text was the same for a missing id and a foreign one, but it was not the specified answer.
- **Decision:** both endpoints answer `404 NOT_FOUND` ("No payment … in <mode> mode") when the referenced payment is missing, belongs to another tenant, or is in the other mode. Malformed bodies stay 422. The console shows non-field problems as a toast. The sandbox form only offers the tenant's own payments, so it can't reach this answer except through a race. The request specs and `smoke-test.sh` pin the 404.

## ADR-048 — No HSTS on the local `tls internal` origin (web-baseline residual)
- **Status:** Accepted (build, autonomous). Precedence: SPEC SEC-1 and the kernel's security-headers reference over the web-baseline header check.
- **Conflict:** the web baseline flags "no Strict-Transport-Security" on `https://localhost:9443`. SEC-1 sends HSTS only when `TRUSTED_CERT=true`, meaning a publicly trusted certificate. HSTS pinned on a `tls internal` localhost certificate would make every other local project on that host name unreachable over plain HTTP for a year, and it can't be undone from the server.
- **Decision:** `web/proxy.ts` emits `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload` only when `TRUSTED_CERT=true`. The edge (`edge/Caddyfile`) never sets it. The local run carries the header check as a recorded residual in REPORT and FRONTEND-AUDIT. A deployment with a trusted certificate sets `TRUSTED_CERT=true`, and the same check then passes. The failure goes on INV-35's `accepted` list in `invariants.json` (`site`/`headers`, ADR-048), so the invariant lint prints it as `carried:`. That entry stops matching, and blocks, if the header check ever passes or fails for another reason. `edge/Caddyfile` also stops advertising HTTP/3 (`protocols h1 h2`), because only TCP 443 is published and an `Alt-Svc: h3` would point clients at a port that is closed.

## ADR-047 — Load test (SPEC §9): isolated `loadtest` environment, k6 mixes
- **Status:** Accepted (build, autonomous).
- **Harness:** `sh scripts/loadtest.sh` (full) or `sh scripts/loadtest.sh sanity` brings up `compose.loadtest.yaml`: an `api-load` service built as `rails-api-loadtest` from `./api` (the running `rails-api` image is never rebuilt), and `grafana/k6:1.3.0`. Four mixes run: read, write, idempotent replay and login. `restore.js` then proves the limiter is back at production limits (a burst draws `429 RATE_LIMITED`). `loadtest/report.mjs` writes `artifacts/loadtest/RESULTS.md` with p50/p95/p99, error rate and throughput per mix against the SPEC §9 targets.
- **Seed:** `loadtest:seed` (`api/lib/loadtest/seed.rb`) refuses to run unless `Rails.env.loadtest?`. It creates merchants through the signup service, then bulk-inserts intents, capture journals and ledger lines in 5,000-row transactions, so the deferred balance trigger still checks every journal.
- **Where `loadtest` differs from production, and nothing else:** its own database `rails_loadtest` (replicated like the rest), Redis logical db 1 with the `loadtest-cache` namespace, and logs at warn. `HIBP_ENABLED=false`, because a load test must not call the internet. `ALLOW_RATE_LIMIT_SCALE` with `RATE_LIMIT_SCALE=1000` is accepted only here, so the mixes measure the app, not the limiter. `OPS_TRIGGER_ENABLED` becomes settable and the overlay pins it to false. k6 calls `http://api-load:3000` directly, so the Caddy edge and web tier are outside the measurement. `RESULTS.md` lists the surfaces the mixes do not exercise.
- **Finding it produced:** the first write mix deadlocked (`ActiveRecord::Deadlocked`, 24.8 % 500s). Fixed by ADR-045.
- **Host caveat:** on the shared development host, other projects' containers compete for CPU, so latency numbers from this machine are recorded with that caveat. CI or a dedicated host is the place to judge the §9 latency targets.
- **Below saturation vs saturation (RB-3, load methodology clauses 5–6):** the 200-VU `sustained` run is past the knee on this host (every mix's step-up saturates at its first step, 50 VUs, against 10 Puma threads), so its multi-second p95s are queueing. A `light` profile (4 VUs × 60 s per mix, fewer than the api's threads) now runs after it, and RESULTS.md reports it as the below-saturation row. It still misses every §9 target: health p95 148 ms (p50 30 ms), the reads 128–184 ms, confirm 843 ms, refund 655 ms, replay 54 ms and login 1.52 s (p50 663 ms, where a cost-12 bcrypt compare alone is about 250 ms on an idle core). With 0 % errors, 0 duplicate charges and the limiter restore passing, the spread between p50 and p95 even for `/api/health` points at CPU contention on the shared 4-CPU Docker VM (other projects' stacks were running) rather than at the app's own service time. The verdict stays **FAIL** and is reported as a finding in REPORT.md. The test was not retuned to pass. Re-running `sh scripts/loadtest.sh` on CI or a dedicated host is the open action, and the write paths (confirm, refund) get a profile there before any change to the lock order (ADR-045).

## ADR-046 — Indexable account entry points, social card, ticker as a scroll region
- **Status:** Accepted (build, autonomous).
- **Indexing:** `/sign-up`, `/sign-in` and `/forgot-password` are public entry points that a search for the product should reach, so they join `PUBLIC_ROUTES` and the sitemap, and get full `pageMetadata` (50–160-char description, canonical, Open Graph, Twitter). The token routes `/reset-password/[token]` and `/invite/[token]` carry a credential in the URL: each segment layout sets `noindex, nofollow`, robots.txt disallows them, and they are never in the sitemap (web-baseline screens marked `public: false`). The `(auth)` group's layout-wide noindex is removed.
- **Social card:** `web/public/og.png` (1200×630), rendered by `npm run og:image` (`scripts/og-image.mjs`, Playwright, template tokens, system fonts). Every public page links it as `og:image` with its size and alt, and as `twitter:image` with `summary_large_image`.
- **Ticker:** SPEC §6 keeps the capability marquee (disabled under reduced motion). It is a labelled, focusable horizontal scroll region rather than a clipped strip, so it can be swiped or arrow-keyed, and it pauses on hover or focus (WCAG 2.2.2). The final CTA's 1100px horizon ring is now a background gradient on its section, so no element overflows the viewport.

## ADR-045 — Write lock order: intent row, merchant audit chain, ledger accounts
- **Status:** Accepted (build, autonomous; found by the ADR-047 load test).
- **Context:** the hash-chained audit trail (SEC-15) serialises appends per merchant with a transaction-scoped advisory lock. Captures took that lock, then ledger account rows (`Hooks.changed!` audits, then posts). Refund settlement posted its journal first, taking ledger account rows, then audited. Two concurrent writes on one merchant could each hold what the other needed, and Postgres aborted one with a deadlock, which surfaced as a 500.
- **Decision:** one lock order for every write: the intent's row (`lock!`), then the merchant's audit chain lock, then ledger account rows. `Audit.lock_chain!(merchant)` is the single place that takes the lock. `Ledger::Post.call!` takes it before it touches an account, and `Audit.record!` takes it again, which is safe because advisory transaction locks are re-entrant. Disputes post through the same path.
- **Consequence:** a merchant's writes still queue on the chain lock, which the hash chain requires (each row hashes its predecessor), but they can no longer deadlock. `spec/services/ledger/post_spec.rb` pins the order on the refund path. The write mix is re-run to confirm.

## ADR-044 — Contrast-safe tokens beside the Design Template palette
- **Status:** Accepted (build, autonomous). Precedence: kernel WCAG 2.2 AA (4.5:1 text) over the design template.
- **Measured:** white on `--violet` #8257FF is 4.41:1, so the template's filled primary button and skip link fail AA. Template `--dim` #6B7894 on `--bg2` is 4.28:1 (the sidebar's group labels). The derived light palette's cyan, amber and green each sat just under 4.5:1 on its surfaces.
- **Decision:** the template's hues stay. `--violet` keeps #8257FF for text, borders and glows. Filled controls behind white text use the new `--violet-fill` #7C53F2 (hover `--violet-fill-hover` #6D44E6, replacing the template's #9170FF hover, which lightens). Dark `--dim` moves to #7683A0, which clears 4.5:1 on bg, bg2 and surf. In the derived light palette, cyan is #00798D, amber #8B5400 and green #007A59. `--on-amber` and `--on-green` choose black or white text on those fills per theme.
- **Consequence:** the web baseline's axe color-contrast check passes in both palettes. Every deviation is listed in DESIGN-NOTES.

## ADR-043 — Verification Gate harness: web baseline in a trusted container, coverage, mobile e2e
- **Status:** Accepted (build, autonomous).
- **Why a container:** the shipped stack serves the console only through the Caddy edge with `tls internal`. The pinned runner (`scripts/web-baseline.mjs` v1.7, unmodified) opens plain Playwright contexts, and on the Windows host neither Chromium (`ERR_CERT_AUTHORITY_INVALID`) nor Node `fetch` (`UNABLE_TO_GET_ISSUER_CERT_LOCALLY`) trusts that CA. Editing the runner is a gate failure, and auditing a bare `next start` would judge headers and cookies that do not ship. Changing the host's trust store is outside the build's remit, and on Windows it needs an interactive confirmation.
- **The harness:** `npm run web:baseline` (`scripts/web-baseline-docker.sh`) runs the runner verbatim inside `mcr.microsoft.com/playwright:v1.64.1-noble`, the same version as `@playwright/test`, with `libnss3-tools` added by `scripts/web-baseline.Dockerfile`. The container joins the compose network. It trusts Caddy's root through its NSS db (Chromium) and `NODE_EXTRA_CA_CERTS` (Node), so no check runs with certificate errors ignored. A loopback forwarder maps `localhost:9443` to `edge:443`, so the audited origin, `https://localhost:9443`, is byte-for-byte the origin users and the e2e suite reach, and CSP, HSTS, cookies and the 404 are judged as shipped. The served image's `.next/BUILD_ID` is copied out with its build-time mtime to `web/.next/BUILD_ID` (`buildIdFile`), so `buildId` and `buildFreshness` compare the running build against the working tree. Runner deps are root dev dependencies pinned exact: `@playwright/test` 1.64.1, `@axe-core/playwright` 4.13.0, `lighthouse` 13.5.0. Each is the newest stable release at least two weeks old.
- **The audited account:** a single session must render all 30 SPEC §6 screens, `/ops` included. `ops:seed_operator` never promotes an existing account, and that guard stays. The wrapper therefore seeds `WEB_BASELINE_USER` as the platform operator through that task. `scripts/web-baseline-seed.mjs` then has a dedicated owner (`web-baseline-owner@example.com`, merchant "Baseline Goods") create one record per detail screen through the public API: a payment, customer, refund, sandbox dispute, webhook endpoint and journal entry. The owner invites the operator as admin, and the operator accepts by the W8.2 flow. The config's detail `exampleRoute`s are rewritten to those real ids. Credentials come from `WEB_BASELINE_USER` / `WEB_BASELINE_PASSWORD`. Locally, a random password is generated once into the gitignored scratch `artifacts/web-baseline/.ca/`, and the config names only the env vars. Before each run the wrapper clears the rack-attack counters on the host, the way the e2e suite does (`config.limiterReset` would run inside the container, which has no docker CLI). `reset-password` and `invite` are audited with the SPEC's example tokens, which render their designed invalid-token states.
- **Theme mapping:** the Design Template makes dark the default palette, and light is the alternate (`:root[data-theme='light']`). The runner's forced pass therefore sets `darkValue: "light"`. Report cells labelled `light` measure the default (dark) palette users get first, and cells labelled `dark` measure the forced light palette. Both palettes are measured, contrast included, and the committed previews show the default palette.
- **Viewports:** 375 / 768 / 1280, the runner default. 375 is the RESEARCH mobile floor, and 768 and 1280 are RESEARCH breakpoints. 640 and 1024 are not separate cells: the shell switches only at 768 (drawer) and 640 (table cards), and the 375 cell covers the card layout.
- **Coverage:** SimpleCov 1.3.1 (`api/spec/rails_helper.rb`) prints statement coverage per group, and the gate judges `Services`, `Domain (models)` and `Lib` at ≥ 80 % (comprehensive). Web unit coverage (`npm run test:coverage`, `@vitest/coverage-v8` pinned to the installed vitest 5.0.5, which must match exactly) is reported, not thresholded.
- **E2E:** Playwright now runs a Desktop Chrome project and a Pixel 7 project. Global setup first proves the server answering `BASE_URL` is RΛILS (a healthy `/api/health` plus the landing `<title>`) and refuses to run otherwise. TLS verification is relaxed only for a localhost target.
- **CI parity:** `.github/workflows/ci.yml` runs api (rubocop, brakeman, bundle-audit, rspec with coverage), then web (typecheck, lint, format, coverage, build, audit), then delivery (launch, smoke, `web:baseline -- --full`, e2e, invariant lint).

## ADR-042 — Platform operator console (F-18)
- **Status:** Accepted (build, autonomous).
- **Non-operators get 404:** every `/v1/ops/*` route inherits `Api::V1::Ops::BaseController`, which checks `platform_role = 'operator'` before anything else and otherwise answers the same `404 NOT_FOUND` ("No route matches …") as an unknown path. SPEC §9 asks for this ("ops authz (non-operator 404)"), and it keeps the cross-tenant surface from being disclosed to merchant users or API keys. F-17's `scheduled_runs` moved onto the base, so its earlier 403 for merchant users is now 404. A disabled trigger is still `403 FORBIDDEN`, because only an operator reaches that check. The `/ops` screen calls `notFound()` for non-operators and renders the site 404.
- **No PII:** the merchant list returns name, slug, country, currency, status, the latest suspension reason and a member count. It never returns the support email or any member identity. DLQ emails show a masked recipient (`k***@example.com`) and never their template params, which can hold one-time links. DLQ webhooks reuse the delivery shape plus the merchant id and name.
- **Suspension:** `PATCH /v1/ops/merchants/:id {status, reason}` requires a reason (1–500 chars) to suspend; it is optional to reactivate. The change runs under a row lock and is audited on the merchant's own chain as `merchant.suspended` / `merchant.reactivated`, with `from`, `to`, `reason` and the operator's id, so the merchant's audit log shows who acted and why. Repeating the current status is a no-op with no audit row. Suspension is enforced by the existing `Authentication#enforce_not_suspended!` (SEC-19): every S and K request answers `403 MERCHANT_SUSPENDED`, except auth (session read, logout, sign-in) and the merchant switcher. Ops controllers are exempt, so an operator can always reactivate. `status_reason` comes from the latest suspension audit row, in one query per page, so no new column is added.
- **Search:** `q` matches name or slug (`ILIKE`, with `%`/`_` escaped) or an exact `acct_` id, and is trimmed to 120 chars. `status` and DLQ `kind` are allow-listed (`422`). The DLQ lists both queues newest first. Email rows have no public id, so their cursor is the row id.
- **Operator access in the console:** an operator with no merchant of their own lands on `/ops` after sign-in, while an explicit `?next=` still wins. The e2e journey seeds a local operator with the same idempotent `ops:seed_operator` task the api runs at boot (local compose only). Remote targets pass `E2E_OPERATOR_EMAIL` / `E2E_OPERATOR_PASSWORD`, or the journey is skipped.

## ADR-041 — Scheduled operations (F-17)
- **Status:** Accepted (build, autonomous).
- **Claim (INV-4):** every T3–T6 run goes through `Scheduled::Run`, which inserts a `scheduled_runs` row with `INSERT … ON CONFLICT (job_name, run_key) DO NOTHING`. The run key is the UTC start of the schedule window (`2026-10-18T12:15Z`), with `:m<merchant id>` appended for a per-tenant run. A second worker, or a retry of the same window, therefore does nothing and returns `nil`. The row moves `running → succeeded | failed` with the counts or the error class and message (500 chars), and a CHECK ties `finished_at` to the terminal states. The clock is injected (`now:`) everywhere.
- **Names:** ARCHITECTURE §6 names win over the plan's file names: `ExpireStaleIntentsJob` (T4) and `RescreenPasswordsJob` (T6). The plan's `dispatch_due_job` is dropped because T1 and T2 already run their own every-minute sweeps. The schedules are cron strings with an explicit `UTC` (`5 * * * *`, `*/15`, `0 3 * * *`, `*/30`) on the `default` queue.
- **T3 ledger integrity:** recomputes every account's balance from its lines (signed in the account's normal direction) and every journal's debit/credit totals and line count. The deferred F-09 trigger already blocks an unbalanced journal at commit, so a finding means drift or a bypassed constraint. Each one is a `ledger.imbalance_detected` audit event (`severity: "dlq"`) on the merchant's chain, and the run counts are `checked`, `journals_checked` and `imbalanced`.
- **T4 stale intents:** `requires_payment_method` / `requires_confirmation` intents older than 24 h are canceled in batches of 500. Each one is re-checked under a row lock, so a confirm that wins the race is left alone, and goes through the state machine and `Payments::Hooks`: it is audited, emits `payment_intent.canceled` to the outbox and live feed, and busts the report cache. Their reason is the system-only `expired`, added to the DB CHECK. The API's `cancellation_reason` input list is unchanged, so a client cannot send it.
- **T5 retention:** deletes idempotency keys older than 30 days (the replay window is 24 h). Webhook events lose their payload at 30 days and the row at 90 days, well past the 72 h dedupe minimum; outbound delivery rows cascade. Sessions that are expired, revoked or idle-dead and password resets that are expired or used are deleted in the global run only, because they belong to users, not merchants.
- **T6 HIBP re-screen without the plaintext:** ADR-006 lets registration and password change proceed when HIBP is unreachable, but nothing can be re-checked later without some form of the password. When the screen is pending, `users.breach_check_sha1` keeps the password's SHA-1, encrypted with Active Record encryption (non-deterministic). A CHECK allows it only while `breach_check_pending`, and it is cleared on every outcome. The job runs the same k-anonymity range query from that hash, so only the 5-character prefix leaves the process:
  - **clean:** the flag and the hash are cleared;
  - **breached:** the flag and the hash are cleared, `password_compromised_at` is set, every session is revoked, `user.password_compromised` is audited and the existing `breach_notice` email is queued. Sign-in then answers `422 PASSWORD_BREACHED` ("Reset it with Forgot password"), but only after the correct password, so a guesser learns nothing. Any new password clears it;
  - **unavailable:** the user is left for the next run.

  An unsalted SHA-1 is weaker than the bcrypt digest. It is kept only for the minutes or hours HIBP is down, only for affected users, and only encrypted under the app key, which is a smaller exposure than never re-screening (SEC-8). A flag set before F-17 has no hash and clears at the user's next password change.
- **Ops endpoints:** `GET /v1/ops/scheduled_runs` (operator session; `job_name` filter, cursor) lists runs newest first. `POST` (operator, non-production and `OPS_TRIGGER_ENABLED=true`; the env schema already refuses that flag in production) runs the job inline and answers `202` with the recorded run, trigger `manual`. An `Idempotency-Key` becomes the run key (`manual:<key>`), so a retried request returns the same run. The tenant idempotency store is not used, because an operator has no active merchant. Each trigger is audited on the platform chain as `scheduled_run.triggered`.

## ADR-040 — Dashboard & analytics (F-14)
- **Status:** Accepted (build, autonomous).
- **Window:** every report reads `[from, to)` in UTC for one merchant, mode and currency (`Reports::Window`). The defaults are the last 30 days, `to` = now and the merchant's default currency. Granularity is `day` (default) or `hour`. A daily window spans at most 366 days and an hourly one at most 7 days; past that the API returns `422 OUT_OF_RANGE`, so a timeseries stays within a few hundred points. Invalid timestamps, currencies or granularities return `422 INVALID` on the named field.
- **Figures:** payments count by their `created_at`, and refunds by their own `created_at`, so a refund lands in the window it was made. `gross = Σ amount_captured_minor`, `fees = Σ fee_minor`, `refunds = Σ succeeded refunds`, `net = gross − fees − refunds`. `success_rate = succeeded / (succeeded + failed)`, or `null` when neither happened, so in-flight payments do not drag the rate down. `by_status` lists every status (zeros included); `by_processor` is sorted by count.
- **Timeseries:** `date_trunc` (the granularity is allow-listed and bound as a parameter) groups the rows. Every bucket in the window is returned and empty buckets are zeros, so the console never fills gaps itself.
- **Replica + cache:** all three reads run under `connected_to(role: :reading)`. Summary and timeseries are cached for 15 s (ARCHITECTURE §Caching) under a key that includes a per-merchant/mode version. `Realtime::Broadcast` (the after-commit hook) bumps that version on every payment, refund or dispute change, so the KPI tiles never lag the live feed that announced the change. Replica lag can still serve a figure up to one TTL old; that is accepted for a dashboard. The ledger stays the source of truth.
- **CSV export (W5.4, SEC-17):** this is a `finance` capability (owner, admin, analyst). It returns one row per payment intent in the window, oldest first, and can be filtered by status. Every cell is escaped: a value starting with `= + - @ TAB CR` gets a leading apostrophe, and cells containing quotes, commas, line breaks or edge whitespace are RFC 4180 quoted. The export is written by hand (no `csv` gem in the bundle). It is capped at 50,000 rows (`422 OUT_OF_RANGE`, "export a shorter range"), sent `Cache-Control: no-store`, and audited as `report.exported` with the window, currency and status filter. The console downloads it with `fetch`, because the `Rails-Mode` header cannot ride on a plain link.
- **Indexes:** `(merchant_id, mode, currency, created_at)` on `payment_intents` and `refunds`, so a window reads its slice instead of the tenant's whole history.
- **Console:** the dashboard shows six KPI tiles (gross, net, fees, refunds, payments, success rate), a captured-volume bar chart and status/processor breakdowns above the live feed. It uses the merchant's default currency and has a range select (24 h hourly, 7/30/90 days daily). `/reports` adds From/To dates, currency, granularity and the export panel; roles without export see a short note instead of the controls. The chart's SVG is `aria-hidden` and the same buckets are in a visually hidden table. Both views reload, debounced, on live payment/refund/dispute events.

## ADR-039 — Realtime console feed (F-13) and a sequential wave 7
- **Status:** Accepted (build, autonomous).
- **Wave 7 runs sequentially:** BUILD-PLAN gives wave 7 to three sub-agents. They would share one test database (`rails_test_orch`), one live compose stack and the e2e rate-limit reset, so parallel runs would interfere with each other's suites and rebuilds. The orchestrator builds F-13 → F-14 → F-17 in order instead. The file sets stay disjoint, so nothing else changes.
- **Socket auth (SEC-13):** `GET /cable` connects only with a live console session cookie (`__Host-rails_session`, the same `Session.authenticate`). API keys cannot subscribe. Origins come only from `ALLOWED_ORIGINS`, and `allow_same_origin_as_host` is off so a Host header cannot widen the list.
- **Channel authorization:** `MerchantEventsChannel` takes `{mode}` and streams `merchant_events:<merchant>:<mode>` for the session's active membership, if the merchant is not suspended. Every 60 s it re-checks the session (idle and absolute expiry included, because an open socket is not activity), the membership and the active merchant. If any changed, it stops streaming and rejects the subscription.
- **Message budget:** each connection may send 20 frames per 10 s. The 21st closes it with `reconnect: false`, and the console shows the feed as paused.
- **Broadcast:** `Payments::Hooks::BROADCAST` resolves to `Realtime::Broadcast`, which runs after commit, so a rolled-back change is never shown. It sends only `payment_intent.*`, `refund.*` and `dispute.*`, and only a summary (`id, object, status, amount_minor, currency, payment_intent, reason, created_at`): no metadata, card details, customer or description. A broadcast failure is logged and never fails the committed request.
- **Console:** a dependency-free client speaks the Action Cable JSON protocol over `WebSocket` (welcome → subscribe → confirm). It reconnects at 1 s doubling to 30 s, and never reconnects after a reject or a no-reconnect disconnect. A new merchant or mode opens a new socket and starts an empty feed. The dashboard's "Live activity" panel keeps the last 20 events, with a status line (Connecting, Live, Reconnecting, Paused). The payments list reloads its first page on a payment or refund event, debounced 400 ms. Once more pages are loaded it offers "Show latest" instead. The CSP's `connect-src 'self'` already covers same-origin `wss:`.
