---
name: frontend-architecture
description: Frontend architecture patterns — invoked when choosing rendering strategies, structuring large applications, designing API layers, state architecture, build tooling, or testing strategy.
---

# Frontend Architecture Patterns

## Rendering Strategies

```
Decision tree:

Content changes how often?
  ├─ Never / rarely (docs, marketing)
  │     → SSG (Astro, Next.js static export)
  │
  ├─ Periodically (news, product catalog)
  │     → ISR (Next.js revalidate, Astro on-demand)
  │
  ├─ Per-request (user dashboard, personalized)
  │     Is fast initial HTML critical?
  │     ├─ YES → SSR + Streaming (Next.js App Router)
  │     └─ NO  → CSR (Vite SPA)
  │
  └─ Real-time (collaboration, trading)
        → CSR + WebSockets/SSE
```

### Comparison

```
Strategy  │ TTFB     │ FCP      │ SEO  │ Personalized │ Infra cost
──────────┼──────────┼──────────┼──────┼──────────────┼───────────
CSR       │ Fast     │ Slow*    │ Poor │ Yes          │ Low (CDN)
SSR       │ Variable │ Fast     │ Good │ Yes          │ Medium
SSG       │ Fastest  │ Fastest  │ Good │ No           │ Lowest
ISR       │ Fast     │ Fast     │ Good │ Partial      │ Low
Streaming │ Fast     │ Fast     │ Good │ Yes          │ Medium

* CSR FCP is slow because JS must download, parse, execute before render
```

### CSR (Client-Side Rendering)

```tsx
// Vite + React — pure SPA
// src/main.tsx
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import { QueryClientProvider, QueryClient } from '@tanstack/react-query';
import { App } from './App';

const queryClient = new QueryClient();

createRoot(document.getElementById('root')!).render(
  <QueryClientProvider client={queryClient}>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </QueryClientProvider>
);
```

### SSG (Static Site Generation)

```tsx
// Next.js App Router — statically generated at build time
// app/blog/[slug]/page.tsx

export async function generateStaticParams() {
  const posts = await fetchAllPosts();
  return posts.map(post => ({ slug: post.slug }));
}

export default async function BlogPostPage({
  params,
}: {
  params: { slug: string };
}) {
  const post = await fetchPost(params.slug); // runs at build time
  return <Article post={post} />;
}
```

### ISR (Incremental Static Regeneration)

```tsx
// app/products/[id]/page.tsx

// Revalidate every 30 minutes
export const revalidate = 1800;

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await fetchProduct(params.id);
  return <ProductDetail product={product} />;
}

// On-demand revalidation (e.g., triggered by CMS webhook)
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';

export async function POST(req: Request) {
  const { secret, type, id } = await req.json();
  if (secret !== process.env.REVALIDATE_SECRET) {
    return Response.json({ error: 'Invalid secret' }, { status: 401 });
  }
  revalidateTag(`product-${id}`);
  return Response.json({ revalidated: true, timestamp: Date.now() });
}
```

### Streaming SSR

```tsx
// app/dashboard/page.tsx — Next.js App Router with React Suspense
import { Suspense } from 'react';

export default function DashboardPage() {
  return (
    <main>
      {/* Renders immediately — shell HTML streamed first */}
      <DashboardShell />

      {/* These stream in as their async Server Components resolve */}
      <Suspense fallback={<KPICardsSkeleton />}>
        <KPICards />          {/* fetches metrics */}
      </Suspense>

      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart />      {/* fetches chart data */}
      </Suspense>

      <Suspense fallback={<TableSkeleton />}>
        <RecentOrdersTable /> {/* fetches orders */}
      </Suspense>
    </main>
  );
}

// Server Component — data fetched directly; zero client JS
async function KPICards() {
  const metrics = await db.metrics.findMany({ where: { period: 'current' } });
  return (
    <div className="kpi-grid">
      {metrics.map(m => <KPICard key={m.id} metric={m} />)}
    </div>
  );
}
```

---

## Application Structure

### Feature-Based Folder Structure

```
src/
├── features/                  ← domain slices
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   └── AuthGuard.tsx
│   │   ├── hooks/
│   │   │   └── useAuth.ts
│   │   ├── api/
│   │   │   └── auth.api.ts
│   │   ├── store/
│   │   │   └── auth.store.ts
│   │   └── types.ts
│   │
│   ├── orders/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/
│   │   └── types.ts
│   │
│   └── products/
│       ├── components/
│       ├── hooks/
│       ├── api/
│       └── types.ts
│
├── shared/                    ← cross-feature utilities
│   ├── components/            ← generic UI (Button, Input, Modal)
│   ├── hooks/                 ← useDebounce, useLocalStorage
│   ├── utils/                 ← formatCurrency, parseDate
│   └── types.ts               ← shared TypeScript types
│
├── app/                       ← app shell, routing, providers
│   ├── providers.tsx
│   ├── router.tsx
│   └── layout.tsx
│
└── pages/ (or app/ for Next.js)
    ├── dashboard/
    ├── orders/
    └── settings/
```

