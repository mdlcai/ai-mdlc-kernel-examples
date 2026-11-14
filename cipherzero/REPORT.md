# REPORT.md — CIPHΞRZER0 // TRUST

Project `c16fd425-f778-4585-87b3-7863e2aca9a1` · build_depth `comprehensive` · review_gates `auto`

## Build Input Reconciliation

Every non-blank input field gets exactly one disposition below. The three key-decision fields (`deployment`, `backend_language`, `auth_model`) are blank in RESEARCH.md; their defaults are recorded as ADR-012, ADR-013 and ADR-014 in DECISIONS.md.

| Field | Value | Disposition | Evidence / rationale |
|-------|-------|-------------|----------------------|
| `build_depth` | comprehensive | Applied | 12–20 features (BUILD-PLAN.md); SPEC.md expanded scope; threat model in ARCHITECTURE.md §7; SECURITY-AUDIT.md; load test |
| `review_gates` | auto | Applied | Stage 1 and 2 gates self-verify (DECISIONS.md ADR-015, ADR-016) |
| `domain` | Security | Applied | ARCHITECTURE.md §1; threat model §7 |
| `domain_signals.has_scheduled_work` | true | Applied | ARCHITECTURE.md §5.4 job catalogue; SPEC.md §5 F-12; invariants INV-11 (manual: trigger and effect tests) and INV-46 (job-claim UNIQUE) |
| `domain_signals.has_payments` | true | Applied | ARCHITECTURE.md §5.6 billing; SPEC.md §5 F-15; invariants INV-8 and INV-47 (no float money), INV-9 (idempotency UNIQUE), INV-10 (webhook dedupe), INV-15 (signature before DB) |
| `protocol_support` | HTTPS only | Applied | `compose.yaml` service `edge` (Caddy 2.11, `tls internal`), publishing `${EDGE_HTTPS_PORT:-443}:443`; `:80` is a redirect only (ARCHITECTURE.md §6.2). HSTS is gated on `HSTS_ENABLED` per reference/security-headers.md (ADR-017) |
| `monitoring` | Prometheus + Grafana | Applied | ARCHITECTURE.md §6.4; compose services `prometheus` and `grafana`; `/metrics` on api, trust-core and worker |
| `database_preference` | PostgreSQL | Applied | ARCHITECTURE.md §4 data layer; compose `postgres` (primary) and `postgres-replica` |
| `pii_handling` | encrypted at rest + in transit | Applied | ARCHITECTURE.md §5.2: column AES-256-GCM plus HMAC blind index; TLS at the edge; mTLS between api and trust-core. Volume encryption is ⚠ delegated (ADR-006) |
| `security_baseline` | HIPAA, PCI-DSS, SOC2, ISO 27001 | Applied | COMPLIANCE.md maps controls only, no certification claim (ADR-006); SPEC.md §4 |
| `rate_limiting` | true | Applied | ARCHITECTURE.md §6.3: Caddy edge plus per-IP and per-tenant Redis token buckets; INV-14 |
| `audit_logging` | true | Applied | EVIDENCE hash chain plus signed Merkle checkpoints (ARCHITECTURE.md §5.5); INV-13 |
| `frontend_framework` | Next.js | Applied | `apps/web`, Next.js 16.4.1 App Router |
| `state_management` | Zustand | Applied | `apps/web/src/stores/*` holds UI state (theme, drawer, filters) |
| `backend_framework` | Express | Applied | `apps/api`, Express 5.3.0 REST gateway |
| `api_style` | gRPC | Applied | Internal gRPC between api and trust-core over mTLS (`proto/trust.proto`); browsers use REST (ADR-001) |
| `orm_preference` | Prisma | Applied | `packages/core/prisma/schema.prisma`, Prisma 7.10.0 with adapter-pg (ADR-010) |
| `background_jobs` | BullMQ | Applied | `apps/worker`; BullMQ 6 job schedulers and a DLQ (ARCHITECTURE.md §5.4) |
| `performance_requirements` | load test 1k+ concurrent; availability 99.95% | Applied | `load/k6.js` at 1,000 VUs (SPEC.md §9). The availability SLO is backed by healthchecks, restart policies and the replica; a single host cannot demonstrate 99.95%, so it is documented as a target |
| `logging_format` | structured JSON | Applied | pino JSON logs carrying requestId, with redaction (ARCHITECTURE.md §6.4) |
| `scale` | large — 50k+ concurrent | Applied | Resolved tier **large**. Read replica, Redis cache, immutable static assets ready for a CDN, stateless api and worker replicas, BullMQ DLQ, edge rate limiting (ADR-007). Signal bump from `has_payments`: idempotency keys and row locks on subscription state |
| `multi_tenant` | true | Applied | `tenantId` on every tenant-owned row; scoped repositories; foreign ids return 404 (SPEC §7; cross-tenant tests in every feature) |
| `target_platforms` | web | Applied | Responsive web plus PWA (ADR-004) |
| `offline_capable` | true | Applied | No external runtime dependency; PWA shell with an offline page; crypto fails closed (ADR-004) |
| `design.archetype` | saas | Applied | Sidebar shell, dashboard led by one metric (SPEC.md §6) |
| `design.template` | DESIGN-TEMPLATE.html | Applied | Derived tokens (ADR-000); landing page ported |

Reconciled fields: **26 of 26**. Conflicts left unresolved: 0.

## Feature Build Log

Logs for each feature are in `artifacts/logs/F-<n>-*.log` (F-11: `f11-*`, F-17: `f17-*`).

| Feature | Wave | Tests added | Cumulative | Notes |
|---|---|---|---|---|
| F-1 Service floor & data layer | 0 | 22 | 22 | Schema + init migration, env/logger/problems/queue/metrics, health, Dockerfile |
| F-2 trust-core, crypto, policy & audit | 0 | 41 | 63 | gRPC over mTLS, AES-GCM + hybrid ML-KEM/ML-DSA, fail-closed PDP, hash chain + Merkle (ADR-032) |
| F-3 design system, shell, landing & PWA | 0 | 21 + 8 e2e | 84 | ADR-000 tokens, UI primitives, saas shell, landing port, nonce CSP, PWA |
| F-4 auth, sessions & chokepoints | 0 | 41 | 125 | Argon2id + breach check, TOTP, __Host- cookies, CSRF, rate limits, idempotency (ADR-037..041) |
| F-5 team & invitations | 1 | 17 + 4 e2e | 142 | Roles, last-owner lock, hashed czi_ invites, uniform 404 (ADR-044) |
| F-6 policy bundles & simulation | 1 | 9 + 4 e2e | 151 | Signed bundles, no self-approval, simulator (ADR-045, ADR-051) |
| F-7 workloads & agent grants | 1 | 11 + 2 e2e | 162 | SPIFFE identities, scoped grants behind step-up MFA (ADR-046) |
| F-8 key lifecycle | 1 | 13 + 5 e2e | 175 | Rotate, disable, two-person crypto-shred destroy (ADR-047) |
| F-9 vault encrypt/decrypt | 1 | 17 + 4 e2e | 192 | Envelope encryption, purpose-bound decrypt, plaintext scan (ADR-048) |
| F-10 evidence & verification | 1 | 12 + 4 e2e | 204 | Chain + Merkle + checkpoint verify, audited export (ADR-049) |
| F-15 billing (built early) | 1 | 22 + 3 e2e | 226 | Hosted checkout (SAQ-A), verified webhooks, idempotent refunds (ADR-043, ADR-050) |
| F-16 dashboard | 2 | 4 + 1 e2e | 230 | Replica-read summary, zero states, chart + table view (ADR-052) |
| F-12 scheduled operations | 2 | 12 + 1 e2e | 242 | Rotation/rewrap, expiry, checkpoints, retention, DLQ retry (ADR-053, ADR-054) |
| F-13 quantum inventory & migration | 2 | 10 + 1 e2e | 252 | Exposure scoring, hybrid successor keys (ADR-055, ADR-056) |
| F-14 APEX analysis & proposals | 2 | 10 + 1 e2e | 262 | Deterministic detectors, two-person remediation (ADR-057, ADR-058) |
| F-11 approvals inbox | 4 | 7 + 1 e2e | 269 | Separation of duties, step-up, guarded claim, expiry sweep (ADR-060) |
| F-17 delivery & observability | 5 | smoke 39/39 + fixtures 8/8, k6 | 269 | CI parity, Prometheus/Grafana, k6, docs, web-baseline config (ADR-061) |

### Wave checkpoints

Each line: full suite, typecheck, lint, invariants (pass/fail/zero-files), e2e.

- **Wave 0:** 125/125; typecheck and lint clean; invariants 27/21/1 (open items owned by later features).
- **Wave 1:** 226/226; clean; `npm audit --omit=dev` 0 vulnerabilities; invariants 38/10/1; e2e 60/68 → fixed (ADR-051) → 68/68.
- **Wave 2:** 262/262; clean; invariants 45/3/1; e2e 76/78 → fixed (ADR-059) → 78/78.
- **Wave 4:** 269/269; clean; invariants 47/1/1 (INV-2 waits on the web baseline); approvals e2e 2/2.
- **Wave 5:** 269/269; smoke 39/39; k6 smoke thresholds PASS (read p95 29.7 ms, key-op p95 92.1 ms, 0 errors). A `prettier --write .` touched the pinned kernel scripts: BLOCK GD-0018, reverted in 0db39b7 and recommitted with the scripts unchanged (ADR-061).

## Pre-Delivery Completeness Gate

Result: **PASS** (2026-10-14, after F-17).

- **Routes & handlers:** all 82 SPEC §3 endpoints have handlers (scripted mount+route match; `GET /metrics` is the internal listener in `packages/core/src/metrics.ts`). No stubs: the only `not implemented` match is the test double `apps/api/test/helpers/app.ts`.
- **Data layer:** `prisma migrate diff --from-config-datasource --to-schema` → "No difference detected" (exit 0). The single migration `20261015000000_init` has `down.sql`. Roles and plans are seeded by `npm run seed -w @cz/core`.
- **Validation, CRUD, auth, UI:** zod on every route body/query (§7 envelope with field issues). Register, login, logout, sessions and MFA are covered by e2e. INV-18 `ui-coverage` PASS (files=128): every §6 screen, non-internal endpoint and W1–W6 workflow is reachable in the UI and Playwright.
- **Environment:** the `.env.example` diff against `process.env.*`/`env.*` references found one gap, `TRUST_PROTO_PATH` (an optional override), which is now documented. QUICKSTART.md is present. All imports resolve from the workspace manifests (typecheck and production build are clean).
- **Delivery artifacts:** BUILD-PLAN.md (17 manifests); README.md (commands plus attribution); `.github/workflows/ci.yml`; `.editorconfig` + `eslint.config.mjs` + `.prettierrc.json`; Dockerfile (multi-stage, non-root, tini, HEALTHCHECK, `COPY . .` in the build stage covers tsconfig.base.json, packages and lockfile); compose.yaml; invariants.json + pinned runner (`lint:invariants`); pinned `scripts/web-baseline.mjs` + `web-baseline.config.json` (24/24 §6 screens, real fixture ids); ARCHITECTURE §7 threat model (STRIDE) and alternatives considered.
- **Header buffers / ports:** FIXED: the web server (`next start` script and the Dockerfile `web` CMD) lacked `--max-http-header-size=32768`, and both now set it (the api already did). Caddy has `max_header_size 32KB`. Host-published ports are env-interpolated in compose.yaml (`PG_HOST_PORT`, `REDIS_HOST_PORT`, `EDGE_HTTP_PORT`, `EDGE_HTTPS_PORT`, …).

