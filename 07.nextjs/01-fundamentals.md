# Next.js — 01. Fundamentals: App Router, File Conventions, Server Components

> **What / Why / How** — Next.js 14 App Router is the default for new React apps in 2026. Understand RSC vs Client Components or you will fight the framework forever.

---

## 1. What Next.js Is — and What It's Not

### What

`next@14.2` is a React framework with:
- A file-system **router** (App Router or legacy Pages Router).
- A **bundler** (Turbopack in dev, Webpack 5 in production).
- A **renderer** (Server Components, Server-Side Rendering, Static Site Generation, Incremental Static Regeneration, streaming).
- Built-in **image optimization** (`next/image`), **font hosting** (`next/font`), **scripts** (`next/script`).
- A **server runtime** for `route.ts` API endpoints, middleware, and server actions.

### What it isn't

- A backend framework. The server runtime is request/response only — no jobs, queues, websockets out of the box. For those, pair with Bun, Hono 4, an Express app, or a separate service.
- A static-site generator. It can do SSG, but its primary mode is hybrid SSR/streaming.
- React Native. Different stack entirely; covered in a future ecosystem topic.

### App Router vs Pages Router — the 2026 split

| | **App Router** (`app/`) | **Pages Router** (`pages/`) |
|--|-------------------------|------------------------------|
| Released | Next.js 13.4 stable (May 2023) | Next.js 1 (2016) |
| React Server Components | ✅ default | ❌ |
| Streaming SSR + Suspense | ✅ default | Limited |
| Layouts | Nested, persistent | Single `_app.tsx` |
| Data fetching | `await` in Server Components, Server Actions | `getServerSideProps`, `getStaticProps` |
| Parallel + intercepting routes | ✅ | ❌ |
| Recommended for new code | ✅ | ❌ |

**For all new projects in 2026, use the App Router.** The Pages Router is maintenance-only — Vercel maintains it for legacy migrations but recommends App Router for everything new.

---

## 2. Why Next.js (vs Plain Vite + React)

| Concern | Vite + React 18 SPA | Next.js 14 App Router |
|---------|---------------------|------------------------|
| First paint | All-client-rendered → blank screen until JS loads | SSR + streaming → HTML on first byte |
| SEO | Hard for content sites | Default-ready, server-rendered HTML |
| Code splitting | Manual with `lazy()` per route | Automatic per-route |
| Image optimization | Need `vite-imagetools@7` or similar | Built-in `next/image` (WebP/AVIF, responsive sizes, lazy loading) |
| Font loading without layout shift | DIY with `@fontsource/*` | Built-in `next/font` (self-hosted Google Fonts, no CLS) |
| Backend route | Need a separate server | Built-in `route.ts` |
| Auth | Pick a library yourself | `next-auth@5` (Auth.js) integrates cleanly |
| Edge deployment | Need to wire it up | One-line `runtime: 'edge'` per route |
| Deployment | Deploy a static bundle | One-click on Vercel; Docker on others (covered later) |

When Vite still wins:
- A pure SPA (admin dashboard behind login, no SEO need).
- An Electron / Tauri / browser-extension build (Vite is the path of least resistance).
- A library or design system (no app shell to scaffold).

For everything else — content sites, SaaS, e-commerce, internal apps with public pages — Next.js 14 saves weeks of plumbing.

---

## 3. Project Setup

```bash
npx create-next-app@14.2 my-app
```

The CLI prompts:

| Prompt | Pick (2026 default) |
|--------|---------------------|
| TypeScript | **Yes** |
| ESLint | Yes |
| Tailwind CSS | **Yes** (covered in `05.routing-and-styling/02-styling.md`) |
| `src/` directory | Personal preference; either works |
| App Router | **Yes** |
| Customize default import alias | `@/*` (default — fine) |
| Turbopack for `next dev` | **Yes** (Turbopack is stable for `next dev` in v14.2; Webpack still handles `next build`) |

The resulting structure (relevant parts):

```
my-app/
├── app/                        ← App Router root
│   ├── layout.tsx              ← root layout (mandatory)
│   ├── page.tsx                ← / route
│   ├── globals.css             ← global styles
│   └── favicon.ico
├── public/                     ← static assets served at /
├── next.config.js              ← framework config
├── tsconfig.json
├── tailwind.config.ts
└── package.json
```

---

## 4. The App Router File Conventions

The router is file-system based. Specific filenames inside any folder under `app/` have **special meaning**:

| File | Purpose |
|------|---------|
| `page.tsx` | The route's UI. Required to make a folder a route. |
| `layout.tsx` | Wraps `page.tsx` and nested routes. Persists across navigations. |
| `loading.tsx` | Auto-wraps the segment in `<Suspense>` with this fallback. |
| `error.tsx` | React error boundary for the segment. Must be a Client Component. |
| `not-found.tsx` | Rendered when `notFound()` is called or a `404` would otherwise show. |
| `template.tsx` | Like layout, but re-mounts on every navigation (rare). |
| `route.ts` | A backend route handler (replaces Pages Router API routes). |
| `default.tsx` | Used by parallel routes when no slot match exists. |
| `middleware.ts` (project root, not in `app/`) | Edge middleware running before routing. |

