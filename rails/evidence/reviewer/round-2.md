Commit reviewed: 4aa49b4
Previous round reviewed: 03a93c8
Inputs: artifacts/reviewer/round-1.md, git diff 03a93c8..4aa49b4, SPEC.md, ARCHITECTURE.md §9, DECISIONS.md, SECURITY-AUDIT.md
Mechanism: Task tool general-purpose subagent

# Reviewer Gate — round 2

The review was read-only. No suites, builds, docker or load test were run. The scope is the RB-3 diff and the code paths it touches. One throwaway Node script, kept in the session scratchpad and not in the repo, reproduced `safeNext` byte for byte to check N1.

## Round-1 items

| # | Item | R1 | R2 | Evidence |
|---|---|---|---|---|
| 2 | Tests prove behaviour | FAIL | **PASS** | `api/spec/requests/refunds_spec.rb:55` goes through the HTTP `Idempotent` concern. A stubbed `unavailable` gives a 503, then the same key is retried. The spec asserts `Refund.count == 1`, the same `re_` id and status `succeeded`, that `rails_<re_id>` was sent twice, and that `amount_refunded_minor == 2_500`. The sandbox adapter is the memoised `ADAPTERS` instance (`processors/registry.rb:9`), so the stub really bites. `api/spec/requests/payment_intents_spec.rb:95` raises once from `Payments::Confirm.apply!`, which runs after `Router.authorize`. It asserts a 500, then a 201 on retry with the same `pi_` and `PaymentIntent.count == 1`. It checks `authorize` was called twice with `key_prefix: rails_<pi>` and `first_attempt_no: 1`, and that the third call is replayed. `apply!` is a `module_function` singleton call from `route_and_finish!`, so `and_wrap_original` intercepts it. |
| 3 | Architecture conformance (path drift) | PASS (notes) | **PASS** | ARCHITECTURE.md §3 now names `concerns/authorization.rb` + `policies/team_policy.rb`, `app/services/problem.rb`, `app/services/audit.rb`, `components/ui.tsx`, and the context provider locations (diff hunks at ARCHITECTURE.md:102-128). The SSRF line matches ADR-050 (ARCHITECTURE.md:226). |
| 6 | Money-path correctness (retry after 5xx) | FAIL | **PASS** | `api/app/services/idempotency/store.rb:115-120`: `release!` now only unlocks (`locked_at = NULL`) a row whose committed `resource_id` is set. It deletes only rows with no resource. It reads the DB row, so an `attach!` that rolled back is handled correctly. `existing!` (`store.rb:122-136`) checks the fingerprint before anything else (422 on mismatch) and re-claims with a conditional UPDATE (`locked_at IS NULL OR < stale`), so two concurrent retries get one `:resumed` and one 409 `IDEMPOTENCY_IN_PROGRESS`. Claims are UNIQUE (merchant, mode, key), and the resume lookups go through `scoped(...)` (`refunds_controller.rb:174`, `payment_intents_controller.rb:32`), so a resource from another tenant or mode can never be resumed. Refund: `attach!` runs in the transaction that opens the refund (`refunds/create.rb:15-28`) and commits before `Settle.call!`, so the 503 leaves resource_id committed. The retry calls `Settle.call!(refund: prior)` with the same `rails_<re_id>` (`settle.rb:76-79`), and settling is a CAS out of `pending` (`settle.rb:95`). Payment create: `attach!` runs in the create transaction (`payments/create.rb:17-26`). The retry goes `resume_create` → `Confirm.call!(resume: true)` on a `processing` intent, using the same `key_prefix` and `first_attempt_no = attempts.count + 1`; the step-3 attempts rolled back with the error. Confirm: `attach!` before `Confirm.call!` (`payment_intents_controller.rb:47`) extends the same protection to `/confirm`. |
| 7 | Documentation | PASS (advisory) | **PASS (advisory still open)** | The stale F-09/F-10 comment is removed (`payment_intents_controller.rb` diff at :128). `README.md:48` still links `SECURITY-AUDIT.md`, which is **untracked** at 4aa49b4 (`git ls-tree -r 4aa49b4` has no such path; `git status` shows `?? SECURITY-AUDIT.md`). The working-tree copy also still holds the placeholder `RESCAN_BODY` under §3. Commit the file, or drop the link. |
| 9 | Reachability of admitted states | FAIL | **PASS** | `api/db/migrations/20261018001800_rb3_narrow_unreachable_values.rb:7-17` narrows the three CHECKs. The original table-level CHECKs (`20261018000900_f09_ledger.rb:47-48`, `20261018001100_f11_webhook_events.rb:25`) each reference one column, so Postgres named them `journal_entries_kind_check`, `journal_entries_source_type_check` and `webhook_events_source_check`, which are the names the migration drops. `down` restores them. The models follow (`journal_entry.rb:7-9`, `webhook_event.rb:8`), and so do the web vocabularies (`web/lib/ledger.ts:44`, `web/lib/events.ts:3,21-24`). SPEC §2 and §8 are annotated and the change is recorded as ADR-050. In `api/` and `web/`, grep finds no `reversal` outside the historical migrations, no `sandbox` webhook source (`events_controller.rb:21` validates against `WebhookEvent::SOURCES`), and no `ALLOW_HTTP_WEBHOOKS`/`SSRF_ALLOW_HOSTS`. The only other hits are in docs and the ignored local `.env`. `EnvSchema::PROD_FORBIDDEN_TRUE` dropped the flag (`env_schema.rb:34`). The specs that wrote `reversal`/`sandbox` were moved to valid values (`ledger/post_spec.rb:80,93`, `retention_sweep_job_spec.rb:16`). |

