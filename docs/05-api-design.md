# 05 — API Design & Security

## 1. Conventions

| Topic | Rule |
|-------|------|
| Base URL (public) | `https://api.stackshop.local/api/v1/{service-resource}`. The gateway routes by path prefix |
| Style | REST, plural nouns, kebab-case paths, camelCase JSON |
| IDs | String (ULID/GUID v7) |
| Pagination | Cursor-based for feeds (`?cursor=…&limit=20` → `{items, nextCursor}`). Offset-based for admin tables (`?page=1&pageSize=20` → `{items, page, pageSize, total}`) |
| Sorting | `?sort=price,-createdAt` |
| Filtering | Explicit query params (`?status=Paid&from=2026-11-01`) |
| Dates | ISO-8601 UTC |
| Money | `{ "amount": 1099.00, "currency": "USD" }` or flat `price` + an order-level `currency` |
| Errors | RFC 7807 `application/problem+json` (see §3) |
| Idempotency | `Idempotency-Key` header required on `POST /orders` and `POST /payments/*` |
| Concurrency | `ETag` / `If-Match` on admin PUT for products, stock and orders → `412 Precondition Failed` on mismatch |
| Docs | Each service exposes OpenAPI (`/swagger`). The gateway aggregates them. The backend repo generates the TS client and publishes it as the npm package `@ngocdung12112000/api-client`, which the frontend repo consumes (see [02-architecture.md §9.4](./02-architecture.md)) |
| Compatibility | Frontend and backend are released independently, so a public API change must be **backward compatible for at least one frontend release** (expand → migrate → contract). A breaking change requires a new API version (`/api/v2`) or a coordinated deprecation |

## 2. Gateway routes

| Prefix | Upstream | Auth |
|--------|----------|------|
| `/api/v1/auth/**` | identity | Anonymous (rate-limited strictly) |
| `/api/v1/users/**`, `/api/v1/addresses/**` | identity | Customer |
| `/api/v1/products/**`, `/api/v1/categories/**`, `/api/v1/brands/**` | catalog | GET anonymous. Write needs `catalog:write` |
| `/api/v1/search/**` | search | Anonymous |
| `/api/v1/carts/**` | cart | Anonymous (guest cart cookie) or Customer |
| `/api/v1/orders/**` | ordering | Customer / Admin |
| `/api/v1/payments/**` | payment | Customer. `/webhooks/**` uses HMAC instead of JWT |
| `/api/v1/shipments/**` | shipping | Customer / Admin. `/webhooks/**` uses HMAC |
| `/api/v1/reviews/**` | review | GET anonymous. POST Customer |
| `/api/v1/promotions/**`, `/api/v1/coupons/**` | promotion | Admin (except `POST /coupons/validate`) |
| `/api/v1/media/**` | media | Admin / Customer (review images) |
| `/api/v1/wishlist/**` | catalog | Customer |
| `/api/v1/notifications/**`, `/hubs/notifications` | notification | Customer |
| `/api/v1/inventory/**` | inventory | Admin |
| `/api/v1/analytics/**`, `/api/v1/reports/**` | analytics | `/track` anonymous. Others Admin |

Gateway middleware order: `RequestId → Logging → CORS → RateLimiter → Authentication → Authorization (route policy) → YARP proxy`.

Rate limits (Redis sliding window):

| Policy | Limit |
|--------|-------|
| `anonymous` | 60 req/min per IP |
| `authenticated` | 300 req/min per user |
| `auth-strict` (login/register/forgot) | 5 req/min per IP + per email |
| `checkout` | 10 req/min per user |

## 3. Error format

```json
{
  "type": "https://docs.stackshop.local/errors/insufficient-stock",
  "title": "Insufficient stock",
  "status": 409,
  "detail": "Only 2 units of SKU MBA-M3-SLV-8 are available.",
  "instance": "/api/v1/orders",
  "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "errors": { "lines[0].quantity": ["Requested 3, available 2"] },
  "code": "INVENTORY_INSUFFICIENT_STOCK"
}
```

