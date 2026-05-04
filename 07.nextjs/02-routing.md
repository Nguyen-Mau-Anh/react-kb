# Next.js — 02. Routing: Layouts, loading.tsx, error.tsx, Parallel & Intercepting Routes

> **What / Why / How** — every special filename is a tool. Use them right and you write almost no router code.

---

## 1. The File-System Router in One Picture

```
app/
├── layout.tsx                    ← root layout (must define <html>, <body>)
├── page.tsx                      ← /
├── loading.tsx                   ← <Suspense> fallback for / and children
├── error.tsx                     ← Error Boundary for / and children
├── not-found.tsx                 ← 404 page
├── (marketing)/                  ← route group: organizes files, NOT in URL
│   ├── about/page.tsx            ← /about
│   └── pricing/page.tsx          ← /pricing
├── dashboard/
│   ├── layout.tsx                ← persists across /dashboard/*
│   ├── page.tsx                  ← /dashboard
│   ├── @analytics/page.tsx       ← parallel slot
│   ├── @team/page.tsx            ← parallel slot
│   └── settings/page.tsx         ← /dashboard/settings
├── photo/[id]/page.tsx           ← /photo/123
└── feed/
    ├── page.tsx                  ← /feed
    └── (.)photo/[id]/page.tsx    ← intercepting: shows /photo/123 as a modal over /feed
```

Every special filename has one job. Memorize the table from `07.nextjs/01-fundamentals.md` and you've already learned the API.

---

## 2. Layouts — Persistent UI

### What

A `layout.tsx` wraps its `page.tsx` and all nested segments. **Layouts persist across navigations** — only the inner `page.tsx` changes when you navigate within the layout's subtree.

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="grid grid-cols-[240px_1fr] min-h-screen">
      <aside className="border-r p-4">
        <DashboardNav />
      </aside>
      <main className="p-6">{children}</main>
    </div>
  );
}
```

Navigating from `/dashboard` to `/dashboard/settings`:

- `DashboardLayout` does **not** re-render. Its state (e.g. a collapsed-sidebar toggle, scroll position) is preserved.
- Only the inner `page.tsx` swaps. Faster, less flash.

### Why this matters vs. SPA routers

In a Vite SPA with `react-router@6`, persistence is achieved with `<Outlet />` and care about where state lives. Next.js makes it the default — write a layout file, get persistence.

### Layout rules

- Layouts must accept `children` and render them somewhere.
- Layouts can be `async` (Server Components) and `await fetch`.
- The root `app/layout.tsx` is mandatory and **must define `<html>` and `<body>`**.
- Nested layouts compose top-down: root → segment → subsegment.

### Real example — three nested layouts

```
app/layout.tsx                    ← <html> shell, fonts, theme provider
app/(app)/layout.tsx              ← logged-in chrome (header, footer)
app/(app)/projects/[id]/layout.tsx ← project tabs (Overview / Tasks / Settings)
app/(app)/projects/[id]/tasks/page.tsx
```

The user navigating `/projects/42/overview` → `/projects/42/tasks` keeps the project tabs mounted; only the page swaps.

---

## 3. Templates — Layouts That Re-mount

`template.tsx` is identical to `layout.tsx` except **it re-mounts on every navigation**. State inside it is lost.

When to use:
- You need an enter animation that re-runs every navigation.
- You want a `useEffect(() => log(pathname), [])` that fires per route.

In practice: rare. Stick with `layout.tsx` 99% of the time.

---

## 4. `loading.tsx` — Free Suspense Boundaries

### What

A `loading.tsx` next to a `page.tsx` is automatically wrapped in `<Suspense fallback={<Loading />}>`:

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <DashboardSkeleton />;
}
```

While the segment's Server Component is awaiting data, the skeleton renders. As soon as the data resolves, the real UI streams in.

### Real benefit — instant navigation feedback

Without `loading.tsx`: the user clicks a link → stays on the old page until data loads (~500ms blank).

With `loading.tsx`: the layout updates immediately to show the skeleton (~30ms perceived feedback), then the content streams in.

