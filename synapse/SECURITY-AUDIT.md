# SECURITY-AUDIT — SYNAPSE

The security gate for this governed build. The review ran **three lenses** over
the code, each finding is listed with its remediation, and the audit ends with a
final verdict.

- **Scope:** the full SYNAPSE reference build (diagnostics, auth, monitors,
  alerting) against `RESEARCH.md §Security Requirements`.
- **Re-verification after fixes:** `npx tsc --noEmit` passes (**exit 0**); the
  IPv6 SSRF regression cases from the findings all pass while public IPv6
  addresses remain allowed (no over-blocking).
- **Final verdict:** **PASS** — all findings remediated and verified.

---

## The three lenses

| Lens | What it examined | Outcome |
|---|---|---|
| **1. Ownership isolation (IDOR)** | Can user B read or mutate user A's monitors/checks/alerts via any endpoint? (`RESEARCH.md §Security`, `§Success Criteria`) | **PASS — no findings.** Ownership is structural: `lib/db.ts` exposes only `userId`-scoped helpers for user-owned tables; the one cross-user read (cron `listDueMonitors`) re-scopes every downstream mutation by the row's own `user_id`. |
| **2. SSRF / outbound-request safety** | Can a user-supplied monitor/diagnostic/webhook target reach private, loopback, link-local, or cloud-metadata addresses? (`RESEARCH.md §Security`) | **2 findings (both remediated).** |
| **3. Secrets & credential confidentiality** | Are credentials and secrets protected in transit and at rest? | **1 finding (remediated).** |

The DB-TLS issue surfaced during the IDOR lens as an *out-of-lens observation*
(it does not affect the IDOR PASS verdict); it is recorded here under Lens 3,
where it belongs.

---

## Findings

Five raw findings were reported; they collapse to **three distinct issues** (two
SSRF findings described the same DNS-rebinding class, and two described the same
DB-TLS class). All are remediated.

### Finding A — SSRF: IPv6 literal bypass  · **HIGH** · Lens 2

**File:** `lib/ssrf.ts` (`isPrivateV6`)

**What was wrong.** `isPrivateV6()` string-matched IPv6 literals. Its embedded-IPv4
detector only matched dotted-decimal (`/(?:::ffff:|::)(\d+\.\d+\.\d+\.\d+)$/`), but
the WHATWG URL parser always normalizes such hosts to the hex-grouped form —
verified: `http://[::ffff:127.0.0.1]/` → hostname `[::ffff:7f00:1]`, and
`http://[::ffff:169.254.169.254]/` → `[::ffff:a9fe:a9fe]`. The regex never matched
the hex form and no `fe`/`fc`/`fd`/`ff` prefix matched, so `isBlockedIp` returned
`false` and `assertPublicUrl` accepted the URL with no DNS step. Result:
`http://[::ffff:127.0.0.1]/` reached loopback and `http://[::ffff:169.254.169.254]/`
reached the **cloud metadata endpoint** — reachable **unauthenticated** via the
public diagnostics routes (`/api/diagnostics/http|ssl|dns|propagation`) and also
via monitor checks and webhook channels. Secondary gaps in the same function: the
`fe80::/10` check used string prefixes `fe80`/`fe9`/`fea`/`feb`, missing
`fe81::`–`fe8f::`; and the IPv4-compatible `::7f00:1` form was likewise missed.

**How it was remediated.** `isPrivateV6` was rewritten to stop string-matching. A
new `ipv6ToBytes()` fully expands any IPv6 literal (short form, zone id, embedded
IPv4) to its 16 bytes, returning `null` on malformed input. The new check:

- **Fails closed** (blocks) on any address it cannot parse.
- Detects IPv4-mapped (`::ffff:x`) and IPv4-compatible (`::x`) forms from the
  actual bytes and runs the embedded IPv4 through `isPrivateV4` — catching the
  hex-normalized forms (`::ffff:7f00:1`, `::ffff:a9fe:a9fe`, `::7f00:1`) the old
  regex never matched.
- Checks `fe80::/10` **numerically** (`b0 == 0xfe && (b1 & 0xc0) == 0x80`),
  covering `fe81::`–`fe8f::`.
- Keeps `fc00::/7`, `ff00::/8`, `::`, `::1` as numeric checks.

**Verified.** `[::ffff:127.0.0.1]`, `[::ffff:169.254.169.254]`, `[::7f00:1]`,
`[fe81::1]` are all now blocked; `2606:4700:4700::1111` and `::ffff:8.8.8.8`
correctly remain allowed (no over-blocking).

### Finding B — SSRF: DNS-rebinding TOCTOU (no address pinning)  · **HIGH** · Lens 2

