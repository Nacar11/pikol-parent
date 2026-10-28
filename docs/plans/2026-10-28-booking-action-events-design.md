# Booking action events — design

**Date:** 2026-10-28 · **Status:** approved · **Touches:** both repos, the
schema (`action_events`), the staff booking page and the customer result page

## Intent

A booking changes state many times — held, paid, confirmed, completed, or
cancelled, expired, refunded — and today only its *current* state survives.
Staff cannot tell **what happened, in what order, at what exact time, and who
did it.** The user wants a tracker like Shopee's parcel tracking: a vertical
list of events, newest first, each with its own colour, starting from the
moment the booking was made ("awaiting payment"). Staff see it on the booking
page; **customers see it too**, for their own bookings.

## Goals

1. A new append-only **`action_events`** table records every change to a
   booking's lifecycle and its payment, **in the same transaction as the
   change**, so the history can never disagree with the booking.
2. **Staff:** an "Action events" section on `/staff/bookings/{id}`, below the
   summary.
3. **Customers:** the same tracker on `/book/result/{checkoutId}` (the page
   "My bookings" links to), covering every booking in that payment.
4. Each event shows: a **coloured dot with an icon**, a **title**, a
   **coloured status chip carrying its word**, **one line of detail** (amount
   and method, reason, hold deadline…), the **exact Manila time and date**,
   and **who**: a staff member by name, the customer, or "System".
