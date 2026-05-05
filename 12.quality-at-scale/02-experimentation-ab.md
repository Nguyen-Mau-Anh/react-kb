# Quality at Scale — 02. A/B Testing: PostHog Experiments, GrowthBook 3, Statistical Significance

> **What / Why / How** — pick **PostHog Experiments** if already on PostHog; **GrowthBook 3** if you want OSS + Bayesian stats. Run each test with a **predeclared sample size** and **two-week minimum**; never "peek" at intermediate results.

---

## 1. The Real Choices in 2026

A/B testing tools split into **flag-based experimentation** (run on top of a feature-flag service) and **dedicated experimentation platforms**.

| Tool | npm / SDK | Type | Best for | Pricing |
|------|-----------|------|----------|---------|
| **PostHog Experiments** | `posthog-js@1.155`, `posthog-node@4` | Flag + analytics + experiments unified | Already on PostHog; want one tool | Free 1M events/mo |
| **GrowthBook 3** | `@growthbook/growthbook-react@1`, `growthbook@1` (server) | OSS, Bayesian stats by default | Self-hosted, OSS-first, statistical rigor | Free self-hosted; SaaS from $0 |
| **Statsig** | `statsig-js@5`, `statsig-node@5` | Flags + experiments + autotune | Strong free tier, sequential testing built-in | Free up to 1M events/mo |
| **LaunchDarkly Experimentation** | `launchdarkly-react-client-sdk@3` | Add-on to LaunchDarkly flags | Enterprise; already paying for LD | $$$ (add-on) |
| **Optimizely Web Experimentation** | `@optimizely/optimizely-sdk@5` | Long-time enterprise leader | Big-co marketing-led teams | $$$$ |
| **Vercel Web Analytics + Flags + custom analysis** | `@vercel/analytics@1.3`, `@vercel/flags@3` | DIY — flag controls variant; you compute significance externally | Tiny apps, want to own the math | Vercel cost only |
| **VWO** | `@vwo/vwo-js@1` | Visual editor + experiments | Marketing-led teams who don't want to code | $$$$ |
| **AB Tasty / Convert.com** | various | Visual editor | Same as VWO | $$$$ |

**The 2026 default for engineering-led teams**: **PostHog Experiments** (if already on PostHog) or **GrowthBook 3** (for OSS / Bayesian preference). Both are free at small scale, both integrate with the Vercel Flags SDK from `12.quality-at-scale/01-feature-flags.md`.

The "visual editor" tools (VWO, AB Tasty, Optimizely Web) are still around and useful for marketing pages where designers and PMs need to ship variants without engineering. For product-engineering experimentation: stick with the developer-tool path.

---

## 2. Why A/B Testing Is Different From Feature Flags

Feature flags answer "should this user see X?" A/B tests answer "**does X cause** the metric to change vs not seeing X?"

The distinction matters because A/B tests need:

| Concern | Feature flag (release rollout) | A/B test (experiment) |
|---------|--------------------------------|------------------------|
| **Random assignment** | Optional (often by attributes) | Required (uniform random within an audience) |
| **Statistical analysis** | Not needed | Required — confidence intervals, p-values or Bayesian posteriors |
| **Sample size** | Doesn't matter | Pre-declared; insufficient = inconclusive |
| **Stable assignment** | Important for UX | Critical — flipping a user mid-experiment ruins the data |
| **Stop early** | Anytime | Only if pre-registered or via sequential analysis |
| **Multiple variants** | Rare | Common (control + 1–4 treatments) |

A flag-driven percentage rollout is **not** an A/B test unless you actually compute conversion rates per variant. If you don't measure, you only know it didn't crash — not whether it's better.

---

## 3. Statistical Foundations You Actually Need

You need just enough stats to not draw bad conclusions. The minimum:

### Conversion rate, observed effect, and noise

