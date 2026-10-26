# Implementation Plan: User Groups and Staff Console Redesign

> Design: [`2026-10-27-user-groups-and-console-redesign-design.md`](2026-10-27-user-groups-and-console-redesign-design.md).
> Read it first. This plan does not restate its rules. It orders the work
> and says how each step is proven.
>
> **For agentic workers:** execute task by task with
> superpowers:executing-plans or superpowers:subagent-driven-development.
> Each task ends with its repo's full gate green and one commit on `main`.

## Overview

The work has three parts:

1. **Fixed roles become venue-owned user groups.** A built-in Customer
   group sits beside them, and permissions are evaluated against the
   active venue (`X-Venue-ID`).
2. **The web becomes staff-only.**
3. **The staff console moves into an asima-style dark sidebar shell.**
   The palette is black and white, the Schedule page takes the reference
   layout, and two group pages are added: *Masters → User groups* and
   *Setup → Venue staff*.

**Roadmap position:** this is scope **inserted before Phase 9
(deployment)**. It changes the identity model that Phase 9 would ship, so
it lands first. Task 1 records the insertion in the roadmap.

## Architecture decisions

These are taken from the design and repeated here because the task order
depends on them.

1. **Expand, switch, contract.**
   - Task 2 adds the group tables beside `roles`/`users.role_id`/
     `users.venue_id`.
   - Tasks 3–4 move every reader and writer onto the new tables.
   - Task 5 drops the old columns.

   Every task therefore leaves `main` green. A single big-bang migration
   would leave the backend unrunnable across several commits.
2. **`VenueScope` keeps its single-venue shape.**
   - The active venue is resolved once, by `get_venue_access`, from
     `X-Venue-ID`.
   - Courts, pricing, bookings and closures repositories are untouched.
   - Only the users repository changes, because "users at venue V" now
     joins memberships.
3. **The API has one group resource with two UI views.**
   - The API is `/admin/user-groups`.
   - *Masters → User groups* (super admin) and *Venue staff* (master) are
     two frontend views of the same API.
4. **The test helper keeps its signature.**
   `make_user(role=..., venue_id=...)` maps:
   - `VENUE_MANAGER` → master group membership;
   - `VENUE_STAFF` → Front desk membership;
   - `PLAYER` → Customer membership.

   About 48 existing test call sites keep working through the switch. Only
   the tests whose *meaning* changes are rewritten.
5. **Frontend slices shared with staff are kept, not deleted.**
   - `admin-bookings` and `setup` import `catalog`, `availability` and
     `bookings`. Task 12 deletes the player-only routes and components,
     and keeps the shared ones in place.
   - Moving the shared code into new slices is not worth the churn.

## Dependency graph

```
T1 docs (dbml, ADR 0003, brief)
 └─ T2 expand: group tables + backfill + models
     └─ T3 access: X-Venue-ID, per-venue permissions, customer axis, /me/*
         ├─ T4 invitations + user admin on groups
         │   └─ T5 contract: drop roles / role_id / venue_id
         │       ├─ T6 group CRUD API
         │       │   └─ T7 group permissions API
         │       │       └─ T8 group members API
         │       └─ T9 occupancy read
         │
         └─ (frontend, after T5 is pushed)
             T10 palette + venue header + /me/venues plumbing
              ├─ T11 sidebar shell
              │   └─ T12 web staff-only
              │       ├─ T13 schedule header + 2-week strip   (needs T9)
              │       │   └─ T14 court grid + legend
              │       │       └─ T15 day-at-a-glance panel
              │       ├─ T16 groups slice + Venue staff page   (needs T6–T8)
              │       │   └─ T17 Masters → User groups page
              │       └─ T18 Staff accounts: group picker      (needs T4)
              └─ T19 brief + CLAUDE.md sync, final walkthrough
```

- T13–T15, T16–T17 and T18 are independent of each other once T12 lands,
  so they can be done in parallel.
- T6–T9 are sequential, because they share the group service.

---

## Phase A — Record the decision (pikol-parent)

### Task 1: Schema, ADR and brief first

**Description:** Change the source of truth before any code, as the brief
requires.

