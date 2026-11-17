# 06 — Frontend Architecture (Next.js + shadcn/ui)

## 1. Apps

| App | Path | Audience | Rendering focus |
|-----|------|----------|-----------------|
| **web** (storefront) | `apps/web` | Guests and customers | SEO: SSG/ISR for catalog pages, RSC + streaming, minimal client JS |
| **admin** (back-office) | `apps/admin` | Admin and support | Client-heavy data tables and forms, no SEO |

Both apps live in the **`stackshop-frontend` repository** (pnpm + Turborepo workspace) and share `packages/ui` (shadcn components + Tailwind preset + theme tokens). This repo contains no backend code.

Talking to the backend:

- The only backend dependency is the npm package **`@stackshop/api-client`** (typed client + DTO types). The backend repo generates it from OpenAPI and publishes it to GitHub Packages. Pin the version in `package.json` and let Renovate open bump PRs.
- The only runtime dependency is the gateway URL (`API_BASE_URL`, server-side only, never exposed as `NEXT_PUBLIC_*`). Browsers never call the gateway directly, except for the SignalR hub (`NEXT_PUBLIC_REALTIME_URL`).
- For unreleased APIs during local development: `pnpm gen:api --from http://localhost:5000/swagger/v1/swagger.json` generates a local client that overrides the package.
- UI work without a backend: MSW (Mock Service Worker) handlers in `apps/*/mocks` built from the client's types. CI Playwright smoke tests use them too.

## 2. Tech choices

| Concern | Choice |
|---------|--------|
| Framework | Next.js (latest stable, App Router, TypeScript strict) |
| UI kit | shadcn/ui (Radix primitives + Tailwind CSS v4), `lucide-react` icons |
| Forms | `react-hook-form` + `zod` (+ `@hookform/resolvers`). The same zod schemas are reused in Server Actions |
| Server data | React Server Components calling `api-client` server-side. `fetch` cache tags for revalidation |
| Client data | TanStack Query (admin tables, infinite lists, optimistic cart updates) |
| Client state | Zustand (cart drawer open, UI preferences). Keep it minimal |
| Tables | TanStack Table + shadcn `DataTable` |
| Charts | shadcn Charts (Recharts) |
| Auth | Custom BFF: httpOnly cookies, `middleware.ts` refresh (see [05-api-design.md §4](./05-api-design.md)) |
| Real-time | `@microsoft/signalr` client for notifications and order status |
| i18n | `next-intl` (en, vi) |
| Images | `next/image` with a remote pattern for the MinIO/S3 public bucket |
| Testing | Vitest + Testing Library (unit), Playwright (e2e), Storybook (optional, for `packages/ui`) |
| Lint/format | ESLint (next config), Prettier, `tsc --noEmit` in CI |
| Package manager | pnpm workspaces + Turborepo |

## 3. Storefront route map (`apps/web/app`)

```
app/
├── (shop)/
│   ├── page.tsx                        # Home: hero, categories, trending, recommendations (ISR 60s)
│   ├── c/[...categoryPath]/page.tsx    # Category listing (ES search with category filter)
│   ├── p/[slug]/page.tsx               # Product detail (ISR + on-demand revalidate by tag `product:{id}`)
│   ├── search/page.tsx                 # Search results + facets (dynamic, searchParams)
│   ├── brands/[slug]/page.tsx
│   └── cart/page.tsx
├── (checkout)/
│   ├── checkout/page.tsx               # Address → shipping → coupon → review (multi-step)
│   ├── checkout/pay/[orderId]/page.tsx # Redirect to / embed the mock PSP
│   └── checkout/result/[orderId]/page.tsx  # Polls/subscribes to order status
├── (account)/account/
│   ├── page.tsx                        # Overview
│   ├── orders/page.tsx, orders/[id]/page.tsx
│   ├── addresses/page.tsx
│   ├── wishlist/page.tsx
│   ├── reviews/page.tsx
│   ├── notifications/page.tsx
│   └── security/page.tsx               # Change password, sessions
├── (auth)/
│   ├── login/page.tsx, register/page.tsx
│   ├── forgot-password/page.tsx, reset-password/page.tsx
│   └── verify-email/page.tsx
├── api/
│   ├── auth/refresh/route.ts           # Rotates cookies
│   ├── auth/logout/route.ts
│   └── track/route.ts                  # Clickstream beacon → gateway /analytics/track
├── revalidate/route.ts                 # Webhook from catalog to revalidate tags (secret-protected)
├── layout.tsx, not-found.tsx, error.tsx, loading.tsx
└── sitemap.ts, robots.ts
middleware.ts                           # auth refresh, protect /account & /checkout, guest cart cookie
```

