# Next.js — 03. Data Fetching: fetch Caching, Revalidate, Server Actions, useActionState

> **What / Why / How** — fetch on the server when possible. Server Actions replace half of your `/api/*` routes.

---

## 1. The Mental Model — Where Should This Fetch Live?

In Next.js 14 App Router, you have **four** real places to fetch data, in increasing order of cost:

| Fetch site | Runs | Cost | Use for |
|------------|------|------|---------|
| **Server Component (`await fetch`)** | On the server, once per request (or once at build) | Cheapest — zero JS, server-cached | Page content, dashboards, lists |
| **Route Handler (`route.ts`)** | On the server when called from a client | Medium | Public API endpoints, third-party webhooks |
| **Server Action (`'use server'` function)** | On the server when called from a Client Component | Medium | Mutations from forms / button clicks |
| **Client Component (`useEffect`, TanStack Query)** | In the browser | Highest — adds JS, no server cache | Real-time updates, infinite scroll, optimistic UI |

**Default to Server Component fetches.** Most pages need only this. Reach for the others when you need interactivity, mutations, or live data.

---

## 2. Server Component Fetching — `await fetch` in JSX

### What

```tsx
// app/users/[id]/page.tsx — Server Component (no 'use client')
import { notFound } from 'next/navigation';

type Props = { params: { id: string } };

export default async function UserPage({ params }: Props) {
  const r = await fetch(`https://api.example.com/users/${params.id}`);
  if (r.status === 404) notFound();
  if (!r.ok) throw new Error(`Failed: ${r.status}`);
  const user = await r.json();

  return (
    <article>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </article>
  );
}
```

The component is `async`. Next.js `await`s it server-side, renders HTML, streams it to the browser. **No client JS for this UI** — it's pure HTML output.

### Why this beats `useEffect + fetch`

- Zero waterfall. The fetch starts when the route matches, not after the JS bundle loads.
- The data never leaks to the client — secrets and large internal payloads stay on the server.
- The browser receives only the rendered HTML; no spinner flicker.
- SEO crawlers see the full content immediately.

---

## 3. The Fetch Cache — Built In, Sometimes Surprising

Next.js patches the global `fetch` function. Every fetch becomes cacheable by default.

### Default behavior (Next.js 14.0–14.2)

| Call | Cache behavior |
|------|----------------|
| `await fetch(url)` (no options) | **Cached forever** — same as `{ cache: 'force-cache' }`. Result reused across requests until you redeploy or invalidate. |
| `await fetch(url, { cache: 'force-cache' })` | Same as default — explicit. |
| `await fetch(url, { cache: 'no-store' })` | Never cached. Refetched on every request. |
| `await fetch(url, { next: { revalidate: 60 } })` | Cached for 60 seconds. Older calls return stale cache; one request triggers regeneration. |
| `await fetch(url, { next: { tags: ['posts'] } })` | Cached, taggable. Invalidate with `revalidateTag('posts')`. |

### Important: this default flips in Next.js 15

`next@15` changes the default to **`cache: 'no-store'`**. Migrating later is a one-flag change, but the **examples in this file assume Next.js 14 defaults**. Be explicit: write `{ cache: 'force-cache' }` or `{ cache: 'no-store' }` in real code so behavior survives the upgrade.

### Real example — three caching modes side by side

```tsx
// app/dashboard/page.tsx
export default async function Dashboard() {
  // 1. User-specific — never cache
  const user = await fetch('https://api/me', {
    cache: 'no-store',
    headers: { Authorization: `Bearer ${process.env.API_TOKEN}` },
  }).then(r => r.json());

  // 2. Public, slow-changing — revalidate every 5 minutes (ISR-style)
  const products = await fetch('https://api/products', {
    next: { revalidate: 300, tags: ['products'] },
  }).then(r => r.json());

  // 3. Static at build time — cached forever (force-cache is the default in v14, but explicit is better)
  const featureFlags = await fetch('https://api/flags', {
    cache: 'force-cache',
  }).then(r => r.json());

  return <DashboardView {...{ user, products, featureFlags }} />;
}
```

### Three rendering modes that emerge from these flags

| You write | Next.js infers the mode |
|-----------|-------------------------|
| Only `cache: 'force-cache'` (or all-default) fetches | **Static** (SSG) — page generated at build |
| Any `cache: 'no-store'` fetch, or `cookies()` / `headers()` calls | **Dynamic** (SSR) — generated per request |
| `revalidate: N` in any fetch | **ISR** — generated, cached, regenerated on a timer |

You don't pick the mode explicitly; the fetches you make determine it. To force static even with cookies, see `export const dynamic = 'force-static'`.

---

## 4. Segment Config — Forcing the Mode Per Route

```tsx
// app/some-page/page.tsx
export const dynamic = 'force-dynamic'; // always SSR
// or
export const dynamic = 'force-static';  // always SSG (errors at build if dynamic APIs used)
// or
export const revalidate = 60;            // ISR with 60s window for the whole segment
```

When to use:
- `force-dynamic` — pages that depend on time, headers, or user identity but you forgot to add `cache: 'no-store'`.
- `force-static` — marketing pages where you want certainty in the build output.
- `revalidate = N` at the page level — applies to all fetches in that page that don't override it.

---

## 5. Invalidating the Cache — `revalidateTag` and `revalidatePath`

Use cases: after a mutation, you want the cached data to refresh next request.

### Tag-based invalidation

```tsx
// On read — tag the fetch
await fetch('https://api/posts', { next: { tags: ['posts'] } });