## Verification Gate

The gate runs in CI order on a clean `npm ci`. Every command and exit code is in `artifacts/logs/vg-summary.log`.

| Step | Command | Result |
|---|---|---|
| Install | `npm ci` | exit 0 (148 s) |
| Typecheck | `npm run typecheck` | exit 0 |
| Lint (ESLint 9, typescript-eslint, jsx-a11y, react-hooks) | `npm run lint` | exit 0, 0 warnings |
| Format | `npm run format:check` | exit 0. The generated `apps/web/next-env.d.ts` was added to `.prettierignore` because `next build` rewrites it |
| Tests + coverage | `npm run test:coverage` | 271/271; all files 88.41% statements / 91.48% lines (comprehensive floor 80%) |
| Production build | `npm run build` | exit 0 |
| E2E (desktop-chrome + mobile-pixel7, prod servers) | `npm run e2e:run` | 79/80 → fixed → **80/80**; 80/80 again after the baseline fixes |
| Web baseline | `npm run web:baseline` | see Web Delivery Baseline |
| Invariant lint | `npm run lint:invariants` | see Web Delivery Baseline (INV-2 waits on the definitive `--full` report) |

- **Fix: interactive jobs were starved by sweeps (ADR-062).**
  - Symptom: W3's manual rewrap was not claimed within 30 s.
  - Cause: about 1.8k test tenants made the per-tenant cron and startup sweeps queue FIFO ahead of it, with a worker concurrency of 4.
  - Fix: BullMQ priorities. Interactive jobs run at 1 and sweeps at 10, and the job schedulers set the sweep priority.
  - Test: `queue-priority.test.ts`.
- **CI parity:** `.github/workflows/ci.yml` runs the same sequence. That is install → secrets/PKI/migrate/seed (`SEED_DEMO=true`) → typecheck → lint → format → coverage → build → e2e → prod servers → `smoke.mjs --fixtures` → `web:baseline -- --full` → `lint:invariants`.

## Web Delivery Baseline

- **Setup:**
  - Runner `scripts/web-baseline.mjs` (pinned, verbatim) with `web-baseline.config.json`: 24 screens, 375/768/1280, both themes, `buildIdFile` freshness. Auth screens are `dataBound` and use real fixture ids written by `smoke.mjs --fixtures`.
  - Dev dependencies `@axe-core/playwright` 4.13.0 and `lighthouse` 13.5.0.
- **Remediation loop:**

  | Run | Mode | Failures | Groups |
  |---|---|---|---|
  | 1 | `--no-lighthouse` | 226 | 18 |
  | 2 | `--no-lighthouse` | 80 | 4 |
  | 3 | `--failing` | 39 | 2 |
  | 4 | `--failing` | **0** | 0 |

  Fixed by root cause in two batches (ADR-063, ADR-064):
  - light accent contrast;
  - 44px targets and top-bar flex;
  - mobile table header;
  - skip links on auth, offline and 404, plus the focused console skip link that covered the drawer toggle;
  - CLS on policies, team, quantum and invite (settle-gating, placeholder tiles, a server-resolved invite);
  - a page-widening sr-only label;
  - the violet text token;
  - site metadata, robots, sitemap and favicon;
  - config `landing`/`appScreen`/`public`/`dataBound`.

  VERSION 0.1.0-dev → 0.1.1-dev.

### Definitive `--full` runs (wide close)

| Run | Commit | Result | Notes |
|---|---|---|---|
| #1 | 59bbee6 tree | FAIL: Lighthouse performance landing 69, dashboard 49 (LCP 5.5 s) | The client waterfall (session, then summary) delayed LCP → server-read first paint (ADR-071) |
| #2 | wide-close tree | FAIL: perf 69/70, plus 18 screens showing "Internal Server Error" titles, vault CLS 0.168, dashboard CLS 0.109 | Regressions fixed in the same batch (ADR-071) |
| #3 | wide-close tree | perf landing 70 (LCP 2104 ms), dashboard 69 (LCP 2320 ms) only | All 120 cells pass every other check |
| #4 | + SEC-12 | perf landing 69 (LCP 2347), dashboard 71 (LCP 2191) only | |
| #5 | **c78df35 (definitive, committed report)** | perf landing 68, dashboard 71, both **accepted** (INV-2 `accepted`, ADR-071) | Landing LCP 2885 ms came straight after the e2e suite; an isolated re-measure gave 1662 ms |

- Residuals carried: "dashboard: data probes skipped (server-fetched)". The error path is covered by `e2e/dashboard.spec.ts` instead. The second residual is "lighthouse dashboard: SEO not scored (noindex by contract)".
- `npm run lint:invariants` after run #5, closing line verbatim: `49 total / 48 machine (48 pass, 0 fail, 0 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`. INV-2 prints `carried: landing/lighthouse accepted per ADR-071` and `carried: dashboard/lighthouse accepted per ADR-071`.

### E2E at the wide close
- **Full suite on a fresh DB:** `artifacts/logs/e2e-final.log`. 81/82; the one failure is policies W6.5 on mobile, a Playwright `browserContext.close: ENOENT` on its own trace file after every assertion had passed, and it passes on rerun.
- **Before the reset (`e2e-sec12.log`), 77/82.** Three causes:
  - **Spec defect.** On phones the open drawer covers both the menu toggle and the scrim's centre. The spec now taps the scrim where it shows, right of the drawer, and asserts the drawer closed.
  - **Environment, test-tenant pollution.** The local DB had about 3.6k tenants left by unit tests that run an in-process trust service with a random root. Each sweep is one queue job that loops over every tenant, so four sweeps held all four worker slots for minutes. Priority (ADR-062) orders only waiting jobs, so W3's manual rewrap (queued 08:39:02, started 08:40:22) and the billing refund webhook missed their 30 s windows. Audit-checkpoint also failed closed for the tenants whose keys were wrapped under a test root.
  - **Fix.** The local postgres and redis volumes were reset, then migrate, seed and fixtures were re-run. CI always starts from a fresh DB.
- **Advisory, recommended tooling:**
  - run the unit tests against an ephemeral database, or tear their tenants down;
  - split sweeps into per-tenant chunk jobs, or reserve a worker slot for interactive work, before tenant counts reach the thousands (ADR-071).
- **Commit provenance:** `bb9409e` (the wide-close batch) was committed from another Claude Code session, with the trailer of session 01SbM1LE…, while this session was verifying it. Its content is this session's wide-close work. The govern hook decided it on the next commit (GD-0023).

## Reviewer Gate

- **Mechanism:** Claude Code Task tool, general-purpose subagent with fresh context. Rounds 2+ are diff-scoped.
- **Rounds:** 3, which are two review rounds plus the settle check.
  - Round 1 (59bbee6): FAIL, items 1–10 → fix batch 32b23e1 (ADR-065..069).
  - Round 2 (32b23e1): FAIL, items 2, 6 and 9 → settle batch af52231 (ADR-070).
  - Round 3, the settle check (af52231): **PASS**.
- **Newest verdict: PASS.**
- **Caveats carried:**
  - Three secondary writers are not tested on their own; another writer of the same value is tested in each case (round 3 notes).
  - The round-2 billing re-run (12/12) was a manual note.
  - Round diffs carry CRLF noise from `core.autocrlf`, so the reviewers used `--ignore-cr-at-eol`.
- **Narrative assessment (comprehensive):**
  - The codebase is consistently layered: routes → services → Prisma, with the trust-core boundary over mTLS. Invariants are machine-checked (48 rules) and every SPEC §2 state has a production writer and a test.
  - **Risk areas:** the per-tenant sweep design at high tenant counts (see E2E above), the deferred LOW audit residuals, and the client/server split of console data introduced at the wide close (ADR-071), which doubles the read paths that must stay consistent.
  - **Recommended follow-ups:** chunked sweeps, an ephemeral test database, the CI security-job ignore files, and the SEC-2/SEC-4 contract changes.

### Round 1 (verbatim, artifacts/reviewer/round-1.md)

