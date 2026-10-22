# Phase 8 — Frontend Staff Console Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Staff, managers and the owner run the venue from `/staff`:
- read today's schedule
- book walk-ins at the counter
- collect cash
- record outcomes
- cancel bookings
- settle paid-but-unhoused bookings
- close courts

Managers and the owner also:
- edit courts and prices
- invite staff

The owner additionally manages venues and the business parameters.

**Architecture:** New frontend slices on Phase 6–7's shape:
- `features/staff`: the console frame, the venue context and permission-driven navigation.
- `features/admin-bookings`: the schedule, walk-ins, the booking detail with its action routes, and the resolution queue.
- `features/setup`: venues, courts, price rules and closures.
- `features/team`: staff accounts.
- `features/parameters`.

The Phase 7 `AvailabilityGrid` and selection reducer are reused for walk-ins, not copied. Every action is a named server route (spec §7.3) whose response is written back, and none is optimistic. There is one backend task: a parameters admin API, which the spec's endpoint map lists (§7.4) but which no phase built.

**Tech Stack:** as Phase 7. No new dependencies.

**Spec:** design §5.1–5.2 (transitions and guards), §5.5 (resolution queue), §6.2–6.4 (venue scope, the permission map, the two invite routes), §7.3–7.4 (action routes, the endpoint map), §8 (the `staff/` routes), §8.1 (one grid for both surfaces). Backend admin routes as built in Phases 2–5.

## Global Constraints

Everything in Phase 7, plus:

- **The UI is gated on `GET /users/me/permissions`, never on role.** Every screen, nav link and action button names the permission code it needs. Use the table in `../CLAUDE.md` § Roles and permissions. `SUPER_ADMIN` receives every code from that endpoint.
- **The server is the authority on every guard.** Button visibility is a convenience. A 409 (`NOT_CANCELLABLE`, `NOT_STARTED`, `ALREADY_PAID`, `NOT_A_WALK_IN`, `IS_CLOSURE`, `NOT_FLAGGED`, …) or a 422 always renders the server's `detail`, and the booking is then refetched.
- **Transitions are action routes**: `/admin/bookings/{id}/{cancel,mark-paid,outcome,resolve}` and `/admin/closures/{id}/cancel`. There is never a status PATCH.
- **Staff cancellation, closure reopening and resolution require a typed reason** (spec §6.5). The submit button stays disabled until the reason is non-blank.
- **All times render and are entered in Asia/Manila.** A Manila day + hour becomes an instant only through `lib/manila.ts:manilaInstant`.
- **Money is entered in pesos and sent as integer centavos**, through `lib/money.ts:parsePesos` only. A float never touches a price.
- **Privilege fields are never sent by a manager.** The staff invite form has no role and no venue field. Only the `VENUE:Manage` holder's form (`POST /admin/users`) has them (spec §6.4).
- **Out-of-scope ids render "not found"**, matching the backend's 404 (spec §6.2).

## Decisions this plan makes

