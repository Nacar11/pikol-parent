# Phase 5 — Payments (mock Maya first) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A hold comes with a checkout; paying it confirms the booking
through a webhook that is never trusted, only used as a cue to ask the
provider; late payments re-acquire their slots or land in a resolution
queue; the player gets a receipt — all against a mock that speaks Maya's
wire format, so the real adapter later is a transport swap.

**Architecture:** `payments` is a rich module. A `PaymentProvider` port
(ADR 0002 shape: `create_checkout`, `parse_webhook` — untrusted,
`fetch_payment` — authoritative). `maya_wire.py` holds Maya's JSON shapes,
the centavos↔pesos conversion, and the status mapping; `MockProvider` stores
checkouts in process memory **as Maya-shaped JSON** and parses them back
through `maya_wire`, so the conversion code the real adapter will use is
exercised from day one. The webhook handler records the event, re-fetches,
and applies the result through `BookingRepository.set_status` only.

**Spec:** design §5.3–5.6, §7.4–7.6; [ADR 0002](../adr/0002-maya-payment-provider.md)
(authoritative where it differs); dbml `payments`, `webhook_events`.

## Global Constraints

Everything in Phases 2–4, plus:

- **The webhook body is never trusted.** Only the payment id is read from it; status, amount, and reference come from `fetch_payment`.
- **Idempotency by insert:** `webhook_events.provider_event_id = "{payment_id}:{status}"`, unique. A duplicate is a 200 that does nothing.
- **Forward-only:** `payment_status` never leaves `PAID`; a terminal booking never returns to `CONFIRMED` except by re-acquiring a lapsed hold.
- **The external call is outside the booking transaction:** commit the hold, then create the checkout.
- **Payloads are PII — never logged.** Log `provider_event_id` and the outcome.
- Money stays integer centavos; pesos exist only inside `maya_wire.py`.

## Decisions this plan makes

| Decision | Why |
|---|---|
| **`MayaProvider` (the real HTTP adapter) is a later task**, after sandbox keys | Its JSON shapes must be checked against sandbox responses, not docs; the mock already runs the shared wire code |
| `PAYMENT_PROVIDER` accepts only `mock` for now | Config that cannot select a missing adapter |
| The mock's hosted page and "simulate" routes mount only when `MOCK_CHECKOUT_ENABLED=true` | Without them nobody can mark a mock payment paid, so a deploy left on the mock cannot give courts away |
| `provider_payment_status` gains `CANCELLED` (dbml first) | Maya's `PAYMENT_CANCELLED` is distinct from failure |
| `POST /bookings` returns `BookingOut` + `checkout_url`; provider failure → 502 `PAYMENT_PROVIDER_UNAVAILABLE`, hold kept | §5.3 step 3; the player retries with `POST /bookings/me/{id}/checkout` |
| `POST /bookings/me/{id}/refresh-payment` re-fetches the latest checkout | ADR 0002: closes the gap when every webhook landed on a sleeping backend |
| Re-acquisition only for `PENDING`(lapsed)/`EXPIRED` groups whose first slot is still in the future; anything else paid-but-unhousable → `needs_resolution` | ADR 0002 §2 |
| A payment whose amount or reference does not match the group → recorded, not applied, `needs_resolution` | A mismatch is either a bug or an attack; a human looks |
| A second successful payment for an already-paid group → `needs_resolution` | Double charge: refund |
| `POST /admin/bookings/{id}/resolve {resolution}` (`BOOKING:CancelAny`) clears the flag, appending the note | The queue needs an exit |
| Webhook source-IP allowlist: `MAYA_WEBHOOK_ALLOWED_IPS` (empty = off), `TRUSTED_PROXY_HOPS` | Defence in depth only (ADR 0002) |
| Raw-payload retention purge is not built | `pg_cron` hygiene job, per §7.5; Phase 9 |

## Review Focus

1. **Forged webhook** — a body claiming `PAYMENT_SUCCESS` for a payment the provider says is pending confirms nothing. Task 4.
2. **Replayed webhook** — the same success delivered twice sends one receipt and writes one event. Task 4.
3. **Payment after hold lapsed, slot free** — booking restored to `CONFIRMED/PAID`, no flag. Task 5.
4. **Payment after hold lapsed, slot taken** — `EXPIRED/PAID` + `needs_resolution`, the other booking untouched. Task 5.
5. **Webhook lost entirely** — the return page's refresh confirms the booking. Task 6.

---

### Task 1: Port, Maya wire format, and the mock provider

**Files:** `src/payments/domain/{enums,port}.py`, `src/payments/adapters/{maya_wire,mock}.py`. Tests `tests/payments/test_maya_wire.py`, `tests/payments/test_mock_provider.py`.

