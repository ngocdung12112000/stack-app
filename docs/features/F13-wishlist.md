# F13 — Wishlist

| Field | Value |
|-------|-------|
| Phase | 4 |
| Service | `catalog` (module `Wishlist`). It's small, so it doesn't justify its own service |
| Stack | MongoDB (`wishlists` collection, one doc per user), Kafka consumer |
| Frontend | Heart button on ProductCard/PDP, `/account/wishlist` |
| Depends on | F01, F03, F05, F10 |

## 1. User stories

| ID | Story |
|----|-------|
| US-13.1 | As a customer, I can add or remove a product from my wishlist with one click |
| US-13.2 | As a customer, I see my wishlist with current price, stock status and "Add to cart" |
| US-13.3 | As a customer, I'm notified when a wishlisted item goes on sale or comes back in stock (opt-in per item) |
| US-13.4 | As a guest clicking the heart, I'm asked to log in and the item is added after login |

## 2. Rules

- BR-01: Max 200 items. The document is `{_id: userId, items: [{productId, sku?, addedAt, notifyOnSale, notifyOnRestock}]}`. Add uses `$addToSet` semantics on productId. Remove uses `$pull`.
- BR-02: The list view hydrates product cards via the batch product query (`GET /products?ids=`). Archived products show as "No longer available" with a remove button.
- BR-03: Alerts: a consumer on `catalog.product-price-changed` (price drop ≥ 5%) and `inventory.stock-changed` (0 → >0) finds the users via the index `{"items.productId": 1}` and emits `wishlist.alert.v1` → notification (in-app + email). At most 1 alert per product per user per 24h (Redis key `wl:alert:{userId}:{productId}` with a TTL).

## 3. API

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/wishlist` | Items hydrated |
| GET | `/api/v1/wishlist/ids` | Product IDs only (to render hearts on grids) |
| PUT | `/api/v1/wishlist/items/{productId}` | Add (idempotent) `{notifyOnSale?, notifyOnRestock?}` |
| DELETE | `/api/v1/wishlist/items/{productId}` | Remove |

## 4. Frontend

- A `WishlistHeart` toggle with an optimistic update. The IDs are fetched once per session and cached in TanStack Query.
- `/account/wishlist`: a grid of `ProductCard`s with notification toggles (`Switch`) and "Move to cart".
- Guest: the heart opens a login `Dialog`. The pending productId is stored in sessionStorage and added after login.

## 5. Acceptance criteria

- AC-1: Toggling the heart twice leaves the item absent. No duplicates are possible.
- AC-2: A price drop of 10% on a wishlisted product with `notifyOnSale` → an in-app + email alert, once within 24h.
- AC-3: The 201st item → 409 `WISHLIST_LIMIT`.

## 6. Learning focus

Single-document-per-user modelling, multikey indexes, fan-out from events to many users, alert throttling with Redis TTL.
