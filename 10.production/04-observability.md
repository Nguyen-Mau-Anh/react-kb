# Production — 04. Observability: Sentry 8, OpenTelemetry, Vercel Analytics, Log Shipping

> **What / Why / How** — three pillars: errors (Sentry), performance/RUM (Vercel Analytics + Speed Insights or PostHog), and traces+logs (OpenTelemetry). Wire all three from day one, not after the first incident.

---

## 1. The Three Pillars of Observability

You need answers to three different questions when something goes wrong:

| Pillar | Question it answers | Best React/Next.js tool in 2026 |
|--------|----------------------|---------------------------------|
| **Errors** | "What broke and where?" | `@sentry/nextjs@8` |
| **Real User Monitoring (RUM)** | "How is the app actually performing for users?" (LCP, INP, CLS, custom events) | Vercel Analytics + Vercel Speed Insights, or `posthog-js@1.155` |
| **Traces + logs** | "What happened on the server during this request?" | OpenTelemetry (`@vercel/otel@1`, `@opentelemetry/sdk-node@0.50`) → DataDog / Honeycomb / Grafana Tempo |

These aren't interchangeable. Sentry is great at errors but mediocre at structured traces; OpenTelemetry is great at traces but bad at React Web Vitals. Use the right tool per pillar.

---

## 2. The Real Choices in 2026

| Pillar | Tool | npm | Best for | Pricing |
|--------|------|-----|----------|---------|
| Errors | **Sentry 8** | `@sentry/nextjs@8` | Default for any production React app | Free 5K events/mo; $26+/mo |
| Errors | **Bugsnag 8** | `@bugsnag/js@8` | Sentry alternative; cleaner UI | Per event |
| Errors | **Rollbar 4** | `rollbar@2` | Legacy choice; some enterprise shops | Per event |
| RUM | **Vercel Analytics + Speed Insights** | `@vercel/analytics@1.3`, `@vercel/speed-insights@1.0` | Default if you deploy on Vercel | Included with Vercel Pro |
| RUM | **PostHog** | `posthog-js@1.155` | Self-host capable, product analytics + feature flags + RUM | Free 1M events/mo |
| RUM | **Plausible / Fathom / Umami** | `plausible-tracker@2`, `umami@2` | Page-view-only, GDPR-friendly | Per-month flat |
| RUM | **Datadog RUM** | `@datadog/browser-rum@5` | Already on Datadog | Per-session |
| RUM | **New Relic Browser** | `newrelic@11` | Already on New Relic | Per-session |
| Traces | **OpenTelemetry** | `@vercel/otel@1`, `@opentelemetry/api@1`, `@opentelemetry/sdk-node@0.50` | Vendor-neutral default | Free spec; pay your backend |
| Traces backend | **Honeycomb** | OTLP receiver | Best DX for trace exploration | $0–$130/mo |
| Traces backend | **Grafana Tempo + Loki** | OTLP receiver | Self-host, OSS-friendly | Free (you pay the host) |
| Traces backend | **Datadog APM** | `dd-trace@5` (or OTLP) | Enterprise default | $$$ |
| Logs | **`pino@9`** + **Better Stack / Axiom / Logtail** | `pino@9`, `@axiomhq/pino@2`, `@logtail/pino@0.5` | Structured logs shipped to a service | $20–100/mo |
| Logs | **Vercel Logs / Vercel Log Drains** | Built-in | Already on Vercel | Included on Pro |
| Logs | **`winston@3`** | `winston@3` | Older codebases | Free |

**The 80% recipe for a Next.js 14 + Vercel app in 2026:**
1. **Errors** — `@sentry/nextjs@8`.
2. **RUM** — `@vercel/analytics@1.3` + `@vercel/speed-insights@1.0` (or `posthog-js@1.155` if you also want product analytics).
3. **Traces** — `@vercel/otel@1` exporting to Honeycomb (or Datadog if your team is there).
4. **Logs** — `pino@9` + Vercel Log Drains to Axiom/Better Stack/Logtail.

Set up all four from day one. Diagnosing your first production incident with no observability is a bad day.

---

## 3. Errors — Sentry 8 for Next.js 14

### Setup

```bash
npx @sentry/wizard@4 -i nextjs
```

