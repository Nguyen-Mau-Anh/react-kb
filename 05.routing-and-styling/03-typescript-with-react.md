# Routing & Styling — 03. TypeScript with React: Component Types, Generics, Event Types, Utility Types

> **What / Why / How** — TypeScript 5.4 + React 18 is the production default. Avoid `React.FC`, learn five utility types, and stop using `any`.

---

## 1. Setup — The TypeScript Versions That Matter

| Tooling | Version | Notes |
|---------|---------|-------|
| `typescript` | `5.4` (current stable through mid-2026) | `satisfies`, `using`, NoInfer<T>, const generics |
| `@types/react` | `18.3.x` | Matches React 18.3; for React 19 RC, use `19.x` |
| `@types/react-dom` | `18.3.x` | |
| `tsconfig.json` baseline | `"jsx": "react-jsx"`, `"strict": true`, `"moduleResolution": "bundler"` | Vite 5 + Next.js 14 default |

Real `tsconfig.json` baseline used by `create-vite@5` + React template:

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,   // catches arr[i] returning T | undefined
    "noImplicitOverride": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true
  }
}
```

**`noUncheckedIndexedAccess: true` is the single most valuable strict flag** beyond `strict` itself. It surfaces real array/object index bugs the rest of `strict` misses.

---

## 2. Typing Component Props — Don't Use `React.FC`

### What people get told to do

```tsx
// ❌ Don't write this in 2026
const Greeting: React.FC<{ name: string }> = ({ name }) => <h1>Hi {name}</h1>;
```

### Why `React.FC` is the wrong default

1. **Implicitly accepts `children`** until React 18 — surprise prop you didn't declare. (Fixed in `@types/react@18`, but the muscle memory remains.)
2. **No support for generic components** — you can't write `<List<T> items={...}>` while using `FC`.
3. **`defaultProps` typing is weird** — partially deprecated in React 18.3.
4. **Display name inference** is no better than a plain function.

### What to write instead

```tsx
// ✅ Plain function declaration with typed props
type GreetingProps = { name: string };

export function Greeting({ name }: GreetingProps) {
  return <h1>Hi {name}</h1>;
}
```

This is the React docs' current example style ([react.dev — Passing Props](https://react.dev/learn/passing-props-to-a-component)) and what `shadcn/ui`, `next.js`, Vercel templates all use.

### When you *do* want `children`

```tsx
import type { ReactNode } from 'react';

type CardProps = {
  title: string;
  children: ReactNode; // declared explicitly
};

export function Card({ title, children }: CardProps) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div>{children}</div>
    </div>
  );
}
```

`ReactNode` is the right type for "anything React can render" (string, number, JSX, fragment, array, null, undefined, boolean).

| Type | Means | Use when |
|------|-------|----------|
| `ReactNode` | Anything renderable | `children` prop, slot props |
| `ReactElement` | A specific JSX element (`<div>`, `<MyComp />`) | You'll call `React.cloneElement` or check `.type` |
| `JSX.Element` | Mostly synonymous with `ReactElement` | Function return type if you must annotate |
| `ComponentType<P>` | A component (function or class) accepting props `P` | HOCs, render-prop library APIs |

---

## 3. Extending Native HTML Element Props

The single most important pattern: a custom `<Button>` should accept any prop a real `<button>` accepts (`disabled`, `type`, `aria-label`, `onClick`, etc.).

```tsx
import type { ButtonHTMLAttributes } from 'react';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary';
}

