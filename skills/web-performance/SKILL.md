---
name: web-performance
description: Web performance optimization — invoked when diagnosing slow pages, improving Core Web Vitals, optimizing bundles, images, caching, or server rendering strategy.
---

# Web Performance Optimization

## Core Web Vitals

```
LCP (Largest Contentful Paint)     — perceived load speed
INP (Interaction to Next Paint)    — responsiveness
CLS (Cumulative Layout Shift)      — visual stability

         Good    Needs work    Poor
LCP      < 2.5s    2.5–4s      > 4s
INP      < 200ms  200–500ms   > 500ms
CLS      < 0.1     0.1–0.25   > 0.25
```

### LCP — What to Optimize

LCP element is usually: hero image, `<h1>`, or large text block.

```html
<!-- 1. Preload the LCP image — biggest single win -->
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high">

<!-- 2. Mark LCP image as high priority -->
<img src="/hero.webp" fetchpriority="high" loading="eager" alt="Hero">

<!-- 3. Preconnect to image CDN -->
<link rel="preconnect" href="https://cdn.example.com">
```

```
LCP optimization hierarchy:
1. Eliminate render-blocking resources (CSS, sync scripts)
2. Reduce server response time (TTFB) — target < 600ms
3. Preload LCP resource
4. Optimize LCP resource (compress image, use WebP/AVIF)
5. Prioritize LCP resource in CDN cache
```

### INP — Interaction Responsiveness

INP replaced FID in March 2024. It measures the worst interaction latency (p98) during page visit.

```
Interaction lifecycle:
User input → [Input delay] → Event handlers → [Processing time] → Render → [Presentation delay] → Frame

Target: total < 200ms
         ↑            ↑                          ↑
    < 50ms       < 100ms                     < 50ms
  (no long tasks)  (yield to browser)      (no large DOM)
```

**Breaking up long tasks:**

```js
// BAD: 300ms synchronous task blocks input
function processAllItems(items) {
  items.forEach(item => heavyProcess(item)); // blocks for 300ms
}

// GOOD: yield to browser between chunks
async function processAllItems(items) {
  for (let i = 0; i < items.length; i++) {
    heavyProcess(items[i]);

    // Yield every 50 items
    if (i % 50 === 0) {
      await scheduler.yield(); // or: await new Promise(r => setTimeout(r, 0));
    }
  }
}

// BETTER: scheduler.postTask() with priority
await scheduler.postTask(() => heavyWork(), { priority: 'background' });
```

### CLS — Layout Stability

```html
<!-- BAD: image without dimensions causes layout shift -->
<img src="/product.jpg" alt="Product">

<!-- GOOD: explicit dimensions reserve space -->
<img src="/product.jpg" width="400" height="300" alt="Product">

<!-- GOOD: aspect-ratio in CSS also works -->
<style>
  .product-image {
    aspect-ratio: 4 / 3;
    width: 100%;
  }
</style>
```

```css
/* Reserve space for lazy-loaded content */
.ad-slot {
  min-height: 250px; /* know the expected size */
}

/* Font swap causes shift; preload fonts to minimize */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter.woff2') format('woff2');
  font-display: swap; /* show fallback, swap when loaded */
  /* font-display: optional — don't swap if not loaded fast */
}
```

---

## Critical Rendering Path

```
HTML parse → DOM
CSS parse  → CSSOM
               ↓
        DOM + CSSOM = Render Tree
                           ↓
                         Layout
                           ↓
                         Paint
                           ↓
                        Composite
```

**What blocks rendering:**

```html
<!-- render-blocking: browser pauses HTML parse until CSS loads -->
<link rel="stylesheet" href="styles.css">

<!-- render-blocking: browser pauses until script executes -->
<script src="app.js"></script>

<!-- non-blocking: parsed after HTML; order not guaranteed -->
<script async src="analytics.js"></script>

<!-- non-blocking: parsed after HTML; in order; better for deps -->
<script defer src="app.js"></script>
```

**Optimized `<head>`:**

```html
<head>
  <!-- 1. Inline critical CSS (above-fold styles only, < 14KB) -->
  <style>/* critical CSS inlined here */</style>

  <!-- 2. Preconnect to required origins -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://cdn.example.com" crossorigin>

  <!-- 3. Preload critical resources -->
  <link rel="preload" as="image" href="/hero.webp" fetchpriority="high">
  <link rel="preload" as="font" href="/fonts/inter.woff2" crossorigin>

  <!-- 4. Non-critical CSS loaded async -->
  <link rel="preload" href="/styles.css" as="style"
        onload="this.onload=null;this.rel='stylesheet'">
  <noscript><link rel="stylesheet" href="/styles.css"></noscript>

  <!-- 5. Scripts deferred -->
  <script defer src="/app.js"></script>
</head>
```

---

## Resource Loading

### Images

