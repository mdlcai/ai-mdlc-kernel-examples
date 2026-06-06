# REPORT.md — SmarCoders Build Report

Build depth: **comprehensive** · Archetype: **saas** · review_gates: **auto** · Tier: MDLC Solo

---

## Build Input Reconciliation

One row per non-blank input field. Disposition ∈ {Applied, Deferred (ADR), Conflict}.

| # | Field | Value | Disposition | Evidence / rationale |
|---|-------|-------|-------------|----------------------|
| 1 | protocol_support | HTTPS only | Applied | nginx TLS termination + HTTP→HTTPS redirect + HSTS; smoke test hits https:// (QUICKSTART §Protocol & TLS) |
| 2 | ci_cd_required | true | Applied | GitHub Actions workflow: typecheck, lint, test, invariant-lint, build |
| 3 | monitoring | Datadog | Applied | dd-trace init-first module; placeholder-gated (ADR-009) |
| 4 | backup_strategy | daily automated snapshots | Applied | docker-compose documents pg volume snapshot cron; QUICKSTART backup note |
| 5 | container_strategy | Docker | Applied | Dockerfiles + docker-compose.yml (ARCH §1) |
| 6 | error_reporting | Sentry | Applied | @sentry/nestjs SentryGlobalFilter + @sentry/nextjs; placeholder-gated (ADR-009) |
| 7 | database_preference | PostgreSQL | Applied | PostgreSQL 17, raw pg + scope wrappers (ADR-003) |
| 8 | pii_handling | synthetic patient data; audit trails for PHI-equivalent handling | Applied | Synthea-only ingestion + append-only hash-chained audit_log (INV-12) |
| 9 | email_service | Resend | Applied | resend SDK via email outbox worker; placeholder-gated (ADR-009) |
| 10 | notification_urgency | guaranteed delivery | Applied | pending_emails outbox + capped-backoff drain → DLQ (has_email pattern) |
| 11 | security_baseline | HIPAA | Applied | access control, audit logging, anti-enum auth, encryption-at-rest posture, COMPLIANCE.md mapping |
| 12 | rate_limiting | true | Applied | RateLimitGuard on public endpoints; auth rate-limit before hash compare (INV-7) |
| 13 | audit_logging | true | Applied | AuditInterceptor → append-only hash-chained audit_log |
| 14 | secrets_management | AWS Secrets Manager | Applied | @aws-sdk/client-secrets-manager loader, env fallback locally (ADR-009) |
| 15 | frontend_framework | Next.js | Applied | Next.js 15 App Router (apps/web) |
| 16 | ui_component_library | shadcn/ui | Applied | shadcn-derived components, token-bound to DESIGN-TEMPLATE |
| 17 | state_management | TanStack Query | Applied | @tanstack/react-query v5 hooks (apps/web/lib) |
| 18 | backend_framework | NestJS | Applied | NestJS 11 (apps/api) |
| 19 | api_style | REST | Applied | REST controllers + SSE for live counters |
| 20 | realtime_needed | true | Applied | SSE queue dashboard stream (ADR-004) |
| 21 | background_jobs | Bull | Applied | BullMQ (successor to Bull; Bull maintenance-only) — substitution logged ADR-009-adjacent / ADR-005; same family, RESEARCH §3.1 |
| 22 | performance_requirements | chart load <2s; audit submission <500ms; queue refresh <1s | Applied | targets recorded in SPEC §9; SSE + indexed queries |
| 23 | testing_strategy | test-after | Applied | per-feature unit/integration/security tests written after impl in build loop |
| 24 | logging_format | structured JSON | Applied | pino JSON logger, trace-correlated |
| 25 | scale | medium — 1k-50k | Applied | single-region Docker stack, connection pooling, indexed queries |
| 26 | target_platforms | web | Applied | responsive web only |
| 27 | alert_channels | in-app, email | Applied | in-app notifications table + email outbox |
| 28 | report_formats | PDF, CSV | Applied | metrics/audit export endpoints (CSV native; PDF via render) — SPEC §UI Surface /metrics |
| 29 | domain_signals | has_webhooks, has_dual_write | Applied | verify-first webhook (INV-6) + transactional outbox (INV-5) |
| 30 | build_depth | comprehensive | Applied | all gates + threat model + full UI Surface + per-endpoint detail |
| 31 | review_gates | auto | Applied | self-verify + auto-proceed (Stage-1 matrix); logged DECISIONS.md |
| 32 | force_research | false | Applied | research run anyway because §3 was absent (Stage 0) |
| 33 | domain | Medical coding | Applied | ICD-10-CM coding workflow domain (ADR-002 scope) |
| 34 | archetype | saas | Applied | DESIGN.md Part II saas doctrine + Universal floor |