**Why feature-based over type-based:**

```
Type-based (harder to work with):         Feature-based (co-located):
src/components/OrderTable.tsx             src/features/orders/components/OrderTable.tsx
src/hooks/useOrderFilters.ts              src/features/orders/hooks/useOrderFilters.ts
src/api/orders.ts                         src/features/orders/api/orders.api.ts
src/types/order.ts                        src/features/orders/types.ts

To work on Orders, you touch 4 dirs.      Everything in one place.
To delete Orders, you hunt everywhere.    Delete one folder.
```

### Module Federation (Micro-Frontends at Runtime)

```js
// webpack.config.js (host app)
new ModuleFederationPlugin({
  name: 'shell',
  remotes: {
    checkout:  'checkout@https://checkout.example.com/remoteEntry.js',
    analytics: 'analytics@https://analytics.example.com/remoteEntry.js',
  },
  shared: { react: { singleton: true }, 'react-dom': { singleton: true } },
})

// webpack.config.js (checkout micro-frontend)
new ModuleFederationPlugin({
  name: 'checkout',
  filename: 'remoteEntry.js',
  exposes: {
    './CheckoutFlow': './src/CheckoutFlow',
    './CartSummary':  './src/CartSummary',
  },
  shared: { react: { singleton: true } },
})

// In host app:
const CheckoutFlow = lazy(() => import('checkout/CheckoutFlow'));
```

---

## State Architecture

```
Three categories of state — each needs a different tool:

┌─────────────────────────────────────────────────────────────┐
│ SERVER STATE                                                │
│ Data from API; lives on server; cached on client           │
│ Tool: TanStack Query / SWR                                 │
│ Examples: user profile, product list, order history        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ CLIENT UI STATE                                             │
│ Local to UI; doesn't need persistence or sharing           │
│ Tool: useState (local) / Zustand (shared)                  │
│ Examples: modal open, selected tab, form dirty state       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ URL STATE                                                   │
│ Lives in URL; shareable, bookmarkable, browser-history safe│
│ Tool: nuqs / useSearchParams                               │
│ Examples: search query, filters, pagination, sort order    │
└─────────────────────────────────────────────────────────────┘
```

### URL State with nuqs

```tsx
import { useQueryState, parseAsInteger, parseAsString, parseAsArrayOf } from 'nuqs';

function ProductFilters() {
  const [page,     setPage]     = useQueryState('page',     parseAsInteger.withDefault(1));
  const [search,   setSearch]   = useQueryState('q',        parseAsString.withDefault(''));
  const [sort,     setSort]     = useQueryState('sort',     parseAsString.withDefault('relevance'));
  const [categories, setCategories] = useQueryState('cat',  parseAsArrayOf(parseAsString));

  // URL: /products?q=shoes&page=2&sort=price&cat=running,trail

  return (
    <div>
      <SearchInput value={search} onChange={setSearch} />
      <CategoryFilter selected={categories} onChange={setCategories} />
      <SortSelect value={sort} onChange={setSort} />
      <Pagination page={page} onPageChange={setPage} />
    </div>
  );
}
```

### State Colocation Principle

```
State should live as close to where it's used as possible.
Only lift state when two components genuinely need to share it.

Local → Sibling → Parent → Feature store → Global store → URL
        ↑                   ↑               ↑
  (lift when         (Zustand slice    (rare; truly
   siblings share)    when feature      app-wide state)
                      components share)
```

---

## API Layer

### Never call fetch directly in components

