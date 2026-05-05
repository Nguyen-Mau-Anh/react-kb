# Capabilities — 01. Payments: Stripe Payment Element, Checkout vs Elements, Subscriptions, Webhooks

> **What / Why / How** — pick **Stripe Checkout** for fastest time-to-revenue; pick **Stripe Payment Element** when you need an embedded payment form; verify webhooks with the **raw body**; never store card data yourself.

---

## 1. The Real Choices in 2026

| Provider | npm / SDK | Best for | Pricing |
|----------|-----------|----------|---------|
| **Stripe** | `@stripe/stripe-js@4`, `@stripe/react-stripe-js@2.8`, `stripe@16.10` (server) | Default for 95% of SaaS in 2026 | 2.9% + $0.30 per US card transaction |
| **Lemon Squeezy** | `@lemonsqueezy/lemonsqueezy.js@4` | Merchant of record (handles VAT/sales tax) for digital products | 5% + $0.50 |
| **Paddle Billing** | `paddle-billing@1` | Merchant of record for SaaS — Paddle takes the tax burden | 5% + $0.50 |
| **PayPal** | `@paypal/react-paypal-js@8` | Add as a secondary option; never as primary | Per-transaction |
| **Adyen** | `@adyen/adyen-web@6` | Enterprise, multi-region, bank-grade | Negotiated |
| **Polar** | `@polar-sh/sdk@0.x` | OSS-friendly merchant-of-record alternative | 4% + $0.40 |

**The 2026 default is Stripe** — but the merchant-of-record alternatives (Lemon Squeezy, Paddle, Polar) deserve real consideration if you're selling internationally and don't want to handle VAT, GST, sales tax, or invoicing per country.

| | Stripe | Merchant of record (Lemon Squeezy / Paddle / Polar) |
|--|--------|-----------------------------------------------------|
| Who is the seller of record on the invoice | **You** | The platform |
| Who collects sales tax / VAT / GST | **You** (or via Stripe Tax add-on at extra fee) | The platform |
| Who issues invoices | **You** | The platform |
| Who handles chargebacks | **You** | The platform |
| Total fee | ~2.9% + $0.30 | ~5% + $0.50 |
| Best for | US-only or high-volume; you have an accountant | Global digital sales; small team without tax expertise |

**Indie SaaS rule of thumb in 2026**: pick **Lemon Squeezy** or **Paddle** until you cross ~$50K MRR. Above that, the higher fee starts costing more than hiring a tax/accounting solution to use Stripe directly.

This file focuses on **Stripe** because it's the default choice and the patterns transfer to every other provider. Lemon Squeezy's React integration follows the same Checkout-style redirect model.

---

## 2. The Big Architectural Choice — Checkout vs Elements vs Payment Links

Stripe ships three real ways to take a payment. Pick by the level of control you need.

| | **Stripe Checkout** | **Stripe Elements** (Payment Element) | **Payment Links** |
|--|---------------------|---------------------------------------|-------------------|
| What it is | Hosted payment page on stripe.com | Embedded payment form in your UI | Hosted URL — no code |
| Setup time | ~30 minutes | ~half a day | 5 minutes (no code) |
| Brand control | Logo + colors via dashboard | Full — your design | Logo + colors via dashboard |
| Mobile UX | Excellent (Stripe handles it) | Good (you handle it) | Excellent |
| Apple Pay / Google Pay | Free (one toggle) | Free (one toggle) | Free |
| 3DS / SCA compliance | Stripe handles | Stripe handles | Stripe handles |
| Subscription support | Yes | Yes | Yes |
| Custom fields / discount codes | Limited (built-in) | Full (you build it) | Limited |
| PCI scope | Lowest (SAQ A) | Low (SAQ A — iframe) | Lowest (SAQ A) |
| Examples in production | Linear's checkout, Cal.com, Vercel Pro upgrade | Notion's billing, GitHub's pricing flow, Stripe's own dashboards | Indie courses, simple digital downloads |