// On mutation — invalidate the tag
import { revalidateTag } from 'next/cache';

export async function createPostAction(formData: FormData) {
  'use server';
  await db.post.create({ data: { title: formData.get('title') as string } });
  revalidateTag('posts');
}
```

After `revalidateTag('posts')`, every fetch tagged `'posts'` refreshes on the next request. Granular, works across pages.

### Path-based invalidation

```tsx
import { revalidatePath } from 'next/cache';

revalidatePath('/dashboard');           // only the /dashboard segment
revalidatePath('/dashboard', 'layout'); // /dashboard and all children of its layout
revalidatePath('/users/[id]', 'page');  // dynamic path — needs the literal pattern
```

Use when you don't want to tag every fetch and just want "refresh this route".

---

## 6. Server Actions — Mutations Without Writing API Routes

Server Actions are functions marked `'use server'` that run on the server but are **callable from Client Components** like normal functions. Next.js wires up the RPC plumbing for you.

### What

```tsx
// app/posts/actions.ts
'use server';
import { revalidatePath } from 'next/cache';
import { z } from 'zod';

const Schema = z.object({
  title: z.string().min(1),
  body: z.string().min(1),
});

export async function createPost(formData: FormData) {
  const parsed = Schema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors };
  }
  await db.post.create({ data: parsed.data });
  revalidatePath('/posts');
  return { success: true };
}
```

### Wired to a `<form>` directly — works without JavaScript (progressive enhancement)

```tsx
// app/posts/new/page.tsx
import { createPost } from '../actions';

export default function NewPostPage() {
  return (
    <form action={createPost} className="space-y-3">
      <input name="title" placeholder="Title" />
      <textarea name="body" placeholder="Body" />
      <button type="submit">Publish</button>
    </form>
  );
}
```

`<form action={serverAction}>` is the magic. The form submits to the server action, the server runs it, the page revalidates, the user sees the new post. **No `e.preventDefault`, no `fetch`, no JSON, no API route.**

### Why Server Actions matter

- Eliminates an entire layer of `app/api/posts/route.ts` files for internal mutations.
- The same Zod schema is used by the action and (optionally) by the client form — single source of truth.
- Works without client JS — submitting a form pre-hydration still functions.
- Auto-invalidates the cache via `revalidatePath` / `revalidateTag`.

### When NOT to use Server Actions

- **Public API endpoints called by mobile apps or third parties** — write a `route.ts` instead. Server Actions are for internal RPC; the URL/contract isn't stable.
- **Long-running tasks** — keep the server action quick; offload long work to a queue (BullMQ 5, Inngest 0.x, Trigger.dev 3).
- **Streaming responses** — actions return one value. For streaming use a Route Handler.

---

## 7. `useActionState` — The Modern Form Hook (React 19 / Next.js 14.3+)

`useActionState` (formerly `useFormState`) tracks the action's return value and pending state from a Client Component.

```tsx
// app/posts/NewPostForm.tsx
'use client';

