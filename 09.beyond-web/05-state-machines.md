# Beyond the Web — 05. State Machines: xstate 5 for Wizards, Checkouts, Drag-Snap

> **What / Why / How** — for any UI with discrete states and impossible transitions, a state machine is cheaper than a tangle of `useState` + `if`-statements. `xstate@5` is the production default; for simpler cases, a plain reducer is fine.

---

## 1. Why State Machines Belong in React UIs

### The class of problems they solve

Some UIs have **modes that exclude each other**:

- A form is `idle | submitting | success | error` — never two at once.
- A multi-step checkout is on step 1, 2, or 3 — never on 1.5.
- A drag interaction is `idle | dragging | snapping-back | dropped`.
- A media player is `loading | playing | paused | ended | error`.
- A onboarding flow has guards: "step 3 needs a verified email; otherwise jump to step 2.5".

The naive React way:

```tsx
// ❌ Boolean soup — eight illegal states sneak in
const [loading, setLoading] = useState(false);
const [error, setError] = useState(null);
const [success, setSuccess] = useState(false);
const [data, setData] = useState(null);
// You can have loading=true AND success=true. You can have error AND data.
// Nothing in TypeScript prevents this.
```

A state machine **makes those illegal states unrepresentable** by encoding the rules in one place.

```ts
// ✅ Three states, each with their own data shape — impossible to be in two
type FormState =
  | { status: 'idle' }
  | { status: 'submitting' }
  | { status: 'success'; data: User }
  | { status: 'error'; message: string };
```

This is the same idea as discriminated unions for component props (covered in `05.routing-and-styling/03-typescript-with-react.md`), applied to the *whole component's lifecycle*.

---

## 2. The Three Levels of State-Machine Discipline

You don't always need `xstate@5`. Pick the lightest tool that solves your problem.

### Level 1 — Discriminated union + `useReducer` (no library)

For 3–5 states, no parallel regions, no async orchestration. Plain React.

```tsx
// useFormReducer.ts
type State =
  | { status: 'idle' }
  | { status: 'submitting' }
  | { status: 'success'; user: User }
  | { status: 'error'; message: string };

type Action =
  | { type: 'submit' }
  | { type: 'success'; user: User }
  | { type: 'error'; message: string }
  | { type: 'reset' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'submit':  return { status: 'submitting' };
    case 'success': return { status: 'success', user: action.user };
    case 'error':   return { status: 'error', message: action.message };
    case 'reset':   return { status: 'idle' };
  }
}

function useLoginForm() {
  const [state, dispatch] = useReducer(reducer, { status: 'idle' });
  return { state, dispatch };
}
```

Use this for: simple multi-state forms, toggles with side effects, anything fitting on one screen.

### Level 2 — `@xstate/store@2` (lightweight, no charts)

`@xstate/store@2` (the same team as XState) is a tiny, zero-config store. Same author, simpler API. ~3 KB.

```ts
import { createStore } from '@xstate/store';
import { useSelector } from '@xstate/store/react';

const store = createStore({
  context: { count: 0 },
  on: {
    inc: { count: (ctx) => ctx.count + 1 },
    dec: { count: (ctx) => ctx.count - 1 },
  },
});

function Counter() {
  const count = useSelector(store, (s) => s.context.count);
  return <button onClick={() => store.send({ type: 'inc' })}>{count}</button>;
}
```

Closer to Zustand than to a full state machine. Use when you want predictable transitions but don't need parallel states, guards, or actor spawning.

### Level 3 — `xstate@5` (full statecharts)

When the rules get real:
- Parallel regions ("dragging" AND "loading next-page" simultaneously).
- Hierarchical states (a "playing" state has nested "buffering" / "ahead").
- Guards (predicates on transitions).
- Side effects scoped to states (`entry`, `exit`, `invoke`).
- Async actors (children machines orchestrated by a parent).

`xstate@5` is the production tool. ~14 KB gzipped.

---

## 3. xstate 5 — The Real Setup

```bash
npm i xstate@5 @xstate/react@5
```

