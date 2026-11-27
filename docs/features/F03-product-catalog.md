# F03 — Product Catalog

| Field | Value |
|-------|-------|
| Phase | 1 (CRUD + cache), 2 (events) |
| Service | `catalog` |
| Stack | MongoDB (with transactions on a replica set), Redis cache-aside, Kafka (outbox), gRPC |
| Frontend | Storefront home/category/PDP. Admin products, categories, brands |
| Depends on | F01 (admin auth). F14 for images (Phase 4; use image URLs before then) |

## 1. User stories

| ID | Story |
|----|-------|
| US-03.1 | As a shopper, I can browse the category tree from the navigation menu |
| US-03.2 | As a shopper, I can view a product page with images, description, specs, variants, price and stock badge |
| US-03.3 | As a shopper, I can choose variant options (Color, Size) and see the matching SKU's price and availability |
| US-03.4 | As an admin, I can manage categories (tree, up to 3 levels) with an attribute schema per category |
| US-03.5 | As an admin, I can manage brands |
| US-03.6 | As an admin, I can create a product as a draft, add variants, then publish/unpublish/archive it |
| US-03.7 | As an admin, I can bulk-import products from CSV/JSON (stretch goal) |

## 2. Domain model

```
Category { id, name, slug, parentId?, path[], level (1..3), sort, isActive,
           attributeSchema: [ { key, label, type: text|number|bool|enum, unit?, options?[], filterable, required } ] }
Brand    { id, name, slug, logoUrl, isActive }
Product (aggregate root)
  ├─ id, slug, name, brand(ref+name), categoryIds[], primaryCategoryId, description(HTML), attributes{}
  ├─ options[ {name, values[]} ]            // max 3 options, max 20 values each
  ├─ variants[ Variant {sku, options{}, price, compareAtPrice?, weightG, isActive} ]  // max 100
  ├─ images[], tags[], seo{}, status (Draft|Published|Archived), rating{avg,count}, version
  └─ methods: Create, UpdateDetails, SetOptions, UpsertVariant, RemoveVariant, ChangePrice, Publish, Unpublish, Archive
```

## 3. Business rules

- BR-01: `slug` is unique. It is auto-generated from the name (lowercase, ASCII-folded, hyphenated) and gets a `-2` suffix on collision. Editable by the admin.
- BR-02: `sku` is unique across all products (unique index on `variants.sku`).
- BR-03: Each variant's `options` must cover every product option with a valid value. No two variants may have the same option combination.
- BR-04: Product `attributes` are validated against the **primary category's** `attributeSchema` (required fields present, types correct, enum values allowed).
- BR-05: Publishing requires ≥ 1 active variant with price > 0, ≥ 1 image, a description, and a primary category.
- BR-06: `compareAtPrice`, if set, must be > `price` (it shows as a strikethrough "sale" price).
- BR-07: Archived products are hidden from the storefront and search, but stay resolvable for past orders by ID.
- BR-08: Optimistic concurrency. Updates require `If-Match: "{version}"`. A mismatch returns 412.
- BR-09: Category deletion is not allowed while products reference it. A category can only be deactivated.
- BR-10: Price changes emit `product-price-changed` per affected SKU (cart and ordering need this).

## 4. API

**Public (anonymous)**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/categories/tree` | Full active tree (cached `cache:category-tree`) |
| GET | `/api/v1/categories/{slug}` | Category + attribute schema (the frontend uses the filterable attributes for facets) |
| GET | `/api/v1/brands` | Active brands |
| GET | `/api/v1/products/{slug}` | Published product detail (cache-aside `cache:product:{slug}`) |
| GET | `/api/v1/products?ids=a,b,c` | Batch fetch (cards for recommendations and wishlist) |
| GET | `/api/v1/products?category=&brand=&sort=&page=` | Simple Mongo listing (Phase 1 only; replaced by search in Phase 2) |

**Admin (`catalog:write`)**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/admin/products?status=&q=&category=&page=` | Paged list |
| GET | `/api/v1/admin/products/{id}` | Full product incl. drafts. Response has an `ETag` |
| POST | `/api/v1/admin/products` | Create draft |
| PUT | `/api/v1/admin/products/{id}` | Update details/attributes (If-Match) |
| PUT | `/api/v1/admin/products/{id}/options` | Set options |
| PUT | `/api/v1/admin/products/{id}/variants/{sku}` | Upsert variant |
| DELETE | `/api/v1/admin/products/{id}/variants/{sku}` | Remove variant (only if it has no stock; checked via the inventory gRPC) |
| POST | `/api/v1/admin/products/{id}/publish` / `unpublish` / `archive` | Status transitions |
| PUT | `/api/v1/admin/products/{id}/images` | Ordered image list (asset IDs from F14) |
| CRUD | `/api/v1/admin/categories`, `/api/v1/admin/brands` | |
| GET | `/api/v1/internal/products/export?cursor=` | Paged export for the search reindex (cluster-internal) |

