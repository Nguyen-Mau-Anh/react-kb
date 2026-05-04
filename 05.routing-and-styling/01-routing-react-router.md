# Routing & Styling — 01. Routing with React Router 6: Data Routers, Loaders, Actions

> **What / Why / How** — `react-router-dom@6.22` is the de-facto router for Vite/CRA-style React SPAs. Skip it if you're on Next.js (Next has its own router).

---

## 1. When to Use React Router (and When Not To)

| Stack | Router |
|-------|--------|
| Vite 5 + React 18 SPA | `react-router-dom@6.22` ✅ |
| Create React App (legacy) | `react-router-dom@6.22` ✅ |
| Next.js 14 App Router | Built-in `next/navigation` — **don't add React Router** |
| Next.js 14 Pages Router | Built-in `next/router` — **don't add React Router** |
| Remix 2 | Built-in (Remix is a fork of React Router internals) |
| TanStack Start (preview) | `@tanstack/router@1` — different library, type-safe routes |

This topic covers `react-router-dom@6.22`. The next topic on Next.js routing (`07.nextjs/02-routing.md`) covers Next's own file-based router.

---

## 2. The v6 Mental Model — Two API Styles

`react-router@6` ships **two** ways to declare routes:

### Style A — JSX routes (the original v6 API)

```tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route index element={<Home />} />
          <Route path="users" element={<Users />}>
            <Route path=":id" element={<UserDetail />} />
          </Route>
          <Route path="*" element={<NotFound />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
```

Works fine. **No data loading APIs** — you're stuck fetching in `useEffect` everywhere.

### Style B — Data router (v6.4+, the modern API)

```tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/',
    element: <Layout />,
    errorElement: <RootError />,
    children: [
      { index: true, element: <Home /> },
      {
        path: 'users',
        element: <UsersLayout />,
        loader: usersLoader,            // run before render
        children: [
          {
            path: ':id',
            element: <UserDetail />,
            loader: userLoader,
            action: userAction,         // handles form submissions
          },
        ],
      },
    ],
  },
]);

function App() {
  return <RouterProvider router={router} />;
}
```

This unlocks **`loader`**, **`action`**, **`useNavigation`**, **deferred data**, and the rest of the Remix-inspired API. **Use this style for all new code in 2026.** The JSX `<Routes>` form still exists but is essentially feature-frozen.

---

## 3. Loaders — Fetch Before Render, Not With useEffect

### Why loaders beat `useEffect`

The `useEffect + fetch` pattern (covered as an anti-pattern in `02.hooks/02-useeffect.md` and `04.state-and-data/03-data-fetching.md`):

1. Renders an empty layout.
2. Fires `useEffect`.
3. Shows a spinner.
4. Receives data, re-renders.

The data router with a loader:

1. Triggers navigation.
2. Runs the loader **before** rendering the new route.
3. Renders the route already with data.

The user sees one less flash. More importantly, the loader runs in parallel with sibling loaders — nested data fetches no longer waterfall.

### Real example — user detail with parent + child loaders

```tsx
// loaders/users.ts
import { LoaderFunctionArgs } from 'react-router-dom';

export async function usersLoader() {
  const r = await fetch('/api/users');
  if (!r.ok) throw new Response('Failed to load users', { status: r.status });
  return r.json() as Promise<User[]>;
}

export async function userLoader({ params }: LoaderFunctionArgs) {
  const r = await fetch(`/api/users/${params.id}`);
  if (!r.ok) throw new Response('User not found', { status: r.status });
  return r.json() as Promise<User>;
}
```

```tsx
// Routes — the parent loader and child loader run in parallel
{
  path: 'users',
  element: <UsersLayout />,
  loader: usersLoader,
  children: [{ path: ':id', element: <UserDetail />, loader: userLoader }],
}
```

```tsx
// UsersLayout reads the parent loader's data
import { Outlet, useLoaderData } from 'react-router-dom';

function UsersLayout() {
  const users = useLoaderData() as User[];
  return (
    <div className="grid grid-cols-[200px_1fr]">
      <UserList users={users} />
      <Outlet /> {/* nested route renders here */}
    </div>
  );
}

function UserDetail() {
  const user = useLoaderData() as User;
  return <h1>{user.name}</h1>;
}
```

### Throwing a `Response` for typed errors

`throw new Response('msg', { status: 404 })` is caught by the nearest `errorElement`. Combined with `useRouteError()`, you get framework-level error boundaries without writing class components:

