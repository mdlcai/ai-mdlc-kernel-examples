Commit reviewed: 03a93c8

# Reviewer Gate — round 1

HEAD is ff7d921, which adds only a governance record on top of 03a93c8. This review is independent and has fresh context. It was read-only: no suites, builds or load test were run.

## Inputs

- RESEARCH.md, ARCHITECTURE.md (§9 in particular) and SPEC.md.
- The working tree at 03a93c8, including the `git show 03a93c8` diff.
- invariants.json and artifacts/reviewer/lint-summary.txt: 40 total; 39 machine (38 pass, 1 fail on INV-35 pending the wide close, 1 zero-files); 1 manual (INV-40).
- artifacts/web-baseline/FRONTEND-AUDIT.nolh.summary.json and FRONTEND-AUDIT.partial.summary.json. Only the summaries were read.
- docs/previews/*.png (dashboard@375 inspected) and the artifacts/web-baseline screenshots.
- The Verification Gate verdict:
  - RSpec: 332 examples, 0 failures, 96.17% line coverage.
  - rubocop: 0.
  - eslint and tsc: 0.
  - Vitest: 60/60, 53.4% statement coverage.
  - Playwright: 26/26.
  - smoke: 53/53.
  - npm audit: 0.
  - baseline --failing: 0.

## Checklist

1. **Genuine implementation — PASS (with caveat).**
   - Processors are a real `stripe` gem client with failover. `api/app/services/processors/stripe_adapter.rb:68-80,99-112` contains the refund call and error mapping, and `spec/services/processors/router_spec.rb:38` covers failover.
   - The ledger, outbox, idempotency and audit chain all work against real data.
   - Caveat: `api/app/services/risk/ip_country.rb` maps only RFC 5737 documentation ranges, so `country_mismatch` cannot fire on a real client IP. This is a declared limitation, and `amount_ceiling` and `velocity` work on real input.
2. **Tests prove behaviour — FAIL.**
   - The refund retry-after-503 contract is tested only at the service level. `spec/services/refunds/create_spec.rb:95-106` calls `Settle.call!` directly and bypasses the `Idempotent` concern, so it proves the implementation, not the HTTP behaviour that `refunds_controller.rb:4-5` promises, and that behaviour is broken (see item 6).
   - Required change: add a request spec that does `POST /api/v1/refunds` with key K, has the processor return `error`, retries with the same K, and asserts `Refund.count == 1` and that the refund settles.
3. **Architecture conformance — PASS (notes).**
   - The layering matches ARCHITECTURE: controllers call services, which write the outbox in the same transaction.
   - ADR-045 lock order is implemented: `api/app/services/ledger/post.rb:18` takes `Audit.lock_chain!` before any account row, and `audit.rb:30-37` provides it.
   - Path drift for the doc to reconcile (none of it affects behaviour):
     - `policies/policy.rb` is actually `concerns/authorization.rb` plus `policies/team_policy.rb`.
     - `api/lib/problem.rb` is actually `app/services/problem.rb`.
     - `services/audit/*` is actually `services/audit.rb`.
     - `components/ui/*` is actually `components/ui.tsx`.
     - The session and toast contexts are in `components/`.
4. **Scope — PASS.**
   - `api/config/initializers/rack_attack.rb` implements every SEC-7 throttle.
   - It adds `evidence-export/session` 10/min and `api/ip` 300/s beyond SPEC. Both are protective and minor.
   - The `loadtest` environment (`api/config/environments/loadtest.rb`, `compose.loadtest.yaml`) serves SPEC §9. EnvSchema refuses the dangerous flags in production (`api/lib/env_schema.rb:35,48-49`).
5. **Security — PASS.**
   - Authentication and CSRF:
     - CSRF uses an Origin allow-list plus an HMAC token on unsafe verbs (`concerns/authentication.rb:65-77`).
     - The cookie is `__Host-`, secure and httponly (`:100`).
     - Suspension is enforced (`:87-92`).
   - Inbound Stripe events are verified before `Receive` (`stripe_controller.rb:21`, INV-6).
   - The SSRF guard requires https (`security/ssrf_guard.rb:71`).
   - SEC-5: a foreign or missing payment answers 404 in refunds (`refunds/create.rb:13`) and sandbox disputes (`sandbox/disputes_controller.rb:18`).
   - SEC-1: Caddy and the API set no HSTS; `web/proxy.ts:32-33` sets it only behind `TRUSTED_CERT`.
   - No secrets are tracked: `.env` is ignored by `.gitignore:7`, and no key or credential files appear in `git ls-files`.
6. **Correctness of money paths — FAIL (major).** Retrying a refund after a 503 can create a second refund.
   - What happens:
     - `Refunds::Settle.call!` (`api/app/services/refunds/settle.rb:29-35`) answers 503 `SERVICE_UNAVAILABLE` on any non-decline outcome and leaves the refund `pending`. That outcome is `Result.error` for a Stripe timeout or connection error (`stripe_adapter.rb:110-111`) and also `processing`.
     - The Problem is rendered through `rescue_with_handler`, and `finish_idempotency` then calls `Idempotency::Store.release!` on any status of 500 or above (`api/app/controllers/concerns/idempotent.rb:50-53`; unhandled exceptions are released at `:40-45`).
     - `release!` **deletes** the claim row (`api/app/services/idempotency/store.rb:50-52`), and with it the `resource_id` that `attach!` stored (`:40-42`, called from `refunds/create.rb:26`).
     - A retry with the same key therefore gets a fresh `:claimed` claim, not `:resumed`. `RefundsController#create` (`refunds_controller.rb:25-34`) calls `Refunds::Create` again and opens a **second** pending refund under a new processor key `rails_<new re_id>`.
   - Impact:
     - For a partial refund with room left (`refunds/create.rb:18-23` holds pending refunds against the capture), a timeout that actually succeeded at Stripe plus the retry refunds the customer twice. This violates SPEC §5 "Error paths" and INV-2.
     - For a full refund, the retry answers 409 "nothing left to refund" instead of resuming.
     - The controller's own comment (`refunds_controller.rb:4-5`) states the opposite behaviour.
   - Same root cause in payment create: any exception after the processor call inside `Payments::Confirm#route_and_finish!` renders a 500, which releases the claim. The retry creates a new intent, and its new processor key leads to a second charge. `spec/requests/payment_intents_spec.rb:77-91` covers only a crash that left the claim in place.
   - Required change:
     - Never delete a claim that has a `resource_id`. Clear `locked_at` (or leave it to go stale), so that the same key resumes against the linked record. Only delete claims that have no attached resource.
     - Alternatively, `complete!` the 503 together with the resource link and have a resume path re-settle it.
     - Add request specs:
       - (a) refund 503, then retry with the same key: one Refund, and it settles;
       - (b) payment create with an error raised after authorize, then retry with the same key: one PaymentIntent and one processor idempotency key.
7. **Documentation — PASS (ADVISORY note).**
   - README, QUICKSTART, NOTICE and `.env.example` exist and match the commands.
   - `README.md:44` links `SECURITY-AUDIT.md`, which does not exist in the tree. Add it, or drop the link.
   - The stale comment at `payment_intents_controller.rb:127-141` ("arrive with F-09/F-10 … lists are empty") should be removed now that F-09 and F-10 have shipped.
8. **Manual invariants — PASS.**
   - **INV-40 — PASS.** Every scheduled trigger T1–T6 has an effect spec that asserts business effects, not just enqueueing:
     - **T1** `spec/jobs/deliver_pending_webhooks_job_spec.rb:24-35` checks deliveries, status transitions, the untouched future retry and the reclaimed stale claim. Tenant isolation is checked at `:37-52`.
     - **T2** `spec/jobs/deliver_pending_emails_job_spec.rb:12-26` checks sent counts and actual mail deliveries.
     - **T3** `spec/jobs/ledger_integrity_job_spec.rb:16-30` checks counts, the `ledger.imbalance_detected` audit and the dlq severity on drift. The failed run is checked at `:32-36`.
     - **T4** `spec/jobs/expire_stale_intents_job_spec.rb:14-35` checks that intents are canceled with reason `expired`, plus the audit events.
     - **T5** `spec/jobs/retention_sweep_job_spec.rb:20-39` checks per-table deletion counts.
     - **T6** `spec/jobs/rescreen_passwords_job_spec.rb:18-34` checks the clean/breached/unavailable counts, revoked sessions and the queued `breach_notice`.
9. **Reachability of admitted states — FAIL.**
   - Writers found:
     - **PaymentIntent** (`payments/state_machine.rb:6-16`):
       - requires_payment_method and requires_confirmation: `payments/create.rb:19-21`;
       - processing, failed, succeeded and requires_capture: `payments/confirm.rb`;
       - canceled: `payments/cancel.rb` and T4.
     - **Refund:** pending at `refunds/settle.rb:16`, succeeded at `:44`, failed at `:62`.
     - **Dispute:** needs_response, under_review, won and lost in `disputes/lifecycle.rb` (`:242,:257,:271`), driven by `sandbox/disputes_controller.rb` and the Stripe events.
     - **pending_webhooks:** delivering at `outbound_webhooks/dispatcher.rb:45`, delivered/retrying/dlq at `:94,:100`.
     - **pending_emails:** sending at `emails/deliver.rb:33`, retrying/dlq at `:41,:46`. Each template has a writer: password_reset `passwords/reset.rb:13`, welcome `signup.rb:24`, invitation `team/invite.rb:21`, breach_notice T6.
     - **Account flags:** user lock at `passwords/login.rb:38`, merchant suspended at `ops/merchants_controller.rb:30-43`, webhook anomaly at `inbound_webhooks/receive.rb:81-85`.
     - **Configuration:** risk rules at `risk/check.rb:32-37`, `TRUSTED_CERT` read at `web/proxy.ts:32-33`.
   - No writer — flagged:
     - (a) The `journal_entries.kind = 'reversal'` and `source_type = 'JournalEntry'` values are admitted by `api/db/migrations/20261018000900_f09_ledger.rb:47-48` and `app/models/journal_entry.rb:7`, but no production code writes them.
     - (b) The `webhook_events.source = 'sandbox'` value is admitted by `api/db/migrations/20261018001100_f11_webhook_events.rb:25` and `app/models/webhook_event.rb:8`. `InboundWebhooks::Receive` HANDLERS is only `stripe` (`receive.rb:50`), and the only writer is a retention spec.
     - (c) The env flags `SSRF_ALLOW_HOSTS` and `ALLOW_HTTP_WEBHOOKS` are documented in SPEC §8 and `.env.example:63-64`. The first is never read. The second is only validated (`api/lib/env_schema.rb:31,35`); `ssrf_guard.rb:71` always requires https.
   - Required change: for each of these, either remove it or wire it with a test. That means:
     - a new migration that narrows the CHECKs, plus the model enums;
     - removing the env vars from `.env.example`, EnvSchema and SPEC §8;
     - or, to keep any of them, implementing its writer or reader with a spec through that path.

## Baseline note

Both baseline summaries carry gitSha 164bbc2, not 03a93c8.

- The partial re-run clears the docs@375 target size and webhook-endpoint@375 overflow failures.
- The site headers, bundle (139/150 KB against a 400 KB budget) and Lighthouse were not re-measured at this commit.
- INV-35 stays open until the wide-close FRONTEND-AUDIT.json is produced. Not finding HSTS on the tls-internal origin is consistent with SEC-1.

## Narrative

The codebase is careful and largely well built.

- Money is BIGINT throughout.
- Ledger posting is balanced and enforced in Postgres.
- The outbox is written in the same transaction as the change.
- The 03a93c8 remediation (`Audit.lock_chain!` before ledger accounts) correctly implements ADR-045.
- SEC-5 404s and the scheduled-job effect specs are thorough.

The serious risk is in the idempotency layer. "Free the key on 5xx" was designed for requests that did nothing, but it is applied after a record has been created and linked. That turns the documented retry-resumes path for refunds into a duplicate-refund path, and leaves a narrower duplicate-charge window on payment create. Because the test for it bypasses the controller, 332 green examples did not catch it.

Secondary issues are dead declarations (the reversal journal kind, the sandbox webhook source, two unread env flags), path drift in ARCHITECTURE.md, the IP-country table limited to documentation ranges, and a README link to a missing SECURITY-AUDIT.md.

Follow-ups:

- Fix `release!` so it never discards a claim that has a linked resource, and add the two request specs.
- Prune or wire the unreachable values.
- Re-run the wide baseline at the final commit to close INV-35.
- Reconcile the ARCHITECTURE paths and the README link.

Verdict: FAIL