If 5% of users in control complete a checkout and 6% in variant:
- **Observed lift**: +20% relative (from 5% to 6%); or +1 percentage point absolute.
- **Noise**: with 1,000 users in each group, the natural sampling variation is ±~1.4 percentage points at 95% confidence. The +1-point lift is **within noise**.

You can't conclude anything from that. You'd need ~10,000 users per group to detect a 1-point absolute lift at 80% power.

### Power and sample size — calculate before you start

The 2026 standard sizing pre-test:

| You want to detect | Baseline rate | Sample size per arm |
|--------------------|--------------|--------------------|
| 10% relative lift on 5% conversion | 5% | ~31,000 users |
| 5% relative lift on 5% conversion | 5% | ~125,000 users |
| 10% relative lift on 20% conversion | 20% | ~6,200 users |
| 5% relative lift on 20% conversion | 20% | ~25,000 users |

Use the [Evan Miller calculator](https://www.evanmiller.org/ab-testing/sample-size.html) or `power.prop.test` in R. PostHog and GrowthBook expose this directly in the experiment-creation UI.

**If your traffic doesn't support detecting the effect you care about, don't run the test.** Most "I ran an A/B test, no significant difference" stories are sample-size failures — the test couldn't see anything smaller than a 30% effect, but the team was hoping to detect 5%.

### p-value vs confidence interval vs Bayesian posterior

Three real frameworks:

| Framework | What you read | Used by |
|-----------|---------------|---------|
| **Frequentist** (p-values) | "p < 0.05 → reject null hypothesis" | Most older A/B tools, classic stats courses |
| **Confidence interval** (frequentist) | "95% CI of lift: +0.2% to +1.8%" — both bounds positive → significant | Modern frequentist UI |
| **Bayesian posterior** | "92% probability variant beats control" | GrowthBook (default), Statsig, modern PostHog |

In 2026 most experimentation tools default to **Bayesian** because the output is more intuitive: "92% chance variant is better" answers the actual product question, while "p = 0.03" answers a different one ("how surprising is this data if there's no effect?").

For an engineering team: the Bayesian "probability variant wins" framing is easier to communicate to PMs and execs. Both PostHog and GrowthBook default to it.

### The peeking problem — why you can't check daily

Classical A/B tests assume you set a sample size, wait, then look once. **If you look every day and stop the moment results look good, your false-positive rate balloons** (from 5% to ~30% if you peek 10 times).

Two real fixes:

| Solution | When |
|----------|------|
| **Pre-register sample size and time, don't peek** | Easy, classical |
| **Sequential testing** (mSPRT, GST) | Built into Statsig and modern GrowthBook; lets you legitimately monitor and stop early |
| **Bayesian framework** | Less peeking-sensitive; PostHog and GrowthBook default to Bayesian for this reason |

If your tool supports sequential testing, use it — you can monitor without inflating false-positives. If not, **set the duration before starting and stick to it**. A 2-week minimum is the rule of thumb that catches weekly seasonality (weekday vs weekend behavior).

### Multiple comparisons

Running 4 variants vs 1 control = 4 comparisons. The chance of *some* comparison hitting "significant" by luck rises. Mitigate:

- **Bonferroni correction**: divide the significance threshold by the number of comparisons. Conservative but simple.
- **False Discovery Rate (FDR)**: less conservative; tools handle it.
- **Just don't run too many variants at once**. 2-arm tests (control + 1 treatment) are cleanest.

If you're testing 5 button colors at once, that's not 5 A/B tests — that's a multi-armed-bandit problem. Different tool (Statsig has bandit support; PostHog doesn't).

---

## 4. PostHog Experiments — The 2026 Default

### What

`posthog-js@1.155` + `posthog-node@4` ship A/B experimentation as part of the same product as analytics, flags, and session replay. You can run an experiment on any feature flag — PostHog auto-buckets users, tracks an `$exposure` event, and computes Bayesian posteriors against your chosen primary metric.

