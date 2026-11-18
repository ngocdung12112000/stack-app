---
name: ci-workflow
description: Create or modify a GitHub Actions workflow in any StackShop repo (frontend, backend, infra) following the documented pipeline. Covers changed-service matrices, caching, least-privilege permissions, image build/scan/SBOM, cross-repo tag-bump PRs into stackshop-infra, release-please, repository_dispatch for E2E, and validation with actionlint. Use when adding a pipeline step, a new service to CI, or fixing a broken workflow.
---

# GitHub Actions workflow

The workflow inventory per repo is in `docs/07-devops-cicd.md` §4.1. Don't invent new workflows when an existing one should be extended.

## Conventions (every workflow)

- `permissions:` at the top level set to `contents: read`. Grant more per job only (`packages: write` for GHCR/npm, `pull-requests: write`, `security-events: write` for CodeQL, `id-token: write` for OIDC/cosign).
- `concurrency: { group: ${{ github.workflow }}-${{ github.ref }}, cancel-in-progress: true }` for PR workflows.
- Pin actions to a major version (`actions/checkout@v4`) or a full SHA for third-party actions.
- Caching: `actions/setup-dotnet` + NuGet cache keyed on `**/Directory.Packages.props` / `packages.lock.json`; `pnpm/action-setup` + `actions/setup-node` with `cache: pnpm`; Turborepo cache; Buildx `cache-from/to: type=gha,scope=<image>`.
- `timeout-minutes` on every job. Upload test results (`dorny/test-reporter` or a job summary) and coverage.
- Secrets only through `secrets.*`, never echoed. Gitleaks runs in `security.yml`.

## Backend specifics

- `ci.yml`: a `changes` job using `dorny/paths-filter` that outputs the changed service list. **A change under `building-blocks/**`, `contracts/**`, `Directory.*.props` = all services.** The matrix job over the list runs restore → build → unit → arch → integration (Testcontainers; Docker is available on `ubuntu-latest`) → `dotnet format --verify-no-changes`.
- `contracts.yml`: JSON Schema validation, `bufbuild/buf-action` (lint + breaking against `main`), and `oasdiff breaking` against the committed OpenAPI snapshot. Fail unless the PR has the `breaking-api` label.
- `publish-client.yml`: generate the TS client, then `npm publish` to `npm.pkg.github.com` with `GITHUB_TOKEN` (`packages: write`).

## Image build + GitOps bump (backend & frontend)

```yaml
- uses: docker/build-push-action@v6
  with: { context: ., file: services/${{ matrix.svc }}/Dockerfile, push: true,
          tags: ghcr.io/${{ github.repository_owner }}/stackshop-${{ matrix.svc }}:${{ github.sha }},
          cache-from: type=gha,scope=${{ matrix.svc }}, cache-to: type=gha,mode=max,scope=${{ matrix.svc }},
          provenance: true, sbom: true }
- uses: aquasecurity/trivy-action@<ver>     # severity CRITICAL, exit-code 1
- uses: actions/create-github-app-token@v1  # app-id / private-key secrets → token scoped to stackshop-infra
- run: |  # checkout infra with that token, then:
    cd k8s/apps/${{ matrix.svc }}/overlays/dev && kustomize edit set image ghcr.io/...=...:${{ github.sha }}
- uses: peter-evans/create-pull-request@v7   # branch bump/<svc>-<sha>, labels: automerge
```

App repos **never** use a kubeconfig or run kubectl/helm. Deployment happens only through the infra PR → ArgoCD.

## Infra specifics

- `validate.yml`: `kustomize build` every overlay → `kubeconform -strict` → `kube-linter` → `promtool check rules` → `jq` dashboards → post the rendered diff as a PR comment. Auto-merge `automerge`-labelled dev bumps when green.
- `promote.yml` (`workflow_dispatch` inputs: service, from, to): copy the tag and open a PR. Prod PRs need CODEOWNERS review + the `production` environment.
- `post-sync.yml` (`repository_dispatch` from the ArgoCD notification): dispatch `deployed` to the frontend repo for E2E, and run the k6 smoke.

## Frontend specifics

`ci.yml`: lint → typecheck → vitest → `next build` → Playwright smoke (MSW) → Lighthouse CI. `e2e.yml` on `repository_dispatch: [deployed]` with `client_payload.env` → full Playwright against that env.

## Validate (required)

```bash
actionlint .github/workflows/*.yml
```

Then explain how it was (or will be) exercised: a PR run link, or `act` for simple jobs. Report anything that can only be verified on GitHub (cross-repo tokens, environments, packages).
