# COMPLIANCE — spec-to-code traceability

This table maps each SPEC §4 control to the code that implements it and the test that proves it. A row's status is set by the feature that owns the control, and the COMPLIANCE.md Gate re-checks every row against the final tree.

Status values:
- **✓** — built and tested.
- **⚠** — built, but an audit note or known limitation is open: a residual LOW finding, or a deviation recorded as an ADR.
- **✗** — a gap: not built, or an open CRITICAL or HIGH audit finding.

The statuses below come from the COMPLIANCE.md Gate run at the Stage 3 wide close on 2026-10-20. That run used commit a698c87 (RB-17) and the definitive `web:baseline --full` measured at gitSha 1f22df8 (`FRONTEND-AUDIT.summary.json`, buildMode production, buildFreshness current).

## §4.1 Threat-derived controls

| ID | Control (short) | Implementation | Evidence (tests) | Status |
|---|---|---|---|---|
| SEC-1 | HTTPS only at the edge | `edge/Caddyfile` (`tls internal`, :80 → 308), `config.assume_ssl` (TLS terminates at the edge) | smoke test (Stage 4) | ⚠ ADR-048: the edge negotiates TLS 1.3 with X25519MLKEM768 and :80 → 308, but `Strict-Transport-Security` is absent on localhost (`site.headers`). The production edge must add it. |
| SEC-2 | `__Host-` session cookie, idle 30m / absolute 12h, rotation | `app/models/session.rb`, `concerns/authentication.rb` | `spec/requests/auth/sessions_spec.rb`, `signup_spec.rb`; RB-3 SA-3: post-sign-in `next=` stays same-origin — `web/app/(auth)/password-input.test.ts` | ✓ F-02, RB-3 |
| SEC-3 | CSRF: HMAC token + allowed Origin | `concerns/authentication.rb` (`csrf_token_for`), `Api::V1::Auth::Support#verify_origin!` | `sessions_spec.rb`, `signup_spec.rb` (cross-origin), `invitations_spec.rb` | ✓ F-02 |
| SEC-4 | API keys: 32 random bytes, digest only, shown once, revocable, mode-bound | `app/models/api_key.rb`, `api_keys_controller.rb`, migration `20261018000500` (UNIQUE digest, prefix↔mode CHECK) | `spec/requests/api_keys_spec.rb`, `api_key_auth_spec.rb` | ✓ F-05 |
| SEC-5 | Tenant isolation; foreign id → 404 | `concerns/tenant_scoped.rb` (`scoped`, `find_scoped!`), `for_tenant` scopes | `members_spec.rb`, `invitations_spec.rb`, `api_keys_spec.rb` (foreign + other-mode 404) | ✓ all 19 features; INV-10 (no unscoped tenant query) passes; cross-tenant smoke |
| SEC-6 | Object-level authz by role (§4.2) | `concerns/authorization.rb` MATRIX, `app/policies/team_policy.rb` | `team_policy_spec.rb`, `members_spec.rb`, `api_keys_spec.rb`, `api_key_auth_spec.rb` | ✓ all 19 features (§4.2 matrix below, every row ✓) |
| SEC-7 | Rate limits | `config/initializers/rack_attack.rb` (login, signup, reset, accept, api/key 100/s, money 25/s, evidence export 10/min), logo 5/h in `merchant_logos_controller.rb` | `signup_spec.rb`, `sessions_spec.rb`, `api_key_auth_spec.rb` (100/s), `payment_intents_spec.rb` (money 25/s); RB-3 SA-1: throttles match `RateLimitKeys.endpoint` (router-normalised path) and read JSON or form bodies — `sessions_spec.rb` (`/login/`, `/login.json`, form-encoded, mixed-case email share one limit) | ✓ F-08 (refunds F-10), RB-3 |
| SEC-8 | Passwords 15–128, HIBP k-anonymity, bcrypt 12; fail-open re-screen (ADR-006/041) | `app/services/passwords/*`, `jobs/rescreen_passwords_job.rb` (encrypted pending SHA-1, sign-in refused once breached) | `services/passwords/breach_spec.rb`, `signup_spec.rb`, `jobs/rescreen_passwords_job_spec.rb` | ⚠ F-02, F-17; SA-4 LOW deferred (bcrypt reads only the first 72 bytes, SECURITY-AUDIT.md) |
| SEC-9 | Lockout 10 → 15 min, generic error | `app/services/passwords/login.rb` | `services/passwords/login_spec.rb` (RB-3 SA-6: a locked account pays the dummy bcrypt compare) | ✓ F-02, RB-3 |
| SEC-10 | PAN guard | `app/services/security/pan_guard.rb` (customers, payment methods, payment intents) | `spec/services/security/pan_guard_spec.rb`, `customers_spec.rb`, `payment_intents_spec.rb` | ✓ F-08 |
| SEC-11 | Inbound webhook signature before DB write | `api/v1/webhooks/stripe_controller.rb` (`construct_event` with a 300 s tolerance before `InboundWebhooks::Receive`; INV-6) | `requests/webhooks/stripe_spec.rb` (forged/tampered/stale/missing → 400, no row written; unset secret → 503) | ✓ F-11 |
| SEC-12 | Outbound SSRF guard + HMAC | `security/ssrf_guard.rb` (`check_url!` on save; `assert_safe_url!` resolves and pins before `Net::HTTP` in `outbound_webhooks/dispatcher.rb`, INV-5), `outbound_webhooks/signature.rb` | `services/outbound_webhooks/ssrf_guard_spec.rb` (schemes, credentials, internal names, private/CGNAT/metadata/IPv6 literals, rebinding answer), `dispatcher_spec.rb` (private resolution → no request, 3xx not followed, timeout, 64 KB cap, verifiable signature); RB-3: IPv6 transition ranges blocked (SA-5), 5 s overall attempt deadline and slow-endpoint deferral per sweep (SA-2) — `dispatcher_spec.rb`, `jobs/deliver_pending_webhooks_job_spec.rb` | ✓ F-12, RB-3 |
| SEC-13 | WebSocket origin / auth / budget | `config/initializers/cable.rb` (allow-list only, no same-as-host), `channels/application_cable/connection.rb` (session cookie, 20 frames/10 s), `channels/merchant_events_channel.rb` (membership + mode, 60 s re-check) | `spec/channels/application_cable/connection_spec.rb` (missing/revoked/expired session, origins, budget), `spec/channels/merchant_events_channel_spec.rb` (mode, no membership, suspended, revocation/switch, after-commit summary) | ✓ F-13 |
| SEC-14 | PII encrypted at rest, redacted from logs/audit | AR Encryption on `users.email/name`, `merchants.support_email`, `invitations.email`, `pending_emails.to_address/params`, `customers.email/name/phone/billing_address`, `payment_methods.processor_token`; `LogRedaction` (path tokens + query values); filter_parameters | `spec/lib/log_redaction_spec.rb`, `audit_spec.rb`, `models/customer_encryption_spec.rb` | ✓ F-06 |
| SEC-15 | Hash-chained, immutable audit | `app/services/audit.rb`, F-01 trigger, `services/governance/chain_verifier.rb`, `evidence_bundle.rb` | `services/audit_spec.rb`, `services/audit_chain_spec.rb` (tamper), `governance_spec.rb` | ✓ F-15 |
| SEC-16 | Upload hardening | `app/services/uploads/logo_processor.rb`, `merchant_logos_controller.rb` (magic bytes, 1 MB, pixel cap, PNG re-encode, 5/h) | `services/uploads/logo_processor_spec.rb`, `merchant_spec.rb` | ✓ F-16 |
| SEC-17 | CSV formula-injection escape | `services/reports/transactions_csv.rb` (`'` prefix on `= + - @ TAB CR`, RFC 4180 quoting), `controllers/api/v1/reports_controller.rb` (`finance` only, `no-store`, `report.exported` audit) | `spec/services/reports/reports_spec.rb` (each trigger character, quoting, a `=HYPERLINK` description), `spec/requests/reports_spec.rb` (viewer 403, analyst export, audit row), `web/e2e/w5-ledger.spec.ts` W5.4 (downloaded file) | ✓ F-14 |
| SEC-18 | Security headers | `config/initializers/security_headers.rb`, web `proxy.ts` CSP | `errors_spec.rb` (api headers) | ⚠ CSP Profile N (nonce + `'strict-dynamic'`): 0 CSP violations in 150/150 cells (`screens[].checks.console`) and cookies pass (`site.cookies`). HSTS is absent on localhost (ADR-048, `site.headers`). |
| SEC-19 | Suspended merchant → 403 | `concerns/authentication.rb` (`MERCHANT_SUSPENDED`, auth and ops exempt), `api/v1/ops/merchants_controller.rb` (suspend with reason under row lock, audited on the merchant chain; ADR-042) | `api_key_auth_spec.rb`, `requests/ops_spec.rb` (S and K → 403 while suspended, session read 200, reactivate restores; non-operator and K → 404 on every ops route), `e2e/w10-ops.spec.ts` | ✓ F-18 |
| SEC-20 | `RATE_LIMIT_SCALE` refused in production | `config/initializers/00_env_schema.rb` | `spec/lib/env_schema_spec.rb` | ✓ F-01 |