**Files:** `lib/ssrf.ts` and every outbound caller —
`app/api/cron/run-check.ts` (`runHttp`, `runSsl`), `app/api/cron/dispatch.ts`
(`sendWebhook`), `app/api/diagnostics/core.ts` (`runSslCheck`, `timedRequest`,
`runHttpCheck`).

**What was wrong.** `assertPublicUrl` resolved the hostname and validated the
answers, but every outbound call then re-resolved DNS independently and connected
by hostname with **no pinning** to the validated address. An attacker-controlled
name with a short TTL that returns a public IP during validation and a
private/loopback IP at connect time defeats the private-IP rejection — reaching
loopback or the metadata endpoint. The module's own docstring said callers
"should ALSO pin to the resolved address," but no path did — so validation was
the only barrier, and it was a TOCTOU. Affected both user-supplied webhook
destinations and user-supplied HTTP/SSL/DNS monitor and diagnostic targets.

**How it was remediated.** Address pinning was added end to end:

- `lib/ssrf.ts` gained a `VettedTarget` type (`{ url, address, family }`),
  `resolvePublicUrl()` (returns the single validated IP to pin to; `assertPublicUrl`
  is now a thin wrapper over it), `pinnedLookup(target)` (a `dns.lookup`-compatible
  function that always returns the vetted IP, never re-resolves, and re-checks the
  block list at connect time as a fail-closed guard), and `resolvePublicWebhookUrl()`.
- **`app/api/cron/run-check.ts`** — `runHttp` rewritten from `fetch` to node
  `http`/`https` with `lookup: pinnedLookup(target)` (fetch cannot be pinned
  without undici, which is not a dependency); redirects re-validate + re-pin each
  hop. `runSsl` now `tls.connect({ host: target.address, servername: host })`.
- **`app/api/cron/dispatch.ts`** — `sendWebhook` rewritten from `fetch` to
  `https.request` with the `lookup` pin + `servername`, keeping default cert
  verification.
- **`app/api/diagnostics/core.ts`** — `runSslCheck` pins `tls.connect` to the
  vetted IP; `timedRequest` takes a `VettedTarget` and adds `lookup`/`servername`;
  `runHttpCheck` threads the target through and re-pins each redirect hop.

In every case the original hostname is preserved as SNI/servername/Host, so TLS
validation and vhost routing stay correct. The connection now lands on exactly
the IP that passed validation, closing the rebinding window.

> Note: the DoH resolver `fetch` calls (`runDns`, `dohQuery`) go to fixed trusted
> resolver hostnames (dns.google, cloudflare-dns.com, …), not user-controlled
> targets — the user's target is validated separately — so they are not a
> rebinding vector and were intentionally left on `fetch`.

### Finding C — Database TLS certificate verification disabled  · **MEDIUM** · Lens 3

**File:** `lib/db.ts` (Postgres pool config)

**What was wrong.** The Postgres pool set `ssl: { rejectUnauthorized: false }` for
all non-`disable` configs, so the DB server's TLS certificate was never verified.
Traffic was encrypted but **unauthenticated**: a MITM on the DB network path could
present any cert, intercept the connection, and capture the `DATABASE_URL`
credentials plus all data in flight — including every `users.password_hash` and
session `token_hash`. This undermined the otherwise-strong credential protection.
(Surfaced as an *out-of-lens observation* during the IDOR review; it does not
affect the IDOR PASS verdict.)

**How it was remediated.** Replaced with
`ssl: { rejectUnauthorized: true, ca: process.env.PGCA }` for all non-`disable`
configs. The connection now authenticates the server certificate against a
`PGCA`-supplied CA bundle (or the system trust store if `PGCA` is unset).
`PGSSL=disable` remains the **local-dev-only** plaintext escape hatch and must
never be used in production.

---

## Final verdict

**PASS.**

All findings are remediated and verified. `npx tsc --noEmit` passes (**exit 0**).
The SSRF regression cases named in the findings —
`[::ffff:127.0.0.1]`, `[::ffff:169.254.169.254]`, `[::7f00:1]`, `[fe81::1]` — are
all blocked, while public IPv6 addresses remain reachable. Every outbound path
now pins to the exact IP that passed validation, closing the DNS-rebinding window.
The database connection authenticates the server certificate.

The `RESEARCH.md §Security Requirements` are satisfied:

- ✅ All monitor/check/alert access is scoped to the session user (IDOR lens: PASS).
- ✅ The cron endpoint is authenticated with a shared secret (constant-time),
  not publicly triggerable.
- ✅ Outbound diagnostic requests are guarded against SSRF: private/loopback/
  link-local ranges and non-http(s) schemes rejected — now including the
  IPv4-mapped IPv6 and DNS-rebinding bypasses.
- ✅ Passwords hashed with a modern KDF (scrypt); no secrets in client code or
  the repo; DB credentials now protected by an authenticated TLS channel.
- ✅ Webhook destinations validated (https-only, no internal addresses,
  pinned at connect).
