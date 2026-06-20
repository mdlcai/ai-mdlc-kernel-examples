# ARCHITECTURE.md — PatchLAB

**Build version:** 0.1.0 · See `VERSION.md`
**Source of truth precedence:** RESEARCH.md → ARCHITECTURE.md → SPEC.md
**Archetype:** `saas` · **Design template:** uploaded (`DESIGN-TEMPLATE.html`) — its `:root` tokens are canonical (see DECISIONS.md ADR-013).

---

## 1. System Overview

PatchLAB is an AI-assisted sound-design web app for FL Studio producers. A user types a natural-language prompt (e.g. *"dark analog bass with distortion"*); a deterministic **procedural DSP engine** maps prompt keywords → synthesis parameters → renders multiple WAV variations; the user previews (waveform + audio player), optionally refines, and downloads WAV samples, synth presets (Vital `.vital` JSON), or ZIP sample packs. Authenticated users keep a searchable personal library.

There is **no external AI-audio dependency**: generation is local, deterministic NumPy/SciPy DSP. This is the central architectural decision (see §7, ADR-001) — it removes per-generation cost, latency, GPU, and training-data/licensing exposure, at the price of perceptual fidelity vs neural tools (a known trade-off, RESEARCH §6).

```
                         ┌──────────────────────────────────────────────┐
   Browser  ── HTTPS ──▶ │ nginx reverse proxy (TLS term, HTTP→HTTPS,    │
                         │ HSTS, large header buffers, rate-limit assist)│
                         └───────────────┬──────────────┬───────────────┘
                                         │ /            │ /v1/*  (proxy)
                                ┌────────▼───────┐  ┌───▼───────────────────┐
                                │ Next.js (web)  │  │ FastAPI (api)         │
                                │ App Router     │  │ /v1 routers           │
                                │ shadcn/Tailwind│  │  → services (logic)   │
                                │ Zustand stores │  │  → repositories (DB)  │
                                │ wavesurfer.js  │  │  → audio engine (DSP) │
                                └────────────────┘  └───┬───────────┬──────┘
                                                        │           │
                                              ┌─────────▼──┐   ┌────▼─────────┐
                                              │ PostgreSQL │   │ Asset store  │
                                              │ (asyncpg)  │   │ (disk volume)│
                                              └────────────┘   └──────────────┘
```

---

## 2. Layers & Modules

### 2.1 Frontend (`frontend/`, Next.js 16 App Router)
- `src/app/` — routes (App Router). Public: `/` (landing, ported from template), `/login`, `/register`. Authenticated app shell under `/(app)`: `/studio` (generate), `/library`, `/packs`, `/presets`, `/settings`.
- `src/components/ui/` — shadcn/ui primitives (copied-in source). `src/components/` — composite app components (AppShell, Sidebar, PromptBar, VariationTable, AudioPlayer, WaveformView, ThemeToggle).
- `src/stores/` — Zustand stores, one per domain: `playerStore` (current track, transport), `generationStore` (prompt, in-flight render, variations), `libraryStore` (filters, results), `authStore` (session user).
- `src/lib/api.ts` — typed fetch client to `/v1` (sends credentials; maps RFC 9457 problem+json → field errors).
- `src/app/globals.css` — **single design-token layer** (`:root` + `[data-theme="dark"]`), copied verbatim from `DESIGN-TEMPLATE.html` (ADR-013). All components consume tokens; no ad-hoc hex.
- Data flow: client component → `api.ts` → FastAPI `/v1` → store update → render. Audio bytes streamed from `/v1/sounds/{id}/download`; waveform drawn client-side from the fetched buffer.

### 2.2 Backend (`backend/`, FastAPI 0.137)
Strict layering, dependencies point downward only:

| Layer | Path | Responsibility | May import |
|-------|------|----------------|-----------|
| **Routers** | `app/api/v1/` | HTTP I/O, validation (Pydantic schemas), auth deps, rate-limit decorators. No business logic, no direct ORM. | services, schemas, deps |
| **Services** | `app/services/` | Business logic + **all DB mutations** (the canonical scoped-write layer). Owns transactions/commits. Enforces ownership (`assert_owner`). | repositories, audio, models |
| **Repositories** | `app/repositories/` | Query construction over `AsyncSession`. | models |
| **Audio engine** | `app/audio/` | Pure DSP: `synth.py` (oscillators/filters/envelopes), `prompt_parser.py` (allow-list keyword→param map, **no eval**), `render.py` (WAV write via soundfile), `preset.py` (Vital JSON), `pack.py` (bounded ZIP). No DB, no HTTP. | (numpy/scipy/soundfile only) |
| **Models** | `app/models/` | SQLAlchemy 2.0 declarative models. | — |
| **Core** | `app/core/` | `config.py` (pydantic-settings), `security.py` (argon2id, PyJWT), `logging.py` (JSON), `ratelimit.py` (slowapi), `storage.py` (asset paths, traversal-safe). | — |
| **DB** | `app/db/` | async engine, `async_sessionmaker`, session dependency, Alembic migrations. | models |