## §4.2 RBAC matrix

| Capability | Enforcement | Evidence | Status |
|---|---|---|---|
| View | `authorize!(:view)` | `members_spec.rb` plus each feature's resource spec (viewer read, write 403) | ✓ |
| API keys, webhook endpoints, deliveries | `authorize!(:developer_settings)`; a key cannot manage keys | `api_keys_spec.rb` (analyst 403, bearer 403), `webhook_endpoints_spec.rb` (viewer/analyst 403) | ✓ F-05, F-12 |
| Team: invite, change role, remove (admins cannot touch owners) | `authorize!(:team)` + `TeamPolicy` | `team_policy_spec.rb`, `members_spec.rb`, `invitations_spec.rb` | ✓ F-04 |
| API key acts as developer (no refund) | `Authorization` maps `Current.api_key` → developer | `api_key_auth_spec.rb` | ✓ F-05 |
| Customers & payment methods (redact = owner/admin, session) | `authorize!(:write_payments)` / `(:refund)` | `customers_spec.rb` | ✓ F-06 |
| Finance: audit trail, governance evidence | `authorize!(:finance)` | `governance_spec.rb` (developer/viewer/key 403) | ✓ F-15 |
| Merchant settings & logo | `authorize!(:merchant_settings)` | `merchant_spec.rb` | ✓ F-16 |
| Processor settings | `GET` any role (session), `PATCH` `authorize!(:merchant_settings)` | `processor_accounts_spec.rb` (developer 403, API key refused, tenant/mode 404) | ✓ F-07 |
| Payment intents | `GET` `authorize!(:view)`; create/confirm/capture/cancel `authorize!(:write_payments)`; `scoped`/`find_scoped!` by merchant + mode | `payment_intents_spec.rb` (viewer 403, other merchant/mode isolated, Idempotency-Key required for API keys) | ✓ F-08 |
| Ledger accounts & journal entries | `authorize!(:finance)` + `session_only`; `scoped` by merchant + mode | `ledger_spec.rb` (developer 403, API key refused, other merchant/mode 404) | ✓ F-09 |
| Refunds | `GET` `authorize!(:view)`; `POST` `authorize!(:refund)` (owner/admin) or an API key (ADR-036), Idempotency-Key required for API keys; `scoped`/`find_scoped!` | `refunds_spec.rb` (developer/viewer 403, API key allowed, replay, other merchant/mode 404) | ✓ F-10 |
| Event log | `authorize!(:view)` (any role) or an API key; `scoped`/`find_scoped!` by merchant + mode; events with no matched merchant are never listed | `events_spec.rb` (other merchant/mode hidden, other tenant 404, unauthenticated 401) | ✓ F-11 |
| Webhook endpoints & deliveries | `session_only` + `authorize!(:developer_settings)` (owner/admin/developer); `scoped`/`find_scoped!` by merchant + mode, including the retry update | `webhook_endpoints_spec.rb` (viewer/analyst 403, API key refused, other tenant 404, live mode hidden from test) | ✓ F-12 |
| Disputes & evidence, sandbox | `GET` `authorize!(:view)`; evidence `authorize!(:refund)` + `session_only` + PAN guard; sandbox `authorize!(:sandbox)` + `session_only`, test mode only | `disputes_spec.rb` (developer evidence 403, live sandbox 403, PAN 422, other merchant 404) | ✓ F-10 |

