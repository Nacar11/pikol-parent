# pikol

Pikol is a **pickleball court booking** system: one owner, many venues, many
courts, clock-aligned hourly slots, online payment via Maya. Customers book
and pay online (through a mobile app, still to be built); staff book walk-ins
and collect cash at the counter in the web console. Both paths write the same
tables and pass the same constraint.

This file is the **system-level** brief — it covers the whole product across
frontend and backend. Backend-specific rules live in
`pikol-backend/CLAUDE.md` and load automatically when working under that
directory.

## Repo layout

```
pikol-parent/        # this repo — docs, plans, ADRs, the ERD, this brief
├── pikol-backend/   # FastAPI + SQLAlchemy 2.0 async + Postgres. See its own CLAUDE.md.
└── pikol-frontend/  # Next.js 15 App Router. See its own CLAUDE.md.
```

The sub-apps are **separate git repositories**, not submodules, sitting as
siblings under this parent — the same arrangement as `asima-parent`. Each has
its own `origin` and its own `main`.

| Repo | GitHub | State |
|---|---|---|
| `pikol-parent` | `Nacar11/pikol-parent` (public) | docs only, no code |
| `pikol-backend` | `Nacar11/pikol-backend` (public) | user groups done, mock Maya |
| `pikol-frontend` | `Nacar11/pikol-frontend` (public) | staff console, staff only |

Keep them separated: no frontend code in the backend tree, no backend code in
the frontend tree.

## Current state

**Phases 1–8 are complete, plus the user-groups and console-redesign work
inserted before Phase 9** (ADR 0003). Fixed roles are gone: permissions come
from venue-owned user groups, evaluated per active venue, and a built-in
Customer group is what lets an account book as itself. The web is the staff
console only — customers will use a mobile app, so the web player pages were
removed and the backend's customer routes stay for that app. The console is
black and white in an asima-style sidebar, with the new Schedule layout,
Setup → Venue staff for a venue's masters, and Masters → User groups for the
owner. A cleanup pass (2026-10-27, [`docs/standard/cleanup-pass.md`](docs/standard/cleanup-pass.md))
added a real-stack Playwright suite (`npm run e2e`), dead-code checks in both
repos and shared console components. **Next: Phase 9, deployment.** Until the mobile app ships, customers
cannot book online. The real `MayaProvider` is deferred until sandbox keys
exist. The first `SUPER_ADMIN` comes from `uv run python -m src.users.cli
create-super-admin`.

The roadmap and its phase order live in
[`docs/plans/2026-09-09-implementation-roadmap.md`](docs/plans/2026-09-09-implementation-roadmap.md).
Nothing is deployed yet; §9 of the spec describes the *intended* deployment,
not a live system.

## Cross-cutting concepts (read before touching bookings, pricing, or payments)

The vocabulary below is **load-bearing**. Frontend and backend must use the
same words for the same things.

### A booking group is the unit; slots are the occupancy

A multi-hour session is **one `booking_groups` row with N `booking_slots`
rows** — one payment, one cancellation unit. Booking two courts for the same
hour is two groups. `booking_groups.court_id` is authoritative: a group is
exactly one court.

### Two status axes, not one

```
booking_status    PENDING · CONFIRMED · COMPLETED · NO_SHOW · CANCELLED · EXPIRED
payment_status    UNPAID · PAID · FAILED
```

- **Online:** `PENDING/UNPAID` → webhook → `CONFIRMED/PAID`
- **Walk-in:** `CONFIRMED/UNPAID` → staff collects → `CONFIRMED/PAID`

One column cannot express a staff-created walk-in that blocks its slot before
money changes hands. `booking_groups.payment_status` is **authoritative** —
it is what guards read and what the UI shows. `payments.status` tracks one
provider attempt, and a group may accumulate several when a player retries.

`IN_PROGRESS` is **derived, never stored**: `CONFIRMED ∧ now ∈ [starts_at,
starts_at + 1h)`. A stored value drifts between cron ticks; a derived one is
exact at every read.

`needs_resolution` is a **flag, not a status** — a paid-but-unhousable booking
genuinely is `EXPIRED/PAID`, and inventing a status would force every guard to
handle a non-state.

### ⚠️ The PENDING trap

Hold cleanup is **lazy** — it happens on the write path, not on a scheduler —
so a lapsed hold on an uncontested slot keeps its row indefinitely.
`booking_status = 'PENDING'` therefore **never means "currently holding" on
its own**. Every read must add `AND expires_at > now()`.

Miss it on the availability query and the grid is stale for 15 minutes. Miss
it on the hold-cap count and a player who abandoned checkout three times is
**permanently** unable to book, with nothing that ever clears it.

