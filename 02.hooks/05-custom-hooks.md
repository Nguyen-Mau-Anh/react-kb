# Hooks — 05. Custom Hooks: Extraction Patterns, useFetch / useDebounce / useLocalStorage

> **What / Why / How** — custom hooks are the React unit of reuse. Build them like a proper API.

---

## 1. What Is a Custom Hook?

### What

A custom hook is a function whose name starts with `use` and which calls other hooks. It bundles stateful logic so multiple components can share it.

```tsx
function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn(o => !o), []);
  return [on, toggle] as const;
}

// Usage
function Sidebar() {
  const [open, toggleOpen] = useToggle(false);
  return <button onClick={toggleOpen}>{open ? 'Close' : 'Open'}</button>;
}
```

### Why custom hooks beat the alternatives

| Reuse mechanism | Era | Trade-off |
|-----------------|-----|-----------|
| Higher-Order Components (HOC) — `withRouter`, `connect` | React 0.14–16 | Wrapper hell, prop name collisions, `displayName` ceremony |
| Render props — `<Mouse>{({x,y}) => ...}</Mouse>` | React 15–16 | Indentation pyramid, no static types |
| **Custom hooks** | React 16.8+ | Plain functions, full TypeScript, no nesting, composable |

The React docs explicitly recommend hooks over HOCs and render props for new code ([Reusing Logic with Custom Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks)).

### Why naming with `use` matters

The `eslint-plugin-react-hooks@5` linter only enforces the [Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks) (no calling inside conditions, only at top level, only in functions whose name starts with `use` or `Use`) when the function name starts with `use`. Without it, you'd silently break those rules.

---

## 2. Anatomy of a Good Custom Hook

```tsx
import { useEffect, useState, useCallback } from 'react';

/**
 * Persist a value in localStorage. Returns a tuple identical to useState.
 *
 * @param key       — localStorage key (must be stable across renders)
 * @param initial   — fallback when nothing is stored
 */
function useLocalStorage<T>(key: string, initial: T): [T, (next: T) => void] {
  const [value, setValue] = useState<T>(() => {
    try {
      const raw = window.localStorage.getItem(key);
      return raw !== null ? (JSON.parse(raw) as T) : initial;
    } catch {
      return initial;
    }
  });

  const set = useCallback((next: T) => {
    setValue(next);
    try {
      window.localStorage.setItem(key, JSON.stringify(next));
    } catch {/* quota exceeded, private mode */}
  }, [key]);

  return [value, set];
}
```

What this example demonstrates:

- **Lazy `useState` initializer** — runs once, avoids reading localStorage on every render (see `02.hooks/01-usestate.md`).
- **Tuple return** — same shape as `useState`, predictable for callers.
- **`as const` not needed** when return type is explicit.
- **Defensive try/catch** — Safari Private Mode throws on `setItem`.
- **Generics** — `<T>` typed to whatever the caller stores.

---

## 3. Real Custom Hooks Worth Knowing

### `useDebounce` — debounce a value

```tsx
function useDebounce<T>(value: T, delay = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}

// Usage — search input
function Search() {
  const [query, setQuery] = useState('');
  const debounced = useDebounce(query, 300);
  const { data } = useQuery({ queryKey: ['search', debounced], queryFn: () => api.search(debounced) });
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

This is one of the few cases where a custom hook beats library competition. `lodash.debounce@4.0.8` is fine for plain functions but doesn't integrate with React's render cycle.

### `useEventListener` — type-safe DOM event subscription

```tsx
function useEventListener<K extends keyof WindowEventMap>(
  type: K,
  listener: (e: WindowEventMap[K]) => void
) {
  useEffect(() => {
    window.addEventListener(type, listener);
    return () => window.removeEventListener(type, listener);
  }, [type, listener]);
}

useEventListener('keydown', (e) => {
  if (e.key === 'Escape') closeModal();
});
```

### `useMediaQuery` — responsive logic

```tsx
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() =>
    typeof window !== 'undefined' ? window.matchMedia(query).matches : false
  );
  useEffect(() => {
    const mql = window.matchMedia(query);
    const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
    mql.addEventListener('change', handler);
    return () => mql.removeEventListener('change', handler);
  }, [query]);
  return matches;
}

