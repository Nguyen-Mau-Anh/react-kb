# State & Data — 03. Data Fetching: TanStack Query v5 vs SWR 2 vs Raw fetch

> **What / Why / How** — server state has different rules than client state. Use a real cache, not `useEffect`.

---

## 1. Why "Server State" Is a Distinct Problem

When data lives on a server, your client copy is by definition **stale** — the source of truth is somewhere else. Local state libraries (Zustand, Redux Toolkit) treat data as authoritative; server data isn't. You need:

| Concern | Why it's specific to server data |
|---------|-----------------------------------|
| **Caching** | Two components asking for the same user shouldn't fire two requests. |
| **Deduplication** | Three near-simultaneous mounts → one network call. |
| **Stale-while-revalidate (SWR)** | Show cached data instantly, refetch in the background, then update if it changed. |
| **Background refetch on focus / reconnect** | When the user switches back to the tab, data may be old. |
| **Retry with backoff** | Transient `503` shouldn't surface as an error. |
| **Pagination & infinite scroll** | Page state, cursor state, prefetching the next page. |
| **Optimistic updates** | Apply mutation locally, roll back if the server rejects. |
| **Garbage collection** | Drop unused cache entries after N minutes. |

Doing all of this with `useEffect + fetch` is how every "useFetch" wrapper turns into a 600-line module that's still wrong. Use a real library.

---

## 2. The Real Options

| Library | Released | npm | Bundle gzipped | Used by |
|---------|----------|-----|----------------|---------|
| `@tanstack/react-query@5.28` | 2024 (v5 line) | `@tanstack/react-query` | ~13 KB | `shadcn/ui` examples, T3 Stack, Linear web client |
| `swr@2.2` | Vercel team | `swr` | ~5 KB | Vercel.com dashboard, Next.js examples |
| `RTK Query` (part of `@reduxjs/toolkit@2.2`) | Redux team | `@reduxjs/toolkit` | RTK already there | Teams already using Redux |
| Raw `fetch` + `useEffect` | Built-in | — | 0 | Don't, except for one-offs |
| Next.js Server Components `await fetch` | Next.js 14+ | `next` | 0 | Next.js App Router pages — covered in `07.nextjs/03-data-fetching.md` |

The two real client-side picks: **TanStack Query v5** and **SWR 2**.

---

## 3. Side-by-Side: TanStack Query v5 vs SWR 2

| Feature | `@tanstack/react-query@5.28` | `swr@2.2` |
|---------|------------------------------|-----------|
| Conceptual unit | "Query" (queryKey + queryFn) | "Hook" (key + fetcher) |
| Mutations | `useMutation` with onMutate/onError/onSettled | `useSWRMutation` (separate package surface) |
| Optimistic updates | First-class `onMutate` rollback pattern | Possible but more manual |
| Infinite / cursor pagination | `useInfiniteQuery` | `useSWRInfinite` |
| DevTools | Excellent (`@tanstack/react-query-devtools@5`) | Basic (built into the SWR cache) |
| Framework agnostic | Yes (React, Solid, Vue, Svelte adapters) | React-only |
| Server-rendering helpers | `dehydrate` / `HydrationBoundary` | `unstable_serialize` + `<SWRConfig fallback>` |
| Bundle size gzipped | ~13 KB | ~5 KB |
| Backed by | TanStack (Tanner Linsley) | Vercel |
| Learning curve | Steeper but more powerful | Tiny API surface |

**Default recommendation**: `@tanstack/react-query@5.28` for any non-trivial app. It's worth the extra 8 KB for the mutation API alone. Pick `swr@2.2` if you're shipping a Vercel-style dashboard with mostly read-only data and want minimum API surface.

---

## 4. TanStack Query v5 — Real Examples

### Install + setup

```bash
npm i @tanstack/react-query@5.28 @tanstack/react-query-devtools@5.28
```

```tsx
// app/providers.tsx
'use client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  // useState ensures one client per browser session, not per render
  const [client] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 30_000,    // 30s — fresh; no refetch on remount within this window
        gcTime: 5 * 60_000,   // 5min — drop from cache after this idle period
        retry: 2,
        refetchOnWindowFocus: true,
      },
    },
  }));
  return (
    <QueryClientProvider client={client}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

In Next.js 14 App Router, mount this in a Client Component (e.g. `app/providers.tsx` with `'use client'`), import it from `app/layout.tsx`.

### Reading data — `useQuery`

```tsx
import { useQuery } from '@tanstack/react-query';

type User = { id: string; name: string; email: string };