export function Button({ variant = 'primary', className, ...rest }: ButtonProps) {
  return <button className={`btn btn-${variant} ${className ?? ''}`} {...rest} />;
}
```

The matching attribute interface for every HTML element lives in `@types/react`:

| Element | Type |
|---------|------|
| `<button>` | `ButtonHTMLAttributes<HTMLButtonElement>` |
| `<input>` | `InputHTMLAttributes<HTMLInputElement>` |
| `<a>` | `AnchorHTMLAttributes<HTMLAnchorElement>` |
| `<form>` | `FormHTMLAttributes<HTMLFormElement>` |
| `<textarea>` | `TextareaHTMLAttributes<HTMLTextAreaElement>` |
| Generic | `HTMLAttributes<HTMLElement>` |

This is exactly how `shadcn/ui`'s `Button` is declared, verbatim.

---

## 4. Event Handler Types — The Cheat Sheet

```tsx
// Click
const onClick: React.MouseEventHandler<HTMLButtonElement> = (e) => {
  e.currentTarget; // typed as HTMLButtonElement (use this — it's stable)
  e.target;        // EventTarget — needs cast
};

// Change (input)
const onChange: React.ChangeEventHandler<HTMLInputElement> = (e) => {
  const value = e.target.value; // string — typed because input is HTMLInputElement
};

// Form submit
const onSubmit: React.FormEventHandler<HTMLFormElement> = (e) => {
  e.preventDefault();
  const data = new FormData(e.currentTarget);
};

// Keyboard
const onKeyDown: React.KeyboardEventHandler<HTMLInputElement> = (e) => {
  if (e.key === 'Enter') doSubmit();
};

// Drag
const onDragStart: React.DragEventHandler<HTMLDivElement> = (e) => {
  e.dataTransfer.setData('text/plain', '...');
};

// Pointer (drag handles, gesture libs like @dnd-kit/core@6)
const onPointerDown: React.PointerEventHandler<HTMLDivElement> = (e) => {
  e.currentTarget.setPointerCapture(e.pointerId);
};
```

**Rule of thumb**: `e.currentTarget` is typed; `e.target` is `EventTarget` and usually needs a cast. Prefer `currentTarget` whenever you're reading the element the listener is attached to.

---

## 5. Generic Components

The single feature you cannot do with `React.FC`:

```tsx
import { type ReactNode } from 'react';

type ListProps<T> = {
  items: T[];
  renderItem: (item: T) => ReactNode;
  keyOf: (item: T) => string | number;
};

export function List<T>({ items, renderItem, keyOf }: ListProps<T>) {
  return <ul>{items.map(item => <li key={keyOf(item)}>{renderItem(item)}</li>)}</ul>;
}

// Usage — TS infers <User> from props
<List
  items={users}
  keyOf={(u) => u.id}
  renderItem={(u) => <span>{u.name}</span>}
/>
```

### `as const` props for narrow inference

```tsx
type SelectProps<T extends string> = {
  options: readonly T[];
  value: T;
  onChange: (v: T) => void;
};

function Select<T extends string>({ options, value, onChange }: SelectProps<T>) {
  return (
    <select value={value} onChange={e => onChange(e.target.value as T)}>
      {options.map(o => <option key={o} value={o}>{o}</option>)}
    </select>
  );
}

const STATUSES = ['active', 'inactive', 'banned'] as const;
<Select options={STATUSES} value="active" onChange={(v) => /* v: 'active' | 'inactive' | 'banned' */} />
```

`as const` makes `STATUSES` `readonly ['active', 'inactive', 'banned']` instead of `string[]`, so TS narrows `T` to the union.

---

## 6. Discriminated Unions for Variant Props

Mutually-exclusive prop combinations are best modeled with a discriminated union, not optional fields.

```tsx
// ❌ Optional everything — caller can pass nonsense
type AlertProps = {
  variant: 'success' | 'error';
  message: string;
  retry?: () => void; // only meaningful for 'error'
};

// ✅ Discriminated union — TS enforces correctness
type AlertProps =
  | { variant: 'success'; message: string }
  | { variant: 'error';   message: string; retry: () => void };

function Alert(props: AlertProps) {
  if (props.variant === 'error') {
    return <button onClick={props.retry}>{props.message}</button>; // retry is typed
  }
  return <span>{props.message}</span>;
}

