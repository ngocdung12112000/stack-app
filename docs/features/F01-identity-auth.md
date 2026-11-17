# F01 — Identity & Authentication

| Field | Value |
|-------|-------|
| Phase | 1 |
| Service | `identity` |
| Stack | .NET 8, ASP.NET Core Identity (user store only), EF Core + MySQL, Redis, JWT RS256 |
| Frontend | `apps/web` (auth pages), `apps/admin` (login), `middleware.ts` |
| Depends on | Gateway (F17 basic routing) |

## 1. User stories

| ID | As a… | I want to… | So that… |
|----|-------|------------|----------|
| US-01.1 | Guest | register with email + password | I can place orders and track them |
| US-01.2 | Guest | log in and stay logged in for up to 7 days | I don't need to log in every visit |
| US-01.3 | Customer | confirm my email | my account is verified |
| US-01.4 | Customer | reset a forgotten password | I can regain access |
| US-01.5 | Customer | log out (this device / all devices) | my account stays secure |
| US-01.6 | Customer | change my password | I can rotate credentials |
| US-01.7 | Admin | assign roles (Admin, Support) to users | staff can access the back-office |
| US-01.8 | Admin | lock / unlock a user | I can stop abuse |

## 2. Business rules

- BR-01: Email is unique (case-insensitive). Normalise to upper-invariant for lookups.
- BR-02: Password policy: ≥ 8 characters, at least 1 letter and 1 digit. Max 128.
- BR-03: 5 failed logins → locked for 15 minutes (`lockout_end`). The lock is per account. In addition, IP throttling uses the Redis counter `auth:login-attempts:{ip}` (20 per 15 minutes).
- BR-04: Unconfirmed users **can** log in and shop, but a banner reminds them to confirm. They cannot write reviews until confirmed.
- BR-05: Access token TTL is 15 minutes. Refresh token TTL is 7 days (sliding with rotation). The absolute session cap is 30 days per family.
- BR-06: Refresh-token reuse (presenting a token already rotated) → revoke the entire family and return 401. Log a security warning.
- BR-07: Login/forgot-password responses never reveal whether an email exists ("If the account exists, we sent an email").
- BR-08: New users get the `Customer` role. The first admin is created by the Seeder or a CLI.
- BR-09: Locked or deleted users get 401 on refresh. Existing access tokens die within 15 minutes, or immediately if their `jti` is blacklisted.

## 3. Data model

See [03-data-architecture.md §2.1](../03-data-architecture.md): `users`, `roles`, `user_roles`, `role_permissions`, `refresh_tokens`, `email_tokens`.

