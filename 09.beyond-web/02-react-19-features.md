# Beyond the Web — 02. React 19 Features: use(), Actions, useOptimistic, useFormStatus, the React Compiler

> **What / Why / How** — React 19 is mostly about *removing* code: less `useEffect`, less `useState`, less manual memoization. Learn the five APIs that replace them.

---

## 1. The Real Picture in 2026

`react@19.0` shipped December 2024. By mid-2026 it's the default in:
- `next@15` (Next.js made the bump in October 2024)
- `vite@5` + `@vitejs/plugin-react@4`
- Expo SDK 52 and later
- Most new project starters (`create-next-app`, `create-vite` React template, T3 stack)

If you're on `next@14.x` or `react@18.x`, **most APIs in this file work in a backported form** — `useFormState` (renamed `useActionState` in 19), `useOptimistic`, `useFormStatus`, and `<form action={...}>` were already shipping in `react@18.3` and `next@14.3+`. The `use()` hook and the React Compiler are 19-specific.

This file covers all of them with their stable 19 names, noting v18 fallbacks where relevant.

---

## 2. The Big Theme: Remove Effects, Remove State, Remove Memo

A 2025 React team blog summed up React 19 as "the version that lets you delete code." Three concrete reductions:

| Pattern in React 18 | What replaces it in React 19 |
|---------------------|------------------------------|
| `useEffect` to fetch data | `use(promise)` + Suspense, or Server Component `await` |
| `useState` for "submitting / pending / error" around mutations | `useActionState` + Actions (`<form action>` directly) |
| Manual `useMemo` / `useCallback` for stability | The **React Compiler** auto-memoizes |
| Pessimistic UI ("set loading, await, set result") | `useOptimistic` shows the result immediately and rolls back on error |
| `forwardRef` boilerplate | `ref` is a regular prop |

You **don't have to** adopt all of them at once. They compose; pick the ones that fit your team's pace.

---

## 3. `use()` — Read a Promise (or Context) During Render

### What

`use()` is the only React hook that can be called **conditionally**. It reads a promise or a context and suspends the component if the promise isn't resolved yet.

```tsx
import { use } from 'react';

function UserName({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);  // suspends until the promise resolves
  return <h1>{user.name}</h1>;
}

// Wrap in Suspense to define the fallback:
<Suspense fallback={<Skeleton />}>
  <UserName userPromise={fetch('/api/me').then(r => r.json())} />
</Suspense>
```

### Why this beats `useEffect + useState`

The pre-React-19 pattern:

```tsx
// ❌ Old way — 8 lines of state machinery for one fetch
function UserCard() {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  useEffect(() => {
    fetch('/api/me').then(r => r.json()).then(d => { setUser(d); setLoading(false); }).catch(e => { setError(e); setLoading(false); });
  }, []);
  if (loading) return <Skeleton />;
  if (error) return <ErrorBox error={error} />;
  return <h1>{user!.name}</h1>;
}

// ✅ New way — Suspense + use()
function UserCard({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);
  return <h1>{user.name}</h1>;
}
```

The error UI moves to an `ErrorBoundary`, the loading UI moves to a `<Suspense fallback>`. The component body shrinks to one line.

### How to actually create the promise — three real patterns

**Pattern A — Server Component creates and passes down**

```tsx
// app/me/page.tsx (Next.js 15 Server Component)
import { Suspense } from 'react';
import { UserCard } from './UserCard';

export default function Page() {
  const userPromise = fetch('/api/me').then(r => r.json()) as Promise<User>;
  return (
    <Suspense fallback={<Skeleton />}>
      <UserCard userPromise={userPromise} />
    </Suspense>
  );
}
```

The Server Component creates the promise, **does not await it**, and passes it as a prop. The client `<UserCard>` calls `use(userPromise)` and suspends. Streaming SSR sends the skeleton first, then streams the resolved chunk.

**Pattern B — TanStack Query's `suspense: true` mode**

```tsx
function UserCard() {
  const { data } = useQuery({
    queryKey: ['me'],
    queryFn: () => fetch('/api/me').then(r => r.json()),
    suspense: true,
  });
  return <h1>{data.name}</h1>;
}
```

Same `<Suspense>` boundary catches it. `data` is now non-nullable (TanStack v5 is suspense-aware). Covered in `04.state-and-data/03-data-fetching.md`.

**Pattern C — Conditionally read context**

