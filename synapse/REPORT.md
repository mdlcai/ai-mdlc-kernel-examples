# BUILD-REPORT — SYNAPSE

Receipts for a governed build. This is the record of what was built from
[`RESEARCH.md`](./RESEARCH.md), the architecture that was decided, the feature
slices that were implemented, the quality gates that were run, and the final
typecheck status.

- **Build depth:** `standard` (declared in `RESEARCH.md`)
- **Final typecheck:** `npx tsc --noEmit` — **exit 0** (clean, verified after
  clearing the incremental cache)
- **Method:** generated from `RESEARCH.md` through the MDLC kernel

---

## 1. What was built

SYNAPSE is a single, self-contained Next.js application that unifies **on-demand
network diagnostics** with **lightweight continuous monitoring**, exactly as
scoped in `RESEARCH.md §Product Vision`. An engineer runs a diagnostic once
(DNS, SSL, HTTP, reachability) and can promote any check into a monitor that runs
on a schedule and alerts on change.

Everything in `RESEARCH.md §Core Features` shipped, and nothing in
`§Non-Goals` did:

| Feature (RESEARCH.md) | Built | Where |
|---|---|---|
| DNS Lookup (A/AAAA/CNAME/MX/TXT/NS via DoH) | ✅ | `app/tools/dns`, `app/api/diagnostics/dns` |
| SSL Certificate Checker (issuer, validity, SANs, expiry) | ✅ | `app/tools/ssl`, `app/api/diagnostics/ssl` |
| HTTP Header Check (status, redirect chain, phase timing) | ✅ | `app/tools/http`, `app/api/diagnostics/http` |
| DNS Propagation (multi-resolver agreement/drift) | ✅ | `app/tools/propagation`, `app/api/diagnostics/propagation` |
| Monitors (create from any check type, interval, expected value) | ✅ | `app/monitors/*`, `app/api/monitors/*` |
| Scheduled runner (records check, computes status) | ✅ | `app/api/cron/route.ts`, `run-check.ts` |
| Alerting (email/webhook, once per transition) | ✅ | `app/api/cron/dispatch.ts`, `components/alerts/*` |
| Accounts (email/password, own-rows-only) | ✅ | `lib/auth.ts`, `app/api/auth/*`, `app/(auth)/*` |
| Payment/billing, multi-service edge, privileged scanning | ❌ (Non-Goal) | intentionally absent |

The four diagnostic tools are **public / no signup** (`RESEARCH.md §1`); a
verification confirmed no auth guard sits in `app/api/diagnostics/` — anonymous
use is by design. Each tool page carries a "monitor this" call to action that
routes an anonymous visitor toward signup, per `§Key Flows` flow 1.

---

## 2. Architecture decided

A **single deployable app** on Next.js App Router — no multi-worker split
(`RESEARCH.md §Tech Stack`: "No multi-worker split for this build"). The design
is organized as **four vertical feature slices over one shared data foundation**,
so ownership and SSRF invariants live in exactly one place each and every slice
inherits them.

```
Shared foundation (lib/ + db/)
  lib/db.ts      Postgres pool + ONLY ownership-scoped query helpers
                 (every user-owned helper takes userId as its first arg)
  lib/ssrf.ts    the single outbound-safety gate (assertPublicUrl / resolvePublicUrl)
  lib/auth.ts    scrypt hashing + opaque server-side sessions (one getSessionUser)
  lib/types.ts   shared row/DTO shapes — the only coupling between slices
  db/schema.sql  users, monitors, checks, alert_channels, alerts, sessions

Feature slices
  1. Diagnostics  app/tools/* + app/api/diagnostics/*   (public, no auth)
  2. Auth         app/(auth)/* + app/api/auth/*          (scrypt + sessions)
  3. Monitors     app/monitors/* + app/api/monitors/*    (owner-scoped CRUD)
  4. Alerting     app/api/cron/* (route + run-check + dispatch)
```

**Key architecture decisions, and why:**

- **Ownership is enforced in the data layer, not per-route.** `lib/db.ts`
  exposes no unscoped read/mutate path for `monitors`/`checks`/`alert_channels`/
  `alerts`; every helper folds `user_id = $1` (or a join back to `monitors.user_id`)
  into the query. This is what makes the IDOR invariant a structural property
  rather than a per-endpoint discipline (`RESEARCH.md §Data Model` ownership rule).
