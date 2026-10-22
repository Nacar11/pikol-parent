# Phase 7 — Frontend Player Booking Flow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A signed-in player picks a venue, reads the day's grid, taps a run
of consecutive free hours on one court, holds them, pays on the (mock) Maya
page with a visible countdown, lands on a return page that confirms from our
database, and manages everything afterwards from My Bookings.

**Architecture:** Three new slices on Phase 6's shape. `features/catalog`
(venues), `features/availability` (the grid query, the pure selection
reducer, and **the one `AvailabilityGrid` component** Phase 8 reuses for
staff), and `features/bookings` (hold, pay, return, my bookings). Time and
money formatting live in `lib/` because both surfaces need them. Every
server state change goes through a mutation and its result is read back.
Nothing in the grid or a booking is updated optimistically.

**Tech Stack:** as Phase 6. **No new dependencies**: no dialog library
(cancellation uses an inline two-step confirm) and no date library (`Intl`
with `timeZone: 'Asia/Manila'`).

**Spec:** design §5.2–5.3, §7.2, §7.4, §8, §8.1–8.2;
[ADR 0002](../adr/0002-maya-payment-provider.md) §2 (a checkout outlives its
hold). Backend contract as built in Phases 3–5: `GET /venues`,
`GET /availability`, `POST /bookings`, `GET /bookings/me[/{id}]`,
`POST /bookings/me/{id}/{cancel,checkout,refresh-payment}`.

## Global Constraints

Everything in Phase 6, plus:

- **All times render in Asia/Manila**, through `lib/manila.ts` only. No `toLocaleString()` without `timeZone`, no `getHours()`. Tests run with `TZ=Asia/Dubai` so any leak fails.
- **The grid `date` parameter is the Manila calendar day** (`YYYY-MM-DD`). Never derive it from `new Date().toISOString()`.
- **Never optimistic.** A cell is "held" only after `POST /bookings` returns 201. Local selection is shown as *selected*, never as held.
- **The redirect never confirms.** `/checkout/return` reads `GET /bookings/me/{id}` and calls `refresh-payment`. It never trusts its query string beyond `booking_id`.
- **Show the status the server sends.** Status badges render `display_status` (`IN_PROGRESS` and lapsed-hold `EXPIRED` come from the server). The client never recomputes it.
- **Money is integer centavos** until `formatPeso`. Fees and totals come from the server. The client sums only grid cell prices, for a labelled subtotal.
- Branch on error `code`, render `detail`: `SLOT_UNAVAILABLE`, `HOLD_LIMIT`, `TOO_SOON`, `BEYOND_HORIZON`, `NOT_ON_THE_HOUR`, `PAYMENT_PROVIDER_UNAVAILABLE`, `HOLD_NOT_LIVE`, `NOT_CANCELLABLE`, `CANCELLATION_CUTOFF_PASSED`, `NOT_FOUND`.
- Query keys only from each slice's `keys.ts`. Cross-slice imports go through the barrel.

## Decisions this plan makes

