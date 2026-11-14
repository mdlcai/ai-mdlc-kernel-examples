# Security Audit — pass 4 (wide-close diff)

Base commit: e8666d6083f6d96c6895e2a7a3538c6015685ced
Scope: uncommitted working tree vs HEAD (wide close)

Residual: CRITICAL 0 · HIGH 0 · MEDIUM 0 · LOW 2

## Scope reviewed

`git diff --ignore-cr-at-eol --stat HEAD -- apps/ e2e/` shows 10 files with content changes:
`(console)/dashboard/{dashboard-client.tsx,dashboard.module.css,page.tsx}`, `(console)/layout.tsx`,
`(console)/settings/settings-client.tsx`, `app/page.tsx`, `components/shell/ConsoleShell.tsx`, `lib/api.ts`,
`stores/session.ts`, `e2e/w6-apex.spec.ts`. There are 3 new untracked files: `lib/server-api.ts`, `lib/use-hydrated.ts`,
`components/IntentLink.tsx`. The other paths that `git status` lists as modified (apps/api, apps/worker,
`public/sw.js`, the UI components and so on) have line-ending-only differences and no content change. I read every hunk
and all three new files in full. I also read the code they depend on: `proxy.ts`, `lib/security-headers.ts`,
`app/layout.tsx`, `public/sw.js`, `next.config.ts`, the API `/auth/session` route, `middleware/ratelimit.ts`,
`app.ts` (trust proxy), `deploy/caddy/Caddyfile`, Next 16.4.1 `client/components/prefetch.js` and zustand 5.0.15
`esm/react.mjs`.

## 1. Cross-request / cross-user leakage on the server

- **React `cache()`** (`apps/web/src/lib/server-api.ts:29`): `serverSession = cache(() => serverGet('/auth/session'))`.
  In React Server Components, `cache()` memoises per server request only, so one user's result is never shared with
  another request. The function takes no arguments, and the cookie is read inside it through `headers()`, which is
  scoped to the request. PASS.
- **zustand store on the server** (`apps/web/src/stores/session.ts:39`): the store is a module-level singleton. In
  zustand 5.0.15, `useStore` passes `api.getInitialState()` as the `getServerSnapshot` argument of
  `useSyncExternalStore` (`node_modules/zustand/esm/react.mjs:6-10`). Server renders therefore always read the
  initial `idle` state. Every `setState` on the session store runs in a client effect or event handler:
  `ConsoleShell.tsx:53`, `LoginForm.tsx:39`, `RegisterForm.tsx:40`, `AcceptInviteForm.tsx:69`. None runs during
  render, so the module-level store is never written on the server. The per-user server value travels through the
  `SessionSeed` React context (`session.ts:65`), which belongs to each render tree and is not module state. PASS.
- **Fetch cache / full-route cache**: `serverGet` uses `cache: 'no-store'` (`server-api.ts:19`). The root layout
  awaits `headers()` outside any try/catch (`app/layout.tsx:49`), so every route is already dynamic. The new
  `headers()` call inside the try in `server-api.ts:13` cannot turn a route static. The console HTML is therefore
  rendered per request and gets Next's dynamic `Cache-Control: private, no-cache, no-store`, and `proxy.ts` does not
  override it. PASS.
- **Client router cache**: `IntentLink` calls `router.prefetch(href)` with no `kind`. In Next 16.4.1 this resolves to
  `PrefetchKind.AUTO`, which uses the PPR strategy (`next/dist/client/components/prefetch.js:38-47`). It does not
  prefetch the full dynamic payload, so a prefetched console route never carries one user's session or summary for
  the 300 s static stale time. Back/forward history entries are a different matter: see **SEC-12**.

## 2. SSRF / header injection / CSRF token exposure

- **URL and path**: the origin comes from `process.env.API_INTERNAL_URL` or the constant `http://127.0.0.1:4000`
  (`server-api.ts:15`). Both call sites pass constant paths (`'/auth/session'` and `'/dashboard/summary'`) and a
  constant query (`{ range: '24h' }`, `dashboard/page.tsx:9`). The query is encoded with `URLSearchParams`. No
  request-derived value reaches the URL. PASS.
