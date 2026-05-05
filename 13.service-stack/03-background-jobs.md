# Service Stack — 03. Background Jobs: Inngest 3 vs Trigger.dev 3 vs BullMQ 5 vs QStash 2

> **What / Why / How** — pick **`inngest@3`** as the 2026 default for event-driven jobs on serverless; **`@trigger.dev/sdk@3`** for the same niche with a different DX; **`bullmq@5`** when you self-host Redis and want full control; **`@upstash/qstash@2`** for the simplest HTTP-trigger model. Never run long jobs inline in a Next.js Route Handler — Vercel cuts them off at 10–300s.

---

## 1. Why Background Jobs Matter

Three real failure modes if you skip them:

| Scenario | What happens without a queue |
|----------|-------------------------------|
| Send a welcome email on signup | The user waits ~500ms for SMTP. Failures bubble up as 500s. |
| Process a 50MB image upload | The Next.js function times out at 10s; user sees "request failed." |
| Charge a customer + send receipt + sync CRM | Three sequential network calls; any failure means inconsistent state. |
| Re-index 100K products after a price update | The HTTP request that triggered it dies long before the job finishes. |
| Run a daily revenue summary | Your app needs `setInterval`; serverless functions don't run continuously. |

The pattern that fixes all of them: **the user's request returns immediately, the actual work runs in a separate process that retries on failure**.

---

## 2. The Real Choices in 2026

| Tool | Type | Best for | Pricing |
|------|------|----------|---------|
| **Inngest 3** (`inngest@3`) | Event-driven serverless | Default for new Next.js apps; multi-step, long-running, retries, fan-out | Free 50K runs/mo |
| **Trigger.dev 3** (`@trigger.dev/sdk@3`) | Event-driven serverless | Same niche; cleaner UI for some teams | Free tier; $$/mo for higher usage |
| **BullMQ 5** (`bullmq@5`) | Redis-backed queue | Self-hosted; you own the Redis + workers | Free (you pay for Redis + host) |
| **QStash 2** (`@upstash/qstash@2`) | HTTP-trigger queue (Upstash) | Simplest; runs *your* HTTP endpoints on a schedule or after a delay | Free 500 messages/day |
| **Vercel Cron** | Cron-only | Scheduled (not event-driven) tasks; runs your Route Handlers | Bundled with Vercel Pro |
| **Vercel Queues** (`@vercel/queues@beta`) | Native queue (preview) | Simpler than QStash for Vercel-only apps | TBD — preview |
| **AWS SQS + Lambda** | Pull-queue + serverless | AWS-native shops | Pay-per-message |
| **Cloudflare Queues** | Pull-queue + Workers | Cloudflare-native | Generous free tier |
| **Temporal** (`@temporalio/client@1`) | Durable workflow engine | Complex multi-day workflows; financial / healthcare | Self-host or Temporal Cloud |
| **Defer.run** | Cloud workflow service | Same niche as Inngest with different UX | Per-execution |

**The 2026 default**: **Inngest 3**. It hits the sweet spot for 95% of teams — runs as Next.js Route Handlers (no separate server), supports event-driven + scheduled, has multi-step workflows, retries, fan-out, and a generous free tier.

For **self-hosted Redis shops with experienced ops**: **BullMQ 5**. Battle-tested, tons of features, full control.

For **dead-simple "run this URL after 30 minutes"**: **QStash 2**.

For **already on Vercel + scheduled-only needs**: **Vercel Cron** is enough.

---

## 3. Why Inngest Wins by Default

### What

`inngest@3` is an event-driven background-job platform that runs **inside your Next.js app** — no separate server process. You define functions in your codebase that respond to events; Inngest's hosted infra orchestrates them.

The mental model:

```
Your code → inngest.send({ name: 'user.signed-up', data: {...} })
                                      ↓
                            Inngest hosted infra
                                      ↓
            HTTP POST → /api/inngest with the event payload
                                      ↓
        Your function runs (with retries, multi-step, observability)
```

The function lives **inside your Next.js project**. Deploy it like any route. **Inngest infra calls back to your URL when an event fires.**

### Setup

```bash
npm i inngest@3
```

```ts
// inngest/client.ts
import { Inngest } from 'inngest';

export const inngest = new Inngest({
  id: 'acme-app',
  eventKey: process.env.INNGEST_EVENT_KEY,
});
```