- **dbml**
  - Add the new tables: `user_groups`, `user_group_permissions` and
    `user_group_members`, plus the `group_kind` enum.
  - Mark the old ones for removal: `roles`, `role_permissions`,
    `users.role_id` and `users.venue_id`.
- **dbml Appendix A** gets the CHECK and the three partial unique indexes
  as DDL.
- **dbml Appendix B** gets the seed data:
  - the Customer group;
  - the per-venue Venue master and Front desk groups, with their
    permission codes.
- **ADR 0003** records three decisions:
  - groups replace roles;
  - Customer is a group;
  - permissions apply per venue, through `X-Venue-ID`.
- **Roadmap:** insert this work before Phase 9.

**Acceptance criteria:**
- [ ] `pikol.dbml` renders in dbdiagram.io with the three new tables and
      no `roles` table.
- [ ] Appendix A carries every constraint the diagram cannot express:
  - `CHECK ((kind='CUSTOMER') = (venue_id IS NULL))`;
  - one CUSTOMER group;
  - one MASTER per venue;
  - group names unique per venue on `LOWER(name)`.
- [ ] `docs/adr/0003-venue-user-groups.md` states the decision, what it
      replaces, and the rejected "summed permissions" alternative.

**Verification:** manual read. Paste the dbml into dbdiagram.io.

**Dependencies:** None.

**Files:**
- `docs/adr/pikol.dbml`
- `docs/adr/0003-venue-user-groups.md`
- `docs/plans/2026-09-09-implementation-roadmap.md`

**Scope:** S

---

## Phase B — Backend (pikol-backend)

The gate for every task is
`uv run ruff check . && uv run mypy src && uv run pytest`.

### Task 2: Expand — group tables, backfill, models

**Description:** Add the new tables and backfill existing data. The old
`roles` tables and columns stay in place, and nothing reads the new
tables yet.

- **Migration `0006_user_groups`**
  - Creates the `group_kind` enum and the three tables.
  - Writes the Appendix A DDL verbatim.
  - Inserts the Customer group.
  - For each venue, inserts Venue master and Front desk with their seeded
    codes.
  - Moves every user into a group:
    - `PLAYER` → Customer;
    - `VENUE_MANAGER` → master;
    - `VENUE_STAFF` → Front desk.
- **Models and seed constants**
  - `src/users/domain/groups.py` defines `GroupKind`, `OWNER_ONLY`,
    `MASTER_DEFAULT_CODES` and `FRONT_DESK_CODES`. `roles.py` keeps
    `PERMISSIONS`.
  - The migration seeds from these constants, as 0002 does from
    `ROLE_PERMISSIONS`.
- **`make_user` also writes the matching membership.**

**Acceptance criteria:**
- [ ] `alembic upgrade head`, then `downgrade -1`, then `upgrade head`
      round-trips on a database seeded with one of each role.
- [ ] The schema tests prove each Appendix A rule by an `IntegrityError`:
  - a second Customer group;
  - a second master at one venue;
  - a CUSTOMER group with a venue;
  - a STAFF group without a venue;
  - a duplicate name differing only in case.
- [ ] The backfill test asserts each seeded role lands in the right group.

**Verification:** the full backend gate.

**Dependencies:** T1.

**Files:**
- `src/database/migrations/versions/0006_user_groups.py`
- `src/users/domain/groups.py`
- `src/users/persistence/models.py`
- `tests/auth/helpers.py`
- `tests/users/test_group_schema.py`

**Scope:** M

### Task 3: Access — active venue, per-venue permissions, customer axis

**Description:** Switch authorisation onto groups. This is the
highest-risk task. It changes every guarded route, so it comes early.

- **`src/auth/access.py`**
  - Adds `get_venue_access`, which builds `VenueAccess(scope,
    permissions, is_master)` and resolves the header using the design's
    table.
- **`src/auth/dependencies.py`**
  - `require_permission` reads `VenueAccess.permissions`.
  - `get_venue_scope` wraps the access object.
  - `require_player` becomes `require_customer`, which checks Customer
    membership.
- **`/users/me` endpoints**
  - `/users/me/permissions` honours the header, or returns the union
    without it.
  - `/users/me/venues` is new.
  - `/users/me` gains `is_customer`.
- **Writers:**
  - registration joins Customer;
  - venue creation seeds Venue master and Front desk in the same
    transaction.

