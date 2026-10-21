# ADR 0002: Maya replaces PayMongo as the payment provider

**Date:** 2026-10-21
**Status:** Accepted
**Supersedes:** design §2.1 ("Why PayMongo and not Stripe") and the
PayMongo-specific parts of §4.5, §5.3–5.6, §7.4–7.6, and §9.5

## Context

The design chose PayMongo. Before any payment code was written (Phase 5), the
owner switched the provider to **Maya Business — Maya Checkout**. Nothing in
the database or the codebase references a provider yet: Phase 1 migrated only
`parameters`, so this is a documentation change plus a schema change made
before the first migration that would carry it.

The provider port (§5.6) was designed so that this kind of swap is cheap. It
mostly is. But Maya's checkout is **not** shaped like PayMongo's in three
places, and each one changes a design decision rather than an adapter detail.

## What Maya does differently

Verified against Maya's developer docs on 2026-10-21 (sources at the end).

### 1. Webhooks are not signed

PayMongo signs every webhook; the design verified the signature before
trusting a byte. **Maya's online-payment webhooks carry no signature.** Maya's
only documented control is allowlisting its source IPs:

| Environment | Webhook source IPs |
|---|---|
| Sandbox | `13.229.160.234`, `3.1.199.75` |
| Production | `18.138.50.235`, `3.1.207.200` |

An IP allowlist alone is weak here. The backend runs behind Render's proxy, so
the client IP comes from `X-Forwarded-For`, and trusting the wrong hop of that
header is a one-line mistake that makes the allowlist decorative.

**Decision: a webhook is a notification, never evidence.** The handler takes
only the payment ID from the body and **re-fetches the payment from Maya**
(`GET /payments/v1/payments/{id}`, secret key). It acts only on what that call
returns, and only if the fetched amount equals the group's `total_centavos`
and the fetched `requestReferenceNumber` names that group. A forged webhook
can therefore at most trigger a lookup of a real payment — it cannot confirm
anything. The IP allowlist stays as defence in depth, not as the control.

Maya's own docs point the same way: they recommend the Retrieve Payment API to
recover from webhook failures.

### 2. Checkout lifetime is fixed at one hour

PayMongo let us set the checkout session's expiry, and §4.5 derived it as
`hold_duration_minutes − 3` so a checkout could never outlive its hold —
mitigation 1 of the §5.5 late-payment hazard. **Maya Checkout has no expiry
parameter.** A checkout lives one hour from creation (and 15 minutes from the
start of authentication), and Maya documents no API to cancel an unpaid one.

So a player can pay up to ~45 minutes after their 15-minute hold has lapsed.

**Decision: keep the 15-minute hold; demote §5.5 mitigation 1; strengthen
mitigation 2.** Stretching the hold to an hour was rejected: with three holds
allowed per player, abandoned checkouts would block popular slots for an hour
each, which reopens exactly the denial-of-inventory hole the hold cap exists
to close.

What changes instead:

- **Late payment is no longer rare; it is an expected path.** The
  re-acquisition attempt (§5.5 mitigation 2) is the main defence, and it
  succeeds whenever the slot was not taken in the meantime — the common case,
  because hold cleanup is lazy and an uncontested slot keeps its row.
- **Re-acquisition must also refuse a slot that has already started.**
  `min_lead_minutes ≥ hold_duration_minutes` no longer guarantees the slot is
  still in the future when payment lands, because payment can now land up to
  an hour after the hold was created. A late payment for a started slot goes
  to the resolution queue.
- **The resolution queue gains a cheaper remedy.** Maya supports **voiding** a
  payment the same day (before 23:59 Manila), which is cleaner than a refund.
  Void and refund must be enabled on the merchant account by the Maya
  relationship manager — request both during onboarding.
- **The frontend hold countdown matters more.** When the hold lapses, the
  checkout screen must say plainly that paying now may not secure the court.

### 3. Amounts are decimal pesos, not centavos

Maya takes `totalAmount.value` as a decimal number of pesos (e.g. `700.00`)
plus `currency: "PHP"`. **The system stays integer centavos everywhere.**
Conversion happens in exactly one place — the `MayaProvider` adapter — via
`Decimal(amount_centavos) / 100`, never through a float, and the reverse
conversion when reading a fetched payment back checks that it is a whole
number of centavos.

## Other adapter-level differences

- **Checkout ID is the payment ID.** Maya's `checkoutId` serves as the
  `paymentId` for Retrieve Payment, so `payments.provider_checkout_id` and
  `provider_payment_id` hold the same value for Maya.