**The 80% rule**: start with **Stripe Checkout**. Time to revenue beats UI control for early SaaS. Switch to **Payment Element** later if and only if the redirect flow demonstrably hurts conversion.

**Never pick `CardElement`, `CardNumberElement`, etc. (the legacy Elements) for new code.** They're maintenance-only. The unified `PaymentElement` (introduced 2021) replaces them and supports 50+ payment methods automatically.

---

## 3. Stripe Checkout — The Path of Least Resistance

### What

A hosted page at `checkout.stripe.com` that handles the full payment flow: card form, 3DS challenge, Apple/Google Pay, BNPL (Klarna, Afterpay), bank debit, etc. You redirect the user there and they redirect back when done.

### Setup — Server Action that creates a Checkout session

```bash
npm i stripe@16.10
```

```ts
// app/checkout/actions.ts
'use server';
import Stripe from 'stripe';
import { redirect } from 'next/navigation';
import { auth } from '@/auth';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-09-30.acacia', // pin the API version
});

export async function startCheckout(priceId: string) {
  const session = await auth();
  if (!session?.user) throw new Error('Unauthorized');

  const checkout = await stripe.checkout.sessions.create({
    mode: 'subscription',                          // or 'payment' for one-time
    line_items: [{ price: priceId, quantity: 1 }],
    customer_email: session.user.email!,
    client_reference_id: session.user.id,           // ⚠ correlates to your user
    success_url: `${process.env.NEXT_PUBLIC_APP_URL}/billing?success=true&session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${process.env.NEXT_PUBLIC_APP_URL}/billing?canceled=true`,
    automatic_tax: { enabled: true },               // requires Stripe Tax setup
    allow_promotion_codes: true,
  });

  redirect(checkout.url!);
}
```

### Use it from a button

```tsx
// app/billing/page.tsx (Server Component)
import { startCheckout } from '@/app/checkout/actions';

export default function BillingPage() {
  return (
    <form action={startCheckout.bind(null, 'price_1OabcZ2eZvKYlo2C')}>
      <button type="submit">Upgrade to Pro — $20/mo</button>
    </form>
  );
}
```

`<form action={...}>` with a Server Action (covered in `07.nextjs/03-data-fetching.md`) is the cleanest pattern. No `fetch`, no `e.preventDefault`, no client-side error handling for the happy path.

### Why Checkout wins by default

- **Fastest time-to-revenue** — under 30 minutes from `npm install` to taking real money.
- **Smallest PCI scope** — your servers never see card data. Auditors love this.
- **Free upgrades** — when Stripe ships a new payment method (Klarna, BNPL, RTP), your Checkout instantly supports it.
- **No 3DS / SCA code** — Stripe handles regulatory compliance.
- **Mobile UX is solved** — the hosted page is responsive and tested across thousands of merchants.

### Why Checkout sometimes loses

- The redirect breaks "single-page checkout" flows that some teams want.
- Customizing the page beyond logo + colors is impossible.
- Some users distrust redirects to unknown domains. (Less true since 2024 — "checkout.stripe.com" is widely recognized.)

If those matter, switch to the embedded **Payment Element** (next section).

---

## 4. Stripe Payment Element — Embedded Form, Full Brand Control

### What

`<PaymentElement>` from `@stripe/react-stripe-js@2.8` is an embedded iframe that renders a card form inside your page. PCI scope stays low because the iframe is on Stripe's domain, but visually it looks like part of your app.

### Install

```bash
npm i @stripe/stripe-js@4 @stripe/react-stripe-js@2.8 stripe@16.10
```

### Server: create a PaymentIntent

```ts
// app/api/create-payment-intent/route.ts
import { NextResponse } from 'next/server';
import Stripe from 'stripe';
import { auth } from '@/auth';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2024-09-30.acacia' });

export async function POST(req: Request) {
  const session = await auth();
  if (!session?.user) return new Response('Unauthorized', { status: 401 });

  const { amount, currency } = await req.json();

  const intent = await stripe.paymentIntents.create({
    amount,                                        // in smallest currency unit (cents)
    currency,                                      // 'usd', 'eur', 'jpy'
    automatic_payment_methods: { enabled: true }, // unlocks card + Apple/Google Pay + BNPL + bank
    metadata: { userId: session.user.id },
  });

  return NextResponse.json({ clientSecret: intent.client_secret });
}
```

