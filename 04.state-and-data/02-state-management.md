# State & Data — 02. State Management: Context vs Zustand 4 vs Redux Toolkit 2

> **What / Why / How** — pick the right tool: don't drag Redux into a 5-page app, don't drag Context into a real-time dashboard.

---

## 1. The First Question to Ask

**Is this state actually client state, or is it server state?**

| Type | Examples | Right tool |
|------|----------|------------|
| **Server state** | Fetched user list, product catalog, inbox messages | `@tanstack/react-query@5.28` (next topic) — never put server data in client state |
| **URL state** | Current route, filters in `?status=open&page=2` | `react-router@6.22` `useSearchParams`, or Next.js 14 `useSearchParams` from `next/navigation` |
| **Form state** | Inputs, validation, submit-in-flight | `react-hook-form@7.51` (covered in `04.state-and-data/01-forms.md`) |
| **Local UI state** | Modal open/closed, hovered, active tab | `useState` in the component that owns it |
| **Global client state** | Auth user, theme, shopping cart, notification queue | This topic — Context, Zustand 4, or Redux Toolkit 2 |

The most common over-engineering: pulling Redux Toolkit into a project where 80% of the "global state" is actually server state that belongs in TanStack Query. Solve the right problem.

---

## 2. The Three Real Options

| | React Context | `zustand@4.5` | `@reduxjs/toolkit@2.2` |
|--|---------------|---------------|------------------------|
| Bundle size (gzipped) | 0 (built-in) | ~1.1 KB | ~13 KB (incl. `react-redux@9`) |
| Boilerplate per "store" | Provider + hook (~25 lines) | `create((set) => ({ … }))` (~10 lines) | slice + reducer + selector (~40 lines) |
| Selector subscriptions (only consumers of changed slices re-render) | ❌ No — every consumer re-renders | ✅ Yes | ✅ Yes |
| DevTools time-travel | ❌ | ✅ Redux DevTools middleware | ✅ Built in |
| Async / side effects | Manual | Plain async functions in actions | Built-in `createAsyncThunk` + RTK Query |
| TypeScript ergonomics | Good | Excellent | Good (improved a lot in v2 vs v1.x) |
| SSR (Next.js 14 App Router) | Trivial | Per-request store factory | Per-request store factory |
| When team is large + audit-friendly history matters | ⚠ | ⚠ | ✅ |
| When you want zero ceremony for a small/medium app | ⚠ (re-render issues) | ✅ | ❌ overkill |
| When state updates are infrequent (theme, locale, current user) | ✅ | ✅ | ⚠ overkill |
| When state updates fire many times per second (real-time, drag, canvas) | ❌ | ✅ | ⚠ |

---

## 3. Approach 1: React Context (built-in)

Use Context when the value changes rarely and is read by many components.

### Real example: theme + auth user

```tsx
// AuthProvider.tsx
import { createContext, useContext, useState, useMemo, ReactNode, useCallback } from 'react';

type AuthCtx = { user: User | null; signIn: (creds: Creds) => Promise<void>; signOut: () => void };
const Ctx = createContext<AuthCtx | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const signIn = useCallback(async (creds: Creds) => {
    const u = await fetch('/api/login', { method: 'POST', body: JSON.stringify(creds) }).then(r => r.json());
    setUser(u);
  }, []);
  const signOut = useCallback(() => setUser(null), []);

  // Stable reference so consumers don't re-render unnecessarily
  const value = useMemo(() => ({ user, signIn, signOut }), [user, signIn, signOut]);

  return <Ctx.Provider value={value}>{children}</Ctx.Provider>;
}

export function useAuth() {
  const v = useContext(Ctx);
  if (!v) throw new Error('useAuth must be inside <AuthProvider>');
  return v;
}
```

### Why Context fails for high-frequency state

Context distributes the **whole value object**. If you store `{ x, y, hovered }` and `x` changes 60 times per second (mouse move), every consumer re-renders 60 times per second — even ones that only read `hovered`.

Real-world failure case: putting a "currently dragged item" coordinate in Context. The drag is fine for two seconds; then the cursor lags because every theme consumer is re-rendering.

### Mitigations (when Context is *almost* enough)

- **Split contexts** by update frequency (e.g. `<UserContext>` separate from `<NotificationContext>`).
- **Memoize the value** with `useMemo` (shown above).
- **Selector via the use-context-selector library** (`use-context-selector@1.4`) — adds selector semantics to plain Context. ~1 KB. Used as a half-step before a real store.

---

## 4. Approach 2: Zustand 4.5 — The Default Choice for Most Apps

### Why Zustand has won "default global store" mindshare in 2026

- ~1 KB gzipped.
- No Provider needed (the store lives outside React).
- Selector subscriptions out of the box.
- Plain async — no thunks, no middleware ceremony.
- TypeScript inference is excellent.
- Works identically with Next.js 14 App Router (with the per-request factory pattern).
- Used in production by `react-flow@11`, `tldraw@2`, `excalidraw@0.x`, `next-themes@0.3`'s sister packages.

### Install + define a store

```bash
npm i zustand@4.5
```