| HTTP | When |
|------|------|
| 400 | Validation failed (FluentValidation → `errors` dictionary) |
| 401 | Missing or invalid token |
| 403 | Authenticated but lacks the permission |
| 404 | Resource not found **or not owned** by the caller (don't leak existence) |
| 409 | Business conflict (stock, duplicate review, state transition not allowed) |
| 412 | ETag mismatch |
| 422 | Semantically invalid (coupon expired) |
| 429 | Rate limited (`Retry-After` header) |
| 500/503 | Unexpected / dependency down (no internal details exposed) |

Error codes are `SCREAMING_SNAKE`, prefixed by service, and listed in each feature spec.

## 4. Authentication flow

```mermaid
sequenceDiagram
    participant B as Browser
    participant N as Next.js (BFF)
    participant G as Gateway
    participant I as Identity

    B->>N: POST /login (Server Action) {email, password}
    N->>G: POST /api/v1/auth/login
    G->>I: forward
    I-->>N: {accessToken (15m), refreshToken (7d), expiresIn}
    N-->>B: Set-Cookie: at=…; HttpOnly; Secure; SameSite=Lax<br/>Set-Cookie: rt=…; HttpOnly; Secure; SameSite=Strict; Path=/api/auth
    B->>N: GET /account/orders (RSC)
    N->>G: GET /api/v1/orders (Authorization: Bearer at)
    Note over N: middleware.ts: if `at` expires within 60s,<br/>call /auth/refresh with `rt`, rotate both cookies
```

- JWT signed with **RS256**. The public keys are published at `/.well-known/jwks.json` (identity). Keys rotate every 90 days, with both keys kept during the overlap.
- Claims: `sub`, `email`, `name`, `role[]`, `perm[]`, `jti`, `iat`, `exp`, `iss=stackshop-identity`, `aud=stackshop-api`.
- Refresh token **rotation** with reuse detection. If a revoked token is reused, the whole `family_id` is revoked.
- Logout: revoke the refresh token family and put the access token's `jti` in the Redis blacklist until it expires.
- Service-to-service calls (gRPC) inside the cluster: network policies plus a forwarded user token. *Stretch goal:* mTLS with Linkerd.

## 5. Authorization model

| Role | Permissions |
|------|-------------|
| Customer | `orders:read:own`, `orders:create`, `reviews:write:own`, `profile:*:own`, `wishlist:*:own` |
| Support | `orders:read`, `orders:cancel`, `orders:refund`, `reviews:moderate`, `users:read` |
| Admin | everything incl. `catalog:write`, `inventory:write`, `promotions:write`, `reports:read`, `users:write` |

Implemented as ASP.NET Core policies: `[Authorize(Policy = "perm:catalog:write")]`. A custom `IAuthorizationPolicyProvider` builds the policies from the prefix. Ownership checks happen in the application handlers (`order.CustomerId == currentUser.Id`).

## 6. Security checklist (OWASP-aligned)

- [ ] Passwords hashed with ASP.NET Identity's PBKDF2 (100k+ iterations) or Argon2id. Minimum 8 characters, checked against a breached-password list (optional).
- [ ] Account lockout after 5 failed logins (15 minutes).
- [ ] Email confirmation tokens are single-use, hashed, and expire in 24h. Reset tokens expire in 30 minutes.
- [ ] All input validated server-side. Output encoded (React does this by default; sanitise product HTML descriptions with `HtmlSanitizer`).
- [ ] CORS: only the frontend origins.
- [ ] Security headers on Next.js: CSP, HSTS, X-Content-Type-Options, Referrer-Policy, frame-ancestors 'none'.
- [ ] CSRF: SameSite cookies + Server Actions (Next.js checks Origin). Route handlers that mutate check the Origin header.
- [ ] Webhooks: HMAC-SHA256 signature + timestamp tolerance of 5 minutes + event-id dedup.
- [ ] Secrets never in git. Use `dotnet user-secrets` locally, K8s Secrets / External Secrets in clusters. Gitleaks runs in CI.
- [ ] Dependency scanning (Dependabot, `dotnet list package --vulnerable`, `pnpm audit`) and image scanning (Trivy) in CI.
- [ ] Least-privilege DB users per service.
- [ ] Pre-signed S3 upload URLs: content-type and size limits, 5-minute expiry.
- [ ] PII: no emails or addresses in logs (Serilog destructuring policy masks them).

## 7. Example endpoint spec format (used in feature docs)

```
POST /api/v1/orders
Auth: Customer   Headers: Idempotency-Key (required)
Request:
  { "cartId": "string", "shippingAddressId": "string", "couponCode": "string?", "note": "string?" }
Response 202:
  { "orderId": "string", "orderNumber": "SS-20261117-000123", "status": "Pending" }
Errors: 400 VALIDATION, 409 CART_EMPTY, 409 PRICE_CHANGED, 422 COUPON_INVALID
```
