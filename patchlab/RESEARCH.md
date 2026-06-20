# RESEARCH.md — PatchLAB

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "music"

## Product Vision
**Problem:** Music producers spend a significant amount of time searching through sample libraries, designing synthesizer patches, and tweaking sounds before they can begin creating music. Existing libraries often contain thousands of generic or overused sounds, making it difficult to find assets that match a specific creative vision. While AI music tools have emerged in recent years, most focus on generating complete songs rather than the individual sounds, samples, and presets that producers actually need within their workflow.

As a result, producers face creative delays, repetitive sound design tasks, and limited access to truly unique audio assets. There is a need for a solution that can rapidly generate custom, production-ready sounds from simple natural language descriptions and seamlessly integrate with digital audio workstations such as FL Studio.

**Who it affects:** Music Producers

Producers spend significant time searching for samples, designing sounds, and modifying presets before they can begin composing or arranging music. This slows production and can interrupt creative momentum.

FL Studio Users

FL Studio producers often rely heavily on third-party sample packs and synthesizer presets. Finding or creating the right sound can become a bottleneck in the production process, especially when working under deadlines.

Sound Designers

Professional sound designers are expected to create unique and original sounds for artists, games, films, and content creators. Repetitive sound creation tasks consume valuable time that could be spent on higher-value creative work.

Independent Artists and DJs

Independent creators typically have limited budgets and may not have advanced sound design expertise. They often struggle to create distinctive sounds without purchasing large collections of sample packs or spending significant time learning synthesis techniques.

Content Creators and Media Producers

Video creators, streamers, podcasters, and game developers frequently need custom sound effects and audio assets. Existing solutions often require extensive searching through libraries or hiring external specialists.

Beginner Producers

New producers face a steep learning curve when attempting to create their own sounds. The complexity of synthesizers and sound design tools can discourage experimentation and slow skill development.

Business Impact

The problem results in:

Lost creative time
Reduced productivity
Increased reliance on generic sounds
Higher spending on sample packs and presets
Delayed project completion
Reduced differentiation in music productions

By simplifying sound creation through AI, producers and creators can spend more time making music and less time searching for or designing sounds.

**Why existing solutions fall short:** Current sound design and music production tools do not adequately address the need for fast, customized sound creation within a producer's workflow.

Traditional Sample Libraries

Platforms such as sample marketplaces provide access to millions of sounds, but users must manually search, audition, and organize assets. This process is time-consuming and often results in producers using the same sounds as thousands of others, limiting originality.

Synthesizers and Sound Design Tools

Modern synthesizers offer virtually unlimited sound design possibilities but require significant technical knowledge and experience. Creating a desired sound from scratch can take anywhere from minutes to hours, interrupting creative flow and creating a steep learning curve for newer producers.

AI Music Generation Platforms

Most AI music tools focus on generating complete songs, stems, or compositions. While impressive, these solutions do not align with the needs of producers who want individual sounds, one-shots, loops, effects, and presets that can be incorporated into their own projects.

**Solution:** PatchLAB is an AI-powered sound design platform that generates production-ready audio assets from natural language prompts, purpose-built for FL Studio producers. Users describe the sound they want—"deep analog bass," "cinematic pad," "hard-hitting techno kick"—and receive instantly downloadable WAV samples, loops, sound effects, and synthesizer presets optimized for direct FL Studio import. The platform eliminates hours spent designing patches or hunting through sample libraries by delivering unique, customizable audio tailored to the producer's creative vision in seconds. Core differentiator: native FL Studio workflow integration combined with AI-driven sound generation that produces variations on demand, enabling electronic music creators, sound designers, DJs, and beat makers to accelerate production cycles while maintaining creative control over their sonic palette.
MVP Features
Text-to-sound generation
WAV sample export
Multiple sound variations per prompt
Audio preview player
Sample pack generation
FL Studio optimized workflow
Target Users
FL Studio producers
Electronic music creators
Sound designers
DJs and beat makers
Content creators requiring custom audio
Core Value Proposition

Generate unique sounds in seconds instead of spending hours designing patches or searching sample libraries.

Long-Term Vision

Become the leading AI sound design platform and plugin ecosystem for FL Studio, enabling producers to create custom sounds, presets, and sample packs through simple natural language prompts.