| Decision | Why |
|---|---|
| **Backend change: `BookingOut` gains `court_name` and `venue_name`** (Task 1) | My Bookings without names is unusable, and joining client-side is N+1 over venues and courts. This is scope added to the backend, flagged here on purpose. `AdminBookingOut` inherits it, which Phase 8 needs too |
| Routes: `/venues` (list) → `/venues/[venueId]?date=` (grid + hold) → `/bookings/[bookingId]` (pay page) → Maya → `/checkout/return?booking_id=` → `/bookings` (mine) | Spec §8 route shape. `date` is in the URL so a reload or a shared link keeps the day |
| The venue page reads court names from the grid response. It does not call `GET /venues/{id}/courts` | The grid already carries `courts[].name` in grid order. One request per day view (§7.2) |
| Venue name comes from the cached `GET /venues` list (`limit=100`). An unknown id shows not-found | The catalog has no `GET /venues/{id}`, and one owner's venues fit in one page |
| Date strip: today plus 13 days (14 chips), Manila days | Below the default `booking_horizon_days` (30). Players cannot read parameters, so if an admin lowers the horizon, `BEYOND_HORIZON`'s detail is what the player sees |
| **There is a pay page between the hold and Maya.** After the 201 the player lands on `/bookings/[id]`, which shows the countdown, the server's price/fee/total, and a **Pay** button | §8.2 requires the countdown on *our* checkout screen. Maya's hosted page cannot show it |
| Pay uses the `checkout_url` from the create response, cached under `bookingKeys.checkout(id)`. If none is cached (after a reload, or after `payment_status = FAILED`), it calls `POST /bookings/me/{id}/checkout` | One checkout per hold in the common path. The retry route exists for every other path |
| Countdown runs on the client clock against `expires_at`. At zero it refetches the booking and renders whatever `display_status` the server returns | A skewed clock can only make the timer end early or late. The decision itself stays the server's |
| Lapsed hold copy: "Your hold has expired. If you already opened the payment page, paying now may not secure this court." Plus **Book again** back to the grid | ADR 0002 §2: a Maya checkout outlives the hold, so the honest state must say so |
| Return page: poll `GET /bookings/me/{id}` every 2 s for up to 30 s. Call `refresh-payment` at t = 0 s and t = 15 s | Covers a webhook that is late or lost. Two calls per visit stay well inside the 10/min limit |
| `EXPIRED` + `PAID` renders "We received your payment but could not secure the court. Staff will contact you about a refund or a new slot." | That is the `needs_resolution` case. `BookingOut` does not expose the flag, and the two axes already identify it |
| Online booking UI shows only when `canBookOnline(permissions)` is true, i.e. the user lacks `BOOKING:ReadAny`. Staff see the grid read-only | `POST /bookings` is player-only (403 for staff). This matches `landingPath` and needs no role parsing |
| Cancel is offered when `display_status ∈ {PENDING, CONFIRMED}` and the booking has not started. A `CONFIRMED`+`PAID` cancel warns "No refund". The cutoff is enforced by the server (`CANCELLATION_CUTOFF_PASSED`) | Players cannot read `cancellation_cutoff_minutes` |
| `PAYMENT_PROVIDER_UNAVAILABLE` (502) on hold: the hold *was* committed but the body has no id. Navigate to `/bookings` with "Your hold is saved, but the payment page is unavailable. Open it below to try again." | §5.3 step 3. The hold is real and must not look lost |
| Grid refetches every 60 s while visible. The selection is reconciled against each fresh grid and dropped if any selected cell is no longer `AVAILABLE` | A selection over a cell somebody else just took must not survive into a 409 |
| My Bookings uses "Load more" on `has_more` (page size 20) | Matches the list envelope with no pagination component to build |

## Review Focus

1. **Midnight in Manila.** At 2026-10-22T16:30Z (00:30 Manila, 23 Oct), "today" is `2026-10-23` and the first grid row reads "12 AM", on any browser time zone. Tests in Task 2 and Task 3.
2. **Selection over a cell that just became BOOKED** after a 60 s refetch is dropped before the player can submit it. A 409 `SLOT_UNAVAILABLE` on submit clears the selection and refetches the grid. Tests in Task 4.
3. **Webhook lost.** The return page stays "Confirming…" until the t = 15 s `refresh-payment` flips the booking to `CONFIRMED/PAID`, then shows success. A total timeout points to My Bookings and does not claim success. Tests in Task 6.
4. **Hold lapses on the pay page.** The countdown reaches zero, the booking is refetched, the server says `EXPIRED`, and Pay is replaced by the honest lapsed copy. It is never a dead Pay button. Test in Task 5.
5. **Provider down at hold time.** A 502 lands the player on My Bookings with the saved hold visible, not on an error page. Test in Task 4.

---

### Task 1: Backend — names on `BookingOut`

**Repo:** `pikol-backend`.

**Files:**
- Modify: `src/courts/persistence/models.py` (add `venue: Mapped[VenueModel] = relationship(lazy="joined")`, no migration because the FK exists), `src/bookings/dto.py` (`BookingOut.court_name: str`, `venue_name: str`, set in `BookingOut.of` from `group.court.name` / `group.court.venue.name`).
- Test: `tests/bookings/test_my_bookings.py`, `tests/payments/test_checkout.py`.

**Produces:** `BookingOut` (and so `CheckoutBookingOut`, `AdminBookingOut`) carry `court_name`, `venue_name`.

