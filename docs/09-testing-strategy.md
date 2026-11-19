# 09 — Testing Strategy

Which repo owns which tests: **backend** owns unit, architecture, integration, contract and load (k6 in `tests/load`). **Frontend** owns component tests and Playwright E2E (`tests/e2e`), with E2E triggered by infra after each deployment. **Infra** owns manifest validation (kubeconform, promtool).

| Level | Scope | Tools | Where it runs | Target |
|-------|-------|-------|---------------|--------|
| Unit | Domain logic, handlers, validators, React components, zod schemas | xUnit, FluentAssertions, NSubstitute / Vitest, Testing Library | Every PR | Domain + Application ≥ 80% line coverage |
| Architecture | Layer dependency rules (Domain has no refs to Infrastructure) | NetArchTest | Every PR | 100% pass |
| Integration | API + real DB/broker in containers | `WebApplicationFactory` + **Testcontainers** (MySQL, MongoDB, Redis, Kafka, RabbitMQ, LocalStack, Elasticsearch) | Every PR (changed services) | Every endpoint's happy path + key errors |
| Contract | Event schemas between services. HTTP contract between backend and frontend | JSON Schema validation tests, `oasdiff` breaking-change check on OpenAPI, the frontend `tsc` against the published `@ngocdung12112000/api-client`, (stretch) Pact consumer tests from the frontend | Every PR | All published events and public endpoints |
| E2E | Critical user journeys through the browser | Playwright | Post-deploy to staging, nightly | Smoke: browse → search → cart → checkout → pay → order visible |
| Load | Throughput and latency under load | k6 (scripts in `tests/load`), Grafana k6 dashboard | Manual / weekly | 200 RPS browse, 20 checkouts/s with p95 < 500ms |
| Chaos (stretch) | Resilience | Kill pods, pause Kafka, inject latency (Chaos Mesh / toxiproxy) | Manual | Saga completes or compensates correctly |

## Critical E2E journeys

1. Guest browses a category, filters by brand and price, and opens a product.
2. Guest adds to cart → registers → the cart merges.
3. Customer checks out with a coupon → mock payment succeeds → order shows `Paid` → email in Mailpit.
4. Payment fails → order is cancelled → stock is released (verify via admin).
5. Admin creates a product → it becomes searchable within 5s.
6. Customer reviews a delivered product → the review is moderated → the rating updates on the PDP.
7. Admin exports a sales report → download link appears.

## Test data

- Builders / Object Mothers in `BuildingBlocks.Testing`.
- Deterministic clock (`TimeProvider` from .NET 8 + `FakeTimeProvider`).
- E2E runs against a seeded environment that is reset with the Seeder `--reset`.
