# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

**StackShop** is a full-featured e-commerce app built as a learning project. Its purpose is to exercise every layer of a modern stack: Next.js + shadcn, .NET 8 microservices, MySQL, MongoDB, Redis, Kafka, RabbitMQ, SQS, Elasticsearch, Docker/K8s, GitHub Actions, and Prometheus/Grafana.

**Phase 0 is scaffolded locally** in `stackshop-infra/`, `stackshop-backend/` and `stackshop-frontend/` (each its own git repo, `git init` done, nothing committed or pushed yet). Each has its own `CLAUDE.md` with real commands, so prefer that file for command details. The documents in `docs/` are the source of truth. Read the relevant spec before implementing anything, and update the spec if the implementation deviates from it.

- `docs/README.md`: index and reading order
- `docs/10-roadmap.md`: phases 0–6. Implement in this order. Each phase lists its feature specs, which services it introduces, and a demo goal
- `docs/features/F01…F17-*.md`: per-feature specs with user stories (`US-xx.n`), business rules (`BR-n`) and acceptance criteria (`AC-n`). Reference these IDs in PR titles and test names, e.g. `F07_AC2_SameIdempotencyKey_ReturnsSameOrder`
- `docs/adr/`: decision records. Record any change to an architectural decision as a new ADR

## Workspace layout: three independent repos (ADR-001)

This folder is the **design/docs repo** (`github.com/ngocdung12112000/stack-app`). The three code repositories will be checked out inside it as separate git repos. `.gitignore` excludes them, so they are never committed here:

| Repo | Contents |
|------|----------|
| `stackshop-frontend/` | pnpm + Turborepo: `apps/web` (storefront), `apps/admin`, `packages/ui` (shadcn), `packages/config`, `tests/e2e` (Playwright) |
| `stackshop-backend/` | All .NET 8 microservices in `services/<name>/` (each with its own `.sln`, Dockerfile, version), `gateway` (YARP), `mock-psp`, `building-blocks/`, `contracts/{events,protos,openapi}`, `clients/api-client`, `tools/Seeder`, `tests/load` (k6) |
| `stackshop-infra/` | `local/compose.yml` + `local/init/` (all third-party infra for local dev), `k8s/apps/<svc>/{base,overlays}`, `k8s/platform`, `k8s/data`, `monitoring/` (Grafana dashboards, PrometheusRules), `argocd/` |

Repo boundaries to respect:
- The repos never import each other's source. The **frontend depends on the backend only via the npm package `@ngocdung12112000/api-client`** (published from the backend to GitHub Packages) and the gateway URL (`API_BASE_URL`, server-side only).
- App repos contain **no K8s manifests** and never deploy. They build images to GHCR and open tag-bump PRs in `stackshop-infra`, and ArgoCD syncs the clusters.
- Dashboards, alerts, scrape config, env vars, secrets and new services all need an infra-repo change.
- A feature spanning repos is merged in the order **backend → client publish → frontend → infra**. Public API changes must stay backward compatible for one frontend release (expand → migrate → contract).
- Run each git command inside the relevant code repo folder. Git at the workspace root is for docs changes only.

## Subagents (`.claude/agents/`)

Delegate work by repo. Each agent embeds the rules above and reads the relevant docs before acting:

| Agent | Scope |
|-------|-------|
| `backend-engineer` | `stackshop-backend/`: services, gateway, contracts, api-client, backend tests |
| `frontend-engineer` | `stackshop-frontend/`: Next.js apps, `packages/ui`, BFF, frontend tests |
| `devops-engineer` | `stackshop-infra/` (compose, K8s, ArgoCD, monitoring), plus Dockerfiles and CI/CD workflows in the app repos |

A feature that spans repos runs in the order backend → frontend → devops. Each agent ends with a handoff list for the others.

## Project skills (`.claude/skills/`)

Each agent's file maps task types to these skills. Load the matching skill before doing that kind of work.

| Owner | Skills |
|-------|--------|
| Main session | `implement-feature` (orchestrates an Fxx spec across the agents), `sync-docs-adr`, `local-env` |
| backend-engineer | `new-microservice`, `add-api-endpoint`, `add-integration-event`, `add-grpc-contract`, `db-migration` |
| frontend-engineer | `new-page`, `server-action-form`, `bump-api-client` (+ installed `ui-ux-pro-max:*`, `dataviz`) |
| devops-engineer | `k8s-service-manifests`, `add-infra-resource`, `add-dashboard-alert`, `ci-workflow`, `promote-release` |

