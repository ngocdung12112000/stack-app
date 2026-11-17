# 01 — Product Overview

> Working name: **StackShop**. It is a full-featured e-commerce platform built mainly to **learn every layer of a modern software stack** end to end.

## 1. Vision

Build a realistic multi-category online store, similar to a small Shopee/Tiki/Amazon. It should be complex enough that every technology in the stack has a **real job to do**. No technology gets added just so it appears in the list.

## 2. Goals

| # | Goal | How we measure it |
|---|------|-------------------|
| G1 | Learn to design and build a distributed system with .NET 8 microservices | 8+ services, each deployable on its own |
| G2 | Learn polyglot persistence | MySQL, MongoDB, Redis and Elasticsearch each own real data |
| G3 | Learn async messaging patterns | Kafka (event streaming), RabbitMQ (work queues), SQS (cloud queues) each used where it fits best |
| G4 | Learn modern frontend | Next.js App Router storefront and admin, built with shadcn/ui |
| G5 | Learn DevOps | Every commit is built, tested, scanned, containerised and deployed to K8s by GitHub Actions |
| G6 | Learn observability | Every service exposes metrics to Prometheus. Grafana dashboards and alerts are in place |

## 3. Non-goals

- Real payment processing. We use a **mock payment provider** (or Stripe test mode, optional).
- Real shipping carriers. We use a **mock carrier** that sends webhooks.
- Multi-vendor marketplace (sellers onboarding). This could be a phase-2 stretch goal.
- Mobile apps.

## 4. Personas

| Persona | Description | Main needs |
|---------|-------------|------------|
| **Guest** | Anonymous visitor | Browse, search, add to cart |
| **Customer** | Registered shopper | Checkout, track orders, review, wishlist |
| **Admin** | Store operator | Manage catalog, inventory, orders, promotions, see reports |
| **Support** | Customer-service staff | View orders, issue refunds, moderate reviews |

## 5. Feature map (high level)

| ID | Feature | Primary stack touched |
|----|---------|-----------------------|
| F01 | Identity & Authentication | .NET Identity, MySQL, Redis (token blacklist), JWT |
| F02 | User Profile & Address Book | MySQL |
| F03 | Product Catalog | MongoDB, Redis cache, Kafka (product events) |
| F04 | Product Search | Elasticsearch, Kafka consumer |
| F05 | Inventory | MySQL, optimistic concurrency, Redis locks, Kafka |
| F06 | Shopping Cart | Redis (primary store) |
| F07 | Checkout & Orders | MySQL, Saga orchestration, Outbox, Kafka |
| F08 | Payments | MySQL, SQS (webhook ingestion), idempotency |
| F09 | Shipping & Fulfillment | MySQL, SQS, Kafka |
| F10 | Notifications | RabbitMQ (work queue + DLQ), MongoDB (log), SignalR |
| F11 | Reviews & Ratings | MongoDB, Kafka, Elasticsearch (rating aggregate) |
| F12 | Promotions & Coupons | MySQL, Redis (usage counters, Lua) |
| F13 | Wishlist | MongoDB |
| F14 | Media / Image Upload | S3 (MinIO locally), RabbitMQ (thumbnail job) |
| F15 | Admin Back-office & Reports | All read models, SQS (export jobs) |
| F16 | Activity Tracking & Recommendations | Kafka streams, MongoDB, Redis |
| F17 | API Gateway & Platform | YARP, rate limiting (Redis), health checks |

Each feature has a detailed spec in [`features/`](./features/).

## 6. Learning outcomes per technology

| Technology | What you will learn |
|------------|---------------------|
| **Next.js + shadcn/ui** | App Router, Server Components, Server Actions, streaming, ISR, BFF pattern, forms with zod, design system |
| **.NET 8** | Minimal APIs and controllers, Clean Architecture, CQRS with MediatR, EF Core, background workers, gRPC, SignalR, resilience with Polly |
| **MySQL** | Relational modelling, transactions, isolation levels, indexes, migrations, optimistic concurrency |
| **MongoDB** | Document modelling, embedding vs referencing, aggregation pipelines, TTL indexes, change streams |
| **Redis** | Cache-aside, distributed locks, rate limiting, Lua scripts, pub/sub, sorted sets |
| **Kafka** | Topics/partitions, consumer groups, ordering by key, outbox → Kafka, idempotent consumers, replay |
| **RabbitMQ** | Exchanges (direct/topic/fanout), work queues, retry with delay, dead-letter exchanges |
| **SQS** | Standard vs FIFO, visibility timeout, DLQ, LocalStack for local dev |
| **Elasticsearch** | Mappings, analyzers, full-text search, facets/aggregations, autocomplete, reindex with aliases |
| **Docker** | Multi-stage builds, compose for local dev, image hardening |
| **Kubernetes** | Deployments, Services, Ingress, ConfigMaps/Secrets, HPA, probes, Helm/Kustomize |
| **GitHub Actions** | CI matrix, caching, test reports, image build/push, environment promotion, GitOps |
| **Prometheus + Grafana** | RED/USE metrics, custom business metrics, PromQL, dashboards, alerting rules |

## 7. Glossary

| Term | Meaning |
|------|---------|
| SKU | Stock Keeping Unit: one purchasable variant (e.g. "T-shirt / Red / M") |
| Saga | A long-running business transaction split into local transactions with compensations |
| Outbox | A table written in the same DB transaction as business data. A relay then publishes it to the broker, which guarantees at-least-once delivery |
| Inbox | A table of processed message IDs, used to make consumers idempotent |
| BFF | Backend-for-Frontend: the Next.js server layer that talks to APIs on the browser's behalf |
| Read model | A denormalised projection built from events for fast queries |