Items 1, 4, 5 and 8 passed in round 1 and the diff does not touch what they rest on. The new security changes are reviewed below.

## New findings

### N1 — HIGH — `safeNext` (SA-3 fix) now creates an open redirect (regression)

`web/app/(auth)/password-input.tsx:53-61` resolves `next` against `https://console.invalid` and returns `url.pathname + url.search + url.hash`. The WHATWG parser normalises dot segments, so a value that starts with a single `/` can come out with a pathname that starts with `//`. That pathname is protocol-relative, and the function returns it.

Reproduced with the exact function under Node 22:

| `next` | `safeNext(next)` | resolves to |
|---|---|---|
| `/..//evil.example` | `//evil.example` | `https://evil.example/` |
| `/.//evil.example` | `//evil.example` | `https://evil.example/` |
| `/%2e%2e//evil.example/x` | `//evil.example/x` | `https://evil.example/x` |
| `/x/..//evil.example/p` | `//evil.example/p` | `https://evil.example/p` |

- **Failure scenario:**
  1. An attacker sends `https://localhost:9443/sign-in?next=/..//evil.example/login`.
  2. The victim signs in.
  3. `sign-in-form.tsx:27` calls `router.replace(safeNext(next))` with `//evil.example/login`.
  4. The browser leaves the console origin for the attacker's page right after a genuine sign-in. That is exactly the SA-3 / SEC-2 outcome the change was meant to prevent.
- **Regression:** at 03a93c8 the raw `/..//evil.example` was returned unchanged. It stays same-origin, because it starts with `/.` and not `//`.
- **Tests:** `password-input.test.ts` does not cover dot segments. Its `/x/..\/evil.example` case is rejected only because of the backslash.
- **Required fix:** after resolving, reject results whose pathname starts with `//`, or return the original `next` once the origin check passes. Add the four cases above to the test.

### N2 — MEDIUM — SA-1: the per-email throttles still miss the email in the query string

`RateLimitKeys.body_field` (`api/config/initializers/rack_attack.rb:22-34`) reads only the JSON body or `req.POST`. The controllers read `params[:email]` (`auth/sessions_controller.rb:14`) and `params.require(:email)` (`auth/password_resets_controller.rb:12`). Rails `params` also merges the query string.