## Users & Outcomes
**Key Workflows:**
1. Generate a Custom Sound

Goal: Create a unique sound from a text description.

User enters a prompt (e.g., "dark analog bass with distortion").
AI analyzes the prompt and identifies sound characteristics.
Sound generation engine creates multiple variations.
User previews generated sounds.
User selects and downloads the preferred version.
User imports the sound into FL Studio.

Output: WAV sample, loop, or one-shot sound.

2. Refine an Existing Sound

Goal: Iteratively improve generated sounds.

User generates an initial sound.
User provides refinement instructions (e.g., "make it punchier" or "add more reverb").
AI modifies the sound based on the request.
User previews the updated version.
Process repeats until desired result is achieved.

Output: Refined production-ready audio asset.

3. Generate a Synth Preset

Goal: Create a custom synthesizer preset.

User describes a desired sound.
AI generates synthesizer parameters.
Preset is created for a supported synth.
User downloads the preset file.
Preset is loaded into FL Studio.

Output: Synth preset file and optional audio preview.

4. Create a Sample Pack

Goal: Generate multiple related sounds from a single concept.

User enters a genre or theme.
User selects desired sound categories.
AI generates a collection of sounds.
Assets are organized automatically.
User downloads a ZIP package.

Output: Complete sample pack ready for use in FL Studio.

5. Search and Reuse Assets

Goal: Access previously generated sounds.

User searches their library.
System displays matching sounds and metadata.
User previews assets.
User downloads or regenerates variations.

Output: Reusable personal sound library.

6. Export to Production Workflow

Goal: Move generated assets into FL Studio.

User selects assets.
System exports in FL Studio-compatible formats.
User imports sounds into projects.
Assets are used for composition, mixing, and production.

Output: Seamless integration into existing music production workflows.

Primary User Journey (MVP)

Prompt → Generate → Preview → Refine → Download → Import into FL Studio

This workflow should be optimized for speed, requiring only a few clicks from idea to usable sound. The objective is to allow producers to create custom sounds in seconds rather than spending hours designing them manually or searching through sample libraries.

**Success Metrics:**
User Adoption
Number of registered users
Monthly Active Users (MAU)
Weekly Active Users (WAU)
User growth rate
Free-to-paid conversion rate
Engagement
Average sounds generated per user
Average session duration
Number of prompts submitted per session
Percentage of users returning within 30 days
Library usage and asset downloads
Product Effectiveness
Time from prompt to downloadable sound
Sound generation success rate
Percentage of generated sounds downloaded
Percentage of generated sounds refined by users
User satisfaction rating for generated assets
Workflow Efficiency
Reduction in time spent creating sounds
Reduction in time spent searching sample libraries
Number of sounds exported to FL Studio
Repeat usage of generated assets
Quality Metrics
User rating of generated sounds
Prompt-to-sound relevance score
Asset reuse rate
Sound approval/download rate
Customer feedback and support trends
Business Metrics
Monthly Recurring Revenue (MRR)
Annual Recurring Revenue (ARR)
Customer Acquisition Cost (CAC)
Customer Lifetime Value (LTV)
Subscription retention rate
Churn rate

**Non-Goals:**
To maintain focus and deliver a high-quality MVP, the following capabilities are explicitly out of scope.

Full Song Generation

The platform will not generate complete songs, tracks, or finished compositions. The focus is on individual sounds, samples, loops, and presets that producers can incorporate into their own projects.

DAW Replacement

The application is not intended to replace FL Studio or any other Digital Audio Workstation (DAW). Users will continue to compose, arrange, mix, and master within their preferred production software.

Audio Mixing and Mastering

The platform will not provide automated mixing, mastering, or audio engineering services during the MVP phase.

Real-Time Sound Generation

The MVP will not support real-time sound generation within FL Studio or other DAWs. Sounds will be generated through the application and exported for use in production projects.

VST Plugin Development

A native VST/AU plugin is not part of the initial release. Integration will be achieved through downloadable audio assets and preset files.

Marketplace Features

Users will not be able to buy, sell, share, or publish generated sounds through a community marketplace during the MVP phase.

Collaboration Features

Real-time collaboration, project sharing, team workspaces, and social features are not included in the MVP.

Multi-DAW Optimization

