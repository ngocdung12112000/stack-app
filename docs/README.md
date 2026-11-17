# StackShop: Documentation

A full-featured e-commerce platform built to **learn every layer of a modern software stack**.

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js (App Router) + shadcn/ui + Tailwind |
| Backend | .NET 8 microservices (Clean Architecture, CQRS), YARP gateway, gRPC, SignalR |
| Databases | MySQL 8 (transactional), MongoDB 7 (documents / read models) |
| Cache | Redis 7 |
| Messaging | Kafka (domain events), RabbitMQ (work queues), AWS SQS / LocalStack (webhooks, long jobs) |
| Search | Elasticsearch 8 |
| Storage | S3 / MinIO |
| Deploy | Docker, Kubernetes (Kustomize/Helm), GitHub Actions CI/CD (+ ArgoCD optional) |
| Monitoring | Prometheus + Grafana (+ Alertmanager; Loki/Tempo optional) |

**Code layout:** three independent repositories, `stackshop-frontend` (Next.js apps), `stackshop-backend` (all .NET microservices) and `stackshop-infra` (K8s/GitOps, monitoring, local infra). See [02-architecture.md §9](./02-architecture.md) and [ADR-001](./adr/001-three-repositories.md).

## Reading order

| # | Document | What's inside |
|---|----------|---------------|
| 1 | [01-overview.md](./01-overview.md) | Vision, goals, personas, feature map, learning outcomes |
| 2 | [02-architecture.md](./02-architecture.md) | System diagram, service catalogue, flows, cross-cutting decisions, repo layout, ADR summary |
| 3 | [03-data-architecture.md](./03-data-architecture.md) | MySQL schemas, MongoDB collections, Redis keys, Elasticsearch mappings, consistency |
| 4 | [04-messaging-and-events.md](./04-messaging-and-events.md) | Kafka topics/events, RabbitMQ topology, SQS queues, outbox, saga state machine |
| 5 | [05-api-design.md](./05-api-design.md) | REST conventions, gateway routes, errors, auth flow, RBAC, security checklist |
| 6 | [06-frontend.md](./06-frontend.md) | Next.js apps, route maps, data-fetching patterns, design system |
| 7 | [07-devops-cicd.md](./07-devops-cicd.md) | Docker Compose, Dockerfiles, Kubernetes, GitHub Actions pipelines |
| 8 | [08-observability.md](./08-observability.md) | Metrics, dashboards, alerts, SLOs, logging |
| 9 | [09-testing-strategy.md](./09-testing-strategy.md) | Test pyramid, tools, critical E2E journeys |
| 10 | [10-roadmap.md](./10-roadmap.md) | 7 phases with checklists and demos, Definition of Done |
| – | [features/](./features/README.md) | **17 detailed feature specs (F01–F17)** |
| – | [adr/](./adr/000-template.md) | Architecture Decision Records ([ADR-001: three repositories, microservices](./adr/001-three-repositories.md)) |

## How to use these docs

1. Read 01 → 02 to understand the big picture.
2. Follow the [roadmap](./10-roadmap.md) phase by phase. Each phase lists the feature specs to implement.
3. For each feature, implement from its spec and tick the acceptance criteria. Update the spec if reality differs.
4. Record any significant decision you change as an ADR in `adr/`.
