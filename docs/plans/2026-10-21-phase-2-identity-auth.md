# Phase 2 — Identity & Auth Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Players register, verify their email, log in, refresh, log out, and
reset their password; admins and venue managers invite staff, who accept and
set their own password; every venue-owned read takes a mandatory `VenueScope`.

**Architecture:** Thin modules (spec §3.4) — router → service → SQLAlchemy
model, no mapper. `src/users/` owns the identity tables and the self/admin
routers; `src/auth/` owns tokens, credentials, and the request dependencies
(`get_current_user`, `require_permission`, `get_venue_scope`); `src/email/` is
a port with console and in-memory adapters. One migration (`0002_identity`)
creates every identity table, the `venues` table they reference, and the seed
role/permission matrix.

**Tech Stack:** FastAPI, SQLAlchemy 2.0 async, Alembic, Pydantic v2, PyJWT
2.15, argon2-cffi 25.1, slowapi 0.1.10, email-validator 2.3, pytest.

**Spec:** [`2026-09-09-court-booking-system-design.md`](./2026-09-09-court-booking-system-design.md)
§6 and §7.4 (auth/self/admin-staff rows). Schema:
[`../adr/pikol.dbml`](../adr/pikol.dbml) identity group + Appendix A §1 +
Appendix B. Refresh-token rules: asima ADR 0002 (summarised in Task 4).

## Global Constraints

- All routes under `API_PREFIX` (`/api/v1`). Request schemas `extra="forbid"`.
- snake_case end-to-end; no camelCase aliases.
- Access token 15 min; refresh token 7 days, rotated, revocable server-side.
- Native PG enums: `postgresql.ENUM(..., create_type=False)` + explicit `.create(op.get_bind(), checkfirst=True)`.
- `users.email` has **no** column-level unique; uniqueness is `users_email_lower_uq` on `LOWER(email) WHERE deleted_at IS NULL` (Appendix A §1).
- `users.password_hash` is `deferred()` — never loaded by an ordinary read.
- Raw tokens (verification, reset, invitation) exist only inside the email; the DB stores `sha256(token)` hex.
- Manager-facing staff creation fixes `role = VENUE_STAFF` and `venue_id = caller.venue_id`; neither may be read from the body.
- `/{id}` admin routes return **404, not 403**, for rows outside the caller's scope.
- Every repository method touching venue-owned data takes `scope: VenueScope` keyword-only with **no default**.
- Unauthenticated: only the `/auth/*` routes below. Everything else requires a bearer access token.
- `ruff format . && ruff check . && mypy src && pytest` green before every commit.

## Decisions this plan makes (not in the spec)

| Decision | Why |
|---|---|
| `venues` table is created here; its endpoints stay in Phase 3 | `users.venue_id` needs a real FK, and `VenueScope` tests need real venues |
| Error envelope gains `code` (`{detail, code, request_id}`) | §6.6 needs a *distinct* `EMAIL_NOT_VERIFIED` the frontend can branch on |
| Admin invite takes a **role code** (`VENUE_MANAGER`/`VENUE_STAFF`), not `role_id` | Clearer; `SUPER_ADMIN` cannot be invited — only the CLI creates one |
| Manager invite is a separate route, `POST /admin/venue-staff` | §6.4: privilege fields are absent from the schema, not ignored by an `if` |
| Verification and reset links are 24 h and 60 min (code constants) | Security windows, not business tuning; invitation uses `invitation_expiry_days` |
| Issuing a token deletes the user's unused tokens of the same purpose | Only the latest email's link works; "resend" invalidates the old one |
| Forgot-password only issues for users who **have** a password | Otherwise it becomes a back door around a revoked invitation |
| Register with a taken email → `409 EMAIL_TAKEN` | Usability over enumeration; register is rate-limited. Forgot/resend always `202` |
| Password change/reset revokes **all** refresh tokens | Matches logout; the client re-authenticates |
| Email adapters: `console` (dev) and `memory` (tests). A real sender is Phase 9 | No provider chosen yet; nothing in Phase 2–8 needs real delivery |
| Rate limits: login `10/minute`, register/forgot/resend `5/minute`, per client IP | §6.1; in-memory store is correct for one Render instance |