### A maintenance closure is a booking row

Closures are `booking_groups` rows with `type = 'MAINTENANCE'`, no player and
no payment — **not** a separate table. Everything that occupies a court-hour
lives in one table so that one partial unique index can protect it. See
[`docs/adr/0001-booking-slot-uniqueness.md`](docs/adr/0001-booking-slot-uniqueness.md);
read it before changing anything in this area.

### Price is frozen

`price_centavos`, `fee_centavos`, and `total_centavos` are frozen at creation
and **never recomputed** from `price_rules`. The owner raising the 6pm rate
must not retroactively change what somebody already paid.

### Venue scope is a mandatory parameter

Asima is single-tenant and has nothing to copy here. The threat is concrete: a
venue master at Venue A reading Venue B's bookings.

One person may hold groups at several venues, so **every staff request names
the venue it acts at in `X-Venue-ID`** (ADR 0003). The web console sends it on
every request. A person working at exactly one venue may omit it; one working
at several gets `400 VENUE_REQUIRED` without it, and a venue they hold no group
at is `403`.

```python
class VenueAccess:         # resolved once per request, from token + X-Venue-ID
    scope: VenueScope      # all venues for the owner; else exactly the header's
    permissions: frozenset # the caller's groups AT THAT VENUE only — never summed
    is_master: bool

class VenueScope:
    all_venues: bool       # SUPER_ADMIN
    venue_id: int | None   # the active venue, for everyone else
```

Permissions are **per venue, never summed**: a master at A who is front desk
at B holds only front-desk codes while acting at B.

Every repository method touching venue-owned data takes `scope: VenueScope` as
a **required argument with no default**. Omitting it must be a `TypeError` at
call time, not a silent leak in production — the failure mode to design
against is *forgetting*, and a missing filter looks entirely normal in review.

For `/{id}` routes the check is ownership-after-fetch returning **404, not
403** — a 403 confirms existence, which tells a master at Venue A exactly how
many courts Venue B has.

### Config is not parameters

- **`config/`** — environment: secrets, connection strings. Read once at boot.
  Changing it requires a restart.
- **`parameters/`** — business rules: prices, windows, caps. Stored in
  Postgres, tunable without a deploy.

Do not conflate them.

## User groups and permissions

There are no stored roles (ADR 0003). What someone may do is the **user groups**
they belong to:

| Group kind | Venue | How many | Who edits it |
|---|---|---|---|
| `CUSTOMER` | none | exactly one | the owner (members only — it holds no codes) |
| `MASTER` | one | one per venue | the owner |
| `STAFF` | one | any | the owner, or anyone holding `USER:Manage` at that venue |

- **Customer access and staff access are independent.** Customer membership is
  the right to book and pay as yourself (`require_customer`); staff access is a
  venue's `MASTER`/`STAFF` groups. One account may be both — a customer in the
  mobile app and staff in the web console.
- **`SUPER_ADMIN` is `users.system_admin`**, an unconditional bypass orthogonal
  to groups (asima ADR 0001). It is never a group.
- Every venue is created with a "Venue master" and a "Front desk" group.

Permission codes are `RESOURCE:Action`. The frontend gates UI from
`GET /api/v1/users/me/permissions` (a flat string array — the active venue's
codes when `X-Venue-ID` is sent, the union otherwise, which the server never
authorises on) — never by parsing anything client-side.

| Code | Seeded on Venue master | Seeded on Front desk |
|---|---|---|
| `VENUE:Manage` | owner only — never on any group | |
| `PARAMETER:Manage` | owner only — never on any group | |
| `COURT:Manage`, `PRICE:Manage`, `USER:Manage` | ✅ | — |
| `CLOSURE:Manage` | ✅ | ✅ |
| `BOOKING:ReadAny`, `CreateWalkIn`, `MarkPaid`, `SetOutcome`, `CancelAny` | ✅ | ✅ |

`parameters` moves money (fee percentage, default price), so it stays with the
owner; a venue master adjusts pricing through `price_rules` on their own courts.

**Escalation rules, enforced in the group service, each pinned by a test:**

- Owner-only codes are refused for every group, by anyone — the owner too.
- A non-owner adds or removes only codes they hold at that venue; codes they
  don't hold stay on the group as they were.
- Only the owner changes `MASTER` or `CUSTOMER` membership, or edits or
  deactivates a master.
- **The server decides a new group's venue and kind, and an invitee's venue —
  never the request body.** Taking them from a body the caller controls is how
  a master mints a second master or a credential at another venue.
- Nobody removes their own last `USER:Manage` at a venue (`SELF_LOCKOUT`).

## API contract conventions

