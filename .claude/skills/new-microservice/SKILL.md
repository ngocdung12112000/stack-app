---
name: new-microservice
description: Scaffold a new .NET 8 microservice in stackshop-backend following StackShop's Clean Architecture template (Domain/Application/Infrastructure/Api), with health checks, metrics, Swagger, Dockerfile, tests, a gateway route and the no-cross-service-reference architecture test. Use when a roadmap phase or feature spec introduces a service (identity, catalog, search, inventory, cart, ordering, payment, shipping, notification, review, promotion, media, analytics, mock-psp).
---

# New microservice

Check first: the service is listed in `docs/02-architecture.md` §3 (service catalogue) and its phase has arrived (`docs/10-roadmap.md`). Note its **store(s)**, **published/consumed events** and **sync APIs** from that table.

## 1. Create the structure

Prefer the template once it exists: `dotnet new stackshop-service -n <Name> -o services/<name>`. Otherwise create it by hand and **copy the conventions from an existing service** (e.g. `catalog`):

```
services/<name>/
├── <Name>.sln
├── Dockerfile                       # multi-stage; context = backend repo root (docs/07 §2)
├── src/
│   ├── <Name>.Domain/               # entities, value objects, domain events — NO package refs except BuildingBlocks.Common
│   ├── <Name>.Application/          # MediatR handlers, FluentValidation, interfaces (ports)
│   ├── <Name>.Infrastructure/       # persistence, messaging, external clients, DI extension AddInfrastructure()
│   └── <Name>.Api/                  # Program.cs, endpoint groups, auth, health, metrics
└── tests/
    ├── <Name>.UnitTests/
    ├── <Name>.IntegrationTests/     # WebApplicationFactory + Testcontainers fixtures from BuildingBlocks.Testing
    └── <Name>.ArchitectureTests/    # NetArchTest
```

Project references only point inward: Api → Infrastructure → Application → Domain. Packages come from `Directory.Packages.props` (no versions in csproj).

## 2. Wire up Program.cs (minimum)

- `AddStackShopObservability(serviceName)` → OTel metrics/traces, Serilog JSON, `app.MapPrometheusScrapingEndpoint()`
- `AddStackShopAuthentication()` (JWT via JWKS) + permission policy provider
- Health checks: `/health/live` (no dependency checks) and `/health/ready` (checks this service's DB/cache/broker only)
- ProblemDetails + global exception handler, API versioning `/api/v1`, Swagger with a bearer scheme
- Options classes with `ValidateOnStart()`. Config from `appsettings.json` + env vars
- Graceful shutdown: `HostOptions.ShutdownTimeout = 25s`
- If it has consumers: support `Mode=Worker` (config switch) so the same image can run as API or worker

## 3. Store and messaging setup

Only the stores the catalogue assigns to this service:
- MySQL: DbContext, schema `<name>_db`, outbox/inbox tables, first migration (`db-migration` skill)
- MongoDB: collections + indexes created idempotently at startup, the Mongo outbox
- Redis: `IConnectionMultiplexer` + key prefix from `docs/03-data-architecture.md` §4

## 4. Mandatory tests

- An architecture test asserting:
  - Domain has no dependency on Application/Infrastructure/Api or EF/Mongo/Redis packages
  - Application has no dependency on Infrastructure
  - **No reference to any other `services/*` assembly**
- An integration smoke test: `/health/ready` returns 200 with Testcontainers for its stores.

## 5. Connect it to the rest of the system

- Gateway: add the route(s) + cluster to `services/gateway` config per `docs/05-api-design.md` §2, with an auth policy and rate-limit policy.
- `docker-compose.yml` (backend repo): add the service on the external `stackshop` network, with env vars and a healthcheck.
- CI: make sure the changed-service matrix in `.github/workflows/ci.yml` picks up `services/<name>/**` (and `build-images.yml`).
- `release-please` config: add the package `services/<name>`.

## 6. Verify

`dotnet build services/<name>/<Name>.sln` and `dotnet test services/<name>/<Name>.sln`, then `docker compose up -d --build <name>` → `curl localhost:5000/...` through the gateway → `/metrics` responds.

## 7. Handoff to devops-engineer (always)

New service name and port 8080, image name `stackshop-<name>`, env vars/secrets, DB schema + user, topics/queues it needs, migration Job command, health paths, worker Deployment needed (y/n), ingress needed (no; only via the gateway).