```html
<!-- Responsive images with srcset -->
<img
  src="/hero-800.webp"
  srcset="/hero-400.webp 400w,
          /hero-800.webp 800w,
          /hero-1600.webp 1600w"
  sizes="(max-width: 768px) 100vw,
         (max-width: 1200px) 50vw,
         800px"
  width="800"
  height="450"
  alt="Hero image"
  loading="lazy"    <!-- lazy for below-fold -->
  decoding="async"
>

<!-- LCP image: eager + high priority -->
<img
  src="/hero.webp"
  fetchpriority="high"
  loading="eager"
  decoding="sync"
  width="1200"
  height="600"
  alt="Hero"
>
```

**Image format decision:**

```
PNG → WebP: ~30% smaller, same quality, broad support
JPG → WebP: ~25-35% smaller
Any → AVIF: ~50% smaller vs WebP, but encoding is slow; use for hero images

Use <picture> for format negotiation:
<picture>
  <source srcset="/hero.avif" type="image/avif">
  <source srcset="/hero.webp" type="image/webp">
  <img src="/hero.jpg" alt="Hero">
</picture>
```

### Fonts

```html
<!-- 1. Preload the most important font file -->
<link rel="preload" href="/fonts/inter-regular.woff2" as="font" crossorigin>

<!-- 2. Self-host fonts (avoid Google Fonts round-trip) -->
<style>
  @font-face {
    font-family: 'Inter';
    src: url('/fonts/inter-regular.woff2') format('woff2');
    font-weight: 400;
    font-style: normal;
    font-display: swap;
    /* Subset: only include characters you use */
    unicode-range: U+0000-00FF; /* Latin */
  }
</style>
```

---

## JavaScript Performance

### Bundle Analysis

```bash
# Vite
npx vite build --report
# Opens rollup-plugin-visualizer treemap in browser

# Webpack
npx webpack-bundle-analyzer stats.json

# Quick size check
npx bundlephobia lodash  # check a package before installing
```

### Tree Shaking

```tsx
// BAD: imports entire lodash (500KB+)
import _ from 'lodash';
const result = _.groupBy(items, 'category');

// GOOD: import only what you need (tree-shakeable)
import { groupBy } from 'lodash-es'; // ESM version required

// BETTER: write it yourself — groupBy is 4 lines
const result = Object.groupBy(items, item => item.category); // native ES2024
```

### Dynamic Imports

```tsx
// Split large features
async function handleExport() {
  // Only loads when user clicks Export
  const { exportToPDF } = await import('./utils/pdf-export');
  await exportToPDF(data);
}

// Split large libraries
async function renderChart(data: ChartData) {
  const { Chart } = await import('chart.js/auto');
  new Chart(canvas, { type: 'line', data });
}
```

### Web Workers

```tsx
// worker.ts
self.addEventListener('message', (e: MessageEvent) => {
  const { data, threshold } = e.data;
  const result = heavyDataProcessing(data, threshold); // runs off main thread
  self.postMessage(result);
});

// main thread
const worker = new Worker(new URL('./worker.ts', import.meta.url), { type: 'module' });

function processData(data: unknown[], threshold: number): Promise<unknown> {
  return new Promise((resolve, reject) => {
    worker.postMessage({ data, threshold });
    worker.onmessage = e => resolve(e.data);
    worker.onerror = e => reject(e);
  });
}
```

---

## Caching Strategy

```
Cache hierarchy:

Browser Memory Cache (fastest, session-only)
       ↓
Browser Disk Cache (fast, controlled by Cache-Control)
       ↓
Service Worker Cache (programmable, offline support)
       ↓
CDN Edge Cache (global, fast)
       ↓
Origin Server (slowest)
```

### HTTP Cache Headers

```
# Content-hashed assets (JS/CSS bundles, fonts, images with hash in name)
Cache-Control: max-age=31536000, immutable
# Browser caches for 1 year; immutable = skip revalidation

# HTML documents (never cache aggressively)
Cache-Control: no-cache
# Or: Cache-Control: max-age=0, must-revalidate
# Forces revalidation on every request (ETag comparison)

# API responses
Cache-Control: private, max-age=60
# 60 seconds, not shared in CDN
```

### Service Worker (Workbox)

```js
// sw.js — using Workbox (generated by vite-plugin-pwa or similar)
import { registerRoute } from 'workbox-routing';
import { StaleWhileRevalidate, CacheFirst, NetworkFirst } from 'workbox-strategies';

// Static assets: cache-first (they have content hashes)
registerRoute(
  ({ request }) => request.destination === 'script' || request.destination === 'style',
  new CacheFirst({ cacheName: 'static-assets' })
);

// Images: cache-first, expire after 30 days
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [new ExpirationPlugin({ maxAgeSeconds: 30 * 24 * 60 * 60 })]
  })
);

// API: network-first, fall back to cache when offline
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({ cacheName: 'api-cache', networkTimeoutSeconds: 3 })
);

// HTML: stale-while-revalidate
registerRoute(
  ({ request }) => request.destination === 'document',
  new StaleWhileRevalidate({ cacheName: 'html-cache' })
);
```

