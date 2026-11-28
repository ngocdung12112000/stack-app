# 03 — Data Architecture

## 1. Store ownership

| Store | Owner services | Databases / namespaces |
|-------|----------------|------------------------|
| **MySQL 8** | identity, inventory, ordering, payment, shipping, promotion | One **schema per service**: `identity_db`, `inventory_db`, `ordering_db`, `payment_db`, `shipping_db`, `promotion_db`. Each has its own DB user with grants on its own schema only |
| **MongoDB 7** | catalog, review, notification, analytics, wishlist (in catalog or its own) | DBs: `catalog`, `review`, `notification`, `analytics` |
| **Redis 7** | cart, gateway, catalog (cache), promotion, identity, SignalR backplane | Logical prefixes: `cart:`, `rl:`, `cache:`, `lock:`, `idem:`, `promo:`, `session:` |
| **Elasticsearch 8** | search | Index `products-v{n}` with alias `products`. `suggest-v{n}` with alias `suggest` |
| **S3 / MinIO** | media | Buckets `stackshop-media-original`, `stackshop-media-public` |

## 2. MySQL schemas

Conventions: `snake_case` names. PK is `id BINARY(16)` (GUID v7) or `CHAR(26)` (ULID). Choose one; examples use `CHAR(36)` for readability. Every table has `created_at`, `updated_at` as `DATETIME(6)` UTC. Concurrency uses `row_version` (`INT` incremented by EF as a concurrency token).

### 2.1 identity_db

```sql
users (
  id CHAR(36) PK,
  email VARCHAR(256) UNIQUE NOT NULL,
  normalized_email VARCHAR(256) UNIQUE NOT NULL,
  password_hash VARCHAR(512) NOT NULL,
  full_name VARCHAR(200),
  phone VARCHAR(30),
  avatar_asset_id CHAR(36) NULL,  -- media asset reference (F02)
  email_confirmed BOOLEAN DEFAULT 0,
  status ENUM('Active','Locked','Deleted') DEFAULT 'Active',
  failed_login_count INT DEFAULT 0,
  lockout_end DATETIME(6) NULL,
  row_version INT NOT NULL,      -- optimistic concurrency token
  created_at, updated_at
)
roles (id, name UNIQUE)                          -- Customer, Admin, Support
user_roles (user_id FK, role_id FK, PK(user_id, role_id))
role_permissions (role_id FK, permission VARCHAR(100))
refresh_tokens (
  id PK, user_id FK, token_hash CHAR(64) UNIQUE, family_id CHAR(36),
  expires_at, revoked_at NULL, replaced_by_id NULL, created_ip VARCHAR(45), user_agent VARCHAR(300)
)
email_tokens (id, user_id, purpose ENUM('Confirm','Reset'), token_hash CHAR(64), created_at, expires_at, used_at)
addresses (
  id PK, user_id FK INDEX, recipient_name, phone, line1, line2, ward, district, city,
  country_code CHAR(2), postal_code VARCHAR(20), is_default BOOLEAN, created_at, updated_at
  -- one default per user is enforced in the domain (AddressBook) under a row lock on users, not by an index (F02)
)
outbox_messages (see §2.7)
```

### 2.2 inventory_db

```sql
warehouses (id, code UNIQUE, name, address, is_active)
stock_items (
  id PK, sku VARCHAR(64), warehouse_id FK,
  on_hand INT NOT NULL CHECK (on_hand >= 0),
  reserved INT NOT NULL DEFAULT 0 CHECK (reserved >= 0),
  -- available = on_hand - reserved
  low_stock_threshold INT DEFAULT 5,
  row_version INT NOT NULL,
  UNIQUE(sku, warehouse_id)
)
reservations (
  id PK, order_id CHAR(36) INDEX, status ENUM('Active','Committed','Released','Expired'),
  expires_at DATETIME(6) INDEX, created_at
)
reservation_lines (reservation_id FK, sku, warehouse_id, quantity)
stock_movements (           -- append-only ledger, used for audit and reports
  id BIGINT AUTO_INCREMENT PK, sku, warehouse_id, delta INT,
  reason ENUM('Restock','Reserve','Release','Commit','Adjust','Return'),
  reference_id VARCHAR(64), created_by, created_at
)
outbox_messages, inbox_messages
```

### 2.3 ordering_db