The wizard:
- Adds `@sentry/nextjs@8`.
- Generates `sentry.client.config.ts`, `sentry.server.config.ts`, `sentry.edge.config.ts`.
- Wraps `next.config.js` with `withSentryConfig` for source map upload.
- Adds an example route + a `try-throw` to verify it works.

### Three configs because three runtimes

```ts
// sentry.client.config.ts — runs in the browser
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 0.1,                 // 10% of transactions for perf
  replaysSessionSampleRate: 0.1,         // 10% of sessions get a Replay
  replaysOnErrorSampleRate: 1.0,         // 100% of sessions WITH errors get a Replay
  integrations: [
    Sentry.replayIntegration({
      maskAllText: true,                 // ⚠ redacts user text from session replays
      blockAllMedia: true,
    }),
  ],
});
```

```ts
// sentry.server.config.ts — runs in Node Route Handlers / Server Components
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  tracesSampleRate: 0.1,
});
```

```ts
// sentry.edge.config.ts — runs in middleware and Edge route handlers
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  tracesSampleRate: 0.1,
});
```

Each runtime is its own initialization because they bundle differently — Edge can't import Node-only Sentry plugins.

### Source map upload — the single most-skipped step

Without source maps, errors show minified `t.k.x is not a function`. With them, you see real file paths and line numbers.

```js
// next.config.js
const { withSentryConfig } = require('@sentry/nextjs');

module.exports = withSentryConfig(
  { /* your normal next.config */ },
  {
    org: 'your-org',
    project: 'web-app',
    authToken: process.env.SENTRY_AUTH_TOKEN, // ⚠ from Vercel env, not committed
    silent: !process.env.CI,
    widenClientFileUpload: true,
    hideSourceMaps: true,                     // sourcemaps NOT served to browsers in prod
  }
);
```

Run with `SENTRY_AUTH_TOKEN` in CI/Vercel and source maps upload on every build. **This is the difference between Sentry being usable and useless.**

### Capturing context — what to attach

```ts
import * as Sentry from '@sentry/nextjs';

// In a Route Handler or Server Component:
Sentry.setUser({ id: session.user.id, email: session.user.email });
Sentry.setTag('feature', 'checkout');
Sentry.setContext('cart', { itemCount: cart.items.length, total: cart.total });

try {
  await processPayment();
} catch (err) {
  Sentry.captureException(err, { extra: { orderId } });
  throw err;
}
```

**Don't** attach raw user input (passwords, PII) to Sentry events. Sentry has a `beforeSend` filter for scrubbing — set it up in `sentry.client.config.ts`:

```ts
Sentry.init({
  beforeSend(event) {
    if (event.request?.headers) {
      delete event.request.headers['authorization'];
      delete event.request.headers['cookie'];
    }
    return event;
  },
});
```

### Session Replay — the killer Sentry feature in 2026

Sentry's Session Replay records the user's clicks, page navigations, network calls, and DOM changes. When an error fires, you get a video-like replay of the 30 seconds before it happened.

- Set `replaysSessionSampleRate: 0.1` to record 10% of *all* sessions.
- Set `replaysOnErrorSampleRate: 1.0` to **always** record sessions where an error occurred.

`maskAllText: true` is critical — without it, you're recording everything users type, which is a compliance nightmare. Mask by default, unmask specific elements with `data-sentry-unmask`:

```tsx
<p data-sentry-unmask>Plan: {plan.name}</p>
```

### Error boundaries with Sentry

```tsx
// app/error.tsx — covered in 07.nextjs/02-routing.md
'use client';
import * as Sentry from '@sentry/nextjs';
import { useEffect } from 'react';

export default function Error({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  useEffect(() => {
    Sentry.captureException(error);
  }, [error]);

  return (
    <div>
      <h2>Something went wrong</h2>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

For a top-level fallback, `app/global-error.tsx` does the same job at the root.

### Checking it works

```bash
curl http://localhost:3000/api/sentry-test  # the wizard generates this route
```

If a Sentry event appears in your dashboard within ~30 seconds, you're wired up.

---

## 4. RUM — Vercel Analytics + Speed Insights

### What

Two separate packages from Vercel:

| Package | Tracks | Bundle |
|---------|--------|--------|
| `@vercel/analytics@1.3` | Page views, custom events (clicks, conversions), audience segmentation | ~3 KB |
| `@vercel/speed-insights@1.0` | Core Web Vitals (LCP, INP, CLS, TTFB, FCP) per route, real users | ~2 KB |

Both ship a single `<Analytics />` and `<SpeedInsights />` component you mount in your root layout.

### Setup

```bash
npm i @vercel/analytics@1.3 @vercel/speed-insights@1.0
```

```tsx
// app/layout.tsx
import { Analytics } from '@vercel/analytics/react';
import { SpeedInsights } from '@vercel/speed-insights/next';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <Analytics />
        <SpeedInsights />
      </body>
    </html>
  );
}
```

That's it. Vercel's dashboard now shows route-level performance (covered in `06.testing-perf/02-performance.md`'s Core Web Vitals targets) and visit metrics. **No env vars, no consent screen, no setup beyond mounting the component.**

### Custom events

```tsx
'use client';
import { track } from '@vercel/analytics';