---

## Server-Side Performance

```
Rendering strategy decision:

Is content the same for every user?
  YES → SSG (Astro, Next.js static)
        Does it update?
          RARELY (< once/hour) → SSG with rebuild on publish
          PERIODICALLY         → ISR (Next.js, revalidate: 3600)

  NO  → Is fast TTFB critical?
         YES → SSR (Next.js, streaming)
         NO  → CSR (Vite SPA)
```

### Streaming SSR (React 18 + Next.js)

```tsx
// app/dashboard/page.tsx (Next.js App Router)
import { Suspense } from 'react';

export default function DashboardPage() {
  return (
    <main>
      {/* Shell renders immediately */}
      <DashboardHeader />

      {/* Streams in as data resolves */}
      <Suspense fallback={<MetricsSkeleton />}>
        <MetricsPanel />   {/* async Server Component */}
      </Suspense>

      <Suspense fallback={<TableSkeleton />}>
        <OrdersTable />    {/* async Server Component */}
      </Suspense>
    </main>
  );
}

// Async Server Component — data fetched on server, no client JS needed
async function MetricsPanel() {
  const metrics = await fetchMetrics(); // direct DB/API call
  return <MetricsDisplay data={metrics} />;
}
```

### ISR (Incremental Static Regeneration)

```tsx
// Next.js: revalidate every hour
export const revalidate = 3600;

export default async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await fetchPost(params.slug);
  return <Article post={post} />;
}

// On-demand revalidation via webhook
// app/api/revalidate/route.ts
import { revalidatePath } from 'next/cache';

export async function POST(req: Request) {
  const { slug, secret } = await req.json();
  if (secret !== process.env.REVALIDATE_SECRET) return new Response('Unauthorized', { status: 401 });

  revalidatePath(`/blog/${slug}`);
  return Response.json({ revalidated: true });
}
```

---

## Measuring Performance

```bash
# Lighthouse CLI — test production URL
npx lighthouse https://example.com --output html --output-path ./report.html

# WebPageTest — detailed waterfall, filmstrip, multiple locations
# https://webpagetest.org

# Chrome DevTools workflow:
# 1. Performance panel → Record → Interact → Stop
#    Look for: long tasks (red triangles), layout/paint (purple/green blocks)
# 2. Network panel → Sort by size; look for uncompressed assets
# 3. Coverage panel → Find unused JS/CSS bytes
```

**Performance budget example:**

```json
{
  "resourceSizes": [
    { "resourceType": "script",     "budget": 200 },
    { "resourceType": "stylesheet", "budget": 50 },
    { "resourceType": "image",      "budget": 500 },
    { "resourceType": "total",      "budget": 900 }
  ],
  "timings": [
    { "metric": "first-contentful-paint", "budget": 1500 },
    { "metric": "largest-contentful-paint", "budget": 2500 },
    { "metric": "cumulative-layout-shift", "budget": 0.1 }
  ]
}
```

---

## Anti-Patterns

- **`loading="lazy"` on the LCP image** — delays the most important resource; use `loading="eager"` + `fetchpriority="high"`
- **Images without explicit `width`/`height`** — causes CLS when images load
- **Importing entire libraries**: `import moment from 'moment'` (300KB) — use `date-fns` tree-shakeable functions
- **`setTimeout` for animations** — use `requestAnimationFrame`; browser paints once per rAF
- **Animating `top`/`left`/`width`** — triggers layout; use `transform` and `opacity`
- **No `Cache-Control` on static assets** — every page load re-downloads assets
- **Synchronous `localStorage` in render** — blocks main thread; read once in `useState` initializer
- **Uncompressed API responses** — enable gzip/brotli on server or CDN

---

## Quick Reference Checklist

**LCP:**
- [ ] Hero image preloaded with `<link rel="preload" fetchpriority="high">`
- [ ] LCP image in WebP or AVIF format
- [ ] TTFB < 600ms (CDN, edge caching, or SSR optimization)
- [ ] Render-blocking CSS/JS eliminated from critical path

**INP:**
- [ ] No long tasks > 50ms on interaction-critical paths
- [ ] Heavy computations yielded with `await scheduler.yield()`
- [ ] CPU work offloaded to Web Workers where possible

**CLS:**
- [ ] All images have explicit `width` + `height` attributes
- [ ] Font loading uses `font-display: swap` with preloaded WOFF2
- [ ] Dynamically injected content doesn't push page content down

**Assets:**
- [ ] Content-hashed assets cached with `max-age=31536000, immutable`
- [ ] Images served as WebP/AVIF with `srcset`
- [ ] JS bundle analyzed; no large unused packages
- [ ] Code split at route level with `React.lazy` + `Suspense`

**Rendering:**
- [ ] Rendering strategy matched to content type (SSG/ISR/SSR/CSR)
- [ ] Streaming SSR with `Suspense` boundaries for async data
- [ ] Service Worker caching for offline / repeat visit performance
