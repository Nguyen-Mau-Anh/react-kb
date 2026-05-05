# Quality at Scale — 03. E2E with Playwright 1.46: Fixtures, Page Objects, Visual Regression, Sharding

> **What / Why / How** — `@playwright/test@1.46` is the 2026 default E2E framework. Skip Cypress for new projects. Build with **fixtures over page objects**, run **trace + video on retry**, **shard in CI** for parallelism, and add **visual regression** only on stable pages.

---

## 1. The Real Choices in 2026

| Tool | npm | Status | Best for |
|------|-----|--------|----------|
| **Playwright 1.46** | `@playwright/test@1.46` | The 2026 default | Cross-browser E2E, component tests, visual regression, API tests |
| **Cypress 13** | `cypress@13` | Active but losing momentum | Existing Cypress codebases; team that prefers its DX |
| **WebdriverIO 9** | `webdriverio@9`, `@wdio/cli@9` | Active | Multi-protocol (WebDriver, native mobile via Appium) |
| **TestCafé 3** | `testcafe@3` | Maintenance mode | Don't pick for new code |
| **Selenium 4** | `selenium-webdriver@4` | Legacy | Existing Selenium suites only |
| **Puppeteer 23** | `puppeteer@23` | Active | Scraping / programmatic browser; not a test framework |

**The 2026 default is Playwright 1.46.** Microsoft maintains it; it has stronger cross-browser coverage than Cypress (Chromium + WebKit + Firefox + branded Edge / Chrome), real parallelism, native API testing, component testing, visual regression, and codegen — all in one tool.

Cypress 13 is fine if you have an existing suite. The 2024 acquisition by BrowserStack hasn't shaken the project, but feature velocity has slowed compared to Playwright. **Don't migrate** an existing Cypress suite without a reason; **don't pick Cypress for new code**.

---

## 2. Setup — Next.js 14 + Playwright 1.46

```bash
npm init playwright@latest
```

The init wizard generates `playwright.config.ts`, `tests/example.spec.ts`, and a GitHub Actions workflow. Pick **TypeScript**, **`tests/`** as the directory, and let it install browsers.

### `playwright.config.ts` — the production config

```ts
import { defineConfig, devices } from '@playwright/test';

const isCI = !!process.env.CI;
const PORT = process.env.PORT ?? '3000';
const BASE = process.env.PLAYWRIGHT_BASE_URL ?? `http://localhost:${PORT}`;

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,                          // tests within a file run in parallel
  forbidOnly: isCI,                             // .only fails the build in CI
  retries: isCI ? 2 : 0,                        // flake-tolerance only in CI
  workers: isCI ? 4 : undefined,                // 4 parallel browsers in CI; auto-detect locally
  reporter: isCI ? [['html'], ['github'], ['blob']] : 'html',

  use: {
    baseURL: BASE,
    trace: 'on-first-retry',                    // record trace only when retrying
    video: 'retain-on-failure',                  // keep video only for failed tests
    screenshot: 'only-on-failure',
    actionTimeout: 10_000,
    navigationTimeout: 30_000,
  },

  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'webkit',   use: { ...devices['Desktop Safari'] } },
    { name: 'firefox',  use: { ...devices['Desktop Firefox'] } },
    { name: 'mobile-chrome', use: { ...devices['Pixel 7'] } },
    { name: 'mobile-safari', use: { ...devices['iPhone 14'] } },
  ],

  webServer: {
    command: 'npm run start',
    url: BASE,
    reuseExistingServer: !isCI,
    timeout: 120_000,
  },
});
```

### Key settings explained

- **`fullyParallel: true`** — tests within a single file run on separate workers. Massive speedup for any suite > 5 tests per file.
- **`retries: 2` in CI only** — most flakes are timing-related; auto-retry catches them. Locally, retries hide real bugs.
- **`trace: 'on-first-retry'`** — records the *full DOM history* of a test that failed and retried. Open with `npx playwright show-trace trace.zip`. **The single most valuable Playwright feature.**
- **`workers: 4`** — 4 parallel browser contexts. Tune to your CI machine's RAM/CPU; on a Vercel / GitHub free runner, 2–4 is the sweet spot.
- **`webServer`** — Playwright spawns your Next.js prod build before tests run. `reuseExistingServer: true` locally lets you run `npm run dev` once and re-run tests fast.

### Browsers and projects

The `projects` array runs every test against every project unless filtered. **For PR runs, use `--project=chromium`** to keep CI fast. **For nightly runs, use all five** to catch WebKit/Firefox-specific issues.

---

## 3. Locators — Test Like a User

The Playwright opinion: **locate by role, accessible name, and text — never by CSS selectors except for last-resort cases.**

```ts
// ✅ User-facing — survives refactors
await page.getByRole('button', { name: 'Save' }).click();
await page.getByLabel('Email').fill('hi@example.com');
await page.getByText('Welcome back').waitFor();
await page.getByPlaceholder('Search…').fill('react');
await page.getByTitle('Close').click();