### Client: render the Payment Element

```tsx
// app/checkout/CheckoutForm.tsx
'use client';
import { Elements, PaymentElement, useElements, useStripe } from '@stripe/react-stripe-js';
import { loadStripe } from '@stripe/stripe-js';
import { useEffect, useState } from 'react';

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!);

export function CheckoutForm({ amount, currency }: { amount: number; currency: string }) {
  const [clientSecret, setClientSecret] = useState<string>();

  useEffect(() => {
    fetch('/api/create-payment-intent', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ amount, currency }),
    })
      .then((r) => r.json())
      .then((d) => setClientSecret(d.clientSecret));
  }, [amount, currency]);

  if (!clientSecret) return <div>Loading…</div>;

  return (
    <Elements stripe={stripePromise} options={{ clientSecret, appearance: { theme: 'stripe' } }}>
      <Form />
    </Elements>
  );
}

function Form() {
  const stripe = useStripe();
  const elements = useElements();
  const [status, setStatus] = useState<'idle' | 'submitting' | 'error'>('idle');
  const [errorMsg, setErrorMsg] = useState<string>();

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (!stripe || !elements) return;
    setStatus('submitting');

    const { error } = await stripe.confirmPayment({
      elements,
      confirmParams: {
        return_url: `${process.env.NEXT_PUBLIC_APP_URL}/billing/success`,
      },
    });

    if (error) {
      setStatus('error');
      setErrorMsg(error.message);
    }
  }

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <PaymentElement />
      <button disabled={status === 'submitting' || !stripe} className="rounded bg-blue-600 px-4 py-2 text-white">
        {status === 'submitting' ? 'Processing…' : 'Pay'}
      </button>
      {errorMsg && <p className="text-sm text-red-600">{errorMsg}</p>}
    </form>
  );
}
```

### What's happening