**Acceptance criteria:**
- [ ] Ana is master at A and Front desk at B.
  - With `X-Venue-ID: B` she gets 403 on `PUT` price rules at B.
  - With `X-Venue-ID: A` the same request succeeds.
  - With no header she gets 400 `VENUE_REQUIRED`.
  - A header naming C, where she has no membership, gets 403.
- [ ] Out-of-scope reads by `/{id}` still return **404**. The existing
      `test_venue_scope.py` cases pass unchanged in meaning.
- [ ] Kim is a customer and Front desk at A.
  - She can hold and pay as herself.
  - She can take a walk-in at A.
  - Once removed from Customer, she gets 403 on `POST /bookings`.

**Verification:**
- The full gate.
- New `tests/auth/test_venue_access.py` and
  `tests/bookings/test_customer_axis.py`.

**Dependencies:** T2.

**Files:**
- `src/auth/access.py` (new)
- `src/auth/dependencies.py`
- `src/auth/scope.py`
- `src/users/controllers/me.py`
- `src/users/persistence/repository.py`
- `src/auth/service.py`
- `src/venues/service.py`
- `src/bookings/controllers/player.py`

**Scope:** L. It stays one task because the guard switch cannot land
half-way. If it runs long, split off the `/me/*` endpoints as T3b.

### Task 4: Invitations and user admin on groups

**Description:** Move user administration onto groups.

- **Invitations**
  - `POST /admin/venue-staff` and `POST /admin/users` take `group_id`.
  - The venue route's group must be a STAFF group at the active venue.
  - An existing email gets 409 `EMAIL_EXISTS`.
- **Target protection.** `_modifiable` follows the design's
  management-rules rows:
  - a master of V can be changed only by the super admin;
  - a non-super-admin may deactivate only someone whose memberships are
    all at V.
- **User listing and output**
  - `/admin/users` lists users by membership at the active venue.
  - The super admin gets `?q=`.
  - `UserOut` drops `role` and `venue_id`, and gains
    `groups: [{id, name, kind}]` filtered to the caller's view.
- **CLI:** `create-super-admin` stops writing `role_id`.

**Acceptance criteria:**
- [ ] A master's invite with a master `group_id` gets 422 or 403. An
      invite whose body carries `venue_id` gets 422.
- [ ] A Front desk lead holding `USER:Manage` gets 403 trying to
      deactivate a master.
- [ ] Deactivating Ana, who is also staff at B, from A's account gets
      403. The super admin can do it.
- [ ] An A-side read of Ana's `UserOut.groups` lists only A's groups.

**Verification:** the full gate, plus a rewritten
`tests/users/test_admin_users.py`.

**Dependencies:** T3.

**Files:**
- `src/users/service.py`
- `src/users/dto.py`
- `src/users/controllers/admin.py`
- `src/users/persistence/repository.py`
- `src/users/cli.py`
- `tests/users/test_admin_users.py`

**Scope:** M

### Task 5: Contract — drop roles

**Description:** Remove the old role model now that nothing reads it.

- **Migration `0007_drop_roles`** drops `role_permissions`, `roles`,
  `users.role_id` and `users.venue_id`.
- **Its downgrade restores them** from memberships, preferring master,
  then staff, then customer.
- `RoleModel`, `role_permissions`, `ROLE_PERMISSIONS` and `RoleCode`
  usages are deleted.
- `make_user` keeps its `role=` parameter as a test-only enum.

**Acceptance criteria:**
- [ ] `grep -rn "role_id\|RoleModel\|ROLE_PERMISSIONS" src` is empty. The
      0002 migration file is the only exception.
- [ ] The full chain round-trips: upgrade to head, `downgrade 0005`,
      upgrade to head.

**Verification:** the full gate.

**Dependencies:** T4.

**Files:**
- `src/database/migrations/versions/0007_drop_roles.py`
- `src/users/persistence/models.py`
- `src/users/domain/roles.py`
- `tests/auth/helpers.py`
- `tests/users/test_identity_schema.py`

**Scope:** M

### Checkpoint: identity switched

- [ ] The backend gate is green. `uv run uvicorn` boots, and the existing
      frontend still signs in and shows the schedule, apart from expected
      `UserOut` field changes.