## Review Focus

1. **Email case** — `Jane@X.com` registering when `jane@x.com` exists must 409, and logging in as `JANE@x.com` must work. Test in Task 3 and Task 4.
2. **Refresh replay** — presenting a rotated-away refresh token must revoke *every* token of that user, not just 401. Test in Task 4.
3. **Deactivated user with a live access token** — must 401 on the next request, not ride out 15 minutes. Test in Task 4 (`get_current_user` loads the user every request).
4. **Manager body smuggling** — `role_id`/`role`/`venue_id` in `POST /admin/venue-staff` must 422, and the created row must carry the caller's venue. Test in Task 6.
5. **Cross-venue probing** — a manager `GET`/`PATCH`ing a user at another venue gets 404, identical to a non-existent id. Test in Task 6.

---

## File structure

```
src/
├── auth/
│   ├── controllers/auth.py      /auth/* routes
│   ├── dependencies.py          get_current_user, require_permission, require_super_admin, get_venue_scope
│   ├── dto.py                   request/response schemas for /auth
│   ├── persistence/models.py    RefreshTokenModel, UserTokenModel
│   ├── rate_limit.py            limiter
│   ├── scope.py                 VenueScope
│   ├── security.py              hash/verify password, JWT encode/decode, opaque tokens
│   └── service.py               AuthService
├── email/
│   ├── adapters.py              ConsoleEmailSender, MemoryEmailSender
│   ├── port.py                  EmailMessage, EmailSender, get_email_sender
│   └── templates.py             verification / reset / invitation messages
├── users/
│   ├── cli.py                   create-super-admin
│   ├── controllers/me.py        /users/me*
│   ├── controllers/admin.py     /admin/users*, /admin/venue-staff
│   ├── domain/roles.py          RoleCode, PERMISSIONS, ROLE_PERMISSIONS
│   ├── dto.py
│   ├── persistence/models.py    UserModel, RoleModel, PermissionModel, role_permissions
│   ├── persistence/repository.py UserRepository
│   └── service.py               UserService (self + admin)
├── venues/persistence/models.py VenueModel (table only — Phase 3 adds the rest)
└── database/migrations/versions/0002_identity.py
```

---

### Task 1: Settings, dependencies, and the coded error envelope

**Files:**
- Modify: `pyproject.toml`, `src/config/settings.py`, `src/utils/errors.py`, `.env.example`, `.github/workflows/ci.yml`
- Test: `tests/config/test_settings.py`, `tests/utils/test_errors.py`

**Interfaces:**
- Produces: `Settings.jwt_secret: SecretStr` (required, ≥ 32 chars), `access_token_minutes: int = 15`, `refresh_token_days: int = 7`, `frontend_base_url: str = "http://localhost:3000"` (no trailing slash), `email_provider: Literal["console", "memory"] = "console"`, `rate_limit_enabled: bool = True`.
- Produces: `class AppError(Exception): __init__(self, status: int, code: str, detail: str)`; `ErrorResponse` gains `code: str | None = None`; a handler that renders `AppError` through `_envelope`.

- [ ] **Step 1:** Add runtime deps `pyjwt>=2.15`, `argon2-cffi>=25.1`, `slowapi>=0.1.10`, `email-validator>=2.3`; `uv sync`.
- [ ] **Step 2: failing tests** — `test_jwt_secret_shorter_than_32_chars_fails_at_boot` (`Settings(database_url=..., jwt_secret="short")` raises `ValidationError`); `test_frontend_base_url_rejects_trailing_slash`; `test_app_error_renders_code_and_request_id` (a throwaway route raising `AppError(403, "EMAIL_NOT_VERIFIED", "…")` returns 403 with body keys `{"detail", "code", "request_id"}` and `code == "EMAIL_NOT_VERIFIED"`).
- [ ] **Step 3:** Implement. Add `JWT_SECRET` to `.env.example` (with a `openssl rand -hex 32` comment), the local `.env`, and the CI `env:` block.
- [ ] **Step 4:** Full gate green. Commit `Add auth settings and a coded error envelope`.

