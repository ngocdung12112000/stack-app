# ADR-001: Split into frontend, backend and infra repositories; backend as microservices

- **Status:** Accepted
- **Date:** 2026-11-17
- **Related features:** all

## Context

The first plan used a single monorepo with frontend, backend and deployment config side by side. The owner wants the frontend and backend fully separated, and the backend built as microservices. The project is for learning, so the setup should look like how many real teams split responsibilities.

## Options considered

1. **One repo with `frontend/` and `backend/` roots.** Easiest to keep in sync, but FE and BE still share history, CI and releases.
2. **Two repos (frontend, backend).** Deployment manifests would have to live in one of them, or be split between them.
3. **Three repos: frontend, backend, infra (GitOps).** Each app repo only builds images. The infra repo is the single desired state that ArgoCD applies.
4. **One repo per microservice** (15+ repos). Maximum independence, but too much overhead for one learner.

## Decision

Option 3. The backend repo holds **all** microservices, and each service is still built, versioned and deployed independently. The services are microservices from Phase 1. There is no modular-monolith stage.

## Consequences

- Positive: independent FE/BE releases. Teaches GitOps (image-tag PRs, ArgoCD), versioned API clients and backward-compatible API evolution. Infra changes (alerts, scaling) don't need an app release.
- Negative: API changes need two steps (backend publishes the client, the frontend upgrades it). Local dev needs three checkouts and a shared Docker network. E2E tests must target a deployed environment or a full local stack.
- Follow-up: create a GitHub App (or fine-grained PAT) that lets FE/BE workflows open PRs in `stackshop-infra`. Set up GitHub Packages for `@ngocdung12112000/api-client`. Set up Renovate in the frontend repo.