- [ ] **Stop for human review** before the new group APIs.

### Task 6: Group CRUD API

**Description:** Build the group resource itself, as a full layered
slice: domain → persistence → service → controller.

- **Endpoints:**
  - `GET /admin/user-groups` (the active venue's groups; the super admin
    may pass `?venue_id=` and `?kind=`);
  - `GET /admin/user-groups/{id}`;
  - `POST /admin/user-groups` (`{name}`; the super admin also sends
    `venue_id`);
  - `PATCH /admin/user-groups/{id}` (`{name}`);
  - `DELETE /admin/user-groups/{id}`.
- **Guard:** every route requires `USER:Manage`.
- **Visibility:**
  - Another venue's group returns 404.
  - The Customer group returns 404 to everyone but the super admin.
- **Deletion:**
  - A group with members gets 409 `GROUP_NOT_EMPTY`.
  - Master and Customer groups cannot be deleted.

**Acceptance criteria:**
- [ ] One test per CRUD row of the design's table, each failing if its
      rule is removed.
- [ ] A body carrying `venue_id` or `kind` from a non-super-admin gets
      422.
- [ ] A duplicate name at the same venue gets 409 `GROUP_NAME_TAKEN`.

**Verification:** the gate, plus `tests/users/test_user_groups.py`.

**Dependencies:** T5.

**Files:**
- `src/users/controllers/groups.py`
- `src/users/group_service.py`
- `src/users/persistence/group_repository.py`
- `src/users/dto.py`
- `src/main.py`

**Scope:** M

### Task 7: Group permissions API

**Description:**
`PUT /admin/user-groups/{id}/permissions {codes}` replaces the group's
permission set.

The rules:

- Owner-only codes are rejected for every caller.
- A non-super-admin's diff may touch only codes they hold at V. Codes
  they don't hold are kept as they are.
- The master group is read-only for anyone but the super admin.
- The Customer group accepts no codes.
- Removing `USER:Manage` from a group the caller is in gets 409
  `SELF_LOCKOUT`.
- `GET /admin/permissions` lists every code with its resource, action and
  description, for the checkbox UI.

**Acceptance criteria:**
- [ ] Rejecting `PARAMETER:Manage` is pinned for the super admin as well
      as everyone else.
- [ ] Preservation test: the group has `COURT:Manage`. A caller without
      it saves `{BOOKING:ReadAny}`, and the group still has
      `COURT:Manage`.
- [ ] Self-lockout test passes.

**Verification:** the gate.

**Dependencies:** T6.

**Files:**
- `src/users/group_service.py`
- `src/users/controllers/groups.py`
- `src/users/dto.py`
- `tests/users/test_group_permissions.py`

**Scope:** S

### Task 8: Group members API

**Description:** Manage who is in each group.

- **Endpoints:**
  - `GET /admin/user-groups/{id}/members` (paginated);
  - `POST /admin/user-groups/{id}/members {email}`;
  - `DELETE /admin/user-groups/{id}/members/{user_id}`.
- **Who can be added:**
  - Non-super-admins find people by exact, case-insensitive email.
  - The person must be active or have a pending invitation. A
    deactivated person gets 422 `USER_INACTIVE`.
- **Restricted groups:** master and Customer membership is super admin
  only.
- **Removal:**
  - A non-master can never remove a master of V.
  - Removing yourself from your last `USER:Manage` group at V gets 409
    `SELF_LOCKOUT`.

**Acceptance criteria:**
- [ ] The not-found email response gives the same 404 whether the account
      is missing or deleted. It never reveals more than "no active account
      with that email".
- [ ] Adding a customer to Front desk at A leaves them in Customer. A
      master at A reading that member sees no B memberships.
- [ ] Every member row of the design table has a test.

**Verification:** the gate.

**Dependencies:** T7.

**Files:**
- `src/users/group_service.py`
- `src/users/controllers/groups.py`
- `src/users/persistence/group_repository.py`
- `tests/users/test_group_members.py`

**Scope:** M

### Task 9: Occupancy read for the two-week strip

**Description:** `GET /admin/bookings/occupancy?from=&days=14` (plus
`venue_id` for the super admin) returns `[{date, booked_hours,
bookable_hours}]`. It requires `BOOKING:ReadAny`.