<Alert variant="error" message="Failed" retry={...} />     // ✅
<Alert variant="error" message="Failed" />                 // ❌ TS error: retry missing
<Alert variant="success" message="Done" retry={...} />     // ❌ TS error: retry not allowed
```

This is a key pattern. Use it for any "two modes" component.

---

## 7. Hook Typing — `useState`, `useReducer`, `useRef`, `useContext`

### `useState`

```tsx
const [count, setCount] = useState(0);                  // inferred number
const [user, setUser] = useState<User | null>(null);    // explicit when initial doesn't carry the type
const [items, setItems] = useState<Item[]>([]);         // empty array → must annotate
```

### `useReducer`

```tsx
type State = { count: number; loading: boolean };
type Action = { type: 'inc' } | { type: 'dec' } | { type: 'reset'; to: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'inc':   return { ...state, count: state.count + 1 };
    case 'dec':   return { ...state, count: state.count - 1 };
    case 'reset': return { ...state, count: action.to };
  }
}

const [state, dispatch] = useReducer(reducer, { count: 0, loading: false });
```

Discriminated unions for `Action` give you exhaustive `switch` checking.

### `useRef`

```tsx
const inputRef = useRef<HTMLInputElement>(null);             // DOM ref — initial null is mandatory
const idRef = useRef<number>();                              // mutable instance value, may be undefined
const idRef2 = useRef<number>(0);                            // mutable instance value with default
```

For DOM refs, initial `null` is required because `ref` is set after mount.

### `useContext`

```tsx
type AuthCtx = { user: User; signOut: () => void };
const Ctx = createContext<AuthCtx | null>(null);

export function useAuth() {
  const v = useContext(Ctx);
  if (!v) throw new Error('useAuth must be inside <AuthProvider>');
  return v; // narrowed to AuthCtx (non-null)
}
```

The `null` default + non-null guard is the idiomatic pattern (covered in `02.hooks/03-useref-usecontext.md`).

---

## 8. Five Utility Types You Will Use Constantly

```tsx
// 1. Partial<T> — make all properties optional (e.g. update payloads)
type UpdateUser = Partial<User>;

// 2. Pick<T, K> — keep only listed keys
type UserPreview = Pick<User, 'id' | 'name' | 'avatar'>;

// 3. Omit<T, K> — drop listed keys
type CreateUser = Omit<User, 'id' | 'createdAt'>;

// 4. Required<T> — make all optional fields required
type FullyConfigured = Required<Settings>;

// 5. Record<K, V> — typed dictionary
type Translations = Record<'en' | 'es' | 'fr', string>;
```

The **most React-specific** ones:

```tsx
// ComponentProps<typeof X> — extract the props of an existing component
type ButtonProps = ComponentProps<typeof Button>;

// ComponentPropsWithoutRef — same, but excludes ref (useful when re-wrapping)
type SafeButtonProps = ComponentPropsWithoutRef<typeof Button>;

// React.PropsWithChildren<P> — extends P with children?: ReactNode
type CardProps = React.PropsWithChildren<{ title: string }>;
```

---

## 9. The `satisfies` Operator — TS 5.x Killer Feature

```tsx
// ❌ Loose: TS forgets the literal types
const ROUTES: Record<string, { path: string; auth: boolean }> = {
  home:    { path: '/',         auth: false },
  profile: { path: '/profile',  auth: true  },
};
ROUTES.profile.auth; // boolean, not literal `true`

// ✅ satisfies: keeps the literal types AND validates shape
const ROUTES = {
  home:    { path: '/',         auth: false },
  profile: { path: '/profile',  auth: true  },
} satisfies Record<string, { path: string; auth: boolean }>;

ROUTES.profile.auth; // exactly `true`
ROUTES.unknown;      // TS error: property doesn't exist
```

Use `satisfies` everywhere you have a constant config object that should be both validated and narrowly typed.

---

## 10. Typing Server Responses with Zod

Don't trust `any` from `fetch`. The 2026 idiomatic pattern:

```bash
npm i zod@3.23
```

```ts
import { z } from 'zod';

const UserSchema = z.object({
  id: z.string(),
  email: z.string().email(),
  name: z.string(),
  age: z.number().int().nonnegative(),
});
type User = z.infer<typeof UserSchema>; // single source of truth for the type

