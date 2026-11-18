---
name: new-page
description: Add a Next.js App Router page or route segment to the StackShop storefront (apps/web) or admin (apps/admin). Covers server-side data fetching through @stackshop/api-client, ISR/cache tags or dynamic rendering, loading/empty/error states, metadata/SEO, next-intl messages, middleware protection and tests. Use when a feature spec's Frontend section or the route maps in docs/06-frontend.md need a page.
---

# New page / route

Check the route map in `docs/06-frontend.md` §3 (storefront) or §4 (admin) and the spec's Frontend section. Use the path exactly as documented.

## 1. Pick the rendering mode

| Page type | Mode |
|---|---|
| Public catalog (home, category, PDP, brand) | Static/ISR: `fetch(..., { next: { tags: ['product:'+id], revalidate: 300 } })`. Revalidated by the `/revalidate` webhook |
| Search / listing with filters | Dynamic (reads `searchParams`). **Filters, sort and page live in the URL** |
| Personalised (account, cart, checkout, admin) | Dynamic (`cookies()`). Personal fragments go inside `<Suspense>` so the shell streams |

## 2. Files to create in the route folder

```
app/(group)/<segment>/
├── page.tsx       # async Server Component: fetch via api-client (server-only module), render
├── loading.tsx    # Skeleton matching the final layout (avoid CLS)
├── error.tsx      # 'use client' error boundary: friendly message + retry; map ProblemDetails code
└── (not-found handled with notFound() when the API returns 404)
```

- Data access goes through a server-only helper (`lib/api/server.ts` with `import 'server-only'`), which attaches the `at` cookie as the bearer token. **Never** call the gateway from client components (except via route handlers / Server Actions).
- Empty state: use `EmptyState` from `packages/ui` with a helpful CTA.
- `generateMetadata`: title/description. For PDP/category, add canonical URLs, Open Graph and JSON-LD (`Product`, `BreadcrumbList`). Add the page to `sitemap.ts` if it's public.
- Text comes from `next-intl` messages (`messages/en.json`, `messages/vi.json`), with no hard-coded strings. Money and dates are formatted with `Intl` in the user's locale.
- Use shadcn components / `packages/ui` building blocks. If you need a new reusable piece, add it to `packages/ui` (see the `ui-ux-pro-max:ui-styling` skill for shadcn patterns).

## 3. Protection

- Storefront `/account/**` and `/checkout/**` are already guarded by `middleware.ts`. New protected prefixes must be added to its matcher.
- Admin pages: check the permission server-side (from the decoded token) and hide the nav item in the sidebar config. The API still enforces it, but the UI must not show forbidden pages.

## 4. Performance and accessibility

- Keep it a Server Component by default. Push `'use client'` down to the smallest interactive leaf.
- `next/image` with explicit sizes. No layout shift from late-loading content.
- One `<h1>`, landmarks, keyboard-reachable controls, visible focus, alt text.
- Storefront budget: LCP < 2.5s, PDP JS < 150 KB gz (check the `next build` output).

## 5. Tests

- Vitest + Testing Library for interactive client components on the page.
- MSW handler for the API calls used (typed from `@stackshop/api-client`).
- Playwright test in `tests/e2e` for the spec's journey on this page, named after the spec ID (`F04_AC3 brand facet keeps other counts`).
- Run `pnpm lint`, `pnpm typecheck`, `pnpm test` and `pnpm build --filter <app>`.

## 6. Handoff

- **backend-engineer:** any missing endpoint/field (don't invent one; mock it with MSW and flag it).
- **devops-engineer:** a new public route needing an ingress/CSP change, or new env vars.
