# F16 — Activity Tracking & Recommendations

| Field | Value |
|-------|-------|
| Phase | 6 |
| Service | `analytics` |
| Stack | Kafka (high-volume `analytics.clickstream`, 12 partitions), Redis sorted sets/lists, MongoDB (time-series + co-occurrence), a .NET `BackgroundService` doing stream processing |
| Frontend | `/api/track` beacon, home "Trending", PDP "Frequently bought together", "Recently viewed" |
| Depends on | F03, F07 |

## 1. User stories

| ID | Story |
|----|-------|
| US-16.1 | As a shopper, I see "Recently viewed" products across sessions (and across devices when logged in) |
| US-16.2 | As a shopper, I see "Trending now" on the home page (most viewed in the last 24h) |
| US-16.3 | As a shopper, I see "Frequently bought together" on the PDP |
| US-16.4 | As an admin, I see the conversion funnel (view → cart → checkout → order) |

## 2. Event tracking

Client events (sent via `navigator.sendBeacon('/api/track', batch)` from the Next.js app, batched every 5s or on `visibilitychange`):

| Event | Properties |
|-------|-----------|
| `page_view` | path, referrer |
| `product_viewed` | productId, categoryId, price |
| `search_performed` | q, resultsCount |
| `add_to_cart` / `remove_from_cart` | sku, qty |
| `checkout_started` | cartValue |
| `web_vital` | name, value, rating |

Identity: an `anonId` cookie (1 year), plus `userId` when logged in, and a `sessionId` (30-minute inactivity window).
The BFF `/api/track` validates the schema (zod), adds the user agent, a geo hint and a server timestamp, and forwards to the gateway → analytics `POST /analytics/track` → **produce to Kafka** without awaiting per message (high throughput: `acks=1` and batching are acceptable here; we accept small loss).

Respect a privacy setting: if the user declined analytics cookies, only send `checkout_started` and order events (or nothing).

## 3. Stream processing (analytics consumers)

| Consumer | Logic | Output |
|----------|-------|--------|
| RecentlyViewed | on `product_viewed`: `LREM` + `LPUSH` + `LTRIM 0 19` on `reco:recent:{userId|anonId}` (TTL 30d). On login, merge anon → user | Redis list |
| Trending | on `product_viewed`: `ZINCRBY reco:trending:{yyyyMMddHH} 1 productId` (TTL 48h). Query: `ZUNIONSTORE` over the last 24 hourly keys with weights (recent hours weigh more), cached for 5 minutes | Redis ZSET |
| Co-occurrence | on `ordering.order-paid`: for every pair (a, b) of products in the order, `$inc related.b` in the doc of a and vice versa. A nightly job keeps the top 10 and normalises scores | Mongo `product_cooccurrence` |
| Funnel | counts distinct sessions per step per day (HyperLogLog `PFADD funnel:{date}:{step} sessionId`), flushed hourly to Mongo `funnel_daily` | Redis HLL → Mongo |
| Raw archive | stores all events in a Mongo time-series collection (TTL 90d) for ad-hoc analysis | Mongo |

## 4. API

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/analytics/track` | Batch of ≤ 50 events (anonymous, rate-limited) |
| GET | `/api/v1/recommendations/recently-viewed` | Uses anonId/userId |
| GET | `/api/v1/recommendations/trending?limit=12&category=` | |
| GET | `/api/v1/recommendations/bought-together/{productId}?limit=4` | Fallback: same-category top sellers |

All return product IDs. The frontend hydrates them via catalog batch (or the API hydrates them server-side through catalog gRPC).

## 5. Frontend

- A `<Tracker />` client component in the root layout + a `track()` helper. Auto page views via `usePathname` changes.
- Home: a "Trending now" carousel (RSC, revalidate 300s).
- PDP: a "Frequently bought together" bundle widget (checkboxes + "Add all to cart") and a "Recently viewed" strip (client component, since it is personal).

## 6. Acceptance criteria

- AC-1: Viewing products A, B, C, then A again → recently viewed = A, C, B.
- AC-2: Products bought together in ≥ 3 seeded orders appear in each other's "Frequently bought together".
- AC-3: A k6 test sending 2,000 events/s to `/track` keeps p95 < 50ms, and consumer lag recovers within 1 minute after the burst.
- AC-4: Login merges anonymous recently-viewed into the user's list.

## 7. Learning focus

High-throughput Kafka producing (batching, acks trade-offs, partitioning by user), consumer scaling with partitions, Redis ZSET/LIST/HyperLogLog, simple collaborative filtering, privacy-aware tracking.
