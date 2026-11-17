# 08 — Observability (Prometheus + Grafana)

## 1. Three pillars

| Pillar | Tooling | Required? |
|--------|---------|-----------|
| **Metrics** | OpenTelemetry .NET → Prometheus exporter (`/metrics`) → Prometheus → Grafana | ✅ Core |
| **Logs** | Serilog JSON to stdout → (optional) Loki + Promtail/Alloy → Grafana | Optional (recommended) |
| **Traces** | OpenTelemetry → OTLP → (optional) Tempo or Jaeger → Grafana | Optional (recommended) |

The stack is installed with **kube-prometheus-stack** (Prometheus Operator, Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics). Locally, Compose runs the plain Prometheus + Grafana containers with a static `prometheus.yml`.

## 2. Standard instrumentation (BuildingBlocks.Observability)

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService(serviceName, serviceVersion: version))
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()          // GC, threadpool, exceptions
        .AddProcessInstrumentation()
        .AddMeter("StackShop.*")              // custom business meters
        .AddPrometheusExporter())
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddRedisInstrumentation()
        .AddSource("StackShop.Messaging")     // manual spans for Kafka/RabbitMQ/SQS produce/consume
        .AddOtlpExporter());
app.MapPrometheusScrapingEndpoint();          // GET /metrics
```

Exporters for infrastructure:

| Component | Exporter |
|-----------|----------|
| MySQL | `prom/mysqld-exporter` |
| MongoDB | `percona/mongodb_exporter` |
| Redis | `oliver006/redis_exporter` |
| Kafka | Strimzi built-in (JMX) + `kafka-exporter` for consumer lag |
| RabbitMQ | Built-in `rabbitmq_prometheus` plugin (port 15692) |
| Elasticsearch | `elasticsearch-exporter` |
| NGINX ingress | Built-in metrics |
| K8s | kube-state-metrics, node-exporter, cAdvisor |

## 3. Custom business metrics

| Metric | Type | Labels | Emitted by |
|--------|------|--------|-----------|
| `stackshop_orders_created_total` | counter | – | ordering |
| `stackshop_orders_status_total` | counter | status | ordering |
| `stackshop_order_value` | histogram (buckets 10…5000) | currency | ordering |
| `stackshop_checkout_saga_duration_seconds` | histogram | outcome (paid/cancelled) | ordering |
| `stackshop_payments_total` | counter | outcome, provider | payment |
| `stackshop_cart_operations_total` | counter | op (add/update/remove/merge) | cart |
| `stackshop_stock_reservations_total` | counter | outcome | inventory |
| `stackshop_search_requests_total` / `_duration_seconds` | counter / histogram | has_results | search |
| `stackshop_emails_sent_total` | counter | template, outcome | notification |
| `stackshop_cache_requests_total` | counter | cache, result (hit/miss) | catalog, gateway |
| `stackshop_outbox_pending` | gauge | service | all MySQL services |
| `stackshop_messages_consumed_total` | counter | broker, topic/queue, outcome | all consumers |
| `stackshop_message_handle_duration_seconds` | histogram | broker, topic/queue | all consumers |

Label rule: **never** put unbounded values (userId, orderId, sku) in labels.

## 4. Grafana dashboards (provisioned as JSON in `stackshop-infra/monitoring/dashboards/`)

Ownership: services emit metrics (code in `stackshop-backend` / `stackshop-frontend`). Dashboards, alert rules and scrape config live in `stackshop-infra`, so an alert threshold can change without an app release. When a service adds a business metric, open a matching infra PR for its panel/alert.

| Dashboard | Panels |
|-----------|--------|
| **Platform overview** | Request rate, error %, p50/p95/p99 latency per service (RED); pod restarts; CPU/memory vs requests |
| **Service detail** (templated `$service`) | RED per endpoint, dependency call latency, GC/threadpool, outbox pending, consumer lag |
| **Business KPIs** | Orders/min, revenue/hour, AOV, checkout conversion (orders_paid / orders_created), payment failure %, cancellation reasons |
| **Messaging** | Kafka consumer lag per group, messages in/out per topic, RabbitMQ queue depth, unacked messages, DLQ size, SQS `ApproximateNumberOfMessagesVisible` (via CloudWatch exporter or LocalStack poller) |
| **Datastores** | MySQL QPS, slow queries, connections; Mongo opcounters; Redis hit ratio, memory, evictions; ES search latency, indexing rate, JVM heap |
| **Kubernetes** | Built-in kube-prometheus-stack dashboards |
| **Frontend** | Web Vitals percentiles (if exported), Next.js server response times |

## 5. Alerts (PrometheusRule)

| Alert | Expression (simplified) | For | Severity |
|-------|-------------------------|-----|----------|
| HighErrorRate | `sum(rate(http_server_request_duration_seconds_count{http_response_status_code=~"5.."}[5m])) by (service) / sum(rate(...[5m])) by (service) > 0.05` | 5m | critical |
| HighLatencyP95 | `histogram_quantile(0.95, sum by (le, service)(rate(http_server_request_duration_seconds_bucket[5m]))) > 1` | 10m | warning |
| PodCrashLooping | `increase(kube_pod_container_status_restarts_total[15m]) > 3` | 0m | critical |
| KafkaConsumerLagHigh | `sum by (consumergroup)(kafka_consumergroup_lag) > 1000` | 10m | warning |
| DLQNotEmpty | `rabbitmq_queue_messages{queue=~".*\\.dlq"} > 0` or kafka dlq offsets increase | 5m | warning |
| OutboxBacklog | `stackshop_outbox_pending > 500` | 5m | warning |
| PaymentFailureSpike | `rate(stackshop_payments_total{outcome="failed"}[10m]) / rate(stackshop_payments_total[10m]) > 0.2` | 10m | critical |
| NoOrdersInBusinessHours | `sum(increase(stackshop_orders_created_total[30m])) == 0` (time-of-day filtered) | 30m | warning |
| RedisMemoryHigh | `redis_memory_used_bytes / redis_memory_max_bytes > 0.9` | 5m | warning |
| ESClusterRed | `elasticsearch_cluster_health_status{color="red"} == 1` | 1m | critical |

Alertmanager routes to email (Mailpit locally) or a Slack/Discord webhook.

## 6. SLOs (learning targets)

| SLI | SLO |
|-----|-----|
| Storefront API availability (non-5xx) | 99.5% / 30d |
| Product page API p95 latency | < 300ms |
| Search p95 latency | < 200ms |
| Checkout success (orders paid / payment attempts, excluding user cancels) | > 98% |
| Event propagation (product update → searchable) | p95 < 5s |

Stretch goal: generate burn-rate alerts with **Sloth** or **Pyrra**.

## 7. Logging standards

- Serilog with the `CompactJsonFormatter` to stdout. Enrich with `service`, `env`, `traceId`, `spanId`, `userId` (hashed) and `correlationId`.
- Levels: `Information` for business events (`Order {OrderNumber} paid`), `Warning` for handled anomalies, `Error` for exceptions.
- No PII or secrets (destructuring policies mask `Email`, `Phone`, `Password`, `Token`).
- Request logging via `UseSerilogRequestLogging` (one line per request).