<button onClick={() => { signup(); track('signup_completed', { plan: 'pro' }); }}>
  Sign up
</button>
```

Custom events show up in the Analytics dashboard and can be filtered/segmented like any other metric.

### Why Vercel Analytics wins by default for Vercel-hosted apps

- **Zero config.** No tracking ID, no script tag, no domain verification.
- **Privacy-respecting.** Cookieless by default. GDPR-compliant out of the box.
- **Real user metrics, not synthetic.** Lighthouse-CI tells you what your build can achieve; Vercel Speed Insights tells you what real users actually see.
- **Per-route P75 / P90 / P99 percentiles.** You see which page has the worst INP, not just an aggregate number.

### When to pick PostHog instead

`posthog-js@1.155` (or `posthog-node@4` for server) is the strongest open-source product-analytics + RUM combo:

```bash
npm i posthog-js@1.155
```

```tsx
// app/posthog-provider.tsx
'use client';
import posthog from 'posthog-js';
import { PostHogProvider } from 'posthog-js/react';
import { useEffect } from 'react';

if (typeof window !== 'undefined') {
  posthog.init(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
    api_host: '/ingest',                          // proxy through your own domain
    person_profiles: 'identified_only',
    capture_pageview: false,                      // we'll do it manually for App Router
  });
}

export function Providers({ children }: { children: React.ReactNode }) {
  return <PostHogProvider client={posthog}>{children}</PostHogProvider>;
}
```

Pick PostHog when you also need:
- **Feature flags** (covered in a future iteration).
- **A/B experiments** with built-in stats.
- **Funnel analysis** ("90% of users who clicked X reached Y").
- **Heatmaps + session recordings** (free at the open-source tier).
- **Self-hosting option.**

PostHog is also valid alongside Vercel Analytics — many teams use Vercel for Speed Insights (Web Vitals) and PostHog for everything else.

### Cookieless analytics — Plausible / Fathom / Umami

If you only want page-view counts and want to be GDPR-friendly without showing a banner:

| Tool | Bundle | Pricing |
|------|--------|---------|
| **Plausible** | `plausible-tracker@2` ~1.4 KB | $9–69/mo |
| **Fathom Analytics** | direct script tag, ~1.8 KB | $14–84/mo |
| **Umami** (self-hosted) | `@umami/node@0.5` | Free if you host |

For a marketing site that just needs page views and doesn't want banners: Plausible is the most-used in 2026.

---

## 5. Traces — OpenTelemetry on Next.js 14

### What is OpenTelemetry?

A vendor-neutral standard for emitting **traces, metrics, and logs**. A trace is a tree of "spans" — each span is one unit of work (a request, a DB call, a queue publish). The output (OTLP) ships to any compatible backend: Honeycomb, Datadog, Grafana Tempo, AWS X-Ray, Jaeger.

### Setup with `@vercel/otel@1`

```bash
npm i @vercel/otel@1 @opentelemetry/api@1
```

```ts
// instrumentation.ts (project root, not in app/)
import { registerOTel } from '@vercel/otel';