```ts
// machines/checkout.ts
import { setup, assign } from 'xstate';

type CheckoutContext = {
  cart: CartItem[];
  shipping: Address | null;
  payment: PaymentMethod | null;
  orderId: string | null;
  errorMessage: string | null;
};

type CheckoutEvent =
  | { type: 'START'; cart: CartItem[] }
  | { type: 'SUBMIT_SHIPPING'; address: Address }
  | { type: 'SUBMIT_PAYMENT'; payment: PaymentMethod }
  | { type: 'CONFIRM' }
  | { type: 'BACK' }
  | { type: 'RETRY' }
  | { type: 'RESET' };

export const checkoutMachine = setup({
  types: {
    context: {} as CheckoutContext,
    events: {} as CheckoutEvent,
  },
  guards: {
    cartNotEmpty: ({ context }) => context.cart.length > 0,
  },
  actors: {
    submitOrder: async ({ input }: { input: { context: CheckoutContext } }) => {
      const r = await fetch('/api/orders', {
        method: 'POST',
        body: JSON.stringify({
          cart: input.context.cart,
          shipping: input.context.shipping,
          payment: input.context.payment,
        }),
      });
      if (!r.ok) throw new Error(await r.text());
      const { orderId } = await r.json();
      return { orderId };
    },
  },
}).createMachine({
  id: 'checkout',
  initial: 'idle',
  context: {
    cart: [],
    shipping: null,
    payment: null,
    orderId: null,
    errorMessage: null,
  },
  states: {
    idle: {
      on: {
        START: {
          target: 'shipping',
          guard: 'cartNotEmpty',
          actions: assign({ cart: ({ event }) => event.cart }),
        },
      },
    },
    shipping: {
      on: {
        SUBMIT_SHIPPING: {
          target: 'payment',
          actions: assign({ shipping: ({ event }) => event.address }),
        },
      },
    },
    payment: {
      on: {
        SUBMIT_PAYMENT: {
          target: 'review',
          actions: assign({ payment: ({ event }) => event.payment }),
        },
        BACK: 'shipping',
      },
    },
    review: {
      on: {
        CONFIRM: 'submitting',
        BACK: 'payment',
      },
    },
    submitting: {
      invoke: {
        src: 'submitOrder',
        input: ({ context }) => ({ context }),
        onDone: {
          target: 'success',
          actions: assign({ orderId: ({ event }) => event.output.orderId }),
        },
        onError: {
          target: 'error',
          actions: assign({ errorMessage: ({ event }) => (event.error as Error).message }),
        },
      },
    },
    success: {
      type: 'final',
    },
    error: {
      on: {
        RETRY: 'submitting',
        RESET: { target: 'idle', actions: assign({ errorMessage: null }) },
      },
    },
  },
});
```

### What this code says, in plain English

- `idle → shipping`: only if the cart isn't empty (the `cartNotEmpty` guard).
- From `shipping`, the user can move to `payment` by submitting an address.
- From `payment`, they can go forward to `review` or back to `shipping`.
- From `review`, `CONFIRM` triggers `submitting`, which `invoke`s the `submitOrder` actor — an async function.
- On success: transition to `success`. On error: transition to `error` with a retryable path.
- States that don't define a transition for an event simply ignore it. Sending `BACK` while `submitting` does nothing — exactly the behavior you want during a network call.

This is ~80 lines for a feature that, written with `useState`, would be ~200 lines of brittle conditionals.

### Wire to React with `@xstate/react@5`

```tsx
'use client';
import { useMachine } from '@xstate/react';
import { checkoutMachine } from '@/machines/checkout';

export function Checkout({ initialCart }: { initialCart: CartItem[] }) {
  const [state, send] = useMachine(checkoutMachine);

  if (state.matches('idle')) {
    return <button onClick={() => send({ type: 'START', cart: initialCart })}>Begin checkout</button>;
  }
  if (state.matches('shipping')) return <ShippingForm onSubmit={(address) => send({ type: 'SUBMIT_SHIPPING', address })} />;
  if (state.matches('payment'))  return <PaymentForm onSubmit={(payment) => send({ type: 'SUBMIT_PAYMENT', payment })} onBack={() => send({ type: 'BACK' })} />;
  if (state.matches('review'))   return <ReviewStep context={state.context} onConfirm={() => send({ type: 'CONFIRM' })} onBack={() => send({ type: 'BACK' })} />;
  if (state.matches('submitting')) return <p>Placing order…</p>;
  if (state.matches('error'))    return <ErrorView message={state.context.errorMessage!} onRetry={() => send({ type: 'RETRY' })} />;
  if (state.matches('success'))  return <OrderConfirmation orderId={state.context.orderId!} />;
  return null;
}
```

`state.matches('payment')` is fully type-narrowed — TypeScript knows `state.context.shipping` is non-null inside that branch.

