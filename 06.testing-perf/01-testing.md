# Testing & Performance — 01. Testing: Vitest 1.6 + React Testing Library 15 vs Jest 29

> **What / Why / How** — test what users see, not how it's implemented. Pick Vitest for new projects; keep Jest if you have it.

---

## 1. The Three Testing Layers

| Layer | What it tests | Tool |
|-------|---------------|------|
| **Unit / component** | A single component or hook in isolation | `vitest@1.6` (or `jest@29`) + `@testing-library/react@15` |
| **Integration** | Multiple components working together with mocked network | Same tools + `msw@2.3` for fetch mocking |
| **End-to-end (E2E)** | Real browser, real backend (or test backend) | `@playwright/test@1.43` |

This file covers the first two. Playwright is its own topic later.

---

## 2. The Testing Philosophy — "Test Behavior, Not Implementation"

The Kent C. Dodds quote that defines React Testing Library:

> "The more your tests resemble the way your software is used, the more confidence they can give you."

Practical translation:

| Don't test | Test instead |
|------------|--------------|
| Internal state (`expect(component.state.count).toBe(1)`) | What the user sees (`expect(screen.getByText('1')).toBeVisible()`) |
| Whether `useState` was called | The behavior that emerges from state |
| CSS class names of internal nodes | Accessible roles + visible text |
| Props of a child component | Output the user observes |

This is why React Testing Library (RTL) **does not give you** access to `state`, `props`, or instance methods. The API forces you to query as a user would: by role, label, text.

---

## 3. Vitest 1.6 vs Jest 29 — Which to Pick

| | `vitest@1.6` | `jest@29` |
|--|--------------|-----------|
| First release | 2021 (Vue ecosystem origin, now framework-agnostic) | 2014 (Facebook) |
| Bundler | Uses Vite directly — same transform pipeline as your dev server | Uses `babel-jest` or `ts-jest` (separate config) |
| Cold start (~50 tests) | ~1.5s | ~5s |
| Watch-mode rerun | <100ms (HMR-driven) | ~1s |
| ESM support | Native | Experimental, frequent edge cases |
| API compatibility | Jest-compatible (`describe`, `it`, `expect`) — drop-in for most assertions | Native |
| Default in | Vite + React templates (`create-vite@5`), Nuxt 3, SvelteKit, Remix templates | Create React App (legacy), Next.js Pages Router examples |
| Mocking | `vi.mock`, `vi.fn`, `vi.spyOn` | `jest.mock`, `jest.fn`, `jest.spyOn` |

### Picking rule

- **New project, Vite-based** → `vitest@1.6`. Same config, faster feedback.
- **New project, Next.js 14** → either works. Vitest is gaining ground; Next.js has a `next/jest` preset that's still popular.
- **Existing Jest 29 codebase** → don't migrate without a reason. Jest is fine.
- **Existing Jest 27 or earlier** → upgrade to 29 first; only then consider Vitest.

The rest of this topic uses Vitest syntax. Jest syntax is identical except for `vi.*` ↔ `jest.*`.

---

## 4. Setting Up — Vite + React + Vitest 1.6 + RTL 15

```bash
npm i -D vitest@1.6 @testing-library/react@15 @testing-library/user-event@14 @testing-library/jest-dom@6.4 jsdom@24
```

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,        // describe/it/expect in scope without imports
    environment: 'jsdom', // simulates a browser DOM
    setupFiles: './src/test/setup.ts',
  },
});
```

```ts
// src/test/setup.ts
import '@testing-library/jest-dom/vitest'; // adds .toBeInTheDocument(), .toHaveClass(), etc.
```

```jsonc
// tsconfig.json — ensure types are visible
{
  "compilerOptions": {
    "types": ["vitest/globals", "@testing-library/jest-dom"]
  }
}
```

---

## 5. The First Real Test — A Counter Component

```tsx
// Counter.tsx
import { useState } from 'react';

export function Counter({ initial = 0 }: { initial?: number }) {
  const [count, setCount] = useState(initial);
  return (
    <div>
      <p data-testid="count">Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <button onClick={() => setCount(initial)}>Reset</button>
    </div>
  );
}
```

```tsx
// Counter.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Counter } from './Counter';

