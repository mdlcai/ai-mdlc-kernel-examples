# DECISIONS.md — ServiceHub

Architecture Decision Records (ADRs), assumptions, deviations, and technical debt.
Referenced as `ADR-NN`. Source-of-truth precedence: RESEARCH > ARCHITECTURE > SPEC.

## Build Strategy (resolved)
- `build_depth: comprehensive` (from RESEARCH.md) — exhaustive spec, per-feature
  security + architecture-adherence reviews, threat model, narrative audit docs.
- `review_gates: auto` — all approval gates and the post-Stage-2 menu are
  suppressed; each stage summarized and the pipeline auto-proceeds. Logged here
  per the kernel's dual/auto gate-logging requirement.
- `prompt_mode`: blank → default `direct`. Logged.
- `force_research: false` — but §3 was absent (fresh dashboard wizard), so
  research ran anyway per the Stage 0 "absent §3" rule.
- `confidence_level`: blank — with `review_gates: auto`, the auto row wins at
  every gate regardless of confidence.

## Design Tokens (binding — from uploaded Design Template)
A design template (`ServiceHub-standalone.html`) was uploaded. Per the
token-source precedence (template `:root` > brand override > archetype defaults),
the template tokens are **binding and override `get_project_config`** wherever
they conflict. **Exact `:root` block copied verbatim:**

```css
:root {
  /* color */
  --color-bg:        #f5f7fb;
  --color-surface:   #ffffff;
  --color-surface-2: #f0f3f9;
  --color-fg:        #0d1526;
  --color-muted:     #5a6b85;
  --color-border:    #e1e7f0;
  --color-accent:    #2563eb;
  --color-accent-ink:#1d4ed8;
  --color-accent-soft:#eaf0ff;
  --color-success:   #15a05a;
  --color-success-soft:#e3f6ec;
  --color-warning:   #c9760a;
  --color-warning-soft:#fcefd9;
  --color-error:     #dc2626;
  --color-error-soft:#fbe4e4;
  /* typography */
  --font-display: "Manrope", sans-serif;
  --font-body:    "Inter", sans-serif;
  --font-mono:    "JetBrains Mono", monospace;
  /* spacing scale */
  --space-1: 4px;  --space-2: 8px;  --space-3: 12px;
  --space-4: 16px; --space-5: 24px; --space-6: 32px;
  --space-7: 48px; --space-8: 64px; --space-9: 96px;
  /* radius */
  --radius-sm: 6px; --radius-md: 12px; --radius-lg: 20px;
  /* shadow */
  --shadow-sm: 0 1px 2px rgba(13, 21, 38, 0.06), 0 1px 1px rgba(13, 21, 38, 0.04);
  --shadow-md: 0 6px 16px -6px rgba(13, 21, 38, 0.12), 0 2px 6px -2px rgba(13, 21, 38, 0.08);
  --shadow-lg: 0 32px 64px -24px rgba(13, 30, 70, 0.28), 0 12px 28px -16px rgba(13, 30, 70, 0.16);
  --maxw: 1200px;
}
```

- **Surfaces ported from the template:** the marketing landing/home surface
  (nav, hero with product mock, trust band, features bento, CTA, footer) →
  `apps/web/src/pages/Landing.tsx`. All authed app screens express these same
  tokens for visual coherence.
- **Fonts** loaded via Google Fonts `<link>` (Manrope, Inter, JetBrains Mono);
  the template inlined them as base64 `@font-face`, which we replace with the
  hosted link for a lighter bundle.

## ADRs

### ADR-01 — Express 5 over Fastify/NestJS
Express 5 is stable and has the richest middleware ecosystem (Helmet,
express-rate-limit, Multer). A layered controller→service→repository structure
gives us NestJS-like organization without the framework weight. Fastify rejected
for smaller middleware ecosystem.

### ADR-02 — TypeORM (constraint-mandated)
`orm_preference: TypeORM` is a hard constraint. `synchronize:false` everywhere;
schema only via reviewed migrations (INV-2).

### ADR-03 — Auth model: JWT access + rotating refresh + Argon2id, optional TOTP
Access token 15 min (Bearer); refresh token rotating, stored httpOnly+Secure
cookie, with reuse detection. Argon2id per OWASP/NIST 800-63B. Optional TOTP MFA
per user. Enterprise SSO (SAML/OIDC) is a documented Non-Goal for v1 (RESEARCH §6).

### ADR-04 — Guaranteed delivery via Transactional Outbox + BullMQ
`notification_urgency: guaranteed delivery` + `webhook_targets` require
at-least-once delivery. Domain change + `outbox_events` row committed atomically;
a relay worker delivers email/webhooks with idempotency keys, retries/backoff, DLQ.