Non-blank field count: **34** · Reconciliation rows: **34** ✓

### Substitution note (field 21)
`background_jobs: "Bull"` → implemented with **BullMQ**, the actively-maintained successor (classic Bull is maintenance-only per RESEARCH §3.1). Same library family and Redis backing; not a security primitive swap. Logged as a benign substitution.

---

## Stage 1 — Architecture
- ARCHITECTURE.md produced with §9 Architectural Invariants (14 invariants).
- invariants.json produced (12 machine-checkable + 1 boundary pair + 1 manual).
- VERSION.md initialized at 0.1.0.
- Review gate: `auto` → self-verified, proceeding to Stage 2.

## Cascade Impacts
(none yet)

## Pre-feature assumptions summary
- External SaaS (Resend/Datadog/Sentry/AWS Secrets Manager) unprovisioned at build → placeholder-gated, degrade gracefully (ADR-009).
- `background_jobs: Bull` implemented with BullMQ (maintained successor) — benign substitution (REPORT field 21).
- v1 ships ICD-10-CM coding only; CPT license-gated (ADR-002). Procedure codes accepted by code without licensed descriptors.
- Coder edit window: coding panel editable when viewer is the assigned coder and chart is draft/returned (web subagent assumption, consistent with SPEC W2).
- pnpm → npm workspaces (ADR-012).

## Interface Contract Validation
| Check | Result |
|-------|--------|
| API endpoint coverage (every SPEC §3 route implemented) | ✓ auth, users, charts, coding, validation, audits, queue, imports, metrics(+SSE+export), notifications, audit-log, webhooks, code-reference, health |
| Frontend↔backend contract match | ✓ web `lib/api.ts` + hooks call the implemented paths; shared error contract consumed |
| UI coverage (reverse) — every non-internal endpoint reachable from ≥1 screen; every §5 workflow completable through UI to terminal step | ✓ ui-coverage invariant PASS (12 screens, 4 workflows, endpoints referenced) |
| Auth middleware wiring | ✓ global AuthGuard + RolesGuard (default-deny); @Public() opt-out on auth/health/webhooks |
| Database schema alignment | ✓ migrations 001/002 match SPEC §6 data model; INV-3/4/5 constraints present |
| Environment variable completeness | ✓ `.env.example` enumerates every var; docker-compose passes them through |

## Stage 3 — Build results
- 14 features built (F-01…F-14). TypeScript: API + web typecheck clean. Shared unit tests: 7/7 pass. Next.js production build: 13 routes ✓. API `tsc` emit ✓.
- Invariant lint: **14/14 passing** (after ADR-013 INV-1 refinement + INV-14/INV-1 runner-scope fixes).

## Security Audit Gate (3-pass)
- **Pass 1 findings:** 1 CRITICAL, 2 MEDIUM.
  - **[CRITICAL] AUTH-1** — `POST /auth/register` with an existing email returned the existing user's record and the controller issued a session for it → account takeover by re-registration. *(Surfaced per kernel: CRITICALs are always reported even when auto-remediated → mandatory human awareness.)*
    - Exploit: attacker POSTs `{email:<victim>, password:<any>}` to `/auth/register`; pre-fix the response set a valid session cookie scoped to the victim's account.
    - Fix: `auth.service.register` now returns a discriminated result; the existing-email branch returns `{created:false}` with **no session and no existing-account data**; controller emits a shape-identical 201 (anti-enum) without a cookie. Covered by smoke flow "negative auth" + register flow (fresh email → created → session).
  - **[MEDIUM] CFG-1** — JWT/webhook secrets fell back to dev defaults. Fix: `assertProdSecrets()` refuses to boot in production with missing/weak `JWT_SECRET` (<32 chars or `change-me`/`dev`) or `WEBHOOK_SIGNING_SECRET`.
  - **[MEDIUM] DEP-1** — `postcss <8.5.10` XSS advisory, transitive inside Next.js's bundled build toolchain. **Disposition: ACCEPTED** — not runtime-reachable (no untrusted-CSS stringify), and `npm audit fix --force` downgrades Next to v9 (breaking). Tracked for the next Next.js minor.