export function register() {
  registerOTel({ serviceName: 'web-app' });
}
```

```js
// next.config.js
module.exports = {
  experimental: { instrumentationHook: true }, // not needed in Next.js 15+
};
```

That's the whole setup. Next.js auto-instruments fetches, route handlers, server actions, and React rendering. Spans flow to whatever OTLP receiver you configured via env vars.

### Sending to Honeycomb

```bash
# .env.production
OTEL_EXPORTER_OTLP_ENDPOINT=https://api.honeycomb.io/v1/traces
OTEL_EXPORTER_OTLP_HEADERS=x-honeycomb-team=YOUR_TEAM_KEY
```

Honeycomb's UI is purpose-built for trace exploration: filter by span attributes, watch a heatmap of latencies, drill into outliers. **Best DX of the trace backends in 2026.**

### Sending to Datadog or Grafana Tempo

Set the OTLP endpoint to your Datadog Agent or Grafana OTel Collector. Same env vars, different backend.

### Custom spans

```ts
import { trace } from '@opentelemetry/api';

const tracer = trace.getTracer('web-app');

export async function processPayment(orderId: string) {
  return tracer.startActiveSpan('processPayment', async (span) => {
    span.setAttribute('order.id', orderId);
    try {
      await charge();
      span.setStatus({ code: 1 }); // OK
    } catch (err) {
      span.recordException(err as Error);
      span.setStatus({ code: 2, message: (err as Error).message });
      throw err;
    } finally {
      span.end();
    }
  });
}
```

Now `processPayment` shows as a span in every trace, with the `order.id` as a queryable attribute.

### Why OpenTelemetry over Sentry-only tracing

- **Vendor-neutral.** Switch backends (Honeycomb → Datadog → Grafana) without rewriting code.
- **First-class server tracing.** Auto-instrumentation of Prisma, Postgres, Redis, gRPC out of the box.
- **Pairs with logs.** Logs can carry the same `traceId`, making "find me the logs for this slow request" trivial.

Sentry's tracing works well for browser → API → DB request graphs but is weaker for multi-service backend traces. Use both: Sentry for errors + frontend traces; OpenTelemetry for backend service-to-service tracing.

---

## 6. Logs — Pino + Log Shipping

### Why structured logs

`console.log` from Server Components / Route Handlers ends up in the Vercel Functions log. Searchable, but not queryable, no aggregation, no alerts.

`pino@9` produces **JSON-structured** logs you can ship to Axiom, Better Stack (Logtail), Grafana Loki, Datadog, or any OTLP/log-receiver.

### Setup

```bash
npm i pino@9 pino-pretty@11
```

```ts
// lib/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  base: { service: 'web-app', env: process.env.VERCEL_ENV ?? 'dev' },
  ...(process.env.NODE_ENV === 'development' && {
    transport: { target: 'pino-pretty', options: { colorize: true } },
  }),
});
```

```ts
// app/api/orders/route.ts
import { logger } from '@/lib/logger';

export async function POST(req: Request) {
  const start = Date.now();
  try {
    const order = await createOrder(/* ... */);
    logger.info({ orderId: order.id, durationMs: Date.now() - start }, 'order.created');
    return Response.json(order);
  } catch (err) {
    logger.error({ err, durationMs: Date.now() - start }, 'order.failed');
    throw err;
  }
}
```

Real production rule: **log levels are events, not free text**. The first arg is structured fields; the second is a stable event name (`order.created`, `payment.failed`). You can then alert on `event:payment.failed AND status:5xx`.

### Shipping logs from Vercel — Log Drains

Vercel can stream all function logs to a target via **Log Drains**. Configure once in the Vercel dashboard:

- **Axiom** — `@axiomhq/pino@2` integration; real-time queryable.
- **Better Stack** (`@logtail/pino@0.5`) — formerly Logtail, generous free tier.
- **Datadog Logs** — for shops already on DD.
- **Grafana Loki** — open-source, self-hostable.

Log Drains are configured per-Vercel-project. No code change needed — Vercel ships every `console.log` and pino-emitted log.

### Correlating logs with traces

If you set the OpenTelemetry trace context, pino can include it:

```ts
import { trace } from '@opentelemetry/api';