### 2.3 Audio engine (DSP) — `app/audio/`
- `prompt_parser.py`: tokenizes the prompt, looks up tokens in a static **allow-list dictionary** (`descriptor → param delta`: e.g. `dark` → lowpass cutoff↓, `analog` → detune+drift, `distortion` → waveshaper amount↑, `bass`/`kick`/`pad`/`lead`/`pluck` → base oscillator+envelope archetype). Unknown tokens are ignored. Output is a fully-clamped `SynthParams` dataclass. **No string is ever `eval`/`exec`'d.**
- `synth.py`: renders a mono/stereo float32 buffer — oscillator bank (sine/saw/square/triangle via `np.sin`/`scipy.signal.sawtooth`/`square`), ADSR envelope, state-variable/`butter`+`sosfilt` filter, optional waveshaper distortion, optional simple reverb (Schroeder), peak-normalize, fade in/out to prevent clicks.
- `render.py`: float buffer → PCM WAV (`soundfile.write`, subtype `PCM_16`/`PCM_24`), enforces caps (max duration, sample rate, channels, total samples) before allocation.
- `variations.py`: deterministic seeded RNG (`numpy.random.default_rng(seed)`) produces N variations by perturbing params; same `(prompt, seed)` → identical output (reproducibility + testability).
- `preset.py`: emits a Vital-compatible `.vital` JSON preset from `SynthParams`.
- `pack.py`: assembles a ZIP from a set of generated sounds with a **running uncompressed-byte budget** (ZIP-bomb guard) and per-pack file/byte caps.

### 2.4 Reverse proxy (`infra/nginx/`)
nginx terminates TLS, forces HTTP→HTTPS (301), sets HSTS, raises header buffers (`large_client_header_buffers 8 32k; client_header_buffer_size 16k;`), proxies `/` → web and `/v1/`,`/api/` → api.

---

## 3. Data Model (PostgreSQL, SQLAlchemy 2.0 async)

```
users
  id            UUID  PK   (default uuid4)
  email         CITEXT/lower  UNIQUE NOT NULL      ← INV-5
  password_hash TEXT  NOT NULL                     (argon2id)
  display_name  TEXT
  is_active     BOOL  NOT NULL default true
  created_at    TIMESTAMPTZ NOT NULL default now()
  updated_at    TIMESTAMPTZ NOT NULL default now()

refresh_tokens
  id          UUID PK
  user_id     UUID FK→users.id ON DELETE CASCADE NOT NULL
  jti         TEXT UNIQUE NOT NULL                 ← INV-6 (revocation/rotation)
  expires_at  TIMESTAMPTZ NOT NULL
  revoked     BOOL NOT NULL default false
  created_at  TIMESTAMPTZ NOT NULL default now()

sounds                                              (generated audio asset)
  id           UUID PK
  owner_id     UUID FK→users.id ON DELETE CASCADE NULL   (NULL = anonymous, TTL-expiring)
  pack_id      UUID FK→packs.id ON DELETE SET NULL NULL
  parent_id    UUID FK→sounds.id ON DELETE SET NULL NULL  (refine lineage)
  prompt       TEXT NOT NULL
  name         TEXT NOT NULL
  category     TEXT NOT NULL          (kick|bass|perc|fx|pad|lead|pluck|loop|oneshot)
  kind         TEXT NOT NULL          (sample|loop|oneshot|preset)
  params       JSONB NOT NULL         (SynthParams snapshot — reproducible)
  seed         BIGINT NOT NULL
  file_path    TEXT NOT NULL          (relative path under ASSET_ROOT; server-owned)
  file_format  TEXT NOT NULL          (wav|vital)
  sample_rate  INT  NOT NULL
  bit_depth    INT  NOT NULL
  duration_ms  INT  NOT NULL
  size_bytes   BIGINT NOT NULL
  expires_at   TIMESTAMPTZ NULL       (set for anonymous assets; cleanup job)
  created_at   TIMESTAMPTZ NOT NULL default now()
  INDEX (owner_id, created_at DESC), INDEX (owner_id, category)

packs
  id          UUID PK
  owner_id    UUID FK→users.id ON DELETE CASCADE NOT NULL
  name        TEXT NOT NULL
  theme       TEXT NOT NULL
  file_path   TEXT NULL              (zip; built on export)
  size_bytes  BIGINT NULL
  created_at  TIMESTAMPTZ NOT NULL default now()
  INDEX (owner_id, created_at DESC)
```