- Day boundaries are Manila days.
- PENDING counts only when `expires_at > now()`.
- MAINTENANCE hours are subtracted from bookable and not counted as
  booked.
- Bookable is active courts × 24.

**Acceptance criteria:**
- [ ] A 12am–1am booking counts toward the Manila day it starts on.
- [ ] A lapsed PENDING hold is not counted.
- [ ] A closure reduces `bookable_hours`.
- [ ] `days` is capped at 31, with 422 above that.

**Verification:** the gate, plus `tests/bookings/test_occupancy.py`.

**Dependencies:** T5.

**Files:**
- `src/bookings/controllers/admin.py`
- `src/bookings/persistence/repository.py`
- `src/bookings/dto.py`
- `tests/bookings/test_occupancy.py`

**Scope:** S

### Checkpoint: backend complete

- [ ] The backend gate is green. Push `pikol-backend`.
- [ ] Use `/docs` to walk through a group session by hand. Create a group
      at A, tick codes, add a customer by email, then sign in as that
      customer with `X-Venue-ID: A` and confirm the new permissions.

---

## Phase C — Frontend (pikol-frontend)

The gate for every task is
`npm run lint:ci && npm run format:check && npm run typecheck && npm test && npm run build`.

### Task 10: Palette, venue header, `/me/venues`

**Description:**

- **Palette.** `globals.css` gets the neutral tokens from the design.
  `destructive` stays red. `accent` and `success` collapse to neutrals.
- **Venue header.** `lib/api-client` gets `setVenueProvider(() => number
  | null)` and sends `X-Venue-ID` whenever the provider returns an ID.
  `lib/` must stay free of feature imports.
- **Auth.** The `auth` schemas follow the new `UserOut`: `is_customer`,
  no `role` or `venue_id`.
- **Venue choice.** `useStaffVenue` reads `/users/me/venues`. The `?venue=`
  value wins, then the remembered choice in `localStorage` (try/catch),
  then the first venue. It registers the provider, and a venue change
  invalidates the staff queries and the permissions query, whose key now
  includes `venueId`.

**Acceptance criteria:**
- [ ] A unit test shows the client sends the header when the provider
      returns 7 and omits it when it returns null.
- [ ] A unit test shows switching venue refetches permissions under a new
      key.
- [ ] No hue is left in tokens except `destructive`. A grep finds no
      `#`/`hsl(` in components.

**Verification:** the gate.

**Dependencies:** backend checkpoint.

**Files:**
- `src/app/globals.css`
- `src/lib/api-client.ts`
- `src/features/auth/schemas.ts`
- `src/features/auth/use-permissions.ts`
- `src/features/staff/use-staff-venue.ts`
- `tests/unit/lib/*`

**Scope:** M

### Task 11: Sidebar shell

**Description:** `components/layout/staff-shell.tsx` follows asima's
`app-shell` shape, with the sidebar contents from the design.

- **Layout behaviour**
  - On mobile the sidebar is a drawer. It has a backdrop, closes on
    Escape and closes on route change.
  - The shell has a skip link.
- **Sidebar contents**
  - The brand.
  - The venue switcher.
  - Operations, Setup and Masters sections. `nav-links.ts` gains
    `section` and `icon` fields.
  - The To resolve badge, fed by the resolution queue total.
  - The user block, with sign-out at the bottom.
- **`staff/layout.tsx`** wraps the pages in `StaffGate` and `StaffShell`.
- **Removed:** `StaffNav`, and `VenuePicker`'s placement in page headers.

**Acceptance criteria:**
- [ ] An item appears only when its code is held at the active venue. A
      section with no items has no header.
- [ ] The active item has `aria-current="page"`. Drawer touch targets are
      at least 44 px.
- [ ] At 375 px there is no horizontal page scroll and the drawer opens
      and closes. At `md` and up the sidebar is persistent.

**Verification:**
- The gate.
- `tests/unit/staff/staff-shell.test.tsx` covers the filtering and the
  sections.
- Manual check in the browser at 375 px and 1440 px.

**Dependencies:** T10.

