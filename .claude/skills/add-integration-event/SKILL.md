---
name: add-integration-event
description: Add a new integration event or message in StackShop. Covers a Kafka domain event (via Transactional Outbox, idempotent consumer with Inbox, DLQ), a RabbitMQ command/job, or an SQS message, with its JSON Schema contract and docs update. Use when a spec lists events to publish or consume, when a service needs to react to another service, or when changing an existing event (versioning).
---

# Add an integration event / message

First pick the broker using `docs/04-messaging-and-events.md` §1. **Kafka** = a fact that happened, with many consumers. **RabbitMQ** = a job one worker must do (email, image). **SQS** = an inbound webhook or a long export job. Don't mix them up.

## A. Kafka domain event

1. **Name and topic:** type `<context>.<event-in-past-tense>.v1` (e.g. `ordering.order-paid.v1`), on the aggregate's topic from §3.2 (create a new topic only for a new aggregate). Key = aggregateId, which keeps per-aggregate ordering.
2. **Contract:** add `contracts/events/<context>/<event>.v1.schema.json` (JSON Schema draft 2020-12, `additionalProperties: false` on `data`). Generate/update the C# record in the contracts project. Include only the data consumers need: **IDs, not PII** (no emails/addresses unless the spec requires them).
3. **Publish (producer service):**
   - Raise a domain event in the aggregate. The `SaveChangesInterceptor` (EF) or Mongo unit of work maps it to the integration event and writes an **outbox row in the same transaction**. Never call the Kafka producer directly from a handler.
   - The outbox relay publishes it with `traceparent`, `correlationId` and `causationId` headers.
4. **Consume (consumer service):**
   - A handler class per event type, registered with the consumer group = the service name.
   - Idempotency: in **one transaction**, check `inbox_messages(message_id, consumer)`, apply the side effect, and insert the inbox row. For Mongo `$inc`-style projections, use the inbox too, because `$inc` isn't idempotent.
   - Tolerate out-of-order and late events (check the aggregate version/state, ignore stale ones, log).
   - Errors: transient → retry in-process (3x) → `<topic>.retry.*` → `<topic>.dlq`. Poison/validation errors go straight to the DLQ. Commit the offset only after handling or dead-lettering.
5. **Metrics:** `stackshop_messages_consumed_total{broker,topic,outcome}` and the handle duration histogram come from building-blocks. Make sure the handler is wrapped.

## B. RabbitMQ command/job

1. A message record (e.g. `SendEmail`) in the contracts project, with an explicit `messageId`.
2. Exchange/queue/routing key as in §4.1 (direct exchange `<service>.commands`). Durable queue, persistent messages, publisher confirms.
3. Consumer: prefetch tuned to the work (CPU-heavy = 1–2). Retry 3 times with delays (e.g. 10s/1m/10m) → `<queue>.dlq`. Make the handler idempotent (a natural key or a delivery log).
4. MassTransit **v8 only** (or raw `RabbitMQ.Client`).

## C. SQS message

1. Queue type: FIFO when per-entity order matters (`MessageGroupId` = entity id, `MessageDeduplicationId` = event id), otherwise standard.
2. Producer: the webhook endpoint verifies the signature, then `SendMessage`, then returns 200.
3. Worker: long poll (20s), batch of 10, delete only after success, extend visibility for long jobs, dedupe in a DB table. maxReceiveCount → DLQ.

## Changing an existing event

- Additive (a new optional field): keep the version and update the schema.
- Breaking (rename/remove/semantics): publish `.v2` **alongside** `.v1` until every consumer has migrated, then remove `.v1` (and note it in the spec).

## Tests

- Producer integration test: the command leads to an outbox row, then the relay puts the message on the topic (Testcontainers Kafka). Assert it matches the schema.
- Consumer: handle → side effect. **The same message twice → one side effect.** A poison message → DLQ.

## Update docs and handoff

- Add the event to the catalogue table in `docs/04-messaging-and-events.md` §3.3 (or the queue tables) and to the spec's Events section (`sync-docs-adr` skill if it's cross-cutting).
- **devops-engineer** (`add-infra-resource`): a new topic/DLQ/retry topics, exchange/queue/DLQ, or SQS queue + DLQ, with partitions/retention/visibility timeout.
