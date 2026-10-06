# Review Round 3: Wide close

- **Commits reviewed:** RB-3 `c5dd7a9` and RB-4 `3fee33a` (`git diff c230639..3fee33a -- web compose.yaml SPEC.md ARCHITECTURE.md`)
- **Date:** 2026-10-05
- **Reviewer:** independent Reviewer (read-only)
- **Context:** ADR-031 (DECISIONS.md:351-378), SPEC.md §6 / §env, ARCHITECTURE.md §9, CLAUDE.md conventions

## Checks run

| Check | Result |
|---|---|
| `npm run typecheck` | clean (server + web) |
| `npm run lint` | clean (`--max-warnings 0`) |
| `cd web && npx vitest run` | 8 files, 30/30 tests pass (includes the new `SessionContext.test.tsx` and `useHydrated.test.tsx`) |
| `npm run lint:invariants` | 41/42 machine checks pass. INV-2 is not passing because it is stale: see R3-8 |

## Scope verification summary

1. **server-session.ts security.** The base URL is fixed: it comes from `process.env.API_INTERNAL_URL` (server-session.ts:12), and only a constant path is resolved against it (:21), so request input cannot redirect the call (no SSRF). The only headers forwarded are `accept`, the single `__Host-tw_session` cookie (rebuilt from `cookies()`, :14,17) and the incoming `x-forwarded-for` (:18-19).
   - The token is base64url (`server/src/lib/crypto.ts:37`), so it holds no `;`, CR or LF. undici rejects invalid header bytes in any case, and that rejection lands in the `catch` (:23), so a bad header becomes `null`.
   - Caddy (`Caddyfile:32-34`) has no `trusted_proxies`, so it discards any XFF the client supplies and sets its own. With `trust proxy 1` (`server/src/api/app.ts:18`), the API's `req.ip` is therefore the real client.
   - Neither `web` nor `api` publishes a host port (compose.yaml:57-79), so nobody can bypass Caddy to spoof XFF.
   - The 2 s timeout uses `AbortSignal.timeout` (:21). Every failure returns `null`, and the client path takes over.
   - Nothing is logged. The RSC payload carries exactly the `SessionInfo` that `GET /auth/session` returns (`server/src/api/routes/auth.ts:121-123`), and that route sets no cookie.
   - Calling `cookies()` makes the route dynamic, so the HTML is not shared across users.
2. **SessionContext.** The provider starts in `ready` from `initial` and skips only the first effect run (SessionContext.tsx:27-28,48-54).
   - When the server read misses (expired or revoked session, API down, variable unset), `initial` is `null`. The client fetch then runs, and its 401 branch still calls `router.replace('/login?next=…')` (:38-40). Tests cover this (SessionContext.test.tsx:524-535).
   - `refresh()` is still the client `load` (:76). The account password change (AccountView.tsx:56) and the organization page (OrganizationView.tsx:73) refresh through it.
   - Logout unmounts the provider via `router.replace('/login')`.
   - The forced password change redirect (:58-60) works from the server-resolved state.
   - StrictMode is on (next.config:17). In dev only, the second effect run performs one extra session fetch, which is harmless.
3. **settle() coalescing.** Each queued apply checks its own `AbortController` when the batch flushes (dashboard/api.ts:44-55). A superseded request (aborted by the next `fetchData`, :39) and an unmounted hook (aborted at :71) are therefore no-ops. An older result can never overwrite a newer one.
   - An urgent `setLoading(true)` issued after a flush but before the transition commits is applied in insertion order, so the final state stays correct.
   - The promise that `fetchData` returns now resolves before state is applied. No consumer awaits it: `reload` is `() => void load()` (:73) and every `useResource` consumer uses `reload` only (DashboardView.tsx:54,217,257; MapView.tsx:57; widgets.tsx:242). The `await load()` calls in AlertsView and IntegrationDetail are those screens' own loaders.
   - Live refreshes are debounced by 750 ms (LiveContext.tsx:167), so the extra 50 ms does not starve applies.
4. **Hydration sweep (`(app)` screens).** Grep for `Date.now`, `new Date()`, `Intl`, `toLocale*`, `window`, `document`, `localStorage`, `navigator`, `matchMedia` and `Math.random`:
   - Every locale- or time-dependent formatter (`fmtTime`, `fmtAgo`, `fmtNum`, chart labels, `countryName`) formats only data fetched on the client, which is `null` on the server render.
   - `window`, `document` and `navigator` are used only in effects or handlers.
   - ThemeToggle uses a `useSyncExternalStore` with a `null` server snapshot.
   - WorldMap reads reduced motion in an effect.
   - The only SSR-rendered time-derived value is the reports date defaults (R3-7). The c5dd7a9 baseline visited every screen and recorded #418 only on `account`, which RB-4 fixes (AccountView.tsx:15-19 with useHydrated.ts:10-16).