**Files:**
- `src/components/layout/staff-shell.tsx`
- `src/features/staff/nav-links.ts`
- `src/features/staff/components/venue-switcher.tsx`
- `src/app/(app)/staff/layout.tsx`
- `src/features/staff/index.ts`

**Scope:** M

### Task 12: Web staff-only

**Description:** Make the web app staff-only.

- **Deleted routes:**
  - `/venues`, `/venues/[venueId]`;
  - `/bookings`, `/bookings/[bookingId]`;
  - `/checkout/return`;
  - `/register`.
- **Deleted components:** the player-only ones, which nothing in
  `admin-bookings` or `setup` imports:
  - `book-venue-day`, `hold-bar`, `hold-countdown`, `pay-button`;
  - `checkout-return`, `my-bookings`, `cancel-booking`;
  - `venue-list`, `register-form`;
  - `AppShell`, `AppNav`.
- **New page:** `/customer-app` says "Pikol for players is coming to
  mobile", with a sign-out button.
- **Landing:**
  - `landingPath` sends staff to `/staff` and everyone else to
    `/customer-app`.
  - `canBookOnline` is deleted.
- **Login:** the "Create an account" link is removed.

**Acceptance criteria:**
- [ ] A customer-only account signing in lands on `/customer-app`. A
      deep link to `/staff/schedule` sends them back there.
- [ ] `npm run build` has no orphaned imports. A dead-export check of the
      removed slices' `index.ts` passes.
- [ ] Unit tests for the removed components are deleted. Tests for the
      kept shared ones still pass.

**Verification:** the gate, plus a manual sign-in as a customer and as
staff.

**Dependencies:** T11.

**Files:**
- `src/app/(app)/{venues,bookings,checkout}/**`
- `src/app/(auth)/register/**`
- `src/app/(app)/customer-app/page.tsx`
- `src/features/auth/landing.ts`
- `src/features/{bookings,availability,catalog}/index.ts` and the
  player-only component files
- `src/app/(app)/layout.tsx`

**Scope:** M. It has many file deletions but little new code.

### Checkpoint: shell and scope

- [ ] The gate is green. Every existing staff page renders inside the
      sidebar shell in black and white.
- [ ] **Stop for human review** of the look before the Schedule redesign.

### Task 13: Schedule header and two-week strip

**Description:** Build the top of the Schedule page.

- **Header:** a breadcrumb, then the Manila long date with "YYYY · Week
  N". On the right: Today, ‹ › for the day, and the New booking button.
- **Strip:** `features/admin-bookings/components/week-strip.tsx` shows 14
  days from the selected week's Monday, with ‹ › to page by week.
- **Per-day utilisation bars** come from `useOccupancy` (the T9 read).
  The selected day is inverted.
- **Keyboard:** ← and → change the day unless focus is in a field.

**Acceptance criteria:**
- [ ] The ISO week number and the Monday start are computed in Manila
      time. A unit test covers a Sunday-night Manila date that is still
      Sunday in UTC−.
- [ ] The bar shows `booked/bookable`, with an `aria-label` such as "Mon 7
      Sep, 9 of 48 hours booked".
- [ ] Arrow keys are ignored while focus is in an input or textarea.

**Verification:** the gate, plus `tests/unit/admin-bookings/week-strip.test.tsx`.

**Dependencies:** T12, T9.

**Files:**
- `src/features/admin-bookings/components/{schedule,schedule-header,week-strip}.tsx`
- `src/features/admin-bookings/{api,keys,schemas}.ts`
- `src/features/admin-bookings/hooks/use-occupancy.ts`
- `src/lib/manila.ts`

**Scope:** M

### Task 14: Court grid and legend

**Description:** Build the court grid and its legend.

- **Grid:** `staff-court-grid.tsx` has time running down the side and one
  column per court.
- **Court headers:** a large two-digit number, the name, "₱N / hr", a
  booked bar and "N of 24 h booked".
- **Bookings:** each one is a single block spanning its hours. It shows
  the time range, the player or "Walk-in", and an uppercase status label.
  It links to the booking's detail page.
- **Legend:** shows the design's seven states and the court-hours total.
- **Reuse:** the existing selection reducer (`tapCell`, `reconcile`) and
  `WalkInBar` are reused. Tapping an open cell selects it.