- **Failure scenario:**
  1. An attacker sends `POST /api/v1/auth/password_resets?email=victim@example.com` with an empty or `{}` JSON body.
  2. `body_field` returns nil, so `reset/email` (SEC-7: 3/h/email, `rack_attack.rb:60`) never applies.
  3. `Passwords::Reset.request!` has no cooldown of its own (`passwords/reset.rb:7-18`). Only `api/ip` (300/s) bounds it, so one IP can queue hundreds of reset emails per second to a victim.
- The same request shape skips `login/email` (5/min/email) on login. Only `login/ip` and the SEC-9 lockout remain.
- SECURITY-AUDIT.md SA-1 says the fix covers "whatever form the path or body takes", and `sessions_spec.rb:82` does not test a query-string email.
- **Required fix:** key on the same merged parameters the controller reads (query + body), and add a spec for `?email=` on both endpoints.

### N3 — LOW (advisory) — SA-2 slow-endpoint deferral still lets one endpoint fill the sweep batch

`deliver_pending_webhooks_job.rb:23-31` takes the 100 oldest due rows first and only then skips a slow endpoint's rows. Skipped rows keep their `next_attempt_at`, so if one tarpit endpoint has 100 or more due rows, every sweep selects the same batch:

- one 5 s attempt;
- 99 rows deferred, all for that endpoint.

Other tenants' retries never enter the batch until that backlog drains, at one row per minute. Fresh deliveries still go out through the per-row job (`outbound_webhooks/enqueue.rb:28`), so only retries starve. That contradicts SA-2's "cannot hold up other tenants". The fix is to select fairly per endpoint (for example `row_number() OVER (PARTITION BY webhook_endpoint_id) <= k`) or to leave slow endpoints out of the query.

The Dispatcher deadline is correct otherwise:

- `Timeout.timeout(DEADLINE, Failed, msg)` constructs `Failed.new(msg)`, which is valid because `code:` is optional (`dispatcher.rb:21-27`).
- The resulting failure takes the `fail!` retry path.
- The 5 s deadline is well under `CLAIM_TTL`.

### N4 — LOW (advisory) — resume edges that RB-3 widens and does not create

- **Stale window shorter than the processor call.** `Idempotency::Store::STALE_AFTER` is 60 s (`store.rb:73`), but the Stripe client keeps the gem defaults: 30 s open, 80 s read, plus network retries (`stripe_adapter.rb:120`). A retry during a slow first call can re-claim and run alongside it. Separately, `release!` and `complete!` do not check that the caller still owns the lock. The processor keys and the settle CAS prevent a double refund or charge, but the two requests can then answer the client inconsistently.
- **Failover resume re-runs the chain from attempt 1.** In `confirm.rb:59-64`, take a first request where account A errored with a transient error, account B succeeded under `rails_<pi>_2`, and the step-3 commit then failed. The retry calls A again under `_1`. If A now succeeds (an error before the request reached Stripe stores no idempotent result), the customer is charged twice. The same applies if the routing order changes between the two requests. This window existed with the stale-claim resume at 03a93c8. RB-3 now also routes every post-processor 5xx through it. Persist the attempts before step 3, or resume from the recorded processor.
- **Retrying `/confirm` after a later failure answers 409.** A `/confirm` that failed with a 5xx after step 3 had committed (`succeeded`) gets 409 `INVALID_STATE_TRANSITION` on retry instead of the intent. `create` handles this case in `resume_create` (`payment_intents_controller.rb:76-84`).

### Other SA changes: no defect found

- **SA-1 path normalisation** (`rack_attack.rb:17-19`): squeezing slashes, stripping a trailing slash and dropping a format suffix covers `/login/`, `//login`, `/login.json` and `/login.json/`. Public ids contain no dots, so the money and invite matchers are not affected.
- **SA-5** (`ssrf_guard.rb:13-19`): adds `2001::/32` (Teredo), `2002::/16`, `64:ff9b:1::/48` and `fec0::/10`. `2001:db8::/32` stays distinct. Each range has a spec literal.
- **SA-6** (`passwords/login.rb:20-23`): the locked branch pays for the dummy bcrypt compare before the generic 401. The spec asserts the compare runs.