5. **CSS pulse.** It is compositor-only (transform and opacity). Under reduced motion the global rule (globals.css:135-141) covers `*::after`: one 0.01 ms iteration, with no fill-mode, so it rests at the base `opacity: 0`. The ring is invisible, which matches the old behaviour. Parity gaps are in R3-1.
6. **SPEC/ARCH consistency.**
   - SPEC.md has the env table row for `API_INTERNAL_URL` (optional, `web`, unset means client fetch only) and the SessionContext note.
   - ARCHITECTURE.md's service table matches compose.yaml:66 and server-session.ts.
   - INV-15's scope ("outside web/", ARCHITECTURE.md:290) matches the ADR-031 rationale, and the lint passes INV-15.

## Findings

| ID | Severity | File:line | Finding | Required fix |
|---|---|---|---|---|
| R3-1 | MINOR | web/src/app/globals.css:91-98, 123-131 | Visual parity of the pulse is not exact, in two ways. (a) The `::after` disc is painted over the dot with no `z-index`, filled with `--color-pulse` (rgba red at 0.7/0.6, tokens.css:83,111). At the start of each cycle the telemetry-coloured dot is therefore tinted red, whereas the old `box-shadow` sat outside the dot only. (b) `scale(3.7)` is tuned for a 6 px dot ("6px dot + 8px ring"). Two of the three users are 8 px (`h-2 w-2`, AppShell.tsx:285, widgets.tsx:107), where the reach is about 10.8 px rather than 8 px. Stat.tsx:49 (6 px) matches. | Put the ring behind the dot: `z-index:-1` on `::after`, with `isolation:isolate` on `.tw-live`. Make the reach size-independent, for example `inset:-8px` animating from a small start scale up to `scale(1)`, or a per-size custom property. Non-blocking. |
| R3-2 | NOTE | web/src/styles/tokens.css:84,112 | `--color-pulse-end` is no longer referenced anywhere. | Remove it in a later cleanup, or keep it with a comment. |
| R3-3 | NOTE | web/src/lib/server-session.ts:23 | The bare `catch` makes every fallback silent: a timeout, a refused connection or a schema problem cannot be seen in the web logs. The client fallback keeps this functionally correct. | Optional: a debug-level log of the cause only, never the cookie or headers. |
| R3-4 | NOTE | DECISIONS.md:358 | ADR-031 says the rightmost XFF entry is "the one Caddy appended". Caddy 2.10 with no `trusted_proxies` replaces any XFF the client supplies rather than appending to it. The outcome is the same, and stronger: one entry, the real client. | Optional wording fix the next time DECISIONS.md is edited. |
| R3-5 | NOTE | web/src/state/SessionContext.tsx:27-28 | `initial` only seeds the first state. A later server re-render of the layout (for example `router.refresh()` in `signOut`, :67) does not re-sync it. This is fine today, because every session refresh goes through the client `load` and sign-out unmounts the provider. | None. If a future screen relies on `router.refresh()` to update the session, it must call `refresh()`. |
| R3-6 | NOTE | web/src/features/dashboard/api.ts:9-22 | `settle()` has no direct test for coalescing within 50 ms, a no-op apply after an abort or unmount, or newer-wins ordering. Correctness rests on inspection (see summary item 3). | Recommended: a fake-timer unit test for `useResource` covering two hooks settling in one window and an abort before the flush. |
| R3-7 | NOTE | web/src/app/(app)/reports/ReportsView.tsx:145-148 | The GenerateForm date defaults derive from `new Date()` during SSR. They use `toISOString` (UTC, :17), so the server and client agree except for a request that straddles UTC midnight. That would be a rare #418 on `/reports` for writers. | Optional: compute the defaults after hydration (`useHydrated`) or in an effect. |
| R3-8 | NOTE | FRONTEND-AUDIT.json (untracked), invariants INV-2 | INV-2 is not passing at HEAD only because the baseline is stale. The report is pinned to `c5dd7a9`, whose 5 unaccepted `account/console` entries are all React #418, the defect RB-4 fixes. No code defect remains in the diff, but the fix is verified only by inspection and unit test, not by the stack run. | The pending `web:baseline --full` at `3fee33a` (BUILD-STATE `next:`) must show zero `account/console` problems, with INV-2 passing before commit or ship. |

No BLOCKER or MAJOR findings.

Verdict: PASS