While exported assets may work in other DAWs, the initial product will be optimized specifically for FL Studio workflows rather than providing dedicated integrations for multiple platforms.

Professional Sample Library Replacement

The goal is not to replace established sample libraries, but to complement them by enabling rapid creation of custom, unique sounds.

Music Education Platform

The application is not intended to teach music theory, production techniques, or sound design concepts, although users may learn through experimentation.


## Build Constraints

```yaml
# Infrastructure & Ops
protocol_support: "HTTPS only"
monitoring: "basic"
container_strategy: "single container (Docker Compose)"

# Data & Storage
database_preference: "PostgreSQL"

# Security & Compliance
rate_limiting: true

# Frontend
frontend_framework: "Next.js"
ui_component_library: "shadcn/ui"
css_approach: "Tailwind CSS"
state_management: "Zustand"

# Backend
backend_framework: "FastAPI"
api_style: "REST"
api_versioning: "URL path (/v1/)"
orm_preference: "SQLAlchemy"
typescript_backend: false

# Performance & Quality
testing_strategy: "test-after"
logging_format: "structured JSON"

# Scope & Platform
scale: "small — under 1k concurrent"
target_platforms: ["web"]
```

## Design Language

### Archetype
Archetype: saas

This product's design archetype is **saas**. Read `DESIGN.md` Part II §`saas` (fetched alongside BUILD.md from the MDLC kernel) and treat its Layout Doctrine, Density, Type System, Color & Atmosphere, Motion Budget, and Signature Components as binding requirements, and its Good-vs-Avoid list as the acceptance rubric. The token tables below are the resolved starting palette; an explicit brand override outranks them per the `DESIGN.md` precedence list. The Universal Excellence floor (`DESIGN.md` Part I) applies on top regardless of archetype.

### Brand Voice
Professional, confident, efficient. Composed information hierarchy; tables and forms done well; full dark mode.

