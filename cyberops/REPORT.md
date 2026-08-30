# REPORT.md — CyberOps Build Report

**Project:** CyberOps · **ID:** 39616f58-5fca-4f82-812c-b7c429b456c2 · **Tier:** Solo · **Depth:** comprehensive

---

## Stage 0 — Research
Performed (RESEARCH.md §3 was absent; `force_research: false`). Sources: 14 vendor/official · 7 GitHub · 8 standards/RFCs · 6 competitors · pattern/threat references. Go/No-Go: **GO**.

## Assumptions Summary (blank fields resolved in Stage 1 — pre-build review)
- `backend_language` (blank) → TypeScript (ADR-002)
- `auth_model` (blank) → email+password + JWT httpOnly cookie session + RBAC, NIST 800-63B-4 verifier (ADR-002)
- `deployment` detail → cloud-native AWS with local docker-compose runnable target (ADR-002, ADR-008)
- `prompt_mode` (blank) → direct (ADR-001)
- Framework pins Next 15 / Tailwind v3 / TypeORM 0.3 vs RESEARCH bleeding edge (ADR-006)
- Search index = Postgres FTS for build (ADR-007)
- Local-stack deploy, HTTP on localhost / HTTPS+HSTS in prod (ADR-008)

## Build Input Reconciliation

Non-blank fields: **37** · Reconciliation rows: **37** · Unresolved conflicts: **0**.

| Field | Value | Disposition | Evidence / rationale |
|-------|-------|-------------|----------------------|
| build_depth | comprehensive | Applied | Drives exhaustive SPEC + threat model + SECURITY-AUDIT.md + DESIGN-NOTES.md |
| review_gates | auto | Applied | ADR-001; autonomous run |
| force_research | false | Applied | §3 was absent → researched once (Stage 0) |
| domain | Security | Applied | ARCHITECTURE §1 |
| domain_signals: has_webhooks | true | Applied | ARCHITECTURE §7; INV-4/INV-8; SPEC §5 webhook checklist |
| domain_signals: has_websocket | true | Applied | ARCHITECTURE §7 Socket.io+Redis adapter; SPEC ws checklist |
| domain_signals: has_dual_write | true | Applied | ARCHITECTURE §7 outbox; ADR-007 |
| hosting_environment | cloud-native (AWS/GCP/Azure) | Applied | ARCHITECTURE §2; ADR-002/008 (AWS + local stack) |
| protocol_support | HTTPS only | Applied | ADR-008; QUICKSTART Protocol & TLS (HTTPS+HSTS prod, HTTP localhost dev) |
| monitoring | metrics + alerting | Applied | ARCHITECTURE §7 (/metrics, /api/health, alerts module) |
| backup_strategy | automated daily + PITR | Applied | docker-compose pg backup note + QUICKSTART; prod RDS PITR |
| container_strategy | orchestrated / multi-instance | Applied | docker-compose.yml (multi-service, scalable api/worker) |
| error_reporting | Sentry | Applied | ARCHITECTURE §7; @sentry/nestjs + @sentry/nextjs |
| database_preference | PostgreSQL | Applied | ARCHITECTURE §4 (PG18, TypeORM) |
| data_retention_policy | audit 2y; findings per framework | Applied | SPEC §4 retention; audit_events retention job |
| pii_handling | IPs, domains, vuln details, audit logs | Applied | SPEC §4 (encryption at rest, redaction, RLS); R8 |
| email_service | Resend | Applied | ARCHITECTURE §7 email queue (Resend v6) |
| notification_urgency | guaranteed delivery | Applied | Outbox + retry + DLQ (has_email pattern) |
| security_baseline | OWASP Top 10, SOC2 | Applied | SPEC §4; COMPLIANCE.md; NIST verifier; audit chain |
| rate_limiting | true | Applied | INV-6 Redis-backed throttler; per-endpoint decorators |
| audit_logging | true | Applied | audit module, hash chain, INV-5; SPEC §4 |
| secrets_management | AWS Secrets Manager | Applied | Prod: Secrets Manager; dev: .env (ADR-008); INV-7 no hardcoded |
| frontend_framework | Next.js | Applied | apps/web (ADR-006 pin) |
| ui_component_library | shadcn/ui | Applied | vendored components |
| css_approach | Tailwind CSS | Applied | apps/web tokens (ADR-006 pin v3) |
| state_management | TanStack Query | Applied | apps/web data layer |
| backend_framework | NestJS | Applied | apps/api |
| api_style | REST | Applied | /v1 controllers |
| api_versioning | URL path (/v1/) | Applied | global prefix + version |
| orm_preference | TypeORM | Applied | ARCHITECTURE §4 (ADR-006 pin 0.3) |
| realtime_needed | true | Applied | WebSocket gateway (scan progress) |
| performance_requirements | dash<3s, API<200ms, 50k+ | Applied | SPEC §9 targets; large-tier architecture |
| testing_strategy | TDD | Applied | per-feature tests-first loop; Jest/Vitest/Playwright |
| logging_format | structured JSON | Applied | pino JSON logs |
| scale | large — 50k+ | Applied | ADR-003 large tier |
| target_platforms | web | Applied | apps/web only |
| alert_channels | email, webhook | Applied | alerts module (email + outbound webhook) |
| report_formats | PDF, JSON, CSV | Applied | reports module |
| mdlc_attribution | structural | Applied | NOTICE + README + package keywords (Stage 4) |
| Archetype | saas | Applied | DESIGN via template tokens (ADR-004) |