// ⚠ Fallback when no semantic option exists
await page.getByTestId('save-button').click();    // requires data-testid="save-button"

// ❌ Brittle — breaks on every refactor
await page.locator('div.btn.btn-primary > span').click();
```

Why: `getByRole('button', { name: 'Save' })` describes the user's intent ("click the Save button"). It survives:
- Tag changes (`<button>` → `<a role="button">`).
- Class renames.
- DOM restructuring.
- Adding wrappers.

The CSS selector `div.btn.btn-primary > span` breaks the moment any of those happens.

### `data-testid` is an escape hatch, not the default

Use it when:
- A button has no accessible name (icon-only, no `aria-label`). Better fix: add the label.
- Two elements have identical text (multiple "Save" buttons). Better fix: distinguish via parent `getByRole('region', { name: 'Settings' }).getByRole('button', { name: 'Save' })`.

If you find yourself writing `getByTestId` everywhere, your app's accessibility is probably weak (covered in `11.capabilities/05-accessibility.md`).

### Auto-waiting is built in

```ts
await page.getByRole('button', { name: 'Save' }).click();
//   ^— Playwright waits up to actionTimeout for the button to be:
//      - attached to DOM
//      - visible
//      - enabled (not disabled)
//      - stable (not animating)
//      - able to receive events (not covered by another element)
```

You almost never need `waitFor`. The exception: explicit waits for application state changes:

```ts
await page.getByRole('status').filter({ hasText: 'Saved' }).waitFor();
```

---

## 4. Fixtures — The Right Abstraction (Not Page Objects)

The 2010s Page Object Model (POM) was great for Selenium/Java. Playwright's **fixtures** are strictly better for TS/Node and what the team officially recommends now.

### What fixtures are

A fixture is a value/setup that's lazily created per test, automatically wired to test parameters, and torn down after. Built-ins: `page`, `context`, `browser`, `request`. Custom ones extend the framework cleanly.

### Real example — login fixture

```ts
// tests/fixtures.ts
import { test as base, expect } from '@playwright/test';

type Fixtures = {
  loggedInPage: import('@playwright/test').Page;
  // Future fixtures: dbSeed, mockedStripe, featureFlag, etc.
};

export const test = base.extend<Fixtures>({
  loggedInPage: async ({ page }, use) => {
    // Setup: log in
    await page.goto('/login');
    await page.getByLabel('Email').fill(process.env.TEST_EMAIL!);
    await page.getByLabel('Password').fill(process.env.TEST_PASSWORD!);
    await page.getByRole('button', { name: 'Sign in' }).click();
    await page.getByRole('heading', { name: 'Dashboard' }).waitFor();

    // Hand the logged-in page to the test
    await use(page);

    // Teardown (after test): nothing — page closes automatically
  },
});

export { expect };
```

### Use in a test file

```ts
// tests/dashboard.spec.ts
import { test, expect } from './fixtures';

test('user can create a project', async ({ loggedInPage }) => {
  await loggedInPage.getByRole('button', { name: 'New project' }).click();
  await loggedInPage.getByLabel('Name').fill('Test project');
  await loggedInPage.getByRole('button', { name: 'Create' }).click();

  await expect(loggedInPage.getByRole('heading', { name: 'Test project' })).toBeVisible();
});
```

The login boilerplate is gone. Every test that needs auth declares `loggedInPage` in its parameter list and Playwright provides a logged-in browser.

### Storage state — share auth across tests

For dozens of tests that all need login, run login once and reuse the cookies/localStorage:

```ts
// playwright.config.ts
projects: [
  {
    name: 'setup',
    testMatch: /global\.setup\.ts/,
  },
  {
    name: 'chromium',
    use: {
      ...devices['Desktop Chrome'],
      storageState: 'playwright/.auth/user.json',
    },
    dependencies: ['setup'],
  },
],
```

```ts
// tests/global.setup.ts
import { test as setup } from '@playwright/test';