When asked to implement a feature spec, start with `implement-feature`.

## Planned local dev commands (from docs/07-devops-cicd.md)

```bash
cd stackshop-infra && docker compose -f local/compose.yml up -d   # DBs, brokers, ES, LocalStack, Mailpit, Prometheus, Grafana (creates network "stackshop")
cd stackshop-backend && docker compose up -d --build               # .NET services; gateway on :5000
cd stackshop-frontend && pnpm install && pnpm dev                  # web :3000, admin :3001
```

Tests: xUnit + Testcontainers (backend), Vitest + Playwright (frontend), and k6 (backend `tests/load`). Once real build/test commands exist, give each repo its own CLAUDE.md with them and trim this section.

## Backend architecture (big picture)

Microservices from day one, with no modular-monolith stage. Services are added phase by phase (P1: gateway, identity, catalog → P2: search, inventory, cart → P3: ordering, payment, shipping, mock-psp → P4: notification, review, promotion, media → P6: analytics).

Each service uses Clean Architecture with 4 projects: `X.Domain` (no dependencies) → `X.Application` (MediatR CQRS, FluentValidation) → `X.Infrastructure` (EF Core/Mongo/Redis/brokers) → `X.Api` (Minimal APIs). Services may reference `building-blocks/*` and contracts only, **never another service**. Architecture tests enforce this. CI builds only the services that changed, and a change to building-blocks or contracts rebuilds all of them.

Rules that span services and are easy to get wrong:

- **Database per service.** Never read another service's store. Use events, gRPC (internal only, e.g. `CatalogQuery.GetSkus`, `Inventory.CheckStock`, `Promotion.Evaluate`), or REST through the gateway.
- **Store ownership.** MySQL: identity, inventory, ordering, payment, shipping, promotion (one schema each). MongoDB: catalog (+ wishlist), review, notification, analytics. Redis: cart (primary store), caches, rate limits, locks, idempotency keys. Elasticsearch: search only.
- **Each broker has one job.** Kafka carries domain events (past tense, topic per aggregate, key = aggregateId). RabbitMQ carries commands/work jobs (email, image processing) with delayed retry + DLX. SQS (LocalStack locally) carries inbound webhooks (payment FIFO, shipping standard) and report export jobs. The frontend never touches any broker.
- **Transactional Outbox + Inbox everywhere.** Publish only by writing an outbox row in the same transaction as the state change (a Mongo transaction for catalog/review). Consumers dedupe by message id. Event naming: `<context>.<event>.v<n>` (see `docs/04-messaging-and-events.md`).
- **The checkout saga is orchestrated by `ordering`** (state in `saga_state`, timeout worker). Order status changes only through the `Order` aggregate state machine. Inventory reserves all-or-nothing and expires reservations independently.
- **Prices and stock are re-validated synchronously at checkout.** Cache, cart and search copies are for display only.
- **Webhooks:** verify HMAC, enqueue to SQS, return 200. Only the webhook (never the browser redirect) decides the payment outcome.
- **Auth:** the identity service issues RS256 JWTs (15m) plus rotating refresh tokens. The Next.js BFF keeps tokens in httpOnly cookies and refreshes them in `middleware.ts`. The YARP gateway validates JWTs and strips inbound `X-User-*` headers, while the services enforce permission policies (`perm:<resource>:<action>`) and ownership checks.

## Conventions

- Errors are RFC 7807 ProblemDetails with a service-prefixed `code` (`INVENTORY_INSUFFICIENT_STOCK`).
- IDs are app-generated GUID v7/ULID. Money is `decimal` / `DECIMAL(18,2)` + a currency code. Times are UTC.
- `Idempotency-Key` header on POST /orders and payments. `ETag`/`If-Match` on admin updates.
- Every service exposes `/health/live`, `/health/ready` and `/metrics` (OpenTelemetry → Prometheus) via `BuildingBlocks.Observability`. Never use IDs as metric labels. No PII in logs.
- Use MassTransit **v8** only (v9+ is commercial).
- Conventional Commits, with the scope set to the service (backend), `web`/`admin`/`ui` (frontend), or `<service>`/`monitoring`/`platform` (infra). Trunk-based development with PRs to `main` in each repo. Releases use `release-please`, with one version per service.
- Per-feature Definition of Done: see the end of `docs/10-roadmap.md`.
