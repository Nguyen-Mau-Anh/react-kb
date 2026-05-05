# Quality at Scale — 01. Feature Flags: PostHog 1 vs LaunchDarkly 7 vs Vercel Edge Config

> **What / Why / How** — pick **PostHog** if you already use it for analytics, **LaunchDarkly 7** for enterprise, **Vercel Edge Config** for ultra-low-latency reads tied to Next.js. Resolve flags **on the server**, not the client; ship them with **kill switches** from day one.

---

## 1. Why Feature Flags Matter for React/Next.js Apps

A feature flag is a **runtime branch** in your code: instead of `if (release === 'v2')`, you write `if (await flag('new-checkout', { user }))`. The decision moves from your build artifact to a remote config service, and you can flip it for a single user, a 5% cohort, or everyone — **without redeploying**.

Real production scenarios where flags pay off:

| Scenario | What flags unlock |
|----------|-------------------|
| **Decouple deploy from release** | Merge to main daily; turn features on later when ready |
| **Gradual rollout** | Ship to 1% → 10% → 50% → 100%; watch error rate at each step |
| **Kill switch** | Production bug? Flip the flag off; no rollback PR or redeploy |
| **A/B / multivariate experiments** | Same flag system, statistical analysis on top (next topic) |
| **Entitlements** | "Pro plan users see this feature" — flag with user attribute targeting |
| **Internal-only beta** | Flag on for `email IN ('@acme.com')` only |
| **Per-tenant features** | Multi-tenant SaaS: different flag values per organization |
| **Trunk-based development** | Long-running branches die; ship behind a flag instead |

The thing that changes when you adopt flags: **deploy stops being a binary "did the feature ship" question**. Code is in production for days or weeks before users see it. This is a major productivity unlock for teams of 5+ engineers.

---

## 2. The Real Choices in 2026

| Tool | npm / SDK | Best for | Pricing |
|------|-----------|----------|---------|
| **PostHog feature flags** | `posthog-js@1.155`, `posthog-node@4` | Already on PostHog for analytics; want flags + experiments + heatmaps in one tool | Free 1M events/mo, $0–$450/mo for typical SaaS |
| **LaunchDarkly 7** | `launchdarkly-node-server-sdk@7`, `launchdarkly-react-client-sdk@3` | Enterprise; the most-used dedicated flag service; strict audit trails | $10/seat/mo + per-MAU; expensive |
| **Vercel Edge Config** | Read with `@vercel/edge-config@1.4` | Ultra-low-latency (<15ms global) flag/config reads on Vercel | Bundled with Vercel Pro |
| **Vercel Flags SDK** | `@vercel/flags@3` (formerly `flags`) | Vercel-native typed flag definitions, integrates with Edge Config + LaunchDarkly + PostHog | Free; the unified abstraction layer |
| **Statsig** | `statsig-js@5`, `statsig-node@5` | Strong free tier; great experimentation primitives | Free up to 1M events/mo |
| **GrowthBook 3** | `@growthbook/growthbook-react@1`, OSS server | Self-hostable, OSS-friendly | Free self-hosted; SaaS from $0 |
| **Flagsmith** | `flagsmith@5` | Self-hostable, GitOps-friendly | Free self-host; SaaS from $0 |
| **Hypertune** | `hypertune@1.x` | Type-safe flags with code-gen, Vercel-friendly | Free tier + paid |
| **Unleash 5** | `unleash-client@5`, `@unleash/proxy-client-react@4` | Self-hostable, OSS-first | Free self-hosted |
| **DIY (env vars + DB row)** | n/a | Tiny apps, prototype | Free, but no UI / audit / targeting |

**The 2026 default for new React/Next.js projects**: **Vercel Flags SDK + a provider underneath**. The Flags SDK is a thin abstraction; you pick the actual evaluation engine separately.

