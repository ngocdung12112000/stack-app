---
name: promote-release
description: Promote a StackShop service or frontend app image between environments (dev → staging → prod) through a GitOps PR in stackshop-infra, check health after ArgoCD sync, and roll back safely. Use when the user says "promote ordering to staging", "release to prod", "roll back catalog", or a deployment is failing.
---

# Promote and roll back

All deployments are **git changes in `stackshop-infra`**. ArgoCD applies them. There are no `kubectl set image` or manual `helm upgrade` calls against shared clusters.

## Promote

1. **Pick the version.** Read the tag currently in `k8s/apps/<svc>/overlays/<from>/kustomization.yaml` (`images[].newTag`). For prod, prefer a released semver tag (`1.4.0`, created by `release.yml`) over a raw sha.
2. **Pre-checks** (report each one):
   - The `<from>` environment has run this tag healthily (ArgoCD `Synced/Healthy`, no firing alerts for the service, E2E green for that deploy).
   - The image scan passed (Trivy) for that tag.
   - DB migrations: does this version include a migration? Is it expand-only (safe), or does it contract (drop/rename)? A contract migration needs every running consumer of the old schema gone first.
   - Cross-repo order: if the frontend release needs this backend version, promote the **backend first** (backend → frontend → infra config).
3. **Open the PR:** run `promote.yml` (`gh workflow run promote.yml -f service=<svc> -f from=staging -f to=prod`) or edit the `<to>` overlay `newTag` on a branch `promote/<svc>-<to>-<tag>`. The PR description includes the tag, changelog link, migrations y/n, the pre-check results and the rollback plan.
4. **Prod:** the PR needs CODEOWNERS approval and the `production` environment. **Never merge a prod promotion without the user's explicit OK.**
5. **After the merge:** watch the ArgoCD sync (`argocd app get <svc>-<env>` / UI) until it's `Healthy`, then check:
   - `kubectl -n stackshop rollout status deploy/<svc>` (read-only)
   - the Grafana service-detail dashboard: error rate and p95 aren't worse than before, no new restarts
   - post-sync E2E/k6 results
   Report it as done only after these checks.

## Roll back

1. **Fastest safe path:** `git revert` the promotion commit in `stackshop-infra` → PR → merge (for prod, still get the user's OK, but treat it as urgent). ArgoCD syncs the previous tag.
2. ArgoCD's own "rollback to revision" is only a temporary emergency measure. Follow it with the git revert so git stays the source of truth (otherwise selfHeal re-applies the bad version).
3. **Migrations:** expand-only migrations don't need reverting (the old code tolerates the new columns). If a contract migration ran, **don't** roll back the code blindly. Tell the user, because the data may need restoring (`docs/03-data-architecture.md` §7 backups).
4. Afterwards: open an issue with a timeline, the cause, which alert fired (or that none did → add one with `add-dashboard-alert`), and the follow-up fix.

## Rules

- Never promote untested tags (only tags that already ran in the previous environment).
- Promote one service at a time unless the services are coupled (then use one PR listing all of them, in dependency order).
- Report clearly what was merged, what is pending approval, and what is verified versus assumed.
