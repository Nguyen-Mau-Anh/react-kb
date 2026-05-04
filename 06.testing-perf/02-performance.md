# Testing & Performance — 02. Performance: React.memo, lazy/Suspense, Profiler, Bundle Analysis

> **What / Why / How** — measure first, optimize second. Three real fixes solve 90% of slow React apps.

---

## 1. The Three Buckets of "Slow React"

| Bucket | Symptom | First-pass fix |
|--------|---------|----------------|
| **Render time too long** (CPU bound) | Typing lags, scroll jitter, devtools shows long flame chart entries | Memoize hot paths, virtualize lists, split state |
| **Bundle too big** (network bound) | Slow first paint, big "Time to Interactive" | Code-split with `lazy()` + Suspense, drop heavy deps |
| **Server-state thrashing** (network bound) | Spinner cascade, refetch storms | Move to `@tanstack/react-query@5.28`, configure `staleTime` |

Profile before optimizing. Skip ahead to **§7 — Tooling** if you don't yet know which bucket you're in.

---

## 2. `React.memo` — Skip Re-renders With Equal Props

### What

`React.memo(Component)` returns a wrapped component that bails out of rendering when its props are shallow-equal to the previous render.

```tsx
import { memo } from 'react';

type RowProps = { user: { id: string; name: string }; onSelect: (id: string) => void };

export const Row = memo(function Row({ user, onSelect }: RowProps) {
  return <li onClick={() => onSelect(user.id)}>{user.name}</li>;
});
```

### Why memo helps

A parent re-render normally re-renders every child, even if the child's props haven't changed. For a list of 500 rows, this means 500 component executions on every parent state change — even ones unrelated to the list.

### When memo is wasted

- The child is cheap (renders one `<div>`).
- Props are inline objects/arrays/functions that change every parent render anyway. The shallow comparison always falls through.

```tsx
// ❌ memo never bails out — `user` is a new object literal every render
<Row user={{ id, name }} onSelect={handleSelect} />

// ✅ Pass primitives, or memoize the object/handler at the parent
<Row userId={id} userName={name} onSelect={onSelect} />
```

### Custom comparator

```tsx
const Row = memo(RowImpl, (prev, next) => prev.user.id === next.user.id);
```

Use sparingly — once you're hand-writing `arePropsEqual`, you're past the point where the React Compiler (covered in `02.hooks/04-usememo-usecallback.md`) would do better.

### memo + `useCallback` for stable handlers

Without `useCallback`, the parent re-creates `onSelect` every render → memo's shallow check fails → all children re-render.

```tsx
function List({ users }: { users: User[] }) {
  // ✅ stable across renders → memo'd Row actually skips
  const onSelect = useCallback((id: string) => console.log(id), []);
  return users.map(u => <Row key={u.id} user={u} onSelect={onSelect} />);
}
```

---

## 3. Code Splitting — `React.lazy` + `<Suspense>`

### What

`React.lazy(() => import('./X'))` returns a component whose code is fetched on first render. Wrap it in `<Suspense fallback={...}>` for the loading state.

```tsx
import { lazy, Suspense } from 'react';

const AdminDashboard = lazy(() => import('./AdminDashboard'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <AdminDashboard />
    </Suspense>
  );
}
```

### Why this matters

A bundle that loads admin code for non-admin users is wasted bytes. With `lazy`, Vite 5 / Next.js 14 emits a separate chunk; the chunk only loads when `<AdminDashboard />` is actually rendered.

### Real wins (numbers from typical refactors)

| Before | After splitting | Saved |
|--------|-----------------|-------|
| 950 KB initial gzipped bundle | 320 KB initial + 4×~150 KB lazy chunks | First paint ~3× faster on slow 3G |
| Charts (Recharts) loaded on every page | Loaded only on `/dashboard` | ~80 KB off the entry chunk |
| Rich-text editor (TipTap) loaded everywhere | Loaded only when user opens "Edit" | ~120 KB off entry |

### Real candidates for `lazy`