**gRPC (internal)**

```proto
service CatalogQuery {
  rpc GetSkus (GetSkusRequest) returns (GetSkusResponse);  // [{sku, productId, name, variantLabel, price, imageUrl, isActive, weightG}]
}
```

Used by cart (add item), ordering (price re-validation at checkout) and promotion (category of a SKU).

Errors: `CATALOG_SLUG_TAKEN`, `CATALOG_SKU_TAKEN`, `CATALOG_INVALID_ATTRIBUTES`, `CATALOG_PUBLISH_REQUIREMENTS_NOT_MET`, `CATALOG_VERSION_MISMATCH` (412), `CATALOG_VARIANT_HAS_STOCK`.

Phase 1 implementation adds (see section 11): `CATALOG_VALIDATION_FAILED` (400), `CATALOG_IF_MATCH_REQUIRED` (428), `CATALOG_INVALID_OPTIONS`, `CATALOG_OPTIONS_IN_USE`, `CATALOG_INVALID_VARIANT`, `CATALOG_VARIANT_COMBINATION_TAKEN`, `CATALOG_TOO_MANY_VARIANTS`, `CATALOG_VARIANT_NOT_FOUND`, `CATALOG_INVALID_TRANSITION`, `CATALOG_PRODUCT_NOT_FOUND`, `CATALOG_CATEGORY_NOT_FOUND`, `CATALOG_CATEGORY_TOO_DEEP`, `CATALOG_CATEGORY_HAS_PRODUCTS`, `CATALOG_CATEGORY_HAS_CHILDREN`, `CATALOG_BRAND_NOT_FOUND`, `CATALOG_BRAND_IN_USE`.

## 5. Events

| Event | Trigger |
|-------|---------|
| `catalog.product-created.v1` | Draft created (consumers mostly ignore drafts) |
| `catalog.product-updated.v1` | Any change. Payload = full snapshot incl. status |
| `catalog.product-price-changed.v1` | Variant price changed (one event per SKU) |
| `catalog.product-deleted.v1` | Archived |
| Compacted `catalog.product-snapshot` | Latest snapshot per product (lets the search index rebuild from Kafka) |

Consumes: `review.rating-updated.v1` → sets `rating`. RabbitMQ `ImageProcessed` → sets image URLs.

Cache invalidation: on any write, delete `cache:product:{slug}` (and the old slug if it changed), then call the Next.js `/revalidate` webhook with tag `product:{id}`.

## 6. Frontend

**Storefront**

- Mega-menu from the category tree (shadcn `NavigationMenu`).
- PDP `/p/[slug]`: image gallery (Carousel + thumbnails), title, brand, rating stars, `PriceTag` (sale), `VariantPicker` (unavailable combinations disabled, using stock from the inventory API), `QuantityStepper`, Add to cart, Wishlist heart, Tabs (Description / Specifications table from attributes + schema labels / Reviews), "Frequently bought together" (F16), breadcrumbs, JSON-LD `Product` schema for SEO.
- ISR with tag-based revalidation.

**Admin**

- Product list DataTable (status filter, search, bulk publish).
- Product editor: tabs for General, Attributes (a form generated **dynamically** from the category schema), Options & Variants (a matrix generator: choose option values → generate SKUs → edit price per row), Images (drag-sort), SEO. Shows a conflict dialog on 412.
- Category tree editor (drag to reorder, attribute schema builder).

## 7. Non-functional

- PDP API p95 < 50ms on a cache hit, < 150ms on a miss.
- Cache hit ratio > 90% under the load test.
- Mongo transactions wrap product + outbox writes.

## 8. Acceptance criteria

- AC-1: Publishing a product with no images returns `CATALOG_PUBLISH_REQUIREMENTS_NOT_MET` with details.
- AC-2: Creating a variant with a duplicate SKU (even in another product) returns 409.
- AC-3: Two admins edit the same product; the second save returns 412 and the UI shows a reload/merge prompt.
- AC-4: Changing a price invalidates the Redis cache and the PDP shows the new price after revalidation (< 5s).
- AC-5: Attributes not matching the category schema are rejected with a field-level error.
- AC-6: Archived products return 404 on the public PDP but appear in admin and in past orders.

## 9. Tasks

1. Mongo setup (replica set in compose), class maps / serialisers, indexes created on startup (idempotent).
2. Domain aggregate + invariant unit tests (variants and options rules).
3. Category schema validator.
4. Repositories with optimistic concurrency (`ReplaceOne` filter on `version`).
5. Redis cache-aside decorator for the product query + lock on miss.
6. Mongo outbox + relay (Phase 2).
7. gRPC `CatalogQuery`.
8. Admin UI: list, editor, variant matrix, category tree.
9. Storefront: menu, PDP, ISR + revalidate route.
10. Integration tests (Testcontainers Mongo + Redis).

