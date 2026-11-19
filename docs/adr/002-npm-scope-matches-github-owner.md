# ADR-002: API client npm scope matches the GitHub owner

- Status: Accepted
- Date: 2026-11-18

## Context
ADR-001 names the backend-to-frontend contract package `@stackshop/api-client`. GitHub Packages only accepts an npm
scope equal to the owner (user or organisation) of the repository, and the repos live under the user `ngocdung12112000`.

## Decision
Publish the package as `@ngocdung12112000/api-client`. Internal, unpublished workspace packages in the frontend
(`@stackshop/web`, `@stackshop/ui`, `@stackshop/config`, `@stackshop/e2e`) keep the `@stackshop` scope.

## Consequences
- `.npmrc` maps `@ngocdung12112000:registry` to GitHub Packages; workflows use `scope: '@ngocdung12112000'`.
- Moving the repos to a GitHub organisation later means renaming the package again (a breaking change for consumers).