- **Routes** — every non-home route in a SPA.
- **Heavy modals / drawers** — that open occasionally (chart-builder, file-upload UI).
- **Admin-only or feature-flagged sections.**
- **Third-party-heavy components** — Mapbox GL JS 3, Three.js r161, `tiptap@2`, `recharts@2.12`, `@codemirror/state@6`.

### Where `lazy` doesn't help

- The home/landing page itself (it loads on every visit anyway).
- A component used immediately on every route (the chunk fetch becomes a serial round trip after the entry).

### React Router 6 + lazy

```tsx
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

Covered in `05.routing-and-styling/01-routing-react-router.md`. Same pattern in Next.js 14: `dynamic(() => import('./X'), { ssr: false, loading: () => <Spinner /> })` from `next/dynamic`.

---

## 4. Suspense for Data — Streaming UI with Promises

`<Suspense>` is no longer just for `lazy`. With React 18+ streaming SSR (Next.js 14 App Router uses this for free):

```tsx
// app/dashboard/page.tsx (Next.js 14 Server Component)
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <>
      <Header />                       {/* streams immediately */}
      <Suspense fallback={<Skeleton />}>
        <SlowAnalytics />              {/* streams when its fetch resolves */}
      </Suspense>
    </>
  );
}

async function SlowAnalytics() {
  const data = await fetch('https://api/analytics', { next: { revalidate: 60 } }).then(r => r.json());
  return <Chart data={data} />;
}
```

Real impact: the user sees the header and shell at ~200ms instead of waiting for analytics (~1500ms). Only the slow piece swaps from skeleton to content when ready.

For client-side Suspense with React Query, the `suspense: true` flag on a query throws to the nearest boundary. Used by Linear's web app and several Vercel template apps.

---

## 5. Lists — Virtualization

A 5,000-row table that renders all rows in DOM is slow regardless of memoization. Virtualize.

| Library | Bundle gzipped | Best for |
|---------|----------------|----------|
| `@tanstack/react-virtual@3` | ~5 KB | Headless — you build the markup. The 2026 default. |
| `react-virtuoso@4` | ~25 KB | Auto-sized rows, chat/feed UIs with dynamic heights |
| `react-window@1.8` | ~3 KB | Simple fixed-height lists; older API |

Real example with `@tanstack/react-virtual@3` is shown in `03.rendering-patterns/02-conditional-lists.md`. The win: only ~15 DOM nodes exist for a 100K-item list.

---

## 6. State Splits — Move State Down or Sideways

Often the cheapest fix is structural, not memo-based.

### Move state down to the component that needs it

```tsx
// ❌ App holds state that only ColorPicker uses → App re-renders the whole tree
function App() {
  const [color, setColor] = useState('#fff');
  return (
    <>
      <Sidebar />
      <Main />
      <ColorPicker color={color} onChange={setColor} />
    </>
  );
}

// ✅ State lives inside ColorPicker — Sidebar and Main don't re-render on color change
function App() {
  return (
    <>
      <Sidebar />
      <Main />
      <ColorPicker />
    </>
  );
}
function ColorPicker() {
  const [color, setColor] = useState('#fff');
  return <input value={color} onChange={e => setColor(e.target.value)} />;
}
```

### Move state sideways (selector-based stores)

If state must be shared but only some consumers care: `zustand@4.5` selectors only re-render the components that read the changed slice. Covered in `04.state-and-data/02-state-management.md`.

### `useTransition` — mark non-urgent updates

For an expensive re-render that shouldn't block input:

```tsx
import { useTransition, useState } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  function onChange(e: React.ChangeEvent<HTMLInputElement>) {
    setQuery(e.target.value); // urgent

    startTransition(() => {
      setResults(filter(allItems, e.target.value)); // can be interrupted
    });
  }

  return (
    <>
      <input value={query} onChange={onChange} />
      <List items={results} dim={isPending} />
    </>
  );
}
```

The input stays responsive while the heavy list re-renders are interruptible. Covered in `01.fundamentals/01-how-react-works.md`.

---

## 7. Tooling — Measure Before You Optimize

### React DevTools Profiler

Browser extension by Meta. Open DevTools → "Profiler" tab → record an interaction → read the flame chart.

What to look for:
- Long bars (high "self time") on hot paths.
- Components rendering despite identical props (memo candidates).
- "Why did this render?" panel — turn it on in DevTools settings.

### `<Profiler>` API for production telemetry

```tsx
import { Profiler } from 'react';

