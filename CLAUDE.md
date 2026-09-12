# pikol

Pikol is a **pickleball court booking** system: one owner, many venues, many
courts, clock-aligned hourly slots, online payment via PayMongo. Registered
players book and pay online; staff book walk-ins and collect cash at the
counter. Both paths write the same tables and pass the same constraint.

This file is the **system-level** brief — it covers the whole product across
frontend and backend. Backend-specific rules live in
`pikol-backend/CLAUDE.md` and load automatically when working under that
directory.

## Repo layout

```
pikol-parent/        # this repo — docs, plans, ADRs, the ERD, this brief
├── pikol-backend/   # FastAPI + SQLAlchemy 2.0 async + Postgres. See its own CLAUDE.md.
└── pikol-frontend/  # Next.js 15 App Router. Created in Phase 6; does not exist yet.
```

The sub-apps are **separate git repositories**, not submodules, sitting as
siblings under this parent — the same arrangement as `asima-parent`. Each has
its own `origin` and its own `main`.

| Repo | GitHub | State |
|---|---|---|
| `pikol-parent` | `Nacar11/pikol-parent` (public) | docs only, no code |
| `pikol-backend` | `Nacar11/pikol-backend` (public) | Phase 1 complete, CI green |
| `pikol-frontend` | not created | Phase 6 |

Keep them separated: no frontend code in the backend tree, no backend code in
the frontend tree.

## Current state

**Phase 1 (backend foundation) is complete** — the app boots, `/health` and
`/health/ready` respond, Alembic migrates, `parameters` is seeded and typed,
and CI is green on `main`. Phase 2 (identity and auth) is next.

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
`VENUE_MANAGER` at Venue A reading Venue B's bookings.

```python
class VenueScope:          # resolved once, from the token, by a dependency
    all_venues: bool       # SUPER_ADMIN
    venue_id: int | None   # VENUE_MANAGER / VENUE_STAFF
```

Every repository method touching venue-owned data takes `scope: VenueScope` as
a **required argument with no default**. Omitting it must be a `TypeError` at
call time, not a silent leak in production — the failure mode to design
against is *forgetting*, and a missing filter looks entirely normal in review.

For `/{id}` routes the check is ownership-after-fetch returning **404, not
403** — a 403 confirms existence, which tells a manager at Venue A exactly how
many courts Venue B has.

### Config is not parameters

- **`config/`** — environment: secrets, connection strings. Read once at boot.
  Changing it requires a restart.
- **`parameters/`** — business rules: prices, windows, caps. Stored in
  Postgres, tunable without a deploy.

Do not conflate them.

## Roles and permissions

Permission codes are `RESOURCE:Action`. The frontend gates UI from
`GET /api/v1/users/me/permissions` (a flat string array) — never by parsing
roles client-side. `SUPER_ADMIN` is an unconditional bypass axis, orthogonal
to permissions (asima ADR 0001).

| | SUPER_ADMIN | VENUE_MANAGER | VENUE_STAFF | PLAYER |
|---|---|---|---|---|
| Scope | all venues | own venue | own venue | self |
| `VENUE:Manage` | ✅ | — | — | — |
| `COURT:Manage` | ✅ | ✅ | — | — |
| `PRICE:Manage` | ✅ | ✅ | — | — |
| `CLOSURE:Manage` | ✅ | ✅ | ✅ | — |
| `BOOKING:ReadAny` | ✅ | ✅ | ✅ | — |
| `BOOKING:CreateWalkIn` | ✅ | ✅ | ✅ | — |
| `BOOKING:MarkPaid` | ✅ | ✅ | ✅ | — |
| `BOOKING:SetOutcome` | ✅ | ✅ | ✅ | — |
| `BOOKING:CancelAny` | ✅ | ✅ | ✅ | — |
| `USER:Manage` | ✅ | ✅ (staff only) | — | — |
| `PARAMETER:Manage` | ✅ | — | — | — |

A player is `role = PLAYER` in the same `users` table, not a separate identity
system. `parameters` moves money (fee percentage, default price), so it stays
admin-only; a venue manager adjusts pricing through `price_rules` on their own
courts.

**When a manager creates staff, `role_id` and `venue_id` are fixed by the
server, never read from the body.** Taking them from a body the manager
controls is how `{"role_id": 1}` makes someone a `SUPER_ADMIN`.

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
- **Admin vs self-service is enforced by schema, never by `if (role)`.**
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
- [`docs/adr/pikol.dbml`](docs/adr/pikol.dbml) — the schema, plus Appendix A
  (DDL the diagram cannot express) and Appendix B (seed data).
- [`docs/adr/0001-booking-slot-uniqueness.md`](docs/adr/0001-booking-slot-uniqueness.md)
  — why a closure is a booking row. Read before touching occupancy.

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
- **`PAYMONGO_SECRET_KEY` and `PAYMONGO_WEBHOOK_SECRET` are server-only.**
  Never `NEXT_PUBLIC_*` — that is compiled into the browser bundle. The
  frontend holds only `NEXT_PUBLIC_API_BASE_URL`.
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
- **PayMongo live mode needs a registered PH business** (DTI/SEC + BIR). That
  is the long pole and no code shortens it — start it early.

Rollout follows the provider port: **mock → PayMongo test → live**.

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
