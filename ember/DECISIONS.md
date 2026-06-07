# DECISIONS.md — Ember

Architectural Decision Records (ADRs): assumptions, blank-field defaults, deviations,
gate outcomes, dependency additions, substitutions, and acknowledgments. Precedence:
RESEARCH > ARCHITECTURE > SPEC.

---

## ADR-001 — Auth model (blank field default)
**Context:** `auth_model` not specified in RESEARCH.md.
**Decision:** First-party email/password accounts with server-side **session cookies**
(HttpOnly, Secure, SameSite=Lax) and **Argon2id** password hashing. No third-party IdP / social
login / magic links in v1.
**Why:** Simplest safe default for a storefront; sessions are revocable; Argon2id meets NIST
SP 800-63B-4. **Rationale logged**; revisit if SSO is requested.

## ADR-002 — Deployment topology (blank field default)
**Context:** `deployment` not specified.
**Decision:** Docker Compose multi-service stack: `proxy` (Caddy), `web` (Next.js), `api`
(Express), `worker` (outbox drain), `db` (Postgres 18), `redis`. Local-stack capable; cloud via
the same compose + managed Postgres/Redis when provisioned.
**Why:** Matches `ci_cd_required`, `target_platforms: web`, and `scale: medium`.

## ADR-003 — Backend language (derived)
**Context:** `backend_language` blank; `backend_framework: Express`.
**Decision:** **TypeScript on Node.js** (Express 5). Shared types with the Next.js frontend.

## ADR-004 — Data migration strategy (blank field default)
**Context:** `data_migration` blank; greenfield project.
**Decision:** **Prisma Migrate**; no legacy data import. Seed script for demo catalog.