### Task 2: Identity schema, `venues`, and the seeded permission matrix

**Files:**
- Create: `src/users/domain/roles.py`, `src/users/persistence/models.py`, `src/auth/persistence/models.py`, `src/venues/persistence/models.py`, `src/database/migrations/versions/0002_identity.py`
- Modify: `src/database/migrations/env.py` (model imports), `src/parameters/persistence/models.py` (FK)
- Test: `tests/users/test_identity_schema.py`

**Interfaces:**
- Produces: `RoleCode(StrEnum)`: `SUPER_ADMIN, VENUE_MANAGER, VENUE_STAFF, PLAYER`. `PERMISSIONS: tuple[str, ...]` — the 11 codes of spec §6.3. `ROLE_PERMISSIONS: dict[RoleCode, frozenset[str]]` exactly per Appendix B (SUPER_ADMIN → empty; `system_admin` bypasses). The migration seeds **from these constants**, so the Python and the database cannot disagree.
- Produces models, columns exactly per `pikol.dbml`: `RoleModel`, `PermissionModel`, `role_permissions` (Table), `UserModel` (`role: Mapped[RoleModel]` joined-eager, `password_hash` deferred), `RefreshTokenModel`, `UserTokenModel`, `VenueModel`. `token_purpose` enum: `EMAIL_VERIFICATION, PASSWORD_RESET, INVITATION`.
- Migration also: `users_email_lower_uq` (Appendix A §1 verbatim), FK `parameters.updated_by_user_id → users.id`, FKs and `ondelete` per the dbml `Ref:` lines. Downgrade drops all of it, including the enum.

- [ ] **Step 1: failing tests** (use `db_session`):
  - `test_seeded_matrix_matches_spec` — query `roles ⨝ role_permissions ⨝ permissions`; assert equals `ROLE_PERMISSIONS`, and `VENUE_MANAGER` has `USER:Manage` but not `PARAMETER:Manage` or `VENUE:Manage`.
  - `test_email_unique_case_insensitively` — insert `jane@x.com` then `JANE@x.com` → `IntegrityError` naming `users_email_lower_uq`.
  - `test_email_reusable_after_soft_delete` — set `deleted_at` on the first, insert the same email → succeeds.
  - `test_password_hash_not_loaded_by_default` — `select(UserModel)` then `"password_hash" not in inspect(user).dict`.
- [ ] **Step 2:** Run → fail (tables missing).
- [ ] **Step 3:** Write models + migration. `uv run alembic upgrade head`, then `downgrade -1` and `upgrade head` again — both must succeed.
- [ ] **Step 4:** Full gate green. Commit `Add the identity schema and seeded role-permission matrix`.

### Task 3: Security primitives, the email port, registration, and verification

**Files:**
- Create: `src/auth/security.py`, `src/email/{port,adapters,templates}.py`, `src/auth/{dto,service}.py`, `src/auth/controllers/auth.py`, `src/users/persistence/repository.py`
- Modify: `src/main.py` (include auth router), `tests/conftest.py` (`outbox` fixture)
- Test: `tests/auth/test_security.py`, `tests/auth/test_register_verify.py`