```ts
// inngest/functions/welcome-email.ts
import { inngest } from '../client';
import { sendEmail } from '@/lib/email/send';

export const sendWelcomeEmail = inngest.createFunction(
  { id: 'send-welcome-email', retries: 3 },
  { event: 'user.signed-up' },
  async ({ event, step }) => {
    // step.run is durable: each step's result is persisted; if a later step fails,
    // earlier steps are NOT re-executed.
    const user = await step.run('load-user', () =>
      db.user.findUniqueOrThrow({ where: { id: event.data.userId } })
    );

    await step.run('send-welcome', () =>
      sendEmail.welcome(user.email, { name: user.name })
    );

    await step.sleep('wait-3-days', '3d');

    await step.run('send-getting-started', () =>
      sendEmail.gettingStarted(user.email)
    );
  }
);
```

```ts
// app/api/inngest/route.ts
import { serve } from 'inngest/next';
import { inngest } from '@/inngest/client';
import { sendWelcomeEmail } from '@/inngest/functions/welcome-email';

export const { GET, POST, PUT } = serve({
  client: inngest,
  functions: [sendWelcomeEmail],
});
```

### Trigger from anywhere

```ts
// In a Server Action, Route Handler, or anywhere:
await inngest.send({
  name: 'user.signed-up',
  data: { userId: user.id },
});
```

The Server Action returns immediately. Inngest's hosted layer queues the event, then calls back to your `/api/inngest` route to run the function. If the function fails, Inngest retries with exponential backoff. **You don't manage any of that.**

### Killer features — `step.run`, `step.sleep`, `step.waitForEvent`

The **multi-step durable execution model** is what sets Inngest apart from a plain queue:

```ts
inngest.createFunction(
  { id: 'order-flow' },
  { event: 'order.placed' },
  async ({ event, step }) => {
    const charge = await step.run('charge-card', () => stripe.charge(event.data));

    await step.sleep('hold-30-min', '30m');                    // function suspends; resumes 30 min later

    const cancelled = await step.waitForEvent('check-cancelled', {
      event: 'order.cancelled',
      timeout: '30m',
      match: 'data.orderId',
    });

    if (cancelled) {
      await step.run('refund', () => stripe.refund(charge.id));
      return { status: 'refunded' };
    }

    await step.run('ship', () => warehouse.ship(event.data));
    return { status: 'shipped' };
  }
);
```

Each `step.run()` is **persisted**. If `ship` fails after `charge-card` succeeded, only `ship` retries. **The card isn't double-charged.**

`step.sleep('30m')` doesn't keep a process alive for 30 minutes. Inngest **suspends the workflow** and resumes it later — runs are stateless across the wait. This is why it works on serverless.

`step.waitForEvent` — pause until another event arrives. Used for human-approval workflows ("wait for `support.approved` event with matching ticket ID").

### Real use cases

| Use case | Inngest pattern |
|----------|-----------------|
| Welcome email on signup | Single-step `step.run('send', sendWelcome)` |
| 3-day onboarding drip | `step.run` → `step.sleep('1d')` → `step.run` × 3 |
| Image processing after upload | `step.run('resize')` + `step.run('thumbs')` + `step.run('mark-ready')` |
| Failed-payment dunning (3 retries over 7 days) | `step.run('charge')` → `step.sleep('1d')` → check status → `step.run('charge')` ... |
| Webhook fan-out (one event → 5 destinations) | One function per destination, all listening to the same event |
| Daily summary email | Cron-triggered: `{ cron: '0 9 * * *' }` |
| Long-running export | `step.run('start-job')` → poll with `step.sleep('30s')` until done |

### Why Inngest beats inline async

```ts
// ❌ Inline — Vercel Function times out at 10s; failed Stripe call leaves user in bad state
export async function POST(req: Request) {
  const { email } = await req.json();
  const user = await db.user.create({ data: { email } });
  await stripe.customer.create(...);                       // 200ms
  await sendEmail.welcome(email, ...);                      // 500ms
  await crm.syncContact(...);                               // 1500ms
  await trigger.notifyTeam(...);                            // 800ms
  return Response.json({ ok: true });
}

// ✅ Inngest — return immediately, do the rest in background with retries
export async function POST(req: Request) {
  const { email } = await req.json();
  const user = await db.user.create({ data: { email } });
  await inngest.send({ name: 'user.signed-up', data: { userId: user.id } });
  return Response.json({ ok: true });
}
```

The user gets a response in ~50ms instead of ~3000ms. Failures don't surface as 500s. Retries are automatic.

