# SMOKE-TEST.md — BookFlow

Manual smoke checklist. The automated equivalent is `scripts/smoke-test.sh`
(`npm run smoke`); the e2e UI equivalent is `apps/web/e2e/workflows.spec.ts`.

## Bring up
- [ ] `docker compose up --build` brings up db · api · worker · web with no crash.
- [ ] `curl http://localhost:4000/api/health` → `{"status":"ok","db":"up",...}`.
- [ ] `http://localhost:3000` renders the landing page.

## Core flows (UI)
- [ ] **W1** Register a new workspace at `/register` → "Check your email" → sign in → dashboard greets you.
- [ ] Sign in with the seeded `member@bookflow.local` / `Password123!`.
- [ ] **W2** Book the **Projector X1** (equipment) for a free future slot → status **confirmed** immediately.
- [ ] **W3** Book **Conference Room A** → status **pending approval**; sign in as `manager@bookflow.local`, open Approvals, Approve → booking **confirmed**.
- [ ] **W4** Book the room again, manager **Declines** with a note → requester sees the rejection note on the booking.
- [ ] **W5** Book the same resource/slot twice → second attempt shows "already booked" (no double-booking persists).
- [ ] **W6** As manager, create a workflow rule on `/app/workflows` → it appears in the list.
- [ ] **W7** As admin, add a resource on `/app/resources` → it appears and is bookable.
- [ ] **W8** Open `/app/reports` → utilization renders; **Export CSV** downloads a file with rows.
- [ ] **W9** Open `/app/audit` → entries (booking.create, approval.*, etc.) are listed.

## Capability generalization (non-seed input)
- [ ] Create a brand-new resource with a name you invent, then book it for an arbitrary future window → confirms/â€‹routes correctly (not just the seeded fixtures).

## Security / negative paths
- [ ] `GET /v1/bookings` with no token → 401 `problem+json`.
- [ ] Submit the new-booking form with an empty title / bad dates → field-level error on the offending control (not "Invalid input.").
- [ ] A second org (your W1 workspace) sees none of the `acme` org's bookings/resources.
- [ ] Wrong password and unknown email both return the identical `invalid_credentials` envelope.

## Data
- [ ] Seed present: org `acme`, three users, three resources, one workflow rule.