- Enums: `PaymentProviderName (MOCK, MAYA)`, `ProviderPaymentStatus (PENDING, PAID, FAILED, EXPIRED, CANCELLED)`; `TERMINAL = {PAID, FAILED, EXPIRED, CANCELLED}`.
- Port dataclasses: `CheckoutSession(checkout_id, checkout_url)`, `WebhookNotice(payment_id)`, `ProviderPayment(payment_id, status, amount_centavos, reference, raw: dict)`; `ProviderError(Exception)`; `PaymentNotFound(ProviderError)`.
- `maya_wire`: `to_pesos(centavos) -> Decimal` (2 dp), `from_pesos(value: str | int | Decimal) -> int` (rejects sub-centavo and float), `MAYA_STATUSES` (`PAYMENT_SUCCESS→PAID`, `PAYMENT_FAILED→FAILED`, `PAYMENT_EXPIRED→EXPIRED`, `PAYMENT_CANCELLED→CANCELLED`, `PENDING_TOKEN`/`PENDING_PAYMENT`→`PENDING`; unknown → `PENDING`), `checkout_request(...) -> dict`, `parse_payment(raw) -> ProviderPayment`, `parse_notice(body: bytes) -> WebhookNotice` (`ProviderError` on non-JSON or no `id`).
- `MockProvider`: `create_checkout` stores a Maya-shaped payment (`PENDING_TOKEN`) keyed by a uuid; `checkout_url = {api_base}/api/v1/mock-maya/checkouts/{id}`; `settle(id, maya_status)`; `fetch_payment` → `parse_payment(stored)`; unknown id → `PaymentNotFound`.
- [ ] Tests: `1648 centavos ↔ "16.48"`; `from_pesos("16.485")` raises; `from_pesos(16.48)` (float) raises; status mapping; `parse_notice` rejects garbage; mock round-trip create → settle → fetch returns `PAID` and the original centavos.

### Task 2: `payments` and `webhook_events` schema

**Files:** dbml (parent) first; `src/payments/persistence/{models,repository}.py`; migration `0005_payments.py`. Test `tests/payments/test_payments_schema.py`.

- Columns per dbml; `provider_payment_status` with `CANCELLED`. Repository: `add_checkout(session, group, provider, session_)`, `latest_for_group(session, group_id)`, `by_checkout_id(session, provider, checkout_id)`, `record_event(session, provider, event_id, payload) -> bool` (savepoint; `False` on duplicate).
- [ ] Tests: duplicate `provider_event_id` rejected; `record_event` returns `False` the second time and leaves the session usable.

### Task 3: Checkout on hold, and retry

**Files:** `src/payments/provider.py` (`get_payment_provider` dependency, settings `payment_provider`, `api_base_url`, `mock_checkout_enabled`), `src/payments/service.py` (`start_checkout`), modify bookings player controller + dto. Test `tests/payments/test_checkout.py`.

- `POST /bookings` → 201 `CheckoutBookingOut` (`BookingOut` + `checkout_url`). `POST /bookings/me/{id}/checkout` → new checkout for a live, unpaid hold (`409 HOLD_NOT_LIVE` otherwise).
- [ ] Tests: hold returns a checkout URL and one `PENDING` payment row for `total_centavos`; provider raising → 502 and the hold still exists; retry creates a second payment row; retry on a lapsed hold → 409.

### Task 4: The webhook

**Files:** `src/payments/controllers/webhooks.py`, service `handle_webhook`/`apply_payment`, `src/email/templates.py` receipt. Test `tests/payments/test_webhook.py`.

- `POST /api/v1/webhooks/maya`: allowlist → `parse_notice` (400) → `fetch_payment` (`PaymentNotFound` → 200 ignored; other `ProviderError` → 502 so Maya retries) → non-terminal → 200 → `record_event` (duplicate → 200) → `apply_payment` → commit → receipt after commit. No auth, no rate limit.
- `apply_payment` (live hold + match): payment row `PAID`, `set_status(CONFIRMED)`, `payment_status=PAID`, `payment_method=ONLINE`. `FAILED` on an unpaid live hold → group `payment_status=FAILED` (still retryable). `EXPIRED`/`CANCELLED` → payment row only.
- [ ] Tests: success confirms and emails one receipt; **forged success body for a still-pending payment confirms nothing**; replay → one event, one receipt; failure marks `FAILED` and a retry checkout then succeeds; amount mismatch → not confirmed, `needs_resolution`; unknown payment id → 200 no-op; allowlist on → other IP 403, listed IP via `X-Forwarded-For` accepted.

### Task 5: Late payments and the resolution queue

**Files:** service `apply_payment` branches; bookings admin controller. Test `tests/payments/test_late_payment.py`.

- Lapsed/EXPIRED + future + free → savepoint: `expire_lapsed`, `set_status(CONFIRMED)`, PAID. Collision (`uq_court_slot_active`) or started → stays/becomes `EXPIRED`, `payment_status=PAID`, `needs_resolution=true`. CANCELLED-by-player + paid, or already PAID → `needs_resolution`.
- `GET /admin/bookings/unfulfilled` (`BOOKING:ReadAny` + scope) and `POST /admin/bookings/{id}/resolve {resolution}` (`BOOKING:CancelAny`).
- [ ] Tests: lapsed + free → confirmed; lapsed + taken → flagged, other booking untouched; started → flagged; double payment → flagged; queue is scoped; resolve clears and records the note.

### Task 6: Mock checkout page and payment refresh

**Files:** `src/payments/controllers/mock_maya.py`, player controller. Test `tests/payments/test_mock_flow.py`.

- `GET /api/v1/mock-maya/checkouts/{id}` → HTML with Pay / Fail / Cancel. `POST /api/v1/mock-maya/checkouts/{id}/{outcome}` (`success|failure|cancel|expire`, `?deliver_webhook=true`) → settles and, if delivering, posts the Maya-shaped body to the real webhook handler; 303 to `{frontend_base_url}/checkout/return?booking_id=`. Mounted only when `mock_checkout_enabled`.
- `POST /bookings/me/{id}/refresh-payment` (rate-limited 10/min) → fetch the latest checkout and apply through the same idempotent path; returns `BookingOut`.
- [ ] Tests: end-to-end pay via the simulate route → booking `CONFIRMED/PAID`; `deliver_webhook=false` then refresh → confirmed; routes absent when disabled (404); another player's refresh → 404.
- [ ] Update briefs and `.env.example`; push; CI green.