### Local dev — Inngest Dev Server

```bash
npx inngest-cli@latest dev
```

Spawns a local mock of the Inngest hosted infra at `localhost:8288`. Your `/api/inngest` registers automatically. Events fired with `inngest.send()` show up in a dashboard with full traces, replays, and step-by-step debugging. **The single best dev DX of any background-job tool.**

---

## 4. Trigger.dev 3 — The Inngest Alternative

### What

`@trigger.dev/sdk@3` is the same idea as Inngest with a different API surface. Big rewrite in v3 (March 2024); much-improved over v2.

### Setup

```bash
npm i @trigger.dev/sdk@3
npx trigger.dev@latest init
```

```ts
// trigger/welcome-email.ts
import { task } from '@trigger.dev/sdk/v3';
import { sendEmail } from '@/lib/email/send';

export const welcomeEmailTask = task({
  id: 'welcome-email',
  retry: { maxAttempts: 3 },
  run: async (payload: { userId: string }) => {
    const user = await db.user.findUniqueOrThrow({ where: { id: payload.userId } });
    await sendEmail.welcome(user.email, { name: user.name });
  },
});
```

```ts
// In a Server Action or Route Handler
import { tasks } from '@trigger.dev/sdk/v3';
import type { welcomeEmailTask } from '@/trigger/welcome-email';

await tasks.trigger<typeof welcomeEmailTask>('welcome-email', { userId: user.id });
```

### Inngest vs Trigger.dev — practical differences

| | Inngest 3 | Trigger.dev 3 |
|--|-----------|---------------|
| Mental model | Events + functions matching events | Tasks invoked by name |
| Multi-step durability | `step.run`, `step.sleep`, `step.waitForEvent` | `await ctx.wait.for(...)`, `await ctx.task.trigger(...)` |
| Default runtime | Your existing Next.js Route Handler | Trigger's own deployed runtime (Node + AWS Lambda) |
| Local dev DX | Excellent (`inngest-cli dev`) | Excellent (`trigger.dev dev`) |
| Dashboard | Production-grade | Production-grade |
| Pricing free tier (mid-2026) | 50K runs/mo | 10K runs/mo |
| Niche where it wins | Default; great event-driven model | Cleaner per-task DX; better long-running (>15min) job handling |

**Picking rule**: Inngest if you prefer the event-driven model; Trigger.dev if you prefer the task-invocation model. For most teams the choice is taste — both are excellent.

For **jobs that run longer than 15 minutes** (heavy ML inference, big exports, video processing): **Trigger.dev v3** runs them on its own infrastructure rather than your serverless functions, so it has the edge.

---

## 5. BullMQ 5 — The Self-Host Workhorse

### What

`bullmq@5` is a Redis-backed Node.js queue. You run a **worker process** alongside your app; it pulls jobs from Redis and processes them.

### When BullMQ wins

- **Already running Redis** for cache or sessions; reuse it.
- **Existing Node ops experience** — you know how to deploy and monitor processes.
- **No vendor cost** — pay only for Redis + the worker host.
- **High throughput** — millions of jobs per day on a single Redis instance.
- **Fine-grained control** — priorities, rate limits, repeatable jobs, delayed jobs, child jobs.

### Setup

```bash
npm i bullmq@5 ioredis@5
```

```ts
// queue.ts
import { Queue, Worker } from 'bullmq';
import IORedis from 'ioredis';

const connection = new IORedis(process.env.REDIS_URL!, { maxRetriesPerRequest: null });

export const emailQueue = new Queue('email', { connection });

// In a separate worker process (e.g., apps/worker)
new Worker(
  'email',
  async (job) => {
    if (job.name === 'send-welcome') {
      await sendEmail.welcome(job.data.email, { name: job.data.name });
    }
  },
  { connection, concurrency: 10 }
);
```

```ts
// In your app
await emailQueue.add('send-welcome', { email: user.email, name: user.name }, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 1000 },
  removeOnComplete: { age: 3600 },                 // keep 1 hour of completed jobs for debugging
});
```

### The required infrastructure

- **Redis** — Upstash, Railway, ElastiCache, or self-hosted.
- **Worker process** — long-running Node process (NOT serverless). Deploy on Railway, Fly.io, AWS ECS (covered in `07.nextjs/05-deployment.md`).
- **Monitoring** — Bull Board (`@bull-board/express@5`), Datadog, custom metrics.
- **Scaling** — multiple worker processes; BullMQ handles distribution.