| You're building... | Pick |
|--------------------|------|
| Anything on Vercel + already on PostHog | `@vercel/flags` + `@vercel/flags/providers/posthog` |
| Vercel + plain config (no provider needed) | `@vercel/flags` + `@vercel/flags/providers/edge-config` |
| Self-hosted Next.js, OSS-first | GrowthBook 3 self-hosted |
| Enterprise with strict audit / RBAC needs | LaunchDarkly 7 |
| Indie SaaS, want flags + analytics + experiments in one tool | PostHog |
| Tiny app, just need a kill switch | Vercel Edge Config + a single `enabled: boolean` |

This file uses **Vercel Flags SDK + PostHog** as the worked example because it's the most-deployed combination on the modern Next.js stack.

---

## 3. The Server-First Resolution Rule

The single most important rule about feature flags in Next.js 14+:

> **Resolve flags on the server, not in the browser.**

Why it matters:

| Concern | Client-resolved | Server-resolved |
|---------|-----------------|-----------------|
| Time to flag-aware HTML | ~200ms (after JS loads, fetches flags) | 0ms (HTML is already correct) |
| Layout shift when flag flips UI | Yes — flicker between control and treatment | No |
| User can manipulate flag values via DevTools | Yes (cosmetic; security via flags is anti-pattern anyway) | No |
| Works with Server Components | Awkward (you'd render twice) | Native |
| Works in Edge Middleware | No | Yes |
| SEO sees the right variant | No (Googlebot rarely runs JS for ranking) | Yes |
| Works without JS | No | Yes |

In Next.js 14+ with App Router and Server Components, server-resolved flags are the natural fit. The pattern: a Server Component or Edge Middleware decides, the React tree below it just renders. **The user never sees the wrong variant first.**

Client-side flags still have a role (toggling small UI bits inside an already-rendered page, e.g., showing a beta badge), but the **default** should be server-resolved.

---

## 4. Vercel Flags SDK — The Modern Default

`@vercel/flags@3` (formerly `flags@2`) gives you typed, server-resolved flags with adapters for every popular provider.

### Install

```bash
npm i @vercel/flags@3
```

### Define a flag

```ts
// flags.ts
import { flag } from '@vercel/flags/next';
import { postHog } from '@vercel/flags/providers/posthog';

export const newCheckoutFlag = flag<boolean>({
  key: 'new-checkout',
  defaultValue: false,
  description: 'Enables the redesigned Stripe Payment Element checkout',
  origin: 'https://app.posthog.com/feature_flags/...',
  options: [
    { label: 'Enabled', value: true },
    { label: 'Disabled', value: false },
  ],
  decide: postHog({ apiKey: process.env.POSTHOG_KEY!, host: process.env.POSTHOG_HOST! }),
  identify: async () => {
    const { auth } = await import('@/auth');
    const session = await auth();
    return session?.user?.id ? { distinctId: session.user.id } : undefined;
  },
});
```

What this declares:
- **Typed value** — `flag<boolean>` returns a typed function. For multi-variant: `flag<'control' | 'variant-a' | 'variant-b'>`.
- **`defaultValue`** — what to return if the provider is unreachable. **Always set this.** A flag service outage shouldn't break your app.
- **`decide`** — the resolver. PostHog adapter ships in `@vercel/flags/providers/posthog`; LaunchDarkly in `/launchdarkly`; Edge Config in `/edge-config`.
- **`identify`** — how to identify the current user for targeting. Server-side: read from your auth session.

### Use the flag in a Server Component

```tsx
// app/checkout/page.tsx
import { newCheckoutFlag } from '@/flags';
import { LegacyCheckout } from './LegacyCheckout';
import { NewCheckout } from './NewCheckout';

export default async function CheckoutPage() {
  const enabled = await newCheckoutFlag();
  return enabled ? <NewCheckout /> : <LegacyCheckout />;
}
```

That's it. The check runs on the server. The HTML the user receives already contains the right variant. No flicker, no layout shift, no hydration mismatch.

### Use in a Route Handler / Server Action

```ts
// app/api/checkout/route.ts
import { newCheckoutFlag } from '@/flags';

export async function POST(req: Request) {
  if (await newCheckoutFlag()) {
    return handleNewCheckout(req);
  }
  return handleLegacyCheckout(req);
}
```

Same shape. The flag is just an `async` function returning the typed value.

### Use in Edge Middleware (the killer pattern)

```ts
// middleware.ts
import { newCheckoutFlag } from '@/flags';
import { NextResponse } from 'next/server';

export async function middleware(req: Request) {
  if (await newCheckoutFlag()) {
    return NextResponse.rewrite(new URL('/checkout-v2', req.url));
  }
  return NextResponse.next();
}

export const config = { matcher: '/checkout' };
```

The user hits `/checkout`. The middleware checks the flag at the edge (sub-30ms global latency on Vercel). It rewrites to `/checkout-v2` if the flag is on — **the user's URL stays `/checkout`**, the rendered page is from `/checkout-v2`. Used by Vercel's own dashboard and Linear's preview-cohort releases.

---

## 5. PostHog Feature Flags — Targeted Rollout

If you're using PostHog (and you should consider it — covered in `10.production/04-observability.md`), feature flags are part of the same product.

### Server-side resolution

```ts
// flags.ts
import { flag } from '@vercel/flags/next';
import { postHog } from '@vercel/flags/providers/posthog';

export const proPlanFeature = flag<boolean>({
  key: 'pro-plan-features',
  defaultValue: false,
  decide: postHog({ apiKey: process.env.POSTHOG_KEY!, host: process.env.POSTHOG_HOST! }),
  identify: async () => {
    const session = await auth();
    if (!session?.user) return undefined;
    return {
      distinctId: session.user.id,
      properties: {
        plan: session.user.plan,                  // 'free' | 'pro' | 'enterprise'
        signupAt: session.user.createdAt,
        teamSize: session.user.teamSize,
      },
    };
  },
});
```

In the PostHog UI, build a release condition:

```
Match users where:
  plan = 'pro'
  AND (rollout: 50% of matching users)
```

The server reads the user's `plan` property when resolving. The user gets a stable bucket — once they're in the 50%, they stay in the 50% across sessions. The hash uses `distinctId`.

### Client-side fallback for late-resolved flags

If you need a flag in a Client Component (e.g., toggling a button label that only matters after JS loads):

```tsx
'use client';
import { useFeatureFlagEnabled } from 'posthog-js/react';

function BetaBadge() {
  const enabled = useFeatureFlagEnabled('beta-badge');
  if (!enabled) return null;
  return <span className="rounded bg-amber-200 px-2 py-0.5 text-xs">Beta</span>;
}
```

`posthog-js@1.155` caches flag values in `localStorage` for fast subsequent loads. Initial load: flag arrives ~100ms after page paint, badge fades in.

For pages where flag values affect critical layout, **don't** rely on this — use the Server Component path.

### Fetch flags ahead-of-render (Server Components mixed with Client)

```tsx
// app/dashboard/page.tsx (Server Component)
import { newDashboardFlag } from '@/flags';
import { DashboardClient } from './DashboardClient';

export default async function DashboardPage() {
  const variant = await newDashboardFlag();
  return <DashboardClient variant={variant} />;
}
```

```tsx
// app/dashboard/DashboardClient.tsx
'use client';
export function DashboardClient({ variant }: { variant: 'control' | 'v2' }) {
  return variant === 'v2' ? <V2 /> : <V1 />;
}
```

The server resolves once, passes the variant down as a prop. No client-side flag fetch at all.

---

## 6. Vercel Edge Config — The Lowest-Latency Path

For flags whose values rarely change but must be read on every request (kill switches, maintenance mode, region-blocking), **Vercel Edge Config** is the right tool. Reads are < 15ms globally.

### Setup

```bash
npm i @vercel/edge-config@1.4 @vercel/flags@3
```

```ts
// flags.ts
import { flag } from '@vercel/flags/next';
import { edgeConfig } from '@vercel/flags/providers/edge-config';

export const maintenanceMode = flag<boolean>({
  key: 'maintenance-mode',
  defaultValue: false,
  decide: edgeConfig(),
});

export const blockedRegions = flag<string[]>({
  key: 'blocked-regions',
  defaultValue: [],
  decide: edgeConfig(),
});
```

In Vercel dashboard → Edge Config → set `maintenance-mode: true` to instantly take the site down for maintenance. Globally propagated within seconds.

```ts
// middleware.ts
import { maintenanceMode } from '@/flags';

export async function middleware(req: Request) {
  if (await maintenanceMode()) {
    return new Response('Be right back!', {
      status: 503,
      headers: { 'Content-Type': 'text/html', 'Retry-After': '120' },
    });
  }
  return NextResponse.next();
}
```

This is the **simplest disaster-recovery primitive**: a global "off switch" any on-call engineer can flip from a phone in 10 seconds.

### When to use Edge Config vs PostHog/LaunchDarkly

| Use case | Tool |
|----------|------|
| Boolean kill switches, maintenance mode, region blocks | Edge Config |
| Per-user / per-cohort targeting with attributes | PostHog or LaunchDarkly |
| A/B experiments with statistical significance | PostHog or LaunchDarkly (covered in next topic) |
| Multi-variant flags (`'control' | 'v1' | 'v2'`) | PostHog or LaunchDarkly |
| Audit trail of who flipped what when | LaunchDarkly (best); PostHog has it |
| OSS / self-hosted | GrowthBook 3 or Flagsmith 5 |

Many apps combine: Edge Config for kill switches, PostHog for product flags. They cost nothing extra together.

---

## 7. LaunchDarkly 7 — Enterprise

LaunchDarkly 7 (`launchdarkly-node-server-sdk@7`, `launchdarkly-react-client-sdk@3`) is the most-deployed dedicated flag service in 2026 enterprises.

### Why LaunchDarkly wins for enterprise

- **Strict RBAC** — multi-team workspaces, environment isolation (dev / staging / prod), SAML SSO.
- **Audit log** — every flag change records who, when, what, with PR-style approval workflows for prod.
- **Targeting language** — segment users by 100+ attributes, geo, device, custom rules with full operators.
- **Evaluations as code** (Code References) — LaunchDarkly scans your repo for stale flag references and flags them in the dashboard.
- **Gateway-grade SLA** — 99.99% uptime, sub-50ms median resolution.

### Setup

```bash
npm i launchdarkly-node-server-sdk@7
```

```ts
// lib/launchdarkly.ts
import * as ld from 'launchdarkly-node-server-sdk';

let client: ld.LDClient | null = null;
export async function getLDClient() {
  if (!client) {
    client = ld.init(process.env.LAUNCHDARKLY_SDK_KEY!);
    await client.waitForInitialization();
  }
  return client;
}

export async function getFlag<T>(key: string, user: { id: string; email: string }, defaultValue: T): Promise<T> {
  const c = await getLDClient();
  return c.variation(key, { kind: 'user', key: user.id, email: user.email }, defaultValue) as T;
}
```

### Use via Vercel Flags SDK adapter

The cleanest pattern: Vercel Flags SDK + LaunchDarkly adapter:

```ts
import { flag } from '@vercel/flags/next';
import { ldAdapter } from '@vercel/flags/providers/launchdarkly';

export const proCheckoutFlag = flag<boolean>({
  key: 'pro-checkout',
  defaultValue: false,
  decide: ldAdapter({ projectKey: 'default', sdkKey: process.env.LAUNCHDARKLY_SDK_KEY! }),
  identify: async () => {
    const session = await auth();
    return session?.user ? { kind: 'user', key: session.user.id, email: session.user.email! } : undefined;
  },
});
```

You get LaunchDarkly's targeting power + Vercel's typed flag definitions. Switch providers later without changing flag-consumer code.

### When LaunchDarkly's price isn't worth it

- Solo developer or < 5-person team — PostHog or GrowthBook covers the same ground free.
- < 10 flags total — Vercel Edge Config is enough.
- No audit/compliance need — overhead doesn't pay off.

LaunchDarkly's typical SaaS cost is ~$10/seat/month + a per-MAU rate. A small SaaS with 5 engineers + 50K MAU pays roughly $300–800/month. Real teams pay this when audit trails or reliability SLAs are non-negotiable.

---

## 8. Designing Flag Schemes That Scale

A bad flag system creates worse code than no flag system. Real conventions that pay off:

### Flag-naming conventions

```
release.<feature>          → release.new-checkout
ops.<switch>               → ops.maintenance-mode
exp.<experiment>           → exp.signup-cta-copy
perm.<entitlement>         → perm.pro-export
config.<value>             → config.max-uploads
```

Prefixing tells you the **lifecycle** at a glance:

| Prefix | Lifecycle | Cleanup window |
|--------|-----------|----------------|
| `release.*` | Days to weeks; remove after 100% rollout | < 30 days |
| `exp.*` | Days; resolve after experiment concludes | < 30 days |
| `ops.*` | Permanent kill switches | Never |
| `perm.*` | Permanent entitlements | Tied to billing |
| `config.*` | Permanent runtime config | Never |

### Make ops/perm flags easy to spot in code

```ts
// ✅ Lifecycle visible at the call site
if (await flag('ops.kill-switch-payments')) { ... }
if (await flag('release.new-checkout')) { ... }
```

When a release flag has been at 100% for 30 days, **delete it**. Dead flags accumulate and become time bombs. Most flag platforms (LaunchDarkly Code References, PostHog Flag Status, Statsig Stale Flags) detect stale flags and flag them in the dashboard.

### Dependency rule

Don't have flag A depend on flag B's value at render time. The dependency makes rollouts hard to reason about. If you need `if (newCheckout && newPayments)`, evaluate both flags into a derived value and document the relationship.

### Use percent-rollouts, not all-or-nothing

```
release.new-dashboard:
  rule 1: emails ending in @acme.com → enabled
  rule 2: 5% of all other users → enabled
  default: disabled
```

5% catches issues before 95% of users feel them. Bump to 25%, 50%, 100% over hours/days based on error rates and engagement metrics.

### One flag per release, not per file

Bundle the entire feature behind one flag. If `new-checkout` flips on, every related component flips. The Stripe Payment Element migration (covered in `11.capabilities/01-payments.md`) is one flag, not eight.

---

## 9. Real-World Patterns

### Kill switch for a critical path

```ts
// flags.ts
export const enableUploads = flag<boolean>({
  key: 'ops.enable-uploads',
  defaultValue: true,
  decide: edgeConfig(),
});
```

```tsx
// app/upload/page.tsx
const enabled = await enableUploads();
if (!enabled) {
  return <p>Uploads are temporarily disabled.</p>;
}
return <UploadForm />;
```

Default `true` means "fail open." If the Edge Config read fails, uploads still work. **Default value should match "is everything fine" semantics.**

### Per-tenant features (multi-tenant SaaS)

```ts
export const advancedReports = flag<boolean>({
  key: 'perm.advanced-reports',
  defaultValue: false,
  decide: postHog({ ... }),
  identify: async () => {
    const session = await auth();
    if (!session?.user) return undefined;
    return {
      distinctId: session.user.id,
      properties: {
        organizationId: session.user.orgId,
        plan: session.user.plan,
      },
    };
  },
});
```

In PostHog UI: target `organizationId IN ['acme-corp', 'wayne-enterprises']` AND `plan = 'enterprise'`. The flag now drives entitlement without DB schema changes — flip it on for new sales accounts via a single config change.

### Flag-driven A/B test (lead-in to next topic)

```ts
export const signupCtaCopy = flag<'control' | 'urgency' | 'curiosity'>({
  key: 'exp.signup-cta-copy',
  defaultValue: 'control',
  decide: postHog({ ... }),
  options: [
    { label: 'Control', value: 'control' },
    { label: 'Urgency', value: 'urgency' },
    { label: 'Curiosity', value: 'curiosity' },
  ],
});
```

In PostHog, attach an experiment to this flag. The experiment auto-buckets users 33/33/33 (configurable), tracks conversion events, and reports significance. Covered in `12.quality-at-scale/02-experimentation-ab.md`.

### Feature gating for new accounts only

```ts
identify: async () => {
  const session = await auth();
  if (!session?.user) return undefined;
  return {
    distinctId: session.user.id,
    properties: {
      signupAt: session.user.createdAt,
      newUser: session.user.createdAt > '2025-06-01',
    },
  };
},
```

Targeting: `newUser = true`. Old accounts keep the existing UI; only new signups see the redesign. Reduces support volume from "where did X go" complaints during a redesign.

---

## 10. Local Development with Flags

You don't want to flip flags in PostHog/LaunchDarkly to test locally. Three patterns:

### Pattern A — env-var override

```ts
export const newCheckoutFlag = flag<boolean>({
  key: 'release.new-checkout',
  defaultValue: false,
  decide: process.env.NODE_ENV === 'development'
    ? () => process.env.FLAG_NEW_CHECKOUT === 'true'
    : postHog({ ... }),
});
```

Toggle with `FLAG_NEW_CHECKOUT=true npm run dev`. Crude but works.

### Pattern B — Vercel Flags SDK toolbar

`@vercel/flags@3` ships a developer toolbar that lets you override flag values per-browser-session. Add to your dev-only layout:

```tsx
// app/layout.tsx
import { FlagValues } from '@vercel/flags/react';
import { encryptFlagValues } from '@vercel/flags';
import { Suspense } from 'react';

async function ConfidentialFlagValues() {
  const values = await Promise.all([newCheckoutFlag(), maintenanceMode()]);
  const encrypted = await encryptFlagValues({
    'release.new-checkout': values[0],
    'ops.maintenance-mode': values[1],
  });
  return <FlagValues values={encrypted} />;
}

<Suspense>{process.env.NODE_ENV !== 'production' && <ConfidentialFlagValues />}</Suspense>
```

The toolbar shows current flag values in dev, lets you override per session, and writes the encrypted state into a cookie that the server resolver respects. You debug variants without touching the upstream provider.

### Pattern C — Per-environment provider

PostHog and LaunchDarkly support multiple environments (dev / staging / prod) with separate flag configs. Engineers flip dev flags freely; prod stays untouched. This is the right pattern at team scale, native in every paid flag tool.

---

## 11. Testing Flag-Aware Code

### Server-side: mock the flag

```ts
// __tests__/checkout.test.tsx (Vitest)
import { describe, it, vi } from 'vitest';
import { newCheckoutFlag } from '@/flags';

vi.mock('@/flags', () => ({
  newCheckoutFlag: vi.fn(),
}));

it('shows new checkout when flag is on', async () => {
  vi.mocked(newCheckoutFlag).mockResolvedValue(true);
  const Page = (await import('@/app/checkout/page')).default;
  // ...assert against rendered output
});
```

### E2E: drive both variants in Playwright

```ts
// e2e/checkout.spec.ts
import { test } from '@playwright/test';

test.describe('checkout — new variant', () => {
  test.use({
    extraHTTPHeaders: { 'x-vercel-flag-overrides': 'release.new-checkout=true' },
  });
  test('completes payment', async ({ page }) => { /* ... */ });
});

test.describe('checkout — control variant', () => {
  test.use({
    extraHTTPHeaders: { 'x-vercel-flag-overrides': 'release.new-checkout=false' },
  });
  test('completes payment', async ({ page }) => { /* ... */ });
});
```

The Vercel Flags SDK reads `x-vercel-flag-overrides` and respects the override. Run both suites in CI; both must pass before the flag goes to 100%.

---

## 12. Observability for Flags

Flags without telemetry are dangerous. Wire flag evaluations into your observability stack (covered in `10.production/04-observability.md`).

### Log every evaluation

PostHog records every flag evaluation automatically. LaunchDarkly does the same with a 1-day retention by default. **Don't disable analytics on flags** — you need to know who's in which cohort to debug issues.

### Tag Sentry events with flag values

```ts
import * as Sentry from '@sentry/nextjs';

Sentry.setTag('flag.new-checkout', String(await newCheckoutFlag()));
```

Now every Sentry error includes the flag state. When errors spike, you can immediately filter by `flag.new-checkout = true` to see if the new variant is the culprit.

### Track exposure

When a flag is evaluated for a user, that user is "exposed." Logging exposures separately from event success lets the analytics layer compute experiment significance. PostHog and LaunchDarkly do this automatically; if you self-host, log a `$feature_flag_called` event whenever a flag is read.

### Alert on flag-correlated errors

In Sentry / Datadog: alert if `flag.new-checkout=true` errors > 2× `flag.new-checkout=false` errors. Auto-creates an incident the moment a release-flag rollout regresses.

---

## 13. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Flag check inside a render-time branch causes hydration mismatch | Resolve on server (Server Component or middleware), pass result as prop |
| Stale flag still referenced in code 6 months later | LaunchDarkly Code References / PostHog stale flag detection; CI gate that fails on dead flags |
| Default value flips when provider is down | Always set `defaultValue` to "fail open" semantics; default of `false` for new feature is safe |
| Same user gets different variants on each refresh | Always pass a stable `distinctId` (user ID, not a per-session UUID) |
| Anonymous user gets buckets that change after sign-in | Use a stable anonymous ID (e.g., a cookie set on first visit), then alias to user ID via `posthog.alias()` after sign-in |
| Flag evaluations slow down every request | Cache provider results; PostHog Node SDK caches by default; LaunchDarkly streams updates |
| Edge function fails on flag fetch | Set `defaultValue` and use `await Promise.race([flagPromise, timeoutPromise])` to time-bound |
| Multiple environments share the same key | Use environment-specific config in your provider; never one flag for dev + prod |
| Flag changes don't propagate fast | LaunchDarkly streams updates within seconds; PostHog polls (default 60s, configurable). Edge Config propagates within ~10s |
| Test mocking breaks because import is dynamic | Mock the flag module before importing the component; use `vi.hoisted()` if needed |
| Targeting by IP / region bypassed by VPN | IP/region targeting is best-effort; use account-level rules for hard entitlements |
| Single flag controls multiple unrelated features | Split. One feature, one flag. |
| Different teams collide on flag namespaces | Adopt the prefix convention (`release.`, `ops.`, `exp.`); namespace per team if needed (`team-x.release.foo`) |

---

## 14. Cleanup — The Boring Discipline

Stale flags are technical debt. Real cleanup process:

1. **Tag flags with an owner and target removal date** when creating them.
2. **PR template** asks: "Is this introducing a flag? When is it removed?"
3. **Quarterly cleanup sprint**: review all flags older than 90 days. Remove or justify.
4. **Code references** — both LaunchDarkly and PostHog scan your repo and flag stale usages.
5. **CI rule**: fail builds that introduce a new flag without an `// flag-removal: 2026-09-01` comment.

Real engineering teams that skip this end up with 200+ flags after 2 years. Each one is a runtime branch that can fail. **Cleanup is part of the cost of using flags.** Plan for it from day one.

---

## 15. Decision Tree

```
What's the use case?
│
├── Kill switch / maintenance mode / region block
│   → Vercel Edge Config + @vercel/flags (boolean flag, fail-open default)
│
├── Gradual feature rollout (5% → 100%)
│   ├── Already on PostHog → PostHog flags via @vercel/flags adapter
│   ├── Enterprise + audit needs → LaunchDarkly 7
│   └── Self-hosted / OSS-first → GrowthBook 3
│
├── A/B / multivariate experiment
│   → Same as gradual rollout; pair with experimentation features (next topic)
│
├── Per-tenant entitlement (Pro plan, organization-level features)
│   → PostHog or LaunchDarkly with attribute targeting
│
└── Tiny app, just one or two flags
   → Vercel Edge Config (boolean) + a hardcoded JSON for non-Vercel hosts

Where to resolve?
│
├── Default → Server Component (Next.js 14+ App Router)
├── Need to gate routing → Edge Middleware
├── Need at request time on a Route Handler → server fetch
└── Late-resolved UI bit (e.g., beta badge) → posthog-js useFeatureFlagEnabled

Where to set the default value?
│
├── Kill switch → match "everything is fine" (usually `true`)
├── New feature → `false` (fail safe; users see existing behavior)
└── Multivariate → 'control' (same as the existing experience)
```

---

## 16. What This Topic Connects To

- **`07.nextjs/02-routing.md`** — middleware as a flag-resolution site.
- **`07.nextjs/03-data-fetching.md`** — Server Components + Server Actions reading flags.
- **`07.nextjs/04-auth.md`** — `auth()` provides the user identity for flag targeting.
- **`10.production/04-observability.md`** — tagging Sentry events with flag values.
- **`12.quality-at-scale/02-experimentation-ab.md`** — flags as the foundation for A/B tests (next topic).
- **`08.ecosystem/04-real-project.md`** — adding a flag-gated "new dashboard" to the Task Manager is a 1-hour PR.

---

## Summary

| Tool | Pick when |
|------|-----------|
| `@vercel/flags@3` | Default abstraction layer for any new Next.js project |
| `@vercel/edge-config@1.4` | Ultra-low-latency boolean flags / kill switches on Vercel |
| `posthog-js@1.155` + `posthog-node@4` | Already on PostHog; want flags + analytics + experiments together |
| `launchdarkly-node-server-sdk@7` | Enterprise — strict audit, RBAC, SLA |
| GrowthBook 3 / Flagsmith 5 / Unleash 5 | Self-hosted, OSS-first |
| Statsig | Strong free tier, experimentation focus |
| DIY env vars | Tiny apps; outgrown quickly |

| Rule | Why |
|------|-----|
| Resolve flags on the server | No flicker, no hydration mismatch, works without JS |
| Always set a `defaultValue` matching "fail open" | Provider outage shouldn't break the app |
| Use stable `distinctId` per user | Bucket consistency across sessions |
| Prefix flags by lifecycle (`release.`, `ops.`, `exp.`, `perm.`, `config.`) | Spot dead flags at a glance |
| One feature, one flag | Independent rollouts, easier to reason about |
| Tag Sentry events with flag values | Diagnose flag-correlated regressions instantly |
| Quarterly cleanup of stale flags | They are runtime branches; they accumulate; they break |

---

## Further reading

- [Vercel Flags SDK docs](https://flags-sdk.dev/)
- [Vercel Edge Config docs](https://vercel.com/docs/storage/edge-config)
- [PostHog feature flags](https://posthog.com/docs/feature-flags)
- [LaunchDarkly Node SDK](https://docs.launchdarkly.com/sdk/server-side/node-js)
- [GrowthBook docs](https://docs.growthbook.io/)
- [Statsig SDK docs](https://docs.statsig.com/)
- [Pete Hodgson — Feature Toggles (Martin Fowler)](https://martinfowler.com/articles/feature-toggles.html) — the canonical taxonomy
