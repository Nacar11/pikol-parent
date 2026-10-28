# Customer login demo — Implementation Plan

**Goal:** The customer booking demo gets its own door, `/customer-login-demo`,
which only Customer-group accounts can pass (enforced by a new
`POST /auth/customer-login`). `/login` becomes the staff console's door.

**Spec:** [`2026-10-28-customer-login-demo-design.md`](2026-10-28-customer-login-demo-design.md).
Read it; this plan does not repeat its rationale.

**Repos touched:** `pikol-backend` (Task 1), `pikol-frontend` (Tasks 2–5),
`pikol-parent` (Task 6). Each commits to its own `main` after its checks pass.

## Architecture decisions (from the design)

- **Separate endpoint, not a body flag:** `POST /auth/customer-login` shares
  `login`'s credential → verified → active checks through one private helper,
  then requires `is_customer`. It returns `403 NOT_A_CUSTOMER`, issues no
  tokens and leaves `last_login_at` alone. There is no `system_admin` bypass.
- **The staff door is checked client-side, before state is set:**
  `AuthProvider.login(input, portal)` refuses a non-staff account *before*
  `status = authenticated`, so `GuestGate` cannot redirect first. This is a UX
  door, not a security boundary: the console stays permission-guarded on the
  server.
- **No token portal claim; `/auth/login` unchanged.**

## Dependency graph

```
T1 backend endpoint ──► T2 frontend API + AuthProvider portals ──► T3 customer page + LoginForm portal
                                                                  │
                                                                  └─► T4 routing (landingPath, GuestGate, AuthGate, sign-out, verify link)
                                                                            │
                                                                            └─► T5 e2e (real stack) ──► T6 docs + snapshot
```

T1 must land first: T2's customer portal calls the real endpoint, and T5 runs
the real stack.

## Task list

### Phase 1: The server refuses non-customers

#### Task 1: `POST /auth/customer-login`
**Description:** Pull `login`'s credential checks into a private helper. Add
`auth_service.customer_login` (helper → `group_repository.is_customer` → issue
pair → commit) and the route in `src/auth/controllers/auth.py`, with the same
`LoginRequest`/`TokenPair` and `@limiter.limit(LOGIN_LIMIT)`.

**Acceptance criteria:**
- [x] Customer → 200 with a token pair; a dual (customer + venue staff) account → 200.
- [x] Staff-only, and a super-admin outside the Customer group → 403
      `NOT_A_CUSTOMER`, with **no `refresh_tokens` row** and `last_login_at` unchanged.
- [x] The order holds: a wrong password for a staff account → 401
      `INVALID_CREDENTIALS`; an unverified customer → 403 `EMAIL_NOT_VERIFIED`;
      `/auth/login` behaviour unchanged (its existing tests pass untouched).

**Verification:** `uv run ruff format . && uv run ruff check . && uv run mypy src && uv run pytest && uv run vulture`

**Files:** `src/auth/service.py`, `src/auth/controllers/auth.py`,
`tests/auth/test_customer_login.py` (new, using `helpers.make_user`/`login`)

**Scope:** S

### Checkpoint A: backend
- [x] All backend gates green; commit + push `pikol-backend`.

### Phase 2: The web has two doors

#### Task 2: Portal-aware sign-in in `AuthProvider`
**Description:** `authApi.customerLogin` (zod-parsed, `skipRefresh`). Change
`AuthProvider.login(input, portal: 'staff' | 'customer')`. The customer portal
calls `customerLogin`. The staff portal calls `login`, then sets the access
token, fetches `me` and permissions, and if `!isStaff` → `authApi.logout()`,
clears tokens and throws an error carrying code `NOT_STAFF`, **without ever
setting `authenticated`**. Make `errorCode`/`errorMessage` (`src/lib/api-error`)
read that error. Update the one existing caller (`LoginForm`) to pass `'staff'`.

**Acceptance criteria:**
- [x] Staff portal with a customer-only account: `logout` called, the stored
      refresh token cleared, status stays `unauthenticated`, `NOT_STAFF` thrown.
- [x] Staff portal with staff or a dual account and the customer portal with a
      customer: signed in exactly as before.
- [x] `NOT_A_CUSTOMER` from the server leaves status `unauthenticated` and stores nothing.

**Verification:** `npm test -- auth` · `npm run typecheck` · `npm run lint:ci`

**Files:** `src/features/auth/api.ts`, `src/features/auth/auth-provider.tsx`,
`src/lib/api-error.ts` (if needed), `src/features/auth/components/login-form.tsx`,
`tests/unit/auth/auth-provider.test.tsx`

**Scope:** M

#### Task 3: `/customer-login-demo` page and `LoginForm` portal
**Description:** `LoginForm` gains `portal` (default `'staff'`). It lands on
`next ?? '/book'` (customer) or `next ?? '/staff/schedule'` (staff), and on
`NOT_A_CUSTOMER`/`NOT_STAFF` shows the error with a link to the other door.
Add the `(auth)/customer-login-demo/page.tsx` page with a "Demo" marker, a
subtitle saying it stands in for the mobile app, and a "Venue staff? Sign in
here" link. `/login` gets the reverse link.

**Acceptance criteria:**
- [x] `/customer-login-demo` renders inside `AuthShell` with the demo marker
      and the link to `/login`, and `/login` links back.
- [x] The refusal alert names the other door and links to it.
- [x] `/login` sign-in for staff still lands on the console.

**Verification:** `npm test -- auth` (extend `screens.test.tsx`) · `npm run typecheck` · `npm run build` via `next start -p 3100` (never over the `:3000` dev server)