const authFile = 'playwright/.auth/user.json';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.TEST_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.getByRole('heading', { name: 'Dashboard' }).waitFor();
  await page.context().storageState({ path: authFile });
});
```

Run order: `setup` runs first → saves cookies + localStorage to `user.json` → all `chromium` tests start logged in. **One login per CI run, not per test.** Add `playwright/.auth/` to `.gitignore`.

For multi-role tests (admin, member, guest), repeat with separate setup files writing separate state files. Reference whichever you need per test.

---

## 5. Network — Mock at the Right Layer

### Mock external APIs with `page.route`

```ts
test('shows error when Stripe is down', async ({ page }) => {
  await page.route('**/api/checkout', (route) =>
    route.fulfill({ status: 503, body: 'Stripe is down' })
  );

  await page.goto('/checkout');
  await page.getByRole('button', { name: 'Pay' }).click();
  await expect(page.getByRole('alert')).toContainText('Try again later');
});
```

`page.route` intercepts requests before they leave the browser. Use for:
- **Failure-mode testing** (5xx responses, slow responses).
- **Third-party isolation** (don't actually charge a real Stripe card; covered in `11.capabilities/01-payments.md`).
- **Faster tests** when the real API is slow.

### Don't mock what your app already mocks

If you already use **MSW 2.3** (covered in `06.testing-perf/01-testing.md`) for component tests, **don't duplicate** it in Playwright. Either:
- Run the dev server with MSW enabled and let Playwright hit it.
- Use Playwright's `page.route` for E2E mocks.

Pick one layer per scenario; don't double-mock.

### Real example — pre-seed test data via API instead of clicking

```ts
test('user can edit a project', async ({ loggedInPage, request }) => {
  // Setup data via API, not the UI
  const project = await request.post('/api/projects', {
    data: { name: 'Test project' },
  }).then((r) => r.json());

  await loggedInPage.goto(`/project/${project.id}`);
  await loggedInPage.getByRole('button', { name: 'Edit' }).click();
  // ...
});
```

Don't drive the UI to set up state. Use the API or DB directly. Tests are faster, less flaky, and isolate the actual feature being tested.

---

## 6. Assertions — Web-First and Auto-Retrying

Playwright's `expect` is **auto-retrying**. It re-checks the assertion until it passes or times out — no manual waits needed.

```ts
// All of these auto-retry until the condition is true (or timeout):
await expect(page.getByRole('heading')).toHaveText('Dashboard');
await expect(page.getByRole('button')).toBeEnabled();
await expect(page.getByLabel('Email')).toHaveValue('hi@example.com');
await expect(page.getByRole('list')).toContainText(['Item 1', 'Item 2']);
await expect(page).toHaveURL(/\/dashboard/);
await expect(page).toHaveTitle('Dashboard | Acme');
await expect(page.locator('.spinner')).not.toBeVisible();
```

**Don't** mix Jest-style assertions:
```ts
// ❌ Doesn't auto-retry; flake-prone
expect(await page.getByRole('button').isEnabled()).toBe(true);

// ✅ Auto-retries up to actionTimeout
await expect(page.getByRole('button')).toBeEnabled();
```

The `await expect(locator).matcher()` pattern is the single most-flake-reducing rule in Playwright tests.

---

## 7. Component Testing (Optional)

Playwright 1.46 ships **component testing** — render React components in a real browser, test like an E2E test, but scoped to one component.

```bash
npm init playwright@latest -- --ct
```

```ts
// src/components/Button.spec.tsx
import { test, expect } from '@playwright/experimental-ct-react';
import { Button } from './Button';

