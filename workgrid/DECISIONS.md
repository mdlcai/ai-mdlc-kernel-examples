# DECISIONS.md — WorkGrid

Architectural decisions, assumptions, deviations, and technical debt. Each ADR is
tagged `ADR-NNNN` and cross-referenced from `ARCHITECTURE.md` §9, `invariants.json`,
per-feature banners, and gate outcomes.

---

## ADR-0001 — Build Strategy defaults
- **Context:** `RESEARCH.md` Build Strategy: `build_depth: comprehensive`, `review_gates: auto`, `force_research: false`. `prompt_mode` is absent.
- **Decision:** Honor `build_depth: comprehensive` (exhaustive spec, threat model, per-feature security + architecture review, full smoke matrix, `DESIGN-NOTES.md`). Run fully autonomous (`review_gates: auto`) — all approval gates self-verify and proceed; the post-Stage-2 menu is suppressed. `prompt_mode` defaults to `direct`.
- **Rationale:** Matches the outer build prompt (autonomous mode) and the RESEARCH header.

## ADR-0002 — Design tokens copied verbatim from uploaded template
- **Context:** `get_design_template` returned a hit ("WorkGrid (standalone).html", sha256 `2f57eaa6…`). The kernel + DESIGN.md require the template's `:root` tokens to be copied verbatim as the canonical token set, overriding `get_project_config` where they conflict.
- **Decision:** Adopt the template `:root` block as the canonical design-token layer (CSS variables + Tailwind theme). The landing/home surface is ported from the template markup. Copied `:root` block (light):
  ```css
  :root {
    --color-bg:#f6f8fb; --color-surface:#ffffff; --color-surface-2:#eef2f8;
    --color-fg:#0d1526; --color-muted:#5a6679; --color-border:#e3e8f0;
    --color-accent:#2563eb; --color-accent-strong:#1d4fd7; --color-accent-soft:#e8effe;
    --color-success:#15885a; --color-warning:#b46710; --color-error:#cc3340;
    --font-display:"Space Grotesk",system-ui,sans-serif;
    --font-body:"Inter",system-ui,sans-serif;
    --font-mono:"JetBrains Mono",ui-monospace,monospace;
    --space-1:4px; --space-2:8px; --space-3:12px; --space-4:16px; --space-5:24px;
    --space-6:32px; --space-7:48px; --space-8:64px; --space-9:96px;
    --radius-sm:5px; --radius-md:10px; --radius-lg:18px;
    --shadow-sm:0 1px 2px rgba(13,21,38,0.05);
    --shadow-md:0 8px 24px -10px rgba(13,21,38,0.16);
    --shadow-lg:0 40px 80px -32px rgba(13,21,38,0.30);
    --maxw:1200px; --ring:0 0 0 3px var(--color-accent-soft);
  }
  ```
  Dark overrides (`[data-theme="dark"]`): `--color-bg:#080b12; --color-surface:#10141d; --color-surface-2:#161c28; --color-fg:#eaf0f8; --color-muted:#94a1b6; --color-border:#222a38; --color-accent:#3b82f6; --color-accent-strong:#2f6ff0; --color-accent-soft:#14233f; --color-success:#34c98a; --color-warning:#e0a44a; --color-error:#f0717c;` (shadows/ring darkened).
- **Rationale:** Template tokens are an explicit brand decision and outrank archetype/config defaults (DESIGN.md precedence). Fonts: Space Grotesk (display), Inter (body), JetBrains Mono (mono).
- **Note:** Accessibility floor (WCAG 2.2 AA) is never waived — any template token failing AA contrast is adjusted the minimum needed and logged.

## ADR-0003 — Pinned stable library lines over newest-major
- **Context:** Newest majors exist for TypeORM (1.0) and Tailwind (4.x).
- **Decision:** Pin TypeORM to the proven 0.3.x line and Tailwind to 3.4.x for shadcn/ui ecosystem compatibility; NestJS 11, Next.js 16, BullMQ 5, socket.io 4.8, Resend 6.
- **Rationale:** Reduces integration risk on a large multi-package build; all required APIs (`@VersionColumn`, migrations, RLS session GUC, shadcn generators) are stable on these lines. Lockfiles committed.

