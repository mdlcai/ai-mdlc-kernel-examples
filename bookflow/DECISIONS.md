# DECISIONS.md — BookFlow

Assumptions, deviations, technical debt, and ADRs. `SPEC.md` is the behavior source of truth; `ARCHITECTURE.md` is the architecture source of truth.

## Build Strategy resolution
- `build_depth: comprehensive` (explicit) → exhaustive SPEC, full gates, threat model, DESIGN-NOTES.
- `review_gates: auto` (explicit, reinforced by outer prompt) → all gates self-verify and auto-proceed; no human halts except hard errors / NO-GO.
- `prompt_mode`: not set → default `direct`. Logged.
- `confidence_level`: not set → with `review_gates: auto`, auto path applies regardless.

## ADR-0001 — Tenant isolation: row-level `org_id` + CLS scoped repository
**Status:** Accepted. **Context:** multi_tenant=true, scale=small (<1k concurrent). **Decision:** every tenant-owned table carries `org_id`; a `ScopedRepository` reads `org_id` from `nestjs-cls` request context and injects it into every query. **Alternatives:** schema-per-tenant (operational overhead, overkill at this tier), Postgres RLS (deferred as future defense-in-depth). **Consequences:** INV-1, INV-14 enforce it; isolation tests required.

## ADR-0002 — Design tokens copied verbatim from uploaded Design Template
**Status:** Accepted. **Context:** customer uploaded `DESIGN-TEMPLATE.html`; DESIGN.md Part III + outer prompt make its `:root` tokens canonical, overriding `get_project_config`. **Decision:** copy the template `:root` block verbatim into `apps/web/app/globals.css`. The exact copied block:
```css
:root {
  --color-bg:          #f4f4f3;
  --color-surface:     #ffffff;
  --color-surface-2:   #ebebe9;
  --color-fg:          #111111;
  --color-muted:       #5c5c5c;
  --color-border:      #dcdcd9;
  --color-accent:      #000000;
  --color-accent-soft: #e6e6e3;
  --color-success:     #1c7c4a;
  --color-warning:     #9a6400;
  --color-error:       #b3261e;
  --font-display: "IBM Plex Sans", sans-serif;
  --font-body:    "Geist", sans-serif;
  --font-mono:    "Geist Mono", monospace;
  --space-1: 4px;  --space-2: 8px;  --space-3: 12px; --space-4: 16px;
  --space-5: 24px; --space-6: 32px; --space-7: 48px; --space-8: 64px;
  --radius-sm: 4px; --radius-md: 6px; --radius-lg: 10px;
  --shadow-sm: 0 1px 0 rgba(17,17,17,0.04);
  --shadow-md: 0 1px 2px rgba(17,17,17,0.06), 0 4px 12px rgba(17,17,17,0.04);
  --shadow-lg: 0 8px 28px rgba(17,17,17,0.10);
}
```
**Note:** these override the `get_project_config` palette (which had accent `#000000` too, but different bg/surface/semantic hexes). Template wins per precedence.

## ADR-0003 — Dark mode derived alongside template tokens
**Status:** Accepted. **Context:** template is light-only; config requests a light+dark toggle. DESIGN.md allows *deriving* a token the template did not declare, never *replacing* one it set. **Decision:** add a `.dark` token block deriving a dark palette (deep neutral surfaces, lightened fg, AA-safe semantics) alongside — light `:root` values remain verbatim. INV-11 conformance checks only the light `:root`.

## ADR-0004 — No-double-booking via Postgres EXCLUDE constraint
**Status:** Accepted. **Decision:** `EXCLUDE USING gist (resource_id WITH =, tstzrange(starts_at, ends_at, '[)') WITH &&) WHERE (status NOT IN ('cancelled','rejected'))` on `bookings` (requires `btree_gist`). App-level availability check is UX-only; the constraint is the guarantee. Constraint violations map to `409 booking_conflict`. (INV-2, ADR for `has_dual_write`/booking write boundary.)

## ADR-0005 — Transactional outbox for email + outbound webhooks
**Status:** Accepted. **Context:** `notification_urgency: guaranteed delivery`, `has_dual_write`. **Decision:** notifications insert an `outbox` row in the same tx as the business change; a worker drains it (`FOR UPDATE SKIP LOCKED`), sends via Resend, marks sent; failure → exponential backoff → DLQ flag. No sync send in request path (INV-3).

