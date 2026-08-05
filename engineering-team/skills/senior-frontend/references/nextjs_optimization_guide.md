# Next.js Optimization Guide

Covers the App Router performance model end to end: what to render where, how to stream it, how to ship fewer bytes, and how to keep pages fast as they age. Pairs with `references/react_patterns.md` (Suspense mechanics) and the profile budgets in `SKILL.md` (`next-app-router`: 2000ms LCP mobile-4G p75, 150 KB-gzip/route).

## 1. Server Components vs. Client Components — the real tradeoff

The default in App Router is a **React Server Component (RSC)**: it renders on the server, never ships its JS to the browser, and can `await` data directly. `'use client'` opts a subtree into the traditional client-rendered model with hooks and interactivity — and ships that component's JS (plus its imports) to the browser.

| | Server Component | Client Component |
|---|---|---|
| Runs | Server only | Server (initial HTML) + browser (hydration + re-renders) |
| Can use `useState`/`useEffect`/event handlers | No | Yes |
| Ships JS to browser | No (zero bundle cost) | Yes |
| Can access backend directly (DB, fs, secrets) | Yes | No |
| Re-renders on client interaction | No — needs a client component ancestor/boundary | Yes |

**The tradeoff that actually matters:** every `'use client'` boundary is a bundle-size decision, not just an architecture decision. A component tree that's 90% presentational and 10% interactive should have the `'use client'` boundary drawn around the 10%, not the whole tree — otherwise you're shipping React runtime code for markup that never changes.

```tsx
// GOOD: only the button ships JS. The card, price, description stay server-rendered.
// app/products/[id]/page.tsx (Server Component, no directive)
async function ProductPage({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id); // direct DB/fetch call, no API round trip
  return (
    <article>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <Price amount={product.price} />
      <AddToCartButton productId={product.id} /> {/* client component */}
    </article>
  );
}
```

```tsx
// components/AddToCartButton.tsx
'use client';
import { useState } from 'react';

export function AddToCartButton({ productId }: { productId: string }) {
  const [pending, setPending] = useState(false);
  return (
    <button
      disabled={pending}
      onClick={async () => {
        setPending(true);
        await addToCart(productId);
        setPending(false);
      }}
    >
      Add to cart
    </button>
  );
}
```

**Common mistake:** marking a layout or page `'use client'` because one child needs interactivity. This drags every descendant — including ones that could've stayed server-only — into the client bundle. Push `'use client'` down to the leaf.

**Passing Server Components as children into Client Components** is allowed and often the fix for "I need a client wrapper but server content inside it":

```tsx
'use client';
export function Modal({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  return open ? <div className="modal">{children}</div> : null;
}

// Server Component page can still pass server-rendered children into the client Modal:
<Modal>
  <ServerRenderedComments postId={id} /> {/* stays a Server Component, no JS shipped for it */}
</Modal>
```

## 2. Streaming SSR with `loading.tsx` and `Suspense`

App Router streams HTML by default. Two mechanisms:

**Route-level:** `loading.tsx` wraps the whole route segment in a Suspense boundary automatically.

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <DashboardSkeleton />;
}
```

**Component-level:** manual `<Suspense>` boundaries let fast content ship immediately while slow content streams in behind it, rather than blocking the entire page on the slowest data source.

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default async function Dashboard() {
  return (
    <div>
      <Header />                          {/* renders immediately */}
      <Suspense fallback={<StatsSkeleton />}>
        <SlowStats />                      {/* streams in when ready, e.g. 800ms later */}
      </Suspense>
      <Suspense fallback={<FeedSkeleton />}>
        <SlowFeed />                       {/* streams in independently */}
      </Suspense>
    </div>
  );
}

async function SlowStats() {
  const stats = await getStats(); // slow query — doesn't block Header or SlowFeed
  return <StatsPanel stats={stats} />;
}
```