## ADR-0004 — Monorepo layout (npm workspaces)
- **Context:** Next.js frontend + NestJS backend + shared types, plus a worker process.
- **Decision:** npm-workspaces monorepo: `apps/api` (NestJS, also hosts the Bull worker via a worker entrypoint), `apps/web` (Next.js), `packages/shared` (shared TypeScript contracts: DTO/response types, error codes, enums).
- **Rationale:** Shared contract types enforce frontend↔backend alignment (Interface Contract Validation); single install/build graph.

## ADR-0005 — Tenant isolation: RLS + per-request GUC, plus scoped service layer
- **Context:** `multi_tenant: true`, scale 50k+, SOC2.
- **Decision:** Defense-in-depth. (1) Every tenant table has `tenant_id uuid not null`; Postgres RLS `ENABLE`+`FORCE` with policy `tenant_id = current_setting('app.current_tenant_id')::uuid`. (2) A per-request interceptor opens a transaction-scoped connection and `SET LOCAL app.current_tenant_id`. (3) All scoped DB access goes through `apps/api/src/common/tenant/*` (the `TenantScope` helper / scoped repositories) — direct `dataSource.getRepository()` / `manager` calls to tenant tables outside that boundary are forbidden (INV-1).
- **Rationale:** DB enforces isolation even when app code forgets a filter; the service boundary keeps the GUC contract honest.

## ADR-0006 — Auth model
- **Context:** `auth_model` not specified.
- **Decision:** Email + password (Argon2id, NIST 800-63B), stateless access JWT (short TTL) + rotating refresh token stored hashed in DB; HTTP-only secure cookies. Anti-enumeration on signup/login (uniform responses + per-IP+email rate limit before hash). Default first user of a new org becomes `owner`.
- **Rationale:** Self-contained, no external IdP dependency for the core build; SSO is a documented future extension.

## ADR-0007 — Dual-write via transactional outbox
- **Context:** Domain signal `has_dual_write`; webhooks to GitHub/Slack and email via Resend must not be lost or phantom-sent.
- **Decision:** All external side effects (outbound webhooks, emails, notifications) are written as `outbox_event` rows inside the same DB transaction as the domain change. A BullMQ relay worker polls/consumes the outbox and delivers with idempotency keys, retries, and a dead-letter state. Inbound webhooks verify HMAC over the raw body **before** any DB read (INV-4).
- **Rationale:** Eliminates dual-write inconsistency; provides at-least-once delivery with idempotent consumers.

## ADR-0008 — Deploy target = local Docker stack (externals unprovisioned)
- **Context:** No cloud target/credentials configured in this environment; Resend/GitHub/Slack secrets are placeholders.
- **Decision:** Per BUILD.md "Deploy Target Reachability" Outcome C, ship a local-stack deploy via `docker compose up`; run the Functional Smoke Test against `http://localhost`. External integrations (Resend/GitHub/Slack outbound) run in a no-op/log adapter when their env vars are placeholders, so the outbox + workflows are still exercised end-to-end locally.
- **Rationale:** Produces a genuinely runnable, smoke-tested deploy without fabricating a cloud URL. QUICKSTART.md documents the cloud upgrade path.

## ADR-0009 — Protocol: HTTPS only
- **Context:** `protocol_support: "HTTPS only"`.
- **Decision:** Production deploy terminates TLS at an nginx reverse proxy with HTTP→HTTPS 301 redirect and HSTS; certs via Let's Encrypt/Certbot (managed) or mkcert for local dev. helmet sets HSTS at the app. Local dev compose exposes the app behind nginx with a mkcert/self-signed cert; the smoke test resolves an `https://localhost` URL (with the redirect verified). Documented in QUICKSTART.md §"Protocol & TLS".
- **Rationale:** Honors the hard constraint; plaintext served only for the redirect.

## ADR-0010 — Multi-Agent Plan (Stage 3)
- **Context:** Stage 3 Multi-Agent Plan Gate, `review_gates: auto` → surface + log + dispatch (no halt).
- **Decision:** Build in waves: W0 foundation (workspaces, shared contracts, NestJS+Next bootstrap, Docker/nginx) → W1 auth+tenancy (F1 auth, F2 orgs/members/RBAC, F3 audit+outbox) → W2 work domain (F4 projects/tasks, F5 tickets/SLA, F6 workflows/approvals) → W3 cross-cutting (F7 dashboard, F8 notifications, F9 reports, F10 webhooks, F11 real-time) → W4 UI completion + Stage-4 scaffold (F12 remaining screens, F13 e2e+invariant runner, F14 handoff scaffold). Foundation+auth sequential; W2 modules file-disjoint in dependency order. Per-feature bundle obligations enumerated in each feature's banner/notes.
- **Rationale:** Dependency-ordered, file-disjoint where safe; keeps tenant/RLS contract centralized before domain modules consume it.

