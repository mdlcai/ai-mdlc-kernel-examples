# DECISIONS.md — Pulse

Architecture Decision Records (ADRs), assumptions, deviations, technical debt, and gate outcomes.
Source-of-truth precedence: **RESEARCH.md > ARCHITECTURE.md > SPEC.md**.

---

## ADR-0001 — Build Strategy defaults
- `build_depth: comprehensive` (explicit). `review_gates: auto` (explicit). `prompt_mode` blank → default `direct`. `confidence_level` absent.
- Disposition: comprehensive depth applied across all stages (exhaustive spec, threat model, per-feature security+design passes, full gate suite). Autonomous run — all approval gates and the post-Stage-2 menu suppressed per `review_gates: auto`.

## ADR-0002 — Design Template tokens copied verbatim (binding)
- `get_design_template` returned a hit. Raw template saved to project root as `DESIGN-TEMPLATE.html`.
- The template's `:root` custom properties are copied verbatim into the token layer (`apps/web/src/styles/tokens.css`) and are the canonical design-token system, overriding `get_project_config` where they conflict.
- Copied `:root` block (verbatim):
```css
:root {
  /* color */
  --color-bg:          #f5f7fb;
  --color-surface:     #ffffff;
  --color-surface-2:   #fbfcfe;
  --color-fg:          #0d1526;
  --color-muted:       #5b6b85;
  --color-border:      #e4e9f2;
  --color-accent:      #2563eb;
  --color-accent-2:    #1d4ed8;
  --color-accent-soft: #eaf1ff;
  --color-success:     #15a35b;
  --color-warning:     #d98307;
  --color-error:       #dc2b3d;
  /* surface for the deep hero panel */
  --color-ink:         #0a1020;
  --color-ink-2:       #111a30;
  --color-ink-line:    #1e2a45;
  --color-ink-muted:   #8a99b8;
  /* typography */
  --font-display: "Manrope", sans-serif;
  --font-body:    "Inter", sans-serif;
  --font-mono:    "JetBrains Mono", monospace;
  /* spacing scale */
  --space-1: 4px;  --space-2: 8px;  --space-3: 12px;
  --space-4: 16px; --space-5: 24px; --space-6: 32px;
  --space-7: 48px; --space-8: 64px; --space-9: 96px;
  /* radius */
  --radius-sm: 6px; --radius-md: 12px; --radius-lg: 20px; --radius-pill: 999px;
  /* shadow */
  --shadow-sm: 0 1px 2px rgba(13, 21, 38, .05), 0 1px 1px rgba(13, 21, 38, .04);
  --shadow-md: 0 8px 24px -8px rgba(13, 21, 38, .14), 0 2px 6px rgba(13, 21, 38, .05);
  --shadow-lg: 0 40px 80px -32px rgba(10, 16, 32, .45), 0 12px 32px -12px rgba(13, 21, 38, .22);
}
```
- Surfaces ported from the template: the marketing landing page (nav, hero with the animated monitoring console, trust strip, bento feature grid, CTA band, footer). Authenticated SaaS app-shell screens express the SAME tokens (light base + ink panels, blue accent, Manrope/Inter/JetBrains Mono).
- Fonts: Manrope (500/600/700/800), Inter (400/500/600), JetBrains Mono (400/500/600), `font-display: swap`, preconnect to Google Fonts.
- Theme: **light only** (project config `theme.mode: light`, no switcher). Dark-mode tokens from `get_project_config` are intentionally NOT shipped as a switcher; the template's ink panels provide dark focal contrast.

## ADR-0003 — Version posture (current-major with documented fallback)
- RESEARCH §7 recommends current majors (Node 24, Express 5, React 19, Vite, Tailwind, PG18). We target current stable majors that install cleanly in the build environment. Where a current major carries migration friction inside a time-boxed autonomous build, the immediately-prior stable major is an acceptable documented fallback. PostgreSQL 16 image satisfies the `database_preference: PostgreSQL` constraint; metrics table is partition-ready (TimescaleDB optional, not required to run).

## ADR-0004 — Archetype
- `Archetype: saas` declared explicitly in RESEARCH §Design Language — no inference required. DESIGN.md Part II §saas is binding (app shell, composed dashboard hierarchy not a tile grid, designed tables/forms, one decisive accent, moderate functional motion). Universal floor (Part I) applies on top.