**Interfaces:**
- `security.py`: `hash_password(raw: str) -> str`, `verify_password(raw: str, hashed: str) -> bool` (argon2id via `argon2.PasswordHasher`; `False` on mismatch, never raises); `new_opaque_token() -> tuple[str, str]` returning `(raw, sha256_hex)` from `secrets.token_urlsafe(32)`; `hash_opaque_token(raw: str) -> str`.
- `port.py`: `@dataclass(frozen=True) EmailMessage(to: str, subject: str, body: str)`; `class EmailSender(Protocol): async def send(self, message: EmailMessage) -> None`; `get_email_sender() -> EmailSender` (FastAPI dependency, chosen by `settings.email_provider`).
- `adapters.py`: `ConsoleEmailSender` logs `to`, `subject`, and `body` at INFO (dev only — the link *is* the point); `MemoryEmailSender.outbox: list[EmailMessage]`.
- `UserRepository.get_by_email(session, email: str) -> UserModel | None` — compares `func.lower(UserModel.email) == email.lower()` and `deleted_at IS NULL`. (Users are not venue-owned for this lookup; scoped methods arrive in Task 6.)
- `AuthService.issue_token(session, user, purpose, ttl: timedelta) -> str` — deletes that user's unused tokens of `purpose`, inserts the hashed one, returns the raw token. `AuthService.consume_token(session, raw, purpose) -> UserModel` — finds by hash + purpose, `used_at IS NULL`, `expires_at > now()`; sets `used_at`; else `AppError(400, "INVALID_OR_EXPIRED_LINK", …)`.
- Routes: `POST /auth/register` `{email: EmailStr, password: str(8..128), first_name, last_name, mobile?}` → 201 `UserOut`; `POST /auth/verify-email` `{token}` → 204; `POST /auth/verify-email/resend` `{email}` → 202 always.
- `UserOut`: `id, email, first_name, last_name, mobile, role: RoleCode, venue_id, is_active, email_verified_at`.
- Email is sent **after** commit; a send failure is logged and swallowed (resend exists).

- [ ] **Step 1: failing tests**
  - security: hash round-trips; wrong password → `False`; two hashes of the same password differ; `new_opaque_token()` raw ≠ hash and `hash_opaque_token(raw) == hash`.
  - `test_register_creates_inactive_player_and_emails_link` — 201, `role == "PLAYER"`, `is_active is False`, `email_verified_at is None`; one message in `outbox` whose body contains `f"{frontend_base_url}/verify-email?token="`.
  - `test_register_rejects_case_variant_of_existing_email` — 409, `code == "EMAIL_TAKEN"`.
  - `test_register_forbids_role_field` — body with `"role": "SUPER_ADMIN"` → 422.
  - `test_verify_email_activates_and_is_single_use` — token from outbox → 204, user active + verified; same token again → 400 `INVALID_OR_EXPIRED_LINK`.
  - `test_expired_verification_link_rejected` — set `expires_at` in the past → 400.
  - `test_resend_invalidates_previous_link_and_hides_unknown_email` — resend → 202, old token → 400, new token → 204; resend for `nobody@x.com` → 202 and outbox unchanged.
- [ ] **Step 2:** Run → fail. **Step 3:** Implement. **Step 4:** Gate green. Commit `Add player registration and email verification`.

### Task 4: Login, refresh rotation, logout, and `get_current_user`

**Files:**
- Modify: `src/auth/{security,service,dto}.py`, `src/auth/controllers/auth.py`
- Create: `src/auth/dependencies.py`
- Test: `tests/auth/test_login_refresh.py`, `tests/auth/helpers.py` (`make_user(session, *, role, venue_id=None, password="correct-horse-1", verified=True, active=True, system_admin=False) -> UserModel` and `login(client, email, password) -> dict`)

