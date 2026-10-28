# Customer login demo — design

**Date:** 2026-10-28 · **Status:** approved · **Amends:** the customer booking
demo ([design](2026-10-27-customer-booking-demo-design.md),
[plan](2026-10-28-customer-booking-demo.md))

## Intent

The customer booking demo (`/book`) currently shares the staff console's door:
one `/login` page, after which `landingPath` sends customers to `/book` and
staff to `/staff/schedule`. The demo stands in for the mobile app, and the app
will have its own sign-in. The web should show the same split: **a customer
demo has its own login page, and the official login is the staff console's.**

## Goals

1. A new page, **`/customer-login-demo`**, clearly marked as the customer demo
   entrance. Only accounts in the **Customer group** get through; they land on
   `/book` (or the `?next=` they were headed to).
2. Staff-only and admin-only accounts are **refused there before any session
   exists**. They get a clear "this sign-in is for customers" message and stay
   signed out.
3. **A dual account** (customer + staff, or owner who is also a customer)
   **signs in there as a customer** and goes to `/book`. *(User's decision.)*
4. The refusal is **enforced by the server**: a separate endpoint issues no
   tokens to non-customers, so editing the page can't get around it.
5. `/book` points at its own door. A signed-out visit to `/book/*` redirects to
   `/customer-login-demo?next=…`, and "Sign out" in the booking shell returns
   there.
6. **`/login` becomes the staff door.** *(Claude's call: the user approved the
   playback without settling this point.)* A customer-only account is refused
   there and pointed to `/customer-login-demo`. A dual account signing in at
   `/login` goes to the console.

## Non-goals

- No token "portal" claim. Once issued, a session is an ordinary session. The
  console stays guarded by per-venue permissions on the server, as it is now;
  this change is about which door you enter by, not about narrowing what a
  token can do.
- No customer registration page. Demo customers still come from
  `demo_seed --customers N`.
- No change to `POST /auth/login`. The e2e fixtures and any future client keep
  using it unchanged.
- No change to the booking flow itself.

## Anti-goals

- A staff account holding a live session after being "refused" on the customer
  page. Refuse before the session exists, not after.
- Deciding customer-ness by group **name** or by parsing anything client-side.
  The server's `is_customer` (Customer-group membership) is the only test.
- A refusal that leaks account state to someone without the password. The
  customer check comes **after** the existing credential → verified → active
  checks.

## Design

### Backend: `POST /api/v1/auth/customer-login`

- Same body (`LoginRequest`), same response (`TokenPair`), same rate limit
  (`LOGIN_LIMIT`) as `/auth/login`. It is a separate endpoint, not a flag on
  the login body, following the rule that privileged or narrowed operations get
  their own endpoint.
- Service: the existing login checks run unchanged and in the same order
  (`INVALID_CREDENTIALS` 401 → `EMAIL_NOT_VERIFIED` 403 → `ACCOUNT_DISABLED`
  403). Then `group_repository.is_customer(session, user.id)`; if false, raise
  **`403 NOT_A_CUSTOMER`** "This sign-in is for customers only." **No tokens
  issued and `last_login_at` untouched.**
- `system_admin` gets **no bypass** here. An owner outside the Customer group
  is refused, just as `require_customer` refuses them at `/checkouts/me`.
- Factor the shared credential checks into one private helper that both
  `login` and `customer_login` call, so the timing-safe order cannot drift
  between them.

### Frontend

- **Page** `src/app/(auth)/customer-login-demo/page.tsx`, inside `(auth)` so it
  gets `AuthShell` + `GuestGate`. It has a heading, a visible "Demo" marker, a
  subtitle saying it stands in for the mobile app, and a link "Venue staff? Sign
  in here" → `/login`. `/login` gets the reverse link to the customer demo.
- **`authApi.customerLogin`** wraps the new endpoint.
- **`AuthProvider.login(input, portal)`** with `portal: 'staff' | 'customer'`:
  - `customer`: calls `customerLogin`. The server has already refused
    non-customers.
  - `staff`: calls `login`, then, **before setting `status = authenticated`**,
    fetches permissions. If `!isStaff(permissions)`, it calls
    `authApi.logout()`, clears the tokens and throws a `NOT_STAFF` error
    ("This sign-in is for venue staff. Customers use the customer demo
    sign-in."). It must not set state first: `GuestGate` reacts to
    `authenticated` and would redirect the user to `/book` before the form
    could undo it. This check is a UX door, not a security boundary, because the
    console is already permission-guarded server-side. So client-side is
    acceptable here, unlike goal 4.
- **`LoginForm` gets a `portal` prop** (default `'staff'`). It picks the
  landing (`customer` → `next ?? '/book'`; `staff` → `next ?? '/staff/schedule'`)
  and renders `NOT_A_CUSTOMER` / `NOT_STAFF` as an error alert with a link to
  the other door.
- **`landingPath`** (used by `/` and `GuestGate`) becomes: staff →
  `/staff/schedule`, else `/book`. A dual account at `/` therefore lands on the
  console.
- **`GuestGate` is path-aware.** On `/customer-login-demo`, a signed-in customer
  goes to `/book`; anyone else falls back to `landingPath`. This also stops it
  racing `LoginForm`'s own redirect for a dual account.
- **`AuthGate`**: a signed-out visitor on a `/book` path goes to
  `/customer-login-demo?next=…`; everything else goes to `/login?next=…` as now.
- **`SignOutButton`** (booking shell) → `/customer-login-demo`.
- **`VerifyEmail`**'s "Sign in" link → `/customer-login-demo` *(Claude's call:
  only self-registered accounts verify email, and those are customers; staff
  come in by invitation)*. Forgot/reset password and accept-invite keep
  pointing at `/login`, and the cross-link on `/login` covers a customer who
  lands there.

### Tests

- **Backend** (`tests/auth/`): customer → 200 + token pair; staff-only → 403
  `NOT_A_CUSTOMER` with no refresh-token row written; super-admin outside the
  Customer group → 403; dual account → 200; wrong password for a staff account
  → 401 `INVALID_CREDENTIALS` (order preserved); unverified customer → 403
  `EMAIL_NOT_VERIFIED`.
- **Frontend unit:** `landingPath` table; `AuthProvider` staff portal refusal
  never sets `authenticated` and calls logout.
- **E2E** (`e2e/customer-login.spec.ts`): customer signs in at the demo page →
  `/book`; front desk refused at the demo page and still signed out; customer
  refused at `/login`; signed-out `/book` → `/customer-login-demo`; sign out
  from `/book` → `/customer-login-demo`. Existing `shell.spec.ts` sign-in must
  still pass.

## Left to the builder

Exact copy and styling of the "Demo" marker; the helper's name; whether
`portal` is a string union or two named functions, as long as the call sites
read clearly.

## Docs to update

Root `CLAUDE.md` (current state: the demo has its own door), the frontend
`CLAUDE.md` route list, and the backend `CLAUDE.md` module table
(`/auth/customer-login`).