- **New booking** selects the next open hour on the first court.
- **Scrolling:** the grid scrolls inside its frame and starts at the
  current Manila hour on today.

**Acceptance criteria:**
- [ ] Every status shows its text label, so the grid reads correctly in
      greyscale and with CSS colours disabled.
- [ ] A 2-hour booking renders as one block, not two cells.
- [ ] IN_PROGRESS is derived from `starts_at` and `now`, as in the brief.
      A unit test covers the boundary minute.

**Verification:** the gate, plus
`tests/unit/admin-bookings/staff-court-grid.test.tsx`.

**Dependencies:** T13.

**Files:**
- `src/features/admin-bookings/components/{staff-court-grid,schedule-legend,booking-block}.tsx`
- `src/features/admin-bookings/components/schedule.tsx`

**Scope:** M

### Task 15: Day-at-a-glance panel

**Description:** `day-glance.tsx` holds the four KPI tiles and the day's
booking list.

- **The tiles:**
  - Booked hours;
  - Utilisation;
  - Revenue, from frozen `total_centavos`, excluding cancelled and
    expired;
  - Awaiting payment, live UNPAID only, which respects the PENDING trap.
- **The booking list** shows the time, player, "Court N · H h" and a
  status chip.
- **The hint footer** explains the controls.
- **Placement:** a 320 px right column at `xl` and up. Below `xl` it
  stacks under the grid. It replaces the old `DayBookings` list.

**Acceptance criteria:**
- [ ] A unit test feeds fixtures and checks the KPI maths, including a
      lapsed PENDING hold being excluded from Awaiting payment.
- [ ] Money is shown in whole pesos from centavos via `lib/money`, with
      `tabular-nums`.

**Verification:** the gate, plus a manual check against the reference
screenshot at 1440 px and at 375 px.

**Dependencies:** T14.

**Files:**
- `src/features/admin-bookings/components/{day-glance,schedule}.tsx`
- `src/features/admin-bookings/components/day-bookings.tsx` (removed or
  folded in)
- `tests/unit/admin-bookings/day-glance.test.tsx`

**Scope:** S

### Checkpoint: Schedule

- [ ] The gate is green. Book a walk-in, mark it paid and check in on the
      new grid, and every state renders.
- [ ] **Stop for human review** against the reference.

### Task 16: Groups slice and Venue staff page

**Description:** Add a new `features/groups` slice and the Venue staff
page.

- **The slice** follows the house shape: `api.ts`, `schemas.ts`,
  `keys.ts`, `hooks/` and `components/`.
- **Shared components**, reused by T17:
  - `GroupList`;
  - `GroupDetail`;
  - `PermissionChecklist`, with plain-English labels and sections;
  - `MemberPanel`.
- **The page** is `/staff/venue-staff`, gated on `USER:Manage`:
  - the master group is pinned at the top of the list, read-only;
  - New group opens an inline name field;
  - Rename and Delete are on the detail, with Delete confirmed and red;
  - the Permissions tab saves and discards, and guards unsaved changes;
  - the Members tab adds by email and removes with a confirmation.
- **Error codes** render inline: `SELF_LOCKOUT`, `GROUP_NOT_EMPTY`,
  `EMAIL_EXISTS`, `USER_INACTIVE`.

**Acceptance criteria:**
- [ ] The checklist matches the server rules. Owner-only codes and codes
      the caller lacks are disabled, with a reason. Codes kept on the
      group but not held by the caller show ticked and disabled.
- [ ] Saving is not optimistic. The UI shows the server's response, and
      a 409 leaves the form dirty with the message inline.
- [ ] Keyboard: the group list moves with ↑ ↓ and opens with Enter, and
      the tabs are a proper `tablist`.

**Verification:**
- The gate.
- Unit tests for the checklist rules and the unsaved-changes guard.
- A manual run against the T8 backend.

**Dependencies:** T12. The backend needs T6–T8.

**Files:**
- `src/features/groups/**`
- `src/app/(app)/staff/venue-staff/page.tsx`
- `src/features/staff/nav-links.ts`

**Scope:** L. If it runs long, split it in two: the slice and the list
first, then the detail tabs.

### Task 17: Masters → User groups page