import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';
import { createPost } from './actions';

type State = { error?: Record<string, string[]>; success?: boolean } | null;

export function NewPostForm() {
  const [state, formAction] = useActionState<State, FormData>(
    async (_prev, formData) => createPost(formData),
    null
  );

  return (
    <form action={formAction}>
      <input name="title" />
      {state?.error?.title && <p className="text-red-600">{state.error.title[0]}</p>}

      <textarea name="body" />
      {state?.error?.body && <p className="text-red-600">{state.error.body[0]}</p>}

      <SubmitButton />
      {state?.success && <p className="text-green-600">Published.</p>}
    </form>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Publishing…' : 'Publish'}
    </button>
  );
}
```

`useFormStatus` is a separate hook (from `react-dom`) that any descendant of a `<form>` can read to know whether it's submitting. No prop drilling.

The pair `useActionState` + `useFormStatus` covers ~95% of form interaction patterns: validation errors, pending state, success messages, redirect.

### Old name: `useFormState`

In React 18 / Next.js 14.0–14.2 it was called `useFormState` from `react-dom`. React 19 renamed it to `useActionState` and moved it to `react`. Same behavior. The Next.js 14.3+ release notes call this out.

---

## 8. Route Handlers (`route.ts`) — When You Need a Real Endpoint

```ts
// app/api/posts/route.ts
import { NextResponse } from 'next/server';
import { z } from 'zod';

export async function GET() {
  const posts = await db.post.findMany();
  return NextResponse.json(posts);
}

export async function POST(req: Request) {
  const body = z.object({ title: z.string() }).parse(await req.json());
  const post = await db.post.create({ data: body });
  return NextResponse.json(post, { status: 201 });
}
```

Each exported HTTP method becomes the handler for that method. Available exports: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`.

### When to use a Route Handler instead of a Server Action

| Use Route Handler | Use Server Action |
|-------------------|-------------------|
| Public API consumed by mobile apps | Internal mutation from a form |
| Webhook receivers (Stripe, GitHub) | Form submit + revalidate |
| Streaming responses (`ReadableStream`) | Single-shot mutations |
| You need a stable URL contract | URL/contract is internal-only |
| File uploads with progress | Simple JSON-body actions |

