---
name: add-dashboard-alert
description: Add or change Grafana dashboard panels and Prometheus alert rules for StackShop in stackshop-infra/monitoring, plus scrape config and exporters. Covers PromQL, provisioned dashboard JSON, PrometheusRule CRs, SLO burn-rate alerts, and validation with promtool. Use when a service adds a business metric, a spec or docs/08-observability.md lists a dashboard/alert/SLO, or the user asks to monitor something.
---

# Dashboards and alerts

Source of truth: `docs/08-observability.md` (metric names §3, dashboards §4, alerts §5, SLOs §6). Metrics are emitted by app code. **Dashboards and rules live only in `stackshop-infra/monitoring/`.**

## 1. Confirm the metric exists

- Get the exact metric name, type and labels from the backend handoff or `/metrics` (`curl localhost:<port>/metrics | grep stackshop_`). OTel names histograms like `http_server_request_duration_seconds_bucket`.
- **Reject unbounded labels** (userId, orderId, sku, email). If a metric has one, hand it back to backend-engineer.
- Make sure it's scraped: ServiceMonitor for K8s (`k8s-service-manifests`), `local/prometheus/prometheus.yml` job for compose. Exporter-based metrics (mysqld, mongodb, redis, kafka-exporter, rabbitmq, elasticsearch) need the exporter deployed (kube-prometheus-stack values / compose).

## 2. Dashboard panel

- Edit `monitoring/dashboards/<dashboard>.json`. One file per dashboard from §4: platform-overview, service-detail (templated `$service`), business-kpis, messaging, datastores, frontend.
- Conventions:
  - datasource as a variable `${datasource}`
  - `$service` / `$env` template variables
  - RED panels per service: `sum(rate(...count[5m]))`, the error % ratio, `histogram_quantile(0.95, sum by (le) (rate(..._bucket[5m])))`
  - units set (s, reqps, percentunit, currency)
  - a description on every panel
- Keep stable `uid`s. Increment `version`. Prefer editing in the Grafana UI, then exporting JSON ("Export for sharing externally" off), and strip `id` fields.
- Local provisioning: `local/grafana/provisioning/dashboards/` points at the same JSON folder. K8s: the dashboards go into ConfigMaps with the label `grafana_dashboard: "1"` (via kustomize `configMapGenerator`).

## 3. Alert rule

`monitoring/rules/<area>.rules.yaml` as a `PrometheusRule` CR (`release: kube-prometheus-stack` label):

```yaml
- alert: KafkaConsumerLagHigh
  expr: sum by (consumergroup) (kafka_consumergroup_lag) > 1000
  for: 10m
  labels: { severity: warning, team: stackshop }
  annotations:
    summary: "Consumer group {{ $labels.consumergroup }} lag > 1000"
    description: "Lag {{ $value }} for 10m. Check consumer pods / DLQ."
    runbook_url: "https://github.com/<owner>/stackshop-infra/blob/main/monitoring/runbooks/kafka-lag.md"
    dashboard: "messaging"
```

- Severity: `critical` (pages: user impact now) or `warning` (needs a look today). Always set a `for:` to avoid flapping.
- Every new alert gets a short runbook in `monitoring/runbooks/<name>.md` (what it means, how to check, how to fix).
- SLO alerts: multi-window burn rate (e.g. 1h/5m at 14.4×, 6h/30m at 6×), or generate them with Sloth/Pyrra (stretch goal).
- Alertmanager routes (`values-kube-prometheus-stack.yaml`): Mailpit locally, Slack/Discord webhook in the cloud (the URL in a secret).

## 4. Validate (required)

```bash
promtool check rules <(yq '.spec' monitoring/rules/*.rules.yaml)      # or extract groups per file
promtool test rules monitoring/rules/tests/*.yaml                     # unit tests for important alerts
jq empty monitoring/dashboards/*.json                                 # valid JSON
```

Locally: reload Prometheus (`curl -X POST localhost:9090/-/reload`). The alert appears under /alerts and the panel renders data in Grafana :3002. To prove an alert fires, trigger the condition if it's safe (stop a consumer → lag climbs).

## 5. Update docs

If you added a metric/alert/dashboard not listed in `docs/08-observability.md`, add it there (`sync-docs-adr`).