- **Server creates a `PaymentIntent`** for the exact amount/currency. The `clientSecret` is a one-time-use token tied to that intent.
- **Client mounts `<PaymentElement>` in `<Elements>`** with that clientSecret. Stripe renders an iframe with the card form (or Apple Pay, Klarna, etc., based on the user's region and the methods you enabled).
- **`stripe.confirmPayment(...)`** triggers the actual charge plus 3DS challenge if needed. Stripe redirects to `return_url` on success or shows the error inline.
- **`automatic_payment_methods: { enabled: true }`** is the modern way — Stripe picks the best methods automatically based on user location and amount.

### Theming

The `appearance` option supports CSS-variable-driven theming:

```ts
options={{
  clientSecret,
  appearance: {
    theme: 'stripe',                  // or 'night', 'flat', 'none'
    variables: {
      colorPrimary: '#2563eb',
      colorBackground: '#ffffff',
      colorText: '#0f172a',
      borderRadius: '8px',
      fontFamily: 'Inter, system-ui, sans-serif',
    },
  },
}}
```

For deeper branding: build your own design tokens object that mirrors your shadcn/ui CSS variables. The Stripe form will visually match the rest of your app.

### Why Payment Element beats `CardElement`

- **One iframe, all methods.** `CardElement` only handles cards; you'd need separate elements for IBAN, BLIK, etc. `PaymentElement` handles every method Stripe supports.
- **Auto-localization.** The form's labels translate to the user's browser language.
- **Accessibility built-in.** Stripe handles ARIA, focus, error announcements.

---

## 5. Subscriptions — The Real Implementation

The subscription model in Stripe in 2026:

```
Product (e.g., "Pro plan")
  └── Price (e.g., "$20/mo USD", "$200/yr USD", "€18/mo EUR")
        └── Subscription (per customer)
              └── Invoices (one per period)
                    └── Payment Intents (the actual charge)
```

### Creating products and prices

You can create them via the Dashboard (one-time setup) or programmatically. Most teams use the Dashboard for production prices and `stripe-cli` for local dev:

```bash
brew install stripe/stripe-cli/stripe
stripe login
stripe products create --name "Pro" --description "Pro plan"
stripe prices create --product prod_xxx --unit-amount 2000 --currency usd --recurring interval=month
```

Save the resulting `price_id` somewhere your app can read it — typed env constants are the standard:

```ts
// lib/billing.ts
export const PRICES = {
  PRO_MONTHLY: 'price_1OabcZ2eZvKYlo2C',
  PRO_YEARLY:  'price_1OabcZ2eZvKYlo3D',
} as const;
```

### Subscribe via Checkout

```ts
const checkout = await stripe.checkout.sessions.create({
  mode: 'subscription',
  line_items: [{ price: PRICES.PRO_MONTHLY, quantity: 1 }],
  // ... rest as in §3
});
```

### Manage subscription — Customer Portal

The killer feature for SaaS: Stripe's hosted **Customer Portal**. Users update card, switch plans, view invoices, cancel — all on Stripe's domain.

```ts
// app/billing/portal/actions.ts
'use server';
import { redirect } from 'next/navigation';
import { db } from '@/server/db';
import { auth } from '@/auth';
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2024-09-30.acacia' });

export async function openCustomerPortal() {
  const session = await auth();
  if (!session?.user) throw new Error('Unauthorized');

  const user = await db.user.findUnique({ where: { id: session.user.id } });
  if (!user?.stripeCustomerId) throw new Error('No Stripe customer');

  const portal = await stripe.billingPortal.sessions.create({
    customer: user.stripeCustomerId,
    return_url: `${process.env.NEXT_PUBLIC_APP_URL}/billing`,
  });

  redirect(portal.url);
}
```

```tsx
<form action={openCustomerPortal}>
  <button type="submit">Manage subscription</button>
</form>
```

This **single button** replaces ~20 features you'd otherwise build: change card, view invoices, see upcoming charges, switch plans, cancel, pause, reactivate, update billing address. Configure what's exposed in the Stripe Dashboard → Settings → Billing → Customer Portal.

---

## 6. Webhooks — The Source of Truth

**Critical principle: never trust the redirect.** A user's browser can close, get hijacked, or the payment can succeed minutes after redirect. The only reliable signal that a payment succeeded is the **webhook from Stripe**.

### The bare-minimum events to handle

| Event | When it fires | What you do |
|-------|---------------|-------------|
| `checkout.session.completed` | Checkout flow successful | Mark order/subscription active in your DB |
| `customer.subscription.created` | New subscription | Set `subscriptionId` and `currentPeriodEnd` on user |
| `customer.subscription.updated` | Plan change, renewal | Update plan / period end |
| `customer.subscription.deleted` | Subscription canceled | Mark user as free tier |
| `invoice.payment_succeeded` | Recurring charge succeeded | Extend access |
| `invoice.payment_failed` | Recurring charge failed | Notify user; start grace period |
| `payment_intent.succeeded` | One-time payment cleared | Mark order paid (Payment Element flows) |
| `charge.refunded` | Refund processed | Reverse access if needed |

### The webhook handler — with critical signature verification

```ts
// app/api/webhooks/stripe/route.ts
import Stripe from 'stripe';
import { db } from '@/server/db';
import { headers } from 'next/headers';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2024-09-30.acacia' });

export async function POST(req: Request) {
  const sig = (await headers()).get('stripe-signature');
  const raw = await req.text();   // ⚠ MUST be raw body, not parsed JSON

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(raw, sig!, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch {
    return new Response('Invalid signature', { status: 400 });
  }

  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object;
      await db.user.update({
        where: { id: session.client_reference_id! },
        data: {
          stripeCustomerId: session.customer as string,
          subscriptionStatus: 'active',
        },
      });
      break;
    }

    case 'customer.subscription.updated':
    case 'customer.subscription.deleted': {
      const sub = event.data.object;
      await db.user.update({
        where: { stripeCustomerId: sub.customer as string },
        data: {
          subscriptionId: sub.id,
          subscriptionStatus: sub.status,
          currentPeriodEnd: new Date(sub.current_period_end * 1000),
          plan: sub.items.data[0].price.id,
        },
      });
      break;
    }

    case 'invoice.payment_failed': {
      const invoice = event.data.object;
      // Notify the user; you'll get more attempts before subscription cancels
      await sendPaymentFailedEmail(invoice.customer_email!);
      break;
    }
  }

  return new Response(null, { status: 200 });
}
```

### Critical gotchas

#### 1. Read the **raw body**, not parsed JSON

```ts
// ❌ Breaks signature verification
const body = await req.json();

// ✅ Required
const raw = await req.text();
```

Stripe's signature is computed over the literal bytes of the request body. Parsing as JSON re-serializes them (different whitespace, different key order) and the verification fails. **This is the #1 reason webhooks "mysteriously fail" on Vercel.**

#### 2. Pin `apiVersion`

```ts
new Stripe(secret, { apiVersion: '2024-09-30.acacia' });
```

Without this, your code uses your Stripe account's "default API version" — which can change without warning when Stripe rolls out new versions. Pinning makes payloads deterministic.

#### 3. Make the handler **idempotent**

Stripe will retry webhooks (up to 3 days) if your endpoint returns non-2xx. Track processed event IDs:

```ts
// Before doing the work
const exists = await db.webhookEvent.findUnique({ where: { id: event.id } });
if (exists) return new Response(null, { status: 200 });
await db.webhookEvent.create({ data: { id: event.id, type: event.type } });
```

#### 4. Always 200 quickly

If the handler takes >10s, Stripe retries. For heavy work (sending emails, calling external APIs), enqueue a job:

```ts
case 'checkout.session.completed': {
  await db.user.update({ ... });        // do the minimum
  await inngest.send({ name: 'send-welcome-email', data: { userId } }); // queue the rest
  break;
}
```

### Local testing — `stripe listen`

```bash
stripe listen --forward-to localhost:3000/api/webhooks/stripe
```

Outputs a webhook signing secret you put in `.env.local`. The CLI proxies real Stripe events from your test account to your local handler. Trigger events with `stripe trigger checkout.session.completed` for unit-style tests.

---

## 7. Schema — User and Subscription Models

The minimum DB columns that let you make decisions like "is this user on the Pro plan and within their period?":

```prisma
// prisma/schema.prisma
model User {
  id                  String    @id @default(cuid())
  email               String    @unique
  // ... other fields ...

  // --- Stripe billing ---
  stripeCustomerId    String?   @unique
  subscriptionId      String?   @unique
  subscriptionStatus  String?   // 'active' | 'trialing' | 'past_due' | 'canceled' | 'incomplete'
  plan                String?   // your price ID
  currentPeriodEnd    DateTime?
  cancelAtPeriodEnd   Boolean   @default(false)
}

model WebhookEvent {
  id        String   @id   // Stripe event.id — guarantees idempotency
  type      String
  createdAt DateTime @default(now())
}
```

A typical "is the user paid?" check:

```ts
function isProUser(user: User) {
  return (
    (user.subscriptionStatus === 'active' || user.subscriptionStatus === 'trialing') &&
    user.currentPeriodEnd &&
    user.currentPeriodEnd > new Date()
  );
}
```

---

## 8. Real Production Patterns

### Apple Pay / Google Pay

With `automatic_payment_methods: { enabled: true }` on the PaymentIntent, Apple Pay and Google Pay **show automatically** in the Payment Element when:
- The user is on Safari (Apple Pay) or Chrome with a saved card (Google Pay).
- The site is HTTPS.
- For Apple Pay: you've registered your domain in the Stripe Dashboard.

No extra code. Conversion lift on mobile is typically 10–30%.

### Trials

```ts
const checkout = await stripe.checkout.sessions.create({
  mode: 'subscription',
  line_items: [{ price: PRICES.PRO_MONTHLY, quantity: 1 }],
  subscription_data: {
    trial_period_days: 14,
    trial_settings: { end_behavior: { missing_payment_method: 'cancel' } },
  },
  payment_method_collection: 'if_required', // free trial without card up-front
});
```

`payment_method_collection: 'if_required'` means: don't collect a card for the trial. After the trial, prompt for a card via the Customer Portal or in-app banner. Massively boosts trial signup conversion at the cost of slightly more complex post-trial flows.

### Discount codes

```ts
const checkout = await stripe.checkout.sessions.create({
  ...,
  allow_promotion_codes: true,   // shows a "Add promotion code" link in Checkout
});
```

Or apply a discount directly:

```ts
const checkout = await stripe.checkout.sessions.create({
  ...,
  discounts: [{ coupon: 'EARLY_BIRD' }],
});
```

Coupons are managed in the Stripe Dashboard; promotion codes are user-facing strings tied to a coupon.

### Sales tax — Stripe Tax

If you go the Stripe route (not a merchant of record): **Stripe Tax** ($0.50 per transaction or 0.5% of volume, whichever is greater) handles VAT/GST/sales tax calculation. Enable it once:

```ts
const checkout = await stripe.checkout.sessions.create({
  ...,
  automatic_tax: { enabled: true },
});
```

For Payment Element, set `automatic_tax: { enabled: true }` on the PaymentIntent. **You still need to register for taxes in jurisdictions where you exceed thresholds** — Stripe Tax just calculates and reports; it doesn't remit on your behalf. (That's the merchant-of-record advantage.)