### Why this beats hand-rolled

- **Impossible states are gone.** You cannot reach `submitting` without having validated shipping AND payment.
- **The XState DevTools (Stately Inspector)** lets you visualize the machine, replay transitions, jump to states. Real production debugging.
- **Visual editor:** [stately.ai/editor](https://stately.ai/editor) generates the same `setup()`/`createMachine()` code from a diagram. Designers and PMs can edit it.
- **Testable in isolation.** Send synthetic events, assert on resulting state — no DOM needed.

---

## 4. xstate 4 → 5 — The Real Migration

`xstate@4` is widely deployed and significantly different. The `@xstate/react@4` API is also different. If you see a tutorial using:

- `Machine({...})` instead of `setup({...}).createMachine({...})` → it's v4.
- `services` instead of `actors` → it's v4.
- `useService` → it's v4.

For new code: **always v5**. The `setup()` function is the headline v5 API — it's where guards, actors, and types are declared. The migration guide is at [stately.ai/docs/migration](https://stately.ai/docs/migration).

---

## 5. When NOT to Use xstate

The honest counter-cases. xstate is overkill when:

- You have 2 states (`open`/`closed`) — `useState` is fine.
- You're tracking server data — that's TanStack Query (`04.state-and-data/03-data-fetching.md`).
- You're tracking form fields — that's `react-hook-form@7.51` (`04.state-and-data/01-forms.md`).
- You're tracking global UI state with frequent updates — that's Zustand (`04.state-and-data/02-state-management.md`).
- The transitions are deterministic and your team is small — a plain `useReducer` discriminated union is enough and has zero learning curve.

xstate **earns** its 14 KB only when you have:

- **5+ states** with branching transitions.
- **Async orchestration** (load → submit → retry → success).
- **Guards** that must be expressed declaratively (especially shared across transitions).
- **Multi-step flows** where you need to model the *graph*, not just the current step.
- **Actors** — sub-machines that run in parallel and the parent machine coordinates.

---

## 6. Three Real Use Cases, Worked Out

### Use Case A — Multi-Step Wizard (Onboarding)

A signup flow: profile → verify email → invite team → done. The verify step blocks progress until the user clicks the link in their email.

```ts
import { setup, assign } from 'xstate';

export const onboardingMachine = setup({
  types: {
    context: {} as { profile: Profile | null; emailVerified: boolean; teamInvited: boolean },
    events: {} as
      | { type: 'PROFILE_SAVED'; profile: Profile }
      | { type: 'EMAIL_VERIFIED' }
      | { type: 'INVITED' }
      | { type: 'SKIP_INVITES' }
      | { type: 'BACK' },
  },
  guards: {
    canProceedToTeam: ({ context }) => context.emailVerified,
  },
}).createMachine({
  id: 'onboarding',
  initial: 'profile',
  context: { profile: null, emailVerified: false, teamInvited: false },
  states: {
    profile: {
      on: { PROFILE_SAVED: { target: 'verifyEmail', actions: assign({ profile: ({ event }) => event.profile }) } },
    },
    verifyEmail: {
      on: {
        EMAIL_VERIFIED: { target: 'team', actions: assign({ emailVerified: true }) },
        BACK: 'profile',
      },
    },
    team: {
      on: {
        INVITED: { target: 'done', actions: assign({ teamInvited: true }) },
        SKIP_INVITES: 'done',
        BACK: { target: 'verifyEmail', guard: 'canProceedToTeam' },
      },
    },
    done: { type: 'final' },
  },
});
```

The `guard: 'canProceedToTeam'` on `BACK` enforces: even on the team step, the email must still be verified — if it lapsed, the user can't navigate back to revisit. The wizard is now resilient to weird flows without `if`-statements scattered through component code.

### Use Case B — Drag-Snap Interaction (Swipe-to-Delete Card)

A card swipes right; if the user releases past 100px, it deletes; otherwise it snaps back. With `@use-gesture/react@10` for input and `framer-motion@11` for animation:

```ts
import { setup } from 'xstate';

export const swipeMachine = setup({
  types: {
    context: {} as { offset: number },
    events: {} as
      | { type: 'DRAG'; offset: number }
      | { type: 'RELEASE' }
      | { type: 'SNAP_DONE' }
      | { type: 'DELETE_DONE' },
  },
  guards: {
    pastThreshold: ({ context }) => context.offset > 100,
  },
}).createMachine({
  initial: 'idle',
  context: { offset: 0 },
  states: {
    idle: {
      on: { DRAG: { target: 'dragging', actions: assign({ offset: ({ event }) => event.offset }) } },
    },
    dragging: {
      on: {
        DRAG: { actions: assign({ offset: ({ event }) => event.offset }) },
        RELEASE: [
          { target: 'deleting', guard: 'pastThreshold' },
          { target: 'snappingBack' },
        ],
      },
    },
    snappingBack: {
      on: { SNAP_DONE: { target: 'idle', actions: assign({ offset: 0 }) } },
    },
    deleting: {
      on: { DELETE_DONE: 'gone' },
    },
    gone: { type: 'final' },
  },
});
```

The two `RELEASE` transitions ordered by guard is exactly what you want: try the threshold-passed branch first, fall through to snap-back. Clear, branching logic in three lines.

### Use Case C — Media Player

`loading → ready → playing ⇄ paused → ended` plus an `error` state reachable from anywhere. Add **parallel regions** for the seek bar:

```ts
states: {
  loading: { /* ... */ },
  ready: {
    type: 'parallel',
    states: {
      playback: {
        initial: 'paused',
        states: {
          paused:  { on: { PLAY: 'playing' } },
          playing: {
            on: { PAUSE: 'paused' },
            after: { 0: { actions: 'tickPosition' } }, // or use an interval actor
          },
        },
      },
      seekBar: {
        initial: 'static',
        states: {
          static: { on: { START_SEEK: 'seeking' } },
          seeking: {
            on: { END_SEEK: 'static' },
          },
        },
      },
    },
  },
  error: { /* reached via .on at the root */ },
}
```

The user can be `playing.seeking` simultaneously — two regions, both active. Modeling this with booleans falls apart fast; xstate handles it cleanly.

---

## 7. Testing State Machines

Pure logic — perfect for unit tests. No DOM, no React.

```ts
import { describe, it, expect } from 'vitest';
import { createActor } from 'xstate';
import { checkoutMachine } from './checkout';

describe('checkout', () => {
  it('does not start with empty cart', () => {
    const actor = createActor(checkoutMachine).start();
    actor.send({ type: 'START', cart: [] });
    expect(actor.getSnapshot().value).toBe('idle'); // guard prevented transition
  });

  it('walks the happy path', () => {
    const actor = createActor(checkoutMachine).start();
    actor.send({ type: 'START', cart: [{ id: 'sku-1', qty: 1 }] });
    actor.send({ type: 'SUBMIT_SHIPPING', address: { /* ... */ } as any });
    actor.send({ type: 'SUBMIT_PAYMENT', payment: { /* ... */ } as any });
    expect(actor.getSnapshot().value).toBe('review');
  });
});
```

Mock the `submitOrder` actor for tests of the async transition:

```ts
const testMachine = checkoutMachine.provide({
  actors: {
    submitOrder: fromPromise(async () => ({ orderId: 'test-1' })),
  },
});
```

Real teams keep an `__tests__/checkout.test.ts` near the machine and skip rendering altogether for transition logic. Component tests (`@testing-library/react@15`, covered in `06.testing-perf/01-testing.md`) verify only the rendering layer.

---

## 8. Visualizing — Stately Studio

[stately.ai](https://stately.ai/editor) is the official xstate visual editor. You can:

- Paste your `setup({...}).createMachine({...})` code → see the diagram.
- Edit the diagram → it generates the code.
- Share read-only links with PM/design teams.

Real workflow: the engineer writes the first machine. PM reviews the diagram on stately.ai. They suggest "what if the user retries from error state?" — you add the transition, push the code, the visual updates.

The Inspector lets you see live transitions in development:

```ts
import { createBrowserInspector } from '@statelyai/inspect';

const inspector = createBrowserInspector(); // dev only

const actor = createActor(checkoutMachine, { inspect: inspector.inspect }).start();
```

Opens a panel showing live state changes as the user clicks through the UI.

---

## 9. The Reducer-First Counter-Argument

Several engineers (e.g., Mark Erikson, Redux maintainer) argue that for most React apps, **a discriminated union with `useReducer` is cheaper than xstate**. They're not wrong. The honest comparison:

| | `useReducer` + discriminated union | `xstate@5` |
|--|------------------------------------|------------|
| Bundle | 0 (built-in) | ~14 KB |
| Learning curve | Hours | Days |
| Async orchestration | Manual via `useEffect` | First-class via `actors` and `invoke` |
| Visual editor | None | stately.ai |
| Parallel regions | Manual | Built-in |
| Hierarchical states | Manual | Built-in |
| Testing in isolation | Easy (pure function) | Easy (`createActor`) |
| Replay / time travel | DIY | Built-in via inspector |

**Real picking rule:**
- 3–5 states, sequential, small async surface → `useReducer`.
- 6+ states, parallel/hierarchical, or async orchestration matters → `xstate@5`.

Don't reach for xstate by reflex. Don't avoid it by reflex either.

---

## 10. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `state.value` is hard to type-narrow | Use `state.matches('paid')` not `state.value === 'paid'` — `matches` narrows the context type |
| `assign` not updating | You returned a value from an action without using `assign(...)` — actions can't directly mutate context in v5 |
| Async transition not firing | The `actors` map wasn't passed to `setup({ actors })`, or the actor function isn't a Promise |
| Machine re-creates per render | `useMachine(machineRef)` — pass the machine **constant**, not a new instance |
| Sending events that don't transition | Check the event name and current state in DevTools / Inspector — events are silently ignored if no transition matches |
| Using `useActor` when you mean `useMachine` | `useActor` is for using a pre-existing actor; `useMachine` creates and runs one |
| Tutorial doesn't match your code | It's v4 — use the v5 docs at [stately.ai/docs/xstate](https://stately.ai/docs/xstate) |
| Invoke result not in `event.output` | v5 renamed `event.data` to `event.output` for actor results |

---

## 11. Decision Tree

```
Does this UI have multiple states that exclude each other?
│   YES → state-machine territory
│   NO  → useState

How many states? (yours, not your todo list)
│
├── 2–3 sequential, no async logic
│   → discriminated union + useReducer
│
├── 3–5 with one async branch (submit + retry)
│   → discriminated union + useReducer is still fine, but xstate isn't overkill
│
├── 5+ with branching, parallel regions, or async orchestration
│   → xstate@5
│
└── A truly global app store, not a per-component machine
    → Zustand (covered in 04.state-and-data/02-state-management.md), not xstate
```

---

## 12. What This Topic Connects To

- **`02.hooks/01-usestate.md`** and **`05.routing-and-styling/03-typescript-with-react.md`** — discriminated unions are the foundation that makes both `useReducer` and xstate type-safe.
- **`04.state-and-data/01-forms.md`** — `react-hook-form` is the form-state machine you don't have to write.
- **`04.state-and-data/03-data-fetching.md`** — TanStack Query is itself a state machine internally (`idle → fetching → success/error`).
- **`08.ecosystem/01-animation.md`** — drag-snap interactions pair perfectly with xstate for "what state are we in?" + Framer Motion for "how do we animate between them?".
- **`08.ecosystem/04-real-project.md`** — adding a guided onboarding to the Task Manager is a textbook xstate use case.

---

## Summary

| Tool | Use for |
|------|---------|
| `useState` | One value, no transitions |
| `useReducer` + discriminated union | 3–5 states, sequential, no parallel regions |
| `@xstate/store@2` | Lightweight typed store with transitions, no charts |
| `xstate@5` + `@xstate/react@5` | Multi-state machines, async orchestration, parallel/hierarchical states |
| Stately Studio | Visualizing, editing, sharing the machine |

| Rule | Why |
|------|-----|
| Make illegal states unrepresentable | TS + state machines together encode the rules once |
| Keep machines pure (no React imports) | Test in isolation, share with non-React surfaces (Node, mobile) |
| Pass the machine constant to `useMachine` | Re-creating per render breaks the actor lifecycle |
| Use `state.matches()` not `state.value === ...` | The first narrows the context type |
| Reach for xstate at 6+ states or async orchestration | Below that, `useReducer` is cheaper |

---

## Further reading

- [xstate 5 docs](https://stately.ai/docs/xstate)
- [`setup()` reference](https://stately.ai/docs/setup)
- [@xstate/react v5](https://stately.ai/docs/xstate-react)
- [Stately Studio (visual editor)](https://stately.ai/editor)
- [Stately Inspector](https://stately.ai/docs/inspector)
- [@xstate/store](https://stately.ai/docs/xstate-store)
- [David Khourshid — Statecharts: A Visual Formalism for Complex Systems](https://www.sciencedirect.com/science/article/pii/0167642387900359) (the academic paper xstate implements)