Folder conventions:

| Folder name | Meaning |
|-------------|---------|
| `app/about/page.tsx` | route `/about` |
| `app/blog/[slug]/page.tsx` | dynamic route `/blog/:slug` |
| `app/shop/[...path]/page.tsx` | catch-all route `/shop/anything/here` |
| `app/(marketing)/about/page.tsx` | route `/about` — `(marketing)` is a route group, not in URL |
| `app/_components/Button.tsx` | private folder — never a route |
| `app/@analytics/page.tsx` | parallel route slot named `analytics` (covered in `07.nextjs/02-routing.md`) |
| `app/(.)photo/[id]/page.tsx` | intercepting route — for modals over a route |

This file-system surface is the entire routing API. Most beginner confusion comes from not knowing which file does what — keep this table handy.

---

## 5. The First Real Pages

```tsx
// app/layout.tsx — the root layout (required, must define <html> and <body>)
import type { Metadata } from 'next';
import './globals.css';

export const metadata: Metadata = {
  title: 'My App',
  description: 'Built with Next.js 14',
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className="bg-white text-gray-900 antialiased">
        <header className="border-b">My App</header>
        <main>{children}</main>
      </body>
    </html>
  );
}
```

```tsx
// app/page.tsx — the / route
export default function HomePage() {
  return <h1 className="p-6 text-3xl">Welcome</h1>;
}
```

```tsx
// app/about/page.tsx — the /about route
export default function AboutPage() {
  return <h1 className="p-6">About</h1>;
}
```

```tsx
// app/blog/[slug]/page.tsx — dynamic route
type Props = { params: { slug: string } };
export default function BlogPost({ params }: Props) {
  return <h1>Post: {params.slug}</h1>;
}
```

Run `next dev` and the dev server picks these up automatically. No router config to write.

---

## 6. The Big Mental Model — React Server Components (RSC)

This is the single concept that determines whether you'll be productive in Next.js 14 or fight it forever.

### Two component runtimes coexist in one tree

| | **Server Component** (default in `app/`) | **Client Component** (`'use client'`) |
|--|------------------------------------------|----------------------------------------|
| Runs where | On the server (Node.js or edge) | On the server during SSR + on the client during hydration |
| Can be `async` | ✅ `await fetch(...)` directly | ❌ |
| Can use hooks (`useState`, `useEffect`, etc.) | ❌ | ✅ |
| Can read environment secrets | ✅ | ❌ (browser sees them) |
| Can import `fs`, `db` clients | ✅ | ❌ |
| Sent to the browser as JS | ❌ Only HTML output | ✅ Full bundle |
| Onclick handlers, browser events | ❌ | ✅ |

### How they nest

A Server Component **can** render a Client Component. A Client Component **cannot directly render** a Server Component, but it can accept one through a `children` prop (composition pattern).

```tsx
// ✅ Server Component renders Client Component
// app/page.tsx (Server Component by default)
import { ClientCounter } from './ClientCounter';

export default async function Page() {
  const data = await fetch('https://api/data').then(r => r.json());
  return (
    <>
      <h1>{data.title}</h1>
      <ClientCounter />
    </>
  );
}
```

```tsx
// app/ClientCounter.tsx
'use client';
import { useState } from 'react';

export function ClientCounter() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>{n}</button>;
}
```

The `'use client'` directive at the top of a file marks **every component exported from that file** (and every component imported transitively into the resulting client bundle) as Client Components. You don't sprinkle the directive inside files — only at the top.

### Real rules of thumb

- **Default to Server.** Don't add `'use client'` unless the component needs hooks, browser APIs, or event handlers.
- **Push `'use client'` as far down the tree as possible.** A `<Layout>` does not need to be client just because one button inside it does. Make the button client; keep the layout server.
- **Pass server-fetched data through props** to client islands.

### Why RSC matters

- Less JavaScript shipped to the browser. Server-only components contribute zero JS to the client bundle.
- Direct database / filesystem / private API access in components without a separate API layer.
- Built-in streaming via `<Suspense>` — slow data doesn't block the rest of the page.

### A common mistake

```tsx
// ❌ Adding 'use client' to a layout.tsx because a child needs state
'use client';
export default function RootLayout({ children }) { /* ... */ }
```