### Customer addresses

For tax calculation and fraud signals:

```ts
const checkout = await stripe.checkout.sessions.create({
  ...,
  billing_address_collection: 'required',
  customer_creation: 'always',     // create a Stripe Customer even on one-time payments
});
```

### Recurring usage-based billing

For "$0.001 per API call" pricing:

```ts
// On every API call your app handles
await stripe.subscriptionItems.createUsageRecord(subscriptionItemId, {
  quantity: callCount,
  timestamp: Math.floor(Date.now() / 1000),
  action: 'increment',
});
```

Combine with metered prices in Stripe. Real implementation: batch usage records and report once per minute via a queue job to avoid Stripe API rate limits.

---

## 9. Webhooks Beyond Stripe — General Pattern

Every payment provider's webhook follows the same pattern:

1. Read **raw body**.
2. Verify signature with provider's shared secret.
3. Idempotency-check by event ID.
4. Match-event → update DB minimally.
5. Return 2xx fast.
6. Enqueue heavy work.

Lemon Squeezy, Paddle, PayPal — same shape, different signature algorithm and event names. Build a small abstraction in `lib/billing/` so your business logic is provider-agnostic.

---

## 10. Test Cards — What to Use in Dev

Stripe test mode accepts these card numbers regardless of CVC/expiry (use any future date, any 3-digit CVC):