- [ ] **Step 1: Failing tests.** `test_my_bookings_carry_court_and_venue_names` asserts `GET /bookings/me` and `GET /bookings/me/{id}` items have `court_name == court.name` and `venue_name == venue.name`. `test_hold_response_carries_names` asserts the same on the `POST /bookings` 201.
- [ ] **Step 2:** `uv run pytest tests/bookings/test_my_bookings.py tests/payments/test_checkout.py -v`. Expected: FAIL on `KeyError: 'court_name'`.
- [ ] **Step 3:** Implement as listed under Files. Joined eager loading chains through `group.court`, so the async session never lazy-loads.
- [ ] **Step 4:** `uv run ruff format . && uv run ruff check . && uv run mypy src && uv run pytest && uv run alembic check`. All green, and `alembic check` reports no changes.
- [ ] **Step 5:** Commit `Add court and venue names to booking responses`. Push `pikol-backend`.

### Task 2: `lib/manila.ts` and `lib/money.ts`

**Repo:** `pikol-frontend` from here on.

**Files:**
- Create: `src/lib/manila.ts`, `src/lib/money.ts`.
- Modify: `vitest.config.ts` (`test.env.TZ = 'Asia/Dubai'`).
- Test: `tests/unit/lib/manila.test.ts`, `tests/unit/lib/money.test.ts`.

**Produces:**
- `MANILA = 'Asia/Manila'`.
- `manilaToday(now?: Date): string` returns `YYYY-MM-DD`.
- `addDays(day: string, n: number): string` does pure calendar arithmetic on the string, with no time zone.
- `isManilaDay(value: string): boolean` validates the `?date=` param.
- `formatHour(iso: string): string` returns "6 PM" / "12 AM".
- `formatDayChip(day: string): { weekday: string; date: string }` returns "Fri" / "23 Oct".
- `formatDayLong(day: string): string` returns "Friday, 23 October 2026".
- `formatTimeRange(startsIso: string, endsIso: string): string` returns "6 PM – 8 PM".
- `formatDateTime(iso: string): string` returns "23 Oct, 6:00 PM".
- `formatCountdown(ms: number): string` returns "14:59", floored and clamped at "0:00".
- `formatPeso(centavos: number): string` returns "₱1,200" when whole and "₱16.48" otherwise.

- [ ] **Step 1: Failing tests.**
  - `manilaToday(new Date('2026-10-22T16:30:00Z'))` is `'2026-10-23'`, and `manilaToday(new Date('2026-10-22T15:59:00Z'))` is `'2026-10-22'`.
  - `addDays('2026-12-31', 1)` is `'2027-01-01'`.
  - `formatHour('2026-10-22T16:00:00Z')` is `'12 AM'`, and `formatHour('2026-10-23T10:00:00Z')` is `'6 PM'`.
  - `formatTimeRange` across midnight gives "11 PM – 1 AM".
  - `formatCountdown(899_999)` is `'14:59'`, and `formatCountdown(-5)` is `'0:00'`.
  - `isManilaDay('2026-02-30')` is false.
  - `formatPeso(120000)` is `'₱1,200'`, and `formatPeso(1648)` is `'₱16.48'`.
- [ ] **Step 2:** `npm test -- manila money`. Expected: FAIL, module not found.
- [ ] **Step 3:** Implement with `Intl.DateTimeFormat('en-PH', { timeZone: MANILA, … })` and `formatToParts`. `addDays` uses `Date.UTC` on the parsed parts.
- [ ] **Step 4:** `npm test -- manila money`. Expected: PASS under `TZ=Asia/Dubai`.
- [ ] **Step 5:** Commit `Add Manila time and peso formatting`.

### Task 3: Catalog and the read-only grid

**Files:**
- Create: `src/lib/page.ts`.
- Create: `src/features/catalog/{schemas,api,keys,index}.ts`, `hooks/use-venues.ts`, and `components/venue-list.tsx`.
- Create: `src/features/availability/{schemas,api,keys,index}.ts`, `hooks/use-availability.ts`, `components/{availability-grid,date-strip}.tsx`.
- Create: `src/app/(app)/venues/[venueId]/page.tsx`. Replace `src/app/(app)/venues/page.tsx`.
- Modify: `src/features/auth/landing.ts` + `index.ts` (export `canBookOnline`).
- Test: `tests/unit/catalog/venue-list.test.tsx`, `tests/unit/availability/grid.test.tsx`.