#### Reviewer Gate — round 1
Commit reviewed: 59bbee66f24280418cfbf0f10865bed24b59956c
Mechanism: Claude Code Task tool, general-purpose subagent (fresh context)
Inputs: RESEARCH.md, ARCHITECTURE.md, SPEC.md, working tree, invariants.json + lint summary, FRONTEND-AUDIT.nolh.summary.json, docs/previews/, artifacts/web-baseline/*.png, Verification Gate verdict, .mdlc/BUILD.md §Reviewer Gate
Verdict: FAIL

Note: the working tree shows many `M` files, but `git diff --stat` shows content changes only in BUILD-STATE.md and smoke-test.log. The other flags are line-ending noise, so the reviewed code is HEAD.

##### Checklist
| # | Item | Verdict | Rationale (file:line) |
|---|---|---|---|
| 1 | RESEARCH requirements implemented genuinely | FAIL | Workflow E / W5.5 ("rotate assets to the hybrid profile, rewrap") is not implemented: `migrateAssets` only creates a `<name>-pq` successor key (apps/api/src/services/quantum.ts:362-430). Existing objects are never rewrapped and the source key is never retired. A plan still reaches `validated` (quantum.ts:556-577) with nothing moved, including a plan whose only vulnerable asset is the transport, which is marked `operator_action_required` and counted as done (quantum.ts:385-392). The other workflows are genuine: the PDP evaluates arbitrary documents (packages/core/src/policy/index.ts:78-113) and the APEX detectors are generic threshold/ratio logic over persisted records (apps/worker/src/apex/detectors.ts). |
| 2 | Every SPEC feature has code and behavioural tests | PASS | Every F-1..F-17 has route, service and UI code, plus supertest/vitest behaviour tests (e.g. apps/api/test/vault.test.ts:424, approvals.test.ts:97-145, apps/worker/test/billing-webhook.test.ts:64-139) and e2e W1–W6 (e2e/w1-protect.spec.ts … w6-apex.spec.ts). Test-path gaps on specific states are scored under item 9. |
| 3 | Matches ARCHITECTURE | PASS | Planes, modules and frameworks match §2.1 (apps/api, apps/trust-core, apps/worker, apps/web, packages/core). Minor variances only: agent grant scope is enforced in the api (apps/api/src/services/vault.ts:100-112), not in trust-core as §2.3 step 3 says; the "tenant-scoped repos" are inline `tenantId` where-clauses, not a repo layer (packages/core/src/db/index.ts); `crypto_operations` is written by trust-core in a separate transaction from the object row (apps/trust-core/src/service.ts:316 vs apps/api/src/services/vault.ts:277-314); the root package pins ESLint 9.39.5, not the 10.12.0 in §8. |
| 4 | No invented features | PASS | Every route in apps/api/src/routes/*.ts maps to a SPEC §3 row, and every page.tsx maps to a SPEC §6 screen. Extra columns (`users.deactivated_at`, `invites.revoked_at`) only serve SPEC features. |
| 5 | Security invariants | FAIL | Agent workloads can bypass grants entirely. `issueCredential` issues a long-lived `czw_` token to any workload, including kind `agent` (apps/api/src/services/workloads.ts:252-272). `bearerPrincipal` maps it to `type:'workload'` (apps/api/src/middleware/authenticate.ts:79-92). `grantScopeViolation` only checks `p.type==='agent'` (apps/api/src/services/vault.ts:106), and the default bundle rule allows workloads (packages/core/src/policy/index.ts:175-178). This breaks ARCHITECTURE §3 ("AI agents … act only through short-lived grants"). Everything else holds: limiter before argon2 (services/auth.ts:254 vs 274), `__Host-` cookies, CSRF (middleware/csrf.ts:27-31), nonce CSP with no `unsafe-inline`, HSTS gated, mTLS + SPIFFE (apps/trust-core/src/server.ts:29-33,67), redaction, no hardcoded secrets. |
| 6 | Cold read: no obvious bugs or broken wiring | FAIL | (a) Grant expiry never fires for used grants: authenticate sets `access_grants.claimed_at` on first use (authenticate.ts:99), and grant-expiry only claims `claimedAt: null` (apps/worker/src/jobs/grant-expiry.ts:17), so used grants stay `active` with no `grant.expire` audit or revocation publish. (b) Tenant-scoped job failures are swallowed (apps/worker/src/runner.ts:82, rethrow only for `global`), so the retry→DLQ path in ARCHITECTURE §5.4 is dead for 7 of 8 jobs. (c) `submitBundle` checks for another `pending_approval` bundle without reconciling rejected or expired approvals first (apps/api/src/services/policies.ts:329-330), so a new submit returns 409 until some GET runs `reconcileRejected`. (d) Evidence verify reads the head after streaming events (apps/api/src/services/evidence.ts:250-272), so concurrent appends can give a false `head_mismatch`. |
| 7 | Runnable | PASS | QUICKSTART.md §1–§3 commands exist and are wired: `dev:secrets`, `dev:infra`, `migrate:deploy`, `seed`, `e2e:run`, `web:baseline` (package.json scripts; packages/core/package.json). compose.yaml orders pki-init → migrate → trust-core → api×2 → worker → web → edge with health conditions. .github/workflows/ci.yml reproduces the Verification Gate sequence on Postgres/Redis services. The recorded gate is all exit 0, 271/271 tests and 80/80 e2e. |
| 8 | Manual invariants upheld | PASS | INV-11 PASS; see below. |
| 9 | Nothing declared is unreachable | FAIL | Approval kind `grant_high_privilege` is declared (apps/api/src/lib/executors.ts:9,17; approvals-client.tsx:65,72) but nothing creates it and no executor is registered. JobRun `failed` and the DLQ have no production-path test. ApexProposal `rejected` has no test at all. PolicyBundle `active`/`superseded`/`rejected` are reached only by calling the executor directly or faking the approval row. AccessGrant `expired` is reached only for never-used grants. Full table below. |

##### Item 8 — manual invariants
- **INV-11 "Every scheduled trigger has an effect test": PASS.** Each job has an effect test:

  | Job | Test | Trigger |
  |---|---|---|
  | key-rotation | apps/worker/test/jobs.test.ts:147 | cron |
  | rewrap | jobs.test.ts:180 | manual; asserts retired at zero dependents |
  | grant-expiry | jobs.test.ts:268 | startup |
  | audit-checkpoint | jobs.test.ts:299 | — |
  | apex-scan | apps/worker/test/apex-scan.test.ts:63 | — |
  | quantum-inventory | apps/worker/test/quantum-inventory.test.ts:70 | manual |
  | retention-purge | jobs.test.ts:322 | — |
  | approval-expiry | apps/worker/test/approval-expiry.test.ts:55 | cron |

  Each trigger kind (cron, manual, startup) is driven through `runJob` (jobs.test.ts:152, :186, :280), and smoke-test.log shows a manual rotation and its effect.
  - Caveat (not blocking): `catchUp()` and `upsertSchedulers()` in apps/worker/src/scheduler.ts are never called by a test; the startup test passes `trigger:'startup'` directly.
  - Caveat (not blocking): `rewrap` has no schedule and is not in catch-up (packages/core/src/queue.ts:23-31; scheduler.ts:30). A failed post-rotation enqueue therefore leaves objects on the old version until someone runs rewrap manually.

##### Item 9 — admitted states and their writers
| Entity | Status | Production writer | Test through that path | Verdict |
|---|---|---|---|---|
| Payment | succeeded / failed / refunded / disputed / charged_back; disputed→succeeded | apps/worker/src/jobs/billing-webhook.ts:82,85,87,89,92,93 | apps/worker/test/billing-webhook.test.ts:64,88,97,114,139 (real processor over stored webhook_events); route half at apps/api/test/billing.test.ts:54-87 | PASS (in simulated mode only Stripe can emit dispute events; the processor is still a real writer) |
| Subscription | pending / active / past_due / canceled; past_due→active | billing.ts (api):276, billing-webhook.ts:82,89,92,93, billing.ts:364 | api billing.test.ts:121,196; worker billing-webhook.test.ts:64,114,139 | PASS |
| PolicyBundle | draft | services/policies.ts:234 | policies.test.ts:94 | PASS |
| PolicyBundle | pending_approval | policies.ts:332-334 | policies.test.ts:136-140, e2e/policies.spec.ts:71 | PASS |
| PolicyBundle | active / superseded | trust-core/service.ts:543-547 via executePolicyActivation (policies.ts:423-468) | policies.test.ts:67-70,161-162 call `getExecutor('policy_activate')` directly, bypassing POST /approvals/:id/approve; no e2e or smoke test approves a policy_activate | FAIL (no test through the approve route) |
| PolicyBundle | rejected | policies.ts:147-173 (lazy `reconcileRejected`, on read only) | policies.test.ts:292-303 sets the approval to `rejected` with a direct DB update | FAIL (test fakes the transition) |
| Key | active / disabled / destroyed | keys.ts:204,312,355,404 | keys.test.ts:73,229,261; approvals.test.ts:97 via /approve | PASS |
| KeyVersion | active / deactivated / retired / destroyed | trust-core service.ts:282,399; worker rewrap.ts:101; service.ts:450 | keys.test.ts:73; jobs.test.ts:147,180,215; trust-core service.test.ts:208; e2e w3-rotation.spec.ts:36 | PASS |
| Workload | active ⇄ suspended | workloads.ts:228 | workloads.test.ts:216; e2e w4-agent.spec.ts:140 | PASS |
| AccessGrant | active / revoked | workloads.ts:392,451 | workloads.test.ts:260; e2e w4-agent.spec.ts:60,135 | PASS |
| AccessGrant | expired | grant-expiry.ts:23 | jobs.test.ts:268 seeds only `claimedAt:null` grants; a grant that was used can never reach it | FAIL (see item 6a) |
| ApprovalRequest | pending / approved / executed / rejected / expired | lib/approvals.ts; services/approvals.ts:148,191,216,110; approval-expiry.ts:22 | approvals.test.ts:97,147,163,173; approval-expiry.test.ts:55 | PASS |
| ApprovalRequest kind | grant_high_privilege | none: no creator and no `registerExecutor`; approving would return 409 at approvals.ts:136 | none | FAIL (unreachable) |
| ApexFinding | open / dismissed / resolved | worker apex/scan.ts:~110,168; api services/apex.ts:316 | apex-scan.test.ts:63,90; apex.test.ts:126; smoke.mjs:592-596 | PASS |
| ApexProposal | proposed / pending_approval / executed / validated | apex.ts:378,481,578; scan.ts:163 | apex.test.ts:143,174; apex-scan.test.ts:90; smoke.mjs:589-592 | PASS (e2e W6 stops at pending_approval) |
| ApexProposal | rejected | apex.ts:135 (on read), scan.ts:206 | none | FAIL (no test) |
| MigrationPlan | draft / testing / migrating / validated | quantum.ts:~480,510,531,577 | quantum.test.ts:121; e2e w5-quantum.spec.ts:16 | PASS for reachability (the semantics of `migrating` fail item 1) |
| JobRun | running / succeeded | runner.ts:47,65 | jobs.test.ts:162 | PASS |
| JobRun | failed (+ DLQ) | runner.ts:76-82; scheduler.ts:51-57 | ops.test.ts:77 inserts a `failed` row directly; ops.test.ts:24-37 adds to the DLQ directly; tenant failures never rethrow (runner.ts:82) | FAIL |

##### Focus items
- **GD-0007 F-6 policy bundles & simulation:**
  - The PDP is a real fail-closed evaluator: digest plus both signatures (policy/index.ts:118-152), and POLICY_UNAVAILABLE when no verified bundle exists (trust-core service.ts:172-186).
  - The simulator takes any document.
  - Gaps: activation is never tested through the approve route; `rejected` is lazy and its test fakes the row; `submitBundle` does not reconcile first.
- **GD-0008 F-7 workloads & agent grants:**
  - TTL ≤ 900 s (routes/workloads.ts:50), issuer MFA (:127), and 401 on revoke are all correct.
  - FAIL: agent workloads can hold `czw_` credentials that bypass grant scope.
  - FAIL: used grants are never expired by the job.
- **GD-0009 F-8 key lifecycle:** create, rotate, disable/enable and destroy-via-approval are sound and tested through the route (approvals.test.ts:97-145). Minor: rotation's `next_rotation_at` uses `Date.now()` rather than the injected clock (trust-core service.ts:407).
- **GD-0010 F-9 vault:**
  - Fresh DEK per object, AAD bound, 404 for a foreign object before any key operation (vault.ts:330), purpose bound, `no-store`, INTEGRITY_FAILURE on tamper.
  - Issue: the agent `czw_` bypass (item 5).
  - Minor: `crypto_operations` is not written in the same transaction as the object row.
- **GD-0011 F-10 evidence:**
  - Head row lock, RFC 9162 Merkle, and verify recomputes the chain, checkpoints and hybrid signatures; tamper tests exist (evidence.test.ts:143-222).
  - Race: a concurrent append during verify can give a false `head_mismatch` (evidence.ts:250-272).
- **GD-0013 F-16 dashboard:**
  - Persisted data only, on `dbRead`; the 24h/7d range drives the buckets (services/dashboard.ts:17-91); tested (dashboard.test.ts:76-148).
  - Screenshots show honest zero states.
- **GD-0014 F-12 scheduled operations:**
  - All 8 jobs have effect tests (INV-11).
  - Defect: tenant job failures never retry or dead-letter (runner.ts:82).
  - Defect: rewrap is unscheduled.
  - Ops routes are owner/admin and tenant-scoped (ops.ts:176,194-221), but an owner or admin can trigger the global `retention-purge` job.
- **GD-0015 F-13 quantum:**
  - Inventory and usage-weighted exposure are genuine (quantum.ts:131-156).
  - The interop test is a real round-trip.
  - The `migrating` phase does not rewrap or retire anything; FAIL item 1.
- **GD-0016 F-14 APEX:**
  - 7 deterministic detectors over persisted records; scores 0–100.
  - INV-4 holds (apex imports, apex.ts:1-13).
  - Execution happens only via the `apex_remediation` approval executor (apex.ts:518).
  - Gap: `rejected` proposals are untested, and e2e W6 does not reach the W6.7 terminal.
- **GD-0017 F-11 approvals inbox:**
  - Not-requester (approvals.ts:96), role-per-kind (:98), MFA (:134) and the DB CHECK (migration.sql:643) are all enforced and tested.
  - The `grant_high_privilege` kind is unreachable.

##### FAIL items (verbatim, actionable)
- [1] The W5.5 quantum migration does not rotate or rewrap assets; a plan validates while every ciphertext stays on the classical key — apps/api/src/services/quantum.ts:362-430, 556-577 — At `migrating`, rewrap each source key's protected_objects onto the hybrid successor (trust-core `Rewrap`, or an enqueued `rewrap` targeting the successor) and disable or retire the source key. Block `migrating → validated` until zero objects remain on source keys and no step is `operator_action_required`, or exclude transport assets from plans. Assert this in apps/api/test/quantum.test.ts:121.
- [5] An agent-kind workload can be issued a long-lived `czw_` credential that skips grant TTL, key and purpose scope — apps/api/src/services/workloads.ts:252-272; apps/api/src/middleware/authenticate.ts:79-92; apps/api/src/services/vault.ts:106 — Refuse `czw_` issuance when `workload.kind==='agent'` (409), and also reject `czw_` bearers whose workload kind is agent in authenticate. Add a test that an agent cannot obtain a `czw_` credential or use one to encrypt or decrypt.
- [6] Used agent grants never transition to `expired`, because first use sets `claimed_at` and grant-expiry requires `claimedAt: null` — apps/api/src/middleware/authenticate.ts:99; apps/worker/src/jobs/grant-expiry.ts:17 — Record first or last use in a separate column (e.g. `first_used_at`), keep `claimed_at` for the job (or drop the `claimedAt:null` filter, since the guarded `status:'active'` update already serializes), and update the APEX unused-grant detector (detectors.ts:256). Add a test: issue → use token → advance clock → run grant-expiry → `expired` plus a `grant.expire` audit event.
- [6/9] Tenant-scoped job failures are recorded `failed` and then swallowed, so nothing retries or reaches `cz-dead-letter`; JobRun `failed` and the DLQ have no production-path test — apps/worker/src/runner.ts:76-82; apps/worker/src/scheduler.ts:51-57; apps/api/test/ops.test.ts:24-37,77 — Rethrow after recording when any tenant run failed, and make retries re-claim the window. Add a worker test where `def.run` throws, asserting the `failed` job_runs row and, via the worker, the DLQ entry.
- [6] `submitBundle` returns 409 after the previous bundle's approval was rejected or expired, until a GET reconciles it — apps/api/src/services/policies.ts:329-330, 147-173 — Write PolicyBundle `rejected` eagerly when a `policy_activate` approval is rejected (services/approvals.ts:211-230) or expired (approval-expiry job), or at minimum call `reconcileRejected` at the top of `submitBundle`.
- [6] Evidence verify can report a false `head_mismatch` under concurrent appends — apps/api/src/services/evidence.ts:250-272 — Read `audit_heads` first and bound the event scan to `seq <= head.seq`, or run both reads in one REPEATABLE READ transaction.
- [9] Approval kind `grant_high_privilege` is declared but nothing creates it or executes it — apps/api/src/lib/executors.ts:9,17; apps/web/src/app/(console)/approvals/approvals-client.tsx:65,72 — Remove it from APPROVAL_KINDS, DECIDER_ROLES and the UI label maps. Otherwise implement the creator in `issueGrant`, register an executor, and add a route-level test.
- [9] PolicyBundle `active`/`superseded` are never reached through `POST /v1/approvals/:id/approve` in any test — apps/api/test/policies.test.ts:67-70,161-162 — Add an api test where a second user with fresh MFA approves the `policy_activate` request through the route, then asserts the new bundle is `active` and the previous one `superseded`.
- [9] PolicyBundle `rejected` is tested only by faking the approval row — apps/api/test/policies.test.ts:292-303 — Drive it through `POST /v1/approvals/:id/reject` (and an expiry case) and assert the bundle is `rejected`.
- [9] ApexProposal `rejected` has no test — apps/api/src/services/apex.ts:135; apps/worker/src/apex/scan.ts:206 — Add a test: propose → submit → a second person rejects via `POST /v1/approvals/:id/reject` → GET finding or run apex-scan → assert the proposal is `rejected`. Add a twin case for an expired approval.

##### Narrative assessment
This is a well-structured, unusually disciplined codebase.
- **Separation of concerns is strong.** The crypto plane really is isolated: the root key exists only in trust-core and is deleted from `process.env` after load; mTLS uses a SPIFFE allow-list.
- **Correctness guards are consistent.** State changes use guarded `updateMany` with explicit from-states, audit appends are serialized by a row lock, and the error contract and tenancy 404s are uniform.
- **Delivery evidence is strong.** 271 unit tests, 80 e2e tests and all 120 web-baseline cells pass.

The defects are at the seams, not in the core primitives:
- An identity type the grant model didn't anticipate: an agent holding a workload credential.
- An overloaded column: `claimed_at` used both as the job claim and as a first-use marker.
- An error path nobody exercised: tenant job failures are swallowed.
- A workflow phase implemented as bookkeeping rather than effect: quantum `migrating`.
- Lazily reconciled states whose tests fake the transition.

The main risk areas are:
1. Agent privilege containment, which is the product's headline AI-safety claim.
2. Job reliability. Silent failure with no DLQ undermines rotation and rewrap guarantees, and rewrap has no schedule as a fallback.
3. The migration story in QUANTUM, which today overstates what a `validated` plan means.

Recommended follow-ups beyond the FAILs:
- Schedule `rewrap` and add it to catch-up.
- Test `catchUp()` directly.
- Drive e2e W6 through approval to `validated`.
- Use the injected clock in trust-core rotation.
- Consider per-IP-only login throttling in addition to IP+email.
- Restrict `retention-purge` manual runs to a platform operator rather than any tenant owner.

### Round 2 (verbatim, artifacts/reviewer/round-2.md)

#### Reviewer Gate — round 2
Commit reviewed: 32b23e16167f8ae37162304f2df1276224026a59
Previous round reviewed: 59bbee66f24280418cfbf0f10865bed24b59956c
Mechanism: Claude Code Task tool, general-purpose subagent (fresh context)
Inputs: RESEARCH.md, ARCHITECTURE.md, SPEC.md, diff 59bbee6..32b23e1, artifacts/reviewer/round-1.md (FAIL item list: 1, 5, 6, 9), invariants.json + lint summary, FRONTEND-AUDIT.nolh.summary.json, docs/previews/, artifacts/web-baseline/*.png, verification evidence, .mdlc/BUILD.md §Reviewer Gate
Verdict: FAIL

##### Checklist
| # | Item | Verdict | Rationale (file:line) |
|---|------|---------|-----------------------|
| 1 | RESEARCH requirements genuinely implemented | PASS | Workflow E "phased migration governed" is now real: `migrateAssets` re-seals every object of the source key under the hybrid successor via `moveObjectsToKey` (apps/api/src/services/quantum.ts:469-493, apps/api/src/services/vault.ts:430-508) and retires the source (quantum.ts:390-405); validation requires every step migrated (quantum.ts:609-610). Workflow D "agent acts only through short-lived scoped authorization" is enforced at issue (apps/api/src/services/workloads.ts:265-268) and at authentication (apps/api/src/middleware/authenticate.ts:83-90). Test reaches it through the HTTP path with a key/AAD-bound trust double (apps/api/test/quantum.test.ts:177-211). |
| 2 | Every SPEC feature has code and behavior tests | FAIL | The batch changed four SPEC-visible behaviors (mfa/setup body, agent credentials, W5.5 step labels, MFA settings form) but left the Playwright suite on the old contract; CI runs `npm run e2e:run` (.github/workflows/ci.yml:86), so the 80/80 e2e recorded at round 1 no longer reproduces. Specs broken by the batch: e2e/approvals.spec.ts:26 and e2e/keys.spec.ts:26 (POST /auth/mfa/setup without `currentPassword` → 400), e2e/auth.spec.ts:68, e2e/w4-agent.spec.ts:22, e2e/w6-apex.spec.ts:22 (click "Set up authenticator" without filling the new password field → client-side error, no secret), e2e/w4-agent.spec.ts:79-81 ("Issue credential" on an agent — button is now hidden, workload-detail-client.tsx:292), e2e/w5-quantum.spec.ts:75 (expects 'Hybrid successor key provisioned', label is now 'Migrated: data re-sealed, source key disabled', quantum-shared.ts:112). Regression on round-1-passed coverage. |
| 3 | Matches ARCHITECTURE | PASS | Diff stays inside documented layers (api services/routes/middleware, worker runner/scheduler/jobs, web console clients); quantum → vault service call is a service-to-service call within apps/api; ARCHITECTURE §5.7 amended consistently with the code (executors.ts:10). |
| 4 | No invented features | PASS | Additions are remediation of SPEC/ARCHITECTURE requirements (W5.5 rewrap, agent containment, SEC-3, job retry/DLQ per ARCHITECTURE §5.4) plus UI layout; CI security job and README screenshots are delivery artifacts, not product features. Removal of `grant_high_privilege` narrows to SPEC. |
| 5 | Security invariants | PASS | SEC-3: /auth/mfa/setup now requires and verifies `currentPassword` behind the login limiter (apps/api/src/services/auth.ts:353-359, routes/auth.ts:61-62; test auth.test.ts:206-208). SEC-6: SESSION_SECRET refuses the `.env.example` placeholder (packages/core/src/env.ts:44-53; test env.test.ts:44-46). Agent long-lived bearer refused at issue and at auth (workloads.ts:267, authenticate.ts:88; test workloads.test.ts:170-195). Vault migration runs every decrypt/encrypt through trust-core as the acting user with policy/purpose binding and zeroes plaintext (vault.ts:451-503). No secrets in the diff. |
| 6 | Cold read: no obvious bugs / broken wiring | FAIL | Production flows are coherent (policies/apex reconcile before submit/propose/dismiss, evidence snapshot read, runner failed→running takeover, grant-expiry for claimed grants, checkout order summary now renders fields getCheckout already returns, billing.ts:197-198). But the batch left a dead code path: `operator_action_required` (quantum.ts:431-438) can no longer be produced — `createPlan` now rejects `transport` assets (quantum.ts:43-46, 245-253) and every other asset carries a keyId (apps/worker/src/jobs/quantum-inventory.ts:91-97) — and if it ever were produced the plan could never validate because it is not in `COMPLETE` (quantum.ts:387, 610). Same defect as item 9; fix there. |
| 7 | Runnable | PASS | README/QUICKSTART quickstart (`npm run dev:secrets` → .env with random SESSION_SECRET, scripts/dev-secrets.mjs) still satisfies the new placeholder refusal; build exit 0, smoke 39/39 on this tree. (Recorded e2e reproduction is broken by stale specs — counted under item 2.) |
| 8 | Manual invariants upheld | PASS | INV-11 — see section below. |
| 9 | Nothing declared is unreachable | FAIL | `MigrationStep.action` (wire field in `plan.interopResult.migration`, rendered by quantum-shared.ts:111-118): `operator_action_required` has no reachable writer after the batch; `successor_created` had its meaning changed by the batch (now "data still on the source key", blocks validation) and no test reaches it; `failed`, `missing`, `already_target` have production writers (quantum.ts:427-429, 441-447, 444-447, 460-467, 495-498) but no test arrives at any of them. See section below. |

##### Item 8 — manual invariants
- **INV-11** (every scheduled trigger has an effect test) — **PASS**. Effect test per job: key-rotation cron → new active version (apps/worker/test/jobs.test.ts:147); rewrap manual → objects on new version, empty version retired (jobs.test.ts:180); grant-expiry startup → expired + revocation published (jobs.test.ts:268) and claimed grants (jobs.test.ts:297, new); audit-checkpoint → signed checkpoint verifies (jobs.test.ts:314); retention-purge (jobs.test.ts:337); apex-scan → findings from seeded anomalies (apps/worker/test/apex-scan.test.ts:63); quantum-inventory → assets mirror keys (apps/worker/test/quantum-inventory.test.ts:70); approval-expiry cron (apps/worker/test/approval-expiry.test.ts:55). Failure/retry/DLQ path of the trigger runner is new and tested (apps/worker/test/runner-failure.test.ts:44, :70). Smoke log reports 39/39 including W3 rotation.

##### Item 9 — admitted states touched by the diff and their writers/tests
| Admitted state / kind / field | Production writer | Test through that path | Verdict |
|---|---|---|---|
| ApprovalRequest kind `grant_high_privilege` | removed everywhere (executors.ts:10-18, approvals-client.tsx, ARCHITECTURE §5.7); `git grep` finds no residue | n/a | PASS (removed) |
| Key `active → disabled` (quantum retirement) | `retireSource` (quantum.ts:390-405) | quantum.test.ts:211 via POST /quantum/plans/:id/advance | PASS |
| ProtectedObject `key_id`/`rewrapped_at` cross-key move, audit `vault.migrate` | `moveObjectsToKey` (vault.ts:430-508) | quantum.test.ts:202-210 (objects open under successor via POST /vault/objects/:id/decrypt) | PASS |
| MigrationStep.action `migrated` | quantum.ts:485-493 | quantum.test.ts:199 | PASS |
| MigrationStep.action `successor_created` (partial move, re-run in `migrating`) | quantum.ts:474-484 (when a decrypt/encrypt fails) | **none** — no test makes a move fail, leaves the plan in `migrating`, then re-advances to `validated` | FAIL (batch-changed, untested) |
| MigrationStep.action `operator_action_required` | quantum.ts:431-438 — **unreachable** (transport rejected at plan creation; only transport has null keyId) | none | FAIL (unreachable) |
| MigrationStep.action `failed`, `missing`, `already_target` | quantum.ts:460-467 / 427-429, 441-443 / 444-447 | none | FAIL (untested) |
| JobRun `running → failed`, `failed → running` (retry takeover) | runner.ts:65-83, 104-111 | runner-failure.test.ts:44 via runJob | PASS |
| DLQ on exhausted retries | scheduler.ts:41-48 | runner-failure.test.ts:70 | PASS |
| AccessGrant `active → expired` for claimed grants | grant-expiry.ts:17-25 | jobs.test.ts:297 via runGrantExpiryForOrg | PASS |
| PolicyBundle `pending_approval → rejected` (reject + expiry) | policies.ts:147-173 (now also on submit, policies.ts:286) | policies.test.ts:309, :327 via /approvals/:id/reject and /approve | PASS |
| ApexProposal `pending_approval → rejected` | apex.ts:126-147 (on propose/dismiss) | apex.test.ts:214, :231 | PASS |
| Workload credential for `kind=agent` refused | workloads.ts:267, authenticate.ts:88 | workloads.test.ts:170 | PASS |
| SPEC POST /auth/mfa/setup `{currentPassword}` | routes/auth.ts:61, services/auth.ts:347-359 | auth.test.ts:206-208 | PASS |
| `Principal` workload `kind: 'agent'` (apps/api/src/lib/http.ts:12) | no longer written (authenticate.ts:99 always `'service'`) | n/a | Nit — narrow the union to `'service'` |
| Checkout `plan`/`amount` fields | billing.ts:197-198 | rendered checkout-client.tsx:144-159 | PASS |

##### Focus items (one line per GD item)
- GD-0007 F-6 policy bundles & simulation — PASS: rejected/expired requests reconcile on submit; activation and rejection tested through the approvals routes (policies.test.ts:294-342).
- GD-0008 F-7 workloads & agent grants — PASS on API (agent czw_ refused, claimed grants expire); e2e/w4-agent.spec.ts is stale (item 2).
- GD-0009 F-8 key lifecycle — PASS: quantum retirement uses the SPEC `active → disabled` edge with audit; e2e/keys.spec.ts:26 stale (item 2).
- GD-0010 F-9 vault — PASS: `moveObjectsToKey` re-seals via trust-core with AAD rebound to the new key, guarded update, audited.
- GD-0011 F-10 evidence — PASS: verify reads one REPEATABLE READ snapshot; concurrency test evidence.test.ts:214.
- GD-0013 F-16 dashboard — PASS: presentational changes only; web baseline 0 failures.
- GD-0014 F-12 scheduled jobs — PASS: tenant failures rethrown for BullMQ retry, failed-run takeover, DLQ on exhaustion, all tested.
- GD-0015 F-13 quantum — FAIL: data migration now genuine, but `MigrationStep.action` has one unreachable value and four untested values (item 9); e2e/w5-quantum.spec.ts:75 stale.
- GD-0016 F-14 APEX — PASS: stale proposals reconciled before propose/dismiss; tested (apex.test.ts:214-250); e2e/w6-apex.spec.ts:22 stale (item 2).
- GD-0017 F-11 approvals — PASS: kind set narrowed to SPEC; e2e/approvals.spec.ts:26 stale (item 2).

##### FAIL items (verbatim, actionable)
1. [2] Playwright specs not updated to the batch's contract changes; CI e2e step (ci.yml:86) fails on 32b23e1 — e2e/approvals.spec.ts:26, e2e/keys.spec.ts:26, e2e/auth.spec.ts:68, e2e/w4-agent.spec.ts:22, e2e/w6-apex.spec.ts:22, e2e/w4-agent.spec.ts:79-81, e2e/w5-quantum.spec.ts:75 — send `data: { currentPassword: <the registered password> }` on the API mfa/setup calls; in the UI helpers fill `#mfa input[name=currentPassword]` before clicking "Set up authenticator"; in w4-agent assert the agent detail page shows no "Issue credential" button and the "Agents never hold long-lived credentials" text, and move the czw_ reveal/revoke assertions to a `service` workload; in w5-quantum expect 'Migrated: data re-sealed, source key disabled'. Then run `npm run e2e:run` to green.
2. [9]/[6] `operator_action_required` is an admitted `MigrationStep.action` with no reachable writer, and would block validation if produced — apps/api/src/services/quantum.ts:79, :431-438; apps/web/src/app/(console)/quantum/quantum-shared.ts:115 — delete the value from the union and the UI label; replace the branch with `missing` (or `failed`) for an asset without a keyId.
3. [9] `successor_created` (partial move, batch-changed semantics) has no test — quantum.ts:474-484 — add a quantum.test.ts case whose trust double fails decrypt for one object on the first `testing → migrating` advance: assert the step is `successor_created` with remaining > 0, the source key still `active`, the next advance returns 409 and the plan stays `migrating`; then let decrypt succeed, advance again, assert `migrated` + `validated` + source `disabled`.
4. [9] `failed`, `missing`, `already_target` have writers but no test reaching them — quantum.ts:427-429, 441-443, 444-447, 460-467 — add tests through POST /quantum/plans/:id/advance: (a) an asset whose key is already on the target profile → `already_target` and plan validates; (b) a key destroyed (or asset removed) after plan creation → `missing` and the plan stays `migrating` (409); (c) a non-target key already named `<name>-pq` → `failed`.

##### Regressions introduced by the batch
- Playwright suite (7 call sites in 6 specs, FAIL item 1) broken by the mfa/setup body change, the agent-credential refusal and the W5.5 step-label change; not covered by the narrow re-verify (only touched vitest suites ran).
- `operator_action_required` turned from a reachable, validating step into an unreachable, validation-blocking one (FAIL item 2).
- Note (not a FAIL): FRONTEND-AUDIT.nolh.summary.json records `gitSha: 59bbee6…`, so the summary's own header does not prove it ran on 32b23e1; the definitive `--full` baseline should be stamped with the reviewed sha.

##### Narrative assessment (short)
The batch closes the substantive round-1 gaps well: W5.5 now moves real ciphertext under the hybrid key through trust-core and retires the source, agents are fenced to scoped grants at both issue and auth, MFA enrollment needs the password, the placeholder session secret fails closed, and the job runner now retries and dead-letters properly, all with tests that go through production paths. Two problems remain. The end-to-end suite was not updated to any of the contract changes, so CI is red on this commit. And the migration-step vocabulary was half-revised, leaving one dead value and four untested ones. Both are bounded fixes: update the tests, delete one enum value, add four quantum tests. Advisory follow-ups: (a) the successor lookup by name (`<name>-pq`, quantum.ts:450) moves data into any same-profile active key with that name, so link successors explicitly (e.g. a `successor_of` field); (b) `moveObjectsToKey` runs synchronously inside the advance request, so move it to a worker job for large keys; (c) narrow `Principal` workload `kind` to `'service'`.

### Round 3, settle check (verbatim, artifacts/reviewer/round-3.md)

Commit reviewed: af52231a351c7ff3a3c182542dbf23509a8a4517
Previous round reviewed: 32b23e16167f8ae37162304f2df1276224026a59
Type: settle check (round-2 surviving items 2, 6, 9 only)
Mechanism: Claude Code Task tool, general-purpose subagent (fresh context), read-only except this file

Inputs:
- artifacts/reviewer/round-2.md (item 2, 6, 9 definitions and FAIL items 1-4)
- `git diff --ignore-cr-at-eol 32b23e1..af52231` (e2e/*, apps/api/src/services/quantum.ts, apps/api/test/quantum.test.ts, apps/web/src/app/(console)/quantum/quantum-shared.ts)
- DECISIONS.md ADR-070 (the claims; the code was judged, not the claims)
- artifacts/logs/e2e-settle.log, artifacts/logs/smoke-settle.log, artifacts/logs/wb-run8.log (tail)
- Own run: `npx vitest run test/quantum.test.ts` in apps/api, 10/10 passed. The working-tree copies of quantum.ts, quantum.test.ts, vault.ts, quantum-shared.ts and the seven e2e specs below are byte-identical to af52231 (`git diff --stat af52231 -- …` is empty). HEAD e8666d6 differs from af52231 only in GOVERNANCE-DECISIONS.jsonl.

##### Item 2 — tests/e2e exercise the shipped behavior: PASS
Each round-2 call site is fixed to the new contract:
- API MFA setup sends the password: e2e/approvals.spec.ts:26-29 and e2e/keys.spec.ts:26-29 post `data: { currentPassword: GOOD_PASSWORD }`. GOOD_PASSWORD is the password registerViaUi registers (e2e/helpers/app.ts:3, :37).
- UI MFA setup fills the password field: e2e/auth.spec.ts:68, e2e/w4-agent.spec.ts:22 and e2e/w6-apex.spec.ts:22 fill `#mfa input[name=currentPassword]`. That input exists inside `section#mfa`, in the setup form (apps/web/src/app/(console)/settings/settings-client.tsx:236, :254-256).
- Agent credentials: e2e/w4-agent.spec.ts:79-81 now asserts that "Issue credential" has count 0 and that the "Agents never hold long-lived credentials" text is shown (workload-detail-client.tsx:306). The czw_ reveal-once checks moved to the service workload (w4-agent.spec.ts:149-156): the token format, "I have stored it" hides the token, and the 12-character prefix stays visible. The token is then used against the API.
- W5: e2e/w5-quantum.spec.ts:75 expects 'Migrated: data re-sealed, source key disabled', which matches quantum-shared.ts:112.
- Dashboard drawer: e2e/dashboard.spec.ts:39-47 opens the drawer when "Open navigation" is visible, asserts the Primary nav and closes it again. Both aria-labels exist in ConsoleShell.tsx:93 and :103.
- Run evidence: e2e-settle.log has every affected spec passing in both projects. That covers approvals, auth 2FA, dashboard, all keys specs, both w4-agent specs, w5-quantum and w6-apex (log lines 8-87). The total is 79/80. The only failure is e2e/billing.spec.ts:47 (desktop): the refund status was still SUCCEEDED after 30 s. billing.spec.ts was not changed by this batch, and the failure is unrelated to items 2/6/9.
- Caveats, neither a FAIL:
  - The "rerun billing.spec --repeat-each=2: 12/12 pass" line is a one-line manual note appended to the log, not raw Playwright output.
  - The working tree now holds uncommitted billing/route/e2e-helper changes made after af52231. So the e2e log cannot be tied byte-for-byte to af52231 for billing. The specs in scope here are identical to the commit.
  - Recommend that the gate's final full e2e run be stamped with the reviewed sha.

##### Item 6 — cold read, `operator_action_required` dead path: PASS
- The value is gone from the code. The `MigrationStep.action` union is `'migrated' | 'successor_created' | 'already_target' | 'policy_review_required' | 'failed' | 'missing'` (apps/api/src/services/quantum.ts:79).
- `git grep "operator_action_required\|Operator action required\|Hybrid successor key provisioned" af52231` finds no hits outside docs and logs.
- The former branch now records `failed` with "Asset is not bound to a key." (quantum.ts:457-460). `failed` is not in `COMPLETE` (quantum.ts:387), so such an asset correctly blocks validation instead of silently passing.
- Observation (not a FAIL): this branch is defensive only. Transport assets are refused at plan creation (quantum.ts:43-46), and every other inventory asset carries a keyId. The value it writes has other reachable writers that are tested, so no admitted state is dead.
- The new `policy_review_required` state (SEC-9) is coherent:
  - `keyScopeBlocker` (quantum.ts:411-431) checks the active bundle's `rules[].resources.keyIds` and active, unexpired grants' `keyIds`, and blocks unless the successor is named too.
  - It is not in `COMPLETE`, so the plan stays `migrating` (409) until the scope is fixed.
  - It runs before any successor is created, so no orphan key is left (quantum.ts:473-484).

##### Item 9 — nothing declared is unreachable: PASS
Each `MigrationStep.action` value below has a production writer, and a test in apps/api/test/quantum.test.ts reaches it through POST /api/v1/quantum/plans/:id/advance. All tests passed in my run (10/10).

| Value | Production writer | Test through /advance |
|---|---|---|
| `migrated` | quantum.ts:517-525 (after `retireSource`) | :204 (main flow); :283 (after a partial-move retry, source `disabled` :284, object on successor :285) |
| `successor_created` | quantum.ts:507-515 (`moved.remaining > 0`) | :240-286. The trust double fails decrypt for the source key on the testing→migrating advance (:265-270), giving `successor_created`. The source stays `active` and the object stays on the source key (:271-272). A second forced failure gives 409 with the plan still `migrating` (:275-278). The next advance gives `validated` (:280-285). This matches round-2 FAIL item 3 exactly. |
| `already_target` | quantum.ts:467-469 | :316-324. The key is already HYBRID_X25519_MLKEM768, the step is `already_target` with from = to = the key, and the plan then validates. |
| `policy_review_required` | quantum.ts:474-484 via `keyScopeBlocker` :411-431 | :346-408. A deny rule scoped to the source key gives the step plus a detail naming the rule (:388-392). The data does not move and the key stays active (:393-394). The next advance is 409 (:395). Once the rule also names the successor, the retry validates and the object moves (:400-407). |
| `failed` | quantum.ts:493-500 (successor name held by a non-target key), :528-531 (createKey error), :458-459 (defensive, see item 6) | :336-343. The `held-aes-pq` key is AES256_GCM, which gives `failed` with "held-aes-pq is taken". The source stays `active` and the next advance is 409. |
| `missing` | quantum.ts:463-465 (key destroyed), :453-455 (asset row gone) | :327-333. The key is destroyed after plan creation, giving `missing`; the next advance is 409 and the plan stays `migrating`. |

Notes (not FAILs):
- Some writers are not tested on their own; in each case another writer of the same value is tested:
  - the grant arm of `keyScopeBlocker` (:424-429);
  - the "asset left inventory" arm of `missing` (:453-455);
  - the createKey-error arm of `failed` (:528-531).
  Item 9 asks that each admitted state be reachable and tested through its path, and every value meets that.
- The SEC-9 test edits the active policy bundle directly in the DB to arrange the rule. That is an arrangement step; the state under test is still reached through /advance.

UI labels: STEP_LABEL (apps/web/src/app/(console)/quantum/quantum-shared.ts:111-118) has exactly six keys. They match the six union values one-to-one, and each is writable as the table shows. There is no `operator_action_required` key. Its one consumer, quantum-client.tsx:438, falls back to the raw action for unknown values.

Settle verdict: PASS

## Security Audit

- **Verdict: PASS WITH ADVISORY.** It ran 4 passes, and its narrative and findings ledger are in `SECURITY-AUDIT.md`, with the raw reports in `artifacts/security/`.
- **Banner:**
  - Pass 1 findings: 7 (M1 · L6).
  - Auto-remediated: 8 (SEC-1, SEC-3, SEC-6, SEC-8, SEC-9, SEC-10, SEC-11, SEC-12).
  - Pass 2 residual: CRITICAL 0 · HIGH 0 · MEDIUM 0 · LOW 5 (+ the SEC-6 compose residual).
  - There were no CRITICAL findings.
- **Deferred LOW tracking:**

  | ID | Finding | Reason deferred | Tracking |
  |---|---|---|---|
  | SEC-2 | Account existence is enumerable through 409/429 responses | Needs a SPEC §7 contract change | COMPLIANCE SEC-AUTH-PASSWORD |
  | SEC-4 | The AEAD AAD does not include the purpose | Needs an AAD version byte and a re-seal | COMPLIANCE SEC-DATA-INTEGRITY |
  | SEC-5 | System-scope dead letters are visible and retryable by any tenant admin | The handlers are idempotent, so the impact is timing only | COMPLIANCE SEC-OPS-PRIVILEGED |
  | SEC-7 | CI actions are pinned to tags, not SHAs | Supply-chain hardening for the CI owner | COMPLIANCE DELIVERY-CI |
  | SEC-13 | The web server's first-paint reads share the per-IP rate bucket | Needs an internal channel (API contract change); it degrades first paint only | COMPLIANCE SEC-RATELIMIT-LOCKOUT |
  | SEC-6 (residual) | Compose-only `change-me` defaults for Postgres, the replica and Grafana | Loopback only; `dev:secrets` generates real values | COMPLIANCE SEC-ENV |

- **Tooling:** Gitleaks 8.30.1, Semgrep 1.177.0, npm audit, Trivy 0.74.0, plus a manual headers/CSP/cookie review; every scan category had a tool. **Recommended:** add `.gitleaksignore` (fixture fingerprints) and semgrep excludes before the CI `security` job's first run.

## COMPLIANCE.md Gate

- **Posture: 76 ✓ / 7 ⚠ / 0 ✗** over 83 requirements, including the WEB.md §2/§3/§6 web-delivery rows from the definitive run (c78df35).
- The ⚠ rows are the six deferred LOW residuals above, plus WEB-PERF (the Lighthouse performance score carried by ADR-071).
- Under `review_gates: auto`, the posture is logged as ADR-072.
- **Re-entry R-1:** 79 ✓ / 8 ⚠ / 0 ✗ over 87 requirements. It adds the STΛRGΛZΞR observer rows (OBS-AUTHZ, OBS-EXPOSURE,
  OBS-PROVENANCE) and DEMO-READONLY, after Security Audit pass 4 passed (SEC-14 fixed, SEC-15 deferred LOW). ADR-081.

## Design Quality Gate

**Verdict: PASS with an accepted residual. Posture 9 ✓ / 15 ⚠ / 0 ✗ over 24 screens** (`artifacts/design/score-3.md`, ADR-073).

| Round | Posture | Follow-up |
| --- | --- | --- |
| score-1 | 1 ✓ / 22 ⚠ / 1 ✗ | Design batch P1–P9 (ADR-069) |
| score-2 | 6 ✓ / 9 ⚠ / 9 ✗ | Settle batch R1–R9 (ADR-070), wide-close fixes (ADR-071) |
| score-3 | 9 ✓ / 15 ⚠ / 0 ✗ | No ✗ and no regressions; the residual is accepted (ADR-073) |
| score-4 | 9 ✓ / 15 ⚠ / 0 ✗ | Re-entry R-1 (design v2, observer kind); no regressions; new residual R10/R11 (ADR-081) |

- **✓ screens:** login, register, offline, policy-bundle, workloads, workload-detail, apex, billing, checkout.
- **⚠ screens:** cosmetic only (R2 tables, R4 controls, R5 StatTiles, R7 tab fade, R8 borders, R9 landing media).
  The remediation for each, grouped by primitive, is in `score-3.md` and `DESIGN-NOTES.md` §6.
- **Lighthouse performance:** landing 68 and dashboard 71 are carried under ADR-071.
- The design record (resolved system, type and color rationale, archetype, template conformance) is in
  `DESIGN-NOTES.md`.

## Re-entry R-2 Review Round

R-2 covers Quantum Readiness (F-21–F-24: product modes, plan entitlements, the scanner ingest and the customer-side CLI).
The verdicts below are in kernel order.

1. **Reviewer: PASS after the settle check.**
   - Round 4 (`artifacts/reviewer/round-4.md`) FAILED. Fix batch a5f40ab followed.
   - Round 5 (`round-5.md`) FAILED items 3 and 5: an APEX `rotate_key` could rotate a customer key in readiness mode. Its
     focus items GD-0058 (C-3) and GD-0059 (C-4) passed.
   - The round cap applies, so this took the bounded path: one fix (0672dbc, ADR-087, 0.4.2), then a settle check scoped to
     items 3 and 5. That check PASSED (`round-5-settle.md`).
   - Settle-check observations, both fail-closed and neither blocking:
     - The executor result stores the code `CONFLICT`, not `409`.
     - An APEX proposal refused in readiness stays `pending_approval`.
   - Round 6 (`round-6.md`) was a fresh-context focus-item round over the 12 changes still in REVIEW (F-6–F-14, F-16,
     C-3, C-4). It passed 11 and failed F-6 on one item: an activation whose API audit step failed after trust-core had
     committed could never be retried. One fix followed (9abc2f8, ADR-088, 0.4.3). The settle check scoped to that item
     PASSED (`round-6-settle.md`), and `govern.mjs resolve-review` cleared all 12.
   - Round-6 settle observation, not blocking: if a later bundle is fully activated before an interrupted one is
     retried, the interrupted bundle's retry returns 409 and it never gets its own `policy.activate` event.
2. **Security Audit: PASS WITH ADVISORY** (SECURITY-AUDIT.md, Pass 5).
   - CRITICAL 0.
   - Fixed: the body-parser HIGH advisory, SEC-18 MEDIUM, and SEC-19, SEC-21, SEC-22, SEC-24, SEC-25 and SEC-26.
   - Deferred LOW: SEC-20 and SEC-23. SEC-27 is noted.
3. **COMPLIANCE: 88 ✓ / 8 ⚠ / 0 ✗** over 96 requirements.
   - R-2 adds MODE-READINESS, ENT-LIMITS, ENT-DOWNGRADE, SCAN-AUTHZ, SCAN-METADATA-ONLY, SCAN-INTEGRITY, QR-RISK-CBOM,
     QR-PLAN-EVIDENCE and SCAN-CLI, all ✓.
   - The web rows come from the definitive baseline at f3617a8.
4. **Design Quality: 8 ✓ / 16 ⚠ / 0 ✗** over 24 screens (`artifacts/design/score-5.md`).
   - No ✗, so the residual is accepted.
   - New ⚠ items:
     - billing: the Compare plans table is clipped at 375 px.
     - evidence: the badge-wrap borders break (R12 suggests `box-decoration-break: clone`).
     - quantum: a KPI card is orphaned, the risk badge is stacked, and the `cz-scan` command breaks mid-word.
     - invite: the captures show the invalid-token page.
   - The console version label still reads V0.1.0 (from package.json) while VERSION.md says 0.4.2.

**Smoke at a8a1722 (0.4.3): 48/48 pass** (S.1b, W1–W7, Q.1; `smoke-test.log`). It needed a temporary second worker
process: with about 3,800 test tenants in the dev database, each cron sweep runs longer than its interval, so all four
worker slots stay busy and interactive jobs wait (see Residuals). An earlier 48/48 run was at f402a9c.
- An earlier W3.4b/W6.2 timeout was caused by load from the test tenants the worker was sweeping. It passed on a quiet
  system.
- W7.6a now checks the blocking asset ids from ARCHITECTURE §5.8.

**Web baseline at a8a1722 (re-run after the round-6 fix), and before that at 38745f9:** `--full --concurrency 2`, with the worker paused. INV-2 is fresh. The landing and dashboard
Lighthouse scores are carried under ADR-071. At the default concurrency, a different pair of pages missed the 45 s
networkidle load timeout on each run.

## Re-entry R-3 Review Round

R-3 covers clientless external discovery (F-25–F-28: verified domains with a DNS TXT challenge, Certificate
Transparency host discovery, the isolated prober with its egress guard, and external findings in the R-2 inventory;
ADR-089). The verdicts below are in kernel order.

1. **Reviewer: PASS after round 2 and the settle check.**
   - Round 7 (`artifacts/reviewer/round-7.md`, on 393294e) FAILED item 5 and focus item F-25: the internet-facing
     prober mounted the whole PKI volume, including the CA key and every service's private key. Fix batch 0.5.1
     (80fc26b, ADR-094) gave each service its own `pki-<service>` volume.
   - Round 8 (`round-8.md`, on 80fc26b) PASSED, with one non-blocking note.
   - Security round 2 then raised a SEC-30 residual and SEC-37, SEC-38 and SEC-39, fixed in 0.5.2 (c0f949c, ADR-095):
     - a 20-minute run budget per tenant run;
     - two external-discovery slots per worker;
     - the prober exits on a listen error;
     - `bootstrap.mjs --export` refuses a directory that overlaps `--out`;
     - `PROBER_TIMEOUT_MS` is capped at 15 s.
   - The settle check, round 9 (`round-9.md`, on c0f949c), PASSED. Its doc corrections are in 0234872: the budget is
     per tenant run, and the SECURITY-AUDIT INFO count is 5.
   - Settle observation, not blocking: a domain whose probes are all cut by the budget still records an empty report
     and moves to the back of the rotation (ADR-095 Known gaps).
2. **Security Audit: PASS WITH ADVISORY** (SECURITY-AUDIT.md, Pass 6 and its round 2).
   - CRITICAL 0, HIGH 0.
   - Fixed: MEDIUM SEC-28, SEC-29 and SEC-30; LOW SEC-37; INFO SEC-38 and SEC-39.
   - Deferred LOW: SEC-31, SEC-32 and SEC-33. Noted INFO: SEC-34, SEC-35 and SEC-36.
   - Trivy crashed on the image scan, so the Dockerfile and compose changes were reviewed by hand.
   - Upgrades from 0.5.0 should rotate the internal CA, because the prober had read access to the CA key
     (ADR-094 Upgrade, CHANGELOG 0.5.2).
3. **COMPLIANCE: 94 ✓ / 9 ⚠ / 0 ✗** over 103 requirements.
   - R-3 adds EXT-VERIFIED-ONLY, EXT-DOMAIN-NAMES, EXT-PROBER-ISOLATION, EXT-EGRESS-GUARD, EXT-MTLS, EXT-SECRET-GUARD
     and EXT-RATE-AUDIT, all ✓.
   - The web rows now cite the definitive baseline at c6221ce. Its cell counts match f3617a8 exactly.
4. **Design Quality: 8 ✓ / 16 ⚠ / 0 ✗** over 24 screens (`artifacts/design/score-7.md`).
   - Round 1 (`score-6.md`) gave landing a ✗ for mobile LCP (2992 ms). It now measures 2066 ms, so it is back to ⚠.
   - Resolved: the duplicate Quantum "Scan reports" heading, and the old mid-word splits in the `cz-scan` command.
   - Still open:
     - the `cz-scan` command can still break after a hyphen (`-- / upload`); the fix is nowrap spans per token;
     - the Domains panel and the Origin column appear in no capture (no `?tab=domains`, empty scan fixture);
     - the carried ⚠ items from score-5.
   - Scorer note: the `invariants.json` Lighthouse carry is keyed on the whole `lighthouse` check, while ADR-071
     carries the performance score only.

**Smoke at b8b0f99 (0.5.2, after `docker compose up -d --build`): 48/48 pass** (`smoke-test.log`).
- The live mounts show the SEC-28 fix: the prober's `/pki` holds only `ca.crt`, `prober.crt` and `prober.key`.
- As at a8a1722, the run needed extra worker processes, three this time. The dev database's leftover test tenants keep
  every worker slot busy with cron sweeps. Their audit-checkpoint keys were wrapped under a test root key, so
  `audit-checkpoint` fails for them on every sweep (trust-core `IntegrityError`). With one worker, S.1b timed out;
  with two, W3.4b and W5.1 did. This is a property of the dev data set, not of 0.5.2 (see the grant-expiry gap in
  ADR-093).

**Web baseline at c6221ce:** `--full --concurrency 2`, with the worker paused and the `WEB_BASELINE_*` credentials
loaded. INV-2 is fresh. Landing and dashboard Lighthouse performance are carried under ADR-071. Landing measured 81,
73 and 75 on three runs of the same build, so the landing carry that 0.5.2 dropped was restored (2daba7c).

**Wide close at b8b0f99:** typecheck, lint and `lint:invariants` (59/59 machine) pass; vitest 733/733 in 60 files.
`format:check` shows only the two files that were already flagged (`policies.ts`, `DESIGN-TEMPLATE.v1.html`).

## Stage 4

**Deploy target:** local, single host with Docker Compose (ADR-012). There is no remote, so the reachability outcome
is C. All 13 services are healthy behind the Caddy edge at **https://localhost**, and plain HTTP answers 308 to
HTTPS. The edge uses Caddy's private `tls internal` CA.

### Startup verification

- `GET https://localhost/api/health` → 200 `{"status":"ok","version":"0.1.4",…}` in 0.07 s (budget 500 ms).
- Starting the API with `DATABASE_URL` unset exits 1 with `Invalid environment for api: - DATABASE_URL: …`.
- The landing page renders at `/`, and `/login` and `/register` render (smoke R.2; web baseline).

### Deploy fixes (ADR-074)

A clean `docker compose up` exposed a root-owned `pki` volume that pki-init could not write. pki-init now runs
`bootstrap.mjs --out /pki`, the Dockerfile creates `/pki` owned by `node`, and `bootstrap.mjs` fails if any
openssl output is missing.

### Smoke Test Results

`bash smoke-test.sh https://localhost` against the compose stack: **PASS**. 7 of 7 direct checks passed (R.1
health, R.2 landing, N.1 unauthenticated 401, N.2 invalid registration 400 with `errors[]`, N.3 wrong password
`401 INVALID_CREDENTIAL`, N.4 unsigned webhook rejected, N.5 forged session cookie 401). All 39 W1–W6 workflow
steps passed, including APEX deny-spike detection and quantum classification on activity generated live during the
run. Evidence: `smoke-test.log` and `artifacts/logs/s4-smoke-https.log`. On Windows, `smoke-test.sh` adds
`--ssl-no-revoke`, because Schannel cannot check revocation for the private CA.

### HTTPS site check and the sitemap fix (ADR-075)

- `web:baseline --site-only` over HTTPS could not log in: Chromium rejects Caddy's local CA
  (`ERR_CERT_AUTHORITY_INVALID`). The CA was not added to the OS trust store; that is the operator's choice.
- The run showed that an anonymous `/sitemap.xml` was 303-redirected to `/login`, although `robots.txt`
  advertises it. Earlier runs fetched it with a session. The sitemap is now a public path (03c0dee), with a test.
- Checked with curl over HTTPS instead: `__Host-cz_session` is `HttpOnly; Secure; SameSite=Strict`, and the CSP,
  nosniff and Referrer-Policy headers are present. There is no HSTS by design under the local CA (ADR-017), so
  COMPLIANCE WEB-HEADERS is now ⚠ and the posture is **75 ✓ / 8 ⚠ / 0 ✗**.

### Final reconciliation

- Web baseline: `npm run web:baseline -- --full` against the deployed web container (published on
  127.0.0.1:3000) after the fix. 120 cells, build fresh. The only failures are the Lighthouse performance carries,
  landing 72 and dashboard 74 (accepted, ADR-071), plus the 2 standing residuals (dashboard data probes are
  server-fetched; dashboard SEO is not scored).
- Invariants: `49 total / 48 machine (48 pass, 0 fail, 0 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`
- Typecheck 0, lint 0, vitest **34 files / 284 tests passed**.
- Drift scan: 0 drifts.

**Verdict: PASS.** The system is deployed locally, smoke-tested over HTTPS, and every machine invariant passes. The
residuals are listed above and in COMPLIANCE.md.

## Usage

Measured from the Claude Code session transcripts that built this project (main session + subagents), by `scripts/usage-report.mjs`. Tokens are what the model processed; on a subscription plan this is what draws the usage limits, and on the API it is what is billed.

| Metric | Value |
|---|---|
| Sessions / subagents | 6 / 49 |
| Assistant turns | 11646 (8433 main) |
| Cache-read tokens | 1,264,740,484 (main 982.7M, subagents 282.1M) |
| Cache-write tokens | 39,836,246 |
| Output tokens | 9,058,824 |
| Uncached input tokens | 23,608 |
| Context per turn (main) | avg 119K, peak 167K, 0 turns over 400K |
| Compactions | 65 |
| Active time | 30.2 h (24 pauses totalling 732.4 h; span 2026-10-12T00:00:53.913Z to 2026-11-12T18:36:45.342Z) |
| Models | claude-opus-5-5 x11644, <synthetic> x2 |

Per hour (UTC): cache-read / output / turns

| Hour | Cache-read | Output | Turns |
|---|---|---|---|
| 2026-10-12T00Z | 19.0M | 301K | 225 |
| 2026-10-12T07Z | 31K | 6 | 2 |
| 2026-10-13T00Z | 1.4M | 14K | 27 |
| 2026-10-15T01Z | 17.4M | 198K | 162 |
| 2026-10-15T02Z | 59.9M | 1.0M | 530 |
| 2026-10-15T03Z | 128.2M | 1.1M | 1227 |
| 2026-10-15T04Z | 90.2M | 485K | 830 |
| 2026-10-15T05Z | 100.3M | 388K | 966 |
| 2026-10-15T06Z | 78.8M | 331K | 739 |
| 2026-10-15T07Z | 38.0M | 222K | 352 |
| 2026-10-15T08Z | 24.1M | 137K | 226 |
| 2026-10-15T09Z | 52.5M | 215K | 464 |
| 2026-10-15T10Z | 1.1M | 6K | 8 |
| 2026-10-15T11Z | 20.2M | 223K | 186 |
| 2026-10-15T12Z | 27.7M | 204K | 222 |
| 2026-10-15T14Z | 2.3M | 6K | 17 |
| 2026-10-15T22Z | 18.0M | 102K | 173 |
| 2026-10-15T23Z | 28.4M | 86K | 282 |
| 2026-10-16T10Z | 1.7M | 16K | 22 |
| 2026-10-16T11Z | 9.6M | 41K | 83 |
| 2026-10-16T16Z | 56K | 452 | 2 |
| 2026-10-16T22Z | 1.0M | 5K | 9 |
| 2026-10-27T11Z | 383K | 5K | 8 |
| 2026-10-29T01Z | 330K | 3K | 6 |
| 2026-10-31T00Z | 772K | 4K | 12 |
| 2026-10-31T22Z | 2.5M | 14K | 32 |
| 2026-11-08T16Z | 4.6M | 26K | 66 |
| 2026-11-10T01Z | 582K | 8K | 8 |
| 2026-11-10T02Z | 265K | 2K | 3 |
| 2026-11-10T17Z | 11.6M | 205K | 94 |
| 2026-11-10T18Z | 12.5M | 65K | 106 |
| 2026-11-10T19Z | 18.3M | 68K | 153 |
| 2026-11-10T20Z | 31.5M | 178K | 263 |
| 2026-11-10T21Z | 32.8M | 283K | 284 |
| 2026-11-10T22Z | 37.7M | 312K | 321 |
| 2026-11-10T23Z | 33.0M | 197K | 286 |
| 2026-11-11T00Z | 43.9M | 292K | 425 |
| 2026-11-11T01Z | 26.6M | 125K | 235 |
| 2026-11-11T02Z | 25.3M | 123K | 236 |
| 2026-11-11T03Z | 5.9M | 31K | 65 |
| 2026-11-11T04Z | 2.0M | 7K | 17 |
| 2026-11-12T03Z | 31.7M | 395K | 289 |
| 2026-11-12T04Z | 20.1M | 139K | 183 |
| 2026-11-12T05Z | 10.1M | 34K | 69 |
| 2026-11-12T06Z | 32.1M | 225K | 275 |
| 2026-11-12T13Z | 25.7M | 241K | 223 |
| 2026-11-12T14Z | 34.1M | 273K | 271 |
| 2026-11-12T15Z | 29.2M | 214K | 288 |
| 2026-11-12T16Z | 59.7M | 477K | 546 |
| 2026-11-12T17Z | 10.3M | 67K | 114 |
| 2026-11-12T18Z | 1.7M | 4K | 14 |
## Governance

100 decision record(s) over 41 change(s), policy 1.3 (`GOVERNANCE-DECISIONS.jsonl`).

| Final outcome | Changes |
|---|---|
| AUTO_APPROVE | 41 |
| REVIEW | 0 |
| REWORK | 0 |
| HUMAN_APPROVAL | 0 |
| BLOCK | 0 |

**Human decisions:** F-1 approved (GD-0029); F-2 approved (GD-0030); F-3 approved (GD-0031); F-4 approved (GD-0032); F-5 approved (GD-0033); F-15 approved (GD-0034); F-17 approved (GD-0035); F-18 approved (GD-0050); F-19 approved (GD-0051); F-20 approved (GD-0052); F-18 approved (GD-0054); C-2 approved (GD-0076); C-5 approved (GD-0077); C-6 approved (GD-0078); C-7 approved (GD-0079); F-21 approved (GD-0080); F-22 approved (GD-0081); F-23 approved (GD-0082); F-24 approved (GD-0083); C-15 approved (GD-0100).
**Escalated to a human:** F-1 (minimum for project dependencies+infrastructure+payments (planned in F-1)); F-1 (minimum for project infrastructure (planned in F-1)); F-2 (minimum for project dependencies+infrastructure (planned in F-2)); F-3 (minimum for project dependencies+infrastructure+unknown (planned in F-3)); F-4 (minimum for project authentication+dependencies); F-5 (minimum for project authentication+dependencies); F-15 (minimum for project payments (planned in F-15)); F-17 (minimum for project dependencies+infrastructure+payments); F-17 (minimum for project dependencies+infrastructure); F-17 (minimum for project architecture+authentication+infrastructure+payments+security-boundary (planned in F-17)); F-17 (minimum for project authentication); F-17 (minimum for module authentication); F-17 (HUMAN_APPROVAL from GD-0023 stands (minimum for module build-record)); F-17 (HUMAN_APPROVAL from GD-0024 stands (minimum for module build-record+documentation)); F-17 (minimum for project infrastructure); F-17 (HUMAN_APPROVAL from GD-0026 stands (minimum for file security-boundary)); F-17 (HUMAN_APPROVAL from GD-0027 stands (minimum for module build-record+documentation)); F-18 (minimum for service security-boundary); F-19 (minimum for project dependencies (planned in F-19)); F-18 (HUMAN_APPROVAL from GD-0037 stands (minimum for module build-record+documentation)); F-20 (minimum for project architecture+unknown (planned in F-20)); F-20 (HUMAN_APPROVAL from GD-0042 stands (minimum for file application+build-record+documentation+tests (planned in F-20))); F-20 (HUMAN_APPROVAL from GD-0043 stands (minimum for module build-record+documentation)); F-20 (HUMAN_APPROVAL from GD-0044 stands (minimum for file api+security-boundary (planned in F-20))); F-20 (HUMAN_APPROVAL from GD-0045 stands (minimum for module build-record+documentation)); F-20 (HUMAN_APPROVAL from GD-0046 stands (minimum for module build-record+documentation)); F-20 (HUMAN_APPROVAL from GD-0047 stands (minimum for file build-record+documentation)); F-19 (HUMAN_APPROVAL from GD-0039 stands (minimum for service application (planned in F-19))); F-18 (minimum for project architecture+unknown (planned in F-18)); C-2 (minimum for project architecture); F-21 (minimum for project architecture+authentication+infrastructure (planned in F-21)); F-21 (HUMAN_APPROVAL from GD-0060 stands (minimum for module build-record)); F-22 (minimum for project architecture+authentication+dependencies+payments (planned in F-22)); F-22 (HUMAN_APPROVAL from GD-0062 stands (minimum for module build-record+documentation)); F-23 (minimum for project architecture+dependencies); F-23 (HUMAN_APPROVAL from GD-0064 stands (minimum for module build-record)); F-24 (minimum for project dependencies+infrastructure+unknown (planned in F-24)); C-5 (minimum for project architecture+dependencies+security-boundary); C-6 (minimum for project architecture); C-7 (minimum for project architecture); C-15 (minimum for project architecture).
**Blocked:** none.
