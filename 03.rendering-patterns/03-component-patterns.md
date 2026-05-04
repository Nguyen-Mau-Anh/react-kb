# Rendering Patterns — 03. Component Patterns: Compound, Render Props, HOC, Slot Patterns

> **What / Why / How** — five real component patterns. Most you'll *consume*; one or two you'll *write*.

---

## 1. Why Patterns Matter

A "pattern" is a recognized shape for solving a recurring component-design problem. They give you:

- A vocabulary to read other people's code (Radix UI, react-router, react-hook-form all use these names).
- A default answer when designing your own components — so you don't reinvent something worse.

The five patterns covered here, with their real-world owners:

| Pattern | Real library that uses it | What it solves |
|---------|---------------------------|----------------|
| Compound components | `@radix-ui/react-dialog@1.1`, `@radix-ui/react-tabs@1.1`, `@headlessui/react@2` | Many parts that share state without prop drilling |
| Render props | `react-hook-form@7.51` `<Controller render={…}>`, `@tanstack/react-query`'s old `<Query>` | Inverting control of what gets rendered |
| Higher-Order Components (HOC) | `react-redux@9` `connect`, legacy `withRouter`, `next-auth@5`'s `withAuth` middleware | Cross-cutting concerns when hooks aren't an option |
| Slot pattern (`children` + named props) | Next.js 14 layouts, Remix `<Outlet />`, `@radix-ui/react-slot@1.1` | Letting a parent control structure, child control content |
| Provider + custom hook | `react-router@6` `<RouterProvider>`, `@tanstack/react-query`'s `<QueryClientProvider>` | Sharing state down a subtree without context boilerplate at every consumer |

---

## 2. Compound Components

### What

Multiple components that look standalone but are designed to work together as one unit. The parent owns shared state; children consume it via internal context.

```tsx
// Looks like this from the outside — Radix UI Tabs:
import * as Tabs from '@radix-ui/react-tabs';

function MyTabs() {
  return (
    <Tabs.Root defaultValue="profile">
      <Tabs.List>
        <Tabs.Trigger value="profile">Profile</Tabs.Trigger>
        <Tabs.Trigger value="settings">Settings</Tabs.Trigger>
      </Tabs.List>
      <Tabs.Content value="profile">Profile content</Tabs.Content>
      <Tabs.Content value="settings">Settings content</Tabs.Content>
    </Tabs.Root>
  );
}
```

### Why this beats a "kitchen-sink props" component

Compare to a hypothetical monolithic API:

```tsx
// ❌ Anti-pattern — the props balloon as features grow
<Tabs
  tabs={[
    { id: 'profile',  label: 'Profile',  content: <ProfileBody /> },
    { id: 'settings', label: 'Settings', content: <SettingsBody /> },
  ]}
  defaultActive="profile"
  renderTab={(t) => …}
  renderPanel={(t) => …}
  tabClassName="…"
  panelClassName="…"
/>
```

The compound version lets you put any markup between `<Tabs.List>` and `<Tabs.Content>`, mix in your own classes, reorder elements, or omit the list entirely — without the parent component caring.

### How — write your own with Context

```tsx
import { createContext, useContext, useState, ReactNode } from 'react';

type AccordionCtx = { openId: string | null; setOpenId: (id: string | null) => void };
const Ctx = createContext<AccordionCtx | null>(null);
const useCtx = () => {
  const v = useContext(Ctx);
  if (!v) throw new Error('Accordion.* must be inside <Accordion.Root>');
  return v;
};

function Root({ children }: { children: ReactNode }) {
  const [openId, setOpenId] = useState<string | null>(null);
  return <Ctx.Provider value={{ openId, setOpenId }}>{children}</Ctx.Provider>;
}

function Item({ id, children }: { id: string; children: ReactNode }) {
  return <div data-id={id}>{children}</div>;
}

function Trigger({ id, children }: { id: string; children: ReactNode }) {
  const { openId, setOpenId } = useCtx();
  return (
    <button onClick={() => setOpenId(openId === id ? null : id)}>{children}</button>
  );
}

function Content({ id, children }: { id: string; children: ReactNode }) {
  const { openId } = useCtx();
  return openId === id ? <div>{children}</div> : null;
}

export const Accordion = { Root, Item, Trigger, Content };
```

Usage:

```tsx
<Accordion.Root>
  <Accordion.Item id="a">
    <Accordion.Trigger id="a">Show A</Accordion.Trigger>
    <Accordion.Content id="a">A body</Accordion.Content>
  </Accordion.Item>
</Accordion.Root>
```

### When NOT to use compound components

If the relationship is fixed and small (e.g. `<Card><CardHeader /><CardBody /></Card>` with no shared state), a plain composition with `children` is simpler. Compound components only earn their keep when there's shared state to coordinate.

---

## 3. Render Props

### What