```tsx
// BAD: fetch in component — no error handling, no type safety, no reuse
function ProductList() {
  const [products, setProducts] = useState([]);
  useEffect(() => {
    fetch('/api/products').then(r => r.json()).then(setProducts);
  }, []);
}

// GOOD: typed API client layer
// src/features/products/api/products.api.ts
import { api } from '@/shared/api/client';
import type { Product, ProductFilters } from '../types';

export const productsApi = {
  list: (filters: ProductFilters) =>
    api.get<Product[]>('/products', { params: filters }),

  getById: (id: string) =>
    api.get<Product>(`/products/${id}`),

  create: (data: Omit<Product, 'id'>) =>
    api.post<Product>('/products', data),

  update: (id: string, data: Partial<Product>) =>
    api.patch<Product>(`/products/${id}`, data),
};

// src/shared/api/client.ts
import ky from 'ky'; // lightweight fetch wrapper

export const api = ky.create({
  prefixUrl: process.env.NEXT_PUBLIC_API_URL,
  headers: { 'Content-Type': 'application/json' },
  hooks: {
    beforeRequest: [
      req => {
        const token = getAuthToken();
        if (token) req.headers.set('Authorization', `Bearer ${token}`);
      },
    ],
    afterResponse: [
      async (_req, _opts, res) => {
        if (res.status === 401) redirectToLogin();
      },
    ],
  },
}).extend({ parseJson: r => r.json() });
```

### OpenAPI Code Generation

```bash
# Generate typed client from OpenAPI spec
npx openapi-typescript https://api.example.com/openapi.json -o src/shared/api/schema.ts
```

```tsx
// src/shared/api/client.ts — with generated types
import createClient from 'openapi-fetch';
import type { paths } from './schema';

export const client = createClient<paths>({
  baseUrl: process.env.NEXT_PUBLIC_API_URL,
});

// Usage — fully typed, autocomplete on paths and params
const { data, error } = await client.GET('/products/{id}', {
  params: { path: { id: '123' } },
});
// data is typed as the response schema; error has error schema type
```

### tRPC (End-to-End Type Safety)

```ts
// server/routers/products.ts
import { z } from 'zod';
import { router, publicProcedure, protectedProcedure } from '../trpc';

export const productsRouter = router({
  list: publicProcedure
    .input(z.object({ category: z.string().optional(), page: z.number().default(1) }))
    .query(async ({ input, ctx }) => {
      return ctx.db.products.findMany({
        where: input.category ? { category: input.category } : undefined,
        skip: (input.page - 1) * 20,
        take: 20,
      });
    }),

  create: protectedProcedure
    .input(z.object({ name: z.string().min(1), price: z.number().positive() }))
    .mutation(async ({ input, ctx }) => {
      return ctx.db.products.create({ data: input });
    }),
});

// Client — exact same types, no codegen
import { trpc } from '@/utils/trpc';

function ProductList() {
  const { data } = trpc.products.list.useQuery({ category: 'shoes', page: 1 });
  // data is fully typed from server definition
}
```

---

## Build and Tooling

### Vite Configuration

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: true }), // bundle analysis on build
  ],

  resolve: {
    alias: {
      '@': '/src',              // import from '@/components/Button'
      '@features': '/src/features',
      '@shared': '/src/shared',
    },
  },

  build: {
    rollupOptions: {
      output: {
        // Manual chunking strategy
        manualChunks: {
          'vendor-react': ['react', 'react-dom'],
          'vendor-query': ['@tanstack/react-query'],
          'vendor-router': ['react-router-dom'],
        },
      },
    },
    // Generate source maps for production error tracking
    sourcemap: true,
  },

  server: {
    proxy: {
      '/api': 'http://localhost:8080', // dev proxy
    },
  },
});
```

### TypeScript Strict Config

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",

    // Strict mode — all enabled
    "strict": true,
    "noUncheckedIndexedAccess": true,    // arr[i] is T | undefined
    "exactOptionalPropertyTypes": true,  // no extra undefined in optional
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,

    // Path aliases (match vite.config.ts)
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@features/*": ["src/features/*"],
      "@shared/*": ["src/shared/*"]
    },

    "skipLibCheck": true
  }
}
```

---

## Testing Strategy

```
Testing pyramid:

          /\
         /  \
        / E2E \        Playwright — few, critical paths
       /────────\
      / Integration\   Testing Library + MSW — feature flows
     /──────────────\
    /   Unit Tests   \  Vitest — many, fast, component behavior
   /──────────────────\
```

### Unit / Component Tests (Vitest + Testing Library)

```tsx
// Test behavior, not implementation
import { render, screen, userEvent } from '@testing-library/react';
import { vi } from 'vitest';
import { LoginForm } from './LoginForm';

const setup = () => {
  const onSubmit = vi.fn();
  render(<LoginForm onSubmit={onSubmit} />);
  return { onSubmit };
};

test('submits credentials when form is filled', async () => {
  const { onSubmit } = setup();
  const user = userEvent.setup();

  await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
  await user.type(screen.getByLabelText(/password/i), 'secret123');
  await user.click(screen.getByRole('button', { name: /sign in/i }));

  expect(onSubmit).toHaveBeenCalledWith({
    email: 'alice@example.com',
    password: 'secret123',
  });
});

test('shows error when email is invalid', async () => {
  setup();
  const user = userEvent.setup();

  await user.type(screen.getByLabelText(/email/i), 'not-an-email');
  await user.click(screen.getByRole('button', { name: /sign in/i }));

  expect(screen.getByRole('alert')).toHaveTextContent(/valid email/i);
});
```