### Per-segment granularity

```
app/
├── dashboard/
│   ├── layout.tsx
│   ├── loading.tsx       ← while /dashboard segment loads
│   ├── page.tsx
│   └── settings/
│       ├── loading.tsx   ← while /dashboard/settings loads (sibling navigation)
│       └── page.tsx
```

Different segments can show different fallbacks. Granular skeletons feel much faster than one full-page spinner.

### When NOT to use loading.tsx

- The data is cached and resolves instantly — the skeleton flashes for 30ms which is worse than no skeleton.
- The page contains its own per-section `<Suspense>` boundaries (e.g. streaming a slow chart) — `loading.tsx` may interfere.

---

## 5. `error.tsx` — Per-Segment Error Boundary

### What

`error.tsx` is a React Error Boundary that catches errors thrown during render or in Server Components inside its segment.

```tsx
// app/dashboard/error.tsx
'use client'; // error boundaries must be Client Components

import { useEffect } from 'react';

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Send to Sentry / DataDog
    console.error('Dashboard error:', error);
  }, [error]);

  return (
    <div className="p-6">
      <h2 className="text-xl font-semibold">Something went wrong</h2>
      <p className="text-sm text-gray-600">{error.message}</p>
      <button onClick={reset} className="mt-3 rounded bg-blue-600 px-3 py-1.5 text-white">
        Try again
      </button>
    </div>
  );
}
```

### What it catches

- Render errors in Server Components below this segment.
- Render errors in Client Components below this segment.
- Errors thrown inside Server Actions (when not handled).
- Unhandled rejections in `loader`-style fetches inside the segment.

### What it does NOT catch

- Errors in `layout.tsx` of the **same segment**. The error bubbles to the parent's `error.tsx`.
- Errors in `error.tsx` itself (obviously).
- Errors during initial render of the root layout — for those, use `app/global-error.tsx` (must also define `<html>` and `<body>`).

### Real layered pattern

```
app/global-error.tsx       ← catches errors in app/layout.tsx itself
app/error.tsx              ← catches errors in / and unhandled bubble-ups
app/dashboard/error.tsx    ← catches errors only inside /dashboard subtree
```

The closest `error.tsx` wins. Place them where you can render a useful recovery UI.

---

## 6. `not-found.tsx` — 404 UI

### What

```tsx
// app/not-found.tsx
import Link from 'next/link';

export default function NotFound() {
  return (
    <div className="flex flex-col items-center justify-center min-h-[60vh]">
      <h1 className="text-4xl font-bold">404</h1>
      <p>That page doesn&apos;t exist.</p>
      <Link href="/" className="mt-4 underline">Go home</Link>
    </div>
  );
}
```

### Two ways it's triggered

```tsx
// 1. Manually from a Server Component
import { notFound } from 'next/navigation';

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await db.product.findUnique({ where: { id: params.id } });
  if (!product) notFound(); // throws to the nearest not-found.tsx
  return <ProductView product={product} />;
}

// 2. Automatic for routes that don't match any file in app/
```

You can also place per-segment `not-found.tsx` files (e.g. `app/dashboard/not-found.tsx`) for context-specific 404 UI.

---

## 7. Route Groups — `(folder)` for Organization Without URL

A folder named `(group)` does not appear in the URL but lets you:

- Apply a different layout to a sub-section of routes that share a URL space.
- Organize files for clarity without affecting routing.

### Real example — split layouts for marketing vs app

```
app/
├── (marketing)/
│   ├── layout.tsx          ← marketing chrome (top nav, footer)
│   ├── page.tsx            ← /
│   ├── about/page.tsx      ← /about
│   └── pricing/page.tsx    ← /pricing
└── (app)/
    ├── layout.tsx          ← app chrome (sidebar, user menu)
    ├── dashboard/page.tsx  ← /dashboard
    └── settings/page.tsx   ← /settings
```

Both `(marketing)` and `(app)` are at the URL root, but they get totally different layouts. This is the cleanest way to split "marketing site" from "logged-in app" in one Next.js project.

### Caveats