**This is real infrastructure.** Worth it at scale; overkill for a 5-engineer SaaS that ships 10 background jobs a day. Inngest does the same thing with zero infra.

### When NOT to use BullMQ

- Vercel-only deployments — Vercel functions can't run long-lived workers.
- Small team without dedicated ops.
- You don't already have Redis.

---

## 6. QStash 2 — The HTTP-Trigger Queue

### What

`@upstash/qstash@2` is the simplest possible queue: Upstash makes an HTTP call to *your* URL after a delay or on a schedule. Your endpoint does the work.

### Setup

```bash
npm i @upstash/qstash@2
```

```ts
// In a Server Action or Route Handler
import { Client } from '@upstash/qstash';

const qstash = new Client({ token: process.env.QSTASH_TOKEN! });

await qstash.publishJSON({
  url: 'https://yourapp.com/api/jobs/process-image',
  body: { imageId: '123' },
  delay: 30,                                         // wait 30s before calling
  retries: 3,
});
```

```ts
// app/api/jobs/process-image/route.ts
import { verifySignatureAppRouter } from '@upstash/qstash/nextjs';

export const POST = verifySignatureAppRouter(async (req: Request) => {
  const { imageId } = await req.json();
  await processImage(imageId);
  return new Response(null, { status: 200 });
});
```

`verifySignatureAppRouter` checks the QStash signature header — without it, anyone could POST to your endpoint and trigger the job manually.

### Why QStash wins

- **Zero infrastructure** — no Redis, no workers, no client SDK at runtime (just an HTTP call).
- **Pay-per-message** — generous free tier (500/day).
- **Schedules + delays** built in.
- **Already on Vercel + Upstash Redis** — no new vendor.

### Why QStash loses

- **No multi-step workflows.** Each job is a single HTTP call. For "send email → wait 3 days → send another," you'd schedule three separate QStash messages with different delays.
- **No fan-out** — one event = one URL.
- **Limited per-job timeout** — your URL must respond within Vercel function limits (10–300s).

For "run this URL after 30 minutes" or "POST this every Monday morning": QStash. For complex workflows: Inngest.

---

## 7. Vercel Cron — Scheduled Only

```jsonc
// vercel.json
{
  "crons": [
    { "path": "/api/cron/daily-summary", "schedule": "0 9 * * *" }
  ]
}
```

```ts
// app/api/cron/daily-summary/route.ts
export async function GET(req: Request) {
  // Verify the request actually came from Vercel
  const authHeader = req.headers.get('authorization');
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return new Response('Unauthorized', { status: 401 });
  }

  await sendDailySummaries();
  return new Response('ok');
}
```

That's the entire setup. Vercel's infra calls your URL on the schedule. Free with Pro plan; max 100 cron jobs per project.

**When it's enough**: scheduled-only tasks (daily reports, hourly cleanup). **When it's not**: event-driven jobs (need Inngest or QStash); long-running jobs (need Trigger.dev or BullMQ); multi-step workflows.

For a small SaaS in 2026: **Vercel Cron + Inngest is the canonical combination**. Cron for scheduled, Inngest for events.

---

## 8. Real-World Patterns

### Pattern A — Welcome email + drip

```ts
inngest.createFunction(
  { id: 'onboarding' },
  { event: 'user.signed-up' },
  async ({ event, step }) => {
    await step.run('welcome', () => sendEmail.welcome(event.data));
    await step.sleep('wait-1-day', '1d');
    await step.run('day-1-tip', () => sendEmail.tip(event.data));
    await step.sleep('wait-2-days', '2d');
    await step.run('day-3-cta', () => sendEmail.cta(event.data));
  }
);
```

User signs up → drip plays out over 3 days → no infrastructure to manage.

### Pattern B — Stripe webhook fan-out

The Stripe webhook (covered in `11.capabilities/01-payments.md`) just fires events; multiple Inngest functions react.

```ts
// app/api/webhooks/stripe/route.ts
case 'checkout.session.completed':
  await inngest.send({ name: 'stripe.checkout.completed', data: event.data.object });
  break;
```

```ts
// inngest/functions/grant-access.ts
inngest.createFunction({ id: 'grant-access' }, { event: 'stripe.checkout.completed' }, ...);

// inngest/functions/send-receipt.ts
inngest.createFunction({ id: 'send-receipt' }, { event: 'stripe.checkout.completed' }, ...);

// inngest/functions/sync-crm.ts
inngest.createFunction({ id: 'sync-crm' }, { event: 'stripe.checkout.completed' }, ...);
```