## §4.3 Domain-signal checklists

| Signal | Status |
|---|---|
| has_email (outbox, retries 1m/10m/1h → dlq, no tracking pixels) | ✓ F-03 (`deliver_pending_emails_job_spec.rb`, `emails/deliver_spec.rb`, `transactional_mailer_spec.rb`) |
| has_dual_write (outbox rows in the domain transaction) | ✓ emails F-03; webhooks F-12 (`dispatcher_spec.rb`: rollback writes no event or delivery; `webhook_endpoints_spec.rb`: `Payments::Hooks.publish!` fans out in the transaction) |
| has_image_uploads (magic bytes, HEIC rejected, 1 MB, re-encode, EXIF strip, 5/h) | ✓ F-16 |
| has_payments, has_geo | ⚠ Reviewer N4 LOW residual: two idempotent-resume edges (stale-claim window vs Stripe read timeout), see SECURITY-AUDIT.md; ✓ F-08 (state machine, idempotency, integer fees; coarse NUMERIC centroids only, bounds CHECK); ✓ F-09 double-entry ledger (INV-30 immutable rows, INV-31 deferred balance trigger, `spec/services/ledger/post_spec.rb`); ✓ F-10 refunds bounded by the remainder under a row lock, settled once by compare-and-swap, disputes one per payment (INV-22..25, `spec/services/refunds/create_spec.rb`, `spec/services/disputes/lifecycle_spec.rb`) |
| has_webhooks (signature before write, dedupe, anomalies) | ✓ F-11 (INV-3 UNIQUE (source, event_id), INV-6; `spec/services/inbound_webhooks/receive_spec.rb`: replay is a duplicate, rollback then redelivery applies, 100/min burst, PII redaction) |
| has_webhook_send (signed, idempotent, retried, dead-lettered) | ⚠ F-12, Reviewer N3 LOW residual: one endpoint with 100+ due retries can fill a sweep (SECURITY-AUDIT.md) (INV-26..28; `dispatcher_spec.rb` backoff 1m/5m/30m/2h then dlq with audit, claim race, stale claim; `deliver_pending_webhooks_job_spec.rb` T1 sweep) |
| has_websocket (origin, per-connection budget, channel authz) | ✓ F-13 (SEC-13 row; W2.3 e2e watches the live feed from a second tab) |
| has_scheduled_work | ✓ F-17 (INV-4 UNIQUE (job_name, run_key) claim; T3–T6 effect specs `spec/jobs/*_spec.rb` (INV-40) incl. same-window no-op, failed-run record, row-lock re-check, retention windows; non-prod trigger refused unless `OPS_TRIGGER_ENABLED` and never in production, `spec/requests/ops/scheduled_runs_spec.rb`) |