- You can't have two routes resolving to the same URL across groups (e.g. both `(marketing)/page.tsx` and `(app)/page.tsx`). Build will fail.
- Route groups don't share their layouts across the boundary — moving from `/about` (in marketing) to `/dashboard` (in app) re-mounts the layout.

---

## 8. Dynamic Segments — `[param]`, `[...catchAll]`, `[[...optional]]`

| Pattern | Matches | `params` value |
|---------|---------|----------------|
| `app/blog/[slug]/page.tsx` | `/blog/intro` | `{ slug: 'intro' }` |
| `app/shop/[...path]/page.tsx` | `/shop/men/shoes/running` | `{ path: ['men', 'shoes', 'running'] }` |
| `app/help/[[...slug]]/page.tsx` | `/help`, `/help/billing`, `/help/billing/refunds` | `{ slug?: string[] }` (optional) |

### Read params

```tsx
type Props = { params: { slug: string } };

export default async function BlogPost({ params }: Props) {
  const post = await db.post.findUnique({ where: { slug: params.slug } });
  if (!post) notFound();
  return <article>{post.body}</article>;
}
```

### Static generation — `generateStaticParams`

For build-time pre-rendering (replaces `getStaticPaths` from the Pages Router):

```tsx
export async function generateStaticParams() {
  const posts = await db.post.findMany({ select: { slug: true } });
  return posts.map(p => ({ slug: p.slug }));
}
```

Next.js calls this at build, generates one HTML per slug, and serves them as static files.

---

## 9. Parallel Routes — Render Multiple Pages Side-by-Side

### What

Folders prefixed with `@` are **named slots** that get passed as props to the parent `layout.tsx`.

```
app/dashboard/
├── layout.tsx
├── page.tsx              ← children prop
├── @analytics/page.tsx   ← analytics prop
└── @team/page.tsx        ← team prop
```

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
  analytics,
  team,
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  team: React.ReactNode;
}) {
  return (
    <div className="grid grid-cols-2 gap-4">
      <div>{children}</div>
      <div>{analytics}</div>
      <div>{team}</div>
    </div>
  );
}
```

### Real use cases

- **Independent error boundaries / loading states per panel.** Analytics can fail without the team list crashing.
- **Tab-like UIs without re-rendering siblings.** Each slot navigates its own URL state.
- **Modal-over-page patterns** (combined with intercepting routes — next section).

### `default.tsx` — required when slots have soft routing

Each parallel slot needs a `default.tsx` to render when no other route matches that slot during a soft (client-side) navigation. Without it, you'll see "no UI for this slot" errors.

```tsx
// app/dashboard/@analytics/default.tsx
export default function Default() {
  return null; // or a placeholder
}
```

---

## 10. Intercepting Routes — Modals Over a Route Without Losing Context

### What

A folder prefixed with `(.)`, `(..)`, or `(..)(..)` **intercepts** a route from the file tree and renders it differently in the current layout's slot. The URL changes; the page underneath stays mounted.

```
app/
├── feed/
│   ├── page.tsx                  ← /feed
│   └── @modal/
│       ├── default.tsx           ← null when no modal
│       └── (.)photo/[id]/page.tsx ← intercepts /photo/:id when navigated from /feed
└── photo/
    └── [id]/page.tsx             ← /photo/:id (real page, used on direct visit)
