---
name: k8s-service-manifests
description: Create or update the Kubernetes manifests for a StackShop service or Next.js app in stackshop-infra. Covers the Kustomize base (Deployment, Service, HPA, PDB, NetworkPolicy, ServiceMonitor, ConfigMap, Secret refs, EF migration Job, optional worker Deployment), dev/staging/prod overlays and the ArgoCD Application, validated with kustomize build + kubeconform + kube-linter. Use when a new service is added, a service needs new config/ports/workers, or resources/scaling change.
---

# K8s manifests for a service

Location: `stackshop-infra/k8s/apps/<svc>/`. App repos never contain manifests. Requirements come from `docs/07-devops-cicd.md` §3 and the backend handoff (port, env vars, secrets, health paths, worker yes/no, gRPC port, migration command).

## 1. Base (`k8s/apps/<svc>/base/`)

| File | Must contain |
|---|---|
| `deployment.yaml` | image `ghcr.io/<owner>/stackshop-<svc>` (tag set by the overlay), port 8080 (+ `grpc` 8081 if needed); probes: liveness `/health/live`, readiness `/health/ready`, startup (failureThreshold 30, period 2s); `resources.requests` cpu 100m / mem 192Mi, `limits.memory` 384Mi (no CPU limit); `securityContext` runAsNonRoot, readOnlyRootFilesystem, allowPrivilegeEscalation false, drop ALL caps; an `emptyDir` at `/tmp`; `strategy` RollingUpdate maxUnavailable 0 / maxSurge 1; `terminationGracePeriodSeconds: 30`; `envFrom` ConfigMap + Secret; labels `app.kubernetes.io/{name,part-of=stackshop,component=api}` |
| `deployment-worker.yaml` (if consumers) | same image, `Mode=Worker` env, component=worker, **replicas ≤ topic partitions**, no Service |
| `service.yaml` | ClusterIP 80→8080 (named `http`), `grpc` port if needed |
| `hpa.yaml` | min 2 / max 5 (staging/prod), CPU 70%. Workers: KEDA on lag is a stretch goal |
| `pdb.yaml` | `minAvailable: 1` |
| `networkpolicy.yaml` | ingress only from the gateway pods (+ the named gRPC callers + the monitoring namespace for `/metrics`). Egress to the `data` namespace + DNS |
| `servicemonitor.yaml` | scrape `/metrics` on `http` every 15s, label `release: kube-prometheus-stack` |
| `configmap.yaml` | non-secret config (`ASPNETCORE_ENVIRONMENT`, broker hosts, `Services__X__Grpc`) |
| `sealedsecret.yaml` / `externalsecret.yaml` | connection strings, keys. **Never a plain Secret with real values in git** |
| `migration-job.yaml` (MySQL services) | runs the EF bundle; ArgoCD annotations `argocd.argoproj.io/hook: PreSync`, `sync-wave: "-1"`, `hook-delete-policy: BeforeHookCreation`; backoffLimit 2 |
| `kustomization.yaml` | lists the above, `commonLabels`, `images:` entry |

Next.js apps: same pattern, port 3000, health `/api/health`, plus an Ingress (`shop.` / `admin.stackshop.local`). The gateway gets an Ingress at `api.stackshop.local` with WebSocket support for `/hubs`.

## 2. Overlays (`overlays/{dev,staging,prod}/`)

- `kustomization.yaml`: `resources: [../../base]`, `images: [{name: ..., newTag: <sha-or-semver>}]` (**the only line CI bumps**), namespace `stackshop`.
- dev: replicas 1, smaller requests, HPA off. staging: 2 replicas. prod: 2–3 replicas, larger limits, PDB on.
- Env-specific ConfigMap patches (log level, hosts).

## 3. ArgoCD

Add `argocd/apps/<svc>-<env>.yaml` (or a generator entry in the app-of-apps): path `k8s/apps/<svc>/overlays/<env>`, automated sync with prune + selfHeal for dev, **manual sync for prod**.

## 4. Validate (required)

```bash
for env in dev staging prod; do
  kustomize build k8s/apps/<svc>/overlays/$env | kubeconform -strict -summary -ignore-missing-schemas
done
kustomize build k8s/apps/<svc>/overlays/dev | kube-linter lint -
```

If a kind/k3d cluster exists, apply dev and check `kubectl -n stackshop rollout status deploy/<svc>` + `curl` through the ingress → gateway → service. Don't apply to shared clusters without the user's permission.

## 5. Handoff

**backend-engineer:** anything missing to satisfy the manifests (health endpoint, graceful shutdown, `Mode=Worker`, writable path needs under a read-only root filesystem).