A component takes a function as a prop (often `children`) and calls it to render its output, passing data the parent needs.

```tsx
// react-hook-form Controller — the canonical modern render-prop API
import { Controller, useForm } from 'react-hook-form';

const { control } = useForm<{ country: string }>();

<Controller
  name="country"
  control={control}
  render={({ field, fieldState }) => (
    <CustomSelect
      value={field.value}
      onChange={field.onChange}
      error={fieldState.error?.message}
    />
  )}
/>
```

The `Controller` owns form state and validation; the consumer owns *how it looks*.

### Why render props still exist in a hooks world

Hooks replaced render props for sharing **logic**. Render props are still useful for sharing **behavior with rendering control** — when the component owns lifecycle/state but can't predict the markup. That's exactly the case for `<Controller>` because the form library doesn't know if you're using a Radix Select, a MUI TextField, or a custom component.

### How — write your own

```tsx
type DownloadProps<T> = {
  url: string;
  children: (state: { data: T | null; loading: boolean; error: Error | null }) => ReactNode;
};

function Download<T>({ url, children }: DownloadProps<T>) {
  const [state, setState] = useState({ data: null, loading: true, error: null });
  useEffect(() => {
    fetch(url).then(r => r.json())
      .then(data => setState({ data, loading: false, error: null }))
      .catch(error => setState({ data: null, loading: false, error }));
  }, [url]);
  return <>{children(state)}</>;
}

// Usage
<Download<User> url="/api/me">
  {({ data, loading }) => loading ? <Spinner /> : <h1>{data?.name}</h1>}
</Download>
```

### When NOT to use render props

If you only share data (no rendering control needed), use a custom hook. `useUser()` is cleaner than `<UserProvider>{(user) => ...}</UserProvider>`.

---

## 4. Higher-Order Components (HOC)

### What

A function that takes a component and returns a new component with extra props or behavior.

```tsx
// react-redux 9 — connect HOC (still in the API, though hooks are preferred)
import { connect } from 'react-redux';
const mapState = (state: RootState) => ({ user: state.user });
export default connect(mapState)(MyComponent);
```

### Why HOCs are mostly legacy in 2026

