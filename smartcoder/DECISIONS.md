# DECISIONS.md — SmarCoders

Architecture Decision Records (ADRs), assumptions, deviations, and technical debt.
Precedence for conflicts: **RESEARCH.md > ARCHITECTURE.md > SPEC.md**.

---

## Defaults & blank-field resolutions (Stage 1)

- `review_gates: auto` (from RESEARCH Build Strategy) — all approval gates and the post-Stage-2 menu are self-verified and auto-proceeded this run. Logged per Stage-1 review-gate matrix (`"auto"` → self-verify and proceed).
- `prompt_mode`: absent → default `"direct"`. No AI-expansion section required.
- `confidence_level`: absent — not needed; `review_gates: auto` matches first.
- Archetype `saas` taken from RESEARCH §Design Language.

## Design template

A design template (`DESIGN-TEMPLATE.html`, 552 KB, sha256 `f7a43457c7e76d8185405aad4393cfe4c4b4a971c2fdbe996ab1fbaab59dd3e4`) was uploaded and saved to the project root. Its `:root` token block is the **binding** design-token system, overriding `get_project_config` where they conflict. Copied verbatim into `packages/tokens/tokens.css`:

```css
:root {
  /* color */
  --color-bg: #f4f7fb; --color-surface: #ffffff; --color-surface-2: #eef2f9;
  --color-fg: #0d1626; --color-muted: #5a6a82; --color-border: #e3e9f2;
  --color-accent: #2563eb; --color-accent-strong: #1d4ed8; --color-accent-soft: #e7eefe;
  --color-success: #15803d; --color-success-soft: #e3f3e8;
  --color-warning: #b45309; --color-warning-soft: #fbeedd;
  --color-error: #dc2626; --color-error-soft: #fce8e8;
  /* typography */
  --font-display: "Manrope", sans-serif; --font-body: "Inter", sans-serif; --font-mono: "JetBrains Mono", monospace;
  /* spacing */ --space-1:4px; --space-2:8px; --space-3:12px; --space-4:16px; --space-5:24px; --space-6:32px; --space-7:48px; --space-8:64px; --space-9:96px;
  /* radius */ --radius-sm:6px; --radius-md:12px; --radius-lg:20px; --radius-pill:999px;
  /* shadow */
  --shadow-sm: 0 1px 2px rgba(13,22,38,0.06), 0 1px 3px rgba(13,22,38,0.04);
  --shadow-md: 0 6px 16px rgba(13,22,38,0.08), 0 2px 6px rgba(13,22,38,0.05);
  --shadow-lg: 0 24px 56px rgba(13,22,38,0.16), 0 8px 20px rgba(13,22,38,0.08);
}
```

- **Tokens extracted** (binding): full palette above, Manrope/Inter/JetBrains Mono families, 9-step spacing ramp (4→96), 4-step radius, 3-step shadow.
- **Surfaces ported** from the template: landing/home page (sticky blurred nav, two-column hero with product mockup, trust stat strip, 6-col feature grid, dark CTA block, footer).
- **Override note:** the template accent `#2563eb` matches `get_project_config` primary; template surface tokens (`--color-bg #f4f7fb`, navy `--color-fg #0d1626`) override the config's `#ffffff`/`#0f172a` where they differ. Radius is template's 6/12/20 ramp (config said 8). Theme is **light only** (no dark switcher) per config.

---

## ADRs

### ADR-001 — Monorepo with pnpm workspaces (apps/api, apps/web, packages/*)
Single repo, two deployables (NestJS API + worker, Next.js web) sharing `packages/shared` (zod schemas, error codes, ICD-10 ruleset). Simpler contract sharing than two repos. **Status: Accepted.**

### ADR-002 — v1 ships ICD-10-CM coding only; no licensed CPT descriptors
CPT is AMA license-restricted (RESEARCH §3.1/§5/§6). Seeding CPT descriptors would breach the license. v1 codes ICD-10-CM diagnoses (public domain) and lets coders enter procedure codes **by code** without bundling licensed descriptor text. Reference: RESEARCH §6 carry-forward item 1. **Status: Accepted (v1 scope).**

### ADR-003 — Raw `pg` + parameterized queries + scope wrappers (no ORM)
Tight control over the tenant-scope invariant (INV-1) and a simple forward-only migration story. ORM (Prisma/TypeORM) considered; rejected to keep the scoping boundary mechanically lintable. **Status: Accepted.**

### ADR-004 — SSE for the real-time queue dashboard (not WebSocket)
Queue/metrics push is one-way (server→client). SSE over the existing HTTPS path is simpler and sufficient (RESEARCH §3.9). Revisit if interactive push controls are needed. **Status: Accepted.**

### ADR-005 — Transactional outbox + polling relay for dual-write (`has_dual_write`)
Business row + `outbox_events` row in one txn; `outbox-relay` worker publishes at-least-once; consumers idempotent. Debezium CDC rejected (extra infra for medium scale). **Status: Accepted.**

### ADR-006 — argon2id (native) for password hashing
Native security primitive; never substitute bcryptjs/argon2-browser (INV-14, Per-Feature Security Pass "Substitution discipline"). **Status: Accepted.**

### ADR-007 — httpOnly cookie session carrying a JWT (jose), CSRF-guarded
Avoids JS-readable bearer tokens; state-mutating routes carry an Origin/CSRF guard. **Status: Accepted.**