## ADR-0006 — SSE over WebSocket for realtime
**Status:** Accepted. **Context:** `realtime_needed: true`, one-way push (availability/approval updates), ≤1k concurrent. **Decision:** NestJS `@Sse()` `/v1/stream` per-tenant channel. WS upgrade path documented if bidirectional needed later.

## ADR-0007 — WCAG AA enforced floor (config requested AAA)
**Status:** Accepted. **Context:** config `wcagLevel: AAA`; DESIGN.md floor is AA. **Decision:** enforce AA as the hard floor (the gate criterion), target AAA contrast where the token palette allows (the accent `#000` on `#fff` is AAA). Pure-black accent gives very high contrast, easing AAA on primary text/actions.

## ADR-0008 — Pin NestJS v11, TypeORM 0.3.x, Tailwind v3
**Status:** Accepted. **Context:** SOC2 build favors stability; NestJS v12 (ESM) and bleeding-edge majors add migration risk. **Decision:** pin to stable, widely-deployed lines. Logged as RESEARCH §4 note.

## ADR-0009 — Deploy as local docker-compose stack (Outcome C)
**Status:** Accepted. **Context:** no cloud target or external service credentials provisioned in this environment. **Decision:** Stage 4/ship deploy is a local `docker compose up` stack (postgres + api + worker + web); functional smoke test runs against `http://localhost`. Resend is exercised in **contract/stub** mode (no real API key) — emails enqueue to outbox and a stub transport logs them; QUICKSTART documents switching to real Resend. Cloud upgrade path documented.

## ADR-0010 — Multi-Agent Plan Gate: orchestrator-sequential build
**Status:** Accepted (review_gates: auto → logged, dispatched, no halt). **Context:** every feature module integrates against shared Wave-1 base classes (ScopedRepository, guards, problem+json filter); fanning out independent code-writing agents risks non-compiling guesses against the shared base. **Decision:** the orchestrator builds waves sequentially with file-disjoint module ownership and compiles after each slice (integrated `tsc`/`nest build` is the verification gate). Waves: (1) foundation+common+migrations, (2) auth+web shell+landing, (3) resources/bookings/workflows+approvals, (4) outbox/worker/webhooks, (5) audit/reports/SSE/members/dashboard. ~12 vertical-slice features. Plan logged per Multi-Agent Plan Gate.

## ADR-0011 — bcryptjs (pure-JS) instead of native bcrypt
**Status:** Accepted (substitution logged per BUILD.md substitution-discipline). **Context:** SPEC SEC-3 specifies bcrypt cost ≥12. Native `bcrypt` requires node-gyp/native compilation that is brittle in Alpine multi-stage Docker builds. **Decision:** use `bcryptjs`, a pure-JavaScript implementation of the **same bcrypt algorithm** (Blowfish-based, identical `$2a$`/`$2b$` hashes, same cost factor). This is NOT a downgrade of the security primitive (same algorithm + cost 12), only a build-portability choice; it is explicitly distinct from the prohibited "swap to a weaker primitive". Cost remains 12. If native performance is later required, swap back with no hash-format change (compatible).

## ADR-0012 — Outbox worker hardened against placeholder/whitespace Resend keys
**Status:** Accepted. **Context:** the Functional Smoke Test surfaced a runtime-only failure — the worker tried the real Resend API and got "API key is invalid". Root cause: `.env.example` had an **inline comment** after `RESEND_API_KEY=`, so dotenv/compose passed `"   # re_REPLACE_ME…"` as a non-empty key. **Decision:** (a) moved inline comments off all secret lines in `.env.example`; (b) hardened `resend.client.ts` to treat missing/whitespace/placeholder keys (and any key not starting with `re_`) as "no key → stub transport". This is exactly the class of bug the functional smoke test exists to catch (passes build/config, fails at first real send). After the fix the outbox drains cleanly (all rows `sent`, pending=0).

## Defaults applied for blank fields
- `prompt_mode` blank → `direct`.
- `confidence_level` blank → moot under `review_gates: auto`.
- `data_migration` blank → greenfield, TypeORM migrations from empty schema.
- `offline_capable` not set → online-only.
