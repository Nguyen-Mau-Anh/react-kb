# Quality at Scale — 04. Storybook 8.3 + Chromatic: Component-Driven Development

> **What / Why / How** — Storybook 8.3 is the 2026 default for component-driven development. Pair with **Chromatic** for managed visual regression. Most teams misuse it as documentation; the real value is **isolated component states + visual diffs in CI**.

---

## 1. The Real Choices in 2026

| Tool | npm | Best for |
|------|-----|----------|
| **Storybook 8.3** | `storybook@8.3`, `@storybook/react-vite@8.3`, `@storybook/nextjs@8.3` | The 2026 default; deep React/Next.js integrations |
| **Ladle 5** | `@ladle/react@5` | Lightweight Storybook alternative; Vite-only; faster cold start |
| **Histoire 0.x** | `histoire@0.x` | Vue-first, React support; small community |
| **React Cosmos 7** | `react-cosmos@7` | Old-school component sandbox; small but loyal user base |
| **Chromatic** | `chromatic@11` (CLI) | The default visual regression / review tool; built by the Storybook team |
| **Percy** | `@percy/cli@1`, `@percy/storybook@5` | Visual regression alternative; BrowserStack-owned |
| **Applitools Eyes** | `@applitools/eyes-storybook@5` | Enterprise visual AI; pricey but smart diffing |
| **Storybook + Vercel** | `@vercel/style-guide` patterns | Free preview deploys per PR via Vercel |

**The 2026 default for any non-trivial React component library**: **Storybook 8.3 + Chromatic**. The combination is built by the same company (Chromatic acquired Storybook's main contributors in 2017 and has been the project's primary sponsor since).

For a tiny app where Storybook is overkill: **Ladle 5** is a faster, simpler alternative with mostly compatible CSF (Component Story Format).

---

## 2. Why Storybook Matters

The single insight: **components have many more states than your app exercises**. A button has hover, focus, loading, disabled, primary/secondary/destructive, icon-only/with-icon/text-only. Your dashboard renders maybe 3 of those combinations.

Storybook lets you build, review, and test all states **in isolation** — without navigating through your app to reach each one.

| Use case | Without Storybook | With Storybook |
|----------|-------------------|-----------------|
| Design a new button | Build it, render it on a page, click around | Open `Button.stories.tsx`, edit, see every state immediately |
| Hand off to designer | Send a PR link, ask them to navigate | Send a Storybook URL with the exact story |
| Test edge cases (empty, loading, error) | Mock the entire app to reach the state | One story per state |
| Visual regression CI | Screenshot the whole app per page | Screenshot every story in isolation |
| Onboard a new engineer | "Here's the codebase, click around" | "Here's Storybook — every component lives here" |

### When Storybook is overkill