```ts
// stores/cart.ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

type CartItem = { productId: string; qty: number };

type CartState = {
  items: CartItem[];
  total: number;
  add: (productId: string) => void;
  remove: (productId: string) => void;
  clear: () => void;
};

export const useCart = create<CartState>()(
  devtools(
    persist(
      (set, get) => ({
        items: [],
        total: 0,
        add: (productId) => {
          const items = get().items;
          const existing = items.find(i => i.productId === productId);
          const next = existing
            ? items.map(i => i.productId === productId ? { ...i, qty: i.qty + 1 } : i)
            : [...items, { productId, qty: 1 }];
          set({ items: next, total: next.reduce((s, i) => s + i.qty, 0) });
        },
        remove: (productId) => {
          const next = get().items.filter(i => i.productId !== productId);
          set({ items: next, total: next.reduce((s, i) => s + i.qty, 0) });
        },
        clear: () => set({ items: [], total: 0 }),
      }),
      { name: 'cart' } // persist to localStorage under key "cart"
    ),
    { name: 'cart-store' } // shows up in Redux DevTools
  )
);
```

### Consume with selectors — only re-render on what you read

```tsx
function CartBadge() {
  // ✅ Re-renders only when total changes
  const total = useCart(s => s.total);
  return <span>{total}</span>;
}

function AddButton({ id }: { id: string }) {
  // ✅ Action references are stable; no re-render
  const add = useCart(s => s.add);
  return <button onClick={() => add(id)}>+</button>;
}

function CartList() {
  // ✅ shallow-compare the items array
  const items = useCart(s => s.items);
  return <ul>{items.map(i => <li key={i.productId}>{i.productId} × {i.qty}</li>)}</ul>;
}
```

For multi-field selections, use the `shallow` equality function:

```tsx
import { shallow } from 'zustand/shallow';
const { items, total } = useCart(s => ({ items: s.items, total: s.total }), shallow);
```

### Async actions — just async functions

```ts
checkout: async () => {
  set({ checkingOut: true });
  try {
    await api.checkout(get().items);
    set({ items: [], total: 0, checkingOut: false });
  } catch (err) {
    set({ checkingOut: false, error: (err as Error).message });
  }
},
```

No thunks, no `dispatch`, no `payloadAction` types — just JavaScript.

### Next.js 14 App Router gotcha — per-request store factory

For Server Components / SSR, you cannot share a module-level store across requests (it leaks state between users). The pattern:

```tsx
// stores/cart-factory.ts
import { createStore } from 'zustand/vanilla';
export const createCartStore = () => createStore<CartState>()((set) => ({ … }));
```

Then wrap the app in a Provider that creates a fresh store per request. The Zustand docs document this exact pattern under [Setup with Next.js](https://zustand.docs.pmnd.rs/guides/nextjs).

---

## 5. Approach 3: Redux Toolkit 2 — When You Actually Need It

### The honest pitch

Redux Toolkit 2 (RTK) is **not the same product as classic Redux** from 2017. Modern RTK:

- Eliminates boilerplate via `createSlice`.
- Has built-in Immer (mutate-style reducers compile to immutable updates).
- Bundles `RTK Query` for server state (a real alternative to TanStack Query).
- TypeScript-native in v2.

Pick RTK over Zustand when:
- Your team has 10+ engineers and you want enforced action-history audit trails.
- You're using `RTK Query` for server data (it's competitive with TanStack Query and shares stylistic conventions with the rest of your reducers).
- You're already in a Redux codebase and don't want to fight it.

Don't pick RTK because "Redux is the standard" — that hasn't been true since 2021. Zustand and TanStack Query cover most of what teams used Redux for.

### Install

```bash
npm i @reduxjs/toolkit@2.2 react-redux@9
```

### Real example — counter slice + async thunk

```ts
// features/counter/counterSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';

type CounterState = { value: number; loading: boolean };

export const incrementAsync = createAsyncThunk(
  'counter/incrementAsync',
  async (amount: number) => {
    await new Promise(r => setTimeout(r, 500));
    return amount;
  }
);

const slice = createSlice({
  name: 'counter',
  initialState: { value: 0, loading: false } satisfies CounterState,
  reducers: {
    increment: (s) => { s.value += 1; },                   // Immer makes this safe
    incrementBy: (s, a: PayloadAction<number>) => { s.value += a.payload; },
  },
  extraReducers: (b) => {
    b.addCase(incrementAsync.pending, (s) => { s.loading = true; });
    b.addCase(incrementAsync.fulfilled, (s, a) => { s.value += a.payload; s.loading = false; });
  },
});

export const { increment, incrementBy } = slice.actions;
export default slice.reducer;
```

### Wire up the store

```ts
// app/store.ts
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from '../features/counter/counterSlice';
export const store = configureStore({ reducer: { counter: counterReducer } });
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### Typed hooks (idiomatic in 2026)

```ts
// app/hooks.ts
import { useDispatch, useSelector, type TypedUseSelectorHook } from 'react-redux';
import type { RootState, AppDispatch } from './store';
export const useAppDispatch: () => AppDispatch = useDispatch;
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

