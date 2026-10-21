# Phase 6 — Frontend Foundation & Auth Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A `pikol-frontend` repo where a visitor can register, verify their
email, log in, stay logged in across reloads, reset a password, and accept a
staff invitation — and lands on the right home for their permissions.

**Architecture:** asima-frontend's shape, adapted: Next.js App Router with
route groups `(auth)` and `(app)`; feature slices (`features/<slice>/{api,
schemas,keys,hooks,components,index}`) depending one way on `lib/`;
`lib/api-client` is feature-agnostic and has its token and refresh handler
**injected** by `AuthProvider`; every response is zod-parsed in `api.ts`.

**Tech Stack:** Next.js 15.5, React 19, TypeScript 5.9, TanStack Query 5,
react-hook-form 7 + zod 3, Tailwind 3.4 with hand-written shadcn-style
primitives, sonner, lucide-react, Vitest 2 + Testing Library. Majors match
asima so its patterns transfer unchanged.

**Spec:** design §6.1, §6.6, §7.1, §8, §8.3, §9.4; backend `/api/v1/auth/*`
and `/users/me*` as built in Phase 2.

## Global Constraints

- Field names snake_case end-to-end — zod schemas mirror the API exactly.
- The UI is gated on `GET /users/me/permissions`, **never** on a role parsed client-side.
- A production build **fails** if `NEXT_PUBLIC_API_BASE_URL` is unset or not an absolute URL (§8.3). No localhost fallback outside `next dev`.
- `NEXT_PUBLIC_API_BASE_URL` is the only env var; no secrets in the frontend.
- Dev server on **port 3000** — the backend's CORS allow-list.
- Every request carries `X-Request-ID`.
- Errors render the backend's `detail`; flows branch on `code` (`EMAIL_NOT_VERIFIED`, `RATE_LIMITED`, `INVALID_OR_EXPIRED_LINK`), never on message text.

## Decisions

| Decision | Why |
|---|---|
| Access token in memory; refresh token in `localStorage` (as asima) | Survives reloads with no backend cookie work; the XSS exposure is the known asima trade-off, revisited at Phase 9 |
| Refresh sends `{refresh_token}` in the body | That is the Phase 2 backend contract |
| Login does not return the user; the provider follows it with `GET /users/me` | Phase 2 contract |
| `/` and post-login land by permission: `BOOKING:ReadAny` → `/staff`, else `/venues` | §8: staff to today's schedule, players to venues |
| `/venues` and `/staff` are placeholder pages in this phase | Their content is Phases 7 and 8; the landing logic needs real targets now |
| After register, show "check your email" with a resend button; unverified login shows the same resend | §6.6: nobody stranded by a spam-foldered message |
| Visual language: court green primary on neutral, 6px radius, no gradients | A pickleball product, not the default AI palette |
| `GuestGate` sends signed-in users away from `(auth)` pages | A logged-in user hitting /login should not see a form |

## Review Focus

1. **Ten parallel 401s** fire one refresh, not ten — and a replayed refresh token would log the user out everywhere (the backend revokes the family). Test in Task 2.
2. **Refresh fails** — the user is signed out and sent to /login once, not looped. Test in Task 2/3.
3. **`EMAIL_NOT_VERIFIED` on login** — offers resend instead of a dead end. Test in Task 4.
4. **Validation 422** from FastAPI (`detail` is an array) renders a sentence, not `[object Object]`. Test in Task 2.
5. **Token in the URL** for verify/reset/accept is read once; a used or expired link shows a recovery path. Test in Task 4.

---

### Task 1: Scaffold the repo

Files: `package.json`, `tsconfig.json`, `next.config.ts` (security headers + the build guard), `tailwind.config.ts`, `postcss.config.js`, `.eslintrc.json` (slice-barrel and inline-queryKey rules from asima), `.prettierrc`, `vitest.config.ts`, `tests/setup.ts`, `src/app/{layout,globals.css,page}.tsx`, `.env.example`, `.github/workflows/ci.yml`, `CLAUDE.md`, `README.md`.
- [ ] `npm run lint:ci && npm run format:check && npm run typecheck && npm test && npm run build` green; a build with the env var unset fails with the guard's message. Create `Nacar11/pikol-frontend`, push.

### Task 2: `lib/` — API client, errors, query client

`lib/api-client.ts` (`ApiClient` with injected token + refresh handler, single-flight refresh, one retry, `ApiError(status, body)`), `lib/api-error.ts` (`errorCode(err)`, `errorMessage(err, fallback)` handling `{detail: string}` and FastAPI's `{detail: [{msg}]}`), `lib/request-id.ts`, `lib/query-client.ts`.
- [ ] Tests: bearer + request id attached; 401 → refresh → retried once with the new token; **10 concurrent 401s → exactly one refresh call**; refresh failure → `ApiError(401)`; `skipRefresh` honoured; 204 → `undefined`; `errorMessage` for string detail, array detail, and non-ApiError.

### Task 3: The auth slice

`features/auth/{schemas,api,keys,auth-provider,use-auth,use-permissions,landing,index}.ts(x)`, `components/{auth-gate,guest-gate}.tsx`, `src/app/providers.tsx`, `src/app/(app)/layout.tsx`, `src/app/page.tsx` (client redirect by `landingPath`).
- `landingPath(permissions: string[]): '/staff' | '/venues'`.
- [ ] Tests: `landingPath`; provider bootstraps from a stored refresh token (refresh → me → authenticated); a failed bootstrap clears the token and ends `unauthenticated`; logout clears tokens and cached permissions.

### Task 4: Auth screens

`components/ui/{button,input,label,alert}.tsx`, `components/form/field.tsx`, `features/auth/components/{login-form,register-form,check-email,verify-email,forgot-password-form,reset-password-form,accept-invite-form,resend-verification}.tsx`, routes under `src/app/(auth)/`: `login`, `register`, `verify-email`, `forgot-password`, `reset-password`, `accept-invite`.
- [ ] Tests: login success navigates to the landing path; wrong password shows the backend detail; `EMAIL_NOT_VERIFIED` shows a resend button that calls the resend API; register shows "check your email"; verify-email with a dead link shows the resend path; reset/accept submit the token from the URL and route to /login; password confirmation mismatch blocks submit.

### Task 5: App shell placeholders, docs, CI

`components/layout/app-shell.tsx` (brand, user menu with sign out), `src/app/(app)/venues/page.tsx`, `src/app/(app)/staff/page.tsx` (each states what arrives next).
- [ ] Manual check against the running backend: register → copy link from the backend log → verify → login → reload stays signed in → sign out. Update the parent brief; push; CI green.