test('Button calls onClick', async ({ mount }) => {
  let clicked = false;
  const component = await mount(<Button onClick={() => { clicked = true; }}>Click</Button>);
  await component.click();
  expect(clicked).toBe(true);
});
```

### When component tests in Playwright beat Vitest + RTL

- You're already running Playwright for E2E; one tool for both.
- The component depends on **real browser APIs** that JSDOM mocks badly (Canvas, IntersectionObserver edge cases, advanced CSS like container queries).
- You want **visual regression** on the component (covered next).

### When Vitest + RTL still wins

- **Speed** — JSDOM tests are ~10× faster than browser tests. For pure-logic component checks, Vitest is faster.
- **Larger ecosystem** — `@testing-library/react@15` patterns, `@testing-library/user-event@14`, and `jest-axe@9` are battle-tested with Vitest (covered in `06.testing-perf/01-testing.md` and `11.capabilities/05-accessibility.md`).

**Real production split**: Vitest for unit + integration component tests; Playwright for E2E + a small set of "needs a real browser" component tests.

---

## 8. Visual Regression — `toHaveScreenshot`

Playwright's built-in screenshot diffing catches unintended visual changes.

```ts
test('homepage looks correct', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('home.png', {
    maxDiffPixels: 100,                          // tolerate small antialiasing differences
    fullPage: true,
  });
});
```

First run: snapshot is saved. Subsequent runs: page is compared pixel-by-pixel. Failures show in the HTML report with **side-by-side + diff** views.

### When pixel-diff catches real bugs

- Tailwind class typo that subtly changes spacing.
- A new CSS rule that affects unrelated pages.
- Font loading regression that shifts layout.
- Image not loading on production (placeholder shown instead).

### When pixel-diff is a flake factory

- **Animations** running differently per CPU.
- **Webfonts** loading at different speeds.
- **Date/time** in the UI changing every test run.
- **Real network calls** with variable response times.

### Real production rules

```ts
test('checkout page', async ({ page }) => {
  await page.goto('/checkout');

  // Mask volatile content
  await expect(page).toHaveScreenshot({
    mask: [
      page.getByTestId('order-id'),
      page.getByTestId('current-time'),
      page.getByRole('img', { name: 'User avatar' }),
    ],
    animations: 'disabled',
    caret: 'hide',
  });
});
```

- **`mask`** blacks out volatile selectors before diffing.
- **`animations: 'disabled'`** stops CSS animations and pauses Web Animations.
- **`caret: 'hide'`** hides the text-input cursor (which blinks per OS settings).

These three together remove ~95% of flake. The remaining 5% is fonts — set `font-display: block` for tested pages or pre-warm fonts in your fixture.

### Run by platform — screenshots are platform-specific

Pixel rendering differs per OS. Run snapshots on **the same platform** (usually Linux in CI). Update snapshots:

```bash
# Locally on the target platform
npx playwright test --update-snapshots --project=chromium

# Or via Docker to match CI exactly
docker run -it --rm -v $(pwd):/app mcr.microsoft.com/playwright:v1.46.0-jammy bash -c "npx playwright test --update-snapshots"
```

For real production visual regression, use **Chromatic** (covered in the next topic) instead of Playwright's built-in. Chromatic handles cross-browser, cross-DPR, baselines per branch, and review UI better than DIY Playwright snapshots.

---

## 9. Accessibility in E2E

Pair with `@axe-core/playwright@4` (covered in `11.capabilities/05-accessibility.md`):

```ts
import AxeBuilder from '@axe-core/playwright';