## 4. Admin route map (`apps/admin/app`)

```
app/(dashboard)/
├── page.tsx                  # KPIs: revenue, orders, AOV, conversion; charts
├── products/  (list, new, [id]/edit — variants, images, attributes per category schema)
├── categories/  brands/
├── inventory/ (stock by SKU/warehouse, adjust, movement ledger, low-stock list)
├── orders/    (list with filters, [id] detail with timeline, cancel/refund actions)
├── customers/ (list, detail, lock/unlock)
├── promotions/ (promotions + coupons CRUD, usage stats)
├── reviews/   (moderation queue)
├── reports/   (sales, top products, export → SQS job → download link)
├── ops/       (DLQ viewer & replay, search reindex button, feature flags)
└── settings/  (shipping rates, email templates)
```

## 5. Data-fetching patterns

| Situation | Pattern |
|-----------|---------|
| Public catalog page | RSC + `fetch(url, { next: { tags: ['product:'+id], revalidate: 300 } })`. Catalog calls `/revalidate` when a product changes |
| Personalised data (cart count, account) | RSC with `cookies()` → dynamic rendering. Wrap in `<Suspense>` to stream |
| Mutations (add to cart, place order) | **Server Actions** → api-client → `revalidateTag` / `revalidatePath`. Optimistic UI with `useOptimistic` |
| Admin grids | Client components + TanStack Query → Next.js route handler proxy (`/api/proxy/*`) that attaches the token |
| Live order status | SignalR hub via gateway WebSocket. Falls back to polling every 5s |

## 6. UI / design system

- Use shadcn components: Button, Input, Form, Select, Dialog, Sheet (cart drawer), Drawer (mobile filters), Command (⌘K search), Tabs, Table, Badge, Card, Carousel, Skeleton, Toast (Sonner), Pagination, Breadcrumb, Accordion (filters), Slider (price range), RadioGroup (variants), Chart.
- Theme tokens are CSS variables (light and dark). Brand colour comes from a single `--primary`.
- Accessibility: keyboard navigation everywhere (Radix gives most of it). `alt` text required in the product image form. Colour contrast AA.
- Responsive: mobile-first. Product grid has 2/3/4/5 columns at sm/md/lg/xl.
- Components that live in `packages/ui`: `ProductCard`, `PriceTag`, `RatingStars`, `QuantityStepper`, `VariantPicker`, `FacetFilter`, `OrderStatusBadge`, `AddressForm`, `EmptyState`, `DataTable`.

## 7. Performance budget (storefront)

| Metric | Target (mobile, 4G) |
|--------|---------------------|
| LCP | < 2.5s |
| INP | < 200ms |
| CLS | < 0.1 |
| JS on PDP (first load) | < 150 KB gz |

Measure with Lighthouse CI in GitHub Actions on preview builds.

## 8. Frontend observability

- Web Vitals are reported via `useReportWebVitals` → `/api/track` → Kafka → analytics (and optionally to a Prometheus histogram through the gateway).
- Next.js server: OpenTelemetry (`@vercel/otel` or `@opentelemetry/sdk-node`) sends traces with `traceparent` propagated to the gateway.
- `/api/health` route for K8s probes. `output: 'standalone'` for small Docker images.