```tsx
import { useRouteError, isRouteErrorResponse } from 'react-router-dom';

function RootError() {
  const err = useRouteError();
  if (isRouteErrorResponse(err)) {
    return <h1>{err.status} — {err.statusText}</h1>;
  }
  return <h1>Unexpected error</h1>;
}
```

---

## 4. Actions — Forms That Hit the Server Without `e.preventDefault`

### What an `action` does

An action handles `<Form>` submissions, runs your server-side mutation, and triggers automatic revalidation of all loaders for that route.

```tsx
// actions/user.ts
import { ActionFunctionArgs, redirect } from 'react-router-dom';

export async function userAction({ request, params }: ActionFunctionArgs) {
  const formData = await request.formData();
  const update = Object.fromEntries(formData);

  const r = await fetch(`/api/users/${params.id}`, {
    method: 'PATCH',
    body: JSON.stringify(update),
    headers: { 'Content-Type': 'application/json' },
  });

  if (!r.ok) {
    return { error: 'Update failed' }; // available via useActionData()
  }
  return redirect(`/users/${params.id}`);
}
```

```tsx
// UserEdit.tsx — note `Form` (capital F) from react-router-dom
import { Form, useNavigation, useActionData } from 'react-router-dom';

function UserEdit() {
  const navigation = useNavigation();
  const actionData = useActionData() as { error?: string } | undefined;
  const submitting = navigation.state === 'submitting';

  return (
    <Form method="post">
      <input name="name" defaultValue="" />
      <input name="email" defaultValue="" />
      {actionData?.error && <p className="text-red-600">{actionData.error}</p>}
      <button disabled={submitting}>{submitting ? 'Saving…' : 'Save'}</button>
    </Form>
  );
}
```

What you get for free:
- `e.preventDefault()` is unnecessary; the router intercepts the submit.
- After the action returns, the route's loader runs again automatically — UI is consistent without manual cache invalidation.
- `useNavigation().state` (`'idle' | 'loading' | 'submitting'`) drives pending UI.
- Progressive enhancement: the form works even before JS hydrates (in SSR setups like Remix).

---

## 5. Programmatic Navigation — `useNavigate`

```tsx
import { useNavigate } from 'react-router-dom';

function LoginForm() {
  const navigate = useNavigate();

  async function handleSubmit() {
    await api.login();
    navigate('/dashboard', { replace: true }); // replace history entry instead of pushing
  }
}
```

Less common but useful: `navigate(-1)` for "back".

---

## 6. URL State — Forget `useState` for Filters

`useSearchParams` makes the URL the source of truth. Bookmarkable, shareable, server-renderable.

```tsx
import { useSearchParams } from 'react-router-dom';

function ProductList() {
  const [params, setParams] = useSearchParams();
  const status = params.get('status') ?? 'all';
  const page = Number(params.get('page') ?? 1);

  function setStatus(next: string) {
    setParams({ status: next, page: '1' });
  }

  // ...
}
```

Real-world rule (covered briefly in `04.state-and-data/02-state-management.md`): **anything that's bookmarkable belongs in the URL**, not `useState`. Filters, current tab, sort order, pagination — all URL state.

---

## 7. Active Links — `NavLink` with Tailwind

```tsx
import { NavLink } from 'react-router-dom';

<NavLink
  to="/users"
  className={({ isActive }) =>
    isActive ? 'font-bold text-blue-600' : 'text-gray-700 hover:text-gray-900'
  }
>
  Users
</NavLink>
```

`NavLink` exposes `isActive`, `isPending` (data router only — true while a loader for that link is running), and `isTransitioning` (View Transitions API).

---

## 8. Deferred Data — `defer` for Slow Pieces

Sometimes one slow query shouldn't block the whole page. `defer` lets the route render with placeholders, streaming the slow piece in.

```tsx
import { defer, Await, useLoaderData } from 'react-router-dom';
import { Suspense } from 'react';

export async function dashboardLoader() {
  return defer({
    user: fetch('/api/me').then(r => r.json()),                      // fast
    revenue: fetch('/api/revenue/last-90-days').then(r => r.json()), // slow → don't await
  });
}

function Dashboard() {
  const data = useLoaderData() as { user: User; revenue: Promise<Revenue> };
  return (
    <>
      <h1>Hi, {data.user.name}</h1>
      <Suspense fallback={<RevenueSkeleton />}>
        <Await resolve={data.revenue}>
          {(rev) => <RevenueChart data={rev} />}
        </Await>
      </Suspense>
    </>
  );
}
```