const span = trace.getActiveSpan();
const traceId = span?.spanContext().traceId;
logger.info({ orderId, traceId }, 'order.processed');
```

Now in your log backend, click a `traceId` → jump to the full distributed trace in Honeycomb. The killer "what happened on this request?" workflow.

---

## 7. Tying It All Together — One Real Incident

Imagine a user reports "I can't check out." With the above stack:

1. **Sentry** → search by user email or session ID. Finds an error: `TypeError: Cannot read 'total' of undefined` at `processPayment.ts:42`.
2. **Sentry Session Replay** → watch the user click, see the cart appear empty before the click.
3. **Vercel Speed Insights** → check if the page was unusually slow (LCP > 4s often correlates with hydration bugs).
4. **OpenTelemetry trace** → look at the `processPayment` span. See the DB call took 8 seconds (bad cache invalidation).
5. **Logs** → query `event:payment.failed traceId:abc123` in Axiom. Get the full structured log with order ID.
6. **Diagnose**: a recent deploy added a feature flag that emptied the cart for users in a specific cohort. Roll back via flag toggle.

Total time to identify the root cause: ~5 minutes. Without observability: hours of guessing.

---

## 8. Cost Reality Check

Real monthly costs for a small SaaS (10K MAU, 1M API requests/mo) in 2026:

| Service | Plan | $/mo |
|---------|------|------|
| Sentry | Team plan (50K events) | $26 |
| Vercel Analytics + Speed Insights | Bundled with Vercel Pro | $0 (already paying for Vercel) |
| Honeycomb | Self-serve | $0–$100 (with sampling) |
| Axiom (logs) | Free tier | $0 (500 GB/mo retained) |
| **Total** | | **~$26–$130/mo** |

For larger apps (1M MAU, 100M requests):
- Sentry: $80–$500/mo (event volume).
- Datadog APM + Logs: $1000+/mo (per-host pricing adds up).
- Honeycomb Pro: $130–$500/mo with sampling.

**Sample aggressively from day one** (`tracesSampleRate: 0.1` is the standard) — full sampling is the #1 cost surprise.

---

## 9. Privacy & Compliance — Real Rules

| Concern | What to do |
|---------|------------|
| GDPR (EU users) | Use cookieless analytics (Plausible) or get explicit consent before loading Sentry/PostHog |
| CCPA (California users) | Same as GDPR for opt-out preferences |
| HIPAA (US healthcare) | Sign a BAA with Sentry / Datadog; mask all medical data in Session Replay; never log PHI |
| PCI DSS (payments) | Don't log card numbers; Stripe `Payment Element` handles this — covered in a future iteration |
| Right to be forgotten | Implement `Sentry.removeUser()` and `posthog.reset()` on user-data-deletion requests |

`maskAllText: true` and `blockAllMedia: true` in Sentry Replay are the floor, not the ceiling. **Default to over-masking.**

---

## 10. SLOs and Alerting — What's Worth Paging On

Don't alert on every error. Alert on **what users feel**:

| Signal | Threshold | Tool |
|--------|-----------|------|
| 5xx error rate | > 1% over 5 min | Sentry alert |
| Apdex (response time satisfaction) | < 0.85 over 10 min | OpenTelemetry trace + Grafana |
| LCP P75 | > 3 seconds | Vercel Speed Insights alert |
| INP P75 | > 300 ms | Vercel Speed Insights alert |
| Payment success rate | < 95% over 5 min | Sentry/Posthog custom alert |
| Auth failure rate | > 5% over 5 min | Custom log query |

Page on **5xx error rate spikes** and **auth/payment SLO violations**. Page-on-Sentry-error-volume is noise. Tune.

---

## 11. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Sentry stack traces are minified `t.k.x` | Source map upload didn't run. Set `SENTRY_AUTH_TOKEN` in CI; check Sentry's "Source Maps" tab |
| Edge route errors don't appear in Sentry | Forgot `sentry.edge.config.ts`; Edge runtime uses a different bundle |
| Session Replay records passwords | `maskAllText: true` AND mask form inputs explicitly |
| Vercel Analytics shows zero data | `<Analytics />` not mounted, OR ad-blocker is blocking; both common — test with a clean browser |
| OpenTelemetry traces missing | `experimental.instrumentationHook` not set in Next.js 14, OR backend OTLP endpoint wrong |
| Logs say `[object Object]` | Pino logs need structured first arg; you passed an object as the second positional `msg` |
| Sentry costs explode after launch | `tracesSampleRate: 1.0` was left on in prod. Drop to 0.1 |
| PostHog kills CLS / LCP scores | Loading the script blocks render. Use the lazy `posthog.init({ loaded })` pattern or PostHog's reverse-proxy `/ingest` route |
| Trace context not propagated to client → server | Use `@opentelemetry/instrumentation-fetch@0.50` and ensure the same OTel SDK runs in both contexts |
| Deploys overwrite Sentry releases | Sentry needs `SENTRY_RELEASE` env var set to the git SHA; the `withSentryConfig` plugin handles this if `VERCEL_GIT_COMMIT_SHA` is present |
| Production logs include dev-only `console.debug` | Set `LOG_LEVEL=info` in Vercel prod env |

---

## 12. Decision Tree

```
What pillar are you wiring?
│
├── Errors
│   ├── Default → @sentry/nextjs@8
│   └── Already paying for Datadog/New Relic → use their error tracking
│
├── RUM / Web Vitals / Page views
│   ├── On Vercel → @vercel/analytics + @vercel/speed-insights (no setup)
│   ├── Want product analytics + flags + recordings → posthog-js
│   ├── GDPR-strict marketing site → plausible-tracker
│   └── On Datadog → @datadog/browser-rum
│
├── Server traces (request → DB → external API)
│   ├── Vendor-neutral → @vercel/otel@1, ship to Honeycomb (best DX)
│   ├── Already on Datadog → dd-trace OR OTLP to Datadog
│   └── Self-host → Grafana Tempo via @vercel/otel
│
└── Logs
    ├── Vercel-hosted → pino + Vercel Log Drain to Axiom or Better Stack
    ├── Self-host → pino + Loki/Logstash via OTLP
    └── Datadog → pino + Datadog Agent