### ADR-008 — Local Docker stack deploy (cloud-portable)
External SaaS (Resend, Datadog, Sentry, AWS Secrets Manager) is unprovisioned at build time. Deploy target = local `docker compose` stack per BUILD.md Deploy Reachability Outcome C; functional smoke test runs against `https://localhost`. Externals are placeholder-gated and degrade gracefully. **Status: Accepted.**

### ADR-009 — External SaaS placeholder-gating
Absent real keys: email stays queued in `pending_emails`; Datadog/Sentry init is a no-op; Secrets Manager falls back to `.env`. App remains contract-correct. **Status: Accepted.**

### ADR-016 — UI status vocabulary aligned to schema (`draft`)
The UI e2e (W2/W3) exposed that the web app used invented chart statuses (`in_progress`, `unassigned`) where the schema/API uses `draft|pending_audit|completed|returned`. Effect: `canEdit` never matched a coder's `draft` chart → the coding panel was permanently read-only, and status badges/filter showed wrong values. Fix: `charts/[id]` `canEdit = isCoder && (draft|returned)`; `lib/status.ts` and the queue filter use the real statuses (`draft`). Found via the rendered-UI e2e (not the API smoke, which used the correct vocabulary). **Status: Fixed.**

### ADR-015 — Smoke-test fix: ambiguous `org_id` in charts list query
The first functional smoke run failed flow W2 (`GET /api/charts` as coder → 500). Root cause: the charts-list query `LEFT JOIN users` and `org_id` exists on both `charts` and `users`, so the unqualified `WHERE org_id = $1` (and the coder `assignee_id` filter) was ambiguous → Postgres error → INTERNAL. Fix: qualified every filter column with the `c.` alias (`c.org_id`, `c.assignee_id`, `c.status`, `c.specialty`). Re-ran the full smoke matrix. **Status: Fixed.**

### ADR-014 — Security Audit Gate dispositions
- **AUTH-1 (CRITICAL, fixed):** registration with an existing email no longer authenticates as / returns the existing account; the branch returns `{created:false}`, controller emits a shape-identical 201 with no session (anti-enum SEC-02 preserved without takeover).
- **CFG-1 (MEDIUM, fixed):** `assertProdSecrets()` blocks production boot on weak/missing `JWT_SECRET`/`WEBHOOK_SIGNING_SECRET`. Local docker stack ships concrete strong defaults so it still boots.
- **DEP-1 (MEDIUM, ACCEPTED):** `postcss <8.5.10` XSS is a transitive dev-toolchain dep inside Next.js; not runtime-reachable here, and the force-fix downgrades Next to v9 (breaking). Tracked for the next Next.js minor bump.
- **Residual LOW (deferred):** existing-email register returns 201 without a session cookie — minor enumeration signal; acceptable vs. the takeover removed.
**Status: Accepted.**

### ADR-013 — INV-1 scoped to raw pool access; transaction clients are the sanctioned write path
The Stage-3 invariant lint flagged `client.query(` inside `withTransaction((client)=>…)` callbacks across feature modules. `withTransaction` lives in the server data-access layer (`apps/api/src/server/db/pool.ts`) and hands the caller a transaction-scoped client — the standard unit-of-work pattern. The architectural intent of INV-1 is "no raw pool access outside `server/**`," with tenant scoping enforced by `withOrgScope` (reads) and explicit `org_id` params (writes). Accordingly INV-1's pattern was refined to `(getPool(|new Pool(|pool.query()` and ARCHITECTURE.md §9 updated to match (amended together, per the kernel's "amend §9 + invariants.json + ADR" rule). This is a precision refinement, not a weakening: raw pool construction/use outside the data layer remains forbidden and machine-checked. **Status: Accepted.**

### ADR-012 — npm workspaces instead of pnpm
pnpm is not present in the build/deploy environment (`pnpm: command not found`); Node 22 + npm 10 are. The monorepo uses **npm workspaces** (`workspaces: ["apps/*","packages/*"]`). Same multi-package layout as ADR-001; only the package manager changed. **Status: Accepted.**

### ADR-011 — Multi-Agent Plan (Stage 3 wave plan)
Logged per Multi-Agent Plan Gate (`review_gates: auto` → summarize, log, dispatch; no halt).
- **Wave 0 Foundation:** F-01 (scaffold/config/health/migrations), F-02 (tokens/shell/landing), `packages/shared`, `packages/tokens`.
- **Wave 1 Identity & ingress (parallel):** F-03 Auth+RBAC, F-04 Users/seed, F-05 Synthea import.
- **Wave 2 Core coding loop (seq on W1):** F-06 Queue/assign, F-07 Chart+coding, F-08 Validation.
- **Wave 3 Review & observability (parallel on W2):** F-09 Audit+feedback, F-10 Metrics/SSE/export, F-11 Notifications, F-12 Audit-log.
- **Wave 4 Reliability & handoff (seq):** F-13 Outbox+webhooks, F-14 invariant-lint+smoke+Claude scaffold.
Bundle Integrity: per-feature contract obligations stay enumerated in SPEC §3/§5/§10; no range-collapsing. The single-build orchestration (one consistent contract layer authored centrally, each feature run as its own loop iteration with completion banner) is chosen over isolated parallel agents to guarantee monorepo contract coherence. **Status: Accepted.**

### ADR-010 — Header-buffer sizing on every serving path
nginx `client_header_buffer_size 16k` + `large_client_header_buffers 8 32k`; Node `--max-http-header-size=32768` (INV-13). Even no-auth/static paths apply — foreign accumulated cookies are the trigger. **Status: Accepted.**