## ADR-0005 — Auth model: server-side sessions + httpOnly cookie
- No `auth_model` specified. Decision: opaque server-side session tokens stored hashed in `sessions`, delivered via `Secure; HttpOnly; SameSite=Lax` cookie. Rationale: SOC2/OWASP — revocable, no token-in-JS XSS exposure, simpler CSRF posture with SameSite + Origin check. Socket.IO handshake authenticated by the same cookie. Email is globally unique (one user ↔ one org in MVP) so login is a single lookup — avoids a cross-tenant scan.

## ADR-0006 — OS metric ingestion is agentless-first via authenticated push
- RESEARCH gap #1 resolved: CPU/mem/disk arrive at `POST /v1/ingest/metrics` authenticated by an org-scoped API key (hashed at rest). A lightweight agent OR an agentless collector can post. No agent is shipped in MVP scope; the endpoint + API-key model is.

## ADR-0007 — MS Teams via Power Automate Workflows webhook
- RESEARCH gap #2: O365 connector webhooks retired May 2026. Teams channel type posts a MessageCard JSON payload to a Power Automate Workflows webhook URL (stored as an encrypted channel secret).

## ADR-0008 — Notification idempotency + guaranteed delivery
- Each (incident, channel, state-transition) produces exactly one `notifications` row keyed by a deterministic `idempotency_key` with a UNIQUE constraint (INV-2). Delivery uses BullMQ `attempts` + exponential backoff; exhausted attempts → `status = 'dead'` (dead-letter) and an audit-logged failure, never a silent drop. Retries are safe because the unique key + status guard prevent double-send.

## ADR-0009 — Channel secret encryption at rest
- Notification-channel secrets (Slack URL, Teams URL, webhook signing secret, Twilio details) are encrypted with AES-256-GCM using a key derived from `SECRET_ENCRYPTION_KEY` (env). Stored as ciphertext+iv+tag in `notification_channels.config`. Never logged (Pino redaction).

## Gate outcomes
- **Stage 1 review gate:** `review_gates: auto` → self-verify and proceed (matched row: auto/any). Logged here.
- (subsequent gate outcomes appended as the pipeline runs)

## ADR-0010 — Multi-Agent Plan Gate (Stage 3)
- 13 features → multi-agent orchestration. Waves: W0 foundation (shared/db/docker/ci, main loop, sequential barrier); W1 backend API modules (apps/api, main loop); W2 worker (apps/worker, parallel agent); W3 frontend + e2e (apps/web, parallel agent). Waves 2/3 are file-disjoint and run against the frozen shared+db+API contract.
- Bundle integrity: each feature's endpoints/tables/screens enumerated in SPEC §6/§2/§UI Surface (flat, scannable) — no collapsed ranges.
- review_gates: auto → dispatched automatically, no approval halt.
- Native-dep decision: use @node-rs/argon2 (prebuilt, no node-gyp) instead of argon2 to avoid container build friction (ADR-0011).

## ADR-0011 — @node-rs/argon2 over argon2
- argon2 (node-gyp) build is fragile in slim/alpine containers. @node-rs/argon2 ships prebuilt binaries, same Argon2id primitive — not a security downgrade (both Argon2id). Logged per Per-Feature Security Pass "substitution discipline".

## ADR-0012 — Register-conflict deviation from strict anti-enumeration
- `POST /v1/auth/register` returns 409 for an existing email (onboarding clarity) instead of a uniform shape. Compensating controls: login is fully anti-enum (single envelope, timing-padded), per-(ip,email) rate-limited, lockout after 5 failures. ACCEPTED product decision; surfaced as ⚠ in COMPLIANCE.md.

## ADR-0013 — Accept 2 dev-tooling MODERATE advisories (mitigated)
- Residual `npm audit` MODERATEs are a nested vite/esbuild under @vitejs/plugin-react (dev-server path-traversal/SSRF). The vite dev server never runs in production; deployed artifacts ship no executing vite/esbuild (web = nginx static build output; api/worker images run `npm prune --omit=dev`). ACCEPTED with this mitigation. Re-evaluate on plugin-react/vite upgrade.

## Gate outcome — Security Audit Gate
- Pass 1: 7 findings (1 CRITICAL, 2 HIGH, 4 MODERATE). Auto-remediated: drizzle-orm 0.45.2, nodemailer 8.0.10, vitest 4.1.8, web vite 7. Pass 3 residual: 0 CRITICAL / 0 HIGH / 2 MODERATE (dev-only, ACCEPTED). VERSION → 0.1.1. Verdict PASS. No regression (smoke 21/21).