- Single-app project with < 30 components.
- You don't have a design team needing visual review.
- The app is mostly app shell + 3rd-party UI primitives (shadcn/ui's blocks).

### When Storybook earns its keep

- Shared component library across multiple apps.
- Design system maintained by a separate team.
- Strong visual regression discipline in CI.
- Documentation and review-driven development.

For most production SaaS in 2026: **install Storybook for the component library, not for the application chrome**. Stories cover `<Button>`, `<Input>`, `<Card>`, `<DataTable>` — not `<DashboardPage>`.

---

## 3. Setup — Next.js 14 + Storybook 8.3

```bash
npx storybook@8.3 init
```

The wizard auto-detects Next.js, installs `@storybook/nextjs@8.3` (which handles `next/image`, `next/link`, `next/font`, App Router routing), and scaffolds:

```
.storybook/
├── main.ts                          ← framework config
└── preview.tsx                      ← global decorators (theme, fonts)
src/components/
└── Button.stories.tsx               ← example story
```

### `.storybook/main.ts` — production config

```ts
import type { StorybookConfig } from '@storybook/nextjs';

const config: StorybookConfig = {
  framework: '@storybook/nextjs',
  stories: ['../src/**/*.mdx', '../src/**/*.stories.@(js|jsx|ts|tsx)'],
  addons: [
    '@storybook/addon-essentials',                  // controls, actions, viewport, backgrounds, docs
    '@storybook/addon-interactions',                // play() functions for interactive testing
    '@storybook/addon-a11y',                        // axe-core inside Storybook
    '@chromatic-com/storybook',                     // visual review integration
  ],
  staticDirs: ['../public'],
  docs: { autodocs: 'tag' },
  typescript: {
    reactDocgen: 'react-docgen-typescript',         // typed prop docs
    check: true,
  },
};

export default config;
```

### `.storybook/preview.tsx` — global decorators

```tsx
import type { Preview, Decorator } from '@storybook/react';
import { Inter } from 'next/font/google';
import { ThemeProvider } from 'next-themes';
import '../src/app/globals.css';                   // your Tailwind base

const inter = Inter({ subsets: ['latin'] });

const withProviders: Decorator = (Story, context) => (
  <div className={inter.className} data-theme={context.globals.theme}>
    <ThemeProvider attribute="class" defaultTheme="light">
      <Story />
    </ThemeProvider>
  </div>
);

const preview: Preview = {
  parameters: {
    backgrounds: { default: 'light' },
    a11y: {
      config: { rules: [{ id: 'color-contrast', enabled: true }] },
    },
  },
  globalTypes: {
    theme: {
      description: 'Color theme',
      defaultValue: 'light',
      toolbar: {
        title: 'Theme',
        icon: 'circlehollow',
        items: [
          { value: 'light', title: 'Light' },
          { value: 'dark',  title: 'Dark' },
        ],
        dynamicTitle: true,
      },
    },
  },
  decorators: [withProviders],
};

export default preview;
```

The toolbar dropdown lets you flip every story into dark mode without touching individual stories. The `Inter` font + `globals.css` make stories look like the real app.

---

## 4. Writing a Story — CSF 3 Format

Storybook 8.3 uses **Component Story Format 3** — concise, typed, JSX-only.

```tsx
// src/components/Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta = {
  title: 'UI/Button',
  component: Button,
  tags: ['autodocs'],                              // generates a Docs page
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'destructive', 'ghost', 'link'],
    },
    size: { control: 'select', options: ['sm', 'md', 'lg'] },
    disabled: { control: 'boolean' },
  },
  args: { children: 'Save', onClick: () => alert('Clicked') },
} satisfies Meta<typeof Button>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Primary: Story = {
  args: { variant: 'primary' },
};

export const Secondary: Story = {
  args: { variant: 'secondary' },
};

export const Destructive: Story = {
  args: { variant: 'destructive', children: 'Delete' },
};

export const Loading: Story = {
  args: { variant: 'primary', children: 'Saving…', disabled: true },
};

export const IconOnly: Story = {
  args: { variant: 'ghost', size: 'sm', children: <TrashIcon /> },
};

export const InAForm: Story = {
  render: (args) => (
    <form onSubmit={(e) => e.preventDefault()}>
      <Button {...args} type="submit">Submit</Button>
    </form>
  ),
};
```

### Real patterns

- **`args`** at the meta level set defaults; per-story `args` override.
- **`argTypes`** drives the **Controls** addon (live editing of props in the UI).
- **`tags: ['autodocs']`** generates a Docs page from the component's prop types and stories.
- **Story name === named export**. `Primary` becomes "Primary" in the sidebar.
- **`render` function** when args alone aren't enough (composition, multiple components, layout).

---

## 5. Interaction Testing — `play()` Functions

Storybook 8.3 ships `@storybook/test` with built-in `expect`, `userEvent`, and `within` from Testing Library. Stories become integration tests.

```tsx
import type { Meta, StoryObj } from '@storybook/react';
import { expect, userEvent, within } from '@storybook/test';
import { LoginForm } from './LoginForm';

const meta: Meta<typeof LoginForm> = { component: LoginForm };
export default meta;
type Story = StoryObj<typeof meta>;

export const SubmitsCorrectly: Story = {
  play: async ({ canvasElement }) => {
    const c = within(canvasElement);
    await userEvent.type(c.getByLabel(/email/i), 'hi@example.com');
    await userEvent.type(c.getByLabel(/password/i), 'password123');
    await userEvent.click(c.getByRole('button', { name: /sign in/i }));
    await expect(c.getByRole('status')).toHaveTextContent(/welcome/i);
  },
};

export const ShowsValidationErrors: Story = {
  play: async ({ canvasElement }) => {
    const c = within(canvasElement);
    await userEvent.click(c.getByRole('button', { name: /sign in/i }));
    await expect(c.getByRole('alert')).toHaveTextContent(/email is required/i);
  },
};
```

What you get:
- **Visual** — the story is still a normal Storybook entry; you can manually inspect.
- **Replayable** — the Interactions addon shows each step; debug by stepping through.
- **CI-runnable** — `test-storybook@0.x` + `@storybook/test-runner@0.x` runs every story's `play()` in headless Chromium and reports pass/fail.

```bash
npm i -D @storybook/test-runner@0.20
```

```jsonc
// package.json
{
  "scripts": {
    "test-storybook": "test-storybook --url http://localhost:6006"
  }
}
```

```bash
# Run all stories' play() functions in CI
npm run storybook -- --ci &
npx wait-on http://localhost:6006
npm run test-storybook
```

This catches "the component's loading state was broken for 2 weeks" bugs that no E2E test would notice.

---

## 6. Mocking Server-Only Dependencies

Storybook runs in the browser — `next/headers`, `next/navigation`, server actions, Prisma all need mocks.

### `next/navigation` mocks (Storybook 8.3 + `@storybook/nextjs`)

The `@storybook/nextjs@8.3` framework auto-mocks `useRouter`, `useSearchParams`, `usePathname`, `useParams`. Per-story override:

```tsx
import { Meta, StoryObj } from '@storybook/react';
import { TaskRow } from './TaskRow';

const meta = {
  component: TaskRow,
  parameters: {
    nextjs: {
      appDirectory: true,
      navigation: {
        pathname: '/dashboard/projects/abc',
        query: { tab: 'overview' },
      },
    },
  },
} satisfies Meta<typeof TaskRow>;
export default meta;
```

### MSW for fetch / API mocks

`msw-storybook-addon@2.0` (covered in `06.testing-perf/01-testing.md`) makes MSW handlers per-story:

```tsx
import { http, HttpResponse } from 'msw';
import type { Meta, StoryObj } from '@storybook/react';
import { ProjectList } from './ProjectList';

const meta: Meta<typeof ProjectList> = { component: ProjectList };
export default meta;
type Story = StoryObj<typeof meta>;

export const WithProjects: Story = {
  parameters: {
    msw: {
      handlers: [
        http.get('/api/projects', () => HttpResponse.json([
          { id: '1', name: 'Acme website' },
          { id: '2', name: 'Internal tooling' },
        ])),
      ],
    },
  },
};

export const Empty: Story = {
  parameters: {
    msw: { handlers: [http.get('/api/projects', () => HttpResponse.json([]))] },
  },
};

export const Loading: Story = {
  parameters: {
    msw: { handlers: [http.get('/api/projects', () => new Promise(() => {}))] }, // never resolves
  },
};

export const Error: Story = {
  parameters: {
    msw: { handlers: [http.get('/api/projects', () => HttpResponse.json({ error: 'down' }, { status: 500 }))] },
  },
};
```

Four states. Four stories. **Every state of the component is reviewable, screenshottable, and testable** — without needing the backend at all.

### Server Actions in Storybook

Server Actions (covered in `07.nextjs/03-data-fetching.md`) don't run in the browser. Mock them at the import boundary:

```tsx
// stories/Project.stories.tsx
import { Meta } from '@storybook/react';
import { fn } from '@storybook/test';
import { ProjectForm } from './ProjectForm';

// Replace the Server Action import with a Storybook mock
import * as actions from '@/app/actions';
vi.mock?.('@/app/actions', () => ({
  createProject: fn().mockResolvedValue({ id: 'new', name: 'New' }),
}));
```

Or use Storybook 8.3's first-class **mocked imports** via the `mockImport` config in `main.ts`:

```ts
// .storybook/main.ts
const config: StorybookConfig = {
  // ...
  refs: {},
  features: { experimentalRSC: true },
  webpackFinal: (config) => {
    // alias Server Action imports to mocks
    config.resolve!.alias = {
      ...config.resolve!.alias,
      '@/app/actions$': require.resolve('./mocks/actions.ts'),
    };
    return config;
  },
};
```

The 2026 Storybook 8.3 RSC support is still experimental. For Server Components specifically, **don't try to render them in Storybook** — keep stories scoped to Client Components and pass mock data as props.

---

## 7. Chromatic — Visual Regression and Review

Chromatic is the production-grade visual-regression layer on top of Storybook. Built by the Storybook team.

### What Chromatic does

- Runs every story in headless Chrome on push.
- Captures pixel-perfect screenshots.
- Compares against the baseline (the previous commit on the target branch).
- Highlights pixel diffs for review in a dedicated UI.
- Approval workflow: designer or PM reviews diffs, approves or rejects.
- Cross-browser captures (Chrome, Safari, Firefox) on paid plans.
- Per-DPR (device pixel ratio) captures.

### Setup

```bash
npx chromatic --project-token=<your-token> --build-script-name=build-storybook
```

```jsonc
// package.json
{
  "scripts": {
    "build-storybook": "storybook build",
    "chromatic": "chromatic --project-token=$CHROMATIC_PROJECT_TOKEN"
  }
}
```

### CI integration

```yaml
# .github/workflows/chromatic.yml
name: Chromatic
on: push
jobs:
  chromatic:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }                    # required for baseline detection
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - uses: chromaui/action@v11
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          exitOnceUploaded: true                     # don't block PR on review
          onlyChanged: true                          # speeds up runs by ~2-10x via TurboSnap
```

`onlyChanged: true` enables **TurboSnap** — Chromatic analyzes the git diff and only re-screenshots stories whose dependencies changed. A 500-story library with one button tweak runs in ~30 seconds instead of 5 minutes.

### Why Chromatic over DIY Playwright snapshots

| | Playwright `toHaveScreenshot` | Chromatic |
|--|-------------------------------|-----------|
| Setup complexity | Free, in your existing test suite | Separate service, $$ |
| Cross-browser | Yes (you configure projects) | Yes (paid plans) |
| Cross-DPR | Manual | Built in |
| Baseline management | Git-tracked snapshot files | Cloud baselines per branch |
| Review UI | Diff in HTML report | Side-by-side, click to approve/reject |
| Designer/PM access | Engineers only | Anyone with the link |
| Flake handling | DIY (mask, disable animations) | Built-in smart diffs |
| Pricing | Free | Free 5,000 snapshots/mo, $149+/mo |

Chromatic wins on **review workflow**. Playwright snapshots are fine for "did the engineer break this on purpose"; Chromatic is the right tool when designers are part of the review loop.

For solo / small teams: Playwright `toHaveScreenshot` (covered in `12.quality-at-scale/03-e2e-playwright.md`) is enough. For teams with dedicated design review: Chromatic earns its price tag.

### Real workflow

1. Engineer pushes a PR.
2. Chromatic runs on push, comments on the PR with a link to the review UI.
3. UI shows: "8 stories changed — 3 visual changes detected."
4. Designer opens the link, sees side-by-side diffs, approves the intentional ones, rejects the surprises.
5. PR is mergeable only when Chromatic check is green.

This is the **closed-loop design-review workflow** that production design systems run.

---

## 8. Storybook a11y Addon

`@storybook/addon-a11y@8.3` runs `axe-core` inside Storybook, lighting up violations per story:

```tsx
// .storybook/preview.tsx
const preview: Preview = {
  parameters: {
    a11y: {
      config: {
        rules: [
          { id: 'color-contrast', enabled: true },
          { id: 'autocomplete-valid', enabled: true },
        ],
      },
    },
  },
};
```

Each story's accessibility tab shows:
- ✅ Passes (green)
- ⚠ Incomplete (yellow — may be a false positive)
- ❌ Violations (red)

Combined with **per-component `jest-axe`** in Vitest (covered in `11.capabilities/05-accessibility.md`), this is the production-grade pre-merge a11y check.

For CI gating, the test-runner runs axe on every story:

```ts
// .storybook/test-runner.ts
import type { TestRunnerConfig } from '@storybook/test-runner';
import { getStoryContext } from '@storybook/test-runner';
import { injectAxe, checkA11y } from 'axe-playwright';

const config: TestRunnerConfig = {
  async preVisit(page) { await injectAxe(page); },
  async postVisit(page, context) {
    const ctx = await getStoryContext(page, context);
    if (ctx.parameters?.a11y?.disable) return;
    await checkA11y(page, '#storybook-root', {
      detailedReport: true,
      detailedReportOptions: { html: true },
    });
  },
};

export default config;
```

Now `npm run test-storybook` fails CI if any story has a11y violations.

---

## 9. Real Component-Library Patterns

### Pattern A — Stories live next to components

```
src/components/
├── Button.tsx
├── Button.stories.tsx
├── Input.tsx
├── Input.stories.tsx
└── Modal/
    ├── Modal.tsx
    ├── Modal.stories.tsx
    └── ConfirmDialog.tsx              ← composed; its own stories
```

Every story file has the same name + `.stories.tsx`. Easier to maintain than a separate `stories/` folder.

### Pattern B — Three-tier story taxonomy

```
title: 'Foundations/Tokens/Colors'        ← design tokens, not React components
title: 'UI/Button'                          ← primitives
title: 'Patterns/PageHeader'                ← composed app-level patterns
title: 'Pages/Dashboard/Default'            ← whole-page screenshots (sparingly)
```

Foundations + UI live in the shared design system. Patterns + Pages live in the app. Mix as your monorepo grows (covered in `08.ecosystem/03-monorepo.md`).

### Pattern C — One story per realistic state

For a `<Button>`:
- `Primary`, `Secondary`, `Destructive`, `Ghost`, `Link` (variants)
- `Small`, `Medium`, `Large` (sizes)
- `Loading`, `Disabled`, `WithIcon`, `IconOnly` (modifiers)
- `LongText` (overflow check)

Don't write a story for every cartesian product. **Curate to the states real users see.** A 30-story Button is better than 3 stories that hide regressions in the un-storied states.

### Pattern D — `<Story>` composition for design-system docs

```mdx
{/* src/components/Button.mdx */}
import { Meta, Story } from '@storybook/blocks';
import * as ButtonStories from './Button.stories';

<Meta of={ButtonStories} />

# Button

Use buttons to trigger actions. Prefer `Primary` for the main action on a page; `Secondary` for non-destructive alternates; `Destructive` for irreversible actions.

## Variants
<Story of={ButtonStories.Primary} />
<Story of={ButtonStories.Secondary} />
<Story of={ButtonStories.Destructive} />

## When to use
- One primary button per major UI region.
- Destructive variant requires a confirmation dialog.
- Avoid icon-only buttons without `aria-label`.

## When NOT to use
- For navigation between pages → use `<Link>` instead.
- For toggling a value → use `<Switch>` or `<Checkbox>`.
```

The MDX file is your **design-system documentation**, with live components from the same stories tested in CI. Single source of truth.

---

## 10. Deploy Storybook — Vercel + Per-PR Previews

Storybook is a static site. Deploy to:

| Host | Best for |
|------|----------|
| **Vercel** | Most teams; per-PR preview URLs from a separate Vercel project |
| **Chromatic** | Built-in hosted Storybook on every Chromatic build |
| **GitHub Pages** | Public OSS libraries |
| **Netlify** | Same idea as Vercel |
| **Cloudflare Pages** | Cheapest at scale |

For Vercel: create a separate project pointing at `npm run build-storybook` with output dir `storybook-static/`. Every PR gets a preview URL like `storybook-pr-42.vercel.app`. Designers and PMs review against the deployed Storybook, not the dev server.

---

## 11. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Storybook builds 5x slower than the app | Lazy-load heavy stories with `loaders`; disable Webpack source maps in dev |
| Stories pass locally, fail in CI | Run via Docker: `mcr.microsoft.com/playwright:v1.46.0-jammy` matches CI |
| `next/font` doesn't load | Use `@storybook/nextjs@8.3` framework; load fonts in `preview.tsx` |
| Tailwind classes don't apply | Import `globals.css` in `preview.tsx`; verify `content` glob covers `.stories.tsx` |
| Server Component in a story errors | Don't render Server Components in Storybook; pass mock data as props |
| Chromatic snapshots fail every run | Disable animations globally in `preview.tsx`: `chromatic: { delay: 300 }` or `disableAnimations: true` |
| TurboSnap not detecting changes | `fetch-depth: 0` on checkout; ensure imports are static |
| MSW handlers leak between stories | Each story's `parameters.msw.handlers` replaces the previous; that's correct |
| `play()` works in UI but fails in test-runner | Use `@storybook/test`, not `@testing-library/user-event` directly |
| Story sidebar gets cluttered | Use namespaced titles (`UI/Forms/Input`); group small stories in subfolders |
| Decorators apply only to some stories | Hierarchy: `parameters` > `decorators` > globals; debug with the Toolbar |
| Changes to mocks don't trigger Chromatic re-run | Mocks must be in the same git tree as stories; TurboSnap follows imports |
| Designer can't access Chromatic | Add their email to the Chromatic project in the dashboard |
| Component crashes Storybook → blank screen | Wrap with `<ErrorBoundary>` in a global decorator |

---

## 12. Decision Tree

```
Are you building a component library or a single app?
│
├── Component library shared across apps → Storybook is essential
├── Design system maintained by a separate team → Storybook is essential
└── Single app with < 30 components → Storybook is optional; consider Ladle 5

Visual regression?
│
├── Solo / small team → Playwright toHaveScreenshot (free)
├── Mid-size team with design review → Chromatic
└── Enterprise with AI-driven smart diffing → Applitools Eyes

Hosting Storybook?
│
├── Already on Vercel → separate Vercel project
├── Want bundled with Chromatic → Chromatic hosts every build
└── OSS / public library → GitHub Pages

CI gates?
│
├── Visual regression → Chromatic (best DX) or Playwright snapshots
├── Interaction → @storybook/test-runner with play() functions
├── Accessibility → axe via test-runner postVisit hook
└── All three → run together as part of the storybook job
```

---

## 13. What This Topic Connects To

- **`08.ecosystem/02-ui-libraries.md`** — Storybook is the natural home for shadcn/ui customizations.
- **`08.ecosystem/03-monorepo.md`** — Shared `packages/ui` ships its own Storybook.
- **`11.capabilities/05-accessibility.md`** — `@storybook/addon-a11y` wires axe per story.
- **`12.quality-at-scale/03-e2e-playwright.md`** — Playwright covers app flows; Storybook covers component states.
- **`12.quality-at-scale/05-design-system.md`** — Storybook is the design system's UI (next topic).
- **`07.nextjs/01-fundamentals.md`** — `@storybook/nextjs@8.3` integrates `next/image`, `next/font`, App Router.

---

## 14. Summary

| Tool | Pick when |
|------|-----------|
| **Storybook 8.3** + `@storybook/nextjs@8.3` | Default for any non-trivial component library in 2026 |
| **Ladle 5** | Lightweight Storybook alternative, Vite-only |
| **Chromatic** | Production-grade visual regression with designer review |
| **Percy / Applitools** | Alternatives if pricing or features differ |
| **`@storybook/test-runner@0.20`** | CI runner for `play()` functions |
| **`@storybook/addon-a11y`** | axe inside Storybook per story |
| **MSW + `msw-storybook-addon@2`** | Network mocks per story |

| Rule | Why |
|------|-----|
| Stories cover real component states, not cartesian product | Curated > exhaustive |
| Co-locate stories with components (`Button.stories.tsx`) | Keeps them in sync |
| Use `@storybook/test`'s `expect`, `userEvent`, `within` | Bundled, browser-tested |
| Mock at the network layer with MSW per-story | Reusable across unit, story, and Playwright tests |
| Run Chromatic with `onlyChanged: true` (TurboSnap) | 2–10× faster builds |
| Don't render Server Components in Storybook | Pass mock data as props instead |
| Disable animations in CI snapshots | Eliminates 95% of visual flake |

---

## Further reading

- [Storybook docs](https://storybook.js.org/docs)
- [Storybook for Next.js](https://storybook.js.org/docs/get-started/frameworks/nextjs)
- [Storybook Test addon (`@storybook/test`)](https://storybook.js.org/docs/writing-tests/component-testing)
- [Chromatic docs](https://www.chromatic.com/docs/)
- [Chromatic TurboSnap](https://www.chromatic.com/docs/turbosnap/)
- [Storybook a11y addon](https://storybook.js.org/addons/@storybook/addon-a11y)
- [Component Driven UI (Tom Coleman)](https://www.componentdriven.org/) — the canonical philosophy
- [Ladle docs](https://ladle.dev/) — lightweight alternative