Migrations: **Alembic**, reversible. Unique constraints on `users.email` and `refresh_tokens.jti` are migration-defined (INV-5, INV-6).

---

## 4. API Surface (REST, `/v1/`, RFC 9457 errors)

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/api/health` | none | Liveness + dependency status JSON |
| POST | `/v1/auth/register` | none | Create account (anti-enumeration) |
| POST | `/v1/auth/login` | none | Set HttpOnly access+refresh cookies |
| POST | `/v1/auth/logout` | cookie | Revoke refresh, clear cookies |
| POST | `/v1/auth/refresh` | cookie | Rotate tokens |
| GET | `/v1/auth/me` | cookie | Current user |
| POST | `/v1/sounds/generate` | optional | Generate N variations from prompt |
| POST | `/v1/sounds/{id}/refine` | owner | Refine an existing sound → new variations |
| GET | `/v1/sounds` | cookie | List/search owner library (q, category, kind, page) |
| GET | `/v1/sounds/{id}` | owner/anon-cap | Metadata |
| GET | `/v1/sounds/{id}/download` | owner/anon-cap | Stream WAV (FileResponse, traversal-safe) |
| GET | `/v1/sounds/{id}/preset` | owner | Download `.vital` preset |
| DELETE | `/v1/sounds/{id}` | owner | Delete sound + file |
| POST | `/v1/packs` | cookie | Generate a themed pack (multiple sounds) |
| GET | `/v1/packs` | cookie | List packs |
| GET | `/v1/packs/{id}` | owner | Pack detail + items |
| GET | `/v1/packs/{id}/download` | owner | Stream ZIP (bounded) |
| DELETE | `/v1/packs/{id}` | owner | Delete pack + files |

Anonymous generation: assets stored with `owner_id NULL`, `expires_at = now()+TTL`, downloadable by unguessable UUID (capability), stricter rate limits, no library listing. Authenticated assets require ownership on every read/mutate/download (`assert_owner`, INV-12).

---

## 5. Cross-cutting Concerns

- **Auth:** Argon2id password hashing (argon2-cffi). PyJWT HS256, `algorithms` pinned on decode. Access token (15 min) + refresh token (7 d, rotated, DB-tracked `jti`) in **HttpOnly + Secure + SameSite=Lax** cookies. Login/register anti-enumeration: identical response shape & timing-padded for unknown email vs wrong password.
- **Rate limiting (slowapi):** `auth/login`,`register` keyed `(ip,email)` and fired before hash compare; `sounds/generate` & `packs` keyed by `(user|ip)` with stricter anon caps; `download` per-ip. 429 + `Retry-After`.
- **Resource caps (DoS):** generation enforces `MAX_DURATION_S`, `MAX_SAMPLE_RATE`, `MAX_VARIATIONS`, `MAX_TOTAL_SAMPLES`, pack `MAX_PACK_ITEMS`, `MAX_PACK_BYTES` (ADR-007). Validated by Pydantic `Field` bounds + service-layer clamps.
- **Storage safety:** `core/storage.py` builds every path from server-owned UUIDs under a fixed `ASSET_ROOT`, resolves and asserts the realpath stays within root (no client-supplied path, no `../`). Per-user storage quota; anon TTL cleanup job.
- **Logging:** structured JSON (python-json-logger) with request id; secrets/tokens/passwords redacted; bodies not logged.
- **Config:** pydantic-settings from env; `.env.example` enumerates every var.
- **Protocol:** HTTPS-only (RESEARCH Build Constraints). nginx forces redirect + HSTS; smoke test runs against the served origin.

---

## 6. Deployment

Single-host Docker Compose (`container_strategy: single container (Docker Compose)` → realized as a small multi-service compose: `db` (postgres:18), `api` (FastAPI/uvicorn), `web` (Next.js standalone), `proxy` (nginx TLS). Justification for >1 service logged ADR-011: Next.js and Python cannot share one runtime image cleanly; compose remains a single `docker compose up`.) Asset store is a named volume mounted into `api`. Local dev: `docker compose up` → `https://localhost`. Migrations run on api start (`alembic upgrade head`). Scale tier = **small** (<1k concurrent): single Postgres + framework pool, no Redis/queue, synchronous generation, per-route rate limiting — built TO tier, not beyond (RESEARCH §scale table).

---

