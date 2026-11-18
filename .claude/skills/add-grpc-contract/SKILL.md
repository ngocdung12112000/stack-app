---
name: add-grpc-contract
description: Add or change an internal gRPC call between StackShop services. Covers the .proto in contracts/protos, the server implementation in the owning service, a typed client with timeout, retry and circuit breaker in the calling service, buf breaking checks, and tests. Use for synchronous service-to-service queries like CatalogQuery.GetSkus, Inventory.CheckStock or Promotion.Evaluate.
---

# Internal gRPC contract

gRPC is **cluster-internal only**. External clients use REST through the gateway. Use it only when the caller needs an answer *now* (checkout re-validation, add-to-cart). Otherwise prefer an event.

## 1. Define the proto

`contracts/protos/<owner>/<service>.proto`:

```proto
syntax = "proto3";
package stackshop.<owner>.v1;
option csharp_namespace = "StackShop.Contracts.<Owner>.V1";

service CatalogQuery {
  rpc GetSkus (GetSkusRequest) returns (GetSkusResponse);
}
message GetSkusRequest { repeated string skus = 1; }
message SkuInfo { string sku = 1; string product_id = 2; string name = 3; string price = 4; /* decimal as string */ bool is_active = 5; }
message GetSkusResponse { repeated SkuInfo items = 1; }
```

Rules:
- The package is versioned (`.v1`). Never reuse or renumber field numbers. Mark removed fields `reserved`.
- Money goes over the wire as a **string decimal** (or units + nanos). Never use `double`.
- Batch requests (repeated IDs) instead of N calls.
- Run `buf lint` and `buf breaking --against main`. A breaking change needs a new `v2` package.

## 2. Server (owning service, Api layer)

`Grpc.AspNetCore`. `MapGrpcService<CatalogQueryService>()`. The service delegates to a MediatR query (same Application logic as REST). Map domain errors to `RpcException` status codes (`NotFound`, `FailedPrecondition`, `InvalidArgument`). Requires an HTTP/2 port (a second Kestrel endpoint, e.g. 8081 h2c inside the cluster).

## 3. Client (calling service, Infrastructure layer)

```csharp
services.AddGrpcClient<CatalogQuery.CatalogQueryClient>(o => o.Address = new(cfg["Services:Catalog:Grpc"]!))
        .AddStandardResilienceHandler(o => {
            o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(1);
            o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(3);
            o.Retry.MaxRetryAttempts = 3;          // queries are idempotent → safe to retry
        });
```

- Wrap it behind an Application-layer port interface (`ICatalogPricing`). Handlers never see generated gRPC types.
- Decide the **degraded behaviour** when the call fails (per the spec: fail fast with 503, or allow with a warning like cart add-to-cart).
- Propagate trace context (automatic with OTel instrumentation) and the user token if the server needs it.

## 4. Tests

- Server: an integration test using `WebApplicationFactory` + a gRPC channel from `factory.CreateDefaultClient()`.
- Client: test the port adapter against a fake server (in-process) or a stub. Cover the timeout/circuit-open path to check the degraded behaviour.

## 5. Handoff

**devops-engineer:** the gRPC port on the server Service (named port `grpc`), a NetworkPolicy allowing caller → server on that port, and the client address config (`Services__Catalog__Grpc=http://catalog:8081`).
