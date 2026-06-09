# DECISIONS.md — ServiceHub

Architecture Decision Records (ADRs), assumptions, deviations, and gate outcomes. Updated continuously.

## Build Strategy (Stage 1)
- `build_depth: comprehensive` (from RESEARCH.md) → exhaustive spec, threat model, alternatives-considered, per-feature security + architecture-adherence reviews.
- `review_gates: auto` (from RESEARCH.md + outer prompt) → all approval gates and the post-Stage-2 menu are suppressed; summarize each stage and continue automatically.
- `prompt_mode`: blank → default `direct`. Logged.
- `force_research: false`; §3 was absent → research performed in Stage 0 (not a re-research). 

## Design Template (Stage 1, Step 2b)
- `get_design_template` returned a hit: `ServiceHub (standalone).html` (sha256 e47efcd6…b414f4, uploaded 2026-06-09). Saved raw to `DESIGN-TEMPLATE.html`.
- The template is a self-extracting bundler page; the design markup lives inside it (plain hand-written CSS, no Tailwind, no JS). It is a marketing landing page (nav, hero with app mockup, trust strip, feature cards, CTA, footer).
- **Copied `:root` tokens VERBATIM** (binding for the whole build, overriding get_project_config where they conflict):

```css
:root {
  --color-bg: #f6f7f9;
  --color-surface: #ffffff;
  --color-surface-2: #eef1f5;
  --color-fg: #0e1420;
  --color-muted: #5a6573;
  --color-border: #e0e4ea;
  --color-accent: #2563eb;
  --color-accent-soft: rgba(37, 99, 235, 0.10);
  --color-accent-line: rgba(37, 99, 235, 0.28);
  --color-success: #15803d;
  --color-success-soft: rgba(21, 128, 61, 0.12);
  --color-warning: #c2410c;
  --color-warning-soft: rgba(194, 65, 12, 0.12);
  --color-error: #dc2626;
  --color-error-soft: rgba(220, 38, 38, 0.12);
  --font-display: "Space Grotesk", system-ui, sans-serif;
  --font-body: "Inter", system-ui, sans-serif;
  --font-mono: "JetBrains Mono", ui-monospace, monospace;
  --space-1: 4px;  --space-2: 8px;  --space-3: 12px;
  --space-4: 16px; --space-5: 24px; --space-6: 32px;
  --space-7: 48px; --space-8: 64px; --space-9: 96px;
  --radius-sm: 6px;  --radius-md: 12px;  --radius-lg: 20px;
  --shadow-sm: 0 1px 2px rgba(14,20,32,.05), 0 1px 3px rgba(14,20,32,.04);
  --shadow-md: 0 4px 12px rgba(14,20,32,.06), 0 2px 4px rgba(14,20,32,.04);
  --shadow-lg: 0 24px 60px rgba(14,20,32,.14), 0 8px 24px rgba(14,20,32,.08);
}
/* dark — colors + shadows only; font/space/radius inherited */
[data-theme="dark"] {
  --color-bg: #0b0d11; --color-surface: #14171d; --color-surface-2: #1c212a;
  --color-fg: #f1f4f8; --color-muted: #9aa4b3; --color-border: #272d37;
  --color-accent: #3b82f6; --color-accent-soft: rgba(59,130,246,.16); --color-accent-line: rgba(59,130,246,.40);
  --color-success: #4ade80; --color-success-soft: rgba(74,222,128,.14);
  --color-warning: #fb923c; --color-warning-soft: rgba(251,146,60,.14);
  --color-error: #f87171; --color-error-soft: rgba(248,113,113,.14);
  --shadow-sm: 0 1px 2px rgba(0,0,0,.4);
  --shadow-md: 0 4px 14px rgba(0,0,0,.45);
  --shadow-lg: 0 24px 60px rgba(0,0,0,.6), 0 8px 24px rgba(0,0,0,.5);
}
```
- **Surfaces to port from template:** marketing landing (`/`) — nav, hero (with in-product app-window mockup), trust strip, feature cards, CTA block, footer. The authenticated app shell reuses these tokens for full visual coherence.
- Note: template's accent `#2563eb` matches the project-config primary; template uses richer surface tokens (`--color-surface-2`, soft/line accent variants, 3-step shadow scale) which become canonical.

