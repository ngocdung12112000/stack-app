# F15 — Admin Back-office, Dashboard & Reports

| Field | Value |
|-------|-------|
| Phase | 1 (shell) → 6 (dashboard + reports) |
| Services | `analytics` (read models + reports + export), all services' admin APIs |
| Stack | Next.js admin app (shadcn DataTable, Charts), Kafka consumers → MongoDB read models, **SQS** export jobs → S3, Redis |
| Depends on | Most features. Built incrementally |

## 1. User stories

| ID | Story |
|----|-------|
| US-15.1 | As an admin, I see a dashboard: today's revenue, orders, AOV, conversion rate, new customers, with a % change vs the previous period |
| US-15.2 | As an admin, I see charts: revenue over time (day/week/month), orders by status, top 10 products, sales by category |
| US-15.3 | As an admin, I can export orders/sales/inventory reports for a date range to CSV/XLSX and download them when ready |
| US-15.4 | As an admin, I can manage customers (list, detail with order history, lock/unlock) |
| US-15.5 | As an admin (ops), I can view DLQ messages and replay them, and trigger a search reindex |
| US-15.6 | Support users see only the sections their permissions allow |

## 2. Read models (analytics service, MongoDB)

Built from Kafka events (consumer group `analytics`). **Never** query other services' databases.

| Collection | Built from | Shape |
|-----------|-----------|-------|
| `orders_rm` | ordering.order-events, payment events | flat order doc: id, number, customerId, status, totals, items[{productId, categoryIds, qty, revenue}], placedAt, paidAt |
| `daily_sales` | order-paid, order-refunded | `{_id: date, orders, revenue, refunds, aov, newCustomers, byCategory{}, byHour[24]}` (incremental `$inc` upserts) |
| `product_sales` | order-paid | `{_id: productId, name, unitsSold, revenue, last30d[]}` |
| `funnel_daily` | clickstream (F16) + orders | `{_id: date, sessions, productViews, addToCarts, checkouts, orders}` → conversion |

> Idempotency matters here because `$inc` is not idempotent. Use the Inbox per message before applying increments.

## 3. Report export flow (SQS)

```
Admin clicks Export (type=orders, from, to, format=xlsx)
 → POST /api/v1/reports/exports → create ExportJob(Queued) in Mongo → SQS report-export-jobs
 → ExportWorker receives (visibility 300s; extends visibility every 60s for long jobs)
     → stream query orders_rm with cursor → write CSV (CsvHelper) / XLSX (ClosedXML / MiniExcel)
     → multipart upload to S3 private bucket → ExportJob(Ready, s3Key, rowCount)
     → notification in-app "Your export is ready"
 → GET /api/v1/reports/exports/{id}/download → 302 to pre-signed GET URL (10 min)
Failure ×3 → DLQ → ExportJob(Failed)
```

## 4. API (analytics)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/reports/kpis?from=&to=&compare=previous` | KPI cards |
| GET | `/api/v1/reports/revenue?from=&to=&granularity=day|week|month` | Time series |
| GET | `/api/v1/reports/top-products?from=&to=&limit=10` | |
| GET | `/api/v1/reports/sales-by-category?from=&to=` | |
| GET | `/api/v1/reports/funnel?from=&to=` | |
| POST | `/api/v1/reports/exports` | Create export job |
| GET | `/api/v1/reports/exports` | My jobs |
| GET | `/api/v1/reports/exports/{id}/download` | Pre-signed redirect |
| GET | `/api/v1/admin/ops/dlq?broker=kafka|rabbitmq|sqs` | List DLQ messages |
| POST | `/api/v1/admin/ops/dlq/replay` | `{broker, source, messageIds[]}` |

## 5. Admin app layout

- Shell: a collapsible `Sidebar` (shadcn sidebar block) with sections filtered by permissions, a top bar with a global search (`Command`: jump to an order by number, a product by name, a customer by email), notifications and the user menu.
- Dashboard: 5 KPI `Card`s with deltas, an area chart for revenue, a bar chart for orders by status, a donut for category share, a top-products table, and a date-range picker with presets (Today, 7d, 30d, This month).
- Shared `DataTable`: server-side pagination/sort/filter via URL params, column visibility, row selection, CSV export of the current page.
- Customers page: list + detail (profile from identity, orders from `orders_rm`, reviews), with lock/unlock.
- Ops page: DLQ tabs per broker with a JSON viewer and a replay button. A search reindex with a progress bar. Links to Grafana, Kafka UI, RabbitMQ UI and Kibana.

## 6. Acceptance criteria

- AC-1: The dashboard numbers for a seeded day match a direct SQL computation from `ordering_db` (verification script).
- AC-2: Exporting 50k orders completes without worker OOM (streaming), and the download link works for 10 minutes.
- AC-3: A Support user cannot see Promotions or Ops in the sidebar, and the API returns 403 if called directly.
- AC-4: Replaying a DLQ message re-processes it successfully once the underlying issue is fixed.
- AC-5: Replaying Kafka from offset 0 into an empty analytics DB rebuilds identical read models (thanks to the inbox and deterministic projections).

## 7. Learning focus

CQRS read models and projections, rebuilding by replay, SQS long-running jobs (visibility extension), streaming large exports, dashboard UX with shadcn charts, permission-aware UI.