```

---

## 13. What This Topic Connects To

- **`07.nextjs/02-routing.md`** — `app/error.tsx` + `app/global-error.tsx` are the React boundaries you wire to `Sentry.captureException`.
- **`06.testing-perf/02-performance.md`** — Core Web Vitals (LCP, INP, CLS) are exactly what Vercel Speed Insights tracks.
- **`07.nextjs/04-auth.md`** — `Sentry.setUser(...)` after sign-in lets you correlate errors to real users.
- **`07.nextjs/06-api-routes.md`** — Route Handlers and Server Actions are auto-instrumented by `@vercel/otel`.
- **`08.ecosystem/04-real-project.md`** — adding observability to the Task Manager is a 1-hour PR using the patterns here.
- **`09.beyond-web/04-ai-integration.md`** — log AI token spend per user via `streamText`'s `onFinish` hook.

---

## Summary

| Pillar | Tool | Primary use |
|--------|------|-------------|
| Errors | `@sentry/nextjs@8` | Browser + server errors with Session Replay |
| RUM | `@vercel/analytics@1.3` + `@vercel/speed-insights@1.0` | Page views + Web Vitals |
| Product analytics | `posthog-js@1.155` | Funnels, heatmaps, flags, recordings |
| Privacy-friendly views | `plausible-tracker@2` | Cookieless page views |
| Traces | `@vercel/otel@1` → Honeycomb | Server-side trace exploration |
| Logs | `pino@9` + Vercel Log Drains → Axiom | Structured, queryable logs |
| Backend APM | OpenTelemetry → Datadog/Honeycomb | Service-to-service tracing |

| Rule | Why |
|------|-----|
| Wire all four pillars from day one | First incident shouldn't be the discovery moment |
| Always upload source maps to Sentry | Otherwise stacks are useless |
| Use `tracesSampleRate: 0.1` in prod | Full sampling is the #1 cost spike |
| `maskAllText: true` in Session Replay | Default to over-masking for compliance |
| Log levels are events (`order.created`), not sentences | Searchable, alertable, aggregatable |
| Carry `traceId` from spans into logs | One click jumps from log → full trace |
| Alert on user-felt SLOs, not raw error count | Sentry's noise floor is too high to page on |

---

## Further reading

- [Sentry Next.js docs](https://docs.sentry.io/platforms/javascript/guides/nextjs/)
- [Sentry Session Replay](https://docs.sentry.io/product/session-replay/)
- [Vercel Analytics docs](https://vercel.com/docs/analytics)
- [Vercel Speed Insights docs](https://vercel.com/docs/speed-insights)
- [@vercel/otel docs](https://vercel.com/docs/observability/otel-overview)
- [OpenTelemetry JS](https://opentelemetry.io/docs/languages/js/)
- [Honeycomb — getting started with Next.js](https://docs.honeycomb.io/getting-data-in/integrations/javascript/nextjs/)
- [Pino docs](https://github.com/pinojs/pino)
- [Vercel Log Drains](https://vercel.com/docs/observability/log-drains)
- [PostHog docs](https://posthog.com/docs/libraries/next-js)