## ADRs

### ADR-01 — Multi-tenant isolation: Postgres RLS + `SET LOCAL` per transaction
Constraint `multi_tenant: true`, `scale: medium`. Chosen: shared-schema with mandatory `tenant_id` on every tenant-scoped table, enforced by Postgres Row-Level Security policies, with tenant context injected via `SET LOCAL app.tenant_id = $id` inside a per-request transaction. Defense-in-depth app-layer tenant guard. Alternatives: schema-per-tenant (rejected: 1k–50k tenants → migration/ops explosion), app-filter-only (rejected: one forgotten `WHERE` = cross-tenant leak). Mitigates R1. Pitfall designed around: TypeORM pool connections leaking context (#5857) — context is only ever set inside a transaction, never on a pooled connection at rest.

### ADR-02 — Tenant context propagation: nestjs-cls (AsyncLocalStorage)
Chosen ALS via `nestjs-cls` over REQUEST-scoped DI (which rebuilds the DI subtree per request and degrades throughput at medium scale). Per §3.7.

### ADR-03 — Object-level authorization chokepoint
A single `assertResourceAccess(row, principal)` helper enforces row ownership (requester owns ticket) IN ADDITION to org/tenant scope, called on every ticket route AND every sub-resource (`/:id/comments`, `/:id/timeline`, `/:id/attachments`) and child-by-id route. Non-owners receive `404 Not Found` (not 403) to avoid existence leakage. Mitigates R2 / W3.

### ADR-04 — Guaranteed delivery via transactional outbox
`notification_urgency: guaranteed delivery` + `has_webhooks`/`has_email` signals → `pending_emails` and `pending_webhooks` outbox tables written in the same DB transaction as the triggering business change; BullMQ drain workers publish with capped backoff to a DLQ. UNIQUE dedup keys. Never synchronous send. Mitigates R5.

### ADR-05 — SLA computation: single server-side evaluator
SLA `due_at`, `at_risk`, `breached` computed only by a server-side `SlaService` driven by a BullMQ repeatable evaluator job; no client-settable SLA fields. Append-only audit log with server timestamps. Mitigates R8.

### ADR-06 — Storage abstraction (G2)
`StorageService` interface; local-disk driver (outside webroot) for local/dev, S3-compatible driver for cloud via env config. Attachments validated by magic-bytes + extension allowlist + size/quota; randomized object keys; `scan_status` quarantine flag with a pluggable AV hook (G1).

### ADR-07 — Error contract: RFC 9457 problem+json
All API errors return `application/problem+json` per RFC 9457 (obsoletes 7807), with `type`, `title`, `status`, `detail`, `instance`, and an `errors` member carrying field-level validation detail so the UI can attach messages to offending controls.

### ADR-08 — Auth model: JWT access + refresh with rotation
`auth_model` blank → default chosen: JWT bearer (short-lived access) + rotating refresh token stored httpOnly+Secure+SameSite cookie. Fits stateless autoscaling. Anti-enumeration on all auth endpoints (uniform shape + timing, rate-limit before hash, HIBP breach screening per NIST 800-63B). Mitigates R4.

### ADR-09 — Monorepo layout (pnpm + Turborepo-style)
`apps/api` (NestJS), `apps/web` (Next.js), `packages/shared` (DTO/types/contracts). docker-compose orchestrates postgres, redis, api, worker, web. Reverse proxy (Caddy) terminates TLS (HTTPS only) and forces HTTP→HTTPS redirect + HSTS.

### ADR-10 — Gate outcomes
- Stage 1 review gate: `review_gates: auto` → self-verify and proceed. Logged.
- Multi-agent plan gate (Stage 3): `review_gates: auto` → summarize, log, dispatch automatically (no halt).

### ADR-11 — Build tooling: npm workspaces (pnpm not installed)
ADR-09 named "pnpm + Turborepo-style". pnpm is not present in the build environment and the offline-safe action is to use npm workspaces (Node 22 / npm 10), which provide identical monorepo semantics for this project. Tooling-only substitution; NOT a RESEARCH-level deliverable, no cascade. Logged per Substitution discipline.

## Multi-Agent Plan Gate (Stage 3) — review_gates: auto (summarize, log, dispatch; no halt)

Project has >4 features → multi-agent orchestration applies. Waves are file-disjoint where parallel, dependency-coupled where sequential.

- **Wave 0 — Foundation (sequential, integration-critical, authored coherently):** npm-workspace monorepo; `packages/shared` (DTOs, error codes, API contract constants); `apps/api` bootstrap (zod env config, pino, Helmet/CORS, throttler, problem-details filter INV-10, nestjs-cls + TenantTransactionInterceptor INV-1, base TypeORM DataSource + migrations incl. EnableRls INV-3, assertResourceAccess INV-2); docker-compose (postgres·redis·api·worker·web·caddy) + Caddyfile (TLS/HSTS/redirect, header buffers); `scripts/invariant-lint.mjs` INV-16.
- **Wave 1 — Auth & tenancy (sequential after W0):** F-01 Auth (register/login/refresh/logout/forgot/reset; argon2; HIBP screening; anti-enum shape+timing; rate-limit-before-hash INV-9; refresh rotation) · F-02 Users/RBAC/admin · F-03 Audit logging. Tables: tenants, users (UNIQUE tenant_id,email INV-7), refresh_tokens, audit_logs.
- **Wave 2 — Ticketing core (parallel; file-disjoint modules):** F-04 Tickets CRUD + state machine + optimistic lock (INV-4) + object-level authz (404 non-owner) · F-05 Comments (internal/public role-gated) · F-06 Attachments (magic-byte validation, quota, randomized key, ownership-gated download INV-14, scan_status) · F-07 SLA evaluator (server-side, repeatable job, pause/resume) · F-08 Categories/SLA policies + ticket_events timeline.
- **Wave 3 — Async & integrations (parallel; disjoint):** F-09 Outbox email (pending_emails UNIQUE INV-5, Resend stub driver, drain worker, DLQ) + notifications · F-10 Inbound webhooks (verify-before-read INV-8, webhook_events UNIQUE INV-6) · F-11 Realtime gateway (JWT handshake, per-room authz) · F-12 Dashboard metrics + Reports export (PDF/CSV/JSON async).
- **Wave 4 — Frontend (parallel; each route file-disjoint, dispatched to subagents against fixed API contracts):** F-13 Marketing landing (port DESIGN-TEMPLATE.html, copied :root tokens INV-12) · F-14 Auth screens · F-15 Requester portal (W1/W3) · F-16 Agent queue + detail (W2) · F-17 Manager dashboard + reports (W4) · F-18 Admin + notifications + app shell/theme.

Sequencing rationale: W0→W1 sequential (everything depends on env/tenant/auth foundation). W2 depends on W1 (auth/tenant). W3 depends on W2 (tickets emit events). W4 depends on W2/W3 API contracts being fixed; dispatched to parallel subagents once contracts frozen. Estimated ~18 features across 5 waves; backend ~5–6k LOC, frontend ~3–4k LOC, ~120 tests.

Per-feature bundle enumeration: each F-NN above is a single vertical-slice feature with its own endpoints/tables/screens enumerated in SPEC §3/§6/§8; no range-bundling. Banner emitted per feature.

Gate outcome (review_gates: auto): summarized, logged here, dispatching automatically — no approval halt.

### ADR-12 — Security Audit dispositions (Stage 3 audit gate)
- F1 file-type ASF DoS (MEDIUM) → FIXED: removed `file-type`, replaced with `apps/api/src/storage/magic-bytes.ts` (allowlist-scoped sniffer). VERSION 0.1.0→0.1.1.
- F2 postcss XSS-via-stringify (MEDIUM) → ACCEPTED w/ rationale: build-time transitive under Next.js; processes only trusted authored CSS; no untrusted-CSS path. Tracked until Next bumps bundled postcss.
- F3 dev secrets in compose (LOW) → ACCEPTED: secrets from env (SEC-13); compose values are dev defaults; .env.example documents prod replacement.