### Setup — recap from the flags topic

```bash
npm i posthog-js@1.155 posthog-node@4 @vercel/flags@3
```

```ts
// flags.ts
import { flag } from '@vercel/flags/next';
import { postHog } from '@vercel/flags/providers/posthog';

export const signupCtaCopy = flag<'control' | 'urgency' | 'curiosity'>({
  key: 'exp.signup-cta-copy',
  defaultValue: 'control',
  decide: postHog({ apiKey: process.env.POSTHOG_KEY!, host: process.env.POSTHOG_HOST! }),
  identify: async () => {
    const session = await auth();
    return session?.user?.id ? { distinctId: session.user.id } : undefined;
  },
});
```

In PostHog UI:
1. Open the flag `exp.signup-cta-copy`.
2. Click **"Convert to experiment"**.
3. Set **primary metric**: `signup_completed` event.
4. Set **variants**: `control` (33%), `urgency` (33%), `curiosity` (34%).
5. Set **minimum acceptable improvement (MDE)**: e.g., 5%.
6. PostHog calculates required sample size and shows it.
7. Click **"Launch"**.

### Use the variant in code

```tsx
// app/(marketing)/page.tsx (Server Component)
import { signupCtaCopy } from '@/flags';

const COPY = {
  control:   'Sign up free',
  urgency:   'Sign up — limited beta access',
  curiosity: 'Sign up and see what we built',
};

export default async function HomePage() {
  const variant = await signupCtaCopy();
  return (
    <main>
      <h1>Build faster</h1>
      <a href="/signup" data-variant={variant}>
        {COPY[variant]}
      </a>
    </main>
  );
}
```

That's it. The variant resolves on the server, the user sees the right CTA on first paint, **PostHog's exposure event fires automatically** when the flag is read.

### Tracking the conversion event

```tsx
'use client';
import posthog from 'posthog-js';

// In your signup completion handler:
function handleSignupComplete(user: User) {
  posthog.capture('signup_completed', {
    plan: 'free',
    referrer: document.referrer,
  });
}
```

PostHog ties the conversion event to the user's `distinctId`, looks up which experiment variant they were exposed to, and updates the experiment's posterior. **You don't manually compute anything.**

### Reading results

PostHog UI shows:
- **Probability to be best** per variant (Bayesian).
- **Credible interval** of the lift.
- **Status**: Inconclusive / Significant winner / Significant loser.
- **"Decision tree"**: should we ship variant X?

The recommendation aligns with the Bayesian threshold (95% probability by default).

### Real production pattern

```ts
// Stage 1: ship the experiment
await signupCtaCopy(); // returns 'control' | 'urgency' | 'curiosity'

// Stage 2: experiment concludes; PostHog declares 'urgency' the winner
// In PostHog UI: convert experiment to feature flag, set 'urgency' to 100%

// Stage 3: clean up the code
- const variant = await signupCtaCopy();
- return <a>{COPY[variant]}</a>;
+ return <a>Sign up — limited beta access</a>;

// Stage 4: delete the flag in PostHog after deploy
```

Each stage is a separate PR. Stale experiment flags are technical debt (covered in `12.quality-at-scale/01-feature-flags.md`).

---

## 5. GrowthBook 3 — OSS, Bayesian-First

### What

`@growthbook/growthbook-react@1` is the React SDK for GrowthBook 3, an open-source experimentation platform. Self-host (Docker / k8s) or use their hosted SaaS.

### Why pick GrowthBook over PostHog

- **Strong stats foundation**: defaults to Bayesian, supports CUPED variance reduction, sequential testing, multi-armed bandits.
- **Self-hostable**: full feature set free if you run it yourself.
- **GitOps-friendly**: experiments definable as YAML in your repo; CI flows for review.
- **Connect any data warehouse**: BigQuery, Snowflake, Postgres, ClickHouse — analyzes events at the source instead of forcing them through their pipeline.
- **Smaller blast radius**: not also doing analytics + replay + flags + heatmaps — does experimentation specifically very well.

