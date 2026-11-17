# F06 — Shopping Cart

| Field | Value |
|-------|-------|
| Phase | 2 |
| Service | `cart` |
| Stack | Redis (primary datastore: Hash + TTL), gRPC client (catalog, inventory, promotion), Kafka consumer |
| Frontend | Cart drawer (Sheet), `/cart`, header badge |
| Depends on | F03, F05. F12 for coupon preview |

## 1. User stories

| ID | Story |
|----|-------|
| US-06.1 | As a guest, I can add items to a cart that persists for 7 days on this browser |
| US-06.2 | As a customer, my cart follows me across devices for 30 days |
| US-06.3 | When I log in, my guest cart merges into my user cart |
| US-06.4 | I can change quantities, remove items and see the subtotal update instantly |
| US-06.5 | I see a notice when an item's price changed or it went out of stock since I added it |
| US-06.6 | I can preview a coupon's discount in the cart |

## 2. Data model (Redis)

```
cart:{cartId}          HASH  field=sku  value={"productId","sku","name","variantLabel","imageUrl","unitPrice","quantity","addedAt","priceAtAdd"}
cart:meta:{cartId}     HASH  ownerType=guest|user, ownerId, couponCode, currency, updatedAt
cart:user:{userId}     STRING → cartId   (lookup)
TTL: guest 7d, user 30d, refreshed on every write (sliding)
```

- Guest `cartId` = random ULID stored in the httpOnly cookie `cid` (set by Next.js middleware if missing).
- User cart ID = `u-{userId}`.

## 3. Business rules

- BR-01: Max 50 distinct lines. Max quantity per line is 10, and also not more than the available stock.
- BR-02: Adding an existing SKU increments its quantity.
- BR-03: Add-to-cart validates with catalog gRPC `GetSkus` (exists, active, published) and inventory `CheckStock` (available ≥ requested). The line stores a price snapshot for display.
- BR-04: **Merge on login**: for each guest line, if the user cart has the same SKU, take `max(qtyA, qtyB)`; otherwise add it (respecting the 50 limit). Then delete the guest cart. This is atomic via a Lua script or `MULTI`.
- BR-05: On `catalog.product-price-changed` the consumer updates `unitPrice` in carts containing that SKU. A reverse index `cart:sku:{sku}` → SET of cartIds is maintained on add/remove (TTL aligned). This keeps cart prices fresh and teaches reverse indexing in Redis.
- BR-06: `GET /cart` returns `warnings[]`: `PRICE_CHANGED` (unitPrice ≠ priceAtAdd, which is then acknowledged), `OUT_OF_STOCK`, `QUANTITY_REDUCED`, `UNAVAILABLE` (product archived).
- BR-07: The cart never computes final tax/shipping. It shows the subtotal and a coupon preview only. Totals are authoritative at checkout (ordering).
- BR-08: After the order is created successfully, ordering publishes `order-created`. The cart consumer clears the cart for that `cartId` (included in the event) only after the order reaches `Paid`, or earlier (configurable). Default: clear on `order-created` and **restore** on `reservation-failed` from the order snapshot (stretch goal). Simpler: clear on `order-paid`.

## 4. API

| Method | Path | Body | Description |
|--------|------|------|-------------|
| GET | `/api/v1/carts/current` | – | Resolves the cart from the user token or the `X-Cart-Id` header. Returns `{id, lines[], subtotal, itemCount, coupon?, discountPreview, warnings[]}` |
| POST | `/api/v1/carts/current/items` | `{sku, quantity}` | Add |
| PATCH | `/api/v1/carts/current/items/{sku}` | `{quantity}` | Set quantity (0 = remove) |
| DELETE | `/api/v1/carts/current/items/{sku}` | | Remove |
| DELETE | `/api/v1/carts/current` | | Clear |
| POST | `/api/v1/carts/current/coupon` | `{code}` | Validate via promotion gRPC and store the code |
| DELETE | `/api/v1/carts/current/coupon` | | |
| POST | `/api/v1/carts/merge` | `{guestCartId}` | Called by the BFF right after login |

Errors: `CART_LINE_LIMIT`, `CART_QUANTITY_LIMIT`, `CART_SKU_UNAVAILABLE`, `CART_INSUFFICIENT_STOCK`, `COUPON_INVALID`.

## 5. Events

- Consumes `catalog.product-price-changed`, `catalog.product-deleted` (marks lines unavailable), `ordering.order-paid` (clear cart).
- Publishes `cart.item-added.v1` / `cart.item-removed.v1` to `analytics.clickstream` (fire-and-forget, no outbox needed: analytics loss is tolerable).

## 6. Frontend

- "Add to cart" button uses a Server Action + `useOptimistic` for the header badge count. A Sonner toast offers a "View cart" action.
- Cart `Sheet` from the right: line list with `QuantityStepper`, remove, subtotal, "Checkout" button.
- `/cart` page: full table, coupon input, warnings shown as `Alert`, "Continue shopping".
- After the login Server Action → call merge → `revalidateTag('cart')`.

## 7. Non-functional

- Cart ops p95 < 30ms (Redis only; gRPC calls happen only on add).
- Redis persistence: AOF `everysec`.
- Metric: `stackshop_cart_operations_total{op}`. Cart abandonment is computed in analytics.

## 8. Acceptance criteria

- AC-1: A guest adds 2 items, closes the browser, comes back within 7 days → the items are still there.
- AC-2: The guest has SKU A ×2 and the user cart has A ×1 + B ×1 → after login the cart is A ×2 + B ×1.
- AC-3: Adding qty 5 when 3 are available → 409 `CART_INSUFFICIENT_STOCK` with `available: 3`.
- AC-4: The admin changes the price of SKU A → the cart shows the new price and a `PRICE_CHANGED` warning.
- AC-5: After the order is paid, the cart is empty.

## 9. Tasks

1. `CartRepository` on Redis (Hash ops, TTL, reverse index). Lua script for merge.
2. gRPC clients with resilience (timeout 1s, retry, circuit breaker). Degraded mode: if inventory is down, allow add with a warning.
3. Endpoints + validators.
4. Consumers (price change, product deleted, order paid).
5. Next.js: `cid` cookie in middleware, cart Server Actions, Sheet UI, `/cart` page.
6. Integration tests with Testcontainers Redis + WireMock/fake gRPC.

## 10. Learning focus

Redis as a primary store (data structures, TTL, Lua atomicity, persistence trade-offs), reverse indexes, BFF cookie handling, optimistic UI.