### Art Direction
PatchLAB's visual system is a composed, production-grade interface that mirrors the precision and intentionality of professional music production software—think the clarity of FL Studio's mixer or Ableton's session view, not the decorative excess of consumer apps. The palette anchors on a calm, near-black foundation (#0f172a or deeper) with cool grays (#1e293b, #334155) for surfaces and dividers, creating visual rest and reducing cognitive load during long creative sessions; the accent blue (#2563eb) is deployed with surgical restraint—exclusively for interactive states, focus indicators, active generation progress, and the primary call-to-action (generate sound)—never as background wash or decorative flourish, ensuring it reads as intentional authority rather than brand noise. Typography is anchored in a modern, geometric sans-serif (Geist or similar) with a confident, slightly condensed character that conveys efficiency without coldness; hierarchy is established through weight and scale rather than color, with body text at 14–16px for readability during extended use and a restrained line-height (1.5–1.6) that respects density without sacrificing breathing room. Layout follows a disciplined 8px grid with generous but purposeful whitespace; cards, panels, and form fields are minimal (thin borders in #334155, no drop shadows or gradients), tables are the primary information architecture for sound variations and pack contents with clear row dividers and hover states in a subtle blue tint (#1e3a8a at 8% opacity), and the audio preview player is embedded inline with waveform visualization in a muted teal accent (#0d9488) to differentiate it from UI chrome. Motion is functional and restrained—0.2s ease-out for state changes, subtle scale and opacity shifts for button feedback, smooth fade-ins for generated audio previews, and no decorative animations or parallax; dark mode is the default and only mode, honoring the producer's workflow and reducing eye strain during late-night sessions. Accessibility floor: WCAG AA minimum across all interactive elements, 4.5:1 contrast on text, focus states visible at 3px with the accent blue, and all audio generation states communicated through both visual indicators (progress bar, status text) and optional haptic feedback; form inputs use clear labels and error states in a warm amber (#d97706) to distinguish from the primary accent, and the entire interface must remain fully navigable via keyboard with logical tab order reflecting the generation-to-export workflow.

This art-direction brief is a binding directive — honor its palette feel, imagery, type personality, layout mood, and motion over the archetype defaults (per `DESIGN.md` precedence). An uploaded Design Template still outranks it on concrete tokens.

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #2563eb | Buttons, links, active states |
| Secondary | #a92fe1 | Accents, badges, highlights |
| Accent | #dcb04f | Callouts, hover states |
| Background | #fafafa | Page background |
| Surface | #f4f5f5 | Cards, elevated containers |
| Text | #16181d | Headings, body text |
| Text Secondary | #5c6270 | Captions, muted text |
| Success | #1fad53 | Success states, confirmations |
| Warning | #ec9c13 | Warnings, pending states |
| Error | #df2020 | Errors, destructive actions |

### Color System — Dark Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #2563eb | Buttons, links, active states |
| Secondary | #ac36e2 | Accents, badges, highlights |
| Accent | #ddb255 | Callouts, hover states |
| Background | #090a0c | Page background |
| Surface | #121317 | Cards, elevated containers |
| Text | #eaeaec | Headings, body text |
| Text Secondary | #838995 | Captions, muted text |
| Success | #33cc6b | Success states, confirmations |
| Warning | #e2a336 | Warnings, pending states |
| Error | #d74242 | Errors, destructive actions |

### Typography
- Heading: Manrope (600/700 weight)
- Body: Inter (400/500 weight)
- Mono: JetBrains Mono (code, pre, kbd)
- Base size: 14px, scale ratio: 1.2
- Scale: 9.7 / 11.7 / 14 / 16.8 / 20.2 / 24.2 / 29px

### Layout
- Pattern: Sidebar + Content
- Max width: 1440px, sidebar: 224px
- Spacing: Comfortable (12/16/24/32px)
- Breakpoints: 640 / 768 / 1024 / 1280px

### Component Style
- Variant: Sharp
- Border radius: 2px (sm: 0px, lg: 6px, xl: 10px)
- Shadows: None — `none`
- Theme: Light + Dark — ship both palettes with a runtime theme toggle that follows the user's system preference (`prefers-color-scheme`) and persists their explicit choice

### Tailwind Config
```typescript
// tailwind.config.ts — theme.extend
{
  colors: {
    primary: '#2563eb',
    secondary: '#a92fe1',
    accent: '#dcb04f',
    background: '#fafafa',
    surface: '#f4f5f5',
    foreground: '#16181d',
    muted: '#5c6270',
    success: '#1fad53',
    warning: '#ec9c13',
    destructive: '#df2020',
  },
  fontFamily: {
    heading: ['Manrope', 'system-ui', 'sans-serif'],
    body: ['Inter', 'system-ui', 'sans-serif'],
    mono: ['JetBrains Mono', 'monospace'],
  },
  borderRadius: {
    DEFAULT: '2px',
    sm: '0px',
    lg: '6px',
    xl: '10px',
  },
}
```

### CSS Custom Properties
```css
/* Light mode */
:root {
  --color-primary: #2563eb;
  --color-secondary: #a92fe1;
  --color-accent: #dcb04f;
  --color-background: #fafafa;
  --color-surface: #f4f5f5;
  --color-text: #16181d;
  --color-text-secondary: #5c6270;
  --color-success: #1fad53;
  --color-warning: #ec9c13;
  --color-error: #df2020;
  --font-heading: 'Manrope', system-ui, sans-serif;
  --font-body: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
  --font-size-base: 14px;
  --radius: 2px;
}

/* Dark mode */
.dark, [data-theme="dark"] {
  --color-primary: #2563eb;
  --color-secondary: #ac36e2;
  --color-accent: #ddb255;
  --color-background: #090a0c;
  --color-surface: #121317;
  --color-text: #eaeaec;
  --color-text-secondary: #838995;
  --color-success: #33cc6b;
  --color-warning: #e2a336;
  --color-error: #d74242;
}
```

### Accessibility
- WCAG AA compliance
- Lighthouse target: 95+
- Responsive breakpoints: 640 / 768 / 1024 / 1280px
- Reduced motion: Standard animations

## Design Template

An HTML design template was uploaded with this project (built by the customer
in Claude Design / artifacts). Fetch it at build time via the
`get_design_template` MCP tool — that is the source of truth, not this section.

### How to use it
- **Copy the template's `:root` design tokens verbatim FIRST.**
- Use the markup as the visual scaffold for the landing/home surface.
- The copied tokens are binding for the ENTIRE build.
- Save the raw template to the project root as `DESIGN-TEMPLATE.html`.

---

# Stage 0 Research (added by build pipeline — comprehensive depth, verified June 2026)

## 3. Source Categories

### 3.1 Official / Vendor Documentation

All versions verified against npm registry, PyPI, and vendor docs/changelogs as of 19 June 2026.

| # | Component | Official URL | Current stable (Jun 2026) | Confirmed fact / command / signature |
|---|-----------|--------------|---------------------------|--------------------------------------|
| 1 | **Next.js (App Router)** | https://nextjs.org | **16.2.9** | App Router: route from `app/page.tsx`; handlers in `app/.../route.ts` exporting `GET`/`POST`. `npx create-next-app@latest`. Container: `output: 'standalone'`. |
| 2 | **React** | https://react.dev | **19.2.7** | shadcn/ui validated for React 19. |
| 3 | **FastAPI** | https://fastapi.tiangolo.com | **0.137.2** | Starlette + Pydantic. `/v1/` via `APIRouter(prefix="/v1")`. |
| 4 | **SQLAlchemy** | https://docs.sqlalchemy.org/en/20/ | **2.0.51** | Async: `create_async_engine("postgresql+asyncpg://…")`, `async_sessionmaker(expire_on_commit=False)`. Needs `greenlet`. |
| 5 | **Pydantic** | https://docs.pydantic.dev | **2.13.4** / pydantic-settings **2.14.1** | v2 `model_config = ConfigDict(...)`; `BaseSettings` for env config. |
| 6 | **PostgreSQL** | https://www.postgresql.org | **18.4** | Pair with **asyncpg** (`postgresql+asyncpg://`). |
| 7 | **Tailwind CSS** | https://tailwindcss.com | **4.3.1** | CSS-first: `@import "tailwindcss";`, `@theme` tokens. |
| 8 | **shadcn/ui** | https://ui.shadcn.com | CLI **4.11.0** | Copies component source in. `npx shadcn@latest init/add`. Tailwind v4 + React 19 supported. |
| 9 | **Zustand** | https://zustand.docs.pmnd.rs | **5.0.14** | `create((set,get)=>({...}))`, no Provider. |
| 10 | **numpy** | https://numpy.org | **2.4.6** | `np.int16`/`np.float32`, `np.sin`/`np.linspace`/`np.clip`. |
| 11 | **scipy** | https://scipy.org | **1.17.1** | `scipy.signal.butter(..., output='sos')` + `sosfilt`; `sawtooth`, `square`. |
| 12 | **soundfile / libsndfile** | https://python-soundfile.readthedocs.io | soundfile **0.14.0** / libsndfile **1.2.2** | `soundfile.write(file,data,sr,subtype='PCM_16'|'PCM_24'|'FLOAT')`. |
| 13 | **wave (stdlib)** | https://docs.python.org/3/library/wave.html | stdlib | PCM integer only; `setnchannels/setsampwidth/setframerate/writeframes`. |
| 14 | **Docker Compose** | https://docs.docker.com/guides/nextjs/containerize/ | Compose v2 | Next.js `output:'standalone'`; node + python services behind TLS proxy. |
| 15 | **PyJWT** | https://pyjwt.readthedocs.io | **2.13.0** | FastAPI docs use PyJWT. `jwt.encode/decode(..., algorithms=["HS256"])`. |
| 16 | **Password hashing** | argon2-cffi **25.1.0** / bcrypt **5.0.0** | — | Prefer Argon2id (argon2-cffi) or bcrypt directly; avoid passlib. |
| 17 | **slowapi** | https://slowapi.readthedocs.io | **0.1.10** (MIT) | `Limiter(key_func=get_remote_address)`, `@limiter.limit("5/minute")`; route needs `request: Request`; 429 over-limit. |

### 3.2 GitHub Repositories

| Owner/Repo | Stars (~) | Last active | License | Why relevant |
|------------|-----------|-------------|---------|--------------|
| fastapi/fastapi | 99.4k | Active (Jun 2026) | MIT | Backend framework. |
| sqlalchemy/sqlalchemy | 11.9k | Active (2.0.51) | MIT | ORM async. |
| shadcn-ui/ui | 117k | Active (Jun 2026) | MIT | UI components. |
| pmndrs/zustand | 58.4k | Active | MIT | Client state. |
| vercel/next.js | 140k | Active | MIT | Frontend framework. |
| laurentS/slowapi | 2.0k | Active (0.1.10) | MIT | Rate limiting (canonical repo capital-S). |
| bastibe/python-soundfile | 838 | Active (0.14.0) | BSD-3 | WAV PCM_16/24/FLOAT I/O. |
| libsndfile/libsndfile | 1.7k | 1.2.2 (current) | LGPL-2.1 | C lib under soundfile. |
| jiaaro/pydub | 9.8k | stale (2021) | MIT | Optional; maintained fork if needed. |
| spotify/pedalboard | ~5k | Active | GPLv3 | Effects/VST3 (GPL — not core engine). |
| facebookresearch/audiocraft | 23.4k | Active | MIT code / CC-BY-NC weights | Reference for optional future neural audio. |
| vintasoftware/nextjs-fastapi-template | 319 | Active | MIT | Best-matched starter. |
| demberto/PyFLP | ~198 | stale | GPL-3.0 | `.flp` parser reference. |
| fastapi/full-stack-fastapi-template | official | Active | MIT | Project-structure reference. |

### 3.3 Video / Tutorials
1. **Real Python — "Reading and Writing WAV Files in Python"** — https://realpython.com/python-wav-files/ — canonical WAV-generation path.
2. **Zach Denton — "Generate Audio with Python"** — https://zach.se/generate-audio-with-python/ — oscillators, envelopes, raw PCM WAV.
3. **Vinta Software — Next.js + FastAPI Template guide** — https://www.vintasoftware.com/blog/next-js-fastapi-template — Next.js↔FastAPI wiring, auth, Docker.

### 3.4 Articles / Blog Posts
1. **SQLAlchemy 2.0 async ORM guide** — https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html
2. **FastAPI StreamingResponse & FileResponse** — https://fastapi.tiangolo.com/advanced/custom-response/
3. **OWASP Password Storage Cheat Sheet** — https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
4. **Splice AI tools launch coverage (2026)** — https://www.aimusicpreneur.com/ai-tools-news/splice-ai-tools-pay-sample-creators-variations-2026/

### 3.5 Standards / RFCs

| Standard | URL | Status (Jun 2026) | Fact |
|----------|-----|-------------------|------|
| **WAV / RIFF** | https://www.aelius.com/njh/wavemetatools/doc/riffmci.pdf | De-facto (1991) | RIFF chunks; `"fmt "` + `"data"`. |
| **JWT — RFC 7519** | https://datatracker.ietf.org/doc/html/rfc7519 | Proposed Standard | claims `exp/iat/sub/iss/aud/nbf/jti`. |
| **OWASP ASVS** | https://owasp.org/www-project-application-security-verification-standard/ | **5.0.0** (2025) | ~350 testable reqs, L1–L3. |
| **Problem Details — RFC 9457** | https://www.rfc-editor.org/rfc/rfc9457.html | obsoletes 7807 | `application/problem+json`. |
| **WCAG 2.2** | https://www.w3.org/TR/WCAG22/ | W3C Rec (2023) | Target AA. |
| **HTTP RateLimit headers** | https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/ | Draft-11 | `RateLimit` + `RateLimit-Policy`. |
| **NIST SP 800-63B-4** | https://csrc.nist.gov/pubs/sp/800/63/b/4/final | Final (2025) | Authenticator guidance. |

### 3.6 Competing Products
1. **Splice** (https://splice.com) — largest royalty-free lib; 2026 generative AI; credit/cloud, broad-DAW. *PatchLAB differs:* FL-focused, local DSP, no credits, ZIP packs.
2. **Suno** (https://suno.com) — AI full songs w/ vocals; label litigation. *Contrast:* finished songs not one-shots; DSP has no training-data exposure.
3. **Stability AI Stable Audio** (https://stability.ai) — neural cloud tracks/stems/SFX, credit-based. *Differs:* PatchLAB offline deterministic DSP.
4. **ElevenLabs Sound Effects** (https://elevenlabs.io/sound-effects) — text-to-SFX MP3. *Differs:* PatchLAB FL-ready PCM WAV + presets.
5. Output Arcade, Soundraw/Aimi, Google MusicFX, Meta AudioCraft; FL Studio 2025 "Gopher" assistant — adjacent first-party motion.

**Unique cell:** FL-targeted + DSP WAV samples *and* synth presets + ZIP packs + fully local generation.

### 3.7 Community Threads
1. Real Python / SO — float→`np.int16`, 44.1/48 kHz, fades to avoid clicks.
2. FastAPI Discussions — python-jose abandoned → PyJWT (https://github.com/fastapi/fastapi/discussions/11345).
3. PyPI warehouse #15454 — passlib breakage on Python 3.13+ → argon2-cffi/bcrypt.
4. r/FL_Studio — receptive to assistive AI, value royalty clarity + DAW-native WAV.
5. KVR Audio / r/edmproduction — `.wav` universally safe; Vital `.vital` JSON most automatable preset.

### 3.8 APIs / Integrations
**Core (MVP, local):** numpy + scipy.signal (DSP); soundfile/libsndfile (WAV PCM_16/24/FLOAT) + stdlib `wave` fallback; stdlib `zipfile` (bounded packs).
**Optional/future neural:** Stability AI Stable Audio API; ElevenLabs SFX API; Meta AudioCraft (self-host).
**Optional infra (future):** Stripe (subscriptions); S3-compatible storage (presigned expiring URLs); transactional email.

### 3.9 Patterns / Best Practices
1. **Background work vs queue (small scale):** FastAPI `BackgroundTasks` for short renders; arq/RQ+Redis with job-id polling once CPU-heavy (DoS back-pressure control).
2. **Streaming binary:** `StreamingResponse(gen, media_type="audio/wav", headers={Content-Disposition})`; `FileResponse` for finished files; presigned URLs at scale.
3. **Next.js audio + waveform:** native `<audio>` + Web Audio API; wavesurfer.js v7.x (BSD-3) for waveform.
4. **Zustand:** one store per domain, selector subscriptions to limit re-renders.
5. **SQLAlchemy 2.0 async:** `create_async_engine` + `async_sessionmaker(expire_on_commit=False)`, DI `AsyncSession` per request, asyncpg.
6. **Docker:** multi-stage per service; Next.js standalone; structured JSON logging.

---

## 4. Stack Candidates

**Locked stack — confirmed versions (Jun 2026):**

| Layer | Choice | Version |
|-------|--------|---------|
| Frontend | Next.js (App Router) | 16.2.9 |
| UI runtime | React | 19.2.7 |
| Components | shadcn/ui (CLI) | 4.11.0 |
| Styling | Tailwind CSS | 4.3.1 |
| Client state | Zustand | 5.0.14 |
| Waveform | wavesurfer.js | 7.12.8 |
| API | FastAPI | 0.137.2 |
| ORM | SQLAlchemy (async) | 2.0.51 |
| Validation/config | Pydantic / pydantic-settings | 2.13.4 / 2.14.1 |
| Database | PostgreSQL | 18.4 |
| DB driver | asyncpg | current |
| JWT | PyJWT | 2.13.0 |
| Password hashing | argon2-cffi (Argon2id) | 25.1.0 |
| Rate limiting | slowapi | 0.1.10 |
| Container | Docker Compose v2 | — |

**Audio engine — DECISION: NumPy + SciPy (`scipy.signal`) synthesis, soundfile (libsndfile) WAV I/O, stdlib `wave` fallback, stdlib `zipfile` for packs.**
*Rationale:* standard, documented, deterministic, dependency-light, offline; emits FL-friendly PCM 16/24-bit WAV at 44.1/48 kHz.
*Rejected:* pedalboard (effects not oscillators; GPLv3); pydub (unmaintained); neural APIs (cost/latency/GPU/licensing — optional future premium tier).

---

## 5. Risk Register

### Threat Model — Trust Boundaries
- **Boundaries:** Internet → reverse proxy (TLS, HTTPS-only); Next.js → FastAPI `/v1/`; API → PostgreSQL + synthesis worker; API → asset storage. Two trust levels: **anonymous** (rate-limited, capped, no persistence) and **authenticated** (JWT, owns library).
- **Primary assets:** accounts/credentials, JWTs, generated WAV/ZIP + metadata, CPU/RAM/disk (scarce resource).

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|------------|
| 1 | Resource exhaustion / DoS via expensive generation | High | High | Hard caps on duration/sample-rate/channels/layers; per-IP + per-user slowapi limits; bounded concurrency; per-request time budget. |
| 2 | Prompt injection into synth-param parser | Med | High | Never `eval`/`exec`; allow-list keyword→param map with clamping; ignore unknown tokens. |
| 3 | Path traversal on asset download | Med | High | Opaque UUID asset ids, never client paths; resolve within fixed root; sanitize Content-Disposition. |
| 4 | IDOR on library assets | Med | High | Owner == JWT `sub` check on every read/delete; non-sequential UUIDs; deny-by-default; tested. |
| 5 | JWT handling flaws | Med | High | PyJWT pinned `algorithms=["HS256"]`, strong env secret, short `exp` + refresh; HttpOnly+Secure+SameSite cookies. |
| 6 | Rate-limit bypass | Med | Med | Trust XFF only from known proxy; key by auth user; limits on generate/download/auth; `Retry-After`. |
| 7 | ZIP bomb / oversized pack export | Med | High | Cap files-per-pack, per-file + total uncompressed bytes; running byte budget abort; never re-zip user archives. |
| 8 | Storage exhaustion | Med | Med | Per-user quotas; TTL + cleanup; object storage; disk monitoring. |
| 9 | Malformed generation request (NaN/Inf/0Hz) | Med | Med | Pydantic `Field` bounds; guard NaN/Inf; cap duration×rate; try/except → RFC 9457. |
| 10 | Malformed WAV output | Low | Med | soundfile/`wave` correct headers; peak-normalize; fade in/out; validate before serving. |
| 11 | Anonymous-user abuse | High | Med | Stricter anon caps; auth required for packs/library. |
| 12 | Dependency / supply-chain risk | Low | Med | Maintained libs; pin + pip-audit; avoid passlib/stock pydub. |
| 13 | PII/credential exposure in logs | Low | Med | JSON logging with secret/token redaction; log ids not bodies. |
| 14 | Generated-audio licensing ambiguity | Med | Med | ToS grants users rights to procedural output; DSP origin avoids training-data disputes. |

---

## 6. Research Gaps
1. **DSP vs neural fidelity** — perceptual A/B with producers; may justify optional neural premium tier.
2. **Prompt→param mapping quality** — needs empirically tuned descriptor→param taxonomy.
3. **FL preset depth** — `.wav` universal; v1 ships WAV + optional Vital `.vital` JSON.
4. **Licensing of generated audio** — ToS/ownership model needs legal clarity.
5. **Generation CPU scaling** — benchmark sync-vs-queue threshold.
6. **`.vital` compression** — verify JSON vs gzip before emitting presets.
7. **RateLimit header standard still a draft** — implement defensively.

---

## 7. Summary & GO / NO-GO

**Synthesis.** Every component of the confirmed stack is real, current, maintained, and version-verified (Jun 2026): Next.js 16.2.9 / React 19.2.7 / Tailwind 4.3.1 / Zustand 5.0.14 frontend; FastAPI 0.137.2 / SQLAlchemy 2.0.51 async / Pydantic 2.13.4 / PostgreSQL 18.4 backend; PyJWT 2.13.0, argon2-cffi (Argon2id), slowapi 0.1.10 security. The audio engine path is clean and FL-compatible — NumPy + scipy.signal synthesis, soundfile/libsndfile WAV I/O (stdlib fallbacks), emitting PCM 16/24-bit WAV at 44.1/48 kHz.

**Differentiation & risk.** PatchLAB occupies an unoccupied cell: FL-producer-targeted, local/offline procedural generation of WAV samples *and* synth presets bundled as ZIP packs, no external-AI dependency, no per-gen cost, and — DSP-based — no licensing-lawsuit exposure. Dominant risks are resource economics/abuse, all with standard mitigations. Open questions are product fidelity and CPU scaling, not feasibility.

**Recommendation:** Stack sound, current, compatible; audio approach proven and FL-compatible; competitive gap real; principal risks mitigable. Open items are validation/tuning, not blockers.

**Go/No-Go: GO**

**Source counts:** 17 vendor/official + 8 standards (§3.1/§3.5); 14 GitHub repositories (§3.2); ~20 other (§3.3–§3.9).
