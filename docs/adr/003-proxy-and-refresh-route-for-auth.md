# ADR-003: Next.js 16 `proxy.ts` plus a refresh route handler for session refresh

- Status: Accepted
- Date: 2026-11-25

## Context
The F01 spec put the silent token refresh in `middleware.ts`. Two facts make that unworkable here:
1. Next.js 16 renamed Middleware to Proxy (`proxy.ts`). It is meant for cheap checks, not for slow network calls.
2. docs/05 scopes the `rt` cookie to `Path=/api/auth`, so the browser never sends `rt` to pages such as `/account`. The proxy cannot read it there.

## Decision
- `proxy.ts` only reads the `at` cookie. When `at` is expired or has under 60 seconds left, it redirects to `GET /api/auth/refresh?returnUrl=`.
- That route handler reads `rt`, calls `/auth/refresh`, rotates both cookies and redirects back. Dead sessions clear the cookies and go to `/login`.
- Logout is `POST /api/auth/logout` (same cookie-path reason; Origin checked).
- Prefetch requests are excluded from the proxy. Refresh calls are single-flight per `rt`, so parallel requests do not trip reuse detection (BR-06). A short-lived `rf` cookie stops redirect loops.

## Consequences
- The user sees no login page during a refresh (AC-6 still holds), at the cost of one extra redirect.
- The single-flight cache is per Node process. With several frontend replicas, use sticky sessions or add a short grace window for the previous refresh token in the identity service.
- The identity service sees the BFF's address and user agent (for example "node") in the sessions list, not the browser's, until the BFF forwards them.