function UserCard({ id }: { id: string }) {
  const { data, isPending, isError, error, refetch, isFetching } = useQuery<User>({
    queryKey: ['user', id],
    queryFn: async ({ signal }) => {
      const r = await fetch(`/api/users/${id}`, { signal });
      if (!r.ok) throw new Error('Failed');
      return r.json();
    },
    enabled: !!id, // skip until id is truthy
  });

  if (isPending) return <Skeleton />;
  if (isError) return <ErrorBox message={error.message} retry={() => refetch()} />;

  return (
    <div>
      <h1>{data.name}</h1>
      <p>{data.email}</p>
      {isFetching && <small>Refreshing…</small>}
    </div>
  );
}
```

Key points (real-world facts that trip people up):

- The `queryKey` array is the cache identity. **Same key in two components → one network call**, both see the same data.
- Mounting `<UserCard id="abc" />` twice on the same page during the `staleTime` window: zero duplicate fetches.
- The `signal` argument is passed automatically; passing it to `fetch` enables abort on unmount.
- `isPending` (v5 name) replaces v4's `isLoading`. If you see tutorials with `isLoading`, they're v4 — the v5 migration is documented at the [TanStack v5 migration guide](https://tanstack.com/query/v5/docs/framework/react/guides/migrating-to-v5).

### Mutations with optimistic update + rollback

Real example: toggle a todo's `done` flag instantly, roll back if the server rejects.

```tsx
import { useQueryClient, useMutation } from '@tanstack/react-query';

type Todo = { id: string; text: string; done: boolean };

function useToggleTodo() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: async (todo: Todo) => {
      const r = await fetch(`/api/todos/${todo.id}`, {
        method: 'PATCH',
        body: JSON.stringify({ done: !todo.done }),
      });
      if (!r.ok) throw new Error('Update failed');
      return r.json() as Promise<Todo>;
    },
    onMutate: async (todo) => {
      await qc.cancelQueries({ queryKey: ['todos'] });
      const prev = qc.getQueryData<Todo[]>(['todos']);
      qc.setQueryData<Todo[]>(['todos'], (old) =>
        old?.map(t => t.id === todo.id ? { ...t, done: !t.done } : t) ?? []
      );
      return { prev };
    },
    onError: (_err, _todo, ctx) => {
      // Roll back on failure
      if (ctx?.prev) qc.setQueryData(['todos'], ctx.prev);
    },
    onSettled: () => {
      qc.invalidateQueries({ queryKey: ['todos'] });
    },
  });
}

// Usage
function TodoRow({ todo }: { todo: Todo }) {
  const toggle = useToggleTodo();
  return (
    <li>
      <input type="checkbox" checked={todo.done} onChange={() => toggle.mutate(todo)} />
      {todo.text}
    </li>
  );
}
```

This `onMutate` → `onError` rollback pattern is **the** reason TanStack Query mostly displaced raw fetch in production apps.

### Infinite scroll — `useInfiniteQuery`

```tsx
import { useInfiniteQuery } from '@tanstack/react-query';

function Feed() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['feed'],
    queryFn: async ({ pageParam }) => {
      const r = await fetch(`/api/feed?cursor=${pageParam}`);
      return r.json() as Promise<{ items: Post[]; nextCursor: string | null }>;
    },
    initialPageParam: '',
    getNextPageParam: (last) => last.nextCursor ?? undefined,
  });

  return (
    <div>
      {data?.pages.flatMap(p => p.items).map(post => (
        <PostCard key={post.id} post={post} />
      ))}
      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? 'Loading…' : 'Load more'}
        </button>
      )}
    </div>
  );
}
```

Combine with `useIntersectionObserver` (see `02.hooks/05-custom-hooks.md`) for auto-load on scroll.

### Query keys — design them well

The `queryKey` is part of the cache identity. Treat it like a serializable description of the request:

```ts
['user', userId]                                    // ✅ one user
['users', 'list', { status: 'active', page: 2 }]    // ✅ filtered list
['users', 'list', filters]                          // ✅ if `filters` is stable; useMemo it if it's an object
[`/api/users/${userId}`]                            // ⚠ string-only — works but harder to invalidate by prefix
```

`qc.invalidateQueries({ queryKey: ['users'] })` invalidates everything starting with `['users', ...]`. Design the prefix so common invalidations are easy.

---

## 5. SWR 2 — When You Want the Lightest Option

```bash
npm i swr@2.2
```

```tsx
import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(r => r.json());

function User({ id }: { id: string }) {
  const { data, error, isLoading, mutate } = useSWR<User>(`/api/users/${id}`, fetcher, {
    refreshInterval: 0,
    revalidateOnFocus: true,
    dedupingInterval: 2_000,
  });

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorBox onRetry={() => mutate()} />;

  return <h1>{data?.name}</h1>;
}
```

Mutations with `useSWRMutation`:

```tsx
import useSWRMutation from 'swr/mutation';

const updater = (url: string, { arg }: { arg: { name: string } }) =>
  fetch(url, { method: 'PATCH', body: JSON.stringify(arg) }).then(r => r.json());

function EditUser({ id }: { id: string }) {
  const { trigger, isMutating } = useSWRMutation(`/api/users/${id}`, updater);
  return (
    <button onClick={() => trigger({ name: 'New name' })} disabled={isMutating}>
      Save
    </button>
  );
}
```

SWR is good for read-heavy dashboards. For complex mutation flows (optimistic update + rollback + invalidate-by-prefix), TanStack Query is cleaner.

---

## 6. RTK Query — When You're Already in Redux

If you've already chosen `@reduxjs/toolkit@2.2`, RTK Query is the path of least resistance. Defining endpoints:

```ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const api = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({ baseUrl: '/api/' }),
  tagTypes: ['User'],
  endpoints: (b) => ({
    getUser: b.query<User, string>({
      query: (id) => `users/${id}`,
      providesTags: (_r, _e, id) => [{ type: 'User', id }],
    }),
    updateUser: b.mutation<User, { id: string; patch: Partial<User> }>({
      query: ({ id, patch }) => ({ url: `users/${id}`, method: 'PATCH', body: patch }),
      invalidatesTags: (_r, _e, { id }) => [{ type: 'User', id }],
    }),
  }),
});

