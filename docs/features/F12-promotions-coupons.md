# F12 — Promotions & Coupons

| Field | Value |
|-------|-------|
| Phase | 4 |
| Service | `promotion` |
| Stack | EF Core + MySQL, Redis (Lua atomic usage counters), gRPC server, Kafka |
| Frontend | Cart coupon input, checkout summary, PDP sale badge. Admin promotions |
| Depends on | F03, F06, F07 |

## 1. User stories

| ID | Story |
|----|-------|
| US-12.1 | As an admin, I can create promotions: % off, fixed amount off, free shipping, buy X get Y, scoped to the whole order, categories or products, with a date range and a minimum order amount |
| US-12.2 | As an admin, I can generate coupon codes for a promotion (a single public code like `WELCOME10`, or N unique codes) with usage limits |
| US-12.3 | As a customer, I can apply a coupon in the cart/checkout and see the discount |
| US-12.4 | As a customer, I see a clear reason when a coupon doesn't apply ("Minimum order $50", "Expired", "Already used") |
| US-12.5 | As an admin, I can see usage statistics per promotion |

## 2. Rule engine

```
Evaluate(context{customerId, lines[{sku, productId, categoryIds, unitPrice, qty}], subtotal, shippingFee, couponCode?})
  1. Load active automatic promotions (no coupon required) + the coupon's promotion (cached in Redis 60s)
  2. For each candidate: check window, isActive, minOrderAmount, scope match (eligible lines), customer limits
  3. Compute discount:
     PercentOff  : sum(eligibleLines) * value% , capped by maxDiscount
     AmountOff   : min(value, sum(eligibleLines))
     FreeShipping: shippingFee
     BuyXGetY    : for each eligible sku: floor(qty/(X+Y))*Y cheapest units free
  4. Stacking: at most 1 non-stackable promotion (choose the best for the customer) + any stackable ones
  5. Distribute order-level discount across lines proportionally (needed for refunds/partial cancel) — rounding remainder to largest line
  6. Return {discountTotal, lineDiscounts[], appliedPromotions[], rejectedReasons[]}
```

The engine is a pure domain service with no IO, so it is heavily unit-tested with table-driven tests.

## 3. Coupon usage concurrency

The problem: `usage_limit = 100` and 500 people checking out at the same time.

- At checkout (`ordering` → gRPC `ReserveCoupon(code, customerId, orderId)`), run a Redis Lua script atomically:
  ```lua
  -- KEYS[1]=promo:coupon:{code}:used  KEYS[2]=promo:coupon:{code}:cust:{customerId}
  -- ARGV[1]=usageLimit ARGV[2]=perCustomerLimit
  if tonumber(redis.call('GET', KEYS[1]) or '0') >= tonumber(ARGV[1]) then return -1 end
  if tonumber(redis.call('GET', KEYS[2]) or '0') >= tonumber(ARGV[2]) then return -2 end
  redis.call('INCR', KEYS[1]); redis.call('INCR', KEYS[2]); return 1
  ```
- Then insert `coupon_redemptions(status=Reserved)` in MySQL (the source of truth). If the insert fails → compensate with DECR.
- `order-paid` → `Confirmed`, and increment `coupons.used_count` (row_version).
- `order-cancelled` → `Released` + DECR the Redis counters.
- Reconciliation job (hourly): rebuild the Redis counters from MySQL to fix drift.

## 4. API

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/coupons/validate` | user | `{code, cartSnapshot}` → `{valid, discountPreview, reason?}` |
| GET | `/api/v1/promotions/active` | anon | Public automatic promotions (for banners/badges) |
| CRUD | `/api/v1/admin/promotions` | `promotions:write` | |
| POST | `/api/v1/admin/promotions/{id}/coupons/generate` | | `{count, prefix, usageLimit, perCustomerLimit}` → CSV |
| GET | `/api/v1/admin/promotions/{id}/stats` | | Redemptions, discount given, revenue influenced |

gRPC: `Evaluate`, `ReserveCoupon`, `ReleaseCoupon` (ordering uses these).

Errors / reasons: `COUPON_NOT_FOUND`, `COUPON_EXPIRED`, `COUPON_NOT_STARTED`, `COUPON_USAGE_LIMIT_REACHED`, `COUPON_ALREADY_USED_BY_CUSTOMER`, `COUPON_MIN_ORDER_NOT_MET`, `COUPON_NOT_APPLICABLE_TO_ITEMS`.

## 5. Events

- Consumes `ordering.order-paid` (confirm) and `ordering.order-cancelled` (release).
- Publishes `promotion.coupon-redeemed.v1` (analytics) and `promotion.promotion-changed.v1` (catalog/search can show "SALE" badges as a stretch goal).

## 6. Frontend

- Cart and checkout: a coupon `Input` + Apply button. Success shows a green chip with the code and discount and a remove "×". Failure shows the reason as an inline error.
- The order summary shows discount lines per promotion.
- Admin: a promotions DataTable with a create/edit form (type-dependent fields via a zod discriminated union), scope pickers (category tree / product search `Command`), a date range picker, a coupon generator dialog and a stats page with charts.

## 7. Acceptance criteria

- AC-1: `WELCOME10` (10% off, min $50, max $20) on a $300 cart → discount $20.
- AC-2: Buy 2 get 1 on SKU A with qty 3 → the cheapest unit is free. Qty 5 → still 1 free (floor(5/3)=1).
- AC-3: A usage limit of 100 with a k6 load of 300 concurrent applications at checkout → exactly 100 orders get the discount.
- AC-4: The order is cancelled → the coupon usage is released and the customer can use it again.
- AC-5: An expired coupon shows `COUPON_EXPIRED`.
- AC-6: Two non-stackable promotions → only the better one applies.

## 8. Tasks

1. Domain + rule engine + exhaustive unit tests.
2. Persistence + admin CRUD + coupon generator.
3. Redis Lua scripts (load with `SCRIPT LOAD`, call `EVALSHA`) + reconciliation job.
4. gRPC server.
5. Kafka consumers.
6. UI: coupon input, admin pages.
7. k6 concurrency test.

## 9. Learning focus

Pure domain logic and testing, Redis Lua atomicity, Redis-vs-DB source of truth and reconciliation, discriminated-union forms, money distribution and rounding.