## ADR-005 — TLS / reverse proxy (resolves RESEARCH §6.3)
**Context:** `protocol_support: HTTPS only`; choice of proxy left open in research.
**Decision:** **Caddy** — automatic HTTPS (Let's Encrypt) in prod, **mkcert** self-signed for
local dev; forced HTTP→HTTPS 301; HSTS `max-age=63072000; includeSubDomains; preload`.
**Why:** Least-config path to the HTTPS-only + HSTS requirement (RESEARCH §3.5).

## ADR-006 — Card capture = Stripe hosted Checkout (not Elements)
**Context:** PCI baseline; SAQ A eligibility (RESEARCH §3.5/§3.8).
**Decision:** All card entry in **Stripe-hosted Checkout Sessions**; no embedded Elements / custom
PAN fields in v1. **Why:** Keeps card data off Ember servers → SAQ A. Embedded Elements deferred.

## ADR-007 — Secrets management (blank field default)
**Context:** `secrets_management` blank.
**Decision:** Environment variables; **`.env` gitignored**, **`.env.example` committed** with
placeholders; no `sk_*`/`whsec_`/`re_` literals in source (INV-7). Production secrets injected by
the host/orchestrator.

## ADR-008 — Design tokens copied verbatim from uploaded template (override config palette)
**Context:** A Design Template (`Ember-Standalone.html`, 499 KB) was uploaded. Per DESIGN.md
Part III, its `:root` tokens are binding and **override** the `get_project_config` palette where
they conflict.
**Decision:** The canonical token layer is copied verbatim from `DESIGN-TEMPLATE.html`:
```
:root {
  --color-bg:#f6efe6; --color-surface:#fffaf3; --color-surface-2:#efe4d4;
  --color-fg:#2a2018; --color-muted:#6f6155; --color-border:#e4d8c7;
  --color-accent:#2d1e09; --color-accent-soft:#b98a4e; --color-cream:#f3e7d3;
  --color-success:#3f7d4e; --color-warning:#b5781f; --color-error:#b4452f;
  --font-display:"Outfit",sans-serif; --font-body:"Inter",sans-serif; --font-mono:"JetBrains Mono",monospace;
  --space-1..9: 4/8/12/16/24/32/48/64/96px;
  --radius-sm:4px; --radius-md:8px; --radius-lg:18px;
  --shadow-sm/md/lg: warm-tinted; --maxw:1200px;
}
[data-theme="dark"] { --color-bg:#16110b; --color-surface:#211910; --color-fg:#f3e9da;
  --color-accent:#e7b97e; --color-muted:#b3a48f; --color-border:#3a2c1a; … }
```
**Conflicts overridden:** config `background #fafafa`→template `#f6efe6`; config `surface #f5f5f4`→
`#fffaf3`; config "shadows: none"→template defines warm shadows; config radius lg 8/xl 12→template
`--radius-lg:18px`; config maxWidth 1280→template `--maxw:1200px`; success/warning/error adopt
template values. Project-config fonts (Outfit/Inter/JetBrains Mono) **agree** with the template.
**Surfaces ported:** landing/home scaffold (sticky nav: Shop/Brew guides/Subscribe/About; hero
"Roasted to order · Shipped at peak" + "Free shipping over $40"; product-preview cards; subscribe
band; about; footer). Raw template saved to `DESIGN-TEMPLATE.html`.

## ADR-009 — No AI-agent runtime in product
**Context:** ARCHITECTURE §2 orchestration section.
**Decision:** Ember is a conventional web app; "orchestration" = compose services + outbox worker
+ Stripe/Resend webhook choreography. No LLM/tool-calling runtime in product scope.

## ADR-010 — Concurrency primitive for stock (resolves RESEARCH §3.1 Prisma gap)
**Context:** Prisma 7 has no native `SELECT … FOR UPDATE` API.
**Decision:** Primary = conditional atomic decrement `UPDATE products SET stock = stock - :qty
WHERE id = :id AND stock >= :qty` (reject on 0 rows) + DB `CHECK (stock >= 0)`. Hot multi-row
read-then-decide uses `tx.$queryRaw` `SELECT … FOR UPDATE … ORDER BY id ASC` inside an interactive
`$transaction`. **Why:** structurally prevents oversell without an ORM lock helper.

---

## ADR-011 — Multi-Agent Plan Gate (Stage 3 dispatch)
**Context:** 12 feature slices; review_gates: auto.
**Decision:** 3-wave plan. **Wave 0** (sequential, foundation): monorepo, `packages/shared`,
Prisma schema+migration+seed, env, compose/Caddy/Dockerfiles (F-01) — fixes all downstream
contracts. **Wave 1** (parallel, file-disjoint): agent **API** owns `apps/api/**`
(F-03/04/05/06/08/09/10/11 backend + tests); agent **WEB** owns `apps/web/**`
(F-02/03/04/05/07/08/10 frontend). **Wave 2** (sequential): integration, invariant-lint runner
(F-12), smoke tests, CI, then verification + gates. **Bundle integrity:** each agent's ADR
enumerates its endpoints/screens/migrations flatly per SPEC §1/§3/§6. Parallel waves are
file-disjoint (`apps/api` vs `apps/web`); foundation is dependency-root → sequential.
**Gate outcome (auto):** logged, dispatched without halt.

## ADR-012 — Prisma major version pinned to 6 (friction fix, Stage 3)
**Context:** Registry latest is Prisma 7.8.0. Prisma 7 removed `url` from the schema datasource and
requires a `prisma.config.ts` + a driver adapter (`@prisma/adapter-pg`) wired into every
`PrismaClient` construction (db, worker, tests) — a breaking config model.
**Decision:** Pin **Prisma 6.19.3** (`@prisma/client` + `prisma`). Schema keeps `url = env("DATABASE_URL")`;
`new PrismaClient()` works unchanged.
**Why:** Satisfies the `orm_preference: Prisma` constraint; major-version selection within the same
ORM is not a security-primitive substitution (Substitution Discipline). Prisma 6 is stable and current
(2025). Avoids burning the build on the v7 adapter migration. Revisit when adopting the v7 adapter
model is worthwhile. **Cascade Impacts:** none (data model, migrations, and all query code identical).

---
*Stage 1 gate outcome (review_gates: auto): self-verified; Build Input Reconciliation complete
with 0 unresolved Conflicts (see REPORT.md). Proceeded to Stage 2 → Stage 3.*
