# 04 — Messaging & Integration Events

## 1. Which broker for what?

| Question | Kafka | RabbitMQ | SQS |
|----------|-------|----------|-----|
| Message meaning | **Event**: a fact in past tense ("OrderPaid") | **Command / job**: "SendEmail", "ProcessImage" | **External input / long job**: webhook payloads, export jobs |
| Consumers | Many independent consumer groups | Competing workers of **one** logical consumer | Competing workers of one consumer |
| Ordering | Per partition (key = aggregateId) | Per queue (not guaranteed with multiple consumers) | Standard: none. FIFO: per `MessageGroupId` |
| Replay | Yes (retention / offset reset) | No | No |
| Retry | App-level (retry topic or in-process) | Native: delayed requeue + DLX | Native: visibility timeout + redrive to DLQ |
| Used in | catalog, inventory, ordering, payment, shipping, review, search, analytics | notification, media | payment, shipping, analytics |

## 2. Event envelope (all brokers)

```jsonc
{
  "id": "0193f1c2-...-v7",               // unique message id → used for Inbox dedup
  "type": "ordering.order-paid.v1",      // <bounded-context>.<event>.v<version>
  "source": "ordering",
  "occurredAt": "2026-11-17T09:30:12.123Z",
  "correlationId": "req-…",               // from X-Request-Id
  "causationId": "id of the message that caused this one",
  "traceparent": "00-…",                  // W3C trace context
  "data": { /* event specific */ }
}
```

- Serialization: JSON (System.Text.Json, camelCase). *Stretch goal:* Avro + Confluent Schema Registry.
- Contracts live in `stackshop-backend/contracts/events/*.schema.json`. Events are backend-internal: the frontend never consumes Kafka/RabbitMQ/SQS directly. A CI step validates producers against them with contract tests.
- **Versioning:** additive changes keep the version. Breaking changes produce a new type `.v2`, and the producer publishes both until all consumers have migrated.

## 3. Kafka

### 3.1 Cluster config (local)

- Single-broker KRaft (`bitnami/kafka` or `apache/kafka`) + Kafka UI (`provectuslabs/kafka-ui`).
- In K8s: Strimzi operator, 3 brokers, `min.insync.replicas=2`.
- Producer: `acks=all`, `enable.idempotence=true`, `linger.ms=5`, `compression.type=lz4`.
- Consumer: `enable.auto.commit=false`. Commit **after** handling. `auto.offset.reset=earliest` for new groups.

### 3.2 Topics

| Topic | Key | Partitions | Producer | Consumers (group) |
|-------|-----|-----------|----------|-------------------|
| `identity.user-events` | userId | 3 | identity | notification, analytics |
| `catalog.product-events` | productId | 6 | catalog | search, cart, analytics, ordering (price cache) |
| `catalog.product-snapshot` (compacted) | productId | 6 | catalog | search (bootstrap / rebuild) |
| `inventory.stock-events` | sku | 6 | inventory | search, catalog (badge), analytics, notification (low stock → admin) |
| `inventory.reservation-events` | orderId | 6 | inventory | ordering |
| `ordering.order-events` | orderId | 6 | ordering | inventory, payment, shipping, promotion, notification, review, analytics |
| `payment.payment-events` | orderId | 6 | payment | ordering, notification, analytics |
| `shipping.shipment-events` | orderId | 6 | shipping | ordering, notification, analytics |
| `review.review-events` | productId | 3 | review | catalog, search, analytics |
| `analytics.clickstream` | anonId / userId | 12 | gateway / BFF (track endpoint) | analytics |
| `*.dlq` | same as source | 1 | consumer framework | ops / replay tool |

> Several event **types** share one topic per aggregate. This keeps ordering per aggregate. For example, `OrderCreated` → `OrderPaid` → `OrderShipped` for the same order are always in order.

### 3.3 Event catalogue

| Event type | Topic | Payload (data) |
|-----------|-------|----------------|
| `identity.user-registered.v1` | identity.user-events | userId, email, fullName |
| `catalog.product-created.v1` / `-updated.v1` | catalog.product-events | full product snapshot (for simple consumers) |
| `catalog.product-price-changed.v1` | catalog.product-events | productId, sku, oldPrice, newPrice |
| `catalog.product-deleted.v1` | catalog.product-events | productId |
| `inventory.stock-changed.v1` | inventory.stock-events | sku, available, onHand, warehouseId |
| `inventory.low-stock.v1` | inventory.stock-events | sku, available, threshold |
| `inventory.stock-reserved.v1` | inventory.reservation-events | orderId, reservationId, expiresAt |
| `inventory.reservation-failed.v1` | inventory.reservation-events | orderId, unavailableSkus[] |
| `inventory.reservation-released.v1` | inventory.reservation-events | orderId, reason |
| `ordering.order-created.v1` | ordering.order-events | orderId, customerId, lines[{sku, qty, unitPrice}], totals, couponCode |
| `ordering.payment-requested.v1` | ordering.order-events | orderId, amount, currency, customerId |
| `ordering.order-paid.v1` | ordering.order-events | orderId, paymentId, lines, shippingAddress |
| `ordering.order-cancelled.v1` | ordering.order-events | orderId, reason, wasPaid |
| `ordering.order-shipped.v1` / `-delivered.v1` | ordering.order-events | orderId, trackingNumber / deliveredAt |
| `payment.succeeded.v1` | payment.payment-events | paymentId, orderId, amount |
| `payment.failed.v1` | payment.payment-events | paymentId, orderId, failureCode |
| `payment.refunded.v1` | payment.payment-events | paymentId, orderId, refundId, amount |
| `shipping.shipment-created.v1` | shipping.shipment-events | shipmentId, orderId, carrier |
| `shipping.status-changed.v1` | shipping.shipment-events | shipmentId, orderId, status, trackingNumber, occurredAt |
| `review.review-approved.v1` | review.review-events | reviewId, productId, rating |
| `review.rating-updated.v1` | review.review-events | productId, average, count |
| `promotion.coupon-redeemed.v1` | (promotion) | couponCode, orderId, customerId |