## 7. Key Technical Decisions (ADRs summarized; full in DECISIONS.md)

- **ADR-001 Procedural DSP over neural audio** — deterministic, offline, no cost/licensing exposure. Alt considered: Stable Audio/ElevenLabs APIs (rejected: cost, latency, GPU, lawsuit exposure); kept as optional future premium tier.
- **ADR-002 PyJWT + argon2-cffi over python-jose/passlib** — both legacy libs unmaintained (RESEARCH §3.1/§3.7).
- **ADR-003 Synchronous generation, no job queue** — small scale + sub-second renders; queue is over-build at this tier (RESEARCH §scale). Bounded by hard caps + rate limits instead.
- **ADR-004 Capability-URL anonymous assets** — anon generations are unguessable-UUID, TTL-expiring, no PII; avoids forcing auth for the landing "try it" flow while bounding storage. Alt: in-memory only (rejected: breaks download/preview).
- **ADR-005 Local-disk asset store w/ traversal-safe resolver** — small scale; S3/presigned URLs deferred (RESEARCH §3.8). Alt: object storage (deferred, documented).
- **ADR-006 Alembic migrations** — reversible schema, enforced unique constraints.
- **ADR-007 Hard resource caps** — primary DoS mitigation given CPU-bound generation.

Comprehensive-depth threat model lives in RESEARCH §5; per-feature security pass (BUILD.md seven categories) enforced each feature; formal Security Audit at the audit gate; COMPLIANCE.md maps SPEC §security controls to code.

---

## 8. Risks (impl-level; register in RESEARCH §5)
- CPU-bound generation under concurrency → caps + rate limits + small-tier scope; revisit queue at medium tier.
- DSP fidelity vs neural → product risk, not technical; premium-tier escape hatch documented.
- Asset disk growth → quotas + anon TTL cleanup + monitoring.

---

## 9. Architectural Invariants

Every rule below has a machine-checkable or `manual` entry in `invariants.json` (cross-referenced by id). The Stage 4 invariant-lint and the `ship`-time Drift Detection Gate consume that file. Never disable an invariant to pass a gate — amend §9 + `invariants.json` together with an ADR.

| ID | Rule | File / section | Check type |
|----|------|----------------|-----------|
| **INV-1** | No raw f-string/format-interpolated SQL anywhere (SQL-injection guard); all queries use SQLAlchemy constructs/bound params. | `backend/app/**` | forbidden-pattern |
| **INV-2** | No `eval()`/`exec()` of dynamic strings anywhere in backend source (prompt-injection guard for the synth parser). | `backend/app/**` | forbidden-pattern |
| **INV-3** | A `docker-compose.yml` exists so the whole stack starts with one command. | repo root | required-file |
| **INV-4** | A root `.env.example` enumerates every environment variable. | repo root | required-file |
| **INV-5** | `users.email` has a UNIQUE constraint (one account per email; anti-enumeration depends on it). | migration | required-unique-constraint |
| **INV-6** | `refresh_tokens.jti` has a UNIQUE constraint (token rotation/revocation integrity). | migration | required-unique-constraint |
| **INV-7** | No hard-coded hex colors in frontend components; color comes only from the token layer. | `frontend/src/**` outside token files | forbidden-pattern |
| **INV-8** | The design-token layer (`globals.css` with `:root`/dark tokens copied from the template) exists. | `frontend/src/app/globals.css` | required-file |
| **INV-9** | The invariant-lint runner exists and is wired as a build script. | `scripts/invariant-lint.mjs` | required-file |
| **INV-10** | UI coverage: every SPEC §UI Surface screen route renders; every non-internal endpoint is referenced by UI; every §5 workflow has an e2e spec. | `frontend/src/app/**`, `SPEC.md` | ui-coverage |
| **INV-11** | Generated asset file paths are built only by the storage resolver (traversal-safe); no `open(` on request-derived paths in routers. | `backend/app/api/**` | forbidden-pattern |
| **INV-12** | All DB writes (mutations/commits) occur only in the services layer; routers never mutate the session directly (scoped-write + ownership chokepoint). | `backend/app/services/**` | manual |
| **INV-13** | JWT decode pins `algorithms=[...]` (never accepts `alg=none`); secret comes from config, not literal. | `backend/app/core/security.py` | manual |
| **INV-14** | nginx reverse-proxy config exists enforcing HTTPS redirect + HSTS + large header buffers. | `infra/nginx/` | required-file |

Invariant count: **14** (12 machine-checkable: 4× forbidden-pattern, 5× required-file, 2× required-unique-constraint, 1× ui-coverage; 2× manual — INV-12, INV-13. The runner itself is INV-9).