- **Forwarded headers**: only `cookie` (the visitor's own header, unchanged) and `accept` are sent
  (`server-api.ts:18`). `Host`, `X-Forwarded-For`, `X-Forwarded-Host` and other client headers are not forwarded, so
  a client cannot spoof the IP or host the API sees. Undici rejects CR/LF in header values, so there is no header
  splitting. PASS. A side effect of not forwarding the IP is covered in **SEC-13**.
- **CSRF token in the RSC payload**: `SessionInfo.csrfToken` (`session.ts:23`) is now serialised into the
  server-rendered HTML/flight data as the `initialSession` prop (`(console)/layout.tsx:10`, `ConsoleShell.tsx:71`).
  The exposure does not materially increase, for these reasons:
  - The token is already readable by JavaScript in the non-HttpOnly `__Host-cz_csrf` cookie
    (`apps/api/src/middleware/cookies.ts:29`). It was also already in the JSON body of `GET /auth/session`.
  - The client reads the token from the cookie, not from the store (`lib/api.ts:77-78`).
  - The console HTML is dynamic `no-store`, as shown in check 1.
  - The service worker caches only `/offline` and static asset prefixes. Navigations are network-first and are not
    stored, and RSC fetches fall through uncached (`public/sw.js:48-74`).
  - Browsers exclude `no-store` pages from bfcache, or evict them when cookies change, as on logout.

  Advisory: strip `csrfToken` from the session before passing it as a prop. The client does not use it.

## 3. Auth bypass / error handling

- `serverGet` returns `null` when there is no cookie, on a non-2xx response (including 401), on a JSON parse
  failure and on the 3 s timeout (`server-api.ts:14,22-24`).
- When `initialSession` is null, `seed` is null (`ConsoleShell.tsx:42`). The effect then calls `load()`
  (`ConsoleShell.tsx:52-54`). A 401 sets `status: 'signed-out'` (`session.ts:49`), and the shell redirects to
  `/login?next=…` (`ConsoleShell.tsx:57`). Children render only when `status === 'ready'`
  (`ConsoleShell.tsx:153`). The signed-out redirect is preserved.
- The seed applies only while the store is `idle` (`ConsoleShell.tsx:42`, `session.ts:72-75`). After logout
  (`clear()` sets `signed-out`), a stale seed is ignored, the shell shows no user info and does not render children,
  and it redirects.
- `proxy.ts:13-18` is unchanged and still sends a 303 to `/login` for any non-public path without
  `__Host-cz_session`. As before, the matcher skips requests that carry `next-router-prefetch` or
  `purpose: prefetch`. A cookieless prefetch makes `serverGet` return null, so no data is fetched.
- The server-read role (`initialRole`, `dashboard/page.tsx:13`) controls only UI visibility. The API still
  authorises every call.

PASS.

## 4. XSS

- No new `dangerouslySetInnerHTML`. The only one in `src` is the existing nonce'd theme bootstrap at
  `app/layout.tsx:53`.
- `IntentLink` hrefs are constants: the nav items and the landing-page links in `app/page.tsx`. `router.prefetch`
  blocks `javascript:` URLs.
- Server-provided summary and session strings render as React text children. The only data-derived href is
  `/apex/${f.id}` (`dashboard-client.tsx:485`), which is pre-existing, relative and built from an API UUID.
- React escapes the RSC payload when it is embedded.

PASS.

## 5. SEC-11 (credential forms use method="post")

- The fix is present: the TOTP enable/verify form now has `method="post"` (`(console)/settings/settings-client.tsx:223`).
- Every form with a password or one-time-code field has `method="post"`:
  - settings-client.tsx:137, 223 and 254
  - team-client.tsx:279 and 462
  - approvals-client.tsx:372 and 454
  - workload-detail-client.tsx:440 and 544
  - object-client.tsx:233
  - LoginForm.tsx:59, RegisterForm.tsx:66, AcceptInviteForm.tsx:124
- Three forms have no method. None contains a credential field:
  - the profile display-name form (settings-client.tsx:80)
  - the organization form (settings-client.tsx:347)
  - the evidence filter (evidence-client.tsx:251)

PASS.

## 6. Other observations

- `lib/api.ts:107`: the 5xx fallback title is now a fixed product string instead of `res.statusText`. This leaks
  nothing. Neutral or a slight improvement.
- `use-hydrated.ts`, the `formatters` and `FormatCtx` code in `dashboard-client.tsx`, the CSS change and the e2e
  spec change have no security relevance.
- Availability: each console layout render and each dashboard render can block for up to 3 s per API read
  (`server-api.ts:20`). The reads run in parallel on the dashboard. A slow API degrades first paint but does not
  fail open.

## Findings

### SEC-12 — Previous user's server-seeded data can be restored from client history after an in-tab account switch (LOW)

- **Location:**
  - `apps/web/src/app/(console)/dashboard/dashboard-client.tsx:125-135`: `initial` is trusted as the answer to the
    first request.
  - `apps/web/src/app/(console)/layout.tsx:10`
  - `apps/web/src/components/shell/ConsoleShell.tsx:203-209`: sign-out uses `router.replace('/login')`.
  - `apps/web/src/app/login/LoginForm.tsx:39-40`: login uses a soft `router.replace`.
- **Exploit path:**
  1. User A uses the console in a tab, then signs out. Sign-out is a soft client navigation, so the router's
     in-memory history and segment cache keep A's RSC payloads: `initialSession` (including A's `csrfToken`) and
     the dashboard `initial` summary.
  2. User B signs in in the same tab, also through a soft navigation.
  3. B presses Back into A's history entry. Back/forward navigation reuses cached segments. The shell shows B's
     identity because the store is `ready` with B. But `DashboardClient` mounts with A's cached `initial` summary
     and shows it without fetching, because the seeded key matches. B sees A's tenant KPIs, finding titles and job
     status.

  Before this change the dashboard always fetched on mount with the current cookie, so this path did not exist.
  The impact is limited: it needs the same browser tab, B must be an authenticated user, the data is read-only,
  and no writes are affected because the API authorises B's cookie. A signed-out visitor who presses Back sees
  nothing, because children are gated on `status === 'ready'`.