### 3.4 Consumer error handling

```
handle(msg)
  ├─ success → store inbox(msgId, consumer) in the same tx as the side effect → commit offset
  ├─ transient error (DB timeout, 5xx) → in-process retry 3x (100ms, 500ms, 2s)
  │      └─ still failing → publish to `<topic>.retry.1m` → ... → `<topic>.dlq`, commit offset
  └─ poison / validation error → publish straight to `<topic>.dlq` with error headers, commit offset
```

The admin tool `analytics`/`ops` endpoint lists DLQ messages and can **replay** them to the original topic.

## 4. RabbitMQ

### 4.1 Topology

```
Exchange: notification.commands (direct)
  ├─ routing key: email   → queue: notification.email            (consumers: EmailWorker x N, prefetch 10)
  └─ routing key: inapp   → queue: notification.inapp            (consumers: InAppWorker → SignalR)

Exchange: media.commands (direct)
  └─ routing key: process-image → queue: media.process-image     (consumers: ImageWorker, prefetch 2)

Exchange: media.events (fanout)
  └─ queue: catalog.media-image-processed                        (consumer: catalog)

Retry (per queue, via MassTransit or manual):
  queue X  --nack-->  exchange X.retry (x-delayed-message OR TTL queues X.retry.10s / .1m / .10m)
                       └─ after TTL dead-letters back to X
  after max attempts  → exchange X.dlx → queue X.dlq  (alert + manual requeue in management UI)
```

### 4.2 Messages

| Message | Queue | Payload |
|---------|-------|---------|
| `SendEmail` | notification.email | messageId, to, templateId, locale, model{}, priority |
| `PushInApp` | notification.inapp | userId, title, body, link |
| `ProcessImage` | media.process-image | assetId, bucket, key, sizes[] |
| `ImageProcessed` | catalog.media-image-processed | assetId, ownerType, ownerId, variants{original, 800, 400, 120} |

Settings: durable queues, persistent messages, publisher confirms, `prefetch` tuned per worker. Use **quorum queues** in K8s.

## 5. SQS (AWS or LocalStack)

| Queue | Type | Producer | Consumer | Visibility timeout | DLQ (maxReceiveCount) |
|-------|------|----------|----------|--------------------|------------------------|
| `payment-webhooks.fifo` | FIFO (group = providerPaymentId, dedup = eventId) | payment API webhook endpoint | payment worker | 30s | `payment-webhooks-dlq.fifo` (5) |
| `shipping-webhooks` | Standard | shipping API webhook endpoint | shipping worker | 30s | `shipping-webhooks-dlq` (5) |
| `report-export-jobs` | Standard | analytics API (admin requests export) | export worker → CSV/XLSX to S3 → notify | 300s (+ heartbeat extension) | `report-export-jobs-dlq` (3) |

Learning points: long polling (`WaitTimeSeconds=20`), batch receive (10), `ChangeMessageVisibility` for long jobs, delete only after success, redrive from the DLQ.

Local: LocalStack container. Queues are created by the init script `stackshop-infra/local/init/localstack/init-aws.sh`. Kafka topics and RabbitMQ definitions are created the same way (`local/init/`), and in K8s as Strimzi `KafkaTopic` CRs and RabbitMQ topology CRs in `stackshop-infra`.

## 6. Transactional Outbox implementation

1. The command handler changes aggregates. Domain events are collected on the aggregate.
2. The EF `SaveChangesInterceptor` maps domain events → integration events and inserts them into `outbox_messages` **in the same transaction**.
3. `OutboxRelay : BackgroundService` runs every 500ms (or uses MySQL binlog CDC via Debezium as a *stretch goal*):
   - `SELECT ... WHERE processed_at IS NULL ORDER BY occurred_at LIMIT 100 FOR UPDATE SKIP LOCKED`
   - Produce to Kafka (await delivery report) → set `processed_at`.
4. Cleanup job: delete processed rows older than 7 days.

Metrics: `outbox_pending_messages` (gauge), `outbox_publish_duration_seconds`, `outbox_publish_failures_total`.

## 7. Saga: Checkout state machine (Ordering)

| State | On event | Action | Next state |
|-------|----------|--------|-----------|
| Pending | (created) | publish `order-created` | Pending |
| Pending | `stock-reserved` | publish `payment-requested`. Set payment deadline = now + 15m | AwaitingPayment |
| Pending | `reservation-failed` | cancel order, notify | Cancelled |
| AwaitingPayment | `payment.succeeded` | publish `order-paid` | Paid |
| AwaitingPayment | `payment.failed` (final) / deadline passed | publish `order-cancelled` → inventory releases, promotion releases | Cancelled |
| Paid | `shipment-created` | – | Processing |
| Processing | `status-changed: PickedUp/InTransit` | publish `order-shipped` | Shipped |
| Shipped | `status-changed: Delivered` | publish `order-delivered` | Delivered |
| Paid/Processing | admin cancel | publish `order-cancelled{wasPaid:true}` → payment refunds | Cancelled → Refunded |

A `SagaTimeoutWorker` scans `saga_state.payment_deadline < now()` every 30s.
Inventory has its own `ReservationExpiryWorker` as a safety net (it releases expired reservations even if ordering is down).