**Interfaces:**
- `security.py`: `encode_access(user_id: int) -> str`, `encode_refresh(user_id: int, jti: UUID) -> tuple[str, datetime]` (token, expires_at), `decode(token: str, expected_typ: Literal["access","refresh"]) -> dict[str, Any]` — HS256 with `settings.jwt_secret`; claims `sub` (str id), `typ`, `iat`, `exp`, plus `jti` on refresh. Any failure → `AppError(401, "INVALID_TOKEN", …)`.
- `TokenPair`: `access_token, refresh_token, token_type: Literal["bearer"], expires_in: int` (seconds).
- `POST /auth/login` `{email, password}` → 200 `TokenPair`; stamps `last_login_at`. Order of checks: unknown email / no password / wrong password → 401 `INVALID_CREDENTIALS` (same message for all three); then `email_verified_at is None` → 403 `EMAIL_NOT_VERIFIED`; then `not is_active` → 403 `ACCOUNT_DISABLED`. Status leaks only after a correct password.
- `POST /auth/refresh` `{refresh_token}` → 200 `TokenPair`. Rotation is the conditional `UPDATE refresh_tokens SET revoked_at = now() WHERE jti = :jti AND revoked_at IS NULL AND expires_at > now()`. Zero rows: if the row exists **and is revoked** → revoke every row for that user (family revocation), then 401; unknown/expired → plain 401. Also 401 if the user is now inactive or deleted.
- `POST /auth/logout` (auth) → 204; revokes all the caller's active refresh tokens.
- `dependencies.py`: `get_current_user(credentials = Security(HTTPBearer(auto_error=False)), session) -> UserModel` — decode access, load the user each request; missing/invalid/deleted/inactive → 401 `INVALID_TOKEN`.