test('dashboard has no a11y violations', async ({ loggedInPage }) => {
  await loggedInPage.goto('/dashboard');
  const results = await new AxeBuilder({ page: loggedInPage })
    .disableRules(['color-contrast'])              // already covered by Storybook
    .analyze();
  expect(results.violations).toEqual([]);
});
```

Run on every critical page. Combined with per-component `jest-axe@9` (Vitest), this is the production-grade a11y CI gate.

---

## 10. CI Sharding — Real Speed Wins

Even with `workers: 4`, a 200-test suite can take 10+ minutes. **Sharding** splits the suite across multiple machines that run in parallel.

### GitHub Actions example

```yaml
# .github/workflows/e2e.yml
name: E2E
on: [pull_request]
jobs:
  test:
    timeout-minutes: 20
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1/4, 2/4, 3/4, 4/4]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec playwright install --with-deps chromium
      - run: pnpm exec playwright test --project=chromium --shard=${{ matrix.shard }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: blob-report-${{ matrix.shard }}
          path: blob-report
          retention-days: 14

  merge:
    if: always()
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - uses: actions/download-artifact@v4
        with: { path: all-blob-reports, pattern: blob-report-* }
      - run: npx playwright merge-reports --reporter=html ./all-blob-reports
      - uses: actions/upload-artifact@v4
        with: { name: html-report, path: playwright-report }
```

Four runners each take a quarter of the tests. The `merge` job combines blob reports into a unified HTML report. **Real production speed**: a 12-minute suite drops to ~3 minutes.

### Sharding by file or by test

`--shard=1/4` distributes test files (not individual tests). For best balance:
- Keep test files small (~5–15 tests each).
- One feature per file.
- Heavy/slow tests in their own files.

If you have one giant `dashboard.spec.ts`, sharding can't help — the whole file goes to one shard. Split it.

---

## 11. Debugging — Trace Viewer Is the Killer Feature

When a CI test fails, three artifacts:
1. **HTML report** — pass/fail, screenshots, error messages.
2. **Video** — `retain-on-failure` records the test.
3. **Trace** — full DOM snapshot history, network log, console output, every action.

### Open a trace

```bash
npx playwright show-trace path/to/trace.zip
```

The trace viewer is a **time-travel debugger**: scroll through every action, see the DOM snapshot at that moment, replay clicks, inspect network requests. **For most failures, the trace tells you what went wrong in 30 seconds.**

### Local debugging — `--ui` mode

```bash
npx playwright test --ui
```

Opens an interactive UI: re-run tests, watch them execute step by step, time-travel through failures, edit and re-run. The 2026 standard local-dev workflow.

### Codegen — Playwright writes the test for you

```bash
npx playwright codegen http://localhost:3000
```

Opens a browser. Click around, fill forms — Playwright writes the matching test code in real-time. Use as a starting point for a new test, then clean up locators (codegen tends to over-specify).

---

## 12. Real Production Patterns

### Pattern A — One `.spec.ts` per feature

```
tests/
├── auth.spec.ts                   ← signup, login, logout, password reset
├── projects.spec.ts                ← create, edit, delete projects
├── tasks.spec.ts                   ← drag-reorder, complete, delete
├── billing.spec.ts                 ← Stripe Checkout, Customer Portal
├── settings.spec.ts                ← profile, team, notifications
└── _smoke.spec.ts                  ← @smoke tag — runs on every deploy
```

`_smoke.spec.ts` contains 3–5 critical happy-path tests. Tag with `test.describe('@smoke', ...)`. Run via `--grep @smoke` after every production deploy. Smoke takes < 60 seconds.

### Pattern B — Tag tests by speed

```ts
test('@slow user uploads 100MB video', async ({ loggedInPage }) => {
  // ...
});

test('@critical checkout completes', async ({ loggedInPage }) => {
  // ...
});
```

CI can filter: PR runs `--grep-invert @slow`, nightly runs everything. Production deploy gate runs `--grep @critical`.

### Pattern C — Test against production (carefully)

```ts
// tests/production-smoke.spec.ts
test.use({ baseURL: 'https://app.example.com' });

test('@critical homepage loads', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveTitle(/Acme/);
});

test('@critical sign-in page is reachable', async ({ page }) => {
  await page.goto('/login');
  await expect(page.getByRole('heading', { name: 'Sign in' })).toBeVisible();
});
```

Run after every prod deploy. **Read-only paths only** — don't sign up real users or charge real cards in production smoke tests. Pair with **synthetic monitoring** (Checkly, Datadog Synthetics) for continuous coverage between deploys.

### Pattern D — Parallel-friendly test data

Each test creates its own data; tests don't share state.

```ts
test('user can rename a project', async ({ loggedInPage, request }) => {
  // Create unique project per test run
  const id = `test-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`;
  const project = await request.post('/api/projects', {
    data: { name: `Test ${id}` },
  }).then((r) => r.json());

  await loggedInPage.goto(`/project/${project.id}`);
  // ... rename
});
```

Don't write tests that pre-suppose "the database has 3 projects." Tests must run in parallel, in any order, repeatedly.

---

## 13. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Tests pass locally, fail in CI | Run `npx playwright test --browser=chromium` headless locally to match CI; check `actionTimeout`; use Docker image `mcr.microsoft.com/playwright:v1.46.0-jammy` |
| Pixel snapshot fails on every PR | Mask volatile content; disable animations; standardize platform via Docker |
| `await expect(...).toBe(true)` flakes | Use `await expect(locator).toBeVisible()` etc. — auto-retrying matchers |
| Tests slow because every one logs in | Use `storageState` to skip login per test |
| `page.waitForTimeout(2000)` everywhere | Almost always wrong; use `expect(...).toBeVisible()` or `waitFor` on a specific state |
| Tests share data and break in parallel | Each test creates its own; teardown if necessary |
| Strict mode error: locator matched 2 elements | Disambiguate: `getByRole('button', { name: 'Save' }).first()` or scope to a parent |
| `getByText('Save')` matches a button AND a description | Use `getByRole` to scope semantically |
| Test passes but `console.error` warns of React errors | Listen to `page.on('console')` and fail on errors |
| Element click intercepted | Other element is overlapping; auto-wait stability check kicks in. Often a modal closing animation isn't done — wait for it: `await expect(modal).toBeHidden()` |
| `.only` / `.skip` accidentally committed | `forbidOnly: isCI` fails the build |
| Tests rely on Date.now() / Math.random() | Mock with `page.addInitScript()` or accept variability with masks |
| `test.use({ ... })` accidentally affects all tests in file | Scope with `test.describe(...)` |
| CI runs on every commit and is slow | Shard + only run E2E on PRs to main, not feature-branch commits |
| iframe interactions don't work | Use `frameLocator()` to drop into the iframe |

---

## 14. Decision Tree

```
What kind of test?
│
├── Unit / pure function → Vitest 1.6
├── Component in JSDOM → Vitest 1.6 + @testing-library/react@15
├── Component needing real browser → Playwright component testing
└── End-to-end / integration in real browser → Playwright @playwright/test@1.46