**Files:** `src/app/(auth)/customer-login-demo/page.tsx` (new),
`src/app/(auth)/login/page.tsx`, `src/features/auth/components/login-form.tsx`,
`tests/unit/auth/screens.test.tsx`

**Scope:** M

#### Task 4: Route each side to its own door
**Description:** `landingPath(permissions)`: staff → `/staff/schedule`, else
`/book`. Drop the `isCustomer` argument and update `/` and `GuestGate`.
`GuestGate` becomes path-aware: on `/customer-login-demo`, a signed-in
customer → `/book`. `AuthGate`: signed out on `/book*` →
`/customer-login-demo?next=…`. The booking `SignOutButton` →
`/customer-login-demo`. `VerifyEmail`'s sign-in link → `/customer-login-demo`.

**Acceptance criteria:**
- [x] `landingPath` table test: staff, dual and owner → console; customer-only
      and no-group → `/book`.
- [x] `AuthGate` on `/book/mine` while signed out → `/customer-login-demo?next=%2Fbook%2Fmine`;
      on `/staff/courts` → `/login?next=…` as before.
- [x] Signing out of the booking shell lands on `/customer-login-demo`; a
      signed-in dual account visiting `/customer-login-demo` goes to `/book`.

**Verification:** `npm test` · `npm run typecheck` · `npm run lint:ci` · `npm run deadcode`

**Files:** `src/features/auth/landing.ts`, `src/features/auth/components/guest-gate.tsx`,
`src/features/auth/components/auth-gate.tsx`, `src/app/page.tsx`,
`src/features/booking/components/sign-out-button.tsx`,
`src/features/auth/components/verify-email.tsx`, `tests/unit/auth/landing.test.ts`
(+ a gate test). This is 7 small edits of 1–3 lines each, so it stays one task:
splitting it would leave a half-routed app between commits.

**Scope:** M

### Checkpoint B: frontend unit-level
- [x] `npm test`, `typecheck`, `lint:ci`, `format:check`, `deadcode` green.

#### Task 5: E2E on the real stack
**Description:** `e2e/customer-login.spec.ts`, using the seeded accounts
(`customer1.e2e`, `desk.e2e`).

**Acceptance criteria:**
- [x] A customer signs in at `/customer-login-demo` → `/book`; signing out → `/customer-login-demo`.
- [x] The front desk is refused at `/customer-login-demo` (alert shown, still on
      the page, a reload shows the form, not a session); a customer is refused at `/login`.
- [x] Signed-out `/book` → `/customer-login-demo?next=…`; the existing
      `shell.spec.ts` "signing in lands on the schedule" still passes.

**Verification:** `npm run e2e` (full suite, not just the new file), backend on
:8100 per the e2e script. Also check by hand in a browser at `next start -p 3100`.

**Files:** `e2e/customer-login.spec.ts` (new)

**Scope:** S

### Checkpoint C: frontend complete
- [x] All frontend gates plus the full e2e suite green; commit + push `pikol-frontend`.

### Phase 3: Record it

#### Task 6: Docs and plan snapshot
**Description:** Root `CLAUDE.md` current state: the demo enters at
`/customer-login-demo`, and `/login` is the staff door. Frontend `CLAUDE.md`:
the routes list and the landing bullet, which currently says "a customer →
`/book`, even one who is also staff" and is now wrong. Backend `CLAUDE.md`:
`/auth/customer-login` in the `src/auth/service.py` row. Save this plan to
`docs/plans/2026-10-28-customer-login-demo.md` with its checkboxes ticked.

**Acceptance criteria:**
- [x] No doc still describes customers landing via `/login`.
- [x] The snapshot is committed in `pikol-parent`; the backend and frontend doc edits are committed in their own repos.

**Verification:** `grep -rn "even one who is also staff" */CLAUDE.md CLAUDE.md` returns nothing.

**Files:** `CLAUDE.md`, `pikol-frontend/CLAUDE.md`, `pikol-backend/CLAUDE.md`,
`docs/plans/2026-10-28-customer-login-demo.md`

**Scope:** S

### Checkpoint D: done
- [x] Every acceptance criterion in the design's Goals 1–6 is met; all three repos pushed.

## Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| `GuestGate` redirects before the staff-portal refusal runs | High: a customer ends up signed in at `/book` from `/login` | Refuse inside `AuthProvider.login` before setting state (Task 2); e2e asserts no session survives (Task 5) |
| The shared helper reorders credential checks | Med: account-probing leak | Task 1 pins the order with tests on both endpoints |
| Staff-portal `logout` revokes the account's *other* sessions (`/auth/logout` revokes all) | Low: only a customer-only account reaches this path, and its other sessions are demo tabs that must sign in again | Accepted: revoking is safer than leaving an orphaned refresh token alive |
| `npm run build` clobbers the `:3000` dev server | Low | Verify via `next start -p 3100` (memory) |

## Open questions

None. The user approved the design; Claude's calls are marked in it.

## What the build found

- **`GuestGate` dropped `?next=`.** After sign-in, `GuestGate` and `LoginForm`
  both redirect, and the gate won, so a signed-out visit to `/book/mine` ended
  on `/book`. The e2e spec caught it. Both now honour a safe `?next=` first
  (`safeNext` moved to `landing.ts`). The gate reads it from `window.location`
  inside its effect, because it sits outside the pages' Suspense boundary.
- `useFetchPermissions` became dead once `AuthProvider` fetched permissions
  for the staff door, so it was removed.
- In e2e, `getByRole('alert')` also matches Next's route announcer, so the
  refusal assertions are scoped to `main`.

Results: backend 456 tests; frontend 308 unit tests; e2e 23/23.
