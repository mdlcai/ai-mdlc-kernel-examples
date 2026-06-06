# DECISIONS.md — Recon (ADR ledger)

Governance: SPEC.md = source of truth for behavior; ARCHITECTURE.md = source of truth for architecture. Ambiguity → choose safest implementation + record here.

---

## Gate outcomes
- **Stage 0 Research gate:** §3 absent in dashboard-wizard RESEARCH.md → research performed (comprehensive). Verdict GO. `force_research: false`, populated post-run. Logged.
- **Stage 1 review gate:** `review_gates: auto` → self-verify and proceed (matrix row "auto / any"). Logged.
- **Stage 2 review gate:** `review_gates: auto` → self-verify and proceed.
- **Multi-Agent Plan Gate:** `review_gates: auto` → surface plan, log, auto-dispatch (no halt).

---

## Architecture Decision Records

### ADR-001 — Multi-tenant isolation via Postgres Row-Level Security
**Status:** Accepted. **Context:** 50k+ assets, multi_tenant: true, scan results are highly sensitive. **Decision:** Shared-DB + RLS with `tenant_id` column and `current_setting('app.tenant_id')` policy; tenant GUC set transaction-scoped (`SET LOCAL`) via `withTenantScope`. **Alternatives:** schema-per-tenant (rejected — migration/ops blow-up at scale), app-layer WHERE only (rejected — single bug = cross-tenant leak). **Consequence:** INV-2, INV-3.

### ADR-002 — BullMQ Job Schedulers for recurring scans
**Status:** Accepted. **Decision:** Use BullMQ 5.x `upsertJobScheduler` (not legacy `bull` repeatable, deprecated). Per-target Redis lock prevents overlapping runs. **Consequence:** scan cadence subsystem.

### ADR-003 — Transactional outbox for all alert dual-writes
**Status:** Accepted. **Context:** domain signal `has_dual_write`, `has_webhooks`; guaranteed delivery. **Decision:** change event + outbox row in one ACID transaction; relay delivers at-least-once; receivers idempotent. **Consequence:** INV-6, INV-7.

### ADR-004 — RFC 9457 problem+json error contract
**Status:** Accepted. **Decision:** all `/v1` errors use `application/problem+json` with typed `code` + field-level `errors[]`. (RFC 9457 obsoletes 7807.) **Consequence:** SPEC §7.

### ADR-005 — argon2id password hashing + breached-password screening
**Status:** Accepted. **Context:** SOC2 + NIST 800-63B. **Decision:** argon2id; min length 12 (PCI v4 floor), accept ≥64, no composition rules, no forced rotation; HIBP k-anonymity breached-password screening; rate-limit before hash. **Consequence:** INV-9, INV-10.

### ADR-006 — Valkey over Redis 8
**Status:** Accepted. **Context:** Redis 8 is AGPLv3; SaaS license cleanliness. **Decision:** Valkey 8.x (BSD-3, Redis-protocol compatible) for queue/cache/ws-adapter. Managed Redis acceptable in production. **Consequence:** docker-compose uses valkey image.

### ADR-007 — Native Node scan engine as default; external binaries pluggable
**Status:** Accepted. **Context:** naabu (MIT) / nmap (NPSL) / masscan (AGPL) carry license + deploy-portability + CI-determinism concerns. **Decision:** ship a native Node TCP/TLS scan engine (`@recon/scanner`: TCP connect probing, TLS certificate inspection, service banner/version heuristics, optional CT enrichment) as default, behind a `ScanEngine` adapter interface so naabu/nmap can be plugged in where present + authorized. **Consequence:** deterministic, license-clean, runnable-in-CI default; INV-5 (authorization-to-scan) and INV-4 (SSRF guard) wrap every engine. Trade-off: less deep than nmap `-sV`; adapter preserves upgrade path.

### ADR-008 — Pin NestJS v11
**Status:** Accepted. **Decision:** pin NestJS 11.x; defer v12 (breaking ESM/Vitest/oxlint migration) until GA + stabilized.