Which browsers?
│
├── PR runs → chromium only (fast)
└── Nightly → chromium + webkit + firefox

How to authenticate?
│
├── 1–2 tests → fixture per test
└── 3+ tests → storageState reused across tests

How to set up data?
│
├── Always → API or DB directly via request fixture
└── Never → drive the UI to set up preconditions

What about visual regression?
│
├── Built-in → page.screenshot + toHaveScreenshot (DIY)
├── Production grade → Chromatic on top of Storybook (next topic)
└── Marketing/landing pages only → Percy or Applitools

CI sharding?
│
├── Suite < 5 min → don't shard
├── Suite 5–15 min → shard 2–4 ways
└── Suite > 15 min → shard 4–8 ways + tag-based filtering
```

---

## 15. What This Topic Connects To

- **`06.testing-perf/01-testing.md`** — Vitest + RTL for component tests; Playwright for E2E.
- **`11.capabilities/05-accessibility.md`** — `@axe-core/playwright@4` for E2E a11y gates.
- **`12.quality-at-scale/01-feature-flags.md`** — `x-vercel-flag-overrides` header to test both flag arms.
- **`12.quality-at-scale/02-experimentation-ab.md`** — same override pattern for A/B test variants.
- **`10.production/04-observability.md`** — failed Playwright traces feed into observability.
- **`07.nextjs/05-deployment.md`** — production smoke tests run after every Vercel deploy.

---

## Summary

| Tool | Pick when |
|------|-----------|
| `@playwright/test@1.46` | Default E2E for new React/Next.js projects in 2026 |
| `cypress@13` | Existing Cypress codebases only |
| Playwright component testing | E2E + component in one tool, real browser |
| Vitest + `@testing-library/react@15` | Unit + JSDOM component tests (faster) |
| `@axe-core/playwright@4` | E2E accessibility gate |
| Chromatic (next topic) | Production-grade visual regression |

| Rule | Why |
|------|-----|
| Use `getByRole` / `getByLabel` first; `data-testid` last | Tests survive refactors; weaknesses surface accessibility gaps |
| `await expect(locator).toBeVisible()` over `expect(await ...).toBe(true)` | Auto-retrying; flake-tolerant |
| Reuse auth via `storageState` | Login once per CI run, not per test |
| Set up test data via API, not UI | Faster, less flaky, isolates the feature under test |
| Trace on first retry; video on failure | Free debugging when something goes wrong |
| Shard CI for suites > 5 min | Linear speedup with shard count |
| Mask volatile content in screenshots | Cuts ~95% of visual flake |
| Tag tests by speed and criticality | Different CI gates per environment |

---

## Further reading

- [Playwright docs](https://playwright.dev/docs/intro)
- [Playwright — Best practices](https://playwright.dev/docs/best-practices)
- [Playwright — Authentication](https://playwright.dev/docs/auth)
- [Playwright — Visual comparisons](https://playwright.dev/docs/test-snapshots)
- [Playwright — Sharding](https://playwright.dev/docs/test-sharding)
- [Playwright Trace Viewer](https://playwright.dev/docs/trace-viewer)
- [Playwright — Component testing](https://playwright.dev/docs/test-components)
- [Cypress vs Playwright (2024 retrospective)](https://playwright.dev/docs/why-playwright)