## 10. Learning focus

Document modelling (embedding variants), schema-on-read validation, Mongo transactions and indexes, cache-aside with stampede protection, optimistic concurrency with ETags, ISR/tag revalidation, dynamic forms.

## 11. Phase 1 backend implementation notes

Status: backend Phase 1 is implemented in `services/catalog` (AC-1, 2, 3, 5, 6 and the backend half of AC-4). Where it differs from or refines the text above:

- **Auth.** Every `/api/v1/admin/*` endpoint (reads included) needs `perm:catalog:write`. `catalog:read` is not seeded in identity and the spec has no read-only role, so there is no separate read permission. Catalog validates the identity JWT itself (JWKS) and the gateway requires authentication on the admin routes.
- **If-Match is mandatory** on every mutating admin product call (details, options, variants, images, publish, unpublish, archive) and on category and brand updates. Missing header: 428 `CATALOG_IF_MATCH_REQUIRED`. Stale version: 412 `CATALOG_VERSION_MISMATCH`. Every successful admin response (including variant delete) returns the whole product and the new `ETag`. The ETag is `"<version>"`.
- **Status codes.** `CATALOG_PUBLISH_REQUIREMENTS_NOT_MET` is 422 with `errors` keyed `variants`, `images`, `description`, `primaryCategoryId`. Duplicate SKU or slug is 409. `CATALOG_INVALID_ATTRIBUTES` is 400 with `errors["attributes.<key>"]`. A published product is re-checked against BR-05 after every change, so removing its last image or active variant is rejected with the same 422.
- **Transitions.** Draft to Published (publish), Published to Draft (unpublish), Draft or Published to Archived (archive, final). Anything else is 409 `CATALOG_INVALID_TRANSITION`.
- **Slugs.** Categories and brands also get generated slugs. A category slug and its parent are immutable after creation (descendant paths and product breadcrumbs embed them). Product slug and brand slug are editable; the old product slug's cache key is invalidated.
- **SKUs** are normalised to upper case (`^[A-Z0-9][A-Z0-9._-]{1,63}$`), also in the `{sku}` URL segment.
- **Money** is `decimal` with at most 2 decimals, stored as Decimal128. `compareAtPrice` must be greater than `price`. The currency is the service-wide `Catalog:Currency` option (default `USD`); there is no per-product currency yet.
- **Listing** `GET /products` keeps the Phase 0 shape `{items, page, pageSize, totalCount}`. Item fields `id, name, price, currency` are unchanged; `slug, brandName, imageUrl, compareAtPrice, ratingAverage, ratingCount` were added. `price` is the lowest active variant price. Filters: `category` (slug, includes sub-categories), `brand` (slug), `sort` (`newest` default, `price_asc`, `price_desc`, `name`), `page`, `pageSize` (max 100). `ids=a,b,c` switches to batch mode: published products only, input order, max 50.
- **Attributes** need a primary category (the first category is used when none is given); keys not in the schema are rejected. Changing a category's attribute schema does not revalidate existing products (schema-on-read).
- **Category and brand delete.** A category with sub-categories or products, or a brand with products, cannot be deleted (409); deactivate instead. Renaming a brand refreshes the denormalised reference on its products and drops their cache entries.
- **Cache.** `cache:product:{slug}` (10 min plus up to 60 s jitter) and `cache:category-tree` (60 min). Misses take `lock:<key>` (SET NX PX 10s); other callers poll the cache for up to 3 s, then load themselves. Not-found results are not cached. Redis failures never fail a request (the catalog reads Mongo directly). Every write deletes the affected keys after the commit (old and new slug). Metric: `catalog_cache_requests_total{cache,result}`.
- **Mongo.** Ids are GUID v7 stored as strings. Documents carry `schemaVersion` (1). Indexes are created idempotently at startup (see `docs/03-data-architecture.md` section 3.1). The unique SKU index is partial (`variants.sku` of type string) because an empty `variants` array would otherwise collide across products.
- **Not yet done (later phases or other features):** outbox, Kafka events and `product-price-changed` (BR-10, Phase 2); gRPC `CatalogQuery`; `/api/v1/internal/products/export`; the Next.js `/revalidate` webhook (so AC-4 "PDP updates within 5 s" is backend-complete, the frontend half is open); CSV import (US-03.7); consuming `review.rating-updated.v1` and `ImageProcessed`; `CATALOG_VARIANT_HAS_STOCK` (needs the inventory service, F05; the variant delete handler has a TODO); images are plain URLs until F14.