```sql
orders (
  id PK, order_number VARCHAR(20) UNIQUE,      -- human-friendly e.g. SS-20261117-000123
  customer_id CHAR(36) INDEX,
  status ENUM('Pending','AwaitingPayment','Paid','Processing','Shipped','Delivered',
              'Cancelled','RefundRequested','Refunded') INDEX,
  currency CHAR(3),
  subtotal DECIMAL(18,2), discount_total DECIMAL(18,2), shipping_fee DECIMAL(18,2),
  tax_total DECIMAL(18,2), grand_total DECIMAL(18,2),
  coupon_code VARCHAR(50) NULL,
  shipping_address JSON,                         -- snapshot, not a FK
  cancel_reason VARCHAR(200) NULL,
  placed_at, paid_at NULL, cancelled_at NULL, delivered_at NULL,
  row_version INT, created_at, updated_at
)
order_items (
  id PK, order_id FK INDEX, product_id, sku, product_name, variant_label,
  image_url, unit_price DECIMAL(18,2), quantity INT, line_discount DECIMAL(18,2), line_total DECIMAL(18,2)
)
order_status_history (id, order_id FK, from_status, to_status, reason, actor, created_at)
saga_state (                                    -- checkout saga instance
  order_id PK, current_step VARCHAR(50), payment_deadline DATETIME(6),
  reservation_id NULL, payment_id NULL, row_version INT, updated_at
)
idempotency_keys (key VARCHAR(100) PK, customer_id, request_hash, response_json, created_at)
outbox_messages, inbox_messages
```

### 2.4 payment_db

```sql
payments (
  id PK, order_id UNIQUE, customer_id, amount DECIMAL(18,2), currency,
  status ENUM('Pending','RequiresAction','Succeeded','Failed','Cancelled','Refunded','PartiallyRefunded'),
  provider VARCHAR(30), provider_payment_id VARCHAR(100) UNIQUE NULL,
  failure_code VARCHAR(50) NULL, created_at, updated_at, row_version
)
refunds (id PK, payment_id FK, amount, reason, status ENUM('Pending','Succeeded','Failed'), provider_refund_id, created_at)
webhook_events (provider_event_id VARCHAR(100) PK, type, payload JSON, received_at, processed_at NULL)
outbox_messages, inbox_messages
```

### 2.5 shipping_db

```sql
shipments (
  id PK, order_id UNIQUE, carrier VARCHAR(30), tracking_number VARCHAR(50) UNIQUE NULL,
  status ENUM('Created','LabelPrinted','PickedUp','InTransit','OutForDelivery','Delivered','Failed','Returned'),
  shipping_address JSON, estimated_delivery DATE NULL, created_at, updated_at
)
shipment_events (id, shipment_id FK, status, location, description, occurred_at)
shipping_rates (id, zone VARCHAR(30), min_weight_g, max_weight_g, fee DECIMAL(18,2))
outbox_messages, inbox_messages
```

### 2.6 promotion_db

```sql
promotions (
  id PK, name, type ENUM('PercentOff','AmountOff','FreeShipping','BuyXGetY'),
  value DECIMAL(18,2), max_discount DECIMAL(18,2) NULL, min_order_amount DECIMAL(18,2) NULL,
  scope ENUM('Order','Category','Product'), scope_ids JSON NULL,
  starts_at, ends_at, is_active, stackable BOOLEAN DEFAULT 0, created_at, updated_at
)
coupons (
  id PK, promotion_id FK, code VARCHAR(50) UNIQUE, usage_limit INT NULL,
  per_customer_limit INT DEFAULT 1, used_count INT DEFAULT 0, row_version
)
coupon_redemptions (id PK, coupon_id FK, customer_id, order_id UNIQUE, status ENUM('Reserved','Confirmed','Released'), created_at)
outbox_messages, inbox_messages
```

### 2.7 Shared messaging tables (in every MySQL service DB)

```sql
outbox_messages (
  id CHAR(36) PK,               -- = message/event id
  aggregate_type VARCHAR(100), aggregate_id VARCHAR(64),
  topic VARCHAR(150), event_type VARCHAR(150),
  payload JSON, headers JSON,
  occurred_at DATETIME(6), processed_at DATETIME(6) NULL INDEX, attempts INT DEFAULT 0, last_error TEXT NULL
)
inbox_messages (
  message_id CHAR(36), consumer VARCHAR(100), processed_at DATETIME(6),
  PRIMARY KEY(message_id, consumer)
)
```

Migrations use **EF Core migrations** per service. They are applied by a K8s `Job` (or an init container) before the deployment rolls out, never on app startup in production.

## 3. MongoDB collections

### 3.1 catalog.products