**Interfaces:**
- **Catalog**
  - `VenueSchema {id, name, address: string|null, is_active}`. `pageSchema<T>(item)` lives in `src/lib/page.ts`, mirroring `{data, total, page, limit, has_more}`.
  - `catalogApi.venues(): Promise<Page<Venue>>` calls `GET /venues?limit=100`.
  - `catalogKeys.venues()`, `useVenues()`, `useVenue(id: number): Venue | undefined | 'not-found'`.
- **Availability**
  - `GridStatus = 'AVAILABLE'|'BOOKED'|'CLOSED'|'PAST'`.
  - `GridSlotSchema {starts_at, hour, status, price_centavos}`, `GridCourtSchema {court_id, name, slots}`, `AvailabilitySchema {venue_id, date, courts}`.
  - `availabilityApi.day(venueId: number, date: string)` calls `GET /availability?venue_id=&date=`.
  - `availabilityKeys.all`, `availabilityKeys.day(venueId, date)`, `useAvailability(venueId, date)` (`refetchInterval: 60_000`, `refetchIntervalInBackground: false`).
- **Grid component**
  - `<AvailabilityGrid data={Availability} selection={Selection | null} onCellTap?={(courtId: number, index: number) => void} />`. Rows are hours and columns are courts. The time column and the court header row are `sticky`.
  - Cells:
    - `AVAILABLE` shows `formatPeso(price)` and is a `<button>` only when `onCellTap` is passed.
    - `BOOKED` and `CLOSED` are a filled block with an `aria-label` ("Booked" / "Closed").
    - `PAST` is muted and has no price.
  - Every cell's accessible name includes the court, the hour and the state, e.g. "Court 1, 6 PM, available, ₱600".
- `<DateStrip value={string} onChange={(day: string) => void} days={14} />`.
- `canBookOnline(permissions: string[]): boolean`.

- [ ] **Step 1: Failing tests.**
  - The venue list renders the names and links to `/venues/{id}`. Empty data renders "No venues are open for booking yet."
  - The grid renders 24 rows starting "12 AM" for a fixture whose first slot is `…T16:00:00Z`.
  - A `BOOKED` cell has no price text and is not a button.
  - Peak price text appears in the cell (`₱600` at 6 PM against `₱500` at 5 PM).
  - With no `onCellTap` there are no buttons at all.
  - A zod parse of a malformed availability payload throws in `availabilityApi.day`.
  - `canBookOnline(['BOOKING:ReadAny'])` is false, and `canBookOnline([])` is true.
- [ ] **Step 2:** `npm test -- catalog availability landing`. Expected: FAIL.
- [ ] **Step 3: Implement.** The venue page reads `params.venueId` and `?date=`. When the date is missing or `!isManilaDay`, it calls `router.replace` with `manilaToday()`. It renders the venue name (not-found state for unknown ids), the `DateStrip`, and the grid. While loading it shows `PageLoading`, and on error it shows `errorMessage(err)` in an `Alert`.
- [ ] **Step 4:** `npm run lint:ci && npm run format:check && npm run typecheck && npm test`. All green.
- [ ] **Step 5:** Commit `Add the venue list and the availability grid`.

### Task 4: Selection and placing a hold

**Files:**
- Create: `src/features/availability/selection.ts`.
- Create: `src/features/bookings/{schemas,api,keys,index}.ts`, `hooks/use-booking-mutations.ts`.
- Create: `src/features/availability/components/hold-bar.tsx`, `src/features/bookings/components/book-venue-day.tsx`.
- Modify: the venue page, to compose `BookVenueDay` when `canBookOnline` and the bare grid otherwise.
- Test: `tests/unit/availability/selection.test.ts`, `tests/unit/bookings/book-venue-day.test.tsx`.

