# F17 — API Gateway & Platform Concerns

| Field | Value |
|-------|-------|
| Phase | 0 (routing) → 6 (hardening) |
| Service | `gateway` |
| Stack | .NET 8 + YARP, ASP.NET Core rate limiting with a Redis-backed sliding window, JWT validation via JWKS, OpenTelemetry |
| Depends on | F01 |

## 1. Responsibilities

| # | Capability | Detail |
|---|------------|--------|
| 1 | Routing | YARP routes and clusters from config (`appsettings.json` → ConfigMap). Path-prefix routing per [05-api-design.md §2](../05-api-design.md) |
| 2 | AuthN | Validates the JWT (issuer, audience, lifetime, signature via cached JWKS). Checks the `jti` blacklist in Redis (L1 in-memory cache 30s) |
| 3 | Coarse AuthZ | Route-level policies (`anonymous`, `authenticated`, `admin`). Fine-grained checks stay in the services |
| 4 | Rate limiting | Policies per route group (see the API design doc). Redis sliding window via a sorted set or the `RedisRateLimiting` library. Returns 429 + `Retry-After` |
| 5 | Request ID & tracing | Generates `X-Request-Id` if missing. Starts or continues the W3C trace. Forwards headers |
| 6 | Header hygiene | Strips inbound `X-User-*` headers (prevents spoofing). Optionally adds `X-User-Id` after validation |
| 7 | CORS | Allows only the web/admin origins |
| 8 | WebSockets | Proxies `/hubs/**` to notification with sticky-free SignalR (Redis backplane) |
| 9 | Webhooks passthrough | `/payments/webhooks/**`, `/shipments/webhooks/**` skip JWT. Services verify HMAC. IP allow-list optional |
| 10 | Resilience | Per-cluster timeouts, passive health checks (eject failing destinations), active health checks on `/health/ready` |
| 11 | Response compression | Brotli/gzip |
| 12 | API docs | Aggregated Swagger UI listing every service's OpenAPI document |
| 13 | Metrics | `http_server_request_duration_seconds` with `route` and `service` labels. `stackshop_rate_limited_total{policy}` |

## 2. Example YARP config

```json
{
  "ReverseProxy": {
    "Routes": {
      "catalog-public": {
        "ClusterId": "catalog",
        "Match": { "Path": "/api/v1/{**rest}", "Methods": ["GET"] },
        "Order": 100,
        "Metadata": { "service": "catalog" },
        "AuthorizationPolicy": "anonymous",
        "RateLimiterPolicy": "anonymous"
      },
      "orders": {
        "ClusterId": "ordering",
        "Match": { "Path": "/api/v1/orders/{**rest}" },
        "AuthorizationPolicy": "authenticated",
        "RateLimiterPolicy": "checkout"
      }
    },
    "Clusters": {
      "ordering": {
        "Destinations": { "d1": { "Address": "http://ordering" } },
        "HttpRequest": { "ActivityTimeout": "00:00:10" },
        "HealthCheck": { "Active": { "Enabled": true, "Path": "/health/ready", "Interval": "00:00:10" } }
      }
    }
  }
}
```

(In K8s, the destination is the Service DNS name, and kube-proxy load-balances across pods.)

## 3. Platform checklist (applies to every service)

- [ ] `/health/live` (process up) and `/health/ready` (DB, broker, cache reachable) via `AspNetCore.HealthChecks.*` packages.
- [ ] Graceful shutdown: handle SIGTERM, stop consumers, finish in-flight requests (`HostOptions.ShutdownTimeout = 25s`, < K8s `terminationGracePeriodSeconds=30`).
- [ ] Structured logs + `/metrics` + traces (BuildingBlocks.Observability).
- [ ] Config validated at startup (`ValidateOnStart` with options classes).
- [ ] ProblemDetails + global exception handler.
- [ ] OpenAPI document with auth scheme.
- [ ] Resilience pipelines on all outbound calls.
- [ ] Non-root container, read-only filesystem.

## 4. Acceptance criteria

- AC-1: 61 anonymous requests in 1 minute from one IP → the 61st gets 429 with `Retry-After`.
- AC-2: The rate limit is shared across 3 gateway replicas (Redis-backed), not per pod.
- AC-3: A request with a forged `X-User-Id` header has it stripped before reaching the service.
- AC-4: A token signed by an unknown key → 401 at the gateway, and the service is never called.
- AC-5: When one ordering pod returns 5xx consistently, YARP passive health checks eject it (on a multi-destination setup).
- AC-6: `traceparent` from Next.js is continued through the gateway into the service span.

## 5. Learning focus

Reverse proxies, edge auth vs service auth, distributed rate limiting, health checks and graceful shutdown in K8s, header security.