```tsx
import { use, createContext } from 'react';

const ThemeCtx = createContext<'light' | 'dark' | null>(null);

function MaybeThemed({ enabled }: { enabled: boolean }) {
  if (!enabled) return null;
  const theme = use(ThemeCtx); // ✅ legal — `use` allows conditional reads
  return <div data-theme={theme} />;
}
```

`useContext` would error here ("hooks must be called in the same order every render"). `use` is explicitly designed for this.

### Pitfalls

- **Don't create the promise inside the component that calls `use()`.** Each render creates a new promise → infinite suspend loop. Pass it down from a parent or stable cache (TanStack Query, `cache()`).
- **Add `<Suspense>` and an `<ErrorBoundary>`.** Without them, an unresolved promise crashes upward.
- **Don't conflate it with `useEffect`.** `use()` runs during render. Side effects still belong in actions or effects.

---

## 4. Actions — `<form action={fn}>` Directly Calls a Function

### What

A "Server Action" is a function marked `'use server'` that React can call from a `<form>` or a button. The framework handles serialization, network, response handling, and re-rendering. **You don't write `fetch()`.**

This was introduced in `next@14.0` and is built into `react@19` itself.

```tsx
// app/posts/actions.ts (Server Action — runs on the server)
'use server';
import { z } from 'zod';
import { revalidatePath } from 'next/cache';

const Schema = z.object({ title: z.string().min(1), body: z.string().min(1) });

export async function createPost(prev: unknown, formData: FormData) {
  const parsed = Schema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { error: parsed.error.flatten().fieldErrors };
  await db.post.create({ data: parsed.data });
  revalidatePath('/posts');
  return { success: true };
}
```

```tsx
// app/posts/new/page.tsx (Server Component)
import { createPost } from '../actions';

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" />
      <textarea name="body" />
      <button type="submit">Publish</button>
    </form>
  );
}
```

### Why Actions matter

- Forms work **without JavaScript** (progressive enhancement). On the first request, the server processes the form and re-renders; on subsequent navigations, React intercepts and avoids the round-trip.
- No `e.preventDefault()`, no client-side `fetch`, no JSON serialization.
- Auto-invalidates cache via `revalidatePath` / `revalidateTag` (covered in `07.nextjs/03-data-fetching.md`).
- Same Zod schema can be reused on the client form via `react-hook-form` (covered in `04.state-and-data/01-forms.md`).

### Client-side actions — same pattern, runs in the browser

```tsx
'use client';

function ContactForm() {
  async function submit(formData: FormData) {
    // No 'use server' → runs in the browser
    await fetch('/api/contact', { method: 'POST', body: formData });
  }
  return (
    <form action={submit}>
      <input name="email" />
      <button>Send</button>
    </form>
  );
}
```

`<form action={fn}>` is a built-in React feature now — it works for any function, server or client. The framework adapter (Next.js, Remix v3, TanStack Start) handles routing the call.

---

## 5. `useActionState` — Track Pending State and Return Value

### What

`useActionState` (React 19) replaces `useFormState` (React 18) — same hook, renamed. It wraps an Action and exposes:
- The latest return value (state).
- A wrapped action you bind to `<form action={...}>`.
- An `isPending` flag.

```tsx
'use client';
import { useActionState } from 'react';
import { createPost } from './actions';

type State = { error?: Record<string, string[]>; success?: boolean } | null;

export function NewPostForm() {
  const [state, formAction, isPending] = useActionState<State, FormData>(createPost, null);

  return (
    <form action={formAction}>
      <input name="title" />
      {state?.error?.title && <p className="text-red-600">{state.error.title[0]}</p>}

      <textarea name="body" />
      {state?.error?.body && <p className="text-red-600">{state.error.body[0]}</p>}

      <button disabled={isPending}>{isPending ? 'Publishing…' : 'Publish'}</button>
      {state?.success && <p className="text-green-600">Published.</p>}
    </form>
  );
}
```

### Why this beats manual state

