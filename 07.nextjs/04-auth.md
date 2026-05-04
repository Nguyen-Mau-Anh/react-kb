# Next.js — 04. Auth: NextAuth 5 (Auth.js), Middleware, Sessions, Providers

> **What / Why / How** — pick the right auth tool for the job. NextAuth 5 is the default for Next.js. Clerk and Supabase Auth are real alternatives. Don't roll your own.

---

## 1. The Real Auth Options for Next.js 14 in 2026

| Library | npm | Best for | Hosted? |
|---------|-----|----------|---------|
| **Auth.js (NextAuth) v5** | `next-auth@5.0.0-beta` (often shorthand `next-auth@5`) | OAuth + credentials, self-hosted, App Router | No |
| **Clerk** | `@clerk/nextjs@5` | Drop-in UI, organizations, magic links, MFA, SaaS-style auth | Yes (per-MAU pricing) |
| **Supabase Auth** | `@supabase/supabase-js@2` + `@supabase/ssr@0.4` | If you already use Supabase for DB | Yes (free tier) |
| **Lucia v3** | `lucia@3` | Lightweight, fully manual session library, no providers built in | No (you build everything) |
| **better-auth** | `better-auth@0.x` | Modern alternative to Auth.js with cleaner API | No |
| Roll your own | — | Don't | — |

This file covers **Auth.js v5** (the renamed NextAuth) because it's the default for self-hosted Next.js and the most-used auth library in the React ecosystem.

---

## 2. Why Auth.js v5 (Not v4)

| | Auth.js v5 (`next-auth@5`) | NextAuth v4 (`next-auth@4`) |
|--|----------------------------|------------------------------|
| App Router support | First-class | Bolt-on; `getServerSession` everywhere |
| Edge runtime middleware | Yes — split `auth.config.ts` from `auth.ts` for Edge-safe imports | Hard to make work |
| API surface | One `auth()` function for routes/middleware/RSC | Different APIs per context |
| Session in Server Components | `await auth()` directly | `getServerSession(authOptions)` indirection |
| Stable | Beta but production-used (Vercel, T3 stack) | Stable but on maintenance |

For new projects: pick `next-auth@5`. For existing v4 codebases: don't migrate without reason — Auth.js publishes a [v4→v5 upgrade guide](https://authjs.dev/getting-started/migrating-to-v5).

---

## 3. Setup — Auth.js v5 with GitHub OAuth + Database

### Install

```bash
npm i next-auth@5.0.0-beta @auth/prisma-adapter@2 prisma@5.14
```

### Generate `AUTH_SECRET`

```bash
npx auth secret
```

Copies a 32-byte random string into `.env.local` as `AUTH_SECRET`. Required for cookie/JWT signing.

### `auth.config.ts` — Edge-safe config

