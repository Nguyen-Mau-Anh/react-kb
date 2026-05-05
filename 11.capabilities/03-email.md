# Capabilities — 03. Email: react-email 3 + Resend 3, MJML, Deliverability

> **What / Why / How** — write transactional emails as React components with **react-email 3**, send them via **Resend 3** (or Postmark / SendGrid). Verify SPF + DKIM + DMARC on day one or your emails go to spam.

---

## 1. The Real Choices in 2026

| Layer | Tool | npm / SDK | Best for |
|-------|------|-----------|----------|
| **Email templates as React** | `@react-email/components@0.0.31` + `@react-email/render@1.0` | The 2026 default for transactional emails |
| **Email templates with MJML** | `mjml@4` + `mjml-react@2` | Pre-react-email projects; teams that prefer the MJML markup language |
| **Email templates with HTML** | Hand-written + a templating engine (Handlebars 4, EJS) | Legacy systems |
| **Sending — modern transactional** | `resend@3` | The 2026 default for new projects (made by the same team as react-email) |
| **Sending — long-trusted** | `postmark@4` (`postmark-js`) | Best deliverability reputation; ideal for password resets, receipts |
| **Sending — at scale** | `@sendgrid/mail@8` (Twilio SendGrid) | Enterprise; high volume; advanced template suppression |
| **Sending — AWS shop** | `@aws-sdk/client-ses@3` | Already on AWS; cheapest at high volume |
| **Sending — alternative** | `mailtrap@3`, `loops@3` (`loops-sdk@1`) | Loops is product-led; Mailtrap great for testing |
| **Sending — marketing** | `@resend/audiences` (Resend's audience feature), `customer.io@2`, Mailchimp | Newsletter / marketing campaigns |
| **Email testing** | `@react-email/preview@1.0`, `mailtrap@3`, `mailpit@1` (self-hosted) | Local preview without sending real emails |

**The 2026 default for any new React/Next.js app**: `react-email@3` + `resend@3`. They're built by the same team (Resend), they integrate seamlessly, and the developer experience is significantly better than every alternative.

This file focuses on that combo and explains the deliverability fundamentals every team gets wrong on first launch.

---

## 2. The Mental Split — Transactional vs Marketing

| | **Transactional** | **Marketing** |
|--|-------------------|---------------|
| Sent in response to | A user action (signup, purchase, password reset) | A campaign you initiate |
| Volume | 1 per event | Bulk to a list |
| Compliance | Implied consent | Explicit opt-in (CAN-SPAM, GDPR) |
| Best provider | Resend, Postmark, AWS SES | Customer.io, Loops, Mailchimp, Resend Audiences |
| Subscribers can unsubscribe? | No (it's a service notification) | **Mandatory** — unsubscribe link required by law |
| Send domain | Often `noreply@app.com` (or, better, a real reply-to) | Often a separate subdomain like `news.app.com` |

**Use separate domains or subdomains for transactional vs marketing.** A spam complaint on a marketing campaign shouldn't tank the deliverability of your password-reset emails. Most production teams send transactional from `mail.app.com` and marketing from `news.app.com`.

This file covers transactional patterns. Marketing email is a different beast (segmentation, drip campaigns, A/B subject lines) handled by Customer.io or Loops, not raw email APIs.

---

## 3. Why react-email 3 Won

### What

`@react-email/components@0.0.31` is a set of React components — `<Html>`, `<Head>`, `<Body>`, `<Container>`, `<Section>`, `<Row>`, `<Column>`, `<Button>`, `<Text>`, `<Img>`, `<Link>`, `<Hr>` — that render to email-client-compatible HTML.

```tsx
import { Body, Button, Container, Head, Html, Img, Link, Preview, Section, Text } from '@react-email/components';

export function WelcomeEmail({ name }: { name: string }) {
  return (
    <Html>
      <Head />
      <Preview>Welcome to Acme — let's get you set up</Preview>
      <Body style={{ background: '#f6f9fc', fontFamily: 'system-ui, sans-serif' }}>
        <Container style={{ background: '#fff', maxWidth: 580, padding: '32px', borderRadius: 12 }}>
          <Img src="https://example.com/logo.png" width="120" height="32" alt="Acme" />
          <Text style={{ fontSize: 24, fontWeight: 700, marginTop: 24 }}>Hi {name},</Text>
          <Text style={{ fontSize: 15, lineHeight: 1.6, color: '#475569' }}>
            Thanks for signing up. Confirm your email to start.
          </Text>
          <Section style={{ textAlign: 'center', margin: '32px 0' }}>
            <Button
              href="https://example.com/confirm?token=abc"
              style={{
                background: '#2563eb',
                color: '#fff',
                padding: '12px 32px',
                borderRadius: 8,
                textDecoration: 'none',
                fontWeight: 600,
              }}
            >
              Confirm email
            </Button>
          </Section>
          <Text style={{ fontSize: 13, color: '#94a3b8' }}>
            Or paste this link: <Link href="https://example.com/confirm?token=abc">https://example.com/confirm?token=abc</Link>
          </Text>
        </Container>
      </Body>
    </Html>
  );
}
```

### Why react-email beats hand-written HTML

| Concern | Hand-written HTML email | react-email 3 |
|---------|--------------------------|----------------|
| Render in Outlook | Manual `<table>` hacks for every row | Components emit Outlook-safe HTML |
| Dark-mode support | Manual `@media (prefers-color-scheme: dark)` | Built into the components |
| Type-safe variables | None | Full TypeScript on props |
| Test locally | Send to MailHog and inspect | `npx email dev` opens a hot-reloading preview |
| Compose from smaller pieces | Copy-paste tables | Real React composition |
| Edited by designers | They learn email-HTML | They use a `<Section>` |

The killer feature: **`@react-email/preview` is a hot-reloading dev server** showing your email in a browser. Edit the React file, see the rendered email update instantly, including the actual gnarly HTML output.

### Setup

```bash
npm i @react-email/components@0.0.31 @react-email/render@1.0
npm i -D react-email@3
```

```jsonc
// package.json
{
  "scripts": {
    "email:dev": "email dev --dir emails",
    "email:export": "email export --dir emails"
  }
}
```

```
emails/
├── welcome.tsx
├── magic-link.tsx
├── password-reset.tsx
├── invoice.tsx
└── _components/                ← shared layout pieces
    ├── Layout.tsx
    └── Footer.tsx
```

`npm run email:dev` opens `http://localhost:3000` with a list of every template. Click one, see it rendered. Editing the file hot-reloads.

### Render to HTML on the server

```ts
import { render } from '@react-email/render';
import { WelcomeEmail } from '../emails/welcome';

const html = await render(<WelcomeEmail name="Anh" />);
const text = await render(<WelcomeEmail name="Anh" />, { plainText: true });
```

`render()` returns a string. Pass it to whatever sender you use. Always provide a plain-text fallback — some clients still default to text-only, and **plain-text emails have measurably better deliverability** (less likely to get flagged as marketing/spam).

---

## 4. Resend 3 — The 2026 Sending Default

### Why Resend

- **Built by the react-email team.** Tight integration; first-class examples.
- **Per-email pricing** — $0 for 100/day on the free tier; $20/mo for 50K/month.
- **Domain-verification UX is the cleanest in the industry.** Add domain → see DNS records → click verify → done.
- **Built-in idempotency keys**, batched sending (up to 100 emails per call), audit logs, webhooks for delivery events.
- **Audiences** (Resend's contact list feature) and **Broadcasts** (one-shot campaigns) for light-touch marketing without going to Customer.io.

### Setup

```bash
npm i resend@3
```

```ts
// lib/email.ts
import { Resend } from 'resend';

export const resend = new Resend(process.env.RESEND_API_KEY!);
```

### Send a react-email template

```ts
// app/api/signup/route.ts
import { resend } from '@/lib/email';
import { WelcomeEmail } from '../../../emails/welcome';

export async function POST(req: Request) {
  const { email, name } = await req.json();
  const user = await db.user.create({ data: { email, name } });

  const { data, error } = await resend.emails.send({
    from: 'Acme <hi@mail.example.com>',
    to: email,
    subject: 'Welcome to Acme',
    react: <WelcomeEmail name={name} />,            // pass JSX directly!
    headers: { 'X-Entity-Ref-ID': user.id },         // for idempotency
    tags: [{ name: 'category', value: 'transactional' }, { name: 'template', value: 'welcome' }],
  });

  if (error) {
    // log to Sentry — covered in 10.production/04-observability.md
    return Response.json({ error: 'Failed to send' }, { status: 500 });
  }
  return Response.json({ id: data!.id });
}
```

`react: <WelcomeEmail ... />` is the Resend-specific shortcut — no need to call `render()` yourself. Resend handles the conversion server-side and provides plain-text fallback automatically.

### Idempotency — `X-Entity-Ref-ID`

Send the same `X-Entity-Ref-ID` header twice within ~5 minutes → Resend deduplicates. Use the user ID, order ID, or password-reset-token ID. Prevents double-send when a Server Action retries.

### Webhooks for delivery events

```ts
// app/api/webhooks/resend/route.ts
import { Resend } from 'resend';
import { db } from '@/server/db';
import { headers } from 'next/headers';

export async function POST(req: Request) {
  const sig = (await headers()).get('svix-signature');
  const raw = await req.text();
  // ⚠ Must be raw body, like Stripe (covered in 11.capabilities/01-payments.md)

  // Verify Svix signature with @svix/webhooks; details in Resend docs
  const event = JSON.parse(raw);

  switch (event.type) {
    case 'email.delivered':   await db.emailEvent.create({ data: { id: event.data.email_id, status: 'delivered' } }); break;
    case 'email.bounced':     await db.emailEvent.create({ data: { id: event.data.email_id, status: 'bounced',   reason: event.data.reason } }); break;
    case 'email.complained':  await db.emailEvent.create({ data: { id: event.data.email_id, status: 'complained' } }); break;
    case 'email.opened':      /* engagement tracking */ break;
    case 'email.clicked':     /* link-click tracking */ break;
  }
  return new Response(null, { status: 200 });
}
```

The bounce + complaint events are critical: **automatically suppress users who bounce or mark you as spam** to protect your sender reputation:

```ts
case 'email.bounced':
  if (event.data.bounce_type === 'hard') {
    await db.user.update({
      where: { email: event.data.to[0] },
      data: { emailStatus: 'bounced', emailOptOut: true },
    });
  }
  break;
```

Sending to a hard-bounced address again is one of the fastest ways to destroy deliverability.

---

## 5. Postmark 4 — When Reputation Matters Most

### Why Postmark still wins for some cases

`postmark@4` (officially `postmark-js`) has the strongest **deliverability reputation** in the industry. Their entire pitch is: **transactional only**, no marketing emails on shared IPs, infrastructure tuned exclusively for fast inbox placement.

Real-world numbers: Postmark consistently delivers within ~30 seconds, with inbox placement >99% on properly-configured domains. SendGrid and Mailgun have higher variance.

### When to pick Postmark over Resend

- **Password resets** that absolutely must arrive within 60 seconds.
- **Two-factor codes** sent via email.
- **Receipts and order confirmations** where late delivery is a customer-service ticket.
- **B2B SaaS** where the receiving company's IT department has strict spam filters.

### Setup

```bash
npm i postmark@4
```

```ts
import { ServerClient } from 'postmark';

const postmark = new ServerClient(process.env.POSTMARK_SERVER_TOKEN!);

await postmark.sendEmail({
  From: 'hi@mail.example.com',
  To: 'user@example.com',
  Subject: 'Reset your password',
  HtmlBody: html,
  TextBody: text,
  MessageStream: 'outbound',                      // 'outbound' for transactional, 'broadcast' for marketing
});
```

`MessageStream` is Postmark's concept that **enforces transactional/marketing separation** at the API level. Marketing on a transactional stream gets rejected.

### When NOT to pick Postmark

- You need bulk-list features (newsletters, segmentation) — Postmark explicitly doesn't support marketing.
- You want email previews from a React-native dev workflow — Resend's react-email integration is tighter.

### Real production split

Many teams use **both**: Resend for product email (welcome, weekly digest), Postmark for high-stakes (receipts, password resets, 2FA codes). Different DNS subdomains, different deliverability profiles.

---

## 6. Deliverability — The Three Records You MUST Configure

The single most important section of this file. **Email sent without these records goes to spam ~70% of the time.** No amount of beautiful react-email templates fixes this.

### Three DNS records, in order of mandatory-ness

#### 1. SPF (Sender Policy Framework)

Tells receivers which servers are allowed to send mail for your domain.

```
mail.example.com.    TXT    "v=spf1 include:_spf.resend.com ~all"
```

(Or `include:spf.mtasv.net` for Postmark, `include:sendgrid.net` for SendGrid, etc.)

`~all` means "soft-fail anything else" — recommended over `-all` (hard-fail) until your config is stable.

#### 2. DKIM (DomainKeys Identified Mail)

Cryptographically signs each email. Receivers verify the signature against a public key in DNS.

Each provider gives you a TXT record like:
```
resend._domainkey.mail.example.com.   TXT   "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQ..."
```

Without DKIM: emails get marked as "unsigned" and routed to spam. **Enable in the provider's dashboard before sending a single email.**

#### 3. DMARC (Domain-based Message Authentication, Reporting and Conformance)

Policy that tells receivers what to do with emails that fail SPF or DKIM.

```
_dmarc.example.com.    TXT    "v=DMARC1; p=none; rua=mailto:dmarc@example.com; pct=100"
```

- `p=none` — monitor only (start here).
- `p=quarantine` — receivers send failures to spam.
- `p=reject` — receivers drop failures entirely.

**Real launch path**: start with `p=none` for 2 weeks, watch the DMARC reports, fix any unauthorized senders, then move to `p=quarantine`, then to `p=reject` after another 2 weeks. **Going straight to `p=reject` and discovering a misconfigured cron job is sending mail = silently dropped emails for a week.**

### One more record: BIMI (optional, but nice)

`Brand Indicators for Message Identification` — shows your logo next to the sender name in Gmail, Yahoo Mail, Apple Mail. Requires DMARC at `p=quarantine` or `p=reject` plus a verified VMC (Verified Mark Certificate, costs ~$1500/yr from Entrust or DigiCert).

For most apps: skip BIMI until you have DMARC at quarantine/reject. For consumer brands where logo recognition matters: worth it.

### Verification flow with Resend

1. Add domain `mail.example.com` in Resend dashboard.
2. Resend shows you 4 DNS records: SPF, DKIM (×2), and a tracking record.
3. Add to your DNS provider (Cloudflare, Namecheap, etc.).
4. Click "Verify" — usually instant; can take up to 72 hours.
5. Add DMARC record yourself (Resend doesn't manage it).
6. Send a test email; inspect headers in Gmail to confirm `dkim=pass`, `spf=pass`, `dmarc=pass`.

### Test your DKIM/SPF/DMARC

- **`mail-tester.com`** — send an email to the address it shows; get a 0–10 score with explanations.
- **`dmarcian.com`** or **MXToolbox** — DNS lookups + DMARC report parsing.
- **Gmail "Show original"** — every Gmail message has a "Show original" menu that shows the raw `Authentication-Results` header.

A real production SaaS aims for `mail-tester` score ≥ 9/10. Below 8/10 means significant spam-folder placement.

---

## 7. The Avoid-The-Spam-Folder Checklist

Beyond the DNS basics:

- [ ] **From address has a real reply-to.** `noreply@example.com` is treated as suspicious. Use `hi@example.com` or `support@example.com`.
- [ ] **Send from a subdomain** (`mail.example.com`), not the root. Damage to a subdomain's reputation doesn't affect the root.
- [ ] **List-Unsubscribe header on marketing**: `List-Unsubscribe: <mailto:unsub@example.com>, <https://example.com/unsubscribe?token=...>`. Plus `List-Unsubscribe-Post: List-Unsubscribe=One-Click` (RFC 8058 — Gmail / Apple Mail one-click).
- [ ] **Plain-text part ≥ 50% of HTML word count.** Imbalance triggers spam filters.
- [ ] **No image-only emails.** Always include text. Images can be blocked by default; image-only emails appear empty.
- [ ] **Image-to-text ratio < 60%.** A wall of images = spam signal.
- [ ] **Don't use URL shorteners** (`bit.ly`, `t.co`). Use your own domain or your provider's branded tracking link.
- [ ] **Avoid spam trigger words in subject.** "FREE", "URGENT", "LIMITED TIME", excessive `!!!`, all-caps. Filters keyword-match.
- [ ] **Keep HTML under 102 KB.** Gmail clips longer messages and shows "Message clipped — view entire message" — a known engagement killer.
- [ ] **Warm up new sending domains.** Send a small volume initially (50/day → 500/day → 5K/day over 2–4 weeks). Sudden bursts trigger reputation systems.
- [ ] **Process bounce + complaint webhooks.** Auto-suppress (covered in §4).
- [ ] **Honor unsubscribes within 10 business days** (CAN-SPAM legal requirement).

Real-world rule: most teams that complain "our emails go to spam" have skipped 3+ of these. Fix them in order; deliverability follows.

---

## 8. Real Templates — The Set Every SaaS Needs

The minimum template library for a launching SaaS:

| Template | When sent | Critical fields |
|----------|-----------|-----------------|
| `welcome.tsx` | After signup | Name, app URL, login link |
| `magic-link.tsx` | Magic-link auth (`next-auth@5` Email provider) | Sign-in link, expiry time |
| `password-reset.tsx` | Forgot password flow | Reset link, expiry, source IP |
| `email-verification.tsx` | Email change | Verification link |
| `invoice.tsx` | After Stripe payment (covered in `11.capabilities/01-payments.md`) | Amount, line items, payment date, invoice URL |
| `payment-failed.tsx` | Subscription renewal failure | Reason, retry CTA |
| `account-deleted.tsx` | After account deletion | Confirmation, support link |

For each, **send the same content from a Server Action OR from a webhook handler**, but never duplicate the template logic. Build a `lib/email/send.ts` helper:

```ts
// lib/email/send.ts
import { resend } from '@/lib/email';
import { WelcomeEmail } from '../../emails/welcome';
import { PasswordResetEmail } from '../../emails/password-reset';
// ... etc

const FROM = 'Acme <hi@mail.example.com>';

export const sendEmail = {
  async welcome(to: string, props: { name: string }) {
    return resend.emails.send({
      from: FROM,
      to,
      subject: `Welcome to Acme, ${props.name}`,
      react: <WelcomeEmail {...props} />,
      tags: [{ name: 'template', value: 'welcome' }],
    });
  },

  async passwordReset(to: string, props: { resetUrl: string; expiresIn: string }) {
    return resend.emails.send({
      from: FROM,
      to,
      subject: 'Reset your Acme password',
      react: <PasswordResetEmail {...props} />,
      tags: [{ name: 'template', value: 'password-reset' }],
    });
  },
  // ... etc
};
```

Now everywhere in the codebase: `await sendEmail.welcome(user.email, { name: user.name })`. Single point of change for sender-name updates, header changes, tagging, queueing.

---

## 9. Sending from Background Queues

Sending email synchronously from a Server Action is risky:
- The user's request waits 100–500ms for the email API.
- A transient Resend error fails the user's signup.
- Retry logic clutters the action code.

The standard pattern: **enqueue email sends to a background job**.

| Tool | Best for |
|------|----------|
| **Inngest** (`inngest@3`) | Event-driven; free tier covers most apps |
| **Trigger.dev** (`@trigger.dev/sdk@3`) | Same model, also free tier |
| **BullMQ** (`bullmq@5`) | Redis-backed; you run the worker |
| **QStash** (`@upstash/qstash@2`) | HTTP-based queue; great with Vercel |

```ts
// app/api/signup/route.ts
import { inngest } from '@/inngest/client';

export async function POST(req: Request) {
  const { email, name } = await req.json();
  const user = await db.user.create({ data: { email, name } });

  await inngest.send({
    name: 'user.signed-up',
    data: { userId: user.id, email, name },
  });

  return Response.json({ ok: true });
}
```

```ts
// inngest/functions/welcome-email.ts
import { inngest } from '@/inngest/client';
import { sendEmail } from '@/lib/email/send';

export const sendWelcomeEmail = inngest.createFunction(
  { id: 'send-welcome-email', retries: 3 },
  { event: 'user.signed-up' },
  async ({ event, step }) => {
    await step.run('send-welcome', async () => {
      await sendEmail.welcome(event.data.email, { name: event.data.name });
    });
  }
);
```

The user gets an instant signup response. The email sends in the background with built-in retries. If Resend is down, Inngest retries with exponential backoff — the user never sees the error.

---

## 10. Local Development & Testing

You don't want to send real emails during dev.

### `react-email` preview

`npm run email:dev` — already covered in §3. Best for template iteration.

### Mailpit / MailHog — local SMTP receiver

```bash
docker run -d -p 1025:1025 -p 8025:8025 axllent/mailpit
```

Configure your dev SMTP to `localhost:1025`; Mailpit's UI at `localhost:8025` shows every email your app would send. Works with any provider that has an SMTP fallback (SendGrid, AWS SES, plain `nodemailer@6`).

### Mailtrap — hosted email sandbox

For preview environments deployed to staging URLs: **Mailtrap.io** captures all outgoing emails into a virtual inbox without forwarding them. Great for QA teams who don't want to set up local SMTP.

### Testing email content with Vitest

```ts
import { render } from '@react-email/render';
import { WelcomeEmail } from '../../emails/welcome';

it('welcome email contains the user name', async () => {
  const html = await render(<WelcomeEmail name="Anh" />);
  expect(html).toContain('Hi Anh');
  expect(html).toContain('https://example.com/confirm');
});
```

`render()` returns a string; assert against it like any HTML output. For visual regression, render to image with `playwright@1.46` and compare snapshots.

---

## 11. Compliance — The Legal Floor

| Law | Applies to | Requires |
|-----|------------|----------|
| **CAN-SPAM** (US) | All commercial email to US recipients | Physical address in footer, clear unsubscribe link, honor unsub within 10 days |
| **GDPR** (EU) | Marketing email to EU recipients | Explicit opt-in (no pre-checked boxes), right to be forgotten |
| **CASL** (Canada) | Commercial email to Canadians | Express or implied consent, sender ID, easy unsubscribe |
| **PDPA** (Singapore) | Marketing to Singapore residents | Opt-in, do-not-call register check |

### Real implementation — what every footer needs

```tsx
<Section style={{ marginTop: 32, padding: 16, borderTop: '1px solid #e2e8f0' }}>
  <Text style={{ fontSize: 11, color: '#94a3b8' }}>
    Acme Inc., 123 Market St, San Francisco, CA 94103, USA.
  </Text>
  <Text style={{ fontSize: 11, color: '#94a3b8' }}>
    You received this because you signed up at example.com.{' '}
    <Link href="https://example.com/unsubscribe?token={token}">Unsubscribe</Link>.
  </Text>
</Section>
```

The physical address + unsubscribe link must be present on **every marketing email**. Transactional emails (password reset, receipt) don't legally require unsubscribe but should still clearly identify you.

### The one-click unsubscribe (RFC 8058)

For marketing emails, Gmail and Apple Mail show a one-click unsubscribe link in the email header if you set:

```
List-Unsubscribe: <mailto:unsub@example.com>, <https://example.com/unsub?t=abc>
List-Unsubscribe-Post: List-Unsubscribe=One-Click
```

Resend supports this via `headers`:
```ts
await resend.emails.send({
  ...,
  headers: {
    'List-Unsubscribe': '<https://example.com/unsubscribe?token=abc>, <mailto:unsub@example.com>',
    'List-Unsubscribe-Post': 'List-Unsubscribe=One-Click',
  },
});
```

When the user clicks unsubscribe in Gmail, your `/unsubscribe?token=abc` endpoint must accept POST (not just GET) and add them to your suppression list. **Required for sending to Gmail bulk senders since 2024.**

---

## 12. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Emails go to spam in Gmail | DKIM not configured, OR DMARC `p=none` for too long. Move to `quarantine` |
| Outlook breaks the layout | react-email's `<Container>`, `<Section>`, `<Row>` use email-safe tables. Don't write your own |
| Images don't display in some clients | Always set `width` and `height` on `<Img>`; provide `alt` text |
| New domain has ~50% spam placement | You haven't warmed it up. Send 50/day → 500/day → 5K/day over 2–4 weeks |
| Email looks fine in dev but breaks in Outlook 2019 | Test with Litmus or Email on Acid (paid, but worth it for B2B) |
| Click tracking links get rewritten and break | The provider replaced your `https://app.com/x` with `https://track.resend.com/...`. Disable in tag headers if needed |
| Bounce rate climbs over time | Not processing bounce webhooks; sending to dead addresses |
| Spam complaints are over 0.1% | Audit content; remove spam trigger words; review opt-in flow |
| Same template re-renders 60ms per request | Cache the rendered HTML by template+props hash |
| Sending from `noreply@example.com` | Use a real reply-to like `hi@example.com` for trust signals |
| Email subject line in mailclient gets cut at 35 chars | Mobile clients show the first 35 chars; front-load the value prop |
| Unsubscribe link doesn't work | Make `/unsubscribe` accept POST (RFC 8058) and a GET fallback |
| Emails arrive minutes late | You're proxying through SMTP from a serverless function with cold starts. Use a queue (Inngest) and a fast API like Resend |

---

## 13. Decision Tree

```
What kind of email?
│
├── Transactional only (signups, receipts, resets)
│   ├── Default → react-email@3 + resend@3
│   ├── Need 99%+ inbox placement → postmark@4
│   └── Already on AWS → @aws-sdk/client-ses@3
│
├── Marketing campaigns / drip flows
│   ├── Product-led pricing → loops@3 (next-gen) or customer.io@2
│   ├── Bundled with transactional → Resend Audiences/Broadcasts
│   └── High-volume newsletter → Mailchimp / Substack
│
└── Both transactional AND marketing
   → Use SEPARATE subdomains: mail.example.com (transactional) + news.example.com (marketing)

What template engine?
│
├── React-first team → react-email@3 (default)
├── Pre-existing MJML templates → mjml@4 + mjml-react@2
└── Maintaining a 2018-era codebase → keep what works; refactor only when changing content

How to send from your app?
│
├── Synchronous from Server Action → fine for low-volume password resets
├── Production (real volume, retries needed) → Inngest 3 / Trigger.dev 3 / QStash 2 + queue jobs
└── Bulk send (broadcast) → Resend Audiences batch API or marketing platform
```

---

## 14. What This Topic Connects To

- **`07.nextjs/04-auth.md`** — `next-auth@5` Email provider sends magic links via your email helper.
- **`07.nextjs/06-api-routes.md`** — Webhook handler for Resend delivery events.
- **`11.capabilities/01-payments.md`** — Receipt and payment-failed emails fire from Stripe webhooks.
- **`10.production/04-observability.md`** — Track email send rate, bounce rate, click-through rate as real metrics.
- **`08.ecosystem/04-real-project.md`** — Adding a "weekly task summary" email is the natural extension.

---

## Summary

| Tool | Pick when |
|------|-----------|
| `@react-email/components@0.0.31` | Default for transactional templates in 2026 |
| `resend@3` | Default for sending; same team as react-email |
| `postmark@4` | Highest deliverability for password resets / 2FA codes |
| `@aws-sdk/client-ses@3` | Already on AWS; cheapest at scale |
| `loops@3` / `customer.io@2` | Marketing automation, drip flows |
| `mjml@4` + `mjml-react@2` | Pre-react-email projects |
| `mailpit@1` (Docker) | Local SMTP development |
| Inngest 3 / Trigger.dev 3 / QStash 2 | Background email-send queue |

| Rule | Why |
|------|-----|
| Configure SPF + DKIM + DMARC on day one | Without them, ~70% of emails go to spam |
| Send from a subdomain, not root | Insulate root domain reputation |
| Process bounce + complaint webhooks | Auto-suppress dead addresses to protect reputation |
| Use react-email's `<Button>` not raw `<a>` | Outlook-safe HTML |
| Always provide plain-text fallback | Better deliverability + accessibility |
| Implement RFC 8058 one-click unsubscribe | Required by Gmail bulk-sender rules |
| Warm up new domains gradually | 50 → 500 → 5K/day over 2–4 weeks |
| Send from a queue, not a Server Action | Survive provider blips with retries |

---

## Further reading

- [react-email docs](https://react.email/docs/introduction)
- [Resend docs](https://resend.com/docs)
- [Postmark docs](https://postmarkapp.com/developer)
- [DMARC.org — Overview](https://dmarc.org/overview/)
- [Gmail bulk-sender requirements (2024)](https://support.google.com/mail/answer/81126)
- [Apple Mail privacy & unsubscribe](https://support.apple.com/en-us/108197)
- [Mail Tester](https://www.mail-tester.com/)
- [MJML docs](https://mjml.io/documentation/)
- [Inngest docs](https://www.inngest.com/docs)
- [Customer.io docs](https://customer.io/docs/)
