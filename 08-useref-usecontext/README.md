# Topic 08 — useRef & useContext: DOM Access, Context Patterns, Pitfalls

> **What / Why / How** — refs are escape hatches; context solves prop drilling but not state management.

---

## Part 1: useRef

## 1. What Is `useRef`?

### What

`useRef(initialValue)` returns a mutable object `{ current: T }` that persists across renders. **Mutating `.current` does not trigger a re-render.**

```tsx
const ref = useRef<HTMLInputElement>(null);
console.log(ref.current);  // null on first render, HTMLInputElement after mount
```

Two distinct uses:
1. **DOM refs** — pass `ref={ref}` to a JSX element to get the underlying DOM node.
2. **Mutable instance values** — store a value across renders without causing re-renders.

### Why React separates state from refs

| | `useState` | `useRef` |
|---|------------|----------|
| Mutation triggers re-render? | ✅ Yes | ❌ No |
| Read during render? | Always safe | Avoid (refs may not match render output) |
| Use for UI display | ✅ Yes | ❌ No — UI won't update |
| Use for DOM nodes | ❌ No | ✅ Yes |
| Use for timers, sockets, latest-value caches | ❌ No (unnecessary re-renders) | ✅ Yes |

---

## 2. DOM Refs — Real Use Cases

### Focus management

```tsx
function SearchBar() {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    inputRef.current?.focus();   // focus on mount
  }, []);

  return <input ref={inputRef} placeholder="Search..." />;
}
```

### Scroll into view

```tsx
function ChatLog({ messages }: { messages: Msg[] }) {
  const endRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    endRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages.length]);

  return (
    <div className="chat">
      {messages.map(m => <Bubble key={m.id} msg={m} />)}
      <div ref={endRef} />
    </div>
  );
}
```

### Integrating non-React libraries (Mapbox GL JS 3, Three.js r161, Chart.js 4)

```tsx
import mapboxgl from 'mapbox-gl';

function MapView({ lng, lat }: { lng: number; lat: number }) {
  const containerRef = useRef<HTMLDivElement>(null);
  const mapRef = useRef<mapboxgl.Map>();

  useEffect(() => {
    if (!containerRef.current) return;
    mapRef.current = new mapboxgl.Map({
      container: containerRef.current,
      center: [lng, lat],
      zoom: 12,
    });
    return () => mapRef.current?.remove();
  }, []); // initialize once

  useEffect(() => {
    mapRef.current?.flyTo({ center: [lng, lat] });
  }, [lng, lat]); // update when props change

  return <div ref={containerRef} style={{ height: 400 }} />;
}
```

This is the canonical "React + imperative library" pattern — used by every Mapbox, Leaflet, Three.js, and Monaco Editor integration.

---

## 3. `forwardRef` — Forwarding Refs Through Custom Components

By default, custom components don't accept `ref`. To allow `ref` to pass through to a DOM element:

```tsx
import { forwardRef } from 'react';

interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
}

export const TextField = forwardRef<HTMLInputElement, InputProps>(
  ({ label, ...rest }, ref) => (
    <label>
      <span>{label}</span>
      <input ref={ref} {...rest} />
    </label>
  )
);
TextField.displayName = 'TextField';

// Usage:
const ref = useRef<HTMLInputElement>(null);
<TextField ref={ref} label="Email" />
```

This is the same pattern shadcn/ui uses for every form primitive.

### React 19 simplification

In React 19, `ref` is just a regular prop — no `forwardRef` needed:

```tsx
// React 19+
function TextField({ ref, label, ...rest }: InputProps & { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...rest} />;
}
```

---

## 4. Refs as Mutable Storage (No Re-render)

### Latest-value cache (avoiding stale closures)

```tsx
function Chat({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<Msg[]>([]);
  const messagesRef = useRef(messages);
  messagesRef.current = messages; // keep ref in sync

  useEffect(() => {
    const socket = new WebSocket(`/ws/${roomId}`);
    socket.onmessage = (e) => {
      const newMsg = JSON.parse(e.data);
      // Use the ref to read the LATEST messages, not the closed-over one
      console.log('Total now:', messagesRef.current.length + 1);
      setMessages(m => [...m, newMsg]);
    };
    return () => socket.close();
  }, [roomId]);
}
```

### Storing a timer / interval ID

```tsx
function StopWatch() {
  const [seconds, setSeconds] = useState(0);
  const intervalRef = useRef<number>();

  function start() {
    intervalRef.current = window.setInterval(() => setSeconds(s => s + 1), 1000);
  }
  function stop() {
    clearInterval(intervalRef.current);
  }
  // ...
}
```

### When NOT to use a ref for "performance"

Don't replace `useState` with `useRef` to "avoid re-renders" — your UI will stop reflecting reality. Only use refs for values that should NOT trigger UI updates.

---

## 5. `useImperativeHandle` — Custom Ref API

Rare but useful: when a parent needs to call methods on a child component (focus, reset, scrollTo).