## §4.4 PCI-DSS 4.0.1 baseline

| Item | Status |
|---|---|
| PCI-3 No PAN/SAD storage | ✓ F-06; INV-11 (no PAN/CVV columns) passes (tokens + brand/last4/exp only; PAN guard); payment intents carry token + brand/last4/exp only; the raw token is encrypted and never serialised (F-08) |
| PCI-4 TLS in transit | ⚠ SEC-1 (TLS 1.3 at the edge; HSTS on the production edge per ADR-048) |
| PCI-8.3.6 Password ≥12 (15 used) | ✓ F-02 |
| PCI-8.3.4 Lockout ≤10 attempts | ✓ F-02 |
| PCI-8.2.8 Idle timeout | ✓ F-02 (30 min, compensating control ADR-017) |
| PCI-10 Audit trails | ✓ SEC-15 (F-15 verification + evidence) |
| PCI-6.4 Public web app protection | ✓ rack-attack (INV-32), CSP Profile N with 0 violations, SPEC §7 validation on every write |
| AT-REST encryption | ✓ F-06 (`customer_encryption_spec.rb`: round-trip, fresh nonce, tamper rejection) |

## Web delivery (WEB.md §2, §3, §6)

Evidence comes from `FRONTEND-AUDIT.summary.json`, the definitive `--full` run on a quiet host (2026-10-21T19:19Z): 30 screens, 150 cells, 0 failing, gitSha c84c320, build 55758e51f211, production. The only failure is the HSTS header row, carried under ADR-048.