### Why pick PostHog over GrowthBook

- **Single tool** for analytics + flags + experiments + replays.
- **Zero setup** — no warehouse to wire up; events live in PostHog.
- **Larger community**, more tutorials.

### GrowthBook setup

```bash
npm i @growthbook/growthbook-react@1
```

```tsx
// app/providers.tsx
'use client';
import { GrowthBook, GrowthBookProvider } from '@growthbook/growthbook-react';
import { useEffect, useState } from 'react';

export function GrowthBookProviderClient({ children, attributes }: { children: React.ReactNode; attributes: { id: string; email?: string } }) {
  const [gb] = useState(() => new GrowthBook({
    apiHost: process.env.NEXT_PUBLIC_GROWTHBOOK_API_HOST!,
    clientKey: process.env.NEXT_PUBLIC_GROWTHBOOK_CLIENT_KEY!,
    enableDevMode: process.env.NODE_ENV === 'development',
    trackingCallback: (experiment, result) => {
      // Send the exposure event to your analytics
      posthog.capture('$experiment_started', {
        experimentId: experiment.key,
        variantId: result.key,
      });
    },
  }));

  useEffect(() => {
    gb.setAttributes(attributes);
    gb.loadFeatures();
  }, [gb, attributes]);

  return <GrowthBookProvider growthbook={gb}>{children}</GrowthBookProvider>;
}
```

```tsx
'use client';
import { useFeatureValue } from '@growthbook/growthbook-react';

export function CTAButton() {
  const variant = useFeatureValue('signup-cta-copy', 'control');
  const COPY = { control: 'Sign up', urgency: 'Sign up — limited' };
  return <a href="/signup">{COPY[variant as keyof typeof COPY]}</a>;
}
```

### Server-side resolution with GrowthBook

For Server Components, use the `growthbook@1` Node SDK:

```ts
// lib/growthbook.ts
import { GrowthBookClient } from '@growthbook/growthbook';
import { auth } from '@/auth';

export async function getGrowthBook() {
  const session = await auth();
  const gb = new GrowthBookClient({
    apiHost: process.env.GROWTHBOOK_API_HOST!,
    clientKey: process.env.GROWTHBOOK_CLIENT_KEY!,
  });
  await gb.init({
    payload: undefined,                                    // fetches latest
  });
  const userContext = gb.createUserContext({
    attributes: session?.user ? { id: session.user.id, email: session.user.email } : { id: 'anon' },
  });
  return userContext;
}
```

```tsx
// app/page.tsx
import { getGrowthBook } from '@/lib/growthbook';

export default async function Page() {
  const gb = await getGrowthBook();
  const variant = gb.getFeatureValue('signup-cta-copy', 'control');
  return <a href="/signup">{COPY[variant as keyof typeof COPY]}</a>;
}
```

### Connecting GrowthBook to your data warehouse

The GrowthBook killer feature: it doesn't store raw events. You point it at your existing data warehouse:

```sql
-- In GrowthBook UI: define the metric
-- "Signup completed" =
SELECT
  user_id,
  timestamp,
  1 as value
FROM events
WHERE event_name = 'signup_completed'
```

GrowthBook computes lift, posteriors, and recommendations against this query. **Same warehouse you already pay for, no double-counting.** Unique to GrowthBook in 2026; PostHog ties experiments to PostHog events.

---

## 6. Real Experiment Patterns

### Pattern A — Server-Component variant gate

The cleanest pattern for any experiment that affects layout or copy:

```tsx
// app/(marketing)/pricing/page.tsx
import { pricingPageVariant } from '@/flags';

export default async function PricingPage() {
  const variant = await pricingPageVariant(); // 'control' | 'simplified' | 'enterprise-focus'
  if (variant === 'simplified') return <SimplifiedPricing />;
  if (variant === 'enterprise-focus') return <EnterprisePricing />;
  return <ControlPricing />;
}
```

