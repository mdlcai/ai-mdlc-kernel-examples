# DECISIONS.md — PatchLAB

Architecture Decision Records, assumptions, deviations, gate outcomes. Newest context appended over time.

## Build Strategy / gate config
- `build_depth: comprehensive`, `review_gates: auto`, `force_research: false`, `prompt_mode` absent → default `direct` (logged). Archetype `saas`. `confidence_level` absent.
- **Stage 0 gate:** RESEARCH.md had no §3 (fresh dashboard doc) → research performed (comprehensive). §7 Go/No-Go = **GO**. Outcome logged.
- **Stage 1 gate:** `review_gates: auto` → self-verify and proceed (no halt).
- **Stage 2 gate:** `review_gates: auto` → self-verify and proceed.
- **Multi-Agent Plan Gate:** `review_gates: auto` → summarize, log, dispatch automatically (no approval verb).

## Blank-field resolutions (Blank Field Policy)
- **ADR-008 `auth_model` blank → email + password with JWT session.** Key decision field; chosen because the product requires per-user libraries and the RESEARCH workflows assume registered accounts. Argon2id hashing, HttpOnly cookies, refresh rotation. Extra-considered: OAuth social login (deferred — adds provider config, not needed for MVP).
- **ADR-009 `deployment` blank → Docker Compose local-stack (single host).** Aligns with `container_strategy`. Cloud target deferred; QUICKSTART documents the upgrade path.
- **ADR-010 `backend_language` blank → Python** (implied by `backend_framework: FastAPI`, `orm_preference: SQLAlchemy`, `typescript_backend: false`). No conflict.
- **ADR-004 anonymous assets** = capability-URL (unguessable UUID, TTL). Lets the landing "try it" flow work without forcing signup while bounding storage.

## Key ADRs (full rationale)
- **ADR-001 Procedural DSP engine over neural-audio APIs.** Deterministic, offline, zero per-generation cost, no GPU, no training-data/licensing exposure. Cost: lower perceptual fidelity than neural tools (RESEARCH §6 open item). Alternatives: Stability Stable Audio / ElevenLabs SFX / Meta AudioCraft — all rejected for MVP (external dependency, cost, latency, infra), retained as optional future "premium AI" tier.
- **ADR-002 PyJWT + argon2-cffi.** python-jose (last release 2021) and passlib (breaks on Python 3.13+) are unmaintained; FastAPI's own docs moved to PyJWT (RESEARCH §3.1, §3.7).
- **ADR-003 Synchronous generation, no job queue/Redis.** Scale tier = small (<1k concurrent); sub-second DSP renders. Building a queue here violates "build TO the tier" (RESEARCH §scale). DoS bounded by hard caps + rate limits instead. Revisit at medium tier.
- **ADR-005 Local-disk asset store with traversal-safe resolver.** Small scale; S3/presigned URLs deferred (RESEARCH §3.8) and documented as the scale-up path.
- **ADR-006 Alembic migrations**, reversible; unique constraints on `users.email` (INV-5) and `refresh_tokens.jti` (INV-6).
- **ADR-007 Hard resource caps** (`MAX_DURATION_S`, `MAX_SAMPLE_RATE`, `MAX_VARIATIONS`, `MAX_TOTAL_SAMPLES`, `MAX_PACK_ITEMS`, `MAX_PACK_BYTES`) — primary DoS mitigation for CPU-bound generation and ZIP-bomb/storage exhaustion (RESEARCH §5 risks 1, 7, 8).
- **ADR-011 Multi-service Docker Compose** (db/api/web/proxy) rather than one mixed-runtime container — Next.js (Node) and FastAPI (Python) cannot cleanly share one image; still a single `docker compose up`. Reconciles `container_strategy: single container (Docker Compose)` (logged as intentional interpretation, not a Conflict).
- **ADR-012 Protocol = HTTPS only** (Build Constraint `protocol_support`). nginx terminates TLS, forces HTTP→HTTPS 301, sets HSTS. Local dev uses a self-signed/mkcert cert; production uses managed/Let's Encrypt. Smoke test runs against the https origin.

## ADR-013 — Design template tokens copied verbatim (binding)
A design template (`DESIGN-TEMPLATE.html`, `PatchLAB - standalone.html`, sha256 `4613fb72…e9d7a`) was uploaded. Per DESIGN.md precedence, its `:root` tokens are the canonical token set for the whole build, **overriding `get_project_config`** where they conflict (e.g. template accent `#2563eb`, surfaces, teal `#0d9488`/`#2dd4bf`). Copied verbatim into `frontend/src/app/globals.css`:

```css
:root {
  --color-bg: #f6f7f9;
  --color-surface: #ffffff;
  --color-surface-2: #f1f5f9;
  --color-fg: #0f172a;
  --color-muted: #475569;
  --color-border: #e2e8f0;
  --color-accent: #2563eb;
  --color-accent-soft: rgba(37, 99, 235, 0.10);
  --color-teal: #0d9488;
  --color-success: #16a34a;
  --color-warning: #d97706;
  --color-error: #dc2626;
  --font-display: 'Manrope', system-ui, sans-serif;
  --font-body: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', ui-monospace, monospace;
  --space-1: 4px;  --space-2: 8px;  --space-3: 12px;
  --space-4: 16px; --space-5: 24px; --space-6: 32px;
  --space-7: 48px; --space-8: 64px;
  --radius-sm: 2px;
  --radius-md: 4px;
  --radius-lg: 6px;
  --shadow-sm: 0 1px 2px rgba(15, 23, 42, 0.06);
  --shadow-md: 0 2px 6px rgba(15, 23, 42, 0.08);
  --shadow-lg: 0 16px 48px rgba(15, 23, 42, 0.14);
}
[data-theme="dark"] {
  --color-bg: #0a0f1c;
  --color-surface: #111a2e;
  --color-surface-2: #1a2540;
  --color-fg: #f1f5f9;
  --color-muted: #94a3b8;
  --color-border: #2a3956;
  --color-accent: #2563eb;
  --color-accent-soft: rgba(37, 99, 235, 0.14);
  --color-teal: #2dd4bf;
  --color-success: #22c55e;
  --color-warning: #f59e0b;
  --color-error: #ef4444;
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.4);
  --shadow-md: 0 2px 8px rgba(0, 0, 0, 0.45);
  --shadow-lg: 0 24px 64px rgba(0, 0, 0, 0.55);
}
```

- **Fonts** (self-hosted via Google Fonts / next/font): Manrope (display, 500/600/700/800), Inter (body, 400/500/600), JetBrains Mono (mono/eyebrows/labels, 400/500/600).
- **Surfaces ported from template:** sticky nav (logo + nav links + Sign in / Generate free), hero (2-col copy + product mockup), features grid (4 feature cards + full-width sample-pack card), CTA block, footer. These become the `/` landing page.
- **Derived tokens** (added alongside, never replacing template values): none required so far; any new state color will be derived to sit beside the copied palette.
- Default theme: dark is the visually dominant theme (template bundler bg `#0a0f1c`); runtime toggle persists choice and follows `prefers-color-scheme`.