One Stripe event → three independent jobs, each retrying independently. **Failure of one doesn't block the others.**

### Pattern C — Image upload pipeline

```ts
inngest.createFunction(
  { id: 'process-image', retries: 3 },
  { event: 'image.uploaded' },
  async ({ event, step }) => {
    const image = await step.run('load', () => db.image.findUniqueOrThrow({ where: { id: event.data.imageId } }));
    const optimized = await step.run('optimize', () => sharpProcess(image.s3Key, { width: 2000, format: 'webp' }));
    const thumbnail = await step.run('thumb', () => sharpProcess(image.s3Key, { width: 256, format: 'webp' }));
    await step.run('mark-ready', () => db.image.update({
      where: { id: image.id },
      data: { status: 'processed', variants: { optimized, thumbnail } },
    }));
  }
);
```

Triggered after upload completes (covered in `11.capabilities/02-file-uploads.md`). Each step retries independently; a transient sharp error doesn't restart the whole pipeline.

### Pattern D — Cron + event mix

```ts
// Daily 9am summary email
inngest.createFunction(
  { id: 'daily-summary' },
  { cron: '0 9 * * *' },
  async ({ step }) => {
    const users = await step.run('load-users', () => db.user.findMany({ where: { dailyDigest: true } }));
    for (const user of users) {
      // Fan out — Inngest queues each invocation independently
      await step.invoke('send-summary', {
        function: sendDailySummary,
        data: { userId: user.id },
      });
    }
  }
);
```

`step.invoke` triggers another Inngest function asynchronously — fan-out without holding the cron job open.

### Pattern E — Long-running export with progress

```ts
inngest.createFunction(
  { id: 'export-csv' },
  { event: 'export.requested' },
  async ({ event, step }) => {
    const job = await step.run('start', () => db.exportJob.create({ data: { userId: event.data.userId, status: 'running' } }));

    await step.run('process', async () => {
      const data = await db.bigTable.findMany({ where: { ownerId: event.data.userId } });
      const csv = stringify(data);
      const url = await uploadToR2(`exports/${job.id}.csv`, csv);
      await db.exportJob.update({ where: { id: job.id }, data: { status: 'done', url } });
    });

    await step.run('notify', () => sendEmail.exportReady(event.data.userId, job.id));
  }
);
```

The user POSTs to start the export → returns the `jobId` instantly → polls `GET /api/jobs/:id` (covered in `13.service-stack/01-trpc-deep.md`) → background job updates status → user sees "Done" + download link.

---

## 9. Observability — The Boring Discipline

Background jobs fail differently than HTTP handlers. Wire observability:

| Concern | Solution |
|---------|----------|
| Job failure rate | Inngest/Trigger dashboards show this; export to Datadog or Honeycomb via OTLP |
| Slow jobs | Each `step.run` is timed; alert when p99 > N ms |
| Stuck/lost jobs | Inngest retries automatically; `removeOnComplete` configured in BullMQ |
| Sentry per job | Wrap `step.run` body in a try/catch, `Sentry.captureException` |
| Per-user audit | Tag the event with `userId`; Inngest dashboard filters by tag |

```ts
inngest.createFunction(
  { id: 'risky-job', retries: 3 },
  { event: 'risky.event' },
  async ({ event, step }) => {
    Sentry.setUser({ id: event.data.userId });
    Sentry.setTag('inngest.fn', 'risky-job');
    try {
      await step.run('do-thing', () => riskyOp());
    } catch (err) {
      Sentry.captureException(err);
      throw err;                                     // re-throw so Inngest retries
    }
  }
);
```

Covered in `10.production/04-observability.md`.

---

