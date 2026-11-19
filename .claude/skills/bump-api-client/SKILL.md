---
name: bump-api-client
description: Upgrade the @ngocdung12112000/api-client npm package in stackshop-frontend to a new version published by the backend, fix resulting type errors, update MSW mocks, and verify. Also covers generating a temporary local client from a running gateway for unreleased APIs. Use after a backend handoff mentions a new client version, when Renovate opens a bump PR, or when frontend types don't match the API.
---

# Bump `@ngocdung12112000/api-client`

The frontend's **only** link to backend types is this package (GitHub Packages, pinned version). Never hand-write DTOs that the client already provides, and never copy backend code.

## 1. Upgrade

```bash
cd stackshop-frontend
pnpm view @ngocdung12112000/api-client versions --json | tail -5    # check what's published (needs .npmrc auth to npm.pkg.github.com)
pnpm up -r @ngocdung12112000/api-client@<version>                   # every app/package that uses it
```

Read the backend handoff or changelog for the version. A **major** bump means breaking changes, so expect type errors.

## 2. Fix the compile errors

```bash
pnpm typecheck
```

Work through every error:
- **Renamed/removed field:** update usages. If the old field is still needed in the UI, ask the backend. They should have kept it for one release (expand → migrate → contract).
- **New required request field:** add it to the zod schema and the form (`server-action-form` skill).
- **New error codes:** add next-intl messages under `errors.<CODE>` (en + vi) and handle any code that needs special UX.
- **New enum values** (e.g. an order status): update exhaustive `switch` statements, `OrderStatusBadge` and filters. Keep a `never` exhaustiveness check so future values fail at compile time.

## 3. Update the mocks

- MSW handlers in `apps/*/mocks` and fixtures: update response shapes to the new types (they're typed, so `tsc` points at the stale ones).
- Add handlers for new endpoints used by the UI.

## 4. Verify

```bash
pnpm lint && pnpm typecheck && pnpm test && pnpm build
pnpm --filter e2e test:smoke     # Playwright smoke against MSW
```

If a local backend is running, click through the affected pages against the real gateway (`local-env` skill).

## Unreleased API (local only)

```bash
pnpm gen:api --from http://localhost:5000/swagger/v1/swagger.json   # writes a local override client
```

Use it only to develop against a backend branch. **Don't commit the override.** Switch back to the published version before opening the PR, and merge only after the backend has published the release (merge order: backend → client publish → frontend).

## Commit

Commit (only if the user asked) with `chore(web): bump @ngocdung12112000/api-client to <version>`, or `feat(...)` if UI changes are included, and reference the spec ID.