**Interfaces:**
- **Selection** (`selection.ts`, pure)
  - `type Selection = { court_id: number; start: number; end: number }` holds inclusive slot indexes.
  - `MAX_HOURS = 12`.
  - `tapCell(sel: Selection | null, grid: Availability, courtId: number, index: number): Selection | null` follows these rules:
    - A tap on a non-`AVAILABLE` cell leaves the selection unchanged.
    - With no selection, or with the selection on another court, a tap starts a new 1-hour selection there.
    - Tapping the lone selected cell clears it.
    - Tapping after `start` on the same court, where every cell `start..index` is `AVAILABLE` and the span is ≤ 12, sets `end = index`.
    - Any other tap (before start, inside a range, a gap in between, or > 12) restarts at the tapped cell.
  - `reconcile(sel, grid): Selection | null` returns null if the court is gone or any selected cell is not `AVAILABLE`.
  - `toHoldRequest(sel, grid): { court_id; starts_at; hours }`, where `starts_at` is the slot's server ISO string, unmodified.
  - `subtotalCentavos(sel, grid): number`.
- **Bookings schemas**
  - `BookingStatus`, `PaymentStatus`, `DisplayStatus` (adds `IN_PROGRESS`), `PaymentMethod` enums.
  - `BookingSchema` mirrors `BookingOut` exactly, including Task 1's `court_name`, `venue_name`.
  - `CheckoutBookingSchema = BookingSchema.extend({ checkout_url })`.
- **`bookingsApi`**
  - `create(req)` calls `POST /bookings`.
  - `mine(page)` calls `GET /bookings/me?page=&limit=20`.
  - `get(id)` calls `GET /bookings/me/{id}`.
  - `cancel(id)` calls `POST /bookings/me/{id}/cancel` with body `{}`.
  - `checkout(id)` calls `POST /bookings/me/{id}/checkout`.
  - `refreshPayment(id)` calls `POST /bookings/me/{id}/refresh-payment`.
- **`bookingKeys`**: `all`, `mine()`, `detail(id)`, `checkout(id)`.
- **`useCreateHold()`** on success:
  - `setQueryData(bookingKeys.detail(id), booking)`
  - `setQueryData(bookingKeys.checkout(id), checkout_url)`
  - invalidate `availabilityKeys` for that day and `bookingKeys.mine()`
  - `router.push('/bookings/{id}')`
- **`<HoldBar selection subtotal onSubmit pending />`** reads "Court 2 · 6 PM – 8 PM · 2 hours · ₱1,200 + booking fee" and shows a **Hold these hours** button.

- [ ] **Step 1: Failing tests.**
  - `tapCell` gets one test per rule above, plus a 13-hour span restarting.
  - `reconcile` drops a selection whose second cell became `BOOKED` and keeps an untouched one.
  - `toHoldRequest` returns `{court_id, starts_at: slots[start].starts_at, hours: end-start+1}`.
  - `book-venue-day`:
    - **Not optimistic.** Before the request resolves, no cell is labelled held and the button shows pending.
    - 201 navigates to `/bookings/42`.
    - **409 `SLOT_UNAVAILABLE`** shows the detail as a toast, clears the selection and refetches the grid.
    - **`HOLD_LIMIT`** shows the detail and a link to `/bookings`.
    - **502 `PAYMENT_PROVIDER_UNAVAILABLE`** navigates to `/bookings?notice=provider` and invalidates `mine()`.
    - A refetched grid that marks a selected cell `BOOKED` removes the hold bar.
- [ ] **Step 2:** `npm test -- selection book-venue-day`. Expected: FAIL.
- [ ] **Step 3:** Implement. `BookVenueDay` keeps `Selection` in state and applies `reconcile` in an effect keyed on the query's `dataUpdatedAt`.
- [ ] **Step 4:** Run the full gate. Green.
- [ ] **Step 5:** Commit `Add consecutive-hour selection and holds`.

### Task 5: The pay page — countdown, pay, cancel

**Files:**
- Create: `src/features/bookings/hooks/use-booking.ts`, `components/{booking-detail,hold-countdown,pay-button,cancel-booking,status-badge}.tsx`, `src/app/(app)/bookings/[bookingId]/page.tsx`.
- Test: `tests/unit/bookings/booking-detail.test.tsx`.

**Interfaces:**
- **`useBooking(id)`** wraps `bookingKeys.detail(id)`.
- **`usePay(id)`**:
  - returns the cached `bookingKeys.checkout(id)` URL if present
  - else calls `bookingsApi.checkout(id)` and caches its `checkout_url`
  - then `window.location.assign(url)`
  - on `HOLD_NOT_LIVE`, refetches the detail