### Pattern B — Inline branch inside a Server Component

For tiny variants (one button, one heading), inline ternaries are fine:

```tsx
const heroHeadline = await heroVariant();
return (
  <h1>
    {heroHeadline === 'urgency' ? 'Ship faster — limited beta' : 'Build faster'}
  </h1>
);
```

### Pattern C — Multi-step funnel experiment

Test signup form variant + landing CTA in one experiment by tying them through the same flag value:

```ts
export const onboardingVariant = flag<'control' | 'streamlined'>({
  key: 'exp.onboarding-streamlined',
  defaultValue: 'control',
  decide: postHog({ ... }),
});
```

```tsx
// Both pages read the same flag
// app/(marketing)/page.tsx
const variant = await onboardingVariant();
return <CTA copy={variant === 'streamlined' ? 'Get started' : 'Sign up free'} />;

// app/signup/page.tsx
const variant = await onboardingVariant();
return variant === 'streamlined' ? <OneFieldSignup /> : <FullSignup />;
```

Track `signup_completed` as the primary metric. The user is in a single bucket across both touchpoints, so the experiment measures the **end-to-end funnel**.

### Pattern D — Holdout group

For a feature you're confident in but want to verify the long-term impact:

```
Audience: 100% of new users
Variant: 95% see new feature ('treatment')
Variant: 5% never see it ('holdout')
Run for: 4–8 weeks
Primary metric: 30-day retention
```

The 5% holdout is your "what would have happened" counterfactual. Used by mature SaaS to verify the cumulative impact of multiple shipped features.

### Pattern E — Multivariate (factorial) test

Two changes you want to test independently. Don't run two A/B tests; run one 2×2 factorial:

```
Variant A: control headline + control CTA
Variant B: control headline + new CTA
Variant C: new headline + control CTA
Variant D: new headline + new CTA
```

PostHog's "Multivariant feature flag" supports this. Analysis is harder (you're testing main effects + interaction); for most teams, two sequential 2-arm tests are easier to reason about and only a bit slower.

---

## 7. Picking the Right Metric

The single most consequential decision in an A/B test is **which metric you optimize**.

### Primary metric

The single number the test will be judged on. Examples:
- **Signup conversion**: `signup_completed / page_views`.
- **Revenue per visitor**: `total_revenue / unique_visitors` (continuous, harder to power).
- **30-day retention**: `users_active_day_30 / users_signed_up`.

### Guardrail metrics