function onRender(id: string, phase: 'mount' | 'update' | 'nested-update', actualDuration: number) {
  if (actualDuration > 16) {
    sendToSentry({ id, phase, actualDuration });
  }
}

<Profiler id="ProductGrid" onRender={onRender}>
  <ProductGrid />
</Profiler>
```

Real-world use: log slow renders to Sentry / DataDog so you spot regressions in production, not just local dev.

### Bundle analysis — `rollup-plugin-visualizer`

```bash
npm i -D rollup-plugin-visualizer@5.12
```

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({
      filename: 'dist/stats.html',
      template: 'treemap',  // or 'sunburst', 'network'
      gzipSize: true,
      brotliSize: true,
    }),
  ],
});
```

Run `vite build` → open `dist/stats.html`. Treemap shows every chunk and every package's contribution. The single best tool for "what's in my bundle".

What to look for:
- A library you barely use eating 100+ KB (typical: full `lodash@4` instead of `lodash-es` named imports; `moment@2` instead of `date-fns@3`).
- Polyfills you don't need (e.g. `core-js` for browsers your `browserslist` no longer targets).
- Duplicate copies of a library (e.g. `react@18` and `react@17` both in the graph from a transitive dep).

### Bundle analysis — Next.js 14

```bash
npm i -D @next/bundle-analyzer@14
```

```js
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});
module.exports = withBundleAnalyzer({ /* ... */ });
```

Run `ANALYZE=true npm run build` to open the visualizer.

### Lighthouse / PageSpeed Insights

Run on a built+deployed app, not dev mode. Targets:

| Core Web Vital | Good | Source of regressions |
|----------------|------|----------------------|
| **LCP** (Largest Contentful Paint) | < 2.5s | Big hero image, no SSR, slow API |
| **INP** (Interaction to Next Paint, replaced FID in 2024) | < 200ms | Long render after click, big main-thread JS |
| **CLS** (Cumulative Layout Shift) | < 0.1 | Images without dimensions, fonts without fallback |

Lighthouse-CI in GitHub Actions enforces budgets per PR.

---

## 8. Real Quick Wins (Most Apps Have These)

| Win | How |
|-----|-----|
| Replace `lodash@4` defaults with named imports | `import debounce from 'lodash/debounce'` (or use `lodash-es@4`) — saves ~70 KB |
| Replace `moment@2` with `date-fns@3` or `dayjs@1.11` | `moment` is ~70 KB; `date-fns` is ~10 KB tree-shaken |
| Drop `axios@1` for native `fetch` | Saves ~13 KB and removes a maintenance dep |
| Convert client components to Server Components (Next.js 14) | Less JS shipped to the browser entirely |
| Add `loading="lazy"` to below-the-fold `<img>` | Native browser feature, zero cost |
| Use `next/image` or modern `<picture>` with WebP/AVIF | Image bytes are usually >50% of an app's transfer |
| Set `staleTime` in TanStack Query (default: 0) | Stops refetch storms on every mount |
| Memoize hot-path objects in deps arrays | Stops dependent effects from running every render |

---

## 9. The React Compiler — The Future of Memoization

`react-compiler` (formerly "React Forget") auto-memoizes function components and hook outputs at build time. Stable preview shipped May 2024; production-ready in React 19 + Next.js 15. With it enabled, **most manual `useMemo` / `useCallback` calls become unnecessary**.

```js
// next.config.js (Next.js 15)
module.exports = {
  experimental: { reactCompiler: true },
};
```