describe('Counter', () => {
  it('starts at 0', () => {
    render(<Counter />);
    expect(screen.getByText('Count: 0')).toBeInTheDocument();
  });

  it('increments when the user clicks Increment', async () => {
    const user = userEvent.setup();
    render(<Counter />);

    await user.click(screen.getByRole('button', { name: /increment/i }));
    await user.click(screen.getByRole('button', { name: /increment/i }));

    expect(screen.getByText('Count: 2')).toBeInTheDocument();
  });

  it('resets to the initial value', async () => {
    const user = userEvent.setup();
    render(<Counter initial={5} />);

    await user.click(screen.getByRole('button', { name: /increment/i }));
    await user.click(screen.getByRole('button', { name: /reset/i }));

    expect(screen.getByText('Count: 5')).toBeInTheDocument();
  });
});
```

Key points:
- `getByRole('button', { name: /increment/i })` is what an assistive technology would use. Robust against class-name and structural changes.
- `userEvent` (from `@testing-library/user-event@14`) simulates real interactions: it fires `pointerdown`, `mousedown`, `focus`, `click` in order, with realistic timing.
- `await user.click(...)` is required — `user-event@14` is async.
- `screen.getByText` queries from `document.body`; `render`'s container is also queryable but `screen` is cleaner.

---

## 6. The Query Hierarchy — Use the Right Selector

RTL's queries form a priority list. Use the highest you can:

| Priority | Query | Use for |
|----------|-------|---------|
| 1 (best) | `getByRole` | Buttons, links, headings, inputs — anything with an accessible role |
| 2 | `getByLabelText` | Form fields (label/input pairing) |
| 3 | `getByPlaceholderText` | Inputs without labels (legacy) |
| 4 | `getByText` | Static text content |
| 5 | `getByDisplayValue` | Form fields by current value |
| 6 | `getByAltText` | Images |
| 7 | `getByTitle` | Tooltips |
| 8 (worst) | `getByTestId` | Last resort — opt-in `data-testid` attributes |

**`getByTestId` is an escape hatch, not the default.** Tests that lean on it tend to break less, but they also fail to catch real accessibility regressions.

### Three variants per query: `getBy*`, `queryBy*`, `findBy*`

| Variant | Found one | Found none | Found multiple | Use when |
|---------|-----------|-----------|----------------|----------|
| `getBy*` | returns it | throws | throws | Element should be there now |
| `queryBy*` | returns it | returns `null` | throws | Asserting absence (`expect(...).toBeNull()`) |
| `findBy*` | returns it | retries 1s, then throws | throws | Element will appear async |
| `*All*` variants | array | empty array / null | array | Multiple matches expected |

```tsx
// Use findBy for async assertions — replaces waitFor in 80% of cases
expect(await screen.findByText('Welcome back')).toBeInTheDocument();
```

---

## 7. Mocking the Network — `msw@2.3` (Mock Service Worker)

Don't mock `fetch` directly. Use MSW to intercept network at the request level — your code under test runs unchanged.

```bash
npm i -D msw@2.3
```

```ts
// src/test/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users/:id', ({ params }) =>
    HttpResponse.json({ id: params.id, name: 'Anh', email: 'anh@example.com' })
  ),
  http.post('/api/login', async ({ request }) => {
    const body = await request.json() as { email: string; password: string };
    if (body.password === 'wrong') {
      return HttpResponse.json({ error: 'Invalid' }, { status: 401 });
    }
    return HttpResponse.json({ token: 'abc' });
  }),
];
```

```ts
// src/test/setup.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';
import { afterAll, afterEach, beforeAll } from 'vitest';

const server = setupServer(...handlers);

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

Test:

```tsx
import { render, screen } from '@testing-library/react';
import { UserCard } from './UserCard';

it('shows user info from /api/users/:id', async () => {
  render(<UserCard id="42" />);
  expect(await screen.findByText('Anh')).toBeInTheDocument();
});
```

`onUnhandledRequest: 'error'` is the right strict default — any test that hits an unmocked URL fails loudly instead of silently making real network calls.

### Per-test override

```ts
import { server } from './setup';
import { http, HttpResponse } from 'msw';

it('shows error on 500', async () => {
  server.use(
    http.get('/api/users/:id', () => HttpResponse.json({ error: 'boom' }, { status: 500 }))
  );
  render(<UserCard id="42" />);
  expect(await screen.findByRole('alert')).toHaveTextContent(/error/i);
});
```

---

## 8. Testing Hooks — `renderHook`

```tsx
import { renderHook, act } from '@testing-library/react';
import { useToggle } from './useToggle';

it('toggles', () => {
  const { result } = renderHook(() => useToggle(false));
  expect(result.current[0]).toBe(false);

  act(() => result.current[1]());
  expect(result.current[0]).toBe(true);
});
```

For hooks that need a Provider (e.g. TanStack Query), pass a `wrapper`:

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