**Description:** `/staff/groups` is gated on `VENUE:Manage`. It reuses the
`features/groups` components.

- **The list** shows the Customer group, then one "Venue master ·
  <venue>" row per venue, with a filter box.
- **The Customer group** has a Members tab only. It uses
  `/admin/users?q=` search and is paginated.
- **Master groups** have both the Permissions and Members tabs.

**Acceptance criteria:**
- [ ] A non-super-admin never sees the nav item, and the route renders
      "not authorised".
- [ ] Adding someone to a venue's master group makes the new master see
      that venue in their switcher after sign-in.

**Verification:** the gate, plus a manual check as the super admin.

**Dependencies:** T16.

**Files:**
- `src/app/(app)/staff/groups/page.tsx`
- `src/features/groups/components/*`
- `src/features/groups/{api,hooks}`

**Scope:** S

### Task 18: Staff accounts — group picker

**Description:** Change the Staff page into Staff accounts.

- The page is relabelled "Staff accounts".
- The invite form's role select becomes a group select:
  - it lists the active venue's STAFF groups;
  - the super admin sees all groups.
- The team list shows each person's groups at the active venue.
- `EMAIL_EXISTS` shows as a field error with a link to Venue staff.

**Acceptance criteria:**
- [ ] The invite payload carries `group_id` and never `venue_id` or a
      role.
- [ ] A unit test covers the `EMAIL_EXISTS` rendering.

**Verification:** the gate.

**Dependencies:** T12.

**Files:**
- `src/features/team/{schemas,api}.ts`
- `src/features/team/components/{invite-form,team-list}.tsx`
- `src/features/staff/nav-links.ts`

**Scope:** S

### Task 19: Docs sync and final walkthrough

**Description:**

- **System brief (`pikol-parent/CLAUDE.md`):** update the roles table,
  the line "a player is role = PLAYER", the venue-scope section
  (`X-Venue-ID`) and the frontend state.
- **`pikol-frontend/CLAUDE.md`:** update the slices and routes, and note
  that the app is staff-only.
- **`pikol-backend/CLAUDE.md`:** update any role references.
- **Design doc:** mark its status as built.

**Acceptance criteria:**
- [ ] The parent brief does not mention `role_id`, `VENUE_MANAGER` or
      `PLAYER` as a stored role, except in a history note.
- [ ] Walkthrough:
  - as the super admin, make Ana master of A and B;
  - as Ana, at A, create "Weekend crew", tick its codes and add Kim (a
    customer) by email;
  - as Kim, see A only, with exactly those permissions;
  - confirm Kim is still a customer through `/users/me`.

**Verification:** both repos' gates are green, and the walkthrough passes.

**Dependencies:** T13–T18.

**Files:**
- `CLAUDE.md`
- `pikol-frontend/CLAUDE.md`
- `pikol-backend/CLAUDE.md`
- the design doc

**Scope:** S

### Checkpoint: complete

- [ ] Every design acceptance item is met, and each management-rules row
      has a failing-when-removed test.
- [ ] Both repos are pushed to `main`, and the parent docs are committed.

---

## Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| T3 breaks every guarded route at once | High | It comes early. `make_user` keeps its signature, and the existing scope tests must pass unchanged in meaning before T4 starts. |
| A forgotten `X-Venue-ID` makes a multi-venue user see 400s everywhere | Med | The provider is injected once in `api-client`, and a unit test proves the header. Single-venue staff fall back to their one venue server-side. |
| Escalation through the new APIs, which is the anti-goal | High | Every row of the management-rules table has a test that fails when its rule is removed (T6–T8). |
| A wrong backfill in 0006 loses someone's access | Med | The backfill test covers every role, and the round-trip test covers the downgrade. Nothing is deployed, so the only data at risk is local. |
| Deleting player code breaks staff imports | Med | The T12 build gate catches it. Only components with no staff importer are deleted, so the shared ones stay. |
| The occupancy maths drifts from availability | Low | The same Manila-bounds helper and PENDING filter are used, and tested on the 12am slot. |
| Customers cannot book until mobile ships | Accepted | The owner chose this. The backend customer routes stay tested. |

## Open questions

None are blocking. The builder's calls are listed at the end of the design
doc for the owner to check while reading.