Server renders the shell (`Header` + both skeletons) in the first flush, then streams each `Suspense` boundary's content as its data resolves, over the same HTTP connection via chunked transfer encoding. This is the mechanism that lets a page with one slow API call still hit a fast **Time to First Byte** and fast **LCP** for above-the-fold content, instead of the old getServerSideProps model where one slow fetch blocked the entire response.

**Rule of thumb:** put the slowest, least-above-the-fold data fetch behind the outermost/lowest-priority Suspense boundary. Never wrap LCP-critical content in Suspense with a skeleton that's visually different from the final content — that causes CLS when it swaps in.

## 3. Parallel vs. sequential data fetching

```tsx
// BAD: sequential waterfall — 300ms + 300ms + 300ms = 900ms before render
async function Page() {
  const user = await getUser();
  const posts = await getPosts(user.id);
  const comments = await getComments(posts[0].id);
  ...
}

// GOOD: independent fetches run in parallel — bounded by the slowest one (~300ms)
async function Page() {
  const [user, posts] = await Promise.all([getUser(), getPosts()]);
  ...
}
```

When a fetch genuinely depends on a prior result (e.g., `comments` needs `posts[0].id`), that dependency is real and can't be parallelized — but it *can* be pushed into its own Suspense boundary so it doesn't block sibling content.

## 4. Image Optimization

`next/image` handles responsive `srcset` generation, lazy loading, AVIF/WebP re-encoding, and layout-shift prevention automatically — but only if configured correctly.

```tsx
// LCP element (hero/above-the-fold): use `priority` to skip lazy-loading
// and get preloaded via a <link rel="preload">. Omitting this on your LCP
// image is one of the most common causes of a slow LCP in Next.js apps.
<Image src="/hero.jpg" alt="Hero" width={1200} height={600} priority />

// Below-the-fold: default lazy loading is correct, don't add `priority` everywhere —
// preloading every image defeats the purpose and competes with the real LCP image
// for bandwidth.
<Image src="/testimonial.jpg" alt="Customer" width={400} height={300} />

// Fluid/responsive containers: `fill` + `sizes` avoids requesting a full-width
// image for a half-width slot on desktop.
<div className="relative aspect-video">
  <Image
    src="/product.jpg"
    alt="Product"
    fill
    sizes="(max-width: 768px) 100vw, 50vw"
    className="object-cover"
  />
</div>
```

```js
// next.config.js — enable modern formats and remote sources
const nextConfig = {
  images: {
    remotePatterns: [{ protocol: 'https', hostname: 'cdn.example.com' }],
    formats: ['image/avif', 'image/webp'], // AVIF ~50% smaller than JPEG at equal quality
  },
};
```

**CLS impact:** always pass explicit `width`/`height` (or use `fill` inside a sized container). An image without dimensions reserves zero layout space until it loads, then shoves content down — a classic CLS-score killer that can push a page from "Good" (<0.1) to "Poor" (>0.25) on a single hero image.

## 5. Font Optimization

```tsx
// app/layout.tsx
import { Inter } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',       // avoid invisible text (FOIT) while font loads
  variable: '--font-inter',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={inter.variable}>
      <body>{children}</body>
    </html>
  );
}
```

`next/font` self-hosts Google Fonts at build time — zero runtime requests to `fonts.googleapis.com`, no render-blocking external font CSS, and automatic `font-display` + fallback metric matching to prevent layout shift when the real font swaps in (this is what actually drives CLS down when custom fonts are involved, more than `display: swap` alone).

## 6. ISR and Revalidation Strategies

| Strategy | API | Use case |
|---|---|---|
| Static (build time) | default, no fetch options | Content that never changes post-deploy (legal pages) |
| Time-based ISR | `fetch(url, { next: { revalidate: 60 } })` | Product catalogs, blog listings — stale-for-N-seconds is acceptable |
| On-demand revalidation | `revalidatePath('/blog/[slug]')` / `revalidateTag('products')` | CMS webhook fires on publish — page updates within seconds, not minutes |
| Fully dynamic | `export const dynamic = 'force-dynamic'` or uncached fetch | Personalized/auth-walled content that must be fresh every request |

