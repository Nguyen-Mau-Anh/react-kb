# Topic 04 — Side Effects with useEffect: Deps Array, Cleanup, Pitfalls

> **What / Why / How** — `useEffect` is the most misused hook in React. Learn when to NOT use it.

---

## 1. What is `useEffect`?

### What

`useEffect` runs a function **after** React commits changes to the DOM. It's how you "step outside of React" — to talk to a non-React system: the browser API, a websocket, an external store, a third-party library.

```tsx
import { useEffect, useState } from 'react';

function PageTitle({ title }: { title: string }) {
  useEffect(() => {
    document.title = title;            // side effect — touching the DOM directly
    return () => { document.title = 'My App'; }; // cleanup on unmount
  }, [title]);                          // re-run when `title` changes
  return null;
}
```

### When effect runs

```
Render phase (pure)         → React calls your component, builds VDOM
   ↓
Commit phase                → React mutates the real DOM
   ↓
useLayoutEffect             → runs synchronously, BEFORE browser paint (DOM measurement)
   ↓
Browser paints
   ↓
useEffect                   → runs asynchronously, AFTER paint (network, subscriptions)
```

---

## 2. Why `useEffect` Is the Most Abused Hook

The official React docs have a page literally called **["You Might Not Need an Effect"](https://react.dev/learn/you-might-not-need-an-effect)** — because the team saw `useEffect` being used as a hammer for every nail.

### Anti-pattern 1: deriving state from props

```tsx
// ❌ Wrong — uses an effect to sync derived state
function UserCard({ user }) {
  const [fullName, setFullName] = useState('');
  useEffect(() => {
    setFullName(`${user.first} ${user.last}`);
  }, [user]);
  return <h1>{fullName}</h1>;
}

// ✅ Right — compute during render
function UserCard({ user }) {
  const fullName = `${user.first} ${user.last}`;
  return <h1>{fullName}</h1>;
}
```

The effect version causes an extra render cycle and a one-frame flash of empty `fullName`.

### Anti-pattern 2: fetching data with raw `useEffect`

```tsx
// ❌ Most React tutorials show this. Don't write production code like it.
function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  useEffect(() => {
    fetch('/api/users')
      .then(r => r.json())
      .then(data => { setUsers(data); setLoading(false); });
  }, []);
  // No retry, no cache, no dedupe, no error handling, no race conditions.
}

// ✅ Use TanStack Query v5 — covered in Topic 13
import { useQuery } from '@tanstack/react-query';
function UserList() {
  const { data, isLoading } = useQuery({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then(r => r.json()),
  });
}
```

### Anti-pattern 3: notifying parent of state change

```tsx
// ❌ Reactive cascade — parent → child → effect → setParentState → loop
function Form({ onChange }) {
  const [value, setValue] = useState('');
  useEffect(() => { onChange(value); }, [value, onChange]);
}

// ✅ Call onChange directly in the event handler
function Form({ onChange }) {
  const [value, setValue] = useState('');
  function handleChange(e) {
    setValue(e.target.value);
    onChange(e.target.value);
  }
}
```

### When you DO need `useEffect`

| Use case | Real example |
|----------|--------------|
| Subscribing to a non-React store | `window.addEventListener('resize', …)` for a custom hook |
| Connecting to an external system | Open a Supabase realtime channel, close on unmount |
| Manually syncing with a third-party library | Initialize a Mapbox GL JS 3.x map on a `<div>` |
| Updating browser API state | `document.title`, `localStorage`, MutationObserver |

For server data, **always** use TanStack Query v5 / SWR 2 / Next.js Server Components — never `useEffect + fetch`.

---

## 3. The Deps Array — Three States, Three Behaviors

```tsx
// (a) No deps array → run after EVERY render
useEffect(() => { console.log('every render'); });

// (b) Empty deps → run ONCE after first commit
useEffect(() => { console.log('mount'); }, []);

// (c) With deps → run after first commit + whenever deps change (by Object.is)
useEffect(() => { console.log('userId or tab changed'); }, [userId, tab]);
```

### Why React compares deps with `Object.is`

```tsx
// ❌ Inline object/array → new reference every render → effect runs every render
useEffect(() => doStuff(filters), [{ status: 'active' }]);

// ✅ Stable reference
const filters = useMemo(() => ({ status: 'active' }), []);
useEffect(() => doStuff(filters), [filters]);
```

This is the #1 cause of "useEffect runs every render" in real codebases. ESLint rule `react-hooks/exhaustive-deps` from `eslint-plugin-react-hooks@5` will warn you.

---

## 4. Cleanup — Always Mandatory for Subscriptions

```tsx
// Real example — websocket connection
useEffect(() => {
  const socket = new WebSocket('wss://api.example.com');
  socket.onmessage = (e) => setMessages(m => [...m, e.data]);

  return () => socket.close();   // cleanup runs on unmount AND before next effect run
}, [roomId]);
```

**Cleanup runs in two scenarios:**
1. Component unmounts.
2. Effect is about to re-run because deps changed (cleanup of *previous* effect runs first).

Without cleanup, every `roomId` change leaves a dangling socket — memory leak.

### React 18 Strict Mode — effects run twice in development

```tsx
// In <React.StrictMode> (default in Vite + React 18, Next.js 14):
//   mount → effect runs → cleanup runs → effect runs again
```

Why? To surface bugs from missing cleanup. If your effect breaks when run twice in dev, it'll break in production after Fast Refresh, navigation, etc. **Production is not affected** — Strict Mode only double-runs in development.

If your code can't tolerate Strict Mode, fix the code. Don't disable Strict Mode.

---

## 5. Common Pitfalls — Real Bugs

### Pitfall 1: Stale closures

```tsx
// ❌ count is captured at the first render — always 0
useEffect(() => {
  const id = setInterval(() => console.log(count), 1000);
  return () => clearInterval(id);
}, []);  // empty deps means count never refreshes

// ✅ Include count in deps OR use a ref
useEffect(() => {
  const id = setInterval(() => console.log(count), 1000);
  return () => clearInterval(id);
}, [count]);
```

### Pitfall 2: Race conditions in fetch effects

```tsx
// ❌ Two rapid filter changes — older response can arrive last and overwrite newer state
useEffect(() => {
  fetch(`/api/items?q=${query}`).then(r => r.json()).then(setItems);
}, [query]);

// ✅ Cancel via AbortController
useEffect(() => {
  const ctrl = new AbortController();
  fetch(`/api/items?q=${query}`, { signal: ctrl.signal })
    .then(r => r.json())
    .then(setItems)
    .catch(e => { if (e.name !== 'AbortError') throw e; });
  return () => ctrl.abort();
}, [query]);

// ✅✅ Or just use TanStack Query — handles this automatically
```

### Pitfall 3: Setting state in an effect → infinite loop

```tsx
// ❌ count changes → effect runs → setCount → re-render → effect runs → ...
useEffect(() => { setCount(count + 1); }, [count]);

// React will throw "Maximum update depth exceeded" after 25 iterations.
```

---

## 6. `useEffect` vs `useLayoutEffect` vs `useInsertionEffect`

| Hook | Timing | Use case |
|------|--------|----------|
| `useEffect` | After paint | 99% of cases — network, subscriptions, timers |
| `useLayoutEffect` | Before paint, synchronously | DOM measurement (`getBoundingClientRect`), focus, scroll position |
| `useInsertionEffect` | Even earlier, before DOM mutations apply | CSS-in-JS libs only (used internally by `styled-components 6`, `@emotion/react 11`) |

**Real example for `useLayoutEffect`** — animating from a measured size:

```tsx
import { useLayoutEffect, useRef, useState } from 'react';

function GrowingBox({ children }) {
  const ref = useRef<HTMLDivElement>(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    setHeight(ref.current!.getBoundingClientRect().height);
  }, [children]);

  return <div ref={ref} style={{ '--h': `${height}px` }}>{children}</div>;
}
```

If you used `useEffect` here, the user would see the wrong height for one frame before the correct one paints — visual jank.

---

## 7. The Modern Alternative: `use(promise)` (React 19+)

React 19 (RC at time of writing) introduces `use()`, which can read promises directly during render and integrate with `<Suspense>`:

```tsx
// React 19 / Next.js 15 — Server Components
async function UserCard({ id }: { id: string }) {
  const user = await fetch(`/api/users/${id}`).then(r => r.json());
  return <h1>{user.name}</h1>;
}
// No useEffect, no useState, no loading state.
```

For client components, the new `use()` hook unwraps a promise inside a `<Suspense>` boundary. This is the future direction — fewer raw effects, more declarative async.

---

## Summary

| Rule | Why |
|------|-----|
| Don't use `useEffect` to derive state | Compute during render instead |
| Don't fetch data with raw `useEffect` | Use TanStack Query v5, SWR 2, or Next.js Server Components |
| Always return a cleanup function for subscriptions | Otherwise: memory leaks |
| Strict Mode runs effects twice in dev | Embrace it; fix your code, not the warning |
| Inline objects/arrays in deps → infinite re-runs | Memoize with `useMemo` or pull them out |
| Use `useLayoutEffect` only for DOM measurement | Otherwise prefer `useEffect` |

---

## Further reading

- [React docs — You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- [React docs — Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects)
- [Dan Abramov — A Complete Guide to useEffect](https://overreacted.io/a-complete-guide-to-useeffect/)