const isDesktop = useMediaQuery('(min-width: 768px)');
```

### `useIntersectionObserver` — visibility / lazy load / infinite scroll

```tsx
function useIntersectionObserver<T extends HTMLElement>(
  options?: IntersectionObserverInit
): [React.RefObject<T>, boolean] {
  const ref = useRef<T>(null);
  const [intersecting, setIntersecting] = useState(false);
  useEffect(() => {
    if (!ref.current) return;
    const observer = new IntersectionObserver(
      ([entry]) => setIntersecting(entry.isIntersecting),
      options
    );
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, [options]);
  return [ref, intersecting];
}

const [ref, visible] = useIntersectionObserver<HTMLImageElement>({ threshold: 0.5 });
return <img ref={ref} src={visible ? real : placeholder} />;
```

---

## 4. Custom `useFetch` — and Why You Probably Shouldn't Write One

You can write a `useFetch` in 30 lines:

```tsx
function useFetch<T>(url: string): { data: T | null; loading: boolean; error: Error | null } {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const ctrl = new AbortController();
    setLoading(true);
    fetch(url, { signal: ctrl.signal })
      .then(r => r.json())
      .then(setData)
      .catch(e => { if (e.name !== 'AbortError') setError(e); })
      .finally(() => setLoading(false));
    return () => ctrl.abort();
  }, [url]);

  return { data, loading, error };
}
```

This works — but it doesn't give you:

| Feature | Hand-rolled `useFetch` | `@tanstack/react-query@5.28` |
|---------|------------------------|------------------------------|
| Request deduplication (3 components, 1 network call) | ❌ | ✅ |
| Background refetch on window focus | ❌ | ✅ |
| Cache + stale-while-revalidate | ❌ | ✅ |
| Pagination, infinite scroll helpers | ❌ | ✅ |
| Optimistic updates | ❌ | ✅ |
| DevTools | ❌ | ✅ |
| Mutation lifecycle (onMutate, onSuccess, onError, onSettled) | ❌ | ✅ |

For real apps, use TanStack Query (Topic `04.state-and-data/03-data-fetching.md`). Build your own only as a learning exercise or for a one-off use case.

---

## 5. Extracting Logic — When to Make a Custom Hook

| Situation | Extract? |
|-----------|----------|
| Same `useEffect` + `useState` pattern in 3+ components | ✅ Yes |
| Logic depends on the DOM / browser API (resize, scroll, media query) | ✅ Yes |
| Logic needs to be tested in isolation (with `@testing-library/react@15`'s `renderHook`) | ✅ Yes |
| One-off state in a single component | ❌ No — keep it inline |
| Logic is just `useState` + a setter | ❌ No — extraction adds noise without saving lines |

### Single Responsibility Principle for hooks

A custom hook should answer **one question**: "give me a debounced value", "tell me if I'm online", "persist this in localStorage". A hook called `useUserDashboard` that does fetching + filtering + theme + permissions is a code smell — split it.

---

## 6. Testing Custom Hooks — `renderHook` from React Testing Library

```bash
npm i -D @testing-library/react@15 vitest@1.6
```

```tsx
import { renderHook, act } from '@testing-library/react';
import { useToggle } from './useToggle';

test('toggle flips boolean', () => {
  const { result } = renderHook(() => useToggle(false));
  expect(result.current[0]).toBe(false);
  act(() => result.current[1]());
  expect(result.current[0]).toBe(true);
});
```

`renderHook` mounts a tiny wrapper component that calls your hook and lets you assert against `result.current`. Used by every well-tested React hook library.

---

## 7. Library Alternatives — Don't Reinvent

Before writing a custom hook, check these. They're battle-tested:

| Library | What it gives | npm install |
|---------|---------------|-------------|
| `usehooks-ts@3` | 40+ TypeScript-first hooks (useDebounce, useLocalStorage, useMediaQuery, etc.) | `npm i usehooks-ts@3` |
| `react-use@17` | 100+ hooks; older but huge surface area | `npm i react-use@17` |
| `@uidotdev/usehooks@2` | Curated set; smaller, modern | `npm i @uidotdev/usehooks@2` |
| `react-hookz/web@24` | Modern, tree-shakeable, TS-native | `npm i @react-hookz/web@24` |

For a brand-new project in 2026: `usehooks-ts@3` is a solid pick — small bundle impact, all TS, mirrors the names from `usehooks.com`.

---

## 8. Composing Hooks — Build Bigger from Smaller

```tsx
// Compose useDebounce + useLocalStorage to make "saved-but-debounced search query"
function useDebouncedLocalStorage<T>(key: string, initial: T, delay = 500) {
  const [value, setValue] = useLocalStorage<T>(key, initial);
  const debounced = useDebounce(value, delay);
  return [value, setValue, debounced] as const;
}
```

This is the real power of hooks: compose them like LEGO. HOCs can't be composed without nesting hell; render props can't be composed without turning your tree into a maze.

---

## Summary

| Rule | Why |
|------|-----|
| Name starts with `use` | Required for `eslint-plugin-react-hooks` to enforce hook rules |
| Return tuple `[value, setter]` for state-shape hooks | Matches `useState` mental model |
| Return object `{ data, loading, error }` for richer hooks | Self-documenting fields |
| Don't write `useFetch` for production | Use `@tanstack/react-query@5.28` |
| Single responsibility per hook | `useDebounce` good; `useUserDashboard` bad |
| Test with `renderHook` from `@testing-library/react@15` | Same library you already use for components |
| Check `usehooks-ts@3` before writing | Most "I need a custom hook for X" already exists there |

---

## Further reading

- [React docs — Reusing Logic with Custom Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks)
- [usehooks-ts website](https://usehooks-ts.com/)
- [@testing-library/react — renderHook](https://testing-library.com/docs/react-testing-library/api/#renderhook)
- [Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks)