- All routes under `/api/v1/...`, from a shared `API_PREFIX` constant.
  `/health` and `/health/ready` sit deliberately **outside** it —
  infrastructure, not API.
- List endpoints return `{ data, total, page, limit, has_more }`.
- Field names are **snake_case end-to-end** — DB column, domain object, and
  JSON payload. No camelCase translation layer at the boundary.
- **Money is integer centavos.** Never float, never a `Decimal` column.
- Times are `timestamptz` stored UTC. **Day boundaries are Manila-local** —
  never `starts_at::date`, which returns the wrong 24 hours and misplaces the
  12am–1am slot (which has its own price rule) onto the previous day.
- Every response carries `X-Request-ID`. Propagate it from frontend logs and
  any onward HTTP calls.
- Bearer JWT, 15-minute access token, 7-day refresh with rotation and
  server-side revocation.
- **Authentication is required everywhere**, including browsing availability.
- **Transitions are actions, not PATCHes** — `POST /bookings/{id}/cancel`,
  not a status field in a generic body.
- **Admin vs self-service is enforced by schema, never by `if (role)` or a
  group name.**
  `/admin/bookings/{id}` vs `/bookings/me`; identity comes from the token,
  never from the URL. Privileged operations get their own endpoints rather
  than extra fields on a shared body.

## The schema source of truth

[`docs/adr/pikol.dbml`](docs/adr/pikol.dbml) is **authoritative**. Change it
first, then write the migration from it — never the reverse. Paste it into
https://dbdiagram.io to render.

Tables marked ⚠️ are **incomplete in the diagram**. DBML cannot express
partial indexes, functional indexes, `NULLS NOT DISTINCT`, or CHECK
constraints, so those are written as copy-pasteable DDL in its **Appendix A**,
and seed data in **Appendix B**. A migration written from the diagram alone
looks right and is silently wrong — the two unique indexes in Appendix A are
what make double-booking and non-deterministic pricing impossible.

## Plans and todos — `tasks/` vs `docs/plans/`

Working files (`tasks/`) are per-repo and gitignored; committed plan snapshots
live **only** in this parent repo under `docs/plans/`, regardless of which
repo's feature they describe.

| Location | Tracked? | Role |
|---|---|---|
| `tasks/plan.md` | **Gitignored** | The currently-active plan. Mutable. |
| `tasks/todo.md` | **Gitignored** | The currently-active checklist. **Never committed.** |
| `pikol-parent/docs/plans/YYYY-MM-DD-<slug>.md` | **Committed (parent only)** | Audit snapshot, frozen at write time. |

`tasks/` is a *workspace*, not a *record*. Committing every checkbox tick
would bury diffs in noise; forgetting why a decision was made six months later
is its own pain. The `docs/plans/` snapshot solves the second without causing
the first.

Filenames in `docs/plans/` must start with `YYYY-MM-DD-`.

## Where authoritative docs live

- [`docs/plans/2026-09-09-court-booking-system-design.md`](docs/plans/2026-09-09-court-booking-system-design.md)
  — the full system design. Section numbers referenced throughout the code.
- [`docs/plans/2026-09-09-implementation-roadmap.md`](docs/plans/2026-09-09-implementation-roadmap.md)
  — the nine phases, their dependencies, and why that order.
- [`docs/plans/2026-09-09-phase-1-backend-foundation.md`](docs/plans/2026-09-09-phase-1-backend-foundation.md)
  — Phase 1, task by task. Complete.
- [`docs/plans/2026-10-21-phase-2-identity-auth.md`](docs/plans/2026-10-21-phase-2-identity-auth.md)
  — Phase 2, task by task. Complete.
- [`docs/plans/2026-10-21-phase-3-venues-courts-pricing.md`](docs/plans/2026-10-21-phase-3-venues-courts-pricing.md)
  — Phase 3, task by task. Complete.
- [`docs/plans/2026-10-21-phase-4-booking-core.md`](docs/plans/2026-10-21-phase-4-booking-core.md)
  — Phase 4, task by task. Complete.
- [`docs/plans/2026-10-21-phase-5-payments-mock.md`](docs/plans/2026-10-21-phase-5-payments-mock.md)
  — Phase 5 on the mock provider. Complete; the Maya adapter follows.
- [`docs/plans/2026-10-21-phase-6-frontend-foundation.md`](docs/plans/2026-10-21-phase-6-frontend-foundation.md)
  — Phase 6, the frontend foundation and auth screens. Complete.
- [`docs/plans/2026-10-22-phase-7-player-booking-flow.md`](docs/plans/2026-10-22-phase-7-player-booking-flow.md)
  — Phase 7, the player booking flow. Complete.
- [`docs/plans/2026-10-22-phase-8-staff-console.md`](docs/plans/2026-10-22-phase-8-staff-console.md)
  — Phase 8, the staff console. Complete.