- **`useCancelBooking(id)`** updates `detail(id)` with the response and invalidates `mine()` and `availabilityKeys.all`.
- **`<HoldCountdown expiresAt onElapsed />`** ticks once a second via `setInterval` and calls `onElapsed` once at zero.
- **`<StatusBadge status={DisplayStatus} payment={PaymentStatus} />`**.
- **`<BookingDetail id />`** shows the reference code, venue, court, `formatDayLong` + `formatTimeRange`, per-hour prices, the fee, and the total, plus one action block chosen in this order:
  1. `display_status = PENDING`: countdown + Pay (labelled "Pay ₱X") + cancel. If `payment_status = FAILED`, it adds "Your last payment did not go through. Try again."
  2. `display_status = EXPIRED`, `payment_status ≠ PAID`: the lapsed copy from Decisions + **Book again** (`/venues/{venue_id}?date={Manila day of starts_at}`).
  3. `EXPIRED` + `PAID`: the unhoused copy from Decisions.
  4. `CONFIRMED`, not started: confirmed note + cancel ("No refund" warning when `PAID`).
  5. Anything else: the status only.
- **`<CancelBooking>`** confirms in two steps inline (**Cancel booking** → **Yes, cancel** / **Keep it**), then renders the server's `detail` on `NOT_CANCELLABLE` / `CANCELLATION_CUTOFF_PASSED`.

- [ ] **Step 1: Failing tests** (fake timers, fixed `now`).
  - A PENDING booking with `expires_at = now + 90 s` shows "1:30".
  - After advancing 90 s, the detail is refetched. The mock then returns `display_status: 'EXPIRED'`, and the lapsed copy shows with no Pay button.
  - Pay with a cached URL calls `location.assign` with it and **does not** call `checkout`. With no cache it calls `checkout` once.
  - `payment_status: 'FAILED'` shows the retry line.
  - `EXPIRED`+`PAID` shows the unhoused copy.
  - Cancel needs the second click. A `CANCELLATION_CUTOFF_PASSED` response shows its detail.
  - A `CONFIRMED`+`PAID` cancel shows "No refund".
  - `IN_PROGRESS` from the server renders the badge "In progress" and offers no cancel.
  - A 404 renders "Booking not found" with a link to `/bookings`.
- [ ] **Step 2:** `npm test -- booking-detail`. Expected: FAIL.
- [ ] **Step 3:** Implement. Stub `window.location.assign` through a small `navigateExternal(url)` in `src/lib/navigate.ts` so tests can spy on it.
- [ ] **Step 4:** Run the full gate. Green.
- [ ] **Step 5:** Commit `Add the pay page with hold countdown and cancellation`.

### Task 6: The checkout return page

**Files:**
- Create: `src/features/bookings/return-outcome.ts`, `hooks/use-confirm-booking.ts`, `components/checkout-return.tsx`, `src/app/(app)/checkout/return/page.tsx`.
- Test: `tests/unit/bookings/return-outcome.test.ts`, `tests/unit/bookings/checkout-return.test.tsx`.

**Interfaces:**
- **`returnOutcome(b: Booking): ReturnOutcome`** returns one of `'confirmed' | 'failed' | 'unhoused' | 'lapsed' | 'cancelled' | 'pending'`:
  - `PAID` + `booking_status ∈ {CONFIRMED, COMPLETED}` gives `confirmed`.
  - `PAID` + `EXPIRED` gives `unhoused`.
  - `FAILED` gives `failed`.
  - `display_status = EXPIRED` (unpaid) gives `lapsed`.
  - `CANCELLED` gives `cancelled`.
  - Anything else gives `pending`.
- **`useConfirmBooking(id)`**:
  - `useQuery(bookingKeys.detail(id))` with `refetchInterval` 2000 while the outcome is `pending` and under 30 s have elapsed, otherwise `false`
  - fires `bookingsApi.refreshPayment(id)` at mount and at 15 s; each result is written into `detail(id)`, and a 429 or other error is ignored and polling continues
  - returns `{ booking, outcome, timedOut }`
- **`<CheckoutReturn bookingId />`** renders by outcome:
  - `pending` shows "Confirming your booking…" with a spinner.
  - `confirmed` shows "You're booked", the reference code, court and time, and a link to the booking.
  - `failed` shows "Payment didn't go through" + **Try again** linking to `/bookings/{id}`.
  - `unhoused` shows the unhoused copy.
  - `lapsed` shows the lapsed copy.
  - `timedOut` + `pending` shows "We're still waiting for the payment provider. Your booking will update in My Bookings." with a link, and never a success state.
  - A missing or non-numeric `booking_id` shows "We couldn't find that booking" + link.
