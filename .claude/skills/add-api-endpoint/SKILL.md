---
name: add-api-endpoint
description: Add or change a REST endpoint in a StackShop .NET service the standard way. Covers the Minimal API route, MediatR command/query, FluentValidation, ProblemDetails error codes, permission and ownership checks, idempotency/ETag where required, OpenAPI annotations, gateway route, tests named after spec IDs, and publishing a new @stackshop/api-client. Use whenever a feature spec's API table needs implementing or an endpoint changes.
---

# Add / change an API endpoint

Source of truth: the feature spec's **API** table + `docs/05-api-design.md` (conventions §1, routes §2, errors §3, auth §5).

## 1. Design check

- Path `/api/v1/<plural-kebab-resource>`, the correct verb, camelCase JSON, pagination style (cursor for feeds, offset for admin tables).
- Auth: anonymous / authenticated / a permission (`perm:<resource>:<action>`). **Ownership**: when the caller doesn't own the resource, return **404**, not 403.
- Does it need an `Idempotency-Key` (POST orders/payments) or `ETag`/`If-Match` (admin updates → 412)?
- Is it a **breaking** change to an existing endpoint? If yes, use expand → migrate → contract: add the new field/endpoint, keep the old one until the frontend has moved, or create `/api/v2`. Never remove or rename a field in one step.

## 2. Implement (per layer)

1. **Application:** a `record XxxCommand/Query : IRequest<Result<XxxResponse>>` + handler + `AbstractValidator<XxxCommand>`. The business rules (BR-n) live in the Domain and are called from the handler.
2. **Errors:** return `Result.Failure(Error.Conflict("ORDERING_INVALID_TRANSITION", "..."))`. The service-prefixed SCREAMING_SNAKE code is mapped to ProblemDetails by the shared filter. Add any new codes to the spec's error list.
3. **Api:** in the endpoint group file:
   ```csharp
   group.MapPost("/", async (PlaceOrderRequest req, ISender sender, CancellationToken ct) => ...)
        .RequireAuthorization("perm:orders:create")
        .WithName("PlaceOrder")
        .Produces<PlaceOrderResponse>(StatusCodes.Status202Accepted)
        .ProducesProblem(409).ProducesValidationProblem()
        .WithSummary("...").WithTags("Orders");
   ```
   Keep endpoints thin: map the request to a command, send it, map the Result to a response.
4. **Gateway:** add or adjust the YARP route if this is a new path prefix (auth + rate-limit policy).
5. **Observability:** add the business metric if the spec names one. Log business events at Information, with no PII.

## 3. Tests

- Unit: validator cases + handler logic (each BR-n touched).
- Integration (`WebApplicationFactory` + Testcontainers): happy path + each documented error (status + `code`) + auth (401/403) + ownership (404) + idempotency/ETag if applicable.
- Name them after spec IDs: `F07_AC2_SameIdempotencyKey_ReturnsSameOrder`.

## 4. Contract and client

1. Regenerate OpenAPI → `contracts/openapi/<service>.json` (CI diffs it with `oasdiff`). Breaking → the PR needs the `breaking-api` label and a justification.
2. Bump the `@stackshop/api-client` version (minor for additions, major for breaking) so `publish-client.yml` publishes it.

## 5. Handoff

- **frontend-engineer:** endpoint, request/response shape, error codes the UI must handle, the client version to bump to (`bump-api-client` skill).
- **devops-engineer:** only if there's a new gateway path needing ingress/WebSocket, a new rate limit, or new config/secrets.