```jsonc
{
  "_id": "01JD...",                     // ULID string
  "slug": "apple-macbook-air-m3-13",
  "name": "MacBook Air M3 13\"",
  "brand": { "id": "...", "name": "Apple", "slug": "apple" },          // denormalised
  "categoryIds": ["laptops", "apple-laptops"],
  "categoryPath": ["electronics", "computers", "laptops"],            // breadcrumb
  "description": "<p>...</p>",
  "attributes": { "cpu": "M3", "screen": "13.6\"", "weightKg": 1.24 }, // flexible per category
  "options": [ { "name": "Color", "values": ["Silver","Midnight"] }, { "name": "RAM", "values": ["8GB","16GB"] } ],
  "variants": [
    { "sku": "MBA-M3-SLV-8", "options": {"Color":"Silver","RAM":"8GB"},
      "price": 1099.00, "compareAtPrice": 1199.00, "weightG": 1240, "barcode": "...", "isActive": true }
  ],
  "images": [ { "url": "...", "thumbUrl": "...", "alt": "...", "sort": 0 } ],
  "rating": { "average": 4.6, "count": 128 },          // updated from review events
  "status": "Published",                                // Draft | Published | Archived
  "seo": { "title": "...", "description": "..." },
  "tags": ["new", "bestseller"],
  "version": 7,                                         // optimistic concurrency
  "createdAt": ISODate, "updatedAt": ISODate, "publishedAt": ISODate
}
```

Indexes (created idempotently at catalog startup): `ux_products_slug` `{slug:1}` unique; `ux_products_variants_sku` `{"variants.sku":1}` unique and **partial** (`variants.sku` of type string; an empty `variants` array would otherwise collide across products); `{categoryIds:1, status:1}`; `{"brand.id":1}`; `{"brand.slug":1, status:1}`; `{status:1, publishedAt:-1}`; `{status:1, minPrice:1}`; `{status:1, updatedAt:-1}`. `_id` is a GUID v7 stored as a string, money is Decimal128, `minPrice` is the lowest active variant price (stored for sorting), and every document has `schemaVersion` (currently 1).

Other collections: `categories` (`{_id, name, slug, parentId, path (slugs root..self), level, sort, isActive, attributeSchema, version, schemaVersion}`; indexes `ux_categories_slug` unique, `{parentId, sort}`, `{path}`), `brands` (`{_id, name, slug, logoUrl, isActive, version}`; unique slug), `outbox` (same shape as the MySQL outbox. Written in a MongoDB **transaction**, which needs a replica set, even a single-node one locally), `inbox`, `wishlists` (`{_id: userId, items:[{productId, sku, addedAt}]}`).

### 3.2 review.reviews

```jsonc
{ "_id", "productId", "sku", "userId", "userDisplayName", "orderId",
  "rating": 5, "title": "...", "body": "...", "images": [...],
  "verifiedPurchase": true, "status": "Pending|Approved|Rejected",
  "helpfulCount": 12, "helpfulVoters": [userId...] /* capped */, "createdAt", "updatedAt" }
```

Indexes: `{productId:1, status:1, createdAt:-1}`, `{userId:1, productId:1}` unique (one review per product per user).
`review.product_rating_summary`: `{_id: productId, avg, count, histogram: {1:..,5:..}}`, maintained with an aggregation pipeline on change.

### 3.3 notification

- `notifications`: `{_id, userId, channel: "email|inapp", type, title, body, data, readAt, createdAt}`. TTL index on `createdAt` (180 days).
- `templates`: `{_id: "order-paid", channel, subject, bodyHtml (Handlebars/Scriban), locale}`.
- `delivery_log`: `{messageId, to, status, attempts, error, sentAt}`.

### 3.4 analytics

- `events`: raw clickstream (`{type: "product_viewed", userId|anonId, productId, ts, ...}`). Time-series collection, TTL 90 days.
- `daily_sales`: pre-aggregated `{_id: "2026-11-17", orders, revenue, aov, byCategory:{...}}`.
- `product_cooccurrence`: `{_id: productId, related: [{productId, score}]}` (the "frequently bought together" output).

## 4. Redis key design