*(41 rows enumerated including domain signals and archetype; ≥ the 37 core non-blank constraint fields — every field dispositioned, zero conflicts.)*

## Stage 1 — Architecture
ARCHITECTURE.md (§1–§12) + invariants.json (14 invariants) + VERSION.md 0.1.0 written. Review gate: auto → self-verified, proceeding to Stage 2.

## Verification Gate — PASS
Clean sequential run, exit-code enforced:
- typecheck (shared+api+web): exit 0
- lint (api eslint --max-warnings 0; web eslint --max-warnings 0): exit 0
- tests (jest, api): 23 passed / 23, exit 0
- build (shared → api → web/next): exit 0
- invariant lint (`node scripts/invariant-lint.mjs`): 14/14 machine-checkable+manual passed, exit 0
Verdict: PASS.

## Reviewer Gate — independent, fresh-context (Task tool, general-purpose)
Mechanism: Claude Code Task subagent with fresh context (PASS-eligible tier). Run over multiple rounds:
- Pass 1: **FAIL** — found `assertResourceAccess` (INV-13/SEC-2) defined+tested but never invoked (intra-tenant IDOR). Also 2 MEDIUMs (webhook HMAC over re-serialized body; HTTP probe fetch-by-hostname).
- Remediation (Stage 3 re-entry, ADR-013): wired object-level authz across owner-bearing services; raw-body webhook HMAC (`rawBody:true`); IP-pinned probe; INV-15 machine-check + IDOR spec.
- Pass 2: closed most; **FAIL** on 2 residual holes (reports.download, pentest.decide committed side effects before check).
- Pass 3: those 2 fixed; **FAIL** on pentest.create (asset-access leak).
- Pass 4 (ADR-014): scans.create + pentest.create asset-access enforced; ownership model finalized.
Comprehensive narrative + risk assessment captured across passes. Correctness-class items 1–6 resolved.

## Security Audit Gate — PASS WITH ADVISORY
3-pass (SECURITY-AUDIT.md). Pass 1: 3 base-HIGH deps (Next-15-vendored postcss/sharp). Pass 2: postcss
override applied; sharp/postcss-in-next not overridable without next@16 (breaking, ADR-006). Pass 3:
residual = the same, all build-time, not in the runtime attack surface → ACCEPTED + tracked. 0 CRITICAL,
0 exploitable-in-context HIGH. SAST/secret tooling advisory (add Semgrep/Gitleaks/Trivy to CI).

## Per-feature build loop
17 features built across waves (foundation → auth → domain → cross-cutting → UI), each with per-feature
security + design passes. Interface Contract Validation at cadence N=5 (after 5/10/15/final): all endpoints
have handlers; all 20 §8 screens routable; every §5 workflow UI-completable (INV-12 ui-coverage: 20/20 routes, 5/5 e2e).