async function fetchUser(id: string): Promise<User> {
  const r = await fetch(`/api/users/${id}`);
  const json = await r.json();
  return UserSchema.parse(json); // throws ZodError if shape is wrong
}
```

The same `UserSchema` is reused on the server (Next.js Route Handler) for input validation — covered in `04.state-and-data/01-forms.md`. One schema, both sides.

---

## 11. Common TypeScript-with-React Pitfalls

| Pitfall | Fix |
|---------|-----|
| `any` from `JSON.parse(...)` flowing into props | Validate with Zod 3.23 or another runtime checker |
| Casting events with `as` instead of typing the handler | Type the handler: `React.MouseEventHandler<HTMLButtonElement>` |
| `useState<string>('')` then trying to `setState(undefined)` | Type as `useState<string \| undefined>` |
| `useRef<HTMLInputElement>()` (no `null`) | Always `useRef<HTMLInputElement>(null)` for DOM refs |
| `forwardRef` losing generics | Use the React 19 form (no `forwardRef` needed); for React 18 use the cast pattern below |
| Component props leaking implicit `any` from rest spread | Don't use `Object` or `{}`; spread typed `...rest` |

### `forwardRef` + generics workaround (React 18)

```tsx
import { forwardRef, type ForwardedRef } from 'react';

type ListProps<T> = { items: T[] };

function ListInner<T>({ items }: ListProps<T>, ref: ForwardedRef<HTMLUListElement>) {
  return <ul ref={ref}>{items.map((i, k) => <li key={k}>{String(i)}</li>)}</ul>;
}

// Cast preserves the generic
export const List = forwardRef(ListInner) as <T>(
  props: ListProps<T> & { ref?: React.ForwardedRef<HTMLUListElement> }
) => ReturnType<typeof ListInner>;
```

In React 19 this is gone — `ref` is just a regular prop.

---

## 12. Strict Mode Wins

| Compiler flag | What it catches |
|--------------|-----------------|
| `strict: true` | The big bundle: `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, etc. |
| `noUncheckedIndexedAccess: true` | `arr[0]` is `T \| undefined` — forces real bounds checks |
| `noFallthroughCasesInSwitch: true` | Enforces exhaustive switches |
| `exactOptionalPropertyTypes: true` | `{ x?: number }` rejects `{ x: undefined }` — useful for stricter shapes |
| `noUnusedLocals: true` + `noUnusedParameters: true` | Cleans dead code (let your linter handle this if you prefer warnings) |

Real-world default for new React projects: enable all of these. The first two are the most valuable.

---

## Summary

| Pattern | Use for |
|---------|---------|
| `function Foo({...}: Props)` | Default component declaration (skip `React.FC`) |
| `extends ButtonHTMLAttributes<HTMLButtonElement>` | Forward all native props through your component |
| `React.MouseEventHandler<HTMLButtonElement>` and friends | Event handler types |
| Generic `function List<T>(...)` | Components that work over any item type |
| Discriminated union props | Mutually exclusive variant configurations |
| `satisfies` | Constant config objects that need validated shape + literal types |
| `z.infer<typeof Schema>` | Single source of truth for runtime + compile types |

| Rule | Why |
|------|-----|
| Don't use `React.FC` | No generic support, implicit children in older `@types/react` |
| Use `e.currentTarget` over `e.target` | `currentTarget` is typed correctly |
| Always pass `null` to `useRef` for DOM refs | `ref` is set after mount |
| Validate fetched data with Zod | `any` from JSON breaks the whole point of TS |
| Enable `noUncheckedIndexedAccess` | Catches a class of array bugs `strict` alone misses |

---

## Further reading

- [TypeScript + React cheatsheet](https://react-typescript-cheatsheet.netlify.app/)
- [TS 5.x release notes — `satisfies`, `using`, `NoInfer<T>`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html)
- [React docs — TypeScript](https://react.dev/learn/typescript)
- [Zod docs](https://zod.dev/)