Things you don't want to wreck while improving the primary. Track:
- **Engagement**: time-in-app, sessions per user.
- **Reliability**: error rate, p95 latency.
- **Per-segment**: free vs paid users (don't cannibalize paid by improving free).
- **Cancellation rate** (for subscription products).

PostHog and GrowthBook both let you attach guardrail metrics to an experiment. **A "winning" variant that tanks a guardrail is a losing variant.**

### Proxy vs north-star

Be honest about what you're measuring:
- **North star**: long-term retention, LTV, churn.
- **Proxy**: signup, click-through, time on page.

Most experiments use proxies because north-star metrics take months to settle. **A proxy lift doesn't always translate.** Treat proxy wins as hypotheses that need post-launch tracking against the north star.

---

## 8. Real Implementation — Worked Example

### The hypothesis

"Showing social proof on the pricing page increases signup conversion among free-tier visitors."

### Pre-test setup

| Item | Value |
|------|-------|
| Audience | Visitors to `/pricing` who are not logged in |
| Primary metric | `signup_completed` within 7 days of `/pricing` visit |
| Baseline conversion | 4% (from analytics) |
| Minimum detectable effect (MDE) | 15% relative lift (4% → 4.6%) |
| Power | 80% |
| Significance | 95% Bayesian probability-to-be-best |
| Required sample size per arm | ~28,000 users (computed by PostHog) |
| Expected duration | 3 weeks at current traffic |
| Variants | `control`, `with-social-proof` |

### Define the flag

```ts
// flags.ts
export const pricingSocialProof = flag<'control' | 'with-social-proof'>({
  key: 'exp.pricing-social-proof',
  defaultValue: 'control',
  decide: postHog({ apiKey: process.env.POSTHOG_KEY!, host: process.env.POSTHOG_HOST! }),
  identify: async () => {
    const session = await auth();
    if (session?.user) return { distinctId: session.user.id };
    // Anonymous — use the cookie distinct_id PostHog set on first visit
    return undefined;
  },
});
```

### Implement the variant

```tsx
// app/(marketing)/pricing/page.tsx
import { pricingSocialProof } from '@/flags';
import { SocialProof } from '@/components/social-proof';

export default async function PricingPage() {
  const variant = await pricingSocialProof();
  return (
    <main>
      <h1>Pricing</h1>
      {variant === 'with-social-proof' && <SocialProof />}
      <PricingTable />
    </main>
  );
}
```

### Convert flag to experiment in PostHog UI

1. Set primary metric: `signup_completed`.
2. Add guardrail: `7d_retention` (don't nuke paid users by chasing free signups).
3. Set MDE: 15%.
4. Enable Bayesian analysis.
5. Set duration: 3 weeks minimum, monitor weekly with sequential testing.
6. Launch.

### Monitor

Weekly check-ins:
- **Week 1**: insufficient sample. Don't decide.
- **Week 2**: sample sufficient. Bayesian posterior shows 78% probability `with-social-proof` wins. **Below the 95% threshold; keep running.**
- **Week 3**: 96% probability winner. Ship to 100%.

### Conclude and clean up

```
Ship: ramp to 100% over 3 days.
Verify: error rate, retention, latency stay flat.
Code cleanup: delete the variant branch + flag in PostHog.
```

Document the result in a learnings doc — even null results are valuable institutional knowledge.

---

## 9. The Anti-Patterns That Kill Experiments

### Anti-pattern 1 — peeking and stopping early

> "Looking great after week 1, let's call it!"

If you didn't pre-register the early-stop rule (or use sequential testing), you've increased your false-positive rate. Real example: 30% of "wins" called early are actually noise.

**Fix**: Pre-declare duration. Use a tool with sequential testing (Statsig, GrowthBook 3) if you must monitor.

### Anti-pattern 2 — running with insufficient power

> "Inconclusive — we'll just go with our gut."

You ran a test with 500 users per arm. You can only detect ~30% relative lifts. The product question was "does this 5% lift exist?" — you couldn't answer it either way.

**Fix**: Calculate sample size before launching. If insufficient traffic, pick a different test or batch experiments.

### Anti-pattern 3 — bucketing inconsistency

> "Why does this user keep flipping between variants?"

The flag's `distinctId` is unstable (per-session UUID, IP-based). Each refresh re-buckets the user. Their experience is broken; your data is junk.

**Fix**: Stable user-level `distinctId`. Use cookies for anonymous users. Alias to user ID after sign-in.

### Anti-pattern 4 — the metric isn't measured at the right point

> "Signup_completed event fires when the user lands on the dashboard. But the issue is they leave the dashboard quickly."

The metric measured a proxy that didn't capture the goal. Real conversion was different from event-fire.

**Fix**: Map the funnel before designing the test. Pick a metric tied to the actual desired outcome.

### Anti-pattern 5 — testing too many things at once

> "We're A/B testing the headline AND the new pricing structure AND the new layout."

You can't attribute the result to any single change.

**Fix**: One independent variable per experiment. If it must be coupled (e.g., a redesign), accept that you're testing the **package**, not its components.

### Anti-pattern 6 — Simpson's paradox / segmentation traps

> "Variant A wins overall, but loses for both desktop AND mobile users."

This happens when traffic mix differs between arms. The test result hides a per-segment loss.

**Fix**: Always check key segments. PostHog and GrowthBook show per-segment breakdowns by default.

### Anti-pattern 7 — running on contaminated traffic

> "Our marketing team launched a Twitter ad mid-experiment."

The new traffic source bucketed disproportionately into one arm by chance. Result is meaningless.

**Fix**: Stable traffic during the experiment. If you must launch a campaign, segment traffic source in analysis.

---

## 10. Auditing an Experiment Before Shipping

Before declaring a variant "the winner" and shipping to 100%:

- [ ] **Sample size hit**: at or above pre-declared.
- [ ] **Duration ≥ 2 weeks** to capture weekly seasonality.
- [ ] **Posterior probability ≥ 95%** (Bayesian) or **p < 0.05** (frequentist).
- [ ] **Guardrails OK**: retention, errors, latency, paid-tier metrics flat or up.
- [ ] **Per-segment check**: no segment regressed by > 10%.
- [ ] **Holdout sanity check**: results consistent with previous similar experiments.
- [ ] **Pre-registered hypothesis**: result matches what you said you were testing for.
- [ ] **Code is reviewed**: deletion of variant branches doesn't introduce bugs.
- [ ] **Cleanup ticket**: filed to remove the flag within 30 days.

If any item fails, either keep running or kill the experiment. Don't ship "leaning positive" results — they're noise.

---

## 11. Local Dev and Testing for Experiments

### Variant override in dev

Same Vercel Flags SDK toolbar pattern as `12.quality-at-scale/01-feature-flags.md`:

```bash
# .env.local
FLAG_EXP_SIGNUP_CTA_COPY=urgency
```

Or use the toolbar to flip variants per browser session.

### E2E coverage for both arms

```ts
// e2e/pricing.spec.ts
import { test } from '@playwright/test';

for (const variant of ['control', 'with-social-proof']) {
  test(`pricing page renders correctly — ${variant}`, async ({ page }) => {
    await page.context().setExtraHTTPHeaders({
      'x-vercel-flag-overrides': `exp.pricing-social-proof=${variant}`,
    });
    await page.goto('/pricing');
    if (variant === 'with-social-proof') {
      await expect(page.getByTestId('social-proof')).toBeVisible();
    }
  });
}
```

Both arms must pass before launching the experiment.

### Visual regression per variant

Pair with **Storybook 8.3** + **Chromatic** (covered in `12.quality-at-scale/04-storybook.md`). Each variant gets its own snapshot story; visual diffs catch regressions in either arm.

---

## 12. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Sample size ignored, results "inconclusive" | Calculate before launch; don't run if traffic insufficient |
| Peeking and stopping early inflates false positives | Pre-declare duration; use sequential testing if you need to monitor |
| Logged-in vs logged-out users in same experiment | Pick one audience; cross-state experiments have inconsistent assignment |
| User on mobile becomes user on desktop, gets different variant | Use a stable `distinctId` (user ID), not session/device-specific |
| Marketing campaign mid-experiment skews results | Avoid coincident campaigns; segment traffic source if unavoidable |
| Engineering ships another related feature mid-test | Freeze related releases during the experiment window |
| Result wins on conversion but tanks revenue per signup | Add revenue/LTV as guardrail metric |
| Variant code is buggy, conversion drops because of bug not design | Test both arms in QA before launch; visual regression in CI |
| Experiment runs forever | Set a hard end date and a "no decision = ship control" default |
| Flag deleted while users still in cohort | Set rollout to 100% control before deleting; never yank a flag mid-experiment |
| Multiple comparisons inflate false positives | Bonferroni correct, or fewer arms |
| Network latency differs between variants (one fetches more data) | Standardize render budget; speed is a confound |
| Variant tested only on a/b traffic, not real product | Avoid synthetic traffic in test cohorts |
| Result published before posterior settled | Wait for the credible interval to narrow |

---

## 13. Decision Tree

```
Are you running an A/B test?
│
├── Yes → continue
└── No, just rolling out a feature → see 12.quality-at-scale/01-feature-flags.md (use a flag, not an experiment)

What tool?
│
├── Already on PostHog → PostHog Experiments (default 2026)
├── Already on LaunchDarkly → LaunchDarkly Experimentation (add-on)
├── Want OSS, Bayesian, warehouse-native → GrowthBook 3
├── Want strong sequential testing + free tier → Statsig
└── Marketing-led, visual editor needed → Optimizely Web / VWO

What stats framework?
│
├── Bayesian (default in modern tools) → easier to interpret, peeking-tolerant
└── Frequentist → matches academic conventions; pre-register sample size

How long to run?
│
├── At least 2 weeks → capture weekly seasonality
├── Or until pre-declared sample size → stop sooner if sequential testing supports it
└── Hard maximum (e.g., 8 weeks) → kill inconclusive tests

What metric?
│
├── Primary (the one you optimize)
└── Guardrails (3–5 things you don't want to wreck)

Variants?
│
├── 2 (control + treatment) → cleanest analysis
├── 3–4 → adjust significance threshold for multiple comparisons
└── > 4 → consider multi-armed bandit instead
```

---

## 14. What This Topic Connects To

- **`12.quality-at-scale/01-feature-flags.md`** — every A/B test is a feature flag with stats on top.
- **`10.production/04-observability.md`** — guardrail metrics live in your observability stack.
- **`12.quality-at-scale/03-e2e-playwright.md`** — Playwright covers both arms in CI.
- **`12.quality-at-scale/04-storybook.md`** — Chromatic catches per-variant regressions.
- **`07.nextjs/03-data-fetching.md`** — Server Action handlers should track conversion events.
- **`08.ecosystem/04-real-project.md`** — adding "experiment on the new dashboard layout" to the Task Manager is a 1-day PR.

---

## Summary

| Tool | Pick when |
|------|-----------|
| **PostHog Experiments** | Default for engineering-led teams in 2026 |
| **GrowthBook 3** | OSS, Bayesian-first, warehouse-native |
| **Statsig** | Strong free tier, sequential testing, multi-armed bandits |
| **LaunchDarkly Experimentation** | Already on LaunchDarkly; enterprise audit |
| **Optimizely / VWO** | Marketing-led teams with a visual editor |

| Rule | Why |
|------|-----|
| Calculate required sample size before launching | Insufficient power = inconclusive results no matter what |
| Run for ≥ 2 weeks | Capture weekly seasonality |
| Pre-declare duration; don't peek (or use sequential testing) | Avoid inflated false-positive rate |
| Use Bayesian framework when available | Outputs are easier to communicate |
| Stable per-user `distinctId` | Inconsistent bucketing wrecks data |
| Always include guardrail metrics | A "winning" variant that tanks retention is a losing variant |
| One independent variable per experiment | Otherwise you can't attribute the result |
| Clean up the flag within 30 days of launch | Stale experiments are runtime liabilities |

---

## Further reading

- [PostHog Experiments docs](https://posthog.com/docs/experiments)
- [GrowthBook docs](https://docs.growthbook.io/)
- [Statsig — Statistical methodology](https://docs.statsig.com/experiments/methodology/)
- [Evan Miller — A/B sample size calculator](https://www.evanmiller.org/ab-testing/sample-size.html)
- [Sequential testing — mSPRT (Optimizely paper)](https://www.optimizely.com/insights/blog/peeking-problem/)
- [Ron Kohavi — Trustworthy Online Controlled Experiments](https://experimentguide.com/) — the canonical book
- [Vercel Flags SDK + experimentation](https://flags-sdk.dev/)
- [PostHog — Common A/B testing pitfalls](https://posthog.com/tutorials/ab-testing-mistakes)