### Integration Tests with MSW

```tsx
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/products', () =>
    HttpResponse.json([
      { id: '1', name: 'Widget', price: 29.99 },
      { id: '2', name: 'Gadget', price: 49.99 },
    ])
  ),

  http.post('/api/products', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: '3', ...body }, { status: 201 });
  }),
];

// test file
import { setupServer } from 'msw/node';

const server = setupServer(...handlers);
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('product list loads and displays items', async () => {
  render(
    <QueryClientProvider client={new QueryClient()}>
      <ProductList />
    </QueryClientProvider>
  );

  expect(await screen.findByText('Widget')).toBeInTheDocument();
  expect(screen.getByText('$29.99')).toBeInTheDocument();
});
```

### E2E Tests with Playwright

```ts
// tests/checkout.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Checkout flow', () => {
  test.beforeEach(async ({ page }) => {
    // Seed test data via API
    await page.request.post('/test/seed', { data: { scenario: 'checkout' } });
    await page.goto('/products');
  });

  test('user can complete purchase', async ({ page }) => {
    // Add item to cart
    await page.getByRole('button', { name: /add to cart/i }).first().click();
    await expect(page.getByRole('status', { name: /cart/i })).toContainText('1');

    // Proceed to checkout
    await page.getByRole('link', { name: /checkout/i }).click();
    await expect(page).toHaveURL('/checkout');

    // Fill shipping
    await page.getByLabel(/first name/i).fill('Alice');
    await page.getByLabel(/email/i).fill('alice@test.com');

    // Submit
    await page.getByRole('button', { name: /place order/i }).click();
    await expect(page.getByRole('heading', { name: /order confirmed/i })).toBeVisible();
  });
});

// playwright.config.ts
export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'Mobile Safari', use: { ...devices['iPhone 14'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
  },
});
```

---

## Environment Variables

```bash
# Vite
VITE_API_URL=https://api.example.com       # exposed to client (VITE_ prefix required)
API_SECRET=secret                          # server-only (no prefix); NOT exposed to client

# Next.js
NEXT_PUBLIC_API_URL=https://api.example.com  # exposed to client (NEXT_PUBLIC_ prefix)
DATABASE_URL=postgres://...                  # server-only

# Never commit secrets; use .env.local (git-ignored)
```

---

## Anti-Patterns

- **Type-based folder structure**: `src/components/`, `src/hooks/` — causes high coupling between features; use feature-based structure
- **Fetching directly in components with useEffect**: race conditions, no caching; use TanStack Query
- **Storing server state in Redux/Zustand**: doubles the state; TanStack Query IS the store for server data
- **No API abstraction layer**: `fetch('/api/...')` in components — not typesafe, not testable, repeated error handling
- **CSR for SEO-important pages**: crawlers may not execute JS; use SSG or SSR
- **SSR for highly dynamic, auth-gated apps**: overcomplicated for no SEO benefit; CSR is fine
- **Not code-splitting**: single 2MB JS bundle; split at routes and heavy features
- **`any` in TypeScript**: defeats the purpose; use `unknown` + type guards or generated types
- **E2E tests for unit-level concerns**: slow; use unit tests for logic; E2E for critical paths only

---

## Quick Reference Checklist

**Architecture:**
- [ ] Rendering strategy chosen to match content type (SSG/ISR/SSR/CSR)
- [ ] Feature-based folder structure
- [ ] Typed API client abstraction (no raw `fetch` in components)
- [ ] State correctly categorized: server → TanStack Query, URL → nuqs, UI → useState/Zustand

**Build:**
- [ ] TypeScript strict mode enabled
- [ ] Path aliases configured in both tsconfig and vite/webpack config
- [ ] Bundle analyzed; no oversized chunks
- [ ] Code split at route level; heavy libraries dynamic-imported

**Testing:**
- [ ] Unit/component tests with Vitest + Testing Library; test behavior not implementation
- [ ] API calls mocked with MSW in tests
- [ ] E2E tests with Playwright covering critical user journeys
- [ ] Tests run in CI on every PR

**Code quality:**
- [ ] ESLint + Prettier configured
- [ ] Pre-commit hooks (`husky` + `lint-staged`)
- [ ] No `any` in TypeScript (CI fails on `any`)
- [ ] Environment variables typed via `zod` schema validation on startup