### Web design diff: no regressions found

- No new raw colour values. The `#fff`/`rgba(...)` lines in `globals.css` are existing declarations that were only re-indented into a layer.
- No accessibility attributes were removed. The new `aria-describedby` targets (`card-token-hint`, `customer-search-hint`, `invite-role-hint`) exist in their pages, the `aria-label="Key metrics"` on the KPI `<dl>` is kept, and `role="search"`/`role="status"` stay in place.
- The JSX is well formed in the hunks reviewed.

## Narrative

RB-3 fixes the round-1 blocker properly:

- A key that has created a refund or intent survives a 5xx.
- Concurrent retries are serialised by the conditional re-claim.
- Fingerprint, tenant and mode isolation hold.
- The two new request specs exercise the real HTTP path.

ADR-050's removals are consistent across the migration, the models, the web vocabularies, EnvSchema, `.env.example` and SPEC.

The security fixes in the same batch are where the defects are:

- The SA-3 rewrite turns `/..//host` into `//host`, a new open redirect.
- The SA-1 fix still leaves the per-email throttles unenforced when the email arrives in the query string.

Verdict: FAIL

## Re-verify (9335abe)

Scope: `git diff 4aa49b4 9335abe -- web api`. Read-only: no suites, builds or docker were run. `safeNext` was checked with the scratchpad repro (`sn.mjs`), updated to the 9335abe code byte for byte.

| Finding | Result | Evidence |
|---|---|---|
| N1 `safeNext` open redirect | **PASS** | `web/app/(auth)/password-input.tsx:59` now refuses a parsed path that starts with `//`. Every input below returns `/dashboard`: the four round-2 inputs, plus `/\x00`, `/.%2e//evil.example`, `/%2e//evil.example` and `/%2E%2E//evil.example`. These come back unchanged and stay same-origin: `/%5c`, `/%5c/evil.example`, `/;//evil.example`, `/%2f%2fevil.example`, `/..%2f/evil.example`, `/@evil.example`, and `//` inside the query or hash. A further 1,508,597 generated inputs, built from `/ . %2e %2E %2f %2F %5c ; @ ? # .. : %00 %09 x evil.example` up to five pieces deep, gave **0** results that resolve off-origin or start with `//`. The output is already a normalised pathname, so parsing it again cannot change its origin. `password-input.test.ts:19-24` covers the dot-segment cases and keeps `/payments/../customers` → `/customers`. |
| N2 query-string email skips the per-email throttle | **FAIL** (partly fixed) | `api/config/initializers/rack_attack.rb:23-24` (`param_field`) falls back to `req.GET`, so `?email=` is now counted for login (`:64`) and reset (`:71`). The specs cover both: `sessions_spec.rb` (new `it` at the end) and `password_resets_spec.rb` (new `it` at the end). The body branch still decides "JSON" with `req.media_type == "application/json"` (`rack_attack.rb:31`), so a JSON body under another JSON content type still escapes (N5). |

Other N2 probes that do not escape:
- `email[]=…` and a JSON array are not counted (`string_field` returns nil), but `User.find_by_email` digests the stringified array, so they cannot reach a real account and no reset email is sent.
- A JSON body with `"email": null` plus a query email is counted by the query fallback. Rails gives the body precedence, so the controller sees `nil` and nothing is sent.
- A body email together with a different query email keys on the body. That is the same value Rails `params` gives the controller.
- With no Content-Type, Rack 3.2.7 and Rails both read the body as a form, so they agree.

### N5 — MEDIUM — the per-email throttles skip a JSON body sent under a JSON alias content type