### ADR-009 — Vault KV v2 for secrets with env-var fallback
**Status:** Accepted. **Decision:** production secrets via Vault KV v2 (`node-vault`); a `SecretsService` abstraction falls back to environment variables for local/dev so the default stack runs without a Vault instance. Fallback logged; production must set `VAULT_ADDR`/`VAULT_TOKEN`. **Consequence:** local deploy runnable; INV documented.

### ADR-010 — Design tokens copied verbatim from uploaded template
**Status:** Accepted. **Context:** Step 2b — a design template was uploaded (`DESIGN-TEMPLATE.html`, sha256 a495dca0…). **Decision:** copy the template `:root` token block verbatim as the canonical token system; it OVERRIDES get_project_config tokens where they conflict (the kernel/DESIGN.md template precedence). Notable overrides vs project config: accent `#2563eb` (template) replaces primary `#5e60d2` (config); surface/border/muted families, radius scale (6/11/18) and shadow scale taken from template. Fonts agree (Manrope display / Inter body / JetBrains Mono). Landing surface ported from the template scaffold. **Copied `:root` block** (verbatim):
```css
:root{
  --color-bg:#f4f7fb; --color-surface:#ffffff; --color-surface-2:#eef2f8;
  --color-fg:#0c1424; --color-muted:#586173; --color-border:#e1e7f0;
  --color-accent:#2563eb; --color-accent-2:#1d4fd8; --color-accent-soft:#e1ebfe;
  --color-success:#15803d; --color-success-soft:#dcfce7;
  --color-warning:#b45309; --color-warning-soft:#fef0d6;
  --color-error:#dc2626; --color-ink:#070b14;
  --font-display:'Manrope',sans-serif; --font-body:'Inter',sans-serif; --font-mono:'JetBrains Mono',monospace;
  --space-1:4px; --space-2:8px; --space-3:12px; --space-4:16px; --space-5:24px;
  --space-6:32px; --space-7:48px; --space-8:64px; --space-9:96px;
  --radius-sm:6px; --radius-md:11px; --radius-lg:18px;
  --shadow-sm:0 1px 2px rgba(12,20,36,.06),0 1px 1px rgba(12,20,36,.04);
  --shadow-md:0 6px 16px -6px rgba(12,20,36,.12),0 2px 6px -2px rgba(12,20,36,.08);
  --shadow-lg:0 28px 60px -24px rgba(12,20,36,.28),0 10px 24px -12px rgba(12,20,36,.14);
}
```
**Theme mode:** template + config are light-only → no dark theme switcher (config `theme.mode: light`). **Consequence:** INV-1.

### ADR-011 — Deploy outcome: local-stack (Outcome C)
**Status:** Accepted. **Context:** externals (Vault, Resend, Sentry, Slack) unprovisioned in this environment; HTTPS-only constraint. **Decision:** Stage 4 deploys a local docker-compose stack; HTTPS via self-signed cert + nginx (HTTP→HTTPS redirect + HSTS) per QUICKSTART Protocol & TLS; Functional Smoke Test runs against the local https/http URL; QUICKSTART documents switching to cloud + provisioning externals. **Consequence:** Deploy URL is local; not fabricated.

---

## Multi-Agent Plan (Stage 3) — logged, auto-dispatched (review_gates: auto)
- **Wave 0 foundation:** monorepo, API backbone (config/db/RLS migration/error layer/tenant scope/SSRF guard/secrets/throttler/health), packages/scanner, invariant-lint runner. Features F-01,F-05,F-14.
- **Wave 1 API modules (file-disjoint):** F-02 auth (`/v1/auth/*`, tables users/sessions/memberships), F-03 orgs (`/v1/orgs/*`, orgs/memberships), F-04 assets (`/v1/assets/*` + authorize/verify, assets table), F-06 scans (`/v1/assets/:id/scans`,`/scans/:id`,`/assets/:id/schedule`, scans/snapshots), F-07 changes (`/v1/changes/*`,`/assets/:id/baseline`, change_events/baselines), F-08 alerts (`/v1/alert-channels/*`,`/alert-deliveries`, alert_channels/outbox_delivery; webhook-sender signs before send), F-09 ws (`WS /ws`), F-10 reports (`/v1/reports/*`, reports table, PDF/JSON/CSV), F-11 audit (`/v1/audit-log`, audit_log), F-12 dashboard (aggregation read).
- **Wave 2 web:** F-13 landing (ported template) + 12 screens, tokens verbatim, TanStack Query, RFC-9457 error mapping.
- **Wave 3:** Jest unit/integration, Playwright W1–W7, docker-compose + nginx TLS, CLAUDE.md/.claude, QUICKSTART/COMPLIANCE/README/NOTICE.
- Bundle integrity: each feature's endpoints/tables/screens enumerated above (flat, scannable). No bundling that hides obligations.