Split into a runtime-agnostic config (no DB imports — Edge can't run Prisma) and a Node-only `auth.ts`:

```ts
// auth.config.ts — imported by middleware, must run on Edge
import type { NextAuthConfig } from 'next-auth';

export default {
  providers: [], // populated in auth.ts (next file)
  pages: {
    signIn: '/login',
  },
  callbacks: {
    authorized({ auth, request: { nextUrl } }) {
      const isLoggedIn = !!auth?.user;
      const isOnDashboard = nextUrl.pathname.startsWith('/dashboard');
      if (isOnDashboard) return isLoggedIn;
      return true;
    },
  },
} satisfies NextAuthConfig;
```

### `auth.ts` — Node runtime, includes the DB adapter and providers

```ts
// auth.ts
import NextAuth from 'next-auth';
import GitHub from 'next-auth/providers/github';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { prisma } from './lib/prisma';
import authConfig from './auth.config';

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(prisma),
  session: { strategy: 'jwt' },
  ...authConfig,
  providers: [
    GitHub({
      clientId: process.env.AUTH_GITHUB_ID!,
      clientSecret: process.env.AUTH_GITHUB_SECRET!,
    }),
  ],
});
```

### Wire up the route handler

```ts
// app/api/auth/[...nextauth]/route.ts
export { GET, POST } from '@/auth';
```

That single line connects all OAuth callbacks, sign-in, sign-out, etc. to the framework.

### Wire up middleware (uses Edge-safe config)

```ts
// middleware.ts
import NextAuth from 'next-auth';
import authConfig from './auth.config';

export const { auth: middleware } = NextAuth(authConfig);

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
};
```

This runs the `authorized` callback on every matched request — protecting `/dashboard/*` automatically.

---

## 4. The Prisma Schema for Auth.js

The minimal models Auth.js requires when using a database adapter (from the [Auth.js Prisma adapter docs](https://authjs.dev/reference/adapter/prisma)):

```prisma
// prisma/schema.prisma
model User {
  id            String    @id @default(cuid())
  name          String?
  email         String    @unique
  emailVerified DateTime?
  image         String?
  accounts      Account[]
  sessions      Session[]
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime
  @@unique([identifier, token])
}
```

Run `npx prisma migrate dev` to push the schema to your DB.

---

## 5. Reading the Session — Three Real Places

### In a Server Component

```tsx
// app/dashboard/page.tsx
import { auth } from '@/auth';

export default async function DashboardPage() {
  const session = await auth();
  if (!session?.user) {
    return <p>Not signed in.</p>;
  }
  return <h1>Hi, {session.user.name}</h1>;
}
```

`await auth()` runs server-side, reads the cookie/JWT, returns the session. Zero client JS.

### In a Server Action

```tsx
// app/posts/actions.ts
'use server';
import { auth } from '@/auth';

export async function createPost(formData: FormData) {
  const session = await auth();
  if (!session?.user) throw new Error('Unauthorized');

  await db.post.create({
    data: {
      title: formData.get('title') as string,
      authorId: session.user.id,
    },
  });
}
```

### In a Route Handler

```ts
// app/api/posts/route.ts
import { auth } from '@/auth';

export const GET = auth(async (req) => {
  if (!req.auth?.user) {
    return new Response('Unauthorized', { status: 401 });
  }
  // ... business logic
});
```

The `auth()` HOF wraps the handler and exposes `req.auth`. No more session-fetch boilerplate per route.

### In a Client Component

```tsx
// app/components/UserMenu.tsx
'use client';
import { useSession, signIn, signOut } from 'next-auth/react';

export function UserMenu() {
  const { data: session, status } = useSession();
  if (status === 'loading') return null;
  if (!session) return <button onClick={() => signIn('github')}>Sign in</button>;
  return <button onClick={() => signOut()}>Sign out, {session.user?.name}</button>;
}
```

For `useSession` to work, wrap the app in `<SessionProvider>`:

```tsx
// app/providers.tsx
'use client';
import { SessionProvider } from 'next-auth/react';
export function Providers({ children }: { children: React.ReactNode }) {
  return <SessionProvider>{children}</SessionProvider>;
}
```

---

## 6. Sign In and Sign Out Flows

### Sign in via Server Action (recommended)

```tsx
// app/login/page.tsx
import { signIn } from '@/auth';

export default function LoginPage() {
  return (
    <form
      action={async (formData) => {
        'use server';
        await signIn('credentials', { email: formData.get('email'), password: formData.get('password'), redirectTo: '/dashboard' });
      }}
    >
      <input name="email" type="email" />
      <input name="password" type="password" />
      <button type="submit">Sign in</button>
    </form>
  );
}
```

```tsx
// "Sign in with GitHub" — works as a one-liner button
import { signIn } from '@/auth';

<form action={async () => { 'use server'; await signIn('github', { redirectTo: '/dashboard' }); }}>
  <button type="submit">Sign in with GitHub</button>
</form>
```

### Sign out — also a Server Action

```tsx
import { signOut } from '@/auth';

<form action={async () => { 'use server'; await signOut({ redirectTo: '/' }); }}>
  <button type="submit">Sign out</button>
</form>
```

This avoids the `useSession` hook entirely — works without client JS.

---

## 7. Credentials Provider (Email + Password)

OAuth is the easy path. If you must support email/password (and you've decided your team is OK maintaining password resets, breaches, hashing, rate limiting):

```ts
// auth.ts
import Credentials from 'next-auth/providers/credentials';
import bcrypt from 'bcryptjs';
import { z } from 'zod';

providers: [
  Credentials({
    async authorize(credentials) {
      const parsed = z
        .object({ email: z.string().email(), password: z.string().min(8) })
        .safeParse(credentials);
      if (!parsed.success) return null;

      const user = await prisma.user.findUnique({ where: { email: parsed.data.email } });
      if (!user?.passwordHash) return null;

      const ok = await bcrypt.compare(parsed.data.password, user.passwordHash);
      if (!ok) return null;

      return { id: user.id, name: user.name, email: user.email };
    },
  }),
],
```

Required: a `passwordHash` column on `User` (added separately to the Prisma schema). Use `bcrypt` rounds ≥ 12 in 2026.

### Why Auth.js doesn't ship credentials by default

Auth.js calls credentials auth "the most error-prone option" for a reason — you're now responsible for: password reset, account lockout, breach checks, MFA, email verification flows. **If you can use OAuth (GitHub, Google) + magic-link email, do that.** Magic links don't need passwords.

### Magic-link email — no password needed

```ts
import EmailProvider from 'next-auth/providers/nodemailer';

EmailProvider({
  server: process.env.EMAIL_SERVER, // smtps://user:pass@smtp.resend.com:465
  from: process.env.EMAIL_FROM,
}),
```

User enters email → receives a one-time link → clicks → logged in. Used by Notion, Slack, Vercel itself for invites.

For real email delivery: pair with `resend@3` (`@resend/node` or just SMTP), `postmark@4`, or `sendgrid@8`.

---

## 8. Middleware Auth Gating — The Real Pattern

`middleware.ts` runs on the Edge runtime *before* any route renders. With Auth.js's `authorized` callback (defined in `auth.config.ts`), you get fully declarative protection:

```ts
// auth.config.ts
callbacks: {
  authorized({ auth, request: { nextUrl } }) {
    const isLoggedIn = !!auth?.user;
    const isOnDashboard = nextUrl.pathname.startsWith('/dashboard');
    const isOnLogin = nextUrl.pathname === '/login';

    if (isOnDashboard) {
      if (isLoggedIn) return true;
      return false; // redirects to signIn page
    }
    if (isOnLogin && isLoggedIn) {
      return Response.redirect(new URL('/dashboard', nextUrl));
    }
    return true;
  },
},
```

The middleware (set up in §3) reads the JWT, runs `authorized`, and either lets the request through or redirects. **No `getServerSession` calls scattered across pages.**

---

## 9. Sessions — JWT vs Database Strategy

| | JWT strategy (`session.strategy: 'jwt'`) | Database strategy (`session.strategy: 'database'`) |
|--|------------------------------------------|----------------------------------------------------|
| Where the session lives | Signed cookie | Row in `Session` table |
| DB lookups per request | None | One per request |
| Logout invalidates session immediately | ❌ Token still valid until expiry (workaround: maintain a "revoked tokens" list) | ✅ Delete the row, done |
| Edge runtime support | ✅ | ❌ (DB calls don't work in Edge middleware) |
| Default in Auth.js v5 | ✅ when no adapter; or when `session.strategy: 'jwt'` is explicit | When using a DB adapter without override |

For middleware to run on the Edge (you almost certainly want this for performance), pick **JWT strategy** and accept the logout-invalidation tradeoff.

---

## 10. Custom Session Data — `session` and `jwt` Callbacks

By default the session only contains `name`, `email`, `image`. To add custom fields like `role`:

```ts
// auth.ts
callbacks: {
  async jwt({ token, user }) {
    if (user) {
      token.id = user.id;
      token.role = (user as any).role; // from your DB
    }
    return token;
  },
  async session({ session, token }) {
    if (token && session.user) {
      session.user.id = token.id as string;
      (session.user as any).role = token.role;
    }
    return session;
  },
},
```

Also augment the TypeScript types:

```ts
// types/next-auth.d.ts
import { DefaultSession } from 'next-auth';

declare module 'next-auth' {
  interface Session {
    user: { id: string; role: 'user' | 'admin' } & DefaultSession['user'];
  }
}
```

Now `session.user.role` is typed everywhere.

---

## 11. CSRF, Cookies, and Security Defaults

Auth.js handles, by default:
- **CSRF tokens** for the sign-in/sign-out endpoints.
- **`HttpOnly`, `Secure`, `SameSite=Lax`** cookies for the session.
- **Cookie naming** with the `__Secure-` prefix in production.
- **Encrypted JWT** (JWE), not just signed.

You don't need to configure any of this. Don't override the defaults unless you have a specific reason.

---

## 12. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `auth()` returns `null` in middleware on Edge | The DB adapter import leaked into `auth.config.ts`. Split files: Edge-safe config in `auth.config.ts`, Node-only in `auth.ts`. |
| `useSession` returns `undefined` everywhere | Forgot `<SessionProvider>` at the app root |
| Session never includes custom fields | Forgot `jwt` and `session` callbacks; or types/next-auth.d.ts not in the project's `tsconfig` include path |
| OAuth callback URL mismatch | The provider config in GitHub/Google must point to `https://yourdomain/api/auth/callback/{provider}` |
| Logging out doesn't actually log out | JWT strategy: token still valid until expiry. Either accept it, or switch to database strategy (loses Edge support). |
| User signs in repeatedly without ever fully logging in | Check `AUTH_SECRET` is set in production |
| Magic link email fails silently | Misconfigured SMTP. Use `resend@3` with API + verified domain in production. |
| Calling `signIn()` from a Server Component | Either call it from a Server Action, or import the client version `from 'next-auth/react'` and use it from a Client Component. |

---

## 13. When Clerk Beats Auth.js (Real Cases)

Clerk (`@clerk/nextjs@5`) is hosted auth with a polished UI. Worth picking when:

- **You're a small team and want a UI by default** — Clerk ships sign-in / sign-up / user-button components that look good without design work.
- **You need organizations / multi-tenant workspaces** — Clerk's "Organizations" feature is far ahead of what Auth.js + Prisma lets you build quickly.
- **You need passkeys, MFA, magic links, social logins, all of it** — Clerk has them out of the box; Auth.js requires assembling.
- **You're OK with the per-MAU pricing model** — Clerk charges per monthly active user.

Real example trade: a side project at MVP stage often saves 1–2 weeks by using Clerk; a self-hosted SaaS at scale saves thousands per month by using Auth.js + Prisma.

---

## 14. Decision Tree

```
Is this internal-only / fully self-hosted / cost-sensitive?
│   YES → next-auth@5 + Prisma + GitHub or Google OAuth (+ optional magic-link email)
│   NO  ↓
Do you need multi-tenant orgs, passkeys, MFA, polished sign-in UI on day 1?
│   YES → @clerk/nextjs@5
│   NO  ↓
Already using Supabase for the database?
│   YES → Supabase Auth + @supabase/ssr@0.4
│   NO  ↓
Want everything manual, no providers, just sessions?
│   YES → lucia@3
│   NO  → next-auth@5 (the default safe choice)
```

---

## Summary

| Concept | API |
|---------|-----|
| Setup | `auth.config.ts` (Edge-safe) + `auth.ts` (Node-only) split |
| Reading session in RSC / Action / Route Handler | `await auth()` |
| Reading session on client | `useSession()` from `next-auth/react` |
| Sign in / out | `signIn(provider)`, `signOut()` from `'@/auth'` (server) or `'next-auth/react'` (client) |
| Middleware auth | `authorized` callback in `auth.config.ts` |
| Custom session fields | `jwt` + `session` callbacks + module augmentation |
| Database models | Prisma adapter — `User`, `Account`, `Session`, `VerificationToken` |

| Rule | Why |
|------|-----|
| Use `next-auth@5`, not v4, for new App Router projects | Better RSC + Edge support |
| Split `auth.config.ts` from `auth.ts` for Edge middleware | DB adapter imports break Edge runtime |
| Prefer OAuth + magic links over passwords | Less code, fewer security responsibilities |
| Use JWT strategy if middleware must run on Edge | Database strategy doesn't work on Edge |
| Always set `AUTH_SECRET` in production | Cookies/JWTs won't sign without it |
| Don't roll your own auth | Even staff engineers get it wrong; use a library |

---

## Further reading

- [Auth.js (NextAuth) docs](https://authjs.dev/getting-started/installation)
- [Auth.js Edge Compatibility](https://authjs.dev/guides/edge-compatibility)
- [Auth.js Prisma Adapter](https://authjs.dev/reference/adapter/prisma)
- [Clerk for Next.js](https://clerk.com/docs/quickstarts/nextjs)
- [Supabase Auth with Next.js App Router](https://supabase.com/docs/guides/auth/server-side/nextjs)
- [Lucia v3 docs](https://lucia-auth.com/)