- **One system-level cross-user read, quarantined.** The cron scan
  (`listDueMonitors`) is the single intentional cross-user query — the scheduler
  is a system principal, not a logged-in user. It returns each row *with* its
  `user_id`, and every downstream mutation is re-scoped to that owner, so no
  unscoped mutation of a user's data occurs in feature code.
- **"Dispatch exactly once per transition" is enforced in the schema.** A partial
  unique index `alerts_one_open_per_monitor ON alerts (monitor_id) WHERE closed_at
  IS NULL` closes the TOCTOU window; `openAlert` uses `ON CONFLICT ... DO NOTHING`
  and returns non-null only on a genuine transition. The cron loop dispatches
  channels only when `openAlert` returns a new row (`RESEARCH.md §Alerting`).
- **Sessions are opaque + hashed at rest.** Raw token in an HttpOnly cookie, only
  the SHA-256 hash stored in `sessions.token_hash`. Passwords use scrypt
  (`N=16384, r=8, p=1`) with constant-time verification.
- **Alerting duplicates the check engine on purpose.** `app/api/cron/run-check.ts`
  reimplements DNS/SSL/HTTP checks separately from `app/api/diagnostics/core.ts`;
  the two slices coordinate only through `lib/types` shapes, keeping the scheduled
  path self-contained.

---

## 3. Feature slices

**Slice 1 — Diagnostics (public).** Four tools, one shared result layout
(`app/tools/_components/`). API handlers in `app/api/diagnostics/*` with the
shared engine in `core.ts`. All outbound probes pass through the `lib/ssrf.ts`
gate. No authentication — anonymous by design.

**Slice 2 — Auth.** scrypt password hashing, opaque server-side sessions,
case-insensitive unique email (`users_email_lower_idx`) for anti-enumeration.
`getSessionUser()` is the single principal accessor; it returns `null` for
anonymous visitors and never throws (so public tools keep working). Routes:
`signup`, `login`, `logout`.

**Slice 3 — Monitors.** Owner-scoped CRUD over `monitors`, `checks`,
`alert_channels`, `alerts`. Every route resolves the session user and calls the
`userId`-first `lib/db.ts` helpers. Monitor detail shows recent check history and
current status (`app/monitors/[id]`).

**Slice 4 — Alerting.** `POST /api/cron` authenticated by a shared `CRON_SECRET`
compared in **constant time** (not publicly triggerable, `RESEARCH.md §Security`).
Per due monitor: run the check → record it → update status → on a down/warning
transition open an alert and dispatch channels exactly once; on recovery, close
the open alert.

---

## 4. Quality gates run

| Gate | Result |
|---|---|
| **Typecheck** (`tsc --noEmit`, TS strict) | **PASS — exit 0** |
| **Ownership isolation / IDOR review** | **PASS** — no cross-user read/mutate path; verified user B cannot reach user A's rows via any endpoint (`RESEARCH.md §Success Criteria`) |
| **SSRF / outbound-request review (three lenses)** | Findings raised and **all remediated** — see `SECURITY-AUDIT.md` |
| **Integration pass** | **PASS** — cron→`dispatchAlert` wiring verified (dispatch fires only on a newly-opened alert row); no compile-breaking duplicate/conflicting files; home page gap closed (added the missing `/monitors` link on `app/page.tsx`) |
| **Non-Goal containment** | **PASS** — no billing, no multi-service edge split, no privileged scanning entered the build |

The security review is the substantive gate for this build. It ran three lenses
— **ownership isolation (IDOR)**, **SSRF / outbound-request safety**, and
**secrets & credential confidentiality**. IDOR came back a clean PASS. SSRF and
secrets produced findings, which were remediated and re-verified. Full detail,
per-finding, is in `SECURITY-AUDIT.md`.

---

## 5. Final typecheck status

```
$ npx tsc --noEmit
$ echo $?
0
```

Confirmed genuine (not a masked/incremental pass): the incremental cache
(`tsconfig.tsbuildinfo`) was deleted and a deliberate error was injected to prove
tsc actually type-checks these files, then reverted. No `any` was introduced
anywhere in the build or the remediation. The app builds and runs
(`RESEARCH.md §Success Criteria`: "Typecheck passes; the app builds and runs").

---

*Generated as part of a governed MDLC build. See `README.md` for "Built with
MDLC" and `SECURITY-AUDIT.md` for the security gate detail.*