it('useUser fetches a user', async () => {
  const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={qc}>{children}</QueryClientProvider>
  );

  const { result } = renderHook(() => useUser('42'), { wrapper });
  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data?.name).toBe('Anh');
});
```

---

## 9. Testing Forms — react-hook-form + Zod

A real test for a login form:

```tsx
it('shows validation errors and then logs in', async () => {
  const user = userEvent.setup();
  render(<LoginForm />);

  // Submit empty form
  await user.click(screen.getByRole('button', { name: /log in/i }));
  expect(await screen.findByText(/email is required/i)).toBeInTheDocument();

  // Fill it correctly
  await user.type(screen.getByLabelText(/email/i),    'anh@example.com');
  await user.type(screen.getByLabelText(/password/i), 'pw1234567');
  await user.click(screen.getByRole('button', { name: /log in/i }));

  // Submit hit the API → assert success state
  expect(await screen.findByText(/welcome back/i)).toBeInTheDocument();
});
```

The form's submit hits the MSW-mocked `/api/login`. Nothing about the test depends on `react-hook-form`'s internals.

---

## 10. Common Gotchas

### `act()` warnings — usually mean you forgot `await`

```tsx
// ❌ Throws "wrap in act(...)" warnings
fireEvent.click(button);
expect(screen.getByText('Loaded')).toBeInTheDocument();

// ✅ user-event v14 + findBy* handle act() internally
await user.click(button);
expect(await screen.findByText('Loaded')).toBeInTheDocument();
```

`user-event@14` and `findBy*` queries auto-wrap in `act`. If you still see warnings, it's almost always async state that the test moves past too early.

### Cleanup — automatic in Vitest + RTL 15

If you set `globals: true`, RTL's automatic cleanup runs after each test. No manual `afterEach(cleanup)` needed.

### `console.error` warnings should fail the test

```ts
// src/test/setup.ts
beforeEach(() => {
  vi.spyOn(console, 'error').mockImplementation((...args) => {
    throw new Error(args.join(' '));
  });
});
```

This catches React's `act`/key-warning/prop-validation errors that would otherwise be silent.

### Don't snapshot whole components

Snapshot tests on entire rendered components produce huge `.snap` files that nobody reviews. Prefer assertions on specific behavior. Snapshot small pure-data shapes (e.g. a transformer's output), not JSX trees.

---

## 11. Coverage and CI

### Coverage with Vitest

```bash
npm i -D @vitest/coverage-v8@1.6
```

```jsonc
// vitest.config.ts
test: {
  coverage: {
    provider: 'v8',
    reporter: ['text', 'html', 'lcov'],
    thresholds: { lines: 80, statements: 80, branches: 75, functions: 80 },
  },
}
```

CI command: `vitest run --coverage`. The `--run` form (or `vitest run`) is one-shot, the default is watch mode.

### Real CI baseline (GitHub Actions)

```yaml
- run: npm ci
- run: npm run test -- --run --coverage --reporter=verbose
- uses: codecov/codecov-action@v4
```

---

## 12. What Not to Test

- **External library internals** — don't test that `react-hook-form` calls `zodResolver` correctly. That's their team's job.
- **CSS visual appearance** — that's `@playwright/test` + visual snapshots, not RTL.
- **Server functions you don't own** — mock at the HTTP boundary with MSW; test those servers in their own repos.
- **Trivial getters/setters** — testing `function double(x) { return x*2 }` adds noise. Test interesting branches.

---

## Summary

| Tool | Pick when |
|------|-----------|
| `vitest@1.6` | New project, especially Vite-based |
| `jest@29` | Existing Jest codebase; don't migrate without reason |
| `@testing-library/react@15` | Always, for component/integration tests |
| `@testing-library/user-event@14` | Always, paired with RTL — simulates real user interactions |
| `msw@2.3` | Mock the network at the HTTP layer |
| `@playwright/test@1.43` | E2E in a real browser |

| Rule | Why |
|------|-----|
| Test behavior, not implementation | Tests survive refactors |
| Use `getByRole` + `getByLabelText` first | Catches accessibility regressions for free |
| Use `findBy*` for async expectations | Replaces 80% of `waitFor` usage |
| Mock at the network layer with MSW | Don't mock `fetch` directly |
| Treat `console.error` as a test failure | Catches React warnings that would otherwise be silent |
| Don't snapshot large JSX trees | Nobody reviews them; they rot |

---

## Further reading

- [Vitest docs](https://vitest.dev/)
- [Testing Library — Guiding Principles](https://testing-library.com/docs/guiding-principles)
- [Testing Library — About queries](https://testing-library.com/docs/queries/about/)
- [MSW — Getting started](https://mswjs.io/docs/getting-started)
- [Kent C. Dodds — Common mistakes with React Testing Library](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library)
