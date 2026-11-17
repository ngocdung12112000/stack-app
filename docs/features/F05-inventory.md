# F05 — Inventory

| Field | Value |
|-------|-------|
| Phase | 2 (stock + events), 3 (reservations) |
| Service | `inventory` |
| Stack | EF Core + MySQL (row locking, optimistic concurrency), Redis lock, Kafka, gRPC |
| Frontend | PDP stock badge, admin inventory pages |
| Depends on | F03 (SKUs) |

## 1. User stories

| ID | Story |
|----|-------|
| US-05.1 | As a shopper, I see "In stock", "Only N left" (N ≤ 5) or "Out of stock" per variant |
| US-05.2 | As a shopper, I can never buy more units than are available (no overselling) |
| US-05.3 | As an admin, I can set or adjust stock per SKU per warehouse, with a reason |
| US-05.4 | As an admin, I can see a movement ledger (who changed what, when, why) |
| US-05.5 | As an admin, I get notified when stock falls below the threshold |
| US-05.6 | As the system, I reserve stock while a customer pays and release it if they don't |

## 2. Concepts

```
available = on_hand - reserved
Reserve  : reserved += q          (order created)
Commit   : on_hand -= q, reserved -= q   (order paid)
Release  : reserved -= q          (order cancelled / payment timeout / reservation expired)
Adjust   : on_hand += delta       (admin restock / shrinkage), never below reserved
```

Phase-1 simplification: a single warehouse `MAIN`. The multi-warehouse allocation strategy (pick the warehouse with the most stock, or the nearest one) is a stretch goal.

## 3. Business rules

- BR-01: Reservation is **all-or-nothing** per order. If any line lacks stock, nothing is reserved and `reservation-failed` lists the SKUs.
- BR-02: Reservation TTL is 15 minutes (must match the payment deadline). `ReservationExpiryWorker` runs every 30s and releases expired `Active` reservations.
- BR-03: Concurrency. Reserve uses a single SQL transaction with `SELECT ... FOR UPDATE` on the affected `stock_items` rows, **ordered by sku** to avoid deadlocks. *Exercise:* also implement an optimistic version (`row_version`) and compare them under k6 load.
- BR-04: Idempotency. Reserve is keyed by `orderId` (unique). Processing the same `order-created` twice returns the existing reservation result.
- BR-05: Commit/Release on a reservation that is not `Active` is a no-op (idempotent), but it is logged.
- BR-06: Admin adjustments cannot make `on_hand < reserved` (409).
- BR-07: Every change writes a `stock_movements` row (an append-only ledger) and emits `stock-changed` when `available` changes.
- BR-08: When `available` crosses below `low_stock_threshold`, emit `low-stock` once (it resets when stock is back above the threshold).

## 4. API

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/inventory/availability?skus=a,b` | anon | `[{sku, available, status: InStock/Low/Out}]` (Redis-cached 5s) |
| GET | `/api/v1/admin/inventory?search=&status=low&page=` | `inventory:read` | Stock list |
| GET | `/api/v1/admin/inventory/{sku}` | | Per-warehouse detail + recent movements |
| POST | `/api/v1/admin/inventory/{sku}/adjust` | `inventory:write` | `{warehouseId, delta, reason, note}` |
| PUT | `/api/v1/admin/inventory/{sku}/threshold` | | `{threshold}` |
| GET | `/api/v1/admin/inventory/movements?sku=&from=&to=` | | Ledger |
| POST | `/api/v1/admin/inventory/import` | | CSV `sku,warehouse,onHand` (stretch goal) |

gRPC: `rpc CheckStock(CheckStockRequest{lines[]}) returns (CheckStockResponse{lines[{sku, available}]})`. Used by cart validation and checkout pre-check.

Errors: `INVENTORY_INSUFFICIENT_STOCK`, `INVENTORY_BELOW_RESERVED`, `INVENTORY_SKU_NOT_FOUND`.

## 5. Events

| Consumes | Action |
|----------|--------|
| `ordering.order-created` | Reserve → publish `stock-reserved` or `reservation-failed` |
| `ordering.order-paid` | Commit |
| `ordering.order-cancelled` | Release (if Active). If already committed and `wasPaid` (refund case), restock with reason `Return` only when the admin marks it returned (stretch goal) |
| `catalog.product-created/updated` | Ensure `stock_items` rows exist for new SKUs (on_hand = 0) |

| Publishes | Topic |
|-----------|-------|
| `inventory.stock-changed.v1`, `inventory.low-stock.v1` | inventory.stock-events |
| `inventory.stock-reserved.v1`, `inventory.reservation-failed.v1`, `inventory.reservation-released.v1` | inventory.reservation-events |

## 6. Frontend

- PDP: `StockBadge` per selected variant (from the `availability` endpoint, refetched client-side on variant change).
- Cart: shows a warning if a line's quantity is greater than available.
- Admin: Inventory DataTable (SKU, product, on hand, reserved, available, status badge), an "Adjust" dialog, a movement ledger table with filters, and a low-stock tab.

## 7. Acceptance criteria

- AC-1: 50 concurrent checkouts for a SKU with 10 units → exactly 10 reservations succeed and 40 fail. `on_hand` never goes negative (k6 + integration test).
- AC-2: The same `order-created` delivered twice → one reservation.
- AC-3: An unpaid reservation is released after 15 minutes and `available` is restored.
- AC-4: Adjusting below reserved returns 409.
- AC-5: Every change appears in the movement ledger with the actor and reason.
- AC-6: A low-stock event fires once when crossing the threshold, not on every subsequent sale.

## 8. Tasks

1. Schema + EF migrations. Seed the `MAIN` warehouse.
2. `Reserve` with pessimistic locking (raw SQL `FOR UPDATE` via EF `FromSql`). Unit + concurrency integration test.
3. Commit / Release / Expiry worker.
4. Kafka consumers with Inbox. Outbox publisher.
5. gRPC `CheckStock`. Availability endpoint with a short Redis cache.
6. Admin UI pages.
7. k6 script `tests/load/oversell.js`.

## 9. Learning focus

Transactions and isolation levels in MySQL (InnoDB row locks, gap locks, deadlocks), pessimistic vs optimistic locking, ledger/audit patterns, idempotent consumers, background workers.