## ADR-0011 — Refresh-token JWTs carry a unique `jti`
- **Context:** Functional smoke caught login returning 409 when register+login occurred in the same second: the refresh JWT was signed from `{sub}`+`iat` (second granularity), producing an identical token → identical sha256 → unique-constraint collision on `refresh_tokens.token_hash`.
- **Decision:** Add `jti: crypto.randomUUID()` to the refresh token payload so every issuance is unique. Symptom→fix logged. File: `auth.service.ts`.
- **Rationale:** Correct idempotency/uniqueness behavior under concurrent issuance; no security regression (still hashed, rotated, revocable).

## ADR-0012 — Web `useMe` unwraps the `{ data }` envelope
- **Context:** UI e2e caught every org-scoped screen returning "Organization not found" (404). `apiFetch` returns the full `{ data }` envelope, but `useMe` typed/consumed it as the bare `AuthMeDto`, so `activeOrgId` was always `undefined` → no `X-Org-Id` header → TenantGuard NOT_FOUND.
- **Decision:** `useMe` unwraps `(await apiFetch<{data: AuthMeDto}>('/auth/me')).data`. File: `apps/web/src/lib/auth.ts`.
- **Rationale:** Restores tenant context across the whole authenticated app; aligns the hook's runtime value with its declared type. Found and fixed via the Functional/UI smoke (re-entered Stage 3 for the root cause per BUILD.md failure-handling).

## ADR-0013 — Accept 2 MODERATE transitive `postcss` advisories (build-time only)
- **Context:** `npm audit` (Security Audit Gate) reports 2 MODERATE advisories in `postcss` (CSS stringify XSS), pulled in transitively by Next.js build tooling. The offered fix downgrades Next to v9 (breaking).
- **Decision:** ACCEPT/DEFER. postcss runs only at build time over our own first-party CSS — there is no runtime path that stringifies untrusted CSS, so the advisory is not exploitable in the shipped product. Track for the next Next.js minor that bumps the transitive `postcss`. Recorded in COMPLIANCE.md SEC-13 (⚠).
- **Rationale:** No CRITICAL/HIGH; MEDIUM dispositioned with rationale per the gate criteria; a breaking framework downgrade is a worse outcome than the accepted build-time risk.

## ADR-0014 — Local-stack deploy on ports 8081/8444
- **Context:** Host ports 80/443/8080/8443 were already bound; nginx could not bind.
- **Decision:** Map nginx to host 8081 (HTTP→HTTPS redirect) and 8444 (HTTPS); `APP_URL`/`CORS_ORIGINS` set to `https://localhost:8444`. Deploy URL the smoke test resolved: `https://localhost:8444`.
- **Rationale:** Outcome C local-stack deploy (ADR-0008); ports are configurable in `docker-compose.yml`.

## Gate outcomes
- Stage 0 research gate: GO (research performed; §3–§7 written). Logged 2026-06-05.
- Stage 1 review gate: `review_gates: auto` → self-verify and proceed (no halt). Build Input Reconciliation completed in REPORT.md.
- Multi-Agent Plan Gate (`auto`): plan logged (ADR-0010), agents dispatched, no halt.
- Security Audit Gate (`auto`): Pass 1 = 2 MEDIUM (transitive postcss) + per-feature passes clean; auto-remediate = MEDIUM dispositioned (ADR-0013), 2 functional bugs found & fixed (ADR-0011/0012); Pass 2 residual = 0 CRITICAL/HIGH, 1 accepted MEDIUM. Proceeded.
- COMPLIANCE.md Gate (`auto`): A=15 ✓ / B=1 ⚠ / C=0 ✗ → auto-proceeded (the ⚠ is the accepted build-time advisory).
- Design Quality Gate (`auto`): 18/18 screens conform; 0 ⚠ / 0 ✗ → auto-proceeded. Template Conformance: tokens match verbatim, fonts match.
- Drift Detection Gate (`ship`): invariant lint 13/13 machine-checkable PASS, 4 manual upheld → 0 drifts → auto-proceeded to deploy.
