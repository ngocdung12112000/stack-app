# F04 — Product Search

| Field | Value |
|-------|-------|
| Phase | 2 |
| Service | `search` |
| Stack | Elasticsearch 8, Kafka consumer, Redis (popular queries) |
| Frontend | `/search`, `/c/[...path]`, header search with autocomplete (⌘K) |
| Depends on | F03 events, F05 stock events, F11 rating events |

## 1. User stories

| ID | Story |
|----|-------|
| US-04.1 | As a shopper, I can type a query and see relevant products, typo-tolerant and accent-insensitive ("ao thun" matches "áo thun") |
| US-04.2 | As a shopper, I see autocomplete suggestions (products, categories, brands) after 2 characters |
| US-04.3 | As a shopper, I can filter by category, brand, price range, rating, in-stock and category-specific attributes (facets with counts) |
| US-04.4 | As a shopper, I can sort by relevance, price ↑/↓, newest, best-selling, top-rated |
| US-04.5 | As a shopper, I get "Did you mean…" suggestions when there are no results |
| US-04.6 | As an admin, I can trigger a full reindex with zero downtime |

## 2. Indexing pipeline

```
catalog.product-events ──┐
inventory.stock-events ──┼──▶ search-indexer (consumer group "search")
review.review-events  ───┘        │
                                  ├─ product-updated (Published) → build doc → index (upsert by id)
                                  ├─ product-updated (Draft/Archived) / deleted → delete doc
                                  ├─ stock-changed → partial update `inStock` (script: any variant available)
                                  └─ rating-updated → partial update ratingAvg/ratingCount
```

- Document = denormalised product (see the mapping in [03-data-architecture.md §5](../03-data-architecture.md)). `minPrice`/`maxPrice` are computed from active variants.
- Stock per SKU is kept in a small field `availableSkus` (keyword[]). `inStock = availableSkus.length > 0`. Updated via a painless script.
- Use **external versioning** (`version_type=external`, version = product.version) so out-of-order events cannot overwrite newer data.
- Bulk indexing: the consumer batches up to 500 messages or 1 second → `_bulk`.

## 3. Query design

```jsonc
POST products/_search
{
  "query": {
    "bool": {
      "must": [{
        "multi_match": {
          "query": "macbook air",
          "fields": ["name^3", "name.ac^2", "brand^2", "tags", "description"],
          "type": "best_fields", "fuzziness": "AUTO", "operator": "and"
        }
      }],
      "filter": [
        { "term": { "categoryPath": "laptops" } },
        { "range": { "minPrice": { "gte": 500, "lte": 2000 } } },
        { "term": { "inStock": true } }
      ]
    }
  },
  "post_filter": { "terms": { "brand": ["Apple"] } },   // keep brand facet counts unaffected by brand selection
  "aggs": {
    "brands":     { "terms": { "field": "brand", "size": 20 } },
    "price":      { "histogram": { "field": "minPrice", "interval": 100 } },
    "rating":     { "range": { "field": "ratingAvg", "ranges": [{"from":4},{"from":3}] } },
    "attr_ram":   { "terms": { "field": "attributes.ram" } }    // attribute facets come from the category schema (filterable=true)
  },
  "sort": ["_score", { "soldCount": "desc" }],
  "from": 0, "size": 24,
  "highlight": { "fields": { "name": {} } }
}
```

- Relevance boost: `function_score` with `field_value_factor` on `soldCount` (log1p) and `ratingAvg`, plus a small boost for in-stock items.
- Autocomplete: `suggest` index with a `completion` field, or `search_as_you_type` on name. Returns top 5 products + 3 categories + 3 brands.
- "Did you mean": `phrase` suggester on `name`.
- Deep pagination limit: page ≤ 50 (`from + size ≤ 1200`). Beyond that, show "refine your search".

## 4. API

| Method | Path | Params | Response |
|--------|------|--------|----------|
| GET | `/api/v1/search/products` | `q, category, brand[], minPrice, maxPrice, rating, inStock, attr.{key}[], sort, page, pageSize` | `{items:[ProductCard], total, facets:{brands:[{value,count}], price:[...], attributes:{ram:[...]}}, suggestion?}` |
| GET | `/api/v1/search/autocomplete` | `q` (≥ 2 chars) | `{products:[{slug,name,imageUrl,price}], categories:[...], brands:[...]}` |
| GET | `/api/v1/search/popular` | – | Top 10 queries (Redis ZSET `search:popular:{day}`) |
| POST | `/api/v1/admin/search/reindex` | `catalog:write` | 202 `{jobId}`. Runs the alias swap procedure |
| GET | `/api/v1/admin/search/reindex/{jobId}` | | Progress |

## 5. Frontend

- Header `Command` dialog (⌘K / `/` shortcut) with debounced (200ms) autocomplete and keyboard navigation.
- `/search?q=` and category pages share a `SearchLayout`: left facet sidebar (Accordion with Checkbox lists, price Slider, rating radio), and a mobile `Drawer` for filters. Results grid with `ProductCard`. Sort `Select`. Active filter chips with "Clear all". `Pagination`.
- **URL is the state**: every filter maps to search params (bookmarkable, back button works). Server-rendered (RSC) for SEO, with streaming skeletons.
- An empty state shows the "Did you mean" link and popular searches.

## 6. Non-functional

- Search p95 < 200ms. Autocomplete p95 < 80ms.
- Index freshness: event to searchable p95 < 5s (`refresh_interval: 1s`).
- Analyzer handles Vietnamese diacritics (`asciifolding`). Stretch goal: the `analysis-icu` plugin.

## 7. Acceptance criteria

- AC-1: Searching "ao thun" returns products named "Áo thun …".
- AC-2: Searching "mcbook" (typo) returns MacBook products.
- AC-3: Selecting brand=Apple keeps the other brand facet counts visible (post_filter behaviour).
- AC-4: A product unpublished in admin disappears from search within 5s.
- AC-5: When the last unit of every SKU sells, the product shows "Out of stock" and is excluded when `inStock=true`.
- AC-6: During a reindex, search stays available with no errors (alias swap).
- AC-7: Replaying a stale `product-updated` event does not override a newer document (external versioning).

## 8. Tasks

1. ES + Kibana in compose. Index template + alias bootstrap on startup.
2. Kafka consumer with batching + bulk + DLQ.
3. Query builder (filters, facets from category schema, sort, highlight). Unit-test the generated JSON.
4. Autocomplete + suggester.
5. Reindex job (export from catalog → new index → alias swap) with progress in Redis.
6. Frontend search layout, facets, ⌘K command, URL state sync.
7. Integration tests with Testcontainers Elasticsearch.

## 9. Learning focus

Analyzers and tokenisation, mappings, bool queries vs filters, aggregations, post_filter, function_score relevance tuning, zero-downtime reindex, eventual consistency via Kafka.