5. **Colour per status** (user's choice, mockup A): amber = waiting,
   blue = confirmed, green = paid / completed, red = failed, cancelled,
   expired, no-show, refunded, flagged, grey = neutral (closure created,
   reopened, resolved).

## Non-goals

- No editing or deleting events, by anyone. No "undo" from the timeline.
- No audit log for non-booking things (courts, prices, users, groups).
- No notifications (email/push) driven by events.
- No change to how bookings, checkouts or payments transition — this records
  them; it does not alter them.

## Anti-goals

- **A history that can drift from the booking.** Events are written inside the
  transaction that makes the change, never after commit, never best-effort.
- **Invented times.** An event's time is when the thing happened. A lapsed hold
  is dated at its deadline, not at whenever lazy cleanup happened to run.
- **Leaking staff internals to customers.** Customers never see staff names,
  the resolution queue, or `[system]` notes.
- **Colour as the only signal.** Every chip carries its word; the icon differs
  per kind (WCAG, and the console's own rule).

## Design

### Schema — `action_events` (dbml first, then migration `0011`)

| Column | Type | Notes |
|---|---|---|
| `id` | bigint PK | |
| `booking_group_id` | int FK → booking_groups, NULL | set for events about one booking |
| `checkout_id` | int FK → checkouts, NULL | set for every event of a player booking, and alone for payment events |
| `type` | enum `action_event_type` | below |
| `actor_kind` | enum `STAFF · CUSTOMER · SYSTEM` | |
| `actor_user_id` | int FK → users, NULL | NULL for SYSTEM |
| `occurred_at` | timestamptz NOT NULL | when it happened (not always `now()`) |
| `data` | jsonb NOT NULL default `{}` | per-type extras, keys below |

CHECKs: at least one of `booking_group_id`/`checkout_id`;
`(actor_kind = 'SYSTEM') = (actor_user_id IS NULL)`. Indexes on
`(booking_group_id, occurred_at)` and `(checkout_id, occurred_at)`.

**Payment events belong to the checkout** (`booking_group_id` NULL): one
"Payment successful" for a payment covering three courts, not three. A
booking's timeline is its own events **plus** its checkout's
booking-less events.

### Event types

| Type | Written when | Actor | Colour | Customer sees | `data` |
|---|---|---|---|---|---|
| `HOLD_PLACED` | checkout created (per group) | customer | amber | ✅ | `expires_at`, `total_centavos` |
| `WALK_IN_BOOKED` | walk-in created | staff | blue | — (no customer) | `notes?` |
| `CLOSURE_CREATED` | closure created (per court) | staff | grey | — | `reason` |
| `PAYMENT_STARTED` | Maya session first recorded | customer | amber | ✅ | `amount_centavos` |
| `PAYMENT_SUCCEEDED` | provider says paid (checkout) | system | green | ✅ | `amount_centavos`, `method: ONLINE`, `late?` |
| `PAYMENT_FAILED` | provider says failed | system | red | ✅ | |
| `PAYMENT_CANCELLED` | customer backed out at Maya | system | red | ✅ | |
| `CONFIRMED` | player booking confirmed (incl. late re-acquire) | system | blue | ✅ | `late?` |
| `MARKED_PAID` | staff collects cash / GCash | staff | green | ✅ | `method`, `amount_centavos` |
| `COMPLETED` | outcome set | staff | green | ✅ | |
| `NO_SHOW` | outcome set | staff | red | ✅ | |
| `CANCELLED` | staff, customer or payment failure cancels | any | red | ✅ | `reason` |
| `EXPIRED` | hold lapsed (incl. lazy cleanup) | system | red | ✅ | — `occurred_at` = the hold's deadline |
| `REFUNDED` / `VOIDED` | late payment undone | system | red | ✅ | `amount_centavos`, `reference` |
| `FLAGGED` | put in the resolution queue | system | red | ❌ staff only | `reason` |
| `RESOLVED` | staff resolves | staff | grey | ❌ staff only | `resolution` |
| `REOPENED` | closure cancelled | staff | grey | — | `reason` |

**Derived on read, never stored** (they have no write of their own):

- **"Play started"** (blue): `CONFIRMED` and now ≥ first slot's start, dated
  at that start. Mirrors `IN_PROGRESS`, which is also derived.
- **"Hold ended"** (red): a group still `PENDING` whose `expires_at` has
  passed, dated at `expires_at`. Still-PENDING means cleanup never ran, so no
  stored `EXPIRED` event can exist yet. Covers the lapsed-hold-nobody-touched
  case (the PENDING trap) without a scheduler.

### Writing events

- One module, `src/action_events/` (domain enum → model → repository →
  `event_service.record(...)`), called from the existing services **before
  their commit**: `bookings/service.py` (walk-in, closure, mark paid, outcome,
  cancel, reopen, resolve), `checkouts/service.py` (create, cancel),
  `payments/service.py` (start, paid, failed/cancelled, late paid, refund,
  void, flag), and `booking_repository.expire_lapsed` (one `EXPIRED` per
  expired group, dated at its own `expires_at` — read the deadlines before the
  UPDATE clears them).
- `mark_paid` and `set_outcome` gain the acting staff user (the controllers
  already resolve it). `PaymentService._flag` gains the session, so flagging
  records `FLAGGED` in the same transaction.
- Append-only: the repository has `insert` and reads only.

### Reading events

- **Staff:** `GET /api/v1/admin/bookings/{id}/events`, behind
  `BOOKING:ReadAny`, venue-scoped (out of scope → 404, like the booking).
- **Customer:** `GET /api/v1/checkouts/me/{id}/events`, behind
  `require_customer`, own checkouts only (someone else's → 404). Filters out
  `FLAGGED` and `RESOLVED`, and shows staff as **"Venue staff"**, never by
  name, and the customer's own actions as **"You"**. Staff see the customer
  by name.
- Both return the list envelope `{ data, total, page, limit, has_more }`,
  newest first, derived events merged in. Each item: `id` (null when derived),
  `type`, `occurred_at`, `actor: {kind, name}`, `booking_id`,
  `reference_code`, `court_name`, `data`, `derived`.

### Frontend

- A shared **`ActionTimeline`** in the `bookings` slice (already shared by the
  console and the demo): the vertical line, dot + icon, title, chip, detail,
  Manila time ("3:02 PM · Wed 28 Oct"). One mapping
  `type → {title, tone, icon, detail(data)}`.
- **Colour tokens** in `globals.css` (`--event-wait/-confirmed/-done/-bad/-neutral`,
  each with a chip background and foreground meeting 4.5:1), documented as
  the one sanctioned exception to black-and-white, for this section only.
- Staff page: under the summary; refetched after each staff action.
  Customer result page: under the existing status; refetched while it polls.
  A checkout with several courts names the court on each booking event.

### Existing data

Nothing is deployed. The migration backfills only what is known exactly:
creation (`created_at`, as `HOLD_PLACED` / `WALK_IN_BOOKED` /
`CLOSURE_CREATED`) and cancellation (`cancelled_at`, reason, canceller).
Nothing else is invented.

## Tests

- **Backend:** each transition writes exactly the expected events (type,
  actor, data, time); a failed transition (rolled-back savepoint, e.g.
  `SLOT_UNAVAILABLE`) writes none; a lazily expired hold is dated at its
  deadline; a payment covering two courts writes one payment event; the
  customer endpoint hides `FLAGGED`/`RESOLVED` and staff names; out-of-scope
  and someone-else's → 404; derived "Play started" and "Hold ended".
- **Frontend unit:** order, every type's title/chip word, colour class per
  tone, customer view.
- **E2E:** staff walk-in → cash → cancel shows three events in order on the
  booking page; a customer's book-and-pay shows hold → payment started →
  payment successful → confirmed on the result page.

## Calls I made (check these)

- Customers see the tracker on the **result page** (per payment), not a new
  page; staff names become "Venue staff" for them; the resolution queue stays
  staff-only. Cancel reasons **are** shown to customers.
- Payment events are recorded **once per checkout**, not per court.
- "Play started" and "Hold ended" are **derived on read**, matching how
  `IN_PROGRESS` already works, so no scheduler is added.
- **Refunded/voided are red**, as played back.
- Backfill covers only creation and cancellation; nothing is invented.
- Staff are shown by name only, without their group.

## Docs to update

`docs/adr/pikol.dbml` (table, enums, Appendix A CHECKs), root `CLAUDE.md`
(current state; the colour exception), frontend `CLAUDE.md` (the B&W rule's
exception, `ActionTimeline`), backend `CLAUDE.md` (module table).