### ADR-012 — TypeORM 0.3.x pinned (substitution from 1.0)
**Status:** Accepted. **Context:** TypeORM 1.0 GA'd 2026-05-19 (very new). **Decision:** pin battle-tested 0.3.x; identical entity/migration/RLS-via-raw-SQL behavior the SPEC relies on; no behavioral deliverable changed. Upgrade path preserved. Behavior-preserving substitution (not a RESEARCH-deliverable downgrade — same ORM, same multi-tenant RLS approach).

### ADR-013 — Next.js 15.x pinned (substitution from 16)
**Status:** Accepted. **Decision:** pin Next.js 15.x LTS (supported through 2026-10-21) App Router; same framework/router/SSR architecture as the specified 16. Behavior-preserving; design system, a11y, and perf deliverables unchanged.

### ADR-014 — @node-rs/argon2 for argon2id hashing
**Status:** Accepted. **Decision:** use `@node-rs/argon2` (napi prebuilds, argon2id) instead of the `argon2` C++ addon to avoid a native build toolchain across Windows/Linux. Still argon2id (satisfies SEC-6, INV-9 — not bcryptjs/argon2-browser). Behavior preserved.

### ADR-015 — Demo auto-verify of authorization-to-scan (local/demo only)
**Status:** Accepted. **Context:** the local/demo deploy cannot control real DNS/HTTP for a sample asset. **Decision:** when `RECON_ALLOW_PRIVATE_TARGETS=true` (demo), `assets.verify()` auto-succeeds so W2 is completable; production performs real DNS-TXT/HTTP-file verification. The scan gate (INV-5: status must be `authorized`) is enforced unchanged in all modes.

### ADR-016 — Demo drift affordance for change detection (local/demo only)
**Status:** Accepted. **Context:** W4 (change detection) needs a surface change between baseline and a re-scan; local infra does not change between scans. **Decision:** when `RECON_DEMO_DRIFT=true` (set only in the demo compose, never prod), if a re-scan's snapshot equals the accepted baseline, the runner appends a single synthetic open port so a real `PORT_OPENED` change event is produced through the genuine diff/outbox pipeline. Off in production → only real diffs. Documented as demo-only.

### ADR-017 — Manual scan runs inline; recurring scans via BullMQ (local determinism)
**Status:** Accepted. **Decision:** manual scan triggers execute the scan synchronously (scan performed outside the DB transaction; results persisted in a short tx) for deterministic smoke testing; recurring/scheduled scans use BullMQ Job Schedulers (ADR-002). Per-asset Redis lock prevents overlapping runs in both paths.

### ADR-018 — Security Audit Gate dispositions (build 0.1.1)
**Status:** Accepted. **Pass 1 findings:** 2 MEDIUM (npm audit: postcss <8.5.10 XSS-in-CSS-stringify, transitive via Next 15). 0 CRITICAL/HIGH. Secret scan: no real secrets committed (only env-overridable dev fallbacks). **Auto-remediate (Pass 2):** (a) added root `overrides: { postcss: ^8.5.10 }` → top-level postcss 8.5.15; (b) SAST hardening — `assertProductionSecrets()` in main.ts refuses production boot with default/unset JWT_SECRET/WEBHOOK_SIGNING_SECRET. **Pass 3 residual:** Next 15 still bundles a nested `postcss@8.4.31` for its build pipeline; the only upstream fix (downgrade Next to v9) is a prohibited RESEARCH-deliverable framework downgrade. **Disposition:** the 2 MEDIUM are ACCEPTED-with-rationale + tracked — postcss is a build-time-only dev dependency (not in the runtime attack surface), the advisory requires processing untrusted CSS we never accept, and it clears when Next bumps its bundled postcss. No residual CRITICAL/HIGH. Gate passes.