| Decision | Why |
|---|---|
| **Backend task: `GET /admin/parameters` and `PATCH /admin/parameters/{key}`** (`PARAMETER:Manage`) | Spec §7.4 lists `parameters` under Admin — setup, and the roadmap puts it in this phase, but Phases 1–5 built no route for it. A changed value is validated against the **whole** typed `Parameters` model before it is written, so `min_lead_minutes < hold_duration_minutes` is refused with the model's message. The process cache is invalidated on write. |
| Routes: `/staff` redirects to `/staff/schedule`. The other pages are `/staff/schedule?venue=&date=`, `/staff/bookings/[bookingId]`, `/staff/resolution`, `/staff/venues`, `/staff/courts?venue=`, `/staff/courts/[courtId]/pricing`, `/staff/closures?venue=`, `/staff/users` and `/staff/parameters` | Spec §8's `staff/` tree. `venue` and `date` live in the URL so a reload keeps context. |
| **Venue context.** A user whose `user.venue_id` is set works only at that venue, with no picker. A user with no `venue_id` (the all-venues owner) picks a venue with `?venue=`, defaulting to the first active catalog venue. Venue names come from the catalog `GET /venues`. | `GET /admin/venues` needs `VENUE:Manage`, which staff lack. `venue_id` on the user is data from `/users/me`, not a role. |
| **Court pickers** (schedule, closures) read the catalog `GET /venues/{id}/courts`. The courts screen alone reads `GET /admin/venues/{id}/courts`, which includes inactive courts. | Staff hold `CLOSURE:Manage` but not `COURT:Manage`. |
| **Walk-in selection reuses `tapCell`** with a new optional `selectable` predicate. Players get the default (`AVAILABLE` only). Staff use `AVAILABLE` or `PAST`. | For the grid, `PAST` means *free but too soon for a player*. That is exactly the counter's walk-in-now case, and the backend lets staff backdate up to `staff_backdate_days`. |
| The staff date strip runs from 7 days back to 13 days ahead. `DateStrip` gains an optional `start` offset (default 0). | 7 is the seeded `staff_backdate_days`. `TOO_FAR_BACK` renders its detail if the parameter is lowered. |
| **The schedule is the grid plus the day's list.** The list comes from `GET /admin/bookings?venue_id=&date=&limit=200`, showing type, player name (or "Walk-in"/"Closure"), the time range, `display_status`, payment and total. Each row links to the booking. | The grid never reveals who booked (§7.2). The list is where staff see names. One venue-day is at most 24 × courts rows, well under 200. |
| The schedule refetches every 30 s while visible | Holds lapse and players pay while the counter screen is left open. |
| **The booking detail's actions come from one pure function**, `staffActions(b, permissions, nowMs)`. It returns the subset of `markPaid`, `outcome`, `cancel`, `reopen` that are allowed. | The guards (spec §5.2) become one tested table rather than conditions scattered through JSX. The server re-checks every one. |
| Outcome is offered when `display_status = IN_PROGRESS`, or when `booking_status = CONFIRMED` and `starts_at ≤ now` | Spec §5.2: only once `starts_at <= now()`. After the last hour the server's display status is `CONFIRMED` again, so the start time decides. A skewed clock only shows or hides a button; `NOT_STARTED` covers the rest. |
| Mark-paid offers `CASH` and `MANUAL_GCASH`, never `ONLINE` | Spec §5.2 and the backend `MarkPaidRequest`: online money arrives only through the provider. |
| **Invites, by permission.** A `VENUE:Manage` holder gets role (`VENUE_MANAGER`/`VENUE_STAFF`) and venue fields and posts `/admin/users`. Anyone else with `USER:Manage` gets name/email/mobile only and posts `/admin/venue-staff`. | Spec §6.4. The manager's form cannot express a privilege, so it cannot be tampered into one. |
| An invited account is "pending" while `email_verified_at` is null. Pending rows offer **Resend** and **Revoke**; others offer **Deactivate**/**Reactivate** (`PATCH is_active`) | Accepting an invitation verifies the email (Phase 2). |
| Closures list `GET /admin/bookings?venue_id=&type=MAINTENANCE&booking_status=CONFIRMED`, keeping rows whose `ends_at` is in the future | There is no closures list route. Closures are booking rows (ADR 0001), and an ended closure is not actionable. |
| Venues and courts are edited inline (rename, sort order, activate/deactivate), with no delete | The backend never deletes venues or courts: bookings point at them. |
| Price rule day pickers show Sunday–Saturday as 0–6, plus "Any day" and "Any hour". Editing a rule changes only its price. A different day or hour means delete, then add. | Mirrors `PriceRuleCreate`/`PriceRuleUpdate`. The uniqueness constraint stays the arbiter, and its 409 detail renders. |
| The pricing screen previews the selected day's 24 resolved prices (`GET /admin/courts/{id}/prices?date=`) | The owner sees what a player will pay before saving the next rule. |

## Review Focus

1. **Walk-in for the current hour.** A staff tap on a `PAST` (free) cell this hour selects it and books it `CONFIRMED/UNPAID`. A player can never select a `PAST` cell. Tests in Task 2 (`tapCell`) and Task 3.
2. **Manager invite cannot carry privilege.** The `/admin/venue-staff` request body has exactly `{email, first_name, last_name, mobile?}`. Test in Task 9.
3. **Guard drift.** The booking moved on (another staff member marked it paid). Marking it paid again returns 409, the detail renders, and the booking refetches to the true state. Test in Task 4.
4. **Peso entry.** `"600"` → 60000, `"600.5"` → 60050, `"600.505"` and `"abc"` and `"-1"` are rejected, `"1,200"` → 120000. Test in Task 2.
5. **Closures in Manila.** A closure entered as 23 Oct, 10 PM to 24 Oct, 2 AM posts `2026-10-23T22:00:00+08:00` → `2026-10-24T02:00:00+08:00`. Test in Task 8.

---

### Task 1: Backend — the parameters admin API

**Repo:** `pikol-backend`.

**Files:**
- Modify `src/parameters/persistence/repository.py`: add `list_rows(session) -> list[ParameterModel]` ordered by key, and `get_row(session, key) -> ParameterModel | None`.
- Create `src/parameters/dto.py`:
  - `ParameterOut {key, value, value_type, description, updated_at, updated_by_user_id}`
  - `ParameterUpdate {value: str, 1..64}` with `extra="forbid"`
- Create `src/parameters/admin.py` (service): `update(session, caller, key, value) -> ParameterModel`.
- Create `src/parameters/controllers/admin.py`.
- Register the router in `src/main.py`.
- Test: `tests/parameters/test_admin_parameters.py`.

**Interfaces:**
- `GET /api/v1/admin/parameters` (`PARAMETER:Manage`) → `list[ParameterOut]`.
- `PATCH /api/v1/admin/parameters/{key}` (`PARAMETER:Manage`) `{value}` → `ParameterOut`. The service:
  1. Unknown key → 404 `NOT_FOUND`.
  2. Builds the candidate map from `load_all` with the new value, then runs `Parameters.model_validate(candidate)`. A `ValidationError` → 422 `INVALID_PARAMETER`, with the first error's message as `detail`.
  3. Writes `value`, `updated_at = now`, `updated_by_user_id = caller.id`, then commits.
  4. Calls `parameter_service.invalidate()`.

- [ ] **Step 1: Failing tests.**
  - `test_super_admin_lists_every_parameter`: the list contains `hold_duration_minutes`.
  - `test_manager_and_staff_get_403`.
  - `test_patch_updates_value_and_audit`: `updated_by_user_id` is the caller, and a following `parameter_service.get` sees the new value.
  - `test_patch_rejects_cross_field_rule`: `min_lead_minutes = 5` while the hold is 15 → 422 `INVALID_PARAMETER`, and the value is unchanged.
  - `test_patch_rejects_bad_type`: `"abc"` for an int → 422.
  - `test_unknown_key_is_404`.
  - `test_patch_body_forbids_extra_fields`.
- [ ] **Step 2:** `uv run pytest tests/parameters/test_admin_parameters.py -v`. Expected: FAIL (404 on the route).
- [ ] **Step 3:** Implement. Follow the layering in `pikol-backend/CLAUDE.md`. Use `require_permission("PARAMETER:Manage")`.
- [ ] **Step 4:** Run the full gate: `uv run ruff format . && uv run ruff check . && uv run mypy src && uv run pytest && uv run alembic check`. Expected: green.
- [ ] **Step 5:** Commit `Add the parameters admin API`.

### Task 2: The staff frame, venue context, and shared primitives

**Repo:** `pikol-frontend` from here on.

**Files:**
- Modify `src/lib/manila.ts`: add `manilaInstant(day: string, hour: number): string`, which returns `"2026-10-23T22:00:00+08:00"` and throws on a bad day or an hour outside 0–23.
- Modify `src/lib/money.ts`: add `parsePesos(input: string): number | null`. It trims, strips `,` and a leading `₱`, accepts up to 2 decimals, and returns centavos as an integer, computed from the string's digits rather than by float multiplication. It returns null for a negative number, more than 2 decimals, an empty string, or anything that isn't a number.
- Modify `src/features/availability/selection.ts`: give `tapCell` and `reconcile` an optional 5th/3rd argument `selectable: (s: GridStatus) => boolean`, defaulting to `s === 'AVAILABLE'`.
- Modify `src/features/availability/components/date-strip.tsx`: add a `start?: number` day offset, default 0.
- Create `src/features/staff/{index.ts, use-staff-venue.ts, components/{staff-nav,venue-picker,staff-gate}.tsx}`.
- Create `src/app/(app)/staff/layout.tsx`, and replace `src/app/(app)/staff/page.tsx` with a redirect to `/staff/schedule`.
- Modify `src/components/layout/app-nav.tsx` so that staff get the staff links.
- Tests:
  - `tests/unit/lib/manila.test.ts` and `tests/unit/lib/money.test.ts` (extend)
  - `tests/unit/availability/selection.test.ts` (extend)
  - `tests/unit/staff/staff-nav.test.tsx`
  - `tests/unit/staff/use-staff-venue.test.tsx`

**Interfaces:**
- `STAFF_LINKS: { href: string; label: string; permission: string }[]`, in this order:
  - Schedule `/staff/schedule` `BOOKING:ReadAny`
  - To resolve `/staff/resolution` `BOOKING:ReadAny`
  - Closures `/staff/closures` `CLOSURE:Manage`
  - Courts `/staff/courts` `COURT:Manage`
  - Staff `/staff/users` `USER:Manage`
  - Venues `/staff/venues` `VENUE:Manage`
  - Parameters `/staff/parameters` `PARAMETER:Manage`
- `<StaffNav />` renders the links whose permission is held, with `aria-current` on the active one. `AppNav` renders `StaffNav` when `canBookOnline` is false.
- `useStaffVenue(): { venueId: number | null; venues: Venue[]; canPick: boolean; setVenue(id): void; isPending: boolean; error: unknown }`:
  - `user.venue_id` set → that id, and `canPick` false.
  - Otherwise → `?venue=` if it names a catalog venue, else the first catalog venue, and `canPick` true.
  - `setVenue` does a `router.replace` that keeps the other query params.
- `<VenuePicker />` is a `<select>` shown only when `canPick`.
- `<StaffGate permission>` wraps `RequirePermission` from `@/features/auth`. `/staff/layout.tsx` gates on `BOOKING:ReadAny`.

- [ ] **Step 1: Failing tests.**
  - `manilaInstant('2026-10-23', 22)` returns `'2026-10-23T22:00:00+08:00'`.
  - `manilaInstant('2026-10-23', 24)` throws.
  - `parsePesos`: every case in Review Focus 4.
  - `tapCell` with the staff predicate selects a `PAST` cell, and with the default it ignores it. Extending a span across `PAST` + `AVAILABLE` works for staff.
  - `StaffNav`:
    - staff permissions (`CLOSURE:Manage` + booking ops) show Schedule, To resolve and Closures, and **not** Courts/Staff/Venues/Parameters
    - the full permission set shows all seven
  - `useStaffVenue`:
    - a user with `venue_id: 3` gets 3 and no picker, even with `?venue=5`
    - a user with null `venue_id` gets `?venue=5` when venue 5 is listed, and otherwise the first venue
- [ ] **Step 2:** `npm test -- manila money selection staff`. Expected: FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** Run the full gate, which includes `NEXT_PUBLIC_API_BASE_URL=http://localhost:8000/api/v1 npm run build`. Expected: green.
- [ ] **Step 5:** Commit `Add the staff console frame and venue context`.

### Task 3: The schedule — grid, walk-ins, and the day's bookings

**Files:**
- Create `src/features/admin-bookings/{schemas,api,keys,index}.ts`, `hooks/{use-day-bookings,use-admin-booking-mutations}.ts`, and `components/{schedule,walk-in-bar,day-bookings}.tsx`.
- Create `src/app/(app)/staff/schedule/page.tsx`.
- Modify `src/features/bookings/index.ts`: export `BookingSchema` and `StatusBadge`.
- Test: `tests/unit/admin-bookings/schedule.test.tsx`.

**Interfaces:**
- Schemas:
  - `AdminBookingSchema = BookingSchema.extend({ player: {id, first_name, last_name, email} | null, created_by_user_id: number })`
  - `WalkInRequest {court_id, starts_at, hours, notes?}`
- `adminBookingsApi`:
  - `day(venueId, date)` → `GET /admin/bookings?venue_id=&date=&limit=200`
  - `list(params)`
  - `get(id)`
  - `walkIn(req)` → `POST /admin/bookings/walk-in`
  - `cancel(id, reason)`
  - `markPaid(id, method)`
  - `outcome(id, outcome)`
  - `resolve(id, resolution)`
  - `unfulfilled(page)`
  - `reopenClosure(id, reason)` → `POST /admin/closures/{id}/cancel`
- `adminBookingKeys`: `all`, `day(venueId, date)`, `detail(id)`, `unfulfilled()`, `list(params)`.
- `<Schedule />` is the page body:
  - the venue picker, and a `DateStrip` with `start={-7}` and `days={21}`
  - the `AvailabilityGrid` with an `onCellTap` that runs `tapCell` with the staff predicate. Only when the user holds `BOOKING:CreateWalkIn`; otherwise the grid is read-only.
  - `WalkInBar`: the court, the time range, a subtotal, an optional notes input and **Book walk-in**
  - `DayBookings`
  - `useAvailability` and `useDayBookings`, each with `refetchInterval: 30_000`
- Walk-in result handling:
  - 201: invalidate the day's grid and list, clear the selection, and show a toast with the reference code.
  - `SLOT_UNAVAILABLE`: toast the detail, clear the selection, refetch.
  - `TOO_FAR_BACK` and other errors: toast the detail and keep the selection.
- `DayBookings` row: `formatTimeRange`, court name, who (`player` full name / "Walk-in" / "Closure"), `StatusBadge`, `formatPeso(total_centavos)`. Each row links to `/staff/bookings/{id}`. The empty state reads "No bookings on this day."
- The selection is keyed by venue + date, as in Phase 7.

- [ ] **Step 1: Failing tests.**
  - Walk-ins:
    - With `BOOKING:CreateWalkIn`, tapping a `PAST` cell then **Book walk-in** posts `{court_id, starts_at, hours: 1}` and shows a toast with the reference code.
    - Without the permission, the grid has no buttons.
    - 409 `SLOT_UNAVAILABLE` clears the selection.
  - The day list:
    - It shows the player's name for a `PLAYER_BOOKING`, "Walk-in" for a `WALK_IN` and "Closure" for a `MAINTENANCE` row.
    - It renders `IN_PROGRESS` from the payload.
  - Changing the date drops the selection.
- [ ] **Step 2:** `npm test -- schedule`. Expected: FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** Run the full gate. Expected: green.
- [ ] **Step 5:** Commit `Add the staff schedule with walk-ins`.

### Task 4: Booking detail and its action routes

**Files:**
- Create `src/features/admin-bookings/staff-actions.ts`, `hooks/use-admin-booking.ts`, and `components/{admin-booking-detail,mark-paid-form,outcome-buttons,reason-form}.tsx`.
- Create `src/app/(app)/staff/bookings/[bookingId]/page.tsx`.
- Tests: `tests/unit/admin-bookings/staff-actions.test.ts`, `tests/unit/admin-bookings/admin-booking-detail.test.tsx`.

**Interfaces:**
- `type StaffAction = 'markPaid' | 'outcome' | 'cancel' | 'reopen'`.
- `staffActions(b: AdminBooking, permissions: string[], nowMs: number): Set<StaffAction>`:
  - `markPaid`: `BOOKING:MarkPaid` ∧ `type = WALK_IN` ∧ `payment_status = UNPAID` ∧ `booking_status ∈ {CONFIRMED, COMPLETED}`
  - `outcome`: `BOOKING:SetOutcome` ∧ `type ≠ MAINTENANCE` ∧ `booking_status = CONFIRMED` ∧ (`display_status = IN_PROGRESS` ∨ `Date.parse(starts_at) ≤ nowMs`)
  - `cancel`: `BOOKING:CancelAny` ∧ `type ≠ MAINTENANCE` ∧ `booking_status = CONFIRMED`
  - `reopen`: `CLOSURE:Manage` ∧ `type = MAINTENANCE` ∧ `booking_status = CONFIRMED`
- `<ReasonForm label submitLabel onSubmit pending />` has a textarea. Submit is disabled while the reason is blank.
- `<AdminBookingDetail id />` shows:
  - the reference code, type, venue/court, `formatDayLong` + `formatTimeRange`, each hour's price, the fee and the total
  - the player name and email (or "Walk-in"), `payment_method`, `notes`, `cancellation_reason`
  - the `StatusBadge`
  - the actions from `staffActions`:
    - mark-paid: two buttons, **Cash** and **GCash (manual)**, sending `CASH` / `MANUAL_GCASH`
    - outcome: **Completed** and **No-show**
    - cancel: a `ReasonForm` with the warning "No refund is issued."
    - reopen: a `ReasonForm`, "Reopen court"
- Every mutation writes its response into `detail(id)` and invalidates `adminBookingKeys.all` and `availabilityKeys.all`.
- Any error renders `errorMessage(err)` and refetches `detail(id)`.
- An unknown id or a 404 shows "Booking not found" with a link to `/staff/schedule`.

- [ ] **Step 1: Failing tests.**
  - `staffActions`: one allow and one deny per rule. This includes:
    - `outcome` denied before the start
    - `outcome` allowed for `IN_PROGRESS`
    - `outcome` allowed for `CONFIRMED` after the end
    - mark-paid allowed for `COMPLETED`+`UNPAID` walk-ins
    - nothing offered for a `PLAYER_BOOKING` `PENDING` hold
    - a staff user without `BOOKING:MarkPaid` sees no mark-paid
  - Detail:
    - **Cash** posts `{payment_method: 'CASH'}` and the badge shows Paid.
    - **Review Focus 3:** a 409 `ALREADY_PAID` renders the detail and refetches, and the refetched PAID state hides mark-paid.
    - Cancel is disabled until a reason is typed, then posts `{reason}`.
    - Reopen on a closure posts to `/admin/closures/{id}/cancel`.
    - 404 shows not-found.
- [ ] **Step 2:** `npm test -- staff-actions admin-booking-detail`. Expected: FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** Run the full gate. Expected: green.
- [ ] **Step 5:** Commit `Add the staff booking detail and its actions`.

### Task 5: The resolution queue

**Files:**
- Create `src/features/admin-bookings/components/resolution-queue.tsx` and `src/app/(app)/staff/resolution/page.tsx`.
- Test: `tests/unit/admin-bookings/resolution-queue.test.tsx`.

**Interfaces:**
- `<ResolutionQueue />` lists `GET /admin/bookings/unfulfilled` (infinite, **Load more**). Each row shows:
  - the reference code and player, as a link to the detail
  - venue · court and the time
  - `formatPeso(total_centavos)`, the status badge, and `notes`. Notes carry the `[system]` reason.
- **Resolve** shows only with `BOOKING:CancelAny`. It opens a `ReasonForm` labelled "How was this settled? (refund, rebook, void)", posts `/resolve {resolution}`, and on success invalidates `unfulfilled()`. The row disappears.
- `NOT_FLAGGED` renders its detail and refetches.
- The empty state reads "Nothing to resolve. Every paid booking has its court."

- [ ] **Step 1: Failing tests.**
  - Rows render with their notes.
  - Resolve requires text, posts `{resolution}`, and removes the row.
  - Without `BOOKING:CancelAny` there is no Resolve button.
  - Empty state.
- [ ] **Step 2:** `npm test -- resolution-queue`. Expected: FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** Run the full gate. Expected: green.
- [ ] **Step 5:** Commit `Add the resolution queue`.

### Task 6: Venues and courts

**Files:**
- Create `src/features/setup/{schemas,api,keys,index}.ts`, `hooks/use-setup.ts`, and `components/{venue-admin,court-admin}.tsx`.
- Create `src/app/(app)/staff/venues/page.tsx` and `src/app/(app)/staff/courts/page.tsx`.
- Tests: `tests/unit/setup/venue-admin.test.tsx`, `tests/unit/setup/court-admin.test.tsx`.

**Interfaces:**
- **Venues API.**
  - `GET /admin/venues?limit=100` → `Page<Venue>`
  - `POST /admin/venues {name, address?}`
  - `PATCH /admin/venues/{id} {name?, address?, is_active?}`
- **Courts API.**
  - `GET /admin/venues/{id}/courts?limit=100` → `Page<Court>`, where `Court = {id, venue_id, name, sort_order, is_active}`
  - `POST /admin/venues/{id}/courts {name, sort_order?}`
  - `PATCH /admin/courts/{id} {name?, sort_order?, is_active?}`
- **`setupKeys`.** `venues()`, `courts(venueId)`, `rules(courtId)`, `prices(courtId, date)`.
- **`<VenueAdmin />`** is gated on `VENUE:Manage`.
  - It lists venues, inactive ones included and marked.
  - It has an add form with name and address.
  - Each row can be renamed and can **Deactivate**/**Activate**.
  - Changes invalidate `setupKeys.venues()` and the catalog venues key.