## 4. API

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/auth/register` | anon | `{email, password, fullName}` → 201 `{userId}`. Sends the confirmation email |
| POST | `/api/v1/auth/login` | anon | `{email, password}` → 200 `{accessToken, refreshToken, expiresIn, user{id,email,fullName,roles}}` |
| POST | `/api/v1/auth/refresh` | anon | `{refreshToken}` → 200 new token pair (rotation) |
| POST | `/api/v1/auth/logout` | user | `{refreshToken, allDevices?:bool}` → 204 |
| POST | `/api/v1/auth/confirm-email` | anon | `{userId, token}` → 204 |
| POST | `/api/v1/auth/resend-confirmation` | user | → 204 |
| POST | `/api/v1/auth/forgot-password` | anon | `{email}` → 202 (always) |
| POST | `/api/v1/auth/reset-password` | anon | `{email, token, newPassword}` → 204. Revokes all refresh tokens |
| POST | `/api/v1/auth/change-password` | user | `{currentPassword, newPassword}` → 204 |
| GET | `/.well-known/jwks.json` | anon | Public signing keys |
| GET | `/api/v1/users/me` | user | Current profile |
| GET | `/api/v1/users/me/sessions` | user | Active refresh-token families (device, IP, last used) |
| DELETE | `/api/v1/users/me/sessions/{familyId}` | user | Revoke one session |
| GET | `/api/v1/admin/users?search=&role=&status=&page=` | `users:read` | Paged list |
| PUT | `/api/v1/admin/users/{id}/roles` | `users:write` | `{roles:[...]}` |
| POST | `/api/v1/admin/users/{id}/lock` / `unlock` | `users:write` | |

Error codes: `IDENTITY_EMAIL_TAKEN` (409), `IDENTITY_INVALID_CREDENTIALS` (401), `IDENTITY_LOCKED_OUT` (423 or 401 with code), `IDENTITY_TOKEN_INVALID` (400), `IDENTITY_REFRESH_REUSED` (401), `IDENTITY_WEAK_PASSWORD` (400).

## 5. Events

| Event | When |
|-------|------|
| `identity.user-registered.v1` | After registration → notification sends the welcome + confirm email. Analytics counts signups |
| `identity.password-reset-requested.v1` | Forgot password → notification sends the reset email (token in payload, never logged) |
| `identity.user-locked.v1` | Admin locks a user |

## 6. Frontend

- Pages: `/login`, `/register`, `/forgot-password`, `/reset-password?token=&email=`, `/verify-email?userId=&token=`, `/account/security`.
- Forms use shadcn `Form` + zod. Server Actions call the API and set cookies (`at`, `rt`) with `cookies().set(...)`.
- `middleware.ts`:
  - Protects `/account/**` and `/checkout/**` → redirects to `/login?returnUrl=`.
  - If `at` expires in under 60 seconds and `rt` exists → calls refresh and rewrites both cookies on the response.
- Header shows the user menu (Avatar + DropdownMenu) or Login/Register buttons.
- Admin app: only users with the `Admin` or `Support` role can log in (checked after login, otherwise "Not authorised").

## 7. Non-functional

- Login p95 < 300ms (PBKDF2 cost is tuned accordingly).
- Signing keys are loaded from a Secret (PEM). Rotate them with a `kid`.
- Metrics: `stackshop_auth_logins_total{outcome}`, `stackshop_auth_refresh_total{outcome}`.

## 8. Acceptance criteria

- AC-1: **Given** a new email, **when** I register, **then** I get 201, a confirmation email appears in Mailpit, and I can log in.
- AC-2: **Given** a registered email, **when** I register again with different case, **then** I get 409 `IDENTITY_EMAIL_TAKEN`.
- AC-3: **Given** 5 wrong passwords, **when** I try the correct one, **then** I get a locked-out error until 15 minutes pass.
- AC-4: **Given** a refresh token R1 rotated to R2, **when** R1 is used again, **then** 401 and R2 is revoked too.
- AC-5: **Given** I'm logged in, **when** I log out, **then** calling an API with the old access token returns 401 (blacklisted `jti`).
- AC-6: **Given** the access token expired, **when** I navigate to `/account`, **then** the middleware silently refreshes and the page renders.
- AC-7: Forgot-password returns the same response for existing and non-existing emails.
- AC-8: Non-staff users cannot access the admin app.

## 9. Implementation tasks

**Backend**
1. Domain: `User` aggregate, `RefreshToken` entity with `Rotate()`, `Revoke()`.
2. EF configuration + migrations. Seed roles and permissions.
3. `PasswordHasher<User>` from ASP.NET Identity (or Argon2id via `Konscious`).
4. `JwtTokenService` (RS256, `kid`, claims). JWKS endpoint.
5. Commands: Register, Login, Refresh, Logout, ConfirmEmail, ForgotPassword, ResetPassword, ChangePassword. FluentValidation for each.
6. Redis: login attempt counter, `jti` blacklist. Gateway middleware checks the blacklist.
7. Outbox → `identity.user-events` (it can be a no-op relay until Kafka exists in Phase 2. Store in the outbox anyway).
8. Admin user endpoints + permission policies.

**Frontend**
9. Auth pages and forms, Server Actions, cookie helpers, `middleware.ts` refresh logic.
10. `useCurrentUser()` server helper (decodes `at` server-side or calls `/users/me` with cache).

**Tests**
11. Unit tests: token rotation and reuse detection, lockout logic.
12. Integration tests: register/login/refresh with Testcontainers MySQL + Redis.
13. Playwright: register → logout → login.

## 10. Learning focus

JWT internals (header/payload/signature, JWKS), refresh-token rotation, password hashing, EF Core basics with MySQL, Redis counters with TTL, Next.js middleware and httpOnly cookies.