| WEB.md | Contract row | Evidence (summary field) | Status |
|---|---|---|---|
| §2 | WCAG 2.2 AA axe: 0 serious/critical per viewport and theme | `screens.*.failing = 0` over 150 cells (axe in both themes) | ✓ |
| §2 | Contrast in both themes | axe color-contrast within the same cells, dark at 375 and 1280 | ✓ |
| §2 | Skip link first, landmarks, one h1, `html[lang]`, viewport meta | `checks.skipLink` and `checks.structure` within the 0-failing cells | ✓ |
| §2 | Targets 24px, plus 44px for primary controls at 375 | `checks.targets44` within the 0-failing cells | ✓ |
| §2 | Visible focus, keyboard | `checks.focusVisible` within the 0-failing cells; e2e 26/26 (Desktop Chrome + Pixel 7) | ✓ |
| §2 | Reduced motion | `checks.reducedMotion` within the 0-failing cells | ✓ |
| §2 | Mobile nav reachable, toggle with `aria-expanded` and `aria-controls` | `checks.mobileNav` within the 0-failing cells | ✓ |
| §2 | Lighthouse accessibility ≥ 95 | `lighthouse[]`: landing 100, dashboard 100 | ✓ |
| §3 | Per-route title, description, canonical, OG/Twitter on public screens | `site.meta` pass (5 public screens) | ✓ |
| §3 | robots.txt and sitemap.xml | `site.robots` pass, `site.sitemap` pass | ✓ |
| §3 | Icon set and manifest | `site.icon` pass | ✓ |
| §3 | Designed 404 | `site.notFound` pass | ✓ |
| §3 | JSON-LD | `site.jsonLd` skip (not required for archetype `saas`; the landing page ships Organization and WebSite anyway) | ✓ |
| §3 | Lighthouse SEO on public screens | `lighthouse[]` landing 100. Dashboard 63 is a residual by contract (authenticated, `noindex`). | ✓ |
| §4 | Performance ≥ 85, LCP ≤ 2.5 s, CLS ≤ 0.1 | `lighthouse[]` landing 93 (92–95, LCP 1781 ms, CLS 0.002) and dashboard 93 (93–96, LCP 1555 ms, CLS 0), each the median of 3. RB-17, RB-22 and RB-24 cut blocking time. Runs on a contended host scored 83–84 (ADR-054). | ✓ |
| §4 | JS budget 400 KB (saas) | `bundle`: landing 139 KB, app screen 150 KB | ✓ |
| §6 | Header set present | `site.headers` fail: HSTS absent on localhost only, accepted under ADR-048 and carried by INV-35. The rest of the set is present. | ⚠ |
| §6 | CSP Profile N, 0 `securitypolicyviolation` events | `checks.console` within the 0-failing cells | ✓ |
| §6 | Session cookie `__Host-`, HttpOnly, SameSite, Secure | `site.cookies` pass | ✓ |

## Posture

Posture: 62 ✓ / 7 ⚠ / 0 ✗ (COMPLIANCE.md Gate, 2026-10-20). Each ⚠ is a recorded residual and none is a gap:
- **HSTS absent on the localhost edge** (SEC-1, SEC-18, PCI-4, WEB.md §6). ADR-048 accepts it; a production deploy adds the header at the edge.
- **LOW audit residuals:**
  - SA-4: bcrypt reads only the first 72 bytes (SEC-8).
  - N3: webhook sweep fairness (has_webhook_send).
  - N4: idempotent-resume edges (has_payments).