- **`<CourtAdmin />`** is gated on `COURT:Manage`.
  - It uses `useStaffVenue` with its picker.
  - It lists courts by `sort_order` and has an add form (name, sort order).
  - Each row can be renamed, change its sort order, **Deactivate**/**Activate**, and link to **Prices** at `/staff/courts/{id}/pricing`.
  - Changes invalidate `courts(venueId)` and `availabilityKeys.all`.
- Errors render `errorMessage`. Nothing is optimistic.

- [ ] **Step 1: Failing tests.**
  - Adding a venue posts `{name, address}` and the list refetches.
  - Deactivate posts `{is_active: false}`.
  - Courts list in `sort_order`.
  - Adding a court posts to the selected venue.
  - A 409/422 detail renders.
  - Without `COURT:Manage` the page shows "Not available".
- [ ] **Step 2:** `npm test -- venue-admin court-admin`. Expected: FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** Run the full gate. Expected: green.
- [ ] **Step 5:** Commit `Add venue and court management`.

### Task 7: Pricing

**Files:**
- Create `src/features/setup/components/{price-rules,price-preview}.tsx` and `src/app/(app)/staff/courts/[courtId]/pricing/page.tsx`.
- Test: `tests/unit/setup/pricing.test.tsx`.

**Interfaces:**
- **Pricing API.**
  - `GET /admin/courts/{id}/price-rules` → `PriceRule[]`, where `PriceRule = {id, court_id, day_of_week: number|null, hour_of_day: number|null, price_centavos}`
  - `POST /admin/courts/{id}/price-rules {day_of_week?, hour_of_day?, price_centavos}`
  - `PATCH /admin/price-rules/{id} {price_centavos}`
  - `DELETE /admin/price-rules/{id}`
  - `GET /admin/courts/{id}/prices?date=` → `[{starts_at, hour, price_centavos}]`
  - Also `GET /admin/courts/{id}` for the court's name.
- **`<PriceRules courtId />`** is gated on `PRICE:Manage`.
  - Rows show day ("Any day"/"Sunday"…"Saturday"), hour ("Any hour"/`formatHour`-style "6 PM") and price.
  - Price is edited inline through `parsePesos`; an invalid entry shows "Enter an amount like 600 or 600.50."
  - **Delete** asks for confirmation inline.
  - The add form has a day select (Any, 0–6), an hour select (Any, 0–23) and a price.
  - Omit `day_of_week`/`hour_of_day` when "Any".
  - Writes invalidate `rules(courtId)`, `prices(courtId, *)` and `availabilityKeys.all`.
- **`<PricePreview courtId />`** shows a `DateStrip` (today + 13) and the 24 resolved prices for the chosen day.

- [ ] **Step 1: Failing tests.**
  - Adding "Any day / 6 PM / 600" posts `{hour_of_day: 18, price_centavos: 60000}` with **no** `day_of_week` key.
  - Editing a price to `"750.50"` patches `{price_centavos: 75050}`.
  - `"12.345"` is refused client-side with no request.
  - A duplicate rule's 409 detail renders.
  - Delete requires a second click.
  - The preview renders 24 rows starting "12 AM".
- [ ] **Step 2:** `npm test -- pricing`. Expected: FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** Run the full gate. Expected: green.
- [ ] **Step 5:** Commit `Add court pricing rules and preview`.

### Task 8: Closures

**Files:**
- Create `src/features/setup/components/{closure-form,closure-list}.tsx` and `src/app/(app)/staff/closures/page.tsx`.
- Test: `tests/unit/setup/closures.test.tsx`.

**Interfaces:**
- **Closures API.**
  - `POST /admin/closures {court_ids, starts_at, ends_at, reason}` → `AdminBooking[]`
  - The list is `adminBookingsApi.list({venue_id, type: 'MAINTENANCE', booking_status: 'CONFIRMED', limit: 200})`, filtered to `Date.parse(ends_at) > Date.now()` and sorted by `starts_at`.
  - Reopen is `adminBookingsApi.reopenClosure(id, reason)`.
- **`<ClosureForm venueId />`** has:
  - court checkboxes from the catalog courts, with a **Select all** toggle
  - a start day and hour, and an end day and hour, each as a Manila `<input type=date>` plus an hour select
  - a reason
- Submit posts `manilaInstant(startDay, startHour)` → `manilaInstant(endDay, endHour)`.
- Client-side checks, with no request when they fail:
  - at least one court
  - end after start
  - a non-blank reason
- Server results:
  - `SLOT_UNAVAILABLE` renders "Some of those hours are already booked. Nothing was closed." followed by the server detail.
  - `RANGE_TOO_LONG` and `IN_THE_PAST` render their detail.
- **`<ClosureList venueId />`** shows each row's court, range and reason (from `notes`), plus a **Reopen** `ReasonForm`.
- Gated on `CLOSURE:Manage`.

- [ ] **Step 1: Failing tests.**
  - **Review Focus 5:** 23 Oct 10 PM → 24 Oct 2 AM on two courts posts `court_ids: [1, 2]`, `starts_at: '2026-10-23T22:00:00+08:00'`, `ends_at: '2026-10-24T02:00:00+08:00'`.
  - End before start is blocked client-side.
  - A 409 shows the "Nothing was closed" copy.
  - The list hides ended closures.
  - Reopen requires a reason and posts to `/admin/closures/{id}/cancel`.
- [ ] **Step 2:** `npm test -- closures`. Expected: FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** Run the full gate. Expected: green.
- [ ] **Step 5:** Commit `Add court closures`.

### Task 9: Staff accounts

**Files:**
- Create `src/features/team/{schemas,api,keys,index}.ts`, `hooks/use-team.ts`, `components/{team-list,invite-form}.tsx` and `src/app/(app)/staff/users/page.tsx`.
- Test: `tests/unit/team/team.test.tsx`.

**Interfaces:**
- **`teamApi`.**
  - `list(page)` → `GET /admin/users?page=&limit=20` (`Page<User>`, where `User` mirrors `UserOut`)
  - `inviteStaff({email, first_name, last_name, mobile?})` → `POST /admin/venue-staff`
  - `inviteAny({…, role, venue_id})` → `POST /admin/users`
  - `update(id, {is_active})` → `PATCH /admin/users/{id}`
  - `resend(id)` → `POST /admin/users/{id}/invitation`
  - `revoke(id)` → `DELETE /admin/users/{id}/invitation`
- **`<TeamList />`** is gated on `USER:Manage`.
  - Each row shows name, email, role label (`VENUE_MANAGER` → "Manager", `VENUE_STAFF` → "Staff", `SUPER_ADMIN` → "Owner", `PLAYER` → "Player") and status ("Invited" while `email_verified_at` is null; otherwise "Active" or "Deactivated").
  - Invited rows get **Resend** / **Revoke**. Other rows get **Deactivate** / **Reactivate**, except the signed-in user's own row, which gets no deactivate.
  - **Load more** appears while `has_more` is true.
- **`<InviteForm />`.**
  - With `VENUE:Manage`, it adds a role select (Manager / Staff) and a venue select (catalog venues) and calls `inviteAny`.
  - Otherwise it calls `inviteStaff` with **no role or venue in the body**.
  - On success it toasts "Invitation sent to {email}" and invalidates the list.

- [ ] **Step 1: Failing tests.**
  - **Review Focus 2:** a manager's invite posts to `/admin/venue-staff`, and the body's keys are exactly `email, first_name, last_name` (+ `mobile` when filled).
  - An owner's invite posts to `/admin/users` with `role` and `venue_id`.
  - An invited row offers Resend/Revoke, and Revoke calls DELETE.
  - Deactivate patches `{is_active: false}`.
  - Your own row has no Deactivate.
  - The server's 409 (email taken) detail renders.
- [ ] **Step 2:** `npm test -- team`. Expected: FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** Run the full gate. Expected: green.
- [ ] **Step 5:** Commit `Add staff account management`.

### Task 10: Parameters, docs

**Files:**
- Create `src/features/parameters/{schemas,api,keys,index}.ts`, `components/parameter-table.tsx` and `src/app/(app)/staff/parameters/page.tsx`.
- Modify `pikol-frontend/CLAUDE.md` (the new slices and `/staff/*` routes), `pikol-backend/CLAUDE.md` (the parameters admin API in "What exists now") and `../CLAUDE.md`:
  - Current state: Phase 8 complete; next is Phase 9 / the Maya adapter.
  - Repo table: frontend "Phase 8 complete".
  - Docs list: add this plan.
- Test: `tests/unit/parameters/parameter-table.test.tsx`.

**Interfaces:**
- `ParameterSchema {key, value, value_type: 'int'|'decimal'|'bool'|'string', description, updated_at, updated_by_user_id: number|null}`.
- `parametersApi.list()` → `GET /admin/parameters`. `parametersApi.update(key, value)` → `PATCH /admin/parameters/{key} {value}`.
- **`<ParameterTable />`** is gated on `PARAMETER:Manage`.
  - Each row shows the key, the description, the value, an inline editor typed by `value_type` (number input for int/decimal, checkbox for bool, text for string) and `formatDateTime(updated_at)`.
  - **Save** sends the value as a string.
  - A 422 `INVALID_PARAMETER` renders its detail under that row.
  - On success the row updates from the response.
  - The page states: "Changes apply within a minute and affect new bookings only — prices already charged never change."

- [ ] **Step 1: Failing tests.**
  - Rows render with their descriptions.
  - Saving `hold_duration_minutes = 20` patches `{value: '20'}` and shows the new value.
  - A 422 detail renders under the row, and the old value stays.
  - Without `PARAMETER:Manage`, the page shows "Not available".
- [ ] **Step 2:** `npm test -- parameter-table`. Expected: FAIL.
- [ ] **Step 3:** Implement and update the three briefs.
- [ ] **Step 4:** Run the full gate. Expected: green.
- [ ] **Step 5: End-to-end** against the local backend with the mock provider, through the exact HTTP calls the console makes:
  - **Staff:**
    - walk-in for the current hour
    - mark it paid in cash
    - set its outcome
    - cancel a player's booking with a reason
    - close two courts overnight, then reopen
    - check that the grid shows `CLOSED`, then `AVAILABLE`
  - **Manager:**
    - add a court
    - add a 6 PM rule
    - check that the preview shows it
    - invite staff
    - check that the body carries no role or venue
  - **Owner:**
    - change `hold_duration_minutes`
    - check that a cross-field violation is refused
    - resolve a flagged booking
  - **Cross-venue:** staff at venue A get 404 for a venue-B booking id.
- [ ] **Step 6:** Commit `Add the parameters screen` and the brief updates. Push all three repos once green.

## Out of scope

- Reports and revenue dashboards.
- Editing a booking's time (cancel and rebook instead).
- Refunds through Maya (the resolution queue records how a human settled it; spec §5.5).
- Horizontal court scrolling beyond ~8 courts (§8.1, deferred).
- The real Maya adapter.
- Audit-log browsing.