### Consume

```tsx
import { useAppDispatch, useAppSelector } from '../app/hooks';
import { increment, incrementAsync } from './counterSlice';

function Counter() {
  const value = useAppSelector(s => s.counter.value);
  const loading = useAppSelector(s => s.counter.loading);
  const dispatch = useAppDispatch();
  return (
    <>
      <span>{value}</span>
      <button onClick={() => dispatch(increment())}>+</button>
      <button onClick={() => dispatch(incrementAsync(5))} disabled={loading}>+5 async</button>
    </>
  );
}
```

### RTK Query — if you go the Redux route, use this for fetching

```ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const api = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({ baseUrl: '/api/' }),
  tagTypes: ['Post'],
  endpoints: (b) => ({
    getPosts: b.query<Post[], void>({ query: () => 'posts', providesTags: ['Post'] }),
    addPost: b.mutation<Post, Partial<Post>>({
      query: (body) => ({ url: 'posts', method: 'POST', body }),
      invalidatesTags: ['Post'],
    }),
  }),
});

export const { useGetPostsQuery, useAddPostMutation } = api;
```

If you're not committed to RTK, prefer `@tanstack/react-query@5.28` (next topic) — it's framework-agnostic and slightly more popular in the ecosystem.

---

## 6. Honorable Mention: Jotai 2 — Atomic State

`jotai@2` ships with Pmndrs (same authors as Zustand). Different mental model: state is composed from "atoms" — minimal reactive primitives.

```bash
npm i jotai@2
```

```ts
import { atom, useAtom } from 'jotai';

const countAtom = atom(0);
const doubledAtom = atom((get) => get(countAtom) * 2); // derived

function Counter() {
  const [count, setCount] = useAtom(countAtom);
  const [doubled] = useAtom(doubledAtom);
  return <button onClick={() => setCount(c => c + 1)}>{count} ({doubled})</button>;
}
```

When Jotai shines: deeply reactive UIs (form builders, design tools, spreadsheets) where dozens of small derived values flow from a few base values. Used by `wundergraph` and several internal tools at Vercel.

When to skip: if Zustand handles your case with a single store, you don't need atoms.

---

## 7. Decision Tree

```
Is the data on a server (fetched, then cached/synced)?
│   YES → @tanstack/react-query@5.28 or RTK Query (next topic)
│   NO  ↓
Is it just one component's UI state?
│   YES → useState in that component
│   NO  ↓
Is it the route, filters, or search params?
│   YES → react-router@6 useSearchParams / Next.js useSearchParams
│   NO  ↓
Is it form state?
│   YES → react-hook-form@7.51
│   NO  ↓
How frequent are updates? Many per second? (drag, real-time, canvas)
│   YES → zustand@4.5 (selectors prevent re-renders)
│   NO  ↓
Is the app big enough that audit history / time-travel really matters?
│   YES → @reduxjs/toolkit@2.2
│   NO  → React Context with split contexts + memoized values
```

---

## 8. Migration Notes (Real Scenarios)

### From Context spaghetti → Zustand

If you have 5+ Provider wrappers and consumers re-render too much: collapse the Providers into a single Zustand store, keep the same `useFoo()` hook names so callers don't change.

### From classic Redux → Redux Toolkit 2

`createSlice` replaces `actionCreators + reducers + types`. RTK ships a [migration guide](https://redux-toolkit.js.org/usage/migrating-rtk-2). Don't migrate to "modern Redux"; jump straight to RTK.

### From Redux → Zustand or TanStack Query

Most "Redux for fetched data" stores can be deleted in favor of TanStack Query. The remaining client state usually fits in one Zustand store. This is the most common 2024–2026 simplification.

---

## Summary

| Tool | Pick when |
|------|-----------|
| `useState` | One component owns it |
| URL state (router) | It's bookmarkable / shareable |
| `react-hook-form@7.51` | It's form data |
| `@tanstack/react-query@5.28` | It came from a server |
| React Context (split + memoized) | It rarely changes (theme, locale, auth user) |
| `zustand@4.5` | Frequent updates or you want zero ceremony |
| `@reduxjs/toolkit@2.2` | Big team, audit-friendly history, or already in a Redux codebase |
| `jotai@2` | Many small interdependent derived values |

| Rule | Why |
|------|-----|
| Solve server state with TanStack Query first | Most "global state" pain is actually fetched-data pain |
| Split contexts by update frequency | Avoid full re-render cascades |
| Use Zustand selectors with `shallow` for multi-field reads | Otherwise every consumer re-renders on any field change |
| In Next.js 14, use a per-request store factory | Module-level stores leak state across users on the server |

---

## Further reading

- [Zustand docs — Next.js setup](https://zustand.docs.pmnd.rs/guides/nextjs)
- [Redux Toolkit docs](https://redux-toolkit.js.org/)
- [Jotai docs](https://jotai.org/)
- [TanStack Query — why server state is different](https://tanstack.com/query/latest/docs/framework/react/overview)
- [Mark Erikson — When (and when not) to reach for Redux](https://blog.isquaredsoftware.com/2021/01/context-redux-differences/)