| Concern | HOC | Custom hook |
|---------|-----|-------------|
| Wrapping noise in DevTools | `connect(connect(withRouter(MyPage)))` | None |
| Prop name collisions | Real risk (`withFoo` injects `foo`, conflicts with caller's `foo`) | None |
| TypeScript inference | Painful generics: `<P>(C: ComponentType<P & Injected>) => ComponentType<Omit<P, keyof Injected>>` | Trivial: hook return type |
| Composability | Pyramid (`a(b(c(d(Component))))`) | `const a = useA(); const b = useB();` |

The React team officially recommends hooks over HOCs for new code ([React docs — Reusing Logic with Custom Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks)).

### When HOCs still make sense

- **Ecosystem you can't change**: legacy Redux connected components, pre-hooks routers.
- **Wrapping the entire component tree once** at the framework boundary — e.g. Next.js `withAuth` middleware-style wrappers, or `next-i18next@15`'s `appWithTranslation`.
- **Dev-tools instrumentation**: `withErrorBoundary` from `react-error-boundary@4`.

For 99% of new code: write a custom hook instead.

---

## 5. Slot Pattern — `children` and Named Slots

### What

Let the parent decide structure, the child decide content, by passing React nodes through props.

#### Single slot — `children`

```tsx
function Card({ children }: { children: ReactNode }) {
  return <div className="card">{children}</div>;
}
<Card><h1>Title</h1><p>Body</p></Card>
```

#### Named slots — multiple props that are ReactNode

```tsx
type LayoutProps = {
  header: ReactNode;
  sidebar: ReactNode;
  children: ReactNode;
};
function Layout({ header, sidebar, children }: LayoutProps) {
  return (
    <div className="layout">
      <header>{header}</header>
      <aside>{sidebar}</aside>
      <main>{children}</main>
    </div>
  );
}
<Layout header={<NavBar />} sidebar={<Filters />}>
  <Results />
</Layout>
```

This is conceptually how **Next.js 14 parallel routes** work — `app/dashboard/@analytics/page.tsx` and `app/dashboard/@team/page.tsx` are passed as named props to `app/dashboard/layout.tsx`.

### `@radix-ui/react-slot` — the asChild pattern

Radix UI uses a special `<Slot>` primitive so that `<Tooltip.Trigger asChild>` doesn't render its own `<button>` but instead merges its props into the child:

```tsx
import * as Tooltip from '@radix-ui/react-tooltip';

// Without asChild — Trigger renders a default button (extra DOM node)
<Tooltip.Trigger>Hover me</Tooltip.Trigger>

// With asChild — Trigger merges onto the existing <a>, no extra wrapper
<Tooltip.Trigger asChild>
  <a href="/profile">Hover me</a>
</Tooltip.Trigger>
```

This is the cleanest way to compose behavior onto an arbitrary host element. `shadcn/ui` builds nearly every primitive on top of `@radix-ui/react-slot@1.1`.

---

## 6. Provider + Hook Pattern

### What

Pair a context Provider with a custom hook that consumes it. The consumer never touches `useContext` directly.

```tsx
const AuthCtx = createContext<{ user: User | null; signIn: () => void } | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const signIn = useCallback(async () => {
    const u = await api.signIn();
    setUser(u);
  }, []);
  const value = useMemo(() => ({ user, signIn }), [user, signIn]);
  return <AuthCtx.Provider value={value}>{children}</AuthCtx.Provider>;
}

export function useAuth() {
  const v = useContext(AuthCtx);
  if (!v) throw new Error('useAuth must be inside <AuthProvider>');
  return v;
}
```

### Why every modern React library does this

- `@tanstack/react-query@5.28`: `<QueryClientProvider>` + `useQueryClient()`.
- `react-router@6.22`: `<RouterProvider>` + `useNavigate()`, `useLoaderData()`.
- `next-themes@0.3`: `<ThemeProvider>` + `useTheme()`.

The pattern hides context boilerplate from consumers and gives a typed, single-import API.

### Tradeoff already covered

Frequent updates inside the provider re-render every consumer. For high-frequency state, prefer `zustand@4.5` (selector subscriptions) — see `02.hooks/03-useref-usecontext.md` and the upcoming `04.state-and-data/02-state-management.md`.

---

## 7. Pattern Decision Cheat Sheet

| You need to… | Use |
|--------------|-----|
| Build a UI primitive with multiple coordinated parts (Tabs, Accordion, Dialog) | Compound components + Context |
| Own state/lifecycle but let the consumer render markup (form fields, data fetcher) | Render props |
| Inject cross-cutting concerns into a tree once (auth, error boundary) | HOC at the boundary; custom hook everywhere else |
| Let parent compose layout, child fill content | `children` or named-prop slots |
| Share state down a subtree with a clean consumer API | Provider + custom hook |
| Add behavior to an arbitrary child element without a wrapper | `@radix-ui/react-slot` `asChild` |

---

## 8. Anti-Patterns — Don't Do These

### Anti-pattern 1: Passing JSX through state

```tsx
// ❌ State holds React elements
const [content, setContent] = useState(<div>Hello</div>);
```

Elements are immutable; storing them in state defeats reconciliation, breaks Fast Refresh, and confuses everyone. Pass data, render JSX from data.

### Anti-pattern 2: Cloning children with `React.cloneElement` to inject props

```tsx
// ❌ Brittle — assumes specific child shape, breaks composition
function Parent({ children }: { children: ReactElement }) {
  return React.cloneElement(children, { onClick: handleClick });
}
```

Use a Provider + hook instead. `cloneElement` is fine for very narrow cases (single direct child you control), but it's a bug magnet for general use.

### Anti-pattern 3: God-component "configurable everything"

```tsx
// ❌ 30 props later, you've reinvented HTML, badly
<DataTable
  data={…} columns={…} sortable={…} filterable={…} pagination={…}
  renderRow={…} renderCell={…} renderHeader={…} renderEmpty={…}
  classNames={…} stylesObject={…} ariaProps={…} …
/>
```

Break into compound components: `<Table.Root><Table.Header /><Table.Body>{rows.map(…)}</Table.Body></Table.Root>`.

---

## Summary

| Pattern | Owns | Caller controls | Real consumer in 2026 |
|---------|------|-----------------|------------------------|
| Compound | Shared state | Markup + structure | Radix UI 1.1, Headless UI 2 |
| Render prop | State + lifecycle | What gets rendered | react-hook-form Controller |
| HOC | Wrapping logic | Inner component | Mostly legacy; `react-error-boundary@4` |
| Slot (`children` / named) | Container layout | Content | Next.js 14 layouts, MUI Drawer |
| Provider + hook | Subtree state | Read access via hook | Every modern React library |

| Rule | Why |
|------|-----|
| Reach for hooks before HOCs | Better TS, no wrapper hell, no name collisions |
| Use `@radix-ui/react-slot` for `asChild` composition | Avoids extra DOM wrappers in shadcn/ui-style libraries |
| Don't store JSX in state | Pass data; render JSX during render |
| Compound > god-component when 5+ config props pile up | Markup composition scales; prop count doesn't |

---

## Further reading

- [React docs — Passing JSX as children](https://react.dev/learn/passing-props-to-a-component#passing-jsx-as-children)
- [Radix UI primitives source](https://github.com/radix-ui/primitives)
- [shadcn/ui — uses asChild + compound patterns extensively](https://ui.shadcn.com/)
- [Kent C. Dodds — Compound Components With React Hooks](https://kentcdodds.com/blog/compound-components-with-react-hooks)
