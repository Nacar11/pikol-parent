# Customer booking demo — design

Date: 2026-10-27. Covers `pikol-backend` and `pikol-frontend`. Payment rules
live in [ADR 0004](../adr/0004-online-payment-flow-and-statuses.md); this
document does not repeat them, it builds the screens and API around them.

## Intent

The mobile app is not built yet, but the customer flow is the core of the
product and needs testing now. The owner wants a **web stand-in for the app**:
a customer signs in, picks a venue, picks hours on its courts, pays through
Maya Checkout (the mock until sandbox keys exist), and sees the payment's
status. Because it uses the same customer API the app will use, whatever is
found here is a real bug, and the API is proven by the time the app needs it.

## Goals

- Customers book and pay on the web, end to end, against the real backend.
- Every rule in ADR 0004 is exercised: the 5-minute hold, no cart, one venue
  per payment, the five statuses, failed releases the hold, the simple refund
  flow, paid bookings are final.
- The mock Maya can produce every outcome, so all 17 ADR test scenarios can
  be run by hand and by Playwright.
- A small favorite-venues API exists for the mobile app's future Favorites
  tab.

## Non-goals

- No cart, no "save for later", no wishlist.
- No sign-up page on the web. Testers get accounts from a seed command or by
  being added to the Customer group in Masters → User groups.