| Key | Type | TTL | Owner | Purpose |
|-----|------|-----|-------|---------|
| `cart:{cartId}` | Hash (field = sku, value = JSON line) | 30d (user) / 7d (guest), sliding | cart | Cart lines |
| `cart:meta:{cartId}` | Hash | same | cart | ownerId, currency, couponCode, updatedAt |
| `cache:product:{slug}` | String (JSON) | 10m + jitter | catalog | Product detail cache-aside |
| `cache:category-tree` | String | 1h | catalog | Navigation menu |
| `rl:{clientId}:{route}` | String / sorted set | window | gateway | Sliding-window rate limit |
| `idem:{service}:{key}` | String | 24h | ordering, payment | Idempotency response cache |
| `lock:{resource}` | String (SET NX PX) | 5–30s | inventory, jobs | Distributed lock (Redlock-lite) |
| `promo:coupon:{code}:used` | String counter | until coupon expiry | promotion | Fast atomic usage check (Lua) |
| `auth:revoked-jti:{jti}` | String | = token remaining life | identity, gateway | Access-token blacklist on logout |
| `auth:login-attempts:{ip}` | String counter | 15m | identity | Brute-force protection (failed logins per IP) |
| `auth:login-unknown:{sha256(email)[:32]}` | String counter | 15m | identity | Lockout for emails with no account, so lockout never reveals existence |
| `reco:trending:{yyyyMMddHH}` | Sorted set | 48h | analytics | Trending products by views |
| `reco:recent:{userId}` | List (cap 20) | 30d | analytics | Recently viewed |
| `signalr:*` | pub/sub | – | notification | SignalR backplane |

Rules: always set a TTL. Add random jitter (±10%) to cache TTLs to avoid stampedes. Use `GET` → miss → **lock** → load → `SET` for hot keys.

## 5. Elasticsearch

### 5.1 `products` index mapping (abridged)

```jsonc
{
  "settings": {
    "analysis": {
      "analyzer": {
        "text_std": { "tokenizer": "standard", "filter": ["lowercase", "asciifolding"] },
        "autocomplete": { "tokenizer": "standard", "filter": ["lowercase","asciifolding","edge_ngram_2_20"] }
      },
      "filter": { "edge_ngram_2_20": { "type": "edge_ngram", "min_gram": 2, "max_gram": 20 } }
    }
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "id":            { "type": "keyword" },
      "slug":          { "type": "keyword" },
      "name":          { "type": "text", "analyzer": "text_std",
                         "fields": { "ac": { "type": "text", "analyzer": "autocomplete", "search_analyzer": "text_std" },
                                     "kw": { "type": "keyword" } } },
      "description":   { "type": "text", "analyzer": "text_std" },
      "brand":         { "type": "keyword" },
      "categoryIds":   { "type": "keyword" },
      "categoryPath":  { "type": "keyword" },
      "tags":          { "type": "keyword" },
      "attributes":    { "type": "flattened" },
      "minPrice":      { "type": "scaled_float", "scaling_factor": 100 },
      "maxPrice":      { "type": "scaled_float", "scaling_factor": 100 },
      "inStock":       { "type": "boolean" },
      "ratingAvg":     { "type": "float" },
      "ratingCount":   { "type": "integer" },
      "soldCount":     { "type": "integer" },
      "imageUrl":      { "type": "keyword", "index": false },
      "publishedAt":   { "type": "date" },
      "variants":      { "type": "nested", "properties": { "sku": {"type":"keyword"}, "price": {"type":"scaled_float","scaling_factor":100}, "options": {"type":"flattened"} } }
    }
  }
}
```

### 5.2 Reindex strategy

1. Create `products-v{n+1}` with the new mapping.
2. Bulk-load it from catalog (a `search` admin endpoint triggers a full rebuild by calling catalog's paginated export API).
3. Swap the alias `products` atomically (`_aliases` with remove + add).
4. Delete the old index after verification.

Kafka consumers always write to the **alias**, so no consumer change is needed.

## 6. Data consistency matrix

| Data | Source of truth | Copies (eventually consistent) | Max staleness target |
|------|-----------------|-------------------------------|----------------------|
| Product info & price | catalog (Mongo) | search (ES), cart lines, order item snapshot | < 5s |
| Stock availability | inventory (MySQL) | search `inStock`, product page badge | < 5s |
| Rating | review (Mongo) | catalog `rating`, search `ratingAvg` | < 30s |
| Order status | ordering (MySQL) | analytics, notification | < 5s |
| Cart price | catalog | re-validated **synchronously** at checkout | 0 (checked at checkout) |

> **Rule:** prices and stock are always **re-validated synchronously at checkout**. Cached and indexed copies are for display only.

## 7. Backup & data lifecycle (learning scope)

- MySQL: daily `mysqldump` CronJob to S3/MinIO. Practise restore once per phase.
- MongoDB: `mongodump` CronJob.
- ES: snapshot repository on S3/MinIO.
- Redis: AOF on for cart durability (carts are tolerable to lose, but losing them is a poor UX).
- Kafka retention: 7 days for domain topics, `compact` for `catalog.product-snapshot`.
