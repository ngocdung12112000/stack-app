# F10 — Notifications (Email + In-app real-time)

| Field | Value |
|-------|-------|
| Phase | 4 |
| Service | `notification` (API + EmailWorker + InAppWorker) |
| Stack | Kafka consumer → **RabbitMQ** work queues with delayed retry + DLX, MongoDB (templates, log, inbox), SignalR with a Redis backplane, SMTP (Mailpit) |
| Frontend | Bell icon dropdown, `/account/notifications`, live toasts. Admin template editor |
| Depends on | F01, F07, F08, F09 |

## 1. User stories

| ID | Story |
|----|-------|
| US-10.1 | As a customer, I get emails for: welcome/confirm email, password reset, order placed, payment received, order shipped, order delivered, order cancelled/refunded |
| US-10.2 | As a customer, I get in-app notifications (bell with unread count) and live toasts while I'm on the site |
| US-10.3 | As a customer, I can mark notifications as read |
| US-10.4 | As a customer, I can opt out of marketing emails (transactional emails are always sent) |
| US-10.5 | As an admin, I get an in-app alert on low stock and on new reviews awaiting moderation |
| US-10.6 | As an admin, I can edit email templates with a live preview |

## 2. Event → notification mapping

| Kafka event | Email template | In-app | Recipient |
|-------------|----------------|--------|-----------|
| `identity.user-registered` | `welcome-confirm` | – | user |
| `identity.password-reset-requested` | `password-reset` | – | user |
| `ordering.order-created` | `order-placed` | ✓ | customer |
| `ordering.order-paid` | `payment-received` | ✓ | customer |
| `ordering.order-shipped` | `order-shipped` (with tracking) | ✓ | customer |
| `ordering.order-delivered` | `order-delivered` (+ "review your items" CTA) | ✓ | customer |
| `ordering.order-cancelled` | `order-cancelled` | ✓ | customer |
| `payment.refunded` | `refund-issued` | ✓ | customer |
| `inventory.low-stock` | – | ✓ | all admins |
| `review.review-submitted` | – | ✓ | support |

The mapping is config-driven (`appsettings` / a Mongo `rules` collection). Customer email and name are looked up from identity (gRPC/HTTP, cached in Redis for 10 minutes) because events carry IDs, not PII.

## 3. Processing pipeline

```
Kafka consumer (group notification)
  └─ map event → NotificationRequest[] → persist in Mongo notifications (in-app) & delivery_log (Pending)
       → publish RabbitMQ:
           exchange notification.commands, rk=email → queue notification.email
           exchange notification.commands, rk=inapp → queue notification.inapp
EmailWorker (prefetch 10, N replicas)
  └─ render template (Scriban) with model → send SMTP (MailKit) → delivery_log Sent
     failure → retry 10s → 1m → 10m (delayed exchange) → DLQ notification.email.dlq → status Failed + alert
InAppWorker
  └─ SignalR IHubContext → Clients.User(userId).SendAsync("notification", dto)  (Redis backplane for multi-pod)
```

Dedup: `delivery_log` has a unique index on `(sourceEventId, template, recipient)`, so a replayed Kafka event never sends a second email.

## 4. API

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/notifications?cursor=&limit=` | user | In-app list |
| GET | `/api/v1/notifications/unread-count` | user | |
| POST | `/api/v1/notifications/{id}/read` | user | |
| POST | `/api/v1/notifications/read-all` | user | |
| GET/PUT | `/api/v1/notifications/preferences` | user | `{marketingEmail: bool}` |
| WS | `/hubs/notifications` | user (token via `access_token` query, as SignalR requires) | Server → client `notification`, `orderStatusChanged` |
| GET/PUT | `/api/v1/admin/notification-templates/{id}` | Admin | |
| POST | `/api/v1/admin/notification-templates/{id}/preview` | Admin | Render with sample model |
| GET | `/api/v1/admin/notifications/delivery-log?status=Failed` | Admin | |

> Since the browser has no access to the httpOnly token, the BFF exposes `GET /api/realtime-token`, which returns a **short-lived (60s) hub token** that is only valid for the hub audience.

## 5. Frontend

- Header `Popover` with a bell, an unread `Badge`, the last 10 notifications, and "Mark all read".
- `useNotifications()` hook: connects via `@microsoft/signalr` with automatic reconnect. On a message it shows a Sonner toast and invalidates the query.
- The order result and detail pages listen to `orderStatusChanged` for live updates.
- Admin template editor: `Textarea`/code editor (Monaco optional) + an iframe preview.

## 6. Acceptance criteria

- AC-1: Placing and paying for an order → "Order placed" and "Payment received" emails appear in Mailpit, and the in-app bell shows 2 unread.
- AC-2: With Mailpit/SMTP stopped, the email retries 3 times and lands in the DLQ. Restarting SMTP and requeueing from the RabbitMQ UI sends it.
- AC-3: Replaying the `order-paid` Kafka event doesn't send a duplicate email.
- AC-4: With 2 notification pods, a user connected to pod A receives an event processed by pod B (Redis backplane).
- AC-5: A user who opted out of marketing still gets transactional emails.

## 7. Tasks

1. RabbitMQ in compose (with the `rabbitmq_delayed_message_exchange` plugin) + definitions.
2. MassTransit v8 (or raw client) setup: retry/redelivery policies, DLQ naming.
3. Kafka consumer + mapping rules + recipient lookup.
4. Template engine + seed templates (HTML email layout, with MJML optional).
5. EmailWorker (MailKit), InAppWorker + SignalR hub + Redis backplane.
6. API endpoints + realtime-token BFF route.
7. Frontend bell, toasts, notifications page, admin template editor.
8. Tests: template rendering, idempotency, integration with Testcontainers RabbitMQ + Mailpit.

## 8. Learning focus

RabbitMQ exchanges/queues/bindings, prefetch, ack/nack, delayed retry, DLX. The Kafka vs RabbitMQ difference in practice. SignalR scale-out. Template rendering.