For a real Next.js + Stripe webhook example, see the [official Next.js example](https://github.com/vercel/next.js/tree/canary/examples/with-stripe-typescript).

---

## 9. Combining Server Components + Client + TanStack Query

A real production pattern: **seed the cache server-side, hydrate the client.**

```tsx
// app/products/page.tsx (Server Component)
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query';
import { ProductList } from './ProductList';

export default async function ProductsPage() {
  const qc = new QueryClient();
  await qc.prefetchQuery({
    queryKey: ['products'],
    queryFn: () => fetch('https://api/products').then(r => r.json()),
  });

  return (
    <HydrationBoundary state={dehydrate(qc)}>
      <ProductList />
    </HydrationBoundary>
  );
}
```

```tsx
// app/products/ProductList.tsx (Client Component)
'use client';
import { useQuery } from '@tanstack/react-query';

export function ProductList() {
  const { data } = useQuery({
    queryKey: ['products'],
    queryFn: () => fetch('/api/products').then(r => r.json()),
  });
  return <ul>{data?.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

The page renders fully on the server with data, then the client picks up the same TanStack Query cache without re-fetching. Best of both worlds: instant first paint + client-side mutations and refetches.

Covered in deeper detail in `04.state-and-data/03-data-fetching.md`.

---

## 10. Reading Cookies, Headers, and Search Params

```tsx
import { cookies, headers } from 'next/headers';

export default async function MePage() {
  const cookieStore = cookies();
  const session = cookieStore.get('session')?.value;

  const ua = headers().get('user-agent') ?? '';

  // ...
}
```

Both functions:
- Are server-only (used in Server Components, Server Actions, Route Handlers).
- **Force the page to be dynamic** — calling them opts out of static rendering.
- Are synchronous in Next.js 14 but become async in Next.js 15. Write `await cookies()` for forward-compat in new code.

`searchParams` comes in as a prop:

```tsx
export default async function SearchPage({
  searchParams,
}: {
  searchParams: { q?: string; page?: string };
}) {
  const q = searchParams.q ?? '';
  const page = Number(searchParams.page ?? 1);
  // ...
}
```

---

## 11. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Page mysteriously cached at build despite changing data | Add `cache: 'no-store'` or `revalidate: N` to relevant fetches |
| `cookies()` / `headers()` thrown errors in cached page | Their use forces dynamic — check segment config and fetch options |
| Server Action returns `undefined` instead of state | Make sure it `return`s the state object the form expects |
| `useFormState` not found | React 19+ / Next.js 14.3+ renamed it to `useActionState` |
| Trying to use `useState` in a Server Component | Either remove the hook or add `'use client'` to the file |
| Calling a Server Action from outside a `<form>` | Add `'use client'` to the caller, then call it like a regular `await` function |
| Reading `process.env.SECRET` from a Client Component | Server-only env vars must come from Server Components/Actions; or prefix with `NEXT_PUBLIC_` if truly safe to expose |
| Mutating data and not seeing the UI refresh | Add `revalidatePath()` or `revalidateTag()` in the action |

---

## 12. Decision Tree

```
Is this a read or a write?
│
├─ Read — where does the data come from?
│   ├─ Public, slow-changing → fetch in Server Component, default cache or revalidate: N
│   ├─ User-specific / live → fetch in Server Component with cache: 'no-store'
│   ├─ Needs client-side refresh / mutations / optimistic UI → seed with prefetchQuery + TanStack Query
│   └─ Public API for mobile/3rd party → Route Handler (route.ts)
│
└─ Write — what's the trigger?
    ├─ HTML form submit / button click in our own UI → Server Action via <form action={...}>
    ├─ Public API (mobile, 3rd party, webhook) → Route Handler with POST
    └─ Long-running job → Server Action that enqueues to BullMQ / Inngest / Trigger.dev
```

---

## Summary

| Pattern | Use for |
|---------|---------|
| `async` Server Component + `await fetch` | Default for any data on a page |
| `cache: 'force-cache'` (default in v14) | Static / build-time data |
| `cache: 'no-store'` | User-specific or live data |
| `next: { revalidate: N }` | ISR-style timed regeneration |
| `next: { tags: [...] }` + `revalidateTag` | Targeted cache invalidation after mutations |
| Server Action `'use server'` + `<form action={...}>` | Form mutations, no API route needed |
| `useActionState` + `useFormStatus` | Validation errors, pending state, success messages |
| Route Handler `route.ts` | Public APIs, webhooks, streaming |
| `prefetchQuery` + `<HydrationBoundary>` | Server-seed + client TanStack Query |

| Rule | Why |
|------|-----|
| Default to Server Component fetching | Smaller bundle, faster paint, no waterfalls |
| Be explicit with `cache: ...` in Next.js 14 | Defaults flip in v15; explicit code survives the upgrade |
| Use Server Actions for internal mutations | Eliminates a whole API layer |
| Use Route Handlers for public APIs / webhooks | Stable URL contract, multi-method, streaming support |
| Always validate inputs (Zod) on the server | Even if you also validate on the client |
| `revalidatePath` / `revalidateTag` after a mutation | Otherwise UI shows stale data |

---

## Further reading

- [Next.js docs — Data Fetching, Caching, and Revalidating](https://nextjs.org/docs/app/building-your-application/data-fetching/fetching)
- [Next.js docs — Server Actions and Mutations](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)
- [Next.js docs — Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)
- [React docs — useActionState](https://react.dev/reference/react/useActionState)
- [React docs — useFormStatus](https://react.dev/reference/react-dom/hooks/useFormStatus)
- [TanStack Query — Advanced Server Rendering with Next.js](https://tanstack.com/query/v5/docs/framework/react/guides/advanced-ssr)