export const { useGetUserQuery, useUpdateUserMutation } = api;
```

Use it when: your team has decided on Redux for everything. Otherwise, TanStack Query has more momentum and ecosystem support.

---

## 7. Raw `fetch` + `useEffect` — The "When in Doubt" Wrong Answer

The most common React tutorial pattern, and the one to delete first when refactoring:

```tsx
// ❌ Half a dozen bugs in 10 lines
useEffect(() => {
  fetch('/api/me').then(r => r.json()).then(setUser);
}, []);
```

What's missing:
1. No cleanup → component unmounts mid-fetch → setting state on unmounted component (warning + leak).
2. No abort → user navigates away, the in-flight call still completes.
3. No dedupe → two components mount, two requests fire.
4. No cache → re-mounting the same component refetches.
5. No retry → first transient error surfaces to the user.
6. No background revalidation → returning to the tab shows stale data forever.

Use this only for genuinely one-off requests (e.g. firing an analytics event) where the response doesn't drive UI.

---

## 8. Next.js 14 App Router — Different Game

In Next.js 14 with the App Router, **Server Components can `await` fetch directly**:

```tsx
// app/users/[id]/page.tsx (Server Component)
export default async function UserPage({ params }: { params: { id: string } }) {
  const user = await fetch(`https://api/users/${params.id}`, {
    next: { revalidate: 60 }, // ISR: regenerate after 60s
  }).then(r => r.json());

  return <h1>{user.name}</h1>;
}
```

No client cache library needed for the initial render. Use TanStack Query / SWR on the **client component** layer when you need:
- Mutations from the browser
- Real-time updates (websockets, polling)
- Optimistic UI

This server/client split is covered in depth in `07.nextjs/03-data-fetching.md`. The high-level rule: **fetch on the server when possible, hydrate to the client only when interactivity demands it.**

---

## 9. Decision Tree

```
Is this Next.js 14 App Router and the data is read-only?
│   YES → Server Component with `await fetch` + revalidate
│   NO  ↓
Do you need optimistic updates, complex invalidation, or are you mutation-heavy?
│   YES → @tanstack/react-query@5.28
│   NO  ↓
Read-mostly dashboard, you want the smallest bundle?
│   YES → swr@2.2
│   NO  ↓
Already deeply invested in Redux Toolkit?
│   YES → RTK Query
│   NO  → @tanstack/react-query@5.28 (the safe default)
```

---

## 10. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Putting query data into `useState` after `useQuery` returns | Read directly from `data`. Don't sync. |
| Using inline objects in `queryKey` causing constant refetch | `useMemo` the object, or list its primitive fields |
| Setting `staleTime: 0` "to be safe" | Default refetches on every mount; defeats caching. Pick a real value (30s, 5min) per query type |
| Forgetting `enabled: !!id` for queries that depend on a value | Query fires with `id === undefined`, hits a 404 |
| Not invalidating after a mutation | Lists go stale. Use `invalidateQueries` in `onSettled` |
| Doing manual fetch alongside `useQuery` for the same data | Two sources of truth. Drop the manual one. |
| Using `refetchInterval` on every screen | Bandwidth and battery cost. Prefer `refetchOnWindowFocus` + websocket invalidation |

---

## Summary

| Tool | Pick when |
|------|-----------|
| Next.js 14 Server Components `await fetch` | Read-only initial render |
| `@tanstack/react-query@5.28` | Default for client-side data, mutations, real apps |
| `swr@2.2` | Lightweight read-heavy dashboards |
| RTK Query | Already on Redux Toolkit |
| Raw `fetch` in `useEffect` | One-off fire-and-forget calls only |

| Rule | Why |
|------|-----|
| Treat server state as a separate concern | Different rules: caching, dedupe, retry, SWR |
| Design `queryKey`s as `[scope, sub-scope, params]` | Easy prefix invalidation |
| Always include `signal` in queryFn | Abort in-flight requests on unmount |
| Use `onMutate` + `onError` rollback for instant feedback | Optimistic UI without lying to the user |

---

## Further reading

- [TanStack Query v5 docs](https://tanstack.com/query/v5/docs/framework/react/overview)
- [TanStack Query v4 → v5 migration guide](https://tanstack.com/query/v5/docs/framework/react/guides/migrating-to-v5)
- [SWR docs](https://swr.vercel.app/)
- [RTK Query overview](https://redux-toolkit.js.org/rtk-query/overview)
- [Next.js — Data Fetching, Caching, and Revalidating](https://nextjs.org/docs/app/building-your-application/data-fetching/fetching)