### ADR-05 — Multi-tenancy: org-scope wrapper + Postgres RLS (defense-in-depth)
App-level `withOrgScope` is the primary control; RLS keyed on `app.current_org`
is defense-in-depth (INV-1, INV-11). `scale: medium (1k–50k)` does not warrant
DB-per-tenant; shared-schema with org_id + RLS is the right tier.

### ADR-06 — SLA engine via persisted deadlines + delayed jobs (blank-field resolution)
RESEARCH §6 gap: SLA business-hours rules undefined. **Resolution:** default
severity targets from RESEARCH success metrics (critical <2h, high <8h, medium
<24h, low <72h); per-org business-hours calendar (default 24×7); timer pauses on
`pending_customer`/`pending_vendor` statuses. Absolute deadlines persisted;
breach-warning at 80% + breach handled by delayed BullMQ jobs (idempotent).

### ADR-07 — Email provider-agnostic SMTP (blank-field resolution)
RESEARCH §6 gap: provider unselected. **Resolution:** Nodemailer SMTP transport,
MailHog in the dev/local stack, real provider swappable via `SMTP_*` env. No code
change to switch providers.

### ADR-08 — Inbound monitoring webhooks: generic signed receiver (blank-field resolution)
RESEARCH §6 gap: monitoring payload formats unknown. **Resolution:** a generic
HMAC-signed receiver (`/v1/webhooks/inbound/:source`) with a normalized
alert→ticket mapper; per-source shared secret in env; dedupe by `(source,
external_id)` (INV-4); signature verified before DB (INV-5).

### ADR-09 — Data retention defaults (blank-field resolution, SOC2)
RESEARCH §6 gap: retention window. **Resolution:** tickets retained 1 year
(configurable `TICKET_RETENTION_DAYS`), audit logs 7 years (`AUDIT_RETENTION_DAYS`),
PITR window 7 days. Documented in COMPLIANCE.md.

### ADR-10 — Protocol: HTTPS only (constraint)
`protocol_support: HTTPS only`. TLS terminated at a reverse proxy (Caddy/nginx)
with automatic HTTP→HTTPS redirect + HSTS. Local dev uses mkcert or a self-signed
cert; the smoke test targets the https:// (or local http for the container-internal
healthcheck) URL. Documented in QUICKSTART.md §Protocol & TLS.

### ADR-11 — Frontend state/data: TanStack Query + lightweight Zustand
Server state via TanStack Query (cache, retries, optimistic updates); minimal UI
state via Zustand. Avoids Redux boilerplate for a dashboard SPA.

### ADR-12 — Charts: Recharts
Mature, declarative React charts for the manager/executive dashboards; colors
sourced from the token theme module (INV-9 compliant).

## Gate outcomes
- **Stage 0 research gate:** §3 absent → comprehensive research performed; Go/No-Go = **GO**. Logged.
- **Stage 1 review gate:** `review_gates: auto` → self-verify + proceed (no halt). Build Input Reconciliation has no unresolved Conflict.

## Assumptions summary (blank fields resolved → see ADR-06..ADR-10)
SLA calendar, email provider, monitoring webhook format, retention window, and
MFA/SSO scope were blank in RESEARCH; each resolved above with a sensible default.

## Multi-Agent Plan Gate (review_gates: auto → logged, auto-dispatched)
The build runs as the single model cycling roles across ordered waves. Features
are file-disjoint within a wave where possible; cross-cutting foundation is
sequential. Per the Bundle Integrity Rule, each feature below enumerates its
endpoints/tables/screens in SPEC.md (no collapsed ranges).

- **Wave 0 — Foundation (sequential):** F-01 monorepo, Docker (postgres/redis/
  mailhog), TypeORM DataSource (`synchronize:false`) + RLS plumbing, env
  validation, pino logging, RFC 9457 error contract, `/api/health`.
- **Wave 1 — Data & Auth core (sequential):** F-02 entities + migrations + seed;
  F-03 auth (register/login/refresh/logout/me/MFA) + Argon2id + rate-limit-before-
  hash + anti-enumeration + RBAC + tenancy middleware (RLS `SET LOCAL`); F-04 web
  design system (template tokens) + app shell + login/register screens + landing.
- **Wave 2 — Ticketing domain:** F-05 tickets + lifecycle + numbering; F-06
  comments/timeline + append-only audit + attachments (Multer+sharp EXIF strip);
  F-07 SLA engine + routing + escalation (delayed jobs).
- **Wave 3 — Delivery & integrations:** F-08 notifications (in-app + email outbox)
  + realtime (Socket.IO); F-09 inbound (HMAC-before-DB) + outbound (signed, SSRF-
  guarded, outbox) webhooks.
- **Wave 4 — Analytics & finish:** F-10 manager/executive dashboards + approval
  chains; F-11 PDF/CSV reporting; F-12 observability (Sentry/OTel) + CI + invariant
  -lint runner + Claude continuation scaffold.

Dispatched automatically (no "approve" required under auto gates). Plan logged.