| Number | Behavior |
|--------|----------|
| `4242 4242 4242 4242` | Success |
| `4000 0025 0000 3155` | Requires 3DS authentication |
| `4000 0000 0000 9995` | Insufficient funds (declined) |
| `4000 0000 0000 0341` | Attaches successfully but fails on first charge |
| `4000 0000 0000 0002` | Generic decline |
| `4000 0027 6000 3184` | 3DS that always fails |

The full list is at [docs.stripe.com/testing](https://docs.stripe.com/testing). **Test the failure cards too** — your error UI is the most important part of the flow.

---

## 11. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Webhook signature verification fails on Vercel | Use `req.text()`, not `req.json()` — see §6 #1 |
| Subscription status not updating | Webhook isn't reaching production. Check `stripe listen` locally + Vercel function logs |
| User charged but not marked paid | Webhook handler errored; Stripe retries for 3 days. Add idempotency table + observability |
| Same user has two Stripe customers | You called `stripe.customers.create` twice. Always pass `customer_email` to Checkout — Stripe dedupes |
| 3DS challenge breaks redirect-back flow | The `return_url` must be reachable; don't put auth gates in front of it |
| Checkout shows wrong currency | Set `currency` on the price, not the session — currency lives at the Price level |
| Test webhooks work; prod doesn't | Production webhook endpoint not registered in Dashboard, OR using test signing secret |
| Tax not appearing | Stripe Tax not enabled, OR `automatic_tax: true` not set, OR origin address not configured in Dashboard |
| `apiVersion` change breaks payloads | Pin the version explicitly; review release notes when upgrading |
| Indie SaaS hit with VAT audit | You owed taxes in the EU; should have used a merchant of record |
| Refunds don't revert access | Add `charge.refunded` to your webhook handler |

---

## 12. Decision Tree

```
What are you charging for?
│
├── Digital goods, courses, e-books, simple SaaS, indie/global
│   → Lemon Squeezy or Paddle (merchant of record), API or Checkout-style
│
├── SaaS subscription, you're US-based or have an accountant
│   → Stripe Checkout (mode: 'subscription') + Customer Portal
│
├── Physical goods or custom checkout flow with shipping fields
│   → Stripe Checkout with shipping options, OR Payment Element
│
├── Embedded payment form is critical for conversion (real evidence, not guess)
│   → Stripe Payment Element
│
├── One-time tip / pay-what-you-want
│   → Stripe Payment Links (zero code)
│
└── Enterprise, multi-region, bank-grade
   → Adyen with custom integration

Where to handle taxes?
│
├── Sell internationally, < $50K MRR, small team → Lemon Squeezy/Paddle (they handle it)
├── US-only → Stripe Tax add-on
├── Large business with finance team → Stripe + manual filings or Avalara
└── Enterprise → Adyen + dedicated tax solution
```

---

## 13. What This Topic Connects To

- **`07.nextjs/03-data-fetching.md`** — Server Actions wrap Stripe API calls cleanly.
- **`07.nextjs/06-api-routes.md`** — Route Handlers for the webhook endpoint.
- **`07.nextjs/04-auth.md`** — Tying Stripe Customer to the authenticated user.
- **`10.production/04-observability.md`** — Sentry + Vercel logs surface webhook failures fast.
- **`08.ecosystem/04-real-project.md`** — Adding "Upgrade to Pro" to the Task Manager is a 2-hour PR using these patterns.

---

## Summary

| Tool | Pick when |
|------|-----------|
| **Stripe Checkout** + **Customer Portal** | Default for SaaS subscriptions in 2026 |
| **Stripe Payment Element** | You've measured the redirect hurts conversion |
| **Stripe Payment Links** | Tiny digital goods, no app integration needed |
| **Lemon Squeezy / Paddle / Polar** | International digital goods without VAT/tax burden |
| **Adyen** | Enterprise / regulated industries |

| Rule | Why |
|------|-----|
| Start with Stripe Checkout, not Payment Element | Faster to revenue; switch later if conversion data justifies it |
| Read raw body in webhook handlers | Signature verification depends on byte-exact payload |
| Pin `apiVersion` | Stripe rolls new versions; deterministic payloads matter |
| Make webhooks idempotent | Stripe retries; your DB shouldn't double-charge |
| Always 200 fast; enqueue heavy work | Webhook timeouts trigger retries |
| Use Stripe Customer Portal | Replaces ~20 features users would otherwise demand |
| `automatic_payment_methods: { enabled: true }` | Free Apple/Google Pay + 50+ payment methods |
| Pick a merchant of record under $50K MRR | The 5% fee is cheaper than tax compliance overhead |

---

## Further reading

- [Stripe Checkout docs](https://docs.stripe.com/payments/checkout)
- [Stripe Payment Element docs](https://docs.stripe.com/payments/payment-element)
- [Stripe Customer Portal](https://docs.stripe.com/customer-management)
- [Webhooks signing](https://docs.stripe.com/webhooks/signatures)
- [Stripe testing — test cards](https://docs.stripe.com/testing)
- [Stripe Tax](https://docs.stripe.com/tax)
- [Lemon Squeezy docs](https://docs.lemonsqueezy.com/)
- [Paddle Billing docs](https://developer.paddle.com/)
- [Polar docs](https://docs.polar.sh/)
