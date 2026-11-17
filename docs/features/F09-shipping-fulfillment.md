# F09 — Shipping & Fulfillment

| Field | Value |
|-------|-------|
| Phase | 3 |
| Services | `shipping`, `mock-psp` (also hosts the mock carrier) |
| Stack | EF Core + MySQL, SQS (standard) for carrier webhooks, Kafka |
| Frontend | Checkout shipping step, order timeline, admin fulfillment |
| Depends on | F07 |

## 1. User stories

| ID | Story |
|----|-------|
| US-09.1 | As a customer, I see shipping options and fees at checkout |
| US-09.2 | As a customer, I can track my shipment (tracking number + timeline) |
| US-09.3 | As an admin, I see paid orders waiting to be fulfilled, print a (fake) label, and mark them handed to the carrier |
| US-09.4 | As the system, I update order status automatically from carrier events |

## 2. Shipping quote

- Zones by city/country (`shipping_rates` table): `INNER_CITY`, `DOMESTIC`, `INTERNATIONAL` (stretch goal).
- Fee = rate by zone and total weight band. Express = ×1.5. Free shipping if the subtotal ≥ the configured threshold or a FreeShipping promotion applies.
- `GET /api/v1/shipping/quote?city=&country=&weightG=&subtotal=` → `[{method:"Standard", fee, etaDays:"3-5"}, {method:"Express", fee, etaDays:"1-2"}]`.

## 3. Fulfillment flow

```
ordering.order-paid ──▶ shipping: create Shipment(Created)  → publish shipment-created
Admin "Print label"  ──▶ call mock carrier POST /shipments → tracking number → status LabelPrinted
Mock carrier simulates progression (configurable speed, e.g. every 30s in dev):
   PickedUp → InTransit → OutForDelivery → Delivered   (5% chance: Failed → Returned)
   each step → webhook POST /api/v1/shipments/webhooks/carrier (HMAC)
       → enqueue SQS shipping-webhooks → worker → update shipment + shipment_events → Kafka status-changed
ordering consumes status-changed → Shipped / Delivered
```

## 4. Business rules

- BR-01: One shipment per order (split shipments are a stretch goal).
- BR-02: Status transitions only move forward. Out-of-order webhooks (e.g. `InTransit` after `Delivered`) are stored in `shipment_events` but don't regress the status. Ordering uses each event's `occurredAt`.
- BR-03: An order cancelled before `LabelPrinted` → the shipment is cancelled. After pickup, cancellation is not allowed (a return flow is a stretch goal).
- BR-04: Webhook dedup uses `(tracking_number, status, occurred_at)`. Standard SQS can deliver duplicates, so the handler must be idempotent.

## 5. API

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/shipping/quote` | anon | See §2 |
| GET | `/api/v1/shipments/by-order/{orderId}` | Customer (own) | Shipment + events |
| GET | `/api/v1/admin/shipments?status=` | `orders:read` | Fulfillment queue |
| POST | `/api/v1/admin/shipments/{id}/label` | `orders:fulfill` | Create at the carrier → tracking no. |
| POST | `/api/v1/admin/shipments/bulk-label` | | Multiple |
| POST | `/api/v1/shipments/webhooks/carrier` | HMAC | → SQS |
| CRUD | `/api/v1/admin/shipping-rates` | Admin | |

## 6. Frontend

- Checkout step 2 uses a `RadioGroup` of methods with fees and ETAs.
- Order detail: a tracking card with the carrier, tracking number (copy button), and an event timeline.
- Admin fulfillment page: a "To ship" DataTable with checkbox selection → "Print labels" → a printable label view (HTML with a QR code via `qrcode.react`).

## 7. Acceptance criteria

- AC-1: Paid order → a shipment in `Created` appears in the admin queue within 2s.
- AC-2: Printing the label → a tracking number is assigned. The mock carrier then progresses to `Delivered` and the order becomes `Delivered`.
- AC-3: A duplicate webhook is processed once.
- AC-4: An out-of-order `InTransit` after `Delivered` doesn't regress the status.
- AC-5: The shipping fee in the quote equals the fee on the order.

## 8. Tasks

1. Rates + quote service + tests.
2. Shipment domain + consumer for `order-paid` / `order-cancelled`.
3. Mock carrier in `mock-psp` (background progression + webhooks).
4. Webhook → SQS → worker (shared SQS consumer base from BuildingBlocks).
5. Frontend checkout step, tracking card, admin fulfillment + label print.

## 9. Learning focus

Standard vs FIFO SQS (duplicates, ordering), handling out-of-order events, simulating external systems, admin batch operations.