Note `data.user` was awaited inside the loader (we want it for the heading); only `revenue` was deferred.

---

## 9. Route Protection — Auth Guards

There's no built-in `<RequireAuth>` component, but the pattern is well established:

```tsx
import { redirect } from 'react-router-dom';

async function requireAuth() {
  const r = await fetch('/api/me');
  if (!r.ok) throw redirect('/login');
  return r.json();
}

// Use as a loader (or call inside a loader)
{ path: 'admin', element: <Admin />, loader: requireAuth }
```

`throw redirect(...)` short-circuits rendering and redirects. This is the same pattern Remix uses.

---

## 10. TanStack Query + React Router — Best of Both

A real combo used in production:

- **TanStack Query** (`04.state-and-data/03-data-fetching.md`) handles client-side caching, background refetch, optimistic updates.
- **React Router loaders** seed the cache so the route can render immediately.

```tsx
import { QueryClient } from '@tanstack/react-query';

const qc = new QueryClient();

export const userLoader = (qc: QueryClient) => async ({ params }: LoaderFunctionArgs) => {
  return qc.ensureQueryData({
    queryKey: ['user', params.id],
    queryFn: () => fetch(`/api/users/${params.id}`).then(r => r.json()),
  });
};
```

```tsx
// Route declaration
{ path: 'users/:id', element: <UserDetail />, loader: userLoader(qc) }

// Component reads from React Query — same cache instance, instant data
function UserDetail() {
  const { id } = useParams();
  const { data } = useQuery({ queryKey: ['user', id], queryFn: () => fetch(...).then(r => r.json()) });
  return <h1>{data?.name}</h1>;
}
```

The TanStack Query docs document this exact pattern.

---

## 11. Lazy-Loaded Routes — Code Splitting

```tsx
import { createBrowserRouter } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/admin',
    lazy: async () => {
      const { Admin, adminLoader } = await import('./routes/Admin');
      return { Component: Admin, loader: adminLoader };
    },
  },
]);
```

Vite + Rollup automatically split this into a separate chunk; the chunk loads only when the user navigates to `/admin`. Used by every non-trivial dashboard.

---

## 12. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Mixing `<Routes>` JSX style with `createBrowserRouter` | Pick one. Use the data router. |
| Forgetting `<Outlet />` in a layout component | Children never render |
| Using `<a href>` for internal links | Forces full reload. Use `<Link to>` or `<NavLink to>` |
| Calling `navigate()` inside render | Causes infinite redirect loops. Call inside an event handler or loader |
| Shadowing search params with local state | Two sources of truth. Read from `useSearchParams` |
| Keeping form state in `useState` and submitting via fetch | Use `<Form method="post">` + `action` instead — get pending UI for free |

---

## Summary

| Feature | API |
|---------|-----|
| Define routes | `createBrowserRouter([{ path, element, loader, action, errorElement, children }])` |
| Top-level mount | `<RouterProvider router={router} />` |
| Read loader data | `useLoaderData()` |
| Submit a form | `<Form method="post">` + route `action` |
| Get pending state | `useNavigation().state` |
| URL state | `useSearchParams()` |
| Programmatic navigate | `useNavigate()` |
| Active link styling | `<NavLink className={({ isActive }) => ...}>` |
| Stream slow data | `defer({ slow: promise })` + `<Await>` + `<Suspense>` |
| Code-split a route | `lazy: () => import(...)` |

| Rule | Why |
|------|-----|
| Use the data router (`createBrowserRouter`) for new code | Loaders/actions remove `useEffect + fetch` waterfalls |
| Don't add React Router to a Next.js app | Next has its own router |
| Put filters/pagination/sort in `useSearchParams` | Bookmarkable, shareable, no state sync bugs |
| Combine with TanStack Query for full client cache | Loader seeds cache; component reads from React Query |

---

## Further reading

- [React Router docs — picking a router](https://reactrouter.com/en/main/routers/picking-a-router)
- [React Router — Tutorial (uses data APIs)](https://reactrouter.com/en/main/start/tutorial)
- [TanStack Query + React Router pattern](https://tanstack.com/query/v5/docs/framework/react/guides/router-integration)
- [Remix blog — When to fetch](https://remix.run/blog/when-to-fetch) — same mental model as the data router