## 10. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Long inline work in a Route Handler times out | Move to Inngest / Trigger / BullMQ; return early |
| Job retries cause duplicate side effects | Make handlers idempotent; check DB state before mutating |
| Inngest function not registering | `app/api/inngest/route.ts` not deployed, OR functions not in the `serve()` array |
| QStash signature verification fails | Use `verifySignatureAppRouter`; raw body required |
| BullMQ workers die silently | Add a process supervisor (PM2, Railway's auto-restart, systemd); export error metrics |
| Stripe webhook fans out but one consumer fails | Each Inngest function retries independently; failed ones don't block others |
| `step.sleep` longer than function-execution limit | Sleep doesn't keep the function alive; Inngest suspends and resumes — works correctly |
| Sentry tags lost between steps | `step.run` is a fresh execution; set tags inside each step or use a wrapper |
| Cron fires twice | Don't add the same cron in both Vercel Cron AND Inngest schedule |
| Long export blocks the dashboard | Run async; UI polls a status endpoint |
| Job runs before DB transaction commits | Use Stripe-style outbox pattern: insert event row in same TX as state change; emit later |
| Vercel function cold-start delays Inngest webhook | First job per cold-start is slow; pre-warm via cron ping or use Trigger.dev's own runtime |
| Lost events between deploys | Inngest queues survive deploys; BullMQ keeps Redis; no plain in-memory queue between deploys |

---

## 11. Decision Tree

```
What kind of background work?
│
├── Simple "run after delay" or scheduled cron-only
│   ├── On Vercel → Vercel Cron (scheduled) or QStash (delays)
│   └── Off Vercel → BullMQ delayed jobs
│
├── Event-driven, retries, multi-step workflows
│   ├── Default → Inngest 3 (event-driven model)
│   └── Long jobs (>15 min) → Trigger.dev 3 (own runtime)
│
├── High throughput, self-hosted, you own Redis + workers
│   → BullMQ 5
│
├── Complex multi-day workflows (financial, healthcare)
│   → Temporal
│
└── Already on AWS → SQS + Lambda

How to integrate?
│
├── On Vercel/Next.js
│   ├── Inngest 3 → Route Handler at /api/inngest
│   ├── Trigger.dev 3 → trigger.dev's own runtime, app calls SDK
│   ├── QStash → HTTP webhook to your route
│   └── BullMQ → won't fit; need a separate worker on Railway/Fly
│
└── Self-hosted Node app
   └── BullMQ + Redis runs naturally
```

---

## 12. What This Topic Connects To

- **`07.nextjs/03-data-fetching.md`** — Server Actions enqueue jobs; mutation returns immediately.
- **`07.nextjs/06-api-routes.md`** — Route Handlers are the glue between webhooks → events.
- **`11.capabilities/01-payments.md`** — Stripe webhook fans out via Inngest events.
- **`11.capabilities/02-file-uploads.md`** — `image.uploaded` event triggers the sharp pipeline.
- **`11.capabilities/03-email.md`** — `sendEmail.X()` calls inside `step.run`; transactional emails always go through queues.
- **`13.service-stack/01-trpc-deep.md`** — Long-running tRPC mutations enqueue + return early.
- **`07.nextjs/05-deployment.md`** — BullMQ workers need long-lived hosts (Railway, Fly).
- **`10.production/04-observability.md`** — Sentry tags per job; OTLP from Inngest dashboards.

---

## 13. Summary

| Tool | Pick when |
|------|-----------|
| **Inngest 3** | Default for new Next.js apps; event-driven; multi-step; serverless-friendly |
| **Trigger.dev 3** | Same niche; cleaner per-task DX; long-running (>15min) jobs |
| **BullMQ 5** | Self-hosted Redis; existing ops experience; high throughput |
| **QStash 2** | Simplest "run this URL after delay/schedule" |
| **Vercel Cron** | Scheduled only; bundled with Vercel Pro |
| **Temporal** | Complex multi-day durable workflows |
| **AWS SQS / Cloudflare Queues** | Cloud-native shops |

| Rule | Why |
|------|-----|
| Never run >5s of work inline in a Server Action | Vercel function limits; user waits unnecessarily |
| Make handlers idempotent | Retries are inevitable |
| Use multi-step durability for sequences | Avoids re-running successful steps on partial failure |
| Tag jobs with userId for Sentry / DataDog | Diagnose user-specific failures fast |
| Use `step.invoke` for fan-out | Other functions queue independently |
| Pre-warm Vercel cold starts via cron ping | First Inngest job after cold start is slower |
| Outbox pattern for emit-after-commit | Don't fire events inside DB transactions that may roll back |

---

## Further reading

- [Inngest docs](https://www.inngest.com/docs)
- [Trigger.dev docs](https://trigger.dev/docs)
- [BullMQ docs](https://docs.bullmq.io/)
- [QStash docs](https://upstash.com/docs/qstash/overall/getstarted)
- [Vercel Cron docs](https://vercel.com/docs/cron-jobs)
- [Temporal docs](https://docs.temporal.io/)
- [Inngest blog — durable execution](https://www.inngest.com/blog/durable-functions-a-visual-javascript-primer)