- [ ] **Step 1: failing tests**
  - `test_login_returns_working_access_token` — the token reaches a probe route depending on `get_current_user` (use `GET /users/me` once Task 5 lands; until then a test-local router).
  - `test_login_is_case_insensitive_on_email`.
  - `test_wrong_password_and_unknown_email_are_indistinguishable` — same status, code, and detail.
  - `test_unverified_player_gets_distinct_error_after_correct_password` — 403 `EMAIL_NOT_VERIFIED`; with a wrong password it is 401 `INVALID_CREDENTIALS`.
  - `test_refresh_rotates_and_old_token_stops_working`.
  - `test_refresh_replay_revokes_the_whole_family` — log in twice (two families' worth of rows), rotate token A once, present A again → 401, and **every** `refresh_tokens` row for the user has `revoked_at` set.
  - `test_access_token_cannot_be_used_as_refresh` and the reverse.
  - `test_logout_revokes_all_refresh_tokens`.
  - `test_deactivated_user_rejected_on_next_request` — valid access token, flip `is_active=False`, request → 401.
- [ ] **Step 2–4:** Fail, implement, gate green. Commit `Add login, refresh rotation with replay detection, and logout`.

### Task 5: Self-service — `/users/me`, permissions, password change and reset

**Files:**
- Create: `src/users/{dto,service}.py`, `src/users/controllers/me.py`
- Modify: `src/auth/{dependencies,service,dto}.py`, `src/auth/controllers/auth.py`, `src/main.py`
- Test: `tests/users/test_me.py`, `tests/auth/test_password_reset.py`

**Interfaces:**
- `dependencies.py`: `require_permission(code: str) -> Callable[..., Awaitable[UserModel]]` — `system_admin` short-circuits; else 403 `FORBIDDEN` unless the role holds `code`. `require_super_admin` — 403 unless `system_admin`. `UserRepository.permission_codes(session, user) -> list[str]` — all `PERMISSIONS` for `system_admin`, else the role's codes, sorted.
- Routes: `GET /users/me` → `UserOut`; `PATCH /users/me` `{first_name?, last_name?, mobile?}` → `UserOut`; `PATCH /users/me/password` `{current_password, new_password}` → 204 (wrong current → 400 `WRONG_PASSWORD`), revokes all refresh tokens; `GET /users/me/permissions` → `list[str]`.
- `POST /auth/password/forgot` `{email}` → 202 always; issues `PASSWORD_RESET` (60 min) only if the user exists, is not deleted, and has a `password_hash`. `POST /auth/password/reset` `{token, new_password}` → 204; sets the hash, revokes all refresh tokens.

- [ ] **Step 1: failing tests**
  - `test_me_requires_auth` (401 without a token).
  - `test_patch_me_forbids_email_and_role` — 422.
  - `test_permissions_super_admin_gets_every_code` (equals sorted `PERMISSIONS`), `test_permissions_staff_matches_matrix`, `test_permissions_player_is_empty`.
  - `test_change_password_requires_current_and_revokes_sessions` — old refresh token → 401 afterwards; new password logs in.
  - `test_forgot_password_is_silent_for_unknown_email` — 202, empty outbox.
  - `test_forgot_password_ignores_pending_invitee` — invited user with `password_hash = NULL` → 202, empty outbox.
  - `test_reset_password_sets_new_password_and_is_single_use`.
- [ ] **Step 2–4:** Fail, implement, gate green. Commit `Add self-service profile, permissions, and password reset`.

### Task 6: `VenueScope`, staff invitations, and the admin user routes

**Files:**
- Create: `src/auth/scope.py`, `src/users/controllers/admin.py`
- Modify: `src/users/{dto,service}.py`, `src/users/persistence/repository.py`, `src/auth/{service,dto}.py`, `src/auth/controllers/auth.py`, `src/auth/dependencies.py`, `src/main.py`
- Test: `tests/auth/test_venue_scope.py`, `tests/users/test_admin_users.py`, `tests/auth/test_invitation_accept.py`

**Interfaces:**
- `scope.py`: `@dataclass(frozen=True) VenueScope(all_venues: bool, venue_id: int | None)` with `where(column: ColumnElement[int | None]) -> ColumnElement[bool]` — `true()` for `all_venues`, else `column == venue_id`. `get_venue_scope(user = Depends(get_current_user)) -> VenueScope` — `system_admin` → all; `venue_id` set → that venue; otherwise 403 `FORBIDDEN`.
- `UserRepository.list(session, *, scope: VenueScope, page: int, limit: int) -> tuple[list[UserModel], int]` and `UserRepository.get_scoped(session, user_id: int, *, scope: VenueScope) -> UserModel | None`. **No default for `scope`.**
- Routes, all under `/admin`:
  - `POST /admin/users` (`require_super_admin`) `{email, first_name, last_name, mobile?, role: Literal["VENUE_MANAGER","VENUE_STAFF"], venue_id: int}` → 201 `UserOut`; unknown `venue_id` → 422 `UNKNOWN_VENUE`.
  - `POST /admin/venue-staff` (`USER:Manage`, caller must have a `venue_id`, else 403) `{email, first_name, last_name, mobile?}` → 201 `UserOut` with `role = VENUE_STAFF`, `venue_id = caller.venue_id`.
  - Both create `password_hash = NULL`, `is_active = False`, and email an `INVITATION` token valid `invitation_expiry_days` (from `parameter_service`) linking to `/accept-invite?token=`. Taken email → 409 `EMAIL_TAKEN`.
  - `GET /admin/users?page=1&limit=20` (`USER:Manage` + scope) → `Page[UserOut]`, ordered by `id`.
  - `GET /admin/users/{id}`, `PATCH /admin/users/{id}` `{first_name?, last_name?, mobile?, is_active?}` (`USER:Manage` + scope) — out of scope → 404 `NOT_FOUND`. A non-super-admin may modify only `VENUE_STAFF` → 403 `FORBIDDEN`. Nobody may modify themselves here → 403 (use `/users/me`). Setting `is_active=False` revokes the target's refresh tokens.
  - `POST /admin/users/{id}/invitation` → 202 resend (fresh token, old one dead); `DELETE /admin/users/{id}/invitation` → 204 revoke. Both 409 `ALREADY_ACCEPTED` once the user has a password. Same scope/role rules as PATCH.
  - `POST /auth/invitations/accept` `{token, password}` → 204: sets the password, `is_active = True`, `email_verified_at = now()`.

- [ ] **Step 1: failing tests**
  - `test_scope_is_mandatory` — `await repo.list(session, page=1, limit=20)` raises `TypeError`.
  - `test_player_has_no_venue_scope` — 403 on `GET /admin/users`.
  - `test_manager_lists_only_own_venue` — users at venues A and B; manager A sees only A's.
  - `test_manager_cannot_see_other_venue_user_by_id` — 404, and the body equals a request for id `999999`.
  - `test_manager_invite_rejects_privilege_fields` — each of `{"role_id": 1}`, `{"role": "SUPER_ADMIN"}`, `{"venue_id": B}` added to the body → 422.
  - `test_manager_invite_pins_role_and_venue` — created user is `VENUE_STAFF` at the manager's venue, inactive, no password; outbox has an `/accept-invite?token=` link.
  - `test_staff_cannot_invite` — `VENUE_STAFF` → 403.
  - `test_admin_invites_manager_to_any_venue` and `test_admin_invite_unknown_venue_is_422`.
  - `test_manager_cannot_modify_another_manager` → 403; `test_cannot_modify_self_via_admin_route` → 403.
  - `test_deactivating_staff_kills_their_sessions` — their refresh → 401 and access → 401.
  - `test_accept_invitation_activates_and_logs_in`; `test_revoked_invitation_link_is_dead`; `test_resend_after_accept_is_409`.
- [ ] **Step 2–4:** Fail, implement, gate green. Commit `Add venue scoping, staff invitations, and admin user routes`.

### Task 7: Rate limiting, the super-admin CLI, and docs

**Files:**
- Create: `src/auth/rate_limit.py`, `src/users/cli.py`
- Modify: `src/main.py`, `src/auth/controllers/auth.py`, `tests/conftest.py`, `pikol-backend/CLAUDE.md`, `pikol-backend/README.md`
- Test: `tests/auth/test_rate_limit.py`, `tests/users/test_cli.py`

**Interfaces:**
- `rate_limit.py`: `limiter = Limiter(key_func=get_remote_address, enabled=get_settings().rate_limit_enabled)`; `create_app` sets `app.state.limiter` and registers a 429 handler rendering `AppError`-shaped JSON with `code = "RATE_LIMITED"`. Decorate login `10/minute`; register, `verify-email/resend`, `password/forgot` `5/minute`. Conftest autouse fixture calls `limiter.reset()` so tests never share a bucket.
- `cli.py`: `uv run python -m src.users.cli create-super-admin --email … --first-name … --last-name …` prompts for the password twice (`getpass`), creates `role = SUPER_ADMIN`, `system_admin = True`, active and verified. Refuses (exit 1) if the email exists. Core logic in `async def create_super_admin(session, *, email, password, first_name, last_name) -> UserModel` so it is testable without a TTY.

- [ ] **Step 1: failing tests** — `test_login_is_rate_limited` (11th attempt in a minute → 429 `RATE_LIMITED`); `test_create_super_admin_has_every_permission` (via `GET /users/me/permissions` after login); `test_create_super_admin_refuses_existing_email`.
- [ ] **Step 2–4:** Fail, implement, gate green.
- [ ] **Step 5:** Update `pikol-backend/CLAUDE.md` "What exists now" (Phase 2 modules) and README (create the first admin; `JWT_SECRET`). Note for Phase 9: uvicorn must run with `--proxy-headers --forwarded-allow-ips` behind Render, or every client shares Render's IP and one rate-limit bucket. Commit `Add rate limiting and the super-admin bootstrap CLI`.

## Out of scope (flagged, not built)

- A real email provider and SPF/DKIM (§6.6) — Phase 9, before go-live.
- Email change, account deletion, and admin role/venue reassignment endpoints — no spec row asks for them.
- Pruning expired `refresh_tokens` rows — indexed on `expires_at`; a `pg_cron` hygiene job can do it later.