- No favorites UI on the web.
- No real Maya adapter (sandbox keys don't exist yet). The provider port
  gains `void` and `refund`; the mock implements them.
- No customer cancellation of paid bookings, no partial refunds.
- No polish beyond the console's black-and-white system: this is a test page
  for the app's flow, not the storefront.

## Anti-goals

- Two customers paying for the same hour.
- A customer losing money for an hour they did not get, without the system
  voiding or refunding it.
- Any status decided by the browser's redirect from Maya rather than by the
  re-fetched payment (ADR 0002).
- Web-only API shortcuts the mobile app would not have.

## Decided with the owner

- **Who lands where:** everyone in the Customer group lands on the booking
  home after sign-in — including staff who are also customers. Staff who are
  not customers still land on the console. (Replaces today's
  `/customer-app` "use the mobile app" page.)
- **Layout:** option A from the mockups — venue cards; then, for one venue, a
  date strip and a **courts × hours grid**. Taken hours are hatched, free ones
  show their price. Tapping free cells on any courts selects them; a black
  bar shows "N hours · M courts · total + fee" and **Pay now**. One hour
  minimum; one venue per payment.
- **Several dates on one payment:** switching the date strip keeps earlier
  picks in the bar, so one payment can cover different courts on different
  dates at the same venue. The bar and hold page group line items by date.
  Switching venue clears the picks (another venue is another payment).
- **After Pay now** (approved mockup):
  1. **Hold page:** "Pay within 4:52" countdown, the 5-minute warning
     ("hours reopen for everyone after that, and a late payment is voided"),
     line items per court, convenience fee, total, **Continue to Maya**,
     **Cancel and release hours**.
  2. **Result page:** one of Pending ("Confirming your payment…", updates by
     itself, "Check again"), Successful (reference code, "Paid bookings are
     final"), Failed ("hours were released", "Book again"), Cancelled ("hours
     are free again"), Refunded ("Paid too late — fully refunded", nothing
     booked).
  3. **My bookings:** every payment with status pill, reference, hours and
     total; a pending one shows its countdown and **Continue to Maya**.
- **Favorites:** a table and API only (list, add, remove my favorite venues).
- **Parameters:** `hold_duration_minutes` = 5, `booking_horizon_days` (14),
  `min_lead_minutes` (20) and `max_active_holds_per_player` (3) stay editable
  on the Parameters page. The owner must be able to set `min_lead_minutes`
  to **0** (book 1–2 PM at 12:58 PM).
- **Accounts:** no sign-up page; seed customer accounts and/or add people to
  the Customer group by hand.

## My calls (flagged for review)

1. **Drop the rule `min_lead_minutes >= hold_duration_minutes`.** It blocks
   the owner's 0-minute lead. Consequence the owner accepted in
   conversation: a hold may run a few minutes into the hour it holds. An
   hour whose start has passed still can never be held.
2. **A checkout groups several booking groups.** ADR 0001 keeps one court per
   `booking_groups` row; non-consecutive hours on one court are separate
   groups too. One payment now spans every group picked together at one
   venue. Introduce a `checkouts` row (customer, venue, status, totals,
   `expires_at`, reference code) that the groups and the payment attempt
   belong to. All its groups share one `expires_at`, so they are held and
   released together. `docs/adr/pikol.dbml` changes first, then the
   migration. The builder may rename it (`orders`) but not merge it into
   `booking_groups`.
3. **Creating a checkout is all-or-nothing** inside one savepoint, like
   closures today: if any hour is taken, nothing is held and the response
   names the taken hours (409 `SLOT_UNAVAILABLE`).
4. **The hold page comes before Maya** (approved mockup), so the checkout is
   created on **Pay now** and Maya's session on **Continue to Maya**. Both
   happen inside the same 5 minutes; the hold starts on Pay now.
5. **Retry route removed** (`POST /bookings/me/{id}/checkout`), per ADR 0004
   decision 10. **Cancel** (`POST /bookings/me/{id}/cancel`) is allowed only
   while the checkout is pending — it is "Cancel and release hours"; a paid
   booking is refused (ADR 0004 decision 6).
6. **Customer routes move to checkout ids**, under `/api/v1/checkouts/me`
   (create, list mine, get one, cancel, refresh-payment). Identity comes from
   the token, never the URL. The availability and catalog reads stay as they
   are.
7. **Demo seed:** `python -m src.devtools.demo_seed --customers N` creates
   `customer1..N@example.com` (shared dev password, printed) in the Customer
   group. It refuses to run against a database whose name ends in `_e2e`
   or that looks like production (`supabase` in the host), and is idempotent.
8. **Routes:** the booking home is `/book` (venue cards), `/book/[venueId]`
   (grid), `/book/checkout/[id]` (hold page), `/book/result/[id]` (result,
   also the Maya redirect target for success/failure/cancel), `/book/mine`
   (my bookings). A customer-only layout with a plain top bar (brand, "My
   bookings", sign out) — not the staff sidebar.
9. **Polling:** the result page refetches every 3 s for up to 2 minutes while
   Pending, then offers "Check again" (which calls refresh-payment).
10. **Price display:** cells show the resolved price of that hour (already in
    the availability read); the bar and hold page add the convenience fee
    from `convenience_fee_percent`, frozen when the checkout is created
    (system rule: price is frozen).

11. **`max_active_holds_per_player` counts pending checkouts**, not booking
    groups: one Pay now is one hold, however many courts it covers.

## Mock Maya

The existing mock checkout page (`MOCK_CHECKOUT_ENABLED`) gets one button per
outcome — Pay, Decline, Back out, Expire — each sending the matching webhook
so the real `sync_payment` path runs. The mock provider gains `void` and
`refund`, recording which was used. A "pay late" scenario is the tester
waiting past 5 minutes before pressing Pay.

## Testing

- Backend: every ADR 0004 scenario (1–17) has an API test.
- Frontend: unit tests for the grid selection, the bar's totals, the
  countdown, and each result state's copy.
- Playwright (real stack, existing suite): book-and-pay, two customers racing
  for one hour, decline releases the hours, back out releases them, pay late
  with a free hour (booked), pay late with a taken hour (refunded), my
  bookings shows each status.

## Left to the builder

- Table and column names for checkouts, refunds and favorites, within the
  dbml-first rule.
- Exact API payload shapes (snake_case, list envelope).
- Component breakdown of the grid and bar; reuse `StatStrip`, `Pill`,
  `PageHeader`, `notify`, `Skeleton` where they fit.
- The order of work, as long as backend slices land with their tests before
  the screens that use them.
