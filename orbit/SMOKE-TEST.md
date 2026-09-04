# SMOKE-TEST.md — manual verification checklist

Run against the deployed stack (`https://localhost:8443` locally). The automated equivalent is `npm run smoke` (`smoke-test.sh` → `smoke-test.log`); the UI flows are the Playwright `W1`–`W9` specs (`npm run test:e2e`).

## Application starts
- [ ] `docker compose ps` shows `postgres`, `api`, `web`, `caddy` **healthy** and `migrate` **exited (0)**.
- [ ] `GET https://localhost:8443/api/health` → 200 `{"status":"ok", "checks":{"db":{"status":"ok"},"scheduler":{"status":"ok"},"mailer":{...}}}` in < 500 ms.
- [ ] `http://localhost:8080/anything` → 308 redirect to `https://localhost:8443/anything` (no plaintext content is ever served).

## Homepage & shell
- [ ] `/` renders the landing page in both themes; the theme toggle persists across reload.
- [ ] At 375 px the navigation toggle opens a focus-trapped drawer with every primary destination; `Escape` closes it and focus returns to the toggle.
- [ ] `/no-such-route` shows the designed 404 with a link home.

## Registration, sign-in, sessions
- [ ] `/register` creates a workspace and lands on `/dashboard` (onboarding card visible on an empty workspace).
- [ ] `/login` with a wrong password shows one uniform message; with the right password lands on `next` or `/dashboard`.
- [ ] Submitting `/register` with a 10-character password shows the message under the **password** field (not a generic banner).
- [ ] Sign out clears the session; `/dashboard` redirects to `/login?next=/dashboard`.
- [ ] `/forgot-password` always shows "If that email exists…"; the reset link (from `pending_emails.payload` or SMTP) works once.

## Primary CRUD through the UI
- [ ] Create a ticket from `/tickets/new` (category with defaults) → detail shows number, status `new`, SLA due dates.
- [ ] Edit priority inline → saved; open the same ticket in a second tab, edit again in the first, then save in the second → "This ticket changed — reload" (409) keeps your edits.
- [ ] Add a comment; add a private note as an agent; sign in as the requester → the private note is absent from the detail and timeline.
- [ ] Admin deletes the ticket → it disappears for non-admins.

## Capability features generalize (not just seed data)
- [ ] KB search: publish an article whose **body** mentions "authenticator app re-enrolled", then search `authenticator phone lost` → the article is returned.
- [ ] Global search (⌘K): type a fragment of a newly created asset tag → the asset appears under Assets.
- [ ] Automation: create a rule for a category you just created (not the seeded "Network issue") → a new ticket in that category is routed accordingly and appears in the Runs tab.

## Authorization isolation
- [ ] Cross-tenant: an admin of org B requesting org A's ticket URL gets the designed 404.
- [ ] Same-tenant cross-user: requester 2 opening requester 1's ticket URL, its `/comments` and `/timeline` API routes gets 404 for every one of them.
- [ ] A requester visiting `/admin/users` or `/analytics` gets the 404 page; the sidebar never lists those destinations for them.

## Background jobs
- [ ] `pending_emails` rows move `pending → sent` within one scheduler tick (15 s); with `SMTP_URL` unset the API log shows `mail (json transport)`.
- [ ] Set a ticket's SLA policy to a 1-minute resolution target (Admin → SLA) → within 30 s after due the ticket shows **breached**, the assignee gets a notification, and the `sla.breached` rule fires.

## Realtime
- [ ] Open the queue in two browsers; assign a ticket in one → the other updates within 2 s without reload; the bell increments.

## Unhappy paths (structured error contract)
- [ ] `POST /api/v1/tickets` with `{"title":"ab"}` → `application/problem+json` with `errors[0].field = "title"`, and the UI shows that message under the title input.
- [ ] `PATCH /api/v1/tickets/:id` without `If-Match` → 428 `PRECONDITION_REQUIRED`.
- [ ] 11 failed logins in 15 minutes → 429 `RATE_LIMITED` with `Retry-After`; the UI shows the wait time.
- [ ] Expired session cookie → 401 `UNAUTHENTICATED`; the UI redirects to sign-in and returns to the original route after.

## Operations
- [ ] Start the API with `DATABASE_URL` unset → exits non-zero printing `DATABASE_URL`.
- [ ] `docker compose stop api` while a request is in flight → the request completes 2xx; the container stops within 10 s.
- [ ] Replay `/` and `/tickets` with a 20 KB `Cookie` header → 200 (never 400/431).
- [ ] Demo seed: `SEED_DEMO=true` creates the `demo` org, 8 tickets, 6 assets, 4 KB articles, 1 project, 1 change, 2 rules.