The pre-19 pattern needed `useState` for `loading`, `useState` for `error`, `useState` for `success`, and you had to glue them together. With `useActionState`:
- Pending state is automatic.
- Errors and success live in the same `state` object (the action's return value).
- Validation errors flow back through the same channel as success.

### React 18 fallback name

If you're on React 18.3 or `next@14.x`, the hook is called **`useFormState`** and lives in `react-dom`:

```tsx
import { useFormState } from 'react-dom'; // React 18
```

API is identical. The rename to `useActionState` and the move to `react` happened in 19.

---

## 6. `useFormStatus` — Read Pending State from Inside a Form

### What

A hook that reads the **enclosing `<form>`'s** submission status. Any descendant can call it. No prop drilling, no context wiring.

```tsx
'use client';
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending, data, method } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Saving…' : 'Save'}
    </button>
  );
}

// Use anywhere inside a <form>:
<form action={saveProfile}>
  <input name="name" />
  <SubmitButton />
</form>
```

### Why this matters

Before `useFormStatus`, you had to lift `isPending` to the parent and pass it as a prop to every spinner / disabled button / pending-text component. Now any descendant of the `<form>` reads it directly.

Pair it with `useActionState`: the parent uses `useActionState` to track the **return value** (errors, success); descendants use `useFormStatus` to track the **pending state**. They overlap on the `pending` value but `useFormStatus` is local-scoped and avoids prop drilling.

### Pitfall

`useFormStatus` only sees its **direct ancestor `<form>`**. If you nest forms (rare), each `useFormStatus` reads its closest one. It returns `{ pending: false, ...}` if there's no ancestor form (e.g., during SSR before hydration).

---

## 7. `useOptimistic` — Instant UI Feedback With Rollback

### What

`useOptimistic` lets you show a "predicted" UI state immediately, then roll it back if the action fails.

```tsx
'use client';
import { useOptimistic } from 'react';
import { addLike } from './actions';

type LikeState = { count: number; liked: boolean };

export function LikeButton({ post }: { post: { id: string; likes: number; likedByMe: boolean } }) {
  const [optimistic, addOptimistic] = useOptimistic<LikeState, void>(
    { count: post.likes, liked: post.likedByMe },
    (state) => ({
      count: state.count + (state.liked ? -1 : 1),
      liked: !state.liked,
    })
  );

  async function handleClick() {
    addOptimistic();           // UI updates IMMEDIATELY to the predicted state
    await addLike(post.id);    // network call; if it throws, React rolls back
  }

  return (
    <button onClick={handleClick}>
      {optimistic.liked ? '♥' : '♡'} {optimistic.count}
    </button>
  );
}
```

### Why this matters

Before `useOptimistic`, instant feedback required `useState` mirrors of server data + rollback logic in `catch` blocks. With `useOptimistic`:
- Predicted state shows in the next render — **before** the network call.
- React automatically reverts when the wrapping action transitions out of pending state.
- No `try/catch` rollback to write.

### Real fit

- Like / heart / star buttons.
- "Add to cart" with predicted total.
- Send-message UIs (the message appears immediately; if send fails, it greys out).
- Drag-to-reorder lists (list reflects new order before the server confirms).

For drag-reorder specifically, TanStack Query's `onMutate` rollback pattern (covered in `04.state-and-data/03-data-fetching.md`) is more battle-tested. `useOptimistic` is the right tool for one-off optimistic updates inside an Action.

### Pitfall

`useOptimistic` only works when the surrounding update is wrapped in a transition (Action, `startTransition`, or a Server Action via `<form action>`). If you call `addOptimistic()` from a plain handler with no surrounding transition, the predicted state never reverts.

---

## 8. The React Compiler — Auto-Memoization at Build Time

### What

The **React Compiler** (formerly "React Forget") is a Babel plugin (`babel-plugin-react-compiler@1`) that analyzes your function components and hooks and inserts memoization automatically.

```tsx
// You write this:
function ProductList({ products, filter }) {
  const visible = products.filter(p => p.category === filter);
  return <ul>{visible.map(p => <ProductCard key={p.id} product={p} />)}</ul>;
}

// The compiler emits roughly:
function ProductList({ products, filter }) {
  const visible = $.cache(products, filter, () => products.filter(p => p.category === filter));
  return <ul>{visible.map(p => <ProductCard key={p.id} product={p} />)}</ul>;
}
```

`useMemo`, `useCallback`, `React.memo` mostly become unnecessary.

### Why this is a big deal

Hand-written memoization is:
- Easy to forget (deps array maintenance is tedious).
- Easy to over-apply (premature optimization → bookkeeping cost).
- Easy to under-apply (one missed `useMemo` re-renders 100 children).

The compiler is more correct than humans at this — it tracks every value the component reads and memoizes only when it pays off.

### Status as of mid-2026

- **Stable preview** since May 2024. Used in production at Meta on instagram.com.
- Officially recommended for new projects on `react@19` + `next@15`.
- Enabled per-project; coexists with manual `useMemo` (the compiler skips already-memoized code).

### Setup

**Next.js 15:**

```js
// next.config.js
module.exports = {
  experimental: { reactCompiler: true },
};
```

**Vite 5:**

```js
// vite.config.ts
import react from '@vitejs/plugin-react';
export default defineConfig({
  plugins: [react({ babel: { plugins: ['babel-plugin-react-compiler'] } })],
});
```

Then install the plugin:

```bash
npm i -D babel-plugin-react-compiler@1
```

### What changes in your code

- **Existing `useMemo`/`useCallback` calls keep working.** The compiler is conservative and won't fight your code.
- **Stop adding new ones.** Write components plainly; let the compiler handle stability.
- **`React.memo` is mostly unneeded.** The compiler memoizes per-prop by default.

### When the compiler can't help

- **Code that violates the Rules of Hooks** (mutating state during render, conditional hook calls). The `eslint-plugin-react-compiler@1` linter flags these — fix them or the compiler skips the file.
- **Mutable refs or external mutation** that the compiler can't reason about. It treats `useRef().current` as opaque and skips memoization touching it.
- **`'use no memo'` directive** at the top of a file — opts out per-file. Use sparingly; it's an escape hatch for the rare case the compiler gets it wrong.

---

## 9. `ref` as a Regular Prop — Goodbye `forwardRef`

### What

In React 19, `ref` is a normal prop on function components. The `forwardRef` wrapper is no longer needed.

```tsx
// React 18
import { forwardRef } from 'react';
const TextField = forwardRef<HTMLInputElement, Props>((props, ref) => (
  <input ref={ref} {...props} />
));

// React 19
function TextField({ ref, ...props }: Props & { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...props} />;
}
```

### Why

- Removes a layer of wrapping.
- Generic components no longer need the `forwardRef` cast workaround (covered in `05.routing-and-styling/03-typescript-with-react.md`).
- TypeScript inference works without ceremony.

### Migration

`forwardRef` still works in React 19 — the deprecation is non-breaking. Codemod when convenient:

```bash
npx codemod react/19/replace-use-form-state
npx codemod react/19/remove-forward-ref
```

---

## 10. Other Real React 19 Changes Worth Knowing

### Document metadata as regular elements

```tsx
function Page() {
  return (
    <article>
      <title>Article Title</title>
      <meta name="description" content="..." />
      <link rel="canonical" href="https://example.com/article" />
      <h1>Article Title</h1>
    </article>
  );
}
```

React 19 hoists `<title>`, `<meta>`, `<link>` to `<head>` automatically. Useful for libraries; in Next.js you'd still use the Metadata API (covered in `07.nextjs/07-seo-metadata.md`).

### Stylesheets with priority

```tsx
<link rel="stylesheet" href="/styles.css" precedence="default" />
```

React deduplicates by `precedence` and orders correctly across Suspense boundaries. Helps CSS-in-JS libraries author SSR.

### Scripts with `async`

```tsx
<script async src="https://example.com/analytics.js" />
```

React deduplicates and hoists once on first render in a tree. No more "third-party script loaded twice" bugs.

### Better hydration error messages

Hydration mismatches in 19 print a diff of expected vs actual DOM, plus the component path. Significantly better than 18's generic "Text content does not match" error.

---

## 11. Migrating from React 18 to React 19

The official migration guide is at [react.dev/blog/2024/04/25/react-19-upgrade-guide](https://react.dev/blog/2024/04/25/react-19-upgrade-guide). Real-world checklist:

- [ ] Bump `react` and `react-dom` to `^19`.
- [ ] Update `@types/react` and `@types/react-dom` to `^19`.
- [ ] Run codemods: `npx types-react-codemod@latest preset-19` (handles deprecated PropTypes, defaultProps, etc).
- [ ] Replace `useFormState` imports from `react-dom` with `useActionState` from `react`.
- [ ] Replace `forwardRef` calls with direct `ref` props (codemod available).
- [ ] Update libraries: most popular ones (`@tanstack/react-query@5.28+`, `next@15+`, `framer-motion@11.5+`) are already React 19-ready.
- [ ] Watch for deprecated APIs in console: `String refs` (`ref="someName"`), `defaultProps` on function components, `propTypes`, `findDOMNode`. All removed in 19.
- [ ] Re-run TypeScript: stricter inference may surface latent type bugs.

### Don't migrate if

- Your app is on React 17 — bump to 18 first, settle, then 19.
- You depend on a library that explicitly hasn't shipped 19 support (rare; most have by mid-2026).
- You're in the middle of a critical release. Migration takes a day for typical apps but always spawns 1–3 unforeseen issues.

---

## 12. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `use(promise)` infinitely suspends | The promise is recreated each render. Move it to a parent or wrap with TanStack Query / `cache()` |
| `useFormStatus` returns `pending: false` always | The hook is in a component that's not a descendant of any `<form>` |
| `useActionState` errors not displaying | Action returned `undefined` instead of the state object on failure paths |
| `useOptimistic` doesn't revert | The mutation isn't wrapped in a transition; only Actions and `startTransition` reset it |
| React Compiler skipping a component | Run `eslint-plugin-react-compiler` — it points at the rule violation |
| Hydration error in dev only | Open the new diff message; usually nondeterministic render (`Date.now`, `Math.random`, locale-dependent strings) |
| Server Action throws `Cannot read 'use server' from outside a Server Component` | The file's top-level `'use server'` directive is missing or in the wrong file |
| Dropped `forwardRef` but generic component lost type | Use the React 19 generics — `function List<T>({ ref, items }: Props<T> & { ref?: Ref<...> })` works without cast tricks |

---

## 13. Decision Tree — When to Reach for What

```
Need to fetch data on the client?
├── Server Component path available? → fetch in the Server Component, await directly
├── Client component path? → @tanstack/react-query@5.28 with suspense: true + use()
└── Promise comes in as a prop? → use(promise) inside <Suspense>

Building a form?
├── Mostly server-side mutation? → Server Action + <form action> + useActionState
├── Need fine-grained client validation? → react-hook-form@7.51 + Zod, post via Server Action
└── Sub-component needs pending state? → useFormStatus inside the form

Want instant feedback for an action?
└── useOptimistic wrapped around the Action call

Manually memoizing for performance?
├── On React 19 + Next.js 15? → Enable React Compiler, delete most useMemo/useCallback
└── On React 18? → Keep manual memoization; plan upgrade

Adding new ref-using component?
├── React 19? → ref is a normal prop, no forwardRef
└── React 18? → forwardRef as before
```

---

## 14. Connections to the Rest of the Curriculum

- **`02.hooks/04-usememo-usecallback.md`** — the manual memoization the React Compiler replaces.
- **`04.state-and-data/01-forms.md`** — `useActionState` and `useFormStatus` extend the form patterns there.
- **`04.state-and-data/03-data-fetching.md`** — `use(promise)` is the lower-level building block under TanStack Query's suspense mode.
- **`07.nextjs/03-data-fetching.md`** — `<form action>` and Server Actions are detailed there.
- **`05.routing-and-styling/03-typescript-with-react.md`** — `ref` as a regular prop simplifies the generic-component examples.

---

## Summary

| API | Replaces | Available since |
|-----|----------|-----------------|
| `use(promise)` | `useEffect + useState` for fetch | React 19 |
| `use(context)` | `useContext` (when you need conditional reads) | React 19 |
| `<form action={fn}>` | `<form onSubmit>` + manual fetch | React 19 (and `next@14.0` Server Actions) |
| `useActionState` | Manual pending/error state | React 19 (was `useFormState` in 18) |
| `useFormStatus` | Prop drilling pending state to children | React 18.3 / 19 |
| `useOptimistic` | Manual rollback after failed mutation | React 18.3 / 19 |
| React Compiler | Hand-written `useMemo`/`useCallback`/`React.memo` | Stable preview May 2024 |
| `ref` as prop | `forwardRef` boilerplate | React 19 |
| `<title>`, `<meta>`, `<link>` hoisting | DIY `<head>` management | React 19 |

| Rule | Why |
|------|-----|
| Default to Server Actions for mutations | Eliminates an entire layer of API routes for internal flows |
| Pair `useActionState` with `useFormStatus` | Parent owns return value; descendants own pending UI |
| Use `useOptimistic` only inside Actions | It only resets when the surrounding transition resets |
| Enable the React Compiler in new projects | Stop hand-tuning `useMemo`/`useCallback` |
| Don't recreate the promise passed to `use()` | Memoize at a stable scope or use TanStack Query |

---

## Further reading

- [React 19 release notes](https://react.dev/blog/2024/12/05/react-19)
- [React 19 upgrade guide](https://react.dev/blog/2024/04/25/react-19-upgrade-guide)
- [React Compiler — installation](https://react.dev/learn/react-compiler)
- [Next.js 15 — React 19 support](https://nextjs.org/blog/next-15)
- [`useActionState` reference](https://react.dev/reference/react/useActionState)
- [`useOptimistic` reference](https://react.dev/reference/react/useOptimistic)
- [`useFormStatus` reference](https://react.dev/reference/react-dom/hooks/useFormStatus)
- [`use` reference](https://react.dev/reference/react/use)