- **Cause.** Rails 8.1 (actionpack 8.1.4, `action_dispatch/http/mime_types.rb`) registers `:json` as `application/json` with the synonyms `text/x-json` and `application/jsonrequest`, and its params parser decodes the body for all three. The API enforces no content type anywhere: no `415` and no `media_type` check outside `rack_attack.rb`.
- **What the throttle sees.** For `Content-Type: text/x-json`, `body_field` takes the `req.POST` branch (`rack_attack.rb:36`). Rack 3.2.7 does not parse that type as form data, so it gets `{}`, and `param_field` falls back to an empty `req.GET`. The key is nil and no throttle applies.
- **Failure scenario.**
  1. An attacker sends `POST /api/v1/auth/password_resets` with `Content-Type: text/x-json` and the body `{"email":"victim@example.com"}`.
  2. Rails decodes the body, and `Passwords::Reset.request!` queues a reset email on every request.
  3. Only `api/ip` (300/s) limits it. The same works on `/api/v1/auth/login` against `login/email`.
- **Required fix.** Decide "JSON" the way Rails does, for example `Mime::Type.lookup(req.media_type) == Mime[:json]`. Alternatively, read the field through `ActionDispatch::Request.new(req.env).request_parameters` (then rewind) so the throttle always sees what the controller sees. Add a spec that sends the email in a `text/x-json` body.

Verdict after re-verify: FAIL

## Re-verify 2 (d426eab)

Scope: `git diff 9335abe d426eab -- api`. Read-only: no suites, builds or docker were run. The Rails and Rack behaviour cited below is from the actionpack 8.1 / Rack 3.2 sources as I know them; the gems are not installed on this host, so I could not open the files.

| Finding | Result | Evidence |
|---|---|---|
| N5 JSON alias content type skips the per-email throttle | **PASS** | `api/config/initializers/rack_attack.rb:24-30`: `param_field` now reads `ActionDispatch::Request.new(req.env).params[field]`. That is the same parse the controller's `params[:email]` uses (`auth/sessions_controller.rb:14`, `auth/password_resets_controller.rb:12`). It covers the query string, every registered JSON alias (`application/json`, `text/x-json`, `application/jsonrequest`), form-urlencoded, multipart, and a request with no Content-Type. No custom `parameter_parsers` are registered (grep of `app config lib`), so no content type the controller accepts escapes the key. `throttle("login/email")` (`:49`) and `throttle("reset/email")` (`:56`) call it only after `req.post? && endpoint == …` holds, so no other request is parsed or affected. The new spec, `password_resets_spec.rb:81-90`, sends three `text/x-json` bodies, then expects the fourth request to answer 429. |

Behaviour checks:
- **Memoised parse after routing.** `params` is cached under `action_dispatch.request.parameters` while the path params are still empty. `Request#path_parameters=`, which the router calls, deletes that header, so the controller rebuilds the merged hash with the route params. The cached `GET`/`POST` hashes were built without a controller/action for `set_binary_encoding`. That only matters for `skip_parameter_encoding`, which nothing in the app uses.
- **Malformed JSON body.** `parse_formatted_parameters` raises `ActionDispatch::Http::Parameters::ParseError` before anything is memoised. `param_field` rescues it, and the request passes with no email key; the IP throttles evaluated before it still apply. The controller then re-parses, raises the same error, and gives the same 400 as before RB-5. Invalid UTF-8 and over-deep query or body params come back from Rails as `ActionController::BadRequest`, which is rescued and likewise re-raised by the controller.
- **What can still escape the rescue list.** Only Rack multipart parser errors (for example `Rack::Multipart::MultipartPartLimitError`) are not caught. In that case the exception propagates out of Rack::Attack, and `ActionDispatch::ShowExceptions` turns it into the same status the controller's own parse would produce. Rack::Attack is added with `config.middleware.use` at `rack_attack.rb:93`, which puts it after ShowExceptions in the stack. The request never reaches the action, so this is not a bypass. The 9335abe code (`req.POST`) behaved the same way.
- **Body re-read.** `raw_post` is memoised in `RAW_POST_DATA` and the stream is rewound, and the multipart form hash is memoised in the env, so the controller reads the same body.

New findings: none.

Verdict after re-verify 2: PASS