```tsx
import { forwardRef, useImperativeHandle, useRef } from 'react';

type FormHandle = { reset: () => void; submit: () => void };

const Form = forwardRef<FormHandle>((_, ref) => {
  const inputRef = useRef<HTMLInputElement>(null);

  useImperativeHandle(ref, () => ({
    reset: () => { inputRef.current!.value = ''; },
    submit: () => { console.log(inputRef.current!.value); },
  }));

  return <input ref={inputRef} />;
});

// Parent:
const formRef = useRef<FormHandle>(null);
<Form ref={formRef} />
<button onClick={() => formRef.current?.reset()}>Reset</button>
```

Use sparingly — usually props + callbacks are cleaner.

---

## Part 2: useContext

## 6. What Is `useContext`?

### What

Context is React's built-in mechanism for passing values from an ancestor to deeply nested descendants without manually drilling props through every layer.

```tsx
import { createContext, useContext, useState } from 'react';

type Theme = 'light' | 'dark';
const ThemeContext = createContext<{ theme: Theme; toggle: () => void } | null>(null);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light');
  const toggle = () => setTheme(t => t === 'light' ? 'dark' : 'light');
  return (
    <ThemeContext.Provider value={{ theme, toggle }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be inside <ThemeProvider>');
  return ctx;
}

// Anywhere deep in the tree:
function Header() {
  const { theme, toggle } = useTheme();
  return <button onClick={toggle}>{theme}</button>;
}
```

### When context is the right tool

- Theme (light/dark)
- Current authenticated user
- Locale / i18n
- Feature flag values
- A router (React Router v6 uses context internally)

### When context is the wrong tool

- **Server cache** — use TanStack Query v5 instead.
- **Complex global state with frequent updates** — use Zustand 4.5 or Redux Toolkit 2.
- **Form state** — use `react-hook-form@7.51`.

---

## 7. The Context Re-render Problem

```tsx
// ❌ EVERY consumer re-renders when ANY value in `value` changes
<UserContext.Provider value={{ user, setUser, prefs, setPrefs }}>
```

If a child only reads `user`, but `prefs` changes, that child still re-renders. Context distributes the entire `value` reference; React doesn't auto-shallow-compare.

### Three real fixes

**Fix 1: Split contexts by update frequency**

```tsx
<UserContext.Provider value={user}>
  <PrefsContext.Provider value={prefs}>
    {children}
  </PrefsContext.Provider>
</UserContext.Provider>
```

A consumer of `UserContext` no longer re-renders when prefs change.

**Fix 2: Stable value reference**

```tsx
const value = useMemo(() => ({ user, setUser }), [user]);
<UserContext.Provider value={value}>
```

**Fix 3: Use a state library instead of context**

For global state with frequent updates, Zustand 4.5 / Jotai 2 / Redux Toolkit 2 use **selector subscriptions** that only re-render when the selected slice changes:

```tsx
// Zustand
import { create } from 'zustand';
const useStore = create<{ user: User; prefs: Prefs }>(...)

// Component only re-renders when user changes; prefs updates don't trigger this component.
const user = useStore(s => s.user);
```

Topic 12 covers this in depth.

---

## 8. Real-World Pattern: Custom Hook Per Context

Always export a hook, not the context object:

```tsx
// ❌ Consumers do `useContext(ThemeContext)` — easy to forget the null check
export const ThemeContext = createContext<...>(null);

// ✅ Always export the hook with the null guard built in
export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be inside <ThemeProvider>');
  return ctx;
}
```

Used by every well-built React library: `react-router`, `@tanstack/react-query`, `next-themes`, `@radix-ui/react-toast`.

---

## 9. Provider Composition — Avoiding Provider Hell

```tsx
// ❌ Provider pyramid
<QueryClientProvider client={queryClient}>
  <ThemeProvider>
    <AuthProvider>
      <I18nProvider>
        <App />
      </I18nProvider>
    </AuthProvider>
  </ThemeProvider>
</QueryClientProvider>

// ✅ Compose them
function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider>
        <AuthProvider>
          <I18nProvider>{children}</I18nProvider>
        </AuthProvider>
      </ThemeProvider>
    </QueryClientProvider>
  );
}
```

Or use the `react-flatten-providers@1` library if your provider list is huge — but in practice a single `AppProviders` wrapper is enough.

---

## Summary

| Hook | Purpose | Re-render? |
|------|---------|-----------|
| `useRef` for DOM | Imperative DOM access (focus, scroll, third-party libs) | No |
| `useRef` for values | Mutable storage that should NOT trigger renders | No |
| `forwardRef` | Forward `ref` through a custom component | n/a |
| `useImperativeHandle` | Expose method API to parent via ref | n/a |
| `useContext` | Read shared value provided by ancestor | Yes — every consumer on value change |

| Rule | Why |
|------|-----|
| Don't read refs during render | Their value isn't synchronized with the rendered output |
| Split contexts by update frequency | Avoid full re-render cascades |
| Use Zustand / Jotai / Redux Toolkit for frequently-updating global state | Selector subscriptions only re-render needed components |
| Export a `useX()` hook, not the context object | Centralizes the null check; better DX |

---

## Further reading

- [React docs — useRef](https://react.dev/reference/react/useRef)
- [React docs — useContext](https://react.dev/reference/react/useContext)
- [Kent C. Dodds — How to use React Context effectively](https://kentcdodds.com/blog/how-to-use-react-context-effectively)