- [`docs/adr/pikol.dbml`](docs/adr/pikol.dbml) — the schema, plus Appendix A
  (DDL the diagram cannot express) and Appendix B (seed data).
- [`docs/adr/0001-booking-slot-uniqueness.md`](docs/adr/0001-booking-slot-uniqueness.md)
  — why a closure is a booking row. Read before touching occupancy.
- [`docs/adr/0002-maya-payment-provider.md`](docs/adr/0002-maya-payment-provider.md)
  — Maya replaces PayMongo: unsigned webhooks, fixed one-hour checkouts,
  decimal-peso amounts. Read before touching payments.
- [`docs/adr/0003-venue-user-groups.md`](docs/adr/0003-venue-user-groups.md)
  — venue-owned user groups replace roles; Customer is a group; permissions
  are per active venue. Read before touching auth, users or groups.
- [`docs/standard/`](docs/standard/) — repeatable procedures. Run
  [`cleanup-pass.md`](docs/standard/cleanup-pass.md) after a feature push to
  shrink, share and test the code without changing behaviour.
- [`docs/plans/2026-10-27-user-groups-and-console-redesign-design.md`](docs/plans/2026-10-27-user-groups-and-console-redesign-design.md)
  and [its plan](docs/plans/2026-10-27-user-groups-and-console-redesign.md)
  — the groups work, the staff-only web and the console redesign. Complete.

## Deployment (planned — Phase 9, not live)

All free tier, all Singapore (`ap-southeast-1`), the closest region to PH.

| Component | Where | Why |
|---|---|---|
| `pikol-frontend` | Vercel | Next.js native |
| `pikol-backend` | Render, Singapore | long-lived process holds one pool |
| Postgres | Supabase, Singapore | as in asima |

**Rules you must not get wrong:**

- **Use Supabase's session pooler on port 5432, never the transaction pooler
  on 6543.** asyncpg uses prepared statements; 6543 does not support them, and
  it fails intermittently *only* under production load.
- **`MAYA_PUBLIC_KEY` and `MAYA_SECRET_KEY` are server-only.** Never
  `NEXT_PUBLIC_*` — that is compiled into the browser bundle. "Public" is
  Maya's name for the checkout key, not permission to ship it. The frontend
  holds only `NEXT_PUBLIC_API_BASE_URL`.
- **Maya webhooks are unsigned.** Never trust the body: re-fetch the payment
  from Maya and act on that. See
  [`docs/adr/0002-maya-payment-provider.md`](docs/adr/0002-maya-payment-provider.md).
- **Production credentials are never committed.** They live in a gitignored
  `.env.supabase.prod` — named `.prod`, not `.local`, because on a file
  holding production credentials "`.local`" reads as *the safe one*, and that
  misreading ends with a destructive command pointed at the live database.
- **Migrations run from a laptop** against that file, never from CI.
- **CORS origins take no trailing slash.** `https://pikol.vercel.app/` does
  not match `https://pikol.vercel.app`. The backend rejects a trailing slash
  at boot rather than failing silently in production.
- **One Render service only.** A leftover duplicate makes Render's router
  flap: ~50% of requests 404 with `x-render-routing: no-server`.
- **The keep-alive ping must target `/health/ready`, not `/health`.** Supabase
  pauses on *database* inactivity, so a probe that returns a dict literal
  issues no query and does not keep it alive. It is load-bearing here, not
  hygiene: it also keeps Render awake, and a cold start lands squarely on the
  payment confirmation path.
- **Maya live mode needs a registered PH business** (DTI/SEC + BIR). That
  is the long pole and no code shortens it — start it early.

Rollout follows the provider port: **mock → Maya sandbox → live**.

## Working style

- Don't add features beyond what the task requires. Phases are deliberately
  scoped; flag scope drift instead of silently expanding it.
- Prefer editing existing files over creating new ones.
- For any non-trivial backend change, the layered file set (domain →
  persistence → service → controller) must move together. A half-landed slice
  is worse than no change.
- When a plan's own code fails a lint or type gate, fix **both** the code and
  the plan. The plan is copied by later phases; a defect left in it is a
  defect re-introduced four more times.

## Git workflow

- **Commit straight to `main` and push.** Do NOT create feature branches or
  open PRs unless explicitly asked in that request. (This overrides the usual
  "branch first if on the default branch" default.)
- Still **verify before pushing**: run the repo's checks and push only when
  they pass. Direct-to-main skips the branch-and-PR ceremony, not the gates.
- Each repo has its own `origin` and its own `main`; push to the one whose
  files changed. Keep commits scoped to one concern even when several land in
  the same session.