In a 2026 React 19 + compiler codebase, your performance work shifts:
- ✅ Code splitting still matters (compiler doesn't shrink bundles).
- ✅ List virtualization still matters (compiler doesn't change DOM size).
- ✅ Server Components still matter (compiler doesn't move work to the server).
- ❌ Hand-written `useMemo`/`useCallback` mostly stop mattering.

Covered in detail in `02.hooks/04-usememo-usecallback.md`.

---

## 10. Decision Tree

```
Is the slowness on first load (white screen, slow LCP)?
│   YES → Bundle issue. Run rollup-plugin-visualizer or @next/bundle-analyzer.
│         Code-split routes with React.lazy / next/dynamic.
│         Drop heavy deps (lodash@4 → lodash-es, moment → date-fns).
│   NO  ↓
Is it slow during interaction (typing lags, scroll stutter)?
│   YES → Render issue. Open React Profiler.
│         If list >500 items: virtualize with @tanstack/react-virtual@3.
│         If memo candidates exist: React.memo + useCallback.
│         If non-urgent re-renders block input: useTransition.
│         If state updates spread too widely: split state down or move to zustand@4.5.
│   NO  ↓
Is it slow waiting for data (spinner soup)?
│   YES → Server-state issue. Move to @tanstack/react-query@5.28.
│         Set realistic staleTime (30s+ for most queries).
│         Use Suspense + streaming SSR for slow Server Components.
│   NO  ↓
Are Core Web Vitals failing on PageSpeed?
│   YES → LCP: SSR + image optimization (next/image).
│         CLS: width/height on images, fallback fonts.
│         INP: useTransition on heavy click handlers, code-split heavy modals.
```

---

## 11. Pitfalls

| Pitfall | Fix |
|---------|-----|
| Wrapping every component in `React.memo` | Bookkeeping cost > savings; profile first |
| `useMemo` for primitives | Cheaper to compute inline |
| `useCallback` without a memo'd consumer | Useless — the handler ref doesn't matter if nothing memoizes |
| Code-splitting the home page | The chunk loads instantly anyway; serial round trip slows TTI |
| Suspense fallback that doesn't match content size | Layout shift on fill — match dimensions |
| TanStack Query `staleTime: 0` everywhere | Refetch storm; pick a real value per query type |
| Replacing `lodash` import with whole `lodash-es` | Same problem; use named imports or just write `Object.keys(...)` |

---

## Summary

| Tool | Use for |
|------|---------|
| `React.memo` + `useCallback` | Heavy children with stable props |
| `React.lazy` + `<Suspense>` | Per-route, per-heavy-modal code splitting |
| `<Suspense>` + streaming SSR | Don't let the slow piece block the page |
| `@tanstack/react-virtual@3` | Lists with 500+ rows |
| `useTransition` | Non-urgent updates (search filters, large list refresh) |
| `zustand@4.5` selectors | Sharing state without re-rendering everyone |
| `rollup-plugin-visualizer@5.12` | Vite bundle inspection |
| `@next/bundle-analyzer@14` | Next.js bundle inspection |
| React DevTools Profiler | Render-time CPU profiling |

| Rule | Why |
|------|-----|
| Profile before optimizing | Most "perf" intuitions are wrong |
| Lift state DOWN, not up | Smallest possible re-render scope |
| Code-split routes and heavy modals | Cheapest single win for first-load |
| Don't memoize cheap components | The cost of bookkeeping isn't free |
| Plan for the React Compiler | Manual memo will mostly be unnecessary in React 19+ |

---

## Further reading

- [React docs — Profiler](https://react.dev/reference/react/Profiler)
- [React docs — useTransition](https://react.dev/reference/react/useTransition)
- [TanStack Virtual docs](https://tanstack.com/virtual/latest)
- [rollup-plugin-visualizer](https://github.com/btd/rollup-plugin-visualizer)
- [Next.js — bundle analyzer](https://nextjs.org/docs/app/api-reference/config/next-config-js/bundleAnalyzer)
- [web.dev — Core Web Vitals (INP overview)](https://web.dev/articles/inp)