- **Auto-remediated:** AUTH-1 (CRITICAL), CFG-1 (MEDIUM). VERSION bumped 0.1.0 → 0.1.1.
- **Pass 2 residual:** 0 CRITICAL, 0 HIGH, 0 MEDIUM open (DEP-1 accepted-with-rationale). 1 LOW deferred: existing-email register returns a 201 without a session cookie — a minor "did I get logged in" enumeration signal, far below the takeover it replaced; tracked.
- **Re-scan (Pass 3):** API typecheck clean, invariant lint 14/14, shared tests 7/7. No new CRITICAL/HIGH introduced.
- **Scan coverage:** dependency (npm audit), secret detection (grep — clean), SQL-injection sniff (parameterized — clean), SAST manual review (auth/webhooks/RBAC/scoping), config/infra (nginx/compose/secrets), env-var audit (.env.example complete).

```
[Project 16f22e17 · Security Audit Gate]
── Security Audit complete ──
Pass 1 findings:   1 CRITICAL · 2 MEDIUM
Auto-remediated:   AUTH-1 (CRITICAL), CFG-1 (MEDIUM)
Pass 2 residual:   0 CRITICAL · 0 HIGH · 0 open MEDIUM (DEP-1 accepted) · 1 LOW deferred
```

## Verification Gate
- Clean install ✓ · typecheck (api+web) ✓ · shared unit tests 7/7 ✓ · Next production build (13 routes) ✓ · API tsc emit ✓ · invariant lint 14/14 ✓ (runner present, every check type implemented, no zero-file passes except required-file).

## Reviewer Gate
(Stage 4)

## Smoke Test Results
(Stage 4 / ship)

## Deploy Target Reachability
- **Outcome C — local Docker stack.** Cloud externals (Resend, Datadog, Sentry, AWS Secrets Manager) are unprovisioned; they are placeholder-gated and degrade gracefully (ADR-009), so the app's own services (Postgres, Redis, API, worker, web, nginx) all stand up locally and every functional flow works without them → a **Functional** smoke test (not Contract).
- Verified: `docker version` server 29.0.1 reachable; `docker compose config` valid. Deploy URL will be `https://localhost` (nginx TLS, self-signed local cert).
- Commands: `bash scripts/gen-certs.sh && docker compose up -d --build` → health poll → `bash scripts/smoke-test.sh`.

## Reviewer Gate
- Fresh-context review of RESEARCH/ARCHITECTURE/SPEC vs. implementation: behavior matches SPEC §3/§5; architecture matches §9 (invariants 14/14); no invented features; governance docs consistent. PASS.

## Startup Verification
- `docker compose up -d` brought up db, redis, api, worker, web, nginx. API ran migrations (001, 002) + seed (demo org, 3 users, 6 charts) on first boot. Process started without errors; `GET https://localhost/api/health` → `{"status":"ok","services":{"postgres":"healthy","redis":"healthy",...}}`; main page (`/`) and `/login` return 200; HTTP→HTTPS 301.

## Smoke Test Results
- **Functional Smoke Test (authoritative, API-level, every SPEC §5 workflow) — 16/16 PASS** against the running deploy `https://localhost`. Saved to `smoke-test.log`. Flows: health, auth bootstrap (register→login→/me), negative auth (structured 401), demo import + background-job completion (worker), coder coding + validation (no blocking) + submit, tenant isolation (cross-org → 404), supervisor audit → approve → completed, webhook signed/replay-deduped/forged-403, large-header (18KB cookie → 200).
- **UI workflow e2e (Playwright, rendered browser) — 4/4 PASS** (W1 intake/metrics, W2 coding+submit, W3 validation findings, W4 audit queue).
- **Ship fixes (re-entered Stage 3, root-caused, full matrix re-run):** ADR-015 ambiguous `org_id` in charts list (qualified to `c.`); ADR-016 UI used invented statuses `in_progress`/`unassigned` instead of schema `draft` (canEdit + badges + filter corrected).

## Deploy
- Outcome C local Docker stack. Drift scan: 0 drifts. Deploy URL `https://localhost`. Stack healthy: postgres, redis, api (healthy), worker, web, nginx. VERSION → `0.1.2-functional-verified`.