- **Remediation (any one):**
  - Make sign-out and sign-in hard navigations (`window.location.replace(...)`), or call `router.refresh()` after
    `clear()` and after login, so the client router cache is discarded.
  - Or have `DashboardClient` (and any future seeded screen) ignore `initial` unless the seeded session's
    `user.id`/`tenant.id` matches `useSession.getState().info`.
  - Also omit `csrfToken` from the seeded `SessionInfo`.

### SEC-13 — Server-side API reads share one per-IP rate-limit bucket keyed on the web container's address (LOW)

- **Location:**
  - `apps/web/src/lib/server-api.ts:17-21`: no client-IP propagation.
  - `apps/api/src/middleware/ratelimit.ts:18-23`: the 300/min `ip:${req.ip}` bucket.
  - `apps/api/src/app.ts:46`: trust proxy is set to `loopback, uniquelocal`.
- **Exploit path:**
  1. Every server read reaches the API from the web service directly, without `X-Forwarded-For`, so `req.ip` is
     the web container's (or 127.0.0.1) address for all users.
  2. A full load of a console route costs one read, and `/dashboard` costs two. `proxy.ts` checks only that a
     `__Host-cz_session` cookie exists, not that it is valid. So an unauthenticated client can loop
     `GET /dashboard` with `Cookie: __Host-cz_session=x` and drain the shared bucket.
  3. Heavy legitimate traffic drains it as well.
  4. Once the bucket is drained, every user's server reads get 429 and return null. Users fall back to the client
     fetch, which uses their own IP through Caddy.

  The result is a degraded first paint and extra API load (up to 2 internal calls per anonymous request). There is
  no auth bypass or data exposure. Not forwarding the client's `X-Forwarded-For` is the correct choice, because
  forwarding it would let clients spoof the rate-limit key.
- **Remediation:** give the web service its own trusted channel. For example, a shared-secret header that the API
  verifies, with its own higher bucket or keyed on the session/tenant. Alternatively, pass the edge-verified client
  IP in a header that the API trusts only from the web service's address. Also consider skipping `serverGet` when
  the session cookie value is not well-formed.

## Verdict

PASS WITH ADVISORY. None of the in-scope checks fails:
- no cross-request leakage on the server
- no SSRF or header injection
- the signed-out redirect is intact
- no new XSS sinks
- SEC-11 is fixed and every credential form posts

The two LOW findings (SEC-12, SEC-13) are advisories.