```

The convention prefixes: `(.)` = same level, `(..)` = one up, `(..)(..)` = two up, `(...)` = root.

### Real use case — Instagram-style photo modal

User on `/feed`:
- Clicks a photo → URL becomes `/photo/123`.
- Feed stays mounted underneath; a modal opens with the photo.
- User shares the link → opens directly to `/photo/123` (the real page, not the modal).
- User refreshes — same: real page, no modal.

### How the layout uses it (with parallel routes)

```tsx
// app/feed/layout.tsx
export default function FeedLayout({
  children,
  modal,
}: { children: React.ReactNode; modal: React.ReactNode }) {
  return (
    <>
      {children}
      {modal}        {/* renders <PhotoModal> when intercepted */}
    </>
  );
}
```

The modal page renders inside its own slot and can dismiss back to the underlying route.

This pattern is used by Vercel's own dashboard, Linear's web app, and the Next.js example repo (`with-parallel-routes-modal`).

---

## 11. Middleware — Edge-Layer Routing Logic

`middleware.ts` at the project root runs **before** routing for matching paths. It can rewrite, redirect, or set cookies/headers.

```ts
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Auth check for /admin/* using cookies
  if (request.nextUrl.pathname.startsWith('/admin')) {
    const token = request.cookies.get('session')?.value;
    if (!token) {
      const url = request.nextUrl.clone();
      url.pathname = '/login';
      url.searchParams.set('next', request.nextUrl.pathname);
      return NextResponse.redirect(url);
    }
  }
  return NextResponse.next();
}

export const config = {
  matcher: ['/admin/:path*', '/dashboard/:path*'],
};
```

Middleware runs on the **edge runtime** (a smaller, faster JS env — not full Node.js). Restrictions:
- No `fs`, no native Node modules.
- Must be lightweight (executes per request).
- For DB access, prefer doing it in Server Components/Server Actions; use middleware only for auth gating, geolocation routing, A/B testing.

`next-auth@5` middleware integration is covered in `07.nextjs/04-auth.md`.

---

## 12. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Adding `'use client'` to `layout.tsx` because of a child | Push it to the leaf component instead |
| Returning a fragment without `<html>` and `<body>` from root layout | Mandatory in `app/layout.tsx` |
| Forgetting `default.tsx` in a parallel slot | Hard-to-debug "missing slot UI" errors on soft nav |
| Putting `error.tsx` without `'use client'` | Must be a Client Component |
| Using `getServerSideProps` / `getStaticProps` | Pages Router APIs — don't exist in App Router |
| Using `next/router` from App Router | Use `next/navigation` |
| Forgetting `notFound()` returns `never` and continuing logic | It throws, so code after is dead |
| Two pages resolving to the same URL across route groups | Build error: `Two parallel pages resolve to the same path` |
| Route group (`(name)`) with same name as a real folder | Confusing — pick clearly distinct names |

---

## Summary

| File / Folder | What it does |
|---------------|--------------|
| `layout.tsx` | Persistent wrapper for the segment + children |
| `template.tsx` | Re-mounting wrapper (rare) |
| `loading.tsx` | Auto `<Suspense>` fallback for the segment |
| `error.tsx` | Error Boundary for the segment (Client Component) |
| `global-error.tsx` | Error Boundary for `app/layout.tsx` itself |
| `not-found.tsx` | 404 UI; triggered by `notFound()` or unmatched route |
| `(group)/` | Folder organization without a URL segment |
| `[param]/`, `[...slug]/`, `[[...slug]]/` | Dynamic segments, catch-all, optional catch-all |
| `@slot/` | Parallel route slot, passed as a named prop to layout |
| `(.)route/`, `(..)route/`, `(...)route/` | Intercepting route — modal-over-page, etc. |
| `middleware.ts` (root) | Edge-runtime pre-route logic: auth, redirects, A/B |

| Rule | Why |
|------|-----|
| Use route groups for marketing vs app split | Two layouts at the same URL root |
| Add `loading.tsx` near every async page | Free skeleton, instant navigation feedback |
| Place `error.tsx` per segment, not just root | Recovery without crashing the whole app |
| Use parallel + intercepting routes for modals | Shareable URLs with stateful underneath |
| Keep middleware tiny | Edge runtime is restricted |

---

## Further reading

- [Next.js docs — Routing fundamentals](https://nextjs.org/docs/app/building-your-application/routing)
- [Next.js docs — Loading UI and Streaming](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)
- [Next.js docs — Error Handling](https://nextjs.org/docs/app/building-your-application/routing/error-handling)
- [Next.js docs — Parallel Routes](https://nextjs.org/docs/app/building-your-application/routing/parallel-routes)
- [Next.js docs — Intercepting Routes](https://nextjs.org/docs/app/building-your-application/routing/intercepting-routes)
- [Next.js docs — Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware)