This drags the layout (and the entire tree below it that doesn't already opt-in) into client bundling. Move `'use client'` to the leaf component that actually needs interactivity.

---

## 7. Linking Between Routes — `<Link>`

```tsx
import Link from 'next/link';

<Link href="/about" className="underline">About</Link>
<Link href={`/blog/${slug}`} prefetch>Read more</Link>
```

`next/link` does:
- Client-side navigation (no full reload).
- Automatic prefetch of the linked route's JS + data when the link enters the viewport.
- Prefetch can be disabled with `prefetch={false}` for very large pages.

Don't use `<a href>` for internal navigation — it does a full reload and loses your client state.

---

## 8. Programmatic Navigation — `useRouter`

```tsx
'use client';
import { useRouter } from 'next/navigation';

function LoginButton() {
  const router = useRouter();
  return (
    <button onClick={async () => {
      await api.signIn();
      router.push('/dashboard');
      router.refresh(); // re-fetch Server Components for the new state
    }}>
      Sign in
    </button>
  );
}
```

Imports come from `next/navigation` (not `next/router` — that's the legacy Pages Router).

| Hook (from `next/navigation`) | Purpose |
|-------------------------------|---------|
| `useRouter()` | `push`, `replace`, `refresh`, `back`, `forward` |
| `usePathname()` | Current pathname (e.g. `'/blog/intro'`) |
| `useSearchParams()` | Read `?foo=bar` |
| `useParams()` | Read dynamic segment values |

`router.refresh()` is the underrated one — it re-runs the current route's Server Components, re-fetching all the data, without a full page reload.

---

## 9. Static Assets

| Asset | Where | Reference |
|-------|-------|-----------|
| Public files (`favicon.ico`, `robots.txt`) | `public/` | `/favicon.ico` |
| Optimized images | Anywhere — referenced via `next/image` | `<Image src="/hero.png" width={1200} height={600} alt="" />` |
| Self-hosted fonts | `next/font` import | covered briefly below |

```tsx
// app/layout.tsx — Inter loaded via next/font (no FOIT/FOUT, no CLS)
import { Inter } from 'next/font/google';

const inter = Inter({ subsets: ['latin'], display: 'swap' });

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={inter.className}>
      <body>{children}</body>
    </html>
  );
}
```

`next/font/google` self-hosts the font at build time, so production has zero round trips to `fonts.googleapis.com`. This is one of the highest-leverage performance wins built into the framework.

---

## 10. Configuration — `next.config.js`

The most common settings:

```js
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'images.example.com' },
    ],
  },
  experimental: {
    typedRoutes: true,        // type-checked <Link href> values
    serverActions: true,      // default true in v14 — covered in 07.nextjs/03
  },
};

module.exports = nextConfig;
```

`typedRoutes` is worth turning on early — it makes Next type-check every `<Link href="...">` against the actual file system, catching typos at build time.

---

## 11. Common Pitfalls in the First Week

| Pitfall | Fix |
|---------|-----|
| Using `useState` in a Server Component | Add `'use client'` to the file (and ideally only the leaf, not a parent layout) |
| Using `next/router` instead of `next/navigation` | `next/router` is Pages Router only |
| Reading `window` or `document` at module top-level | Server runs on Node.js — guard with `typeof window !== 'undefined'` or move to `useEffect` in a Client Component |
| Importing `'fs'` in a Client Component | Move that code to a Server Component or to a `route.ts` |
| Forgetting `<html>` and `<body>` in `app/layout.tsx` | Required — Next throws at build if missing |
| Calling `fetch` and returning data from a custom hook in `useEffect` for content that could be on the server | Fetch in the Server Component itself; pass data as props |
| Putting `metadata` in a Client Component file | Metadata exports must live in Server Components |
| Using `process.env.X` in a Client Component without `NEXT_PUBLIC_` prefix | Only `NEXT_PUBLIC_*` env vars are inlined into the browser bundle |

---

## 12. Decision Tree — "Server or Client?"

```
Does this component...
├─ use hooks (useState, useEffect, useTransition, etc.)?
│    YES → 'use client'
├─ use browser-only APIs (window, document, navigator, IntersectionObserver)?
│    YES → 'use client'
├─ have onClick / onChange / onSubmit etc. event handlers?
│    YES → 'use client'
├─ depend on real-time/local mutations like a Zustand store?
│    YES → 'use client'
└─ otherwise (display data, layout, navigation, server-fetched content)
     → leave as Server Component (the default — no directive)
```

---

## Summary

| Concept | What you get |
|---------|--------------|
| App Router | File-system routing with layouts, loading, error, parallel/intercepting |
| Server Components | `await fetch` directly, zero JS to browser, secret-safe |
| Client Components | Hooks, events, browser APIs — opt in with `'use client'` |
| `next/link` | Prefetched client-side navigation |
| `next/image`, `next/font`, `next/script` | Built-in optimizations |
| `route.ts` | Backend handlers in the same project |
| `next/navigation` | Modern hooks: `useRouter`, `usePathname`, `useSearchParams` |
| `metadata` export | SEO without manual `<head>` work (covered in `07.nextjs/07-seo-metadata.md`) |

| Rule | Why |
|------|-----|
| Default to Server Components | Smaller bundle, secret-safe, fewer hydration costs |
| Put `'use client'` only on leaves | Avoid pulling parents into client bundling |
| Use `next/navigation`, not `next/router`, in App Router | The latter is Pages Router only |
| Use `next/font` and `next/image` from day one | Free wins on Core Web Vitals |
| Enable `typedRoutes` | Catches `<Link>` typos at build time |

---

## Further reading

- [Next.js docs — App Router](https://nextjs.org/docs/app)
- [React docs — Server Components](https://react.dev/reference/rsc/server-components)
- [Next.js docs — File Conventions](https://nextjs.org/docs/app/api-reference/file-conventions)
- [Vercel blog — Why we built the App Router](https://vercel.com/blog/nextjs-app-router-data-fetching)