- On reaching `confirmed`, invalidate `bookingKeys.mine()` and `availabilityKeys.all`.

- [ ] **Step 1: Failing tests.**
  - One `returnOutcome` case per row above.
  - The return page sees PENDING → PENDING → (the t = 15 s refresh returns CONFIRMED/PAID) → success, and `refreshPayment` is called exactly twice.
  - Webhook already landed: the first GET is CONFIRMED/PAID, so success shows immediately and polling stops.
  - Always PENDING for 30 s shows the timeout copy and **no** "You're booked".
  - `FAILED` shows Try again.
  - `?booking_id=abc` shows not-found without any request.
  - The query string alone (`?booking_id=7&status=success`) never renders success while the server says PENDING.
- [ ] **Step 2:** `npm test -- return-outcome checkout-return`. Expected: FAIL.
- [ ] **Step 3:** Implement. Wrap the page in `Suspense` for `useSearchParams` (Next 15 build requirement).
- [ ] **Step 4:** Run the full gate. Green.
- [ ] **Step 5:** Commit `Add the checkout return page that confirms from the server`.

### Task 7: My Bookings, navigation, docs

**Files:**
- Create: `src/features/bookings/hooks/use-my-bookings.ts` (`useInfiniteQuery` on `bookingKeys.mine()`), `components/my-bookings.tsx`, `src/app/(app)/bookings/page.tsx`, `src/components/layout/app-nav.tsx`.
- Modify: `app-shell.tsx` (a `nav` slot), `src/app/(app)/layout.tsx` (passes `AppNav`), `src/features/bookings/index.ts`.
- Modify: `pikol-frontend/CLAUDE.md` (slices and routes added) and `../CLAUDE.md` (Current state: Phase 7 complete, next Phase 8).
- Test: `tests/unit/bookings/my-bookings.test.tsx`.

**Interfaces:**
- **`<MyBookings />`**:
  - rows show `venue_name · court_name`, `formatDayLong` + `formatTimeRange`, `formatPeso(total_centavos)`, the `StatusBadge` and the reference code, each linking to `/bookings/{id}`
  - "Load more" while `has_more`
  - empty state "No bookings yet" + link to `/venues`
  - `?notice=provider` shows the Task 4 notice
- **`<AppNav />`** shows "Venues" and "My bookings" when `canBookOnline` and nothing otherwise (Phase 8 adds staff links), marking the active link with `usePathname`.

- [ ] **Step 1: Failing tests.**
  - The list renders names, times in Manila (a `…T10:00:00Z` start reads "6 PM") and the server's status, with `IN_PROGRESS` rendered from the payload.
  - Load more requests `page=2` and appends.
  - Empty state.
  - The provider notice shows.
  - The nav is hidden for `['BOOKING:ReadAny']`.
- [ ] **Step 2:** `npm test -- my-bookings`. Expected: FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** Run the full gate, including `npm run build`. Green.
- [ ] **Step 5: Manual end-to-end** against the local backend with `PAYMENT_PROVIDER=mock`, `MOCK_CHECKOUT_ENABLED=true`. Each must be observed:
  - Log in as a player, pick a venue, pick tomorrow, hold 2 consecutive hours, and see the countdown. Pay on the mock page, see the return page confirm, and see the booking in My Bookings as Confirmed/Paid.
  - The same slot then shows booked on the grid.
  - Repeat with `deliver_webhook=false`: the return page confirms through `refresh-payment`.
  - Fail a payment, then use Try again and succeed.
  - Cancel a hold, and the slot frees on the grid.
  - Log in as staff: the grid is read-only and the nav has no booking links.
- [ ] **Step 6:** Commit `Add My Bookings and player navigation`. Push `pikol-frontend` once CI is green. Commit the brief update and push `pikol-parent`.

## Out of scope

Staff schedule, walk-ins, and the grid in staff mode (Phase 8). Profile editing
(`PATCH /users/me`). Horizontal court scrolling beyond ~8 courts (§8.1, deferred).
The real Maya adapter. A public parameters endpoint (horizon, cutoff, hold length).