- **Two keys, both server-side.** Create Checkout authenticates with the
  **public** key; Retrieve Payment and webhook registration use the **secret**
  key. Both are HTTP Basic (`key:` base64-encoded). "Public" is Maya's name,
  not a licence to ship it: checkouts are created by the backend, so both live
  in backend env only. There is **no webhook secret** — Maya has none.
- **Webhook events:** subscribe to `PAYMENT_SUCCESS`, `PAYMENT_FAILED`,
  `PAYMENT_EXPIRED`, `PAYMENT_CANCELLED`. The legacy `CHECKOUT_*` events are
  deprecated and must not be used.
- **Webhook events have no event ID.** The idempotency key stored in
  `webhook_events.provider_event_id` is `"{payment_id}:{payment_status}"`.
  Each payment reaches each terminal status at most once, so the pair is
  unique per real event, and redeliveries collide on the unique index exactly
  as before.
- **Retries:** four attempts in total — immediately, then 5, 15, and 45
  minutes after each failure — then the event is dropped. That is a shorter
  tail than PayMongo's and one more reason the keep-alive ping (§9.3) is
  load-bearing: a backend that sleeps through all four attempts depends on
  the re-fetch from the return page to confirm the booking at all.
- **Redirects:** `redirectUrl.success`, `.failure`, `.cancel`. All three point
  at the frontend's `checkout/return` page; none of them confirms anything.
- **Base URLs:** sandbox `https://pg-sandbox.paymaya.com`, production
  `https://pg.paymaya.com`.

## Consequences for the design

### The port

```python
class PaymentProvider(Protocol):
    async def create_checkout(self, *, amount_centavos: int, reference: str,
                              description: str) -> CheckoutSession: ...
    def parse_webhook(self, payload: bytes) -> WebhookNotice: ...      # untrusted
    async def fetch_payment(self, payment_id: str) -> ProviderPayment: ...  # trusted
```

`verify_webhook(payload, signature)` is gone. `parse_webhook` extracts a
payment ID and nothing it says is believed; `fetch_payment` is the authority.
`MockProvider` keeps payment state in memory: its "simulate payment" endpoint
records the outcome and posts a webhook to itself, and its `fetch_payment`
returns what was recorded — so the mock exercises the same
notify-then-fetch path as Maya.

### The webhook route's exemptions (§7.6)

- **No JWT** — unchanged.
- **No rate limiting** — unchanged, and more important: Maya gives up after
  four attempts.
- **Raw body preservation** — no longer needed for a signature, but kept: the
  stored payload must be the bytes Maya sent.
- **New: source IP allowlist**, reading the client IP from the correct
  `X-Forwarded-For` hop for Render. Configurable and disabled under the mock.

### Return page

The return page, when it finds the group still `PENDING`, may ask the backend
to re-fetch the payment from Maya (rate-limited, idempotent through the same
forward-only transitions). This closes the gap when every webhook attempt
landed on a sleeping backend.

### Schema

- `payment_provider`: `PAYMONGO` → `MAYA`.
- `payment_method`: `PAYMONGO` → **`ONLINE`**. A payment *method* was never a
  provider name; naming it after the provider is what made this swap touch the
  enum at all.
- No migration exists yet that carries either enum, so no data changes.

### Unchanged

The two status axes, the hold-then-confirm flow, the convenience fee as a
single tunable percentage, idempotency by unique insert, forward-only
transitions, and "the webhook — never the browser redirect — confirms a
booking". The fee percentage default (`3.00`) stays until Maya's actual rates
for this merchant are known; it is a parameter precisely so that can change
without a deploy.

## Still to confirm with Maya (before Phase 5)

- That this merchant account has **QR Ph** enabled on Maya Checkout. Players
  are expected to pay with GCash, which reaches Maya Checkout through QR Ph
  rather than as a named wallet.
- The merchant discount rates per method.
- That **void and refund** are enabled.
- Whether Maya has added webhook signing since 2026-10-21; if so, verify the
  signature *in addition to* re-fetching, never instead of it.

**Live mode still requires a registered PH business** (DTI/SEC + BIR) and Maya
merchant onboarding. That remains the long pole, and no code shortens it.

## Sources

- [Maya Checkout — Create Checkout](https://developers.maya.ph/reference/createv1checkout)
- [Configuring your webhook for Maya Checkout](https://developers.maya.ph/reference/configuring-your-webhook-for-maya-checkout)
- [Understanding payment statuses in Maya Checkout](https://developers.maya.ph/reference/understanding-payment-statuses-in-maya-checkout)
- [Managing voids and refunds in Maya Checkout](https://developers.maya.ph/reference/managing-voids-and-refunds-in-maya-checkout)
- [Maya developer forum — webhook signature authentication](https://dev-forum.maya.ph/t/webhook-signature-authentication/325)