```tsx
// Time-based ISR: page is regenerated in the background at most once per 60s,
// visitors always get a cached (possibly-60s-stale) response — no one waits on regen.
async function getProducts() {
  const res = await fetch('https://api.example.com/products', {
    next: { revalidate: 60 },
  });
  return res.json();
}
```

```ts
// app/api/webhooks/cms/route.ts — on-demand revalidation triggered by a CMS publish event
import { revalidateTag } from 'next/cache';

export async function POST(request: Request) {
  const { tag } = await request.json();
  revalidateTag(tag); // invalidates every fetch() call tagged with `products`, `blog-posts`, etc.
  return Response.json({ revalidated: true, now: Date.now() });
}
```

**Decision rule:** default to time-based ISR with a revalidate window matched to how stale the business can tolerate the data being (product prices: 60s; marketing copy: 3600s). Reach for on-demand revalidation only when "stale for up to N seconds" is unacceptable (e.g., "editor hits publish and expects it live now"). Reach for `force-dynamic` only for genuinely per-request content — it opts the whole route out of the CDN cache and pushes full server compute onto every request, which is the difference between one origin hit per minute and one per visitor.

## 7. Bundle Analysis for Next.js

```bash
# Add and run @next/bundle-analyzer
npm install --save-dev @next/bundle-analyzer
ANALYZE=true npm run build
```

```js
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});
module.exports = withBundleAnalyzer(nextConfig);
```

What to look for in the output treemap:
- **Duplicate large deps across chunks** (e.g., `lodash` pulled in by three different feature modules) — usually fixable by deduping the import or moving the dep behind a dynamic import used by only one route.
- **A client component importing a huge server-only-safe library** (e.g., a date-formatting library with locale data, a markdown parser) that's only used for a one-time transform — move the transform into a Server Component or a Server Action so it never reaches the client bundle.
- **Icon libraries imported as a whole barrel** (`import { Icon } from 'lucide-react'` without `optimizePackageImports`) — configure tree-shaking:

```js
const nextConfig = {
  experimental: {
    optimizePackageImports: ['lucide-react', '@heroicons/react', 'date-fns'],
  },
};
```

- **`dynamic()` for below-the-fold or interaction-gated heavy components** (charting libraries, rich text editors, modals):

```tsx
import dynamic from 'next/dynamic';

const ChartPanel = dynamic(() => import('@/components/ChartPanel'), {
  loading: () => <ChartSkeleton />,
  ssr: false, // skip if the component needs browser-only APIs (canvas, window)
});
```

**Budget discipline:** hold each route to the profile's per-route JS budget (150 KB-gzip for `next-app-router`, per `SKILL.md`'s decision-engine profiles). Run the analyzer after adding any new dependency heavier than ~10KB gzipped, not just at release time — bundle bloat is easiest to catch at the commit that introduced it, not three sprints later during a perf audit.

## 8. Metadata and Static Generation for SEO-dependent routes

```tsx
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await getAllPostSlugs();
  return posts.map(post => ({ slug: post.slug })); // pre-render all known posts at build time
}

export async function generateMetadata({ params }: { params: { slug: string } }): Promise<Metadata> {
  const post = await getPost(params.slug);
  return { title: post.title, description: post.excerpt, openGraph: { images: [post.coverImage] } };
}
```

For SEO-critical, low-write content (the `astro-or-static` profile's territory when it's not a Next.js app, or the static-generation slice of `next-app-router` when it is), prefer `generateStaticParams` over runtime `force-dynamic` — it lets the CDN serve every page as static HTML with zero server compute per request, and it guarantees crawlers get fully-rendered HTML on the first request rather than depending on streaming behavior that not all crawlers handle well.
