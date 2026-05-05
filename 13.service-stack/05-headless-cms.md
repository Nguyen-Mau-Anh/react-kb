# Service Stack — 05. Headless CMS: Sanity 3 vs Contentful vs Payload 3 vs Strapi 5

> **What / Why / How** — pick **Payload 3** for self-hosted Next.js native; **Sanity 3** for marketing+content sites where editor UX matters; **Contentful** for enterprise + non-technical editors; **Strapi 5** for OSS API-first projects. Skip a CMS entirely if the content lives in your DB and is edited by engineers.

---

## 1. The Real Choices in 2026

| CMS | Self-hosted? | npm / SDK | Best for | Pricing |
|-----|--------------|-----------|----------|---------|
| **Payload 3** | Yes (default) | `payload@3`, `@payloadcms/next@3` | Self-hosted Next.js apps; Postgres or MongoDB; the 2026 default for engineering-led teams | Free OSS; Cloud from $35/mo |
| **Sanity 3** | No (managed) | `sanity@3.60`, `next-sanity@9.8` | Marketing sites; structured content; GROQ query language; superb editor UX | Free 3 users; pay-per-user/document beyond |
| **Contentful** | No (managed) | `contentful@11`, `@contentful/rich-text-react-renderer@16` | Enterprise; non-technical editors; webhooks + workflows; multi-locale | Free up to 25K records; $$$$ at scale |
| **Strapi 5** | Yes (default) | `@strapi/strapi@5` (server), REST/GraphQL clients | OSS-first; API-driven; admin UI; plugin ecosystem | Free OSS; Strapi Cloud from $15/mo |
| **Hygraph** | No | `graphql-request@7` against Hygraph endpoint | GraphQL-native; multi-locale | Free tier; usage-based |
| **Storyblok 3** | No (managed) | `@storyblok/react@3` | Visual editing; non-technical editors; component-based | Free up to 1K entries; per-MAU pricing |
| **Builder.io** | No (managed) | `@builder.io/react@8` | Visual page builder; A/B testing | Per-MAU |
| **Sanity-style OSS — Outstatic, Markdoc, Velite** | Yes (Git/MDX-based) | various | Content lives in git; engineer-edited; no UI | Free |
| **WordPress + REST/GraphQL** | Yes | `wpgraphql@1.27` (plugin) + Apollo | Existing WordPress; non-technical editors prefer the WP UI | Free (you host) |
| **Decap CMS (formerly Netlify CMS)** | Yes | `decap-cms@3` | Git-based; very simple sites | Free |
| **Notion as CMS** | No | `notion-client@7`, `@notionhq/client@2` | Tiny sites; team already uses Notion | Free for small use |
| **Markdown + git (no CMS)** | Yes | MDX + `next-mdx-remote@5` | Engineering blogs, docs sites | Free |

**The 2026 default decision matrix**:
- Self-hosted Next.js + Postgres + engineering-led → **Payload 3**.
- Marketing site + structured content + designer/PM editors → **Sanity 3**.
- Enterprise + non-tech editors + workflow/approval → **Contentful**.
- OSS-first API + admin panel → **Strapi 5**.
- Engineering blog / docs site → **MDX in your repo** (no CMS).

---

## 2. Should You Use a CMS At All?

The honest decision matrix:

| Situation | What to use |
|-----------|-------------|
| **All content edited by engineers** (changelog, blog from devs) | MDX in git. No CMS. |
| **5 content pages, rarely changing** | MDX in git. |
| **Marketing pages edited by a designer / marketer** | Sanity / Storyblok / Payload |
| **Long-form blog with non-tech writers** | Sanity / Contentful / Payload |
| **Multi-locale marketing site** | Contentful / Sanity / Hygraph |
| **Product catalog with thousands of SKUs** | A real e-commerce platform (Shopify, Medusa) — not a CMS |
| **App content driven by users (UGC)** | Your own DB + Prisma — not a CMS |
| **Internal docs read by employees** | Notion or Confluence — not a CMS |

The 2026 reality: **most Next.js projects don't need a CMS**. Components + Tailwind + MDX in the repo is faster to ship and easier to maintain. **Reach for a CMS only when non-engineers will edit content regularly.**

---

## 3. Payload 3 — The 2026 Default for Self-Host

### What

`payload@3` is a TypeScript-first, Next.js-native headless CMS. The single change in v3 (released Oct 2024) is huge: **Payload now installs INSIDE your Next.js project as Route Handlers**, not as a separate Express server. Same database, same auth, same deploy. Admin UI lives at `/admin` of your existing app.

### Setup

```bash
npx create-payload-app@latest
# Pick: Next.js + Postgres + TypeScript
```

This scaffolds a Next.js app with:

```
app/
├── (app)/                          ← your real app routes
│   └── ...
├── (payload)/
│   ├── admin/[[...segments]]/page.tsx    ← Payload admin UI
│   └── api/
│       └── [...slug]/route.ts             ← Payload REST + GraphQL
├── payload.config.ts                       ← single config file
└── ...
collections/
├── Pages.ts
├── Posts.ts
├── Media.ts
└── Users.ts
```

### Define a collection

```ts
// collections/Posts.ts
import type { CollectionConfig } from 'payload';

export const Posts: CollectionConfig = {
  slug: 'posts',
  admin: { useAsTitle: 'title' },
  access: {
    read: () => true,                              // public reads
    create: ({ req: { user } }) => Boolean(user),  // logged-in writes
  },
  versions: { drafts: true, maxPerDoc: 25 },
  fields: [
    { name: 'title', type: 'text', required: true },
    { name: 'slug', type: 'text', required: true, unique: true, index: true },
    { name: 'status', type: 'select', options: ['draft', 'published'], defaultValue: 'draft' },
    { name: 'publishedAt', type: 'date' },
    { name: 'author', type: 'relationship', relationTo: 'users' },
    { name: 'cover', type: 'upload', relationTo: 'media' },
    {
      name: 'content',
      type: 'richText',                            // Lexical-based editor
      required: true,
    },
    {
      name: 'tags',
      type: 'array',
      fields: [{ name: 'tag', type: 'text' }],
    },
  ],
};
```

```ts
// payload.config.ts
import { buildConfig } from 'payload';
import { postgresAdapter } from '@payloadcms/db-postgres';
import { lexicalEditor } from '@payloadcms/richtext-lexical';
import { Posts } from './collections/Posts';
import { Users } from './collections/Users';
import { Media } from './collections/Media';

export default buildConfig({
  secret: process.env.PAYLOAD_SECRET!,
  db: postgresAdapter({ pool: { connectionString: process.env.DATABASE_URL! } }),
  editor: lexicalEditor(),                          // Lexical editor (covered in 10.production/01)
  collections: [Posts, Users, Media],
  admin: { user: Users.slug },
  upload: {
    storage: { /* S3, R2, Vercel Blob — covered in 11.capabilities/02 */ },
  },
});
```

That's it. Run `next dev` and `/admin` is a fully-functional CMS UI; `/api/posts` is the REST API; GraphQL and Local API also available.

### Read content from your Next.js app

```tsx
// app/blog/[slug]/page.tsx (Server Component)
import { getPayload } from 'payload';
import config from '@/payload.config';
import { notFound } from 'next/navigation';
import { RichText } from '@payloadcms/richtext-lexical/react';

export default async function PostPage({ params }: { params: { slug: string } }) {
  const payload = await getPayload({ config });

  const { docs } = await payload.find({
    collection: 'posts',
    where: { slug: { equals: params.slug }, status: { equals: 'published' } },
    depth: 2,                                       // populate author and cover
    limit: 1,
  });

  const post = docs[0];
  if (!post) notFound();

  return (
    <article>
      <h1>{post.title}</h1>
      <RichText data={post.content} />
    </article>
  );
}
```

`getPayload({ config })` is the **Local API** — direct in-process access to the CMS. No HTTP overhead. **Same database, same auth, same process.** The single biggest reason to pick Payload over Sanity/Contentful for a Next.js app.

### ISR + on-demand revalidation

```tsx
// app/blog/[slug]/page.tsx
export const revalidate = 60;                       // ISR: regen every 60s

// On a publish hook (Payload's afterChange):
import { revalidatePath } from 'next/cache';

export const Posts: CollectionConfig = {
  // ...
  hooks: {
    afterChange: [
      ({ doc }) => {
        if (doc.status === 'published') {
          revalidatePath(`/blog/${doc.slug}`);
        }
      },
    ],
  },
};
```

When an editor publishes a post, the page revalidates instantly. Combined with Next.js Metadata API (covered in `07.nextjs/07-seo-metadata.md`), you get a fully SEO-ready blog with zero extra services.

### Why Payload 3 wins for Next.js

- **Single-process deploy** — same Vercel project, same DB, same auth.
- **Local API** — no HTTP round trip from your Server Components to the CMS.
- **TypeScript-first** — generates `payload-types.ts` from your collections; full type safety.
- **OSS + self-host** — no per-document or per-user pricing.
- **Powerful access control** — per-collection, per-document, per-field.
- **Drafts + versions + scheduled publish** built in.
- **Lexical-based rich text** — connects to `10.production/01-rich-text-editors.md`.

### Why Payload can lose

- The Next.js admin route (`/admin/...`) increases your bundle's API surface area; you might want a separate admin domain in some setups.
- Editor UX, while good, is less polished than Sanity Studio for content-heavy editorial workflows.
- Smaller plugin ecosystem than Strapi or WordPress.

For 2026 engineering-led teams shipping a Next.js app: **Payload 3 is the new default**.

---

## 4. Sanity 3 — Best Editor UX

### What

`sanity@3.60` is a managed CMS with a powerful **content modeling** approach and a custom query language called **GROQ**. Studio (the editor UI) is renowned — designers love it because it's customizable per-content-type and supports real-time collaboration.

### Setup

```bash
npm create sanity@latest                            # creates a Studio
npm i next-sanity@9.8 @sanity/image-url@1
```

Studio runs separately (locally or hosted at `*.sanity.studio`) and connects to your Sanity-hosted dataset. Your Next.js app reads via the public CDN endpoint or via the GraphQL API.

### Define a schema

```ts
// schemas/post.ts
import { defineField, defineType } from 'sanity';

export const post = defineType({
  name: 'post',
  type: 'document',
  fields: [
    defineField({ name: 'title', type: 'string', validation: (R) => R.required() }),
    defineField({ name: 'slug', type: 'slug', options: { source: 'title' } }),
    defineField({ name: 'publishedAt', type: 'datetime' }),
    defineField({ name: 'author', type: 'reference', to: [{ type: 'author' }] }),
    defineField({ name: 'cover', type: 'image' }),
    defineField({
      name: 'content',
      type: 'array',
      of: [{ type: 'block' }, { type: 'image' }],   // Portable Text — Sanity's rich content
    }),
  ],
});
```

### Read with GROQ

```ts
// lib/sanity.ts
import { createClient } from 'next-sanity';

export const sanity = createClient({
  projectId: process.env.NEXT_PUBLIC_SANITY_PROJECT_ID!,
  dataset: process.env.NEXT_PUBLIC_SANITY_DATASET!,
  apiVersion: '2024-10-01',
  useCdn: process.env.NODE_ENV === 'production',
});
```

```ts
// queries.ts
import { groq } from 'next-sanity';

export const POST_BY_SLUG = groq`
  *[_type == "post" && slug.current == $slug][0] {
    _id,
    title,
    publishedAt,
    "slug": slug.current,
    author->{ name, "avatar": avatar.asset->url },
    cover { asset->{ url, metadata { dimensions } } },
    content
  }
`;
```

```tsx
// app/blog/[slug]/page.tsx (Server Component)
import { sanity } from '@/lib/sanity';
import { POST_BY_SLUG } from '@/queries';
import { PortableText } from '@portabletext/react';
import { notFound } from 'next/navigation';

export const revalidate = 60;

export default async function PostPage({ params }: { params: { slug: string } }) {
  const post = await sanity.fetch(POST_BY_SLUG, { slug: params.slug });
  if (!post) notFound();

  return (
    <article>
      <h1>{post.title}</h1>
      <PortableText value={post.content} />
    </article>
  );
}
```

GROQ is denser than GraphQL but fits how Sanity content nests — `author->` joins the relation; `[0]` gets a single document. Steep learning curve; pays off when you need it.

### Visual editing + real-time preview

`next-sanity@9.8` integrates with Sanity's **Visual Editing**:

```tsx
import { VisualEditing } from 'next-sanity';
{process.env.NEXT_PUBLIC_SANITY_PREVIEW === 'true' && <VisualEditing />}
```

Editors click a piece of content in the live site; Studio jumps to that field. Designers and marketers love it. **The reason to pick Sanity over Payload** if non-engineers are doing day-to-day content work.

### Why Sanity wins

- **Best-in-class editor UX** — Studio is highly customizable and beautiful.
- **Real-time collaboration** in Studio.
- **Portable Text** for structured rich content (better than HTML for typed editing).
- **Visual Editing** with click-to-edit overlays.
- **GROQ** is more expressive than GraphQL for nested fetches.
- **Generous free tier** (3 users, 10K documents, 1M API requests/mo).

### Why Sanity loses

- **GROQ learning curve** — your team has to learn a new query language.
- **Pricing scales by user + document** — large editorial teams get expensive fast.
- **Vendor lock-in** — Sanity hosts your data; migrating off is non-trivial.
- **Two deploys** — your Next.js app + the Studio (though both can live in one repo).

For marketing + content sites: **Sanity 3 is genuinely best-in-class for editor experience**.

---

## 5. Contentful — Enterprise Choice

### What

`contentful@11` is the long-standing managed enterprise CMS. Used by Spotify, Adidas, BMW. Strong workflow + approvals + multi-locale + asset management. The 2026 reality: Contentful is **less innovative** than Sanity or Payload, but **more enterprise-ready**.

### Setup

```bash
npm i contentful@11 @contentful/rich-text-react-renderer@16
```

```ts
// lib/contentful.ts
import { createClient } from 'contentful';

export const contentful = createClient({
  space: process.env.CONTENTFUL_SPACE!,
  accessToken: process.env.CONTENTFUL_DELIVERY_TOKEN!,
});
```

```tsx
// app/blog/[slug]/page.tsx
import { contentful } from '@/lib/contentful';
import { documentToReactComponents } from '@contentful/rich-text-react-renderer';
import { notFound } from 'next/navigation';

export const revalidate = 60;

export default async function PostPage({ params }: { params: { slug: string } }) {
  const { items } = await contentful.getEntries({
    content_type: 'post',
    'fields.slug': params.slug,
    limit: 1,
  });
  const post: any = items[0];
  if (!post) notFound();

  return (
    <article>
      <h1>{post.fields.title}</h1>
      <div>{documentToReactComponents(post.fields.content)}</div>
    </article>
  );
}
```

### When Contentful wins

- **Workflow + approvals** — content goes through review before publish.
- **Roles + permissions** at fine granularity.
- **Multi-locale** with first-class fallbacks.
- **Asset management** with auto-resize, focal point, transformations.
- **SLAs** big enterprises require.
- **Compliance** (SOC2, ISO 27001) built in.

### When Contentful loses

- **Pricing** is enterprise-tier; "Free" tier is very limited. Real teams pay $300–10000+/month.
- **Slower innovation** than Sanity / Payload.
- **Less polished editor experience** than Sanity Studio.
- **No self-host** option.

For Fortune 500 companies with editorial workflows: Contentful. For everyone else: cheaper alternatives serve better.

---

## 6. Strapi 5 — OSS Admin + API

### What

`@strapi/strapi@5` (released Oct 2024) is the most-installed OSS headless CMS. Self-hosted. Generates REST and GraphQL APIs from a content-type builder UI. Strong plugin ecosystem.

### Setup

```bash
npx create-strapi-app@latest my-cms --quickstart
```

Spawns a Node.js server with a SQLite DB and an admin UI. Define content types via the admin UI (clicking) or in code (`/src/api/post/content-types/post/schema.json`).

### Connect from Next.js

```ts
// lib/strapi.ts
const STRAPI_URL = process.env.NEXT_PUBLIC_STRAPI_URL!;

export async function strapiFetch<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${STRAPI_URL}${path}`, {
    ...init,
    headers: {
      ...init?.headers,
      Authorization: `Bearer ${process.env.STRAPI_TOKEN}`,
    },
    next: { revalidate: 60 },
  });
  if (!res.ok) throw new Error(`Strapi ${res.status}`);
  return res.json();
}
```

```tsx
// app/blog/[slug]/page.tsx
import { strapiFetch } from '@/lib/strapi';
import { notFound } from 'next/navigation';

export default async function PostPage({ params }: { params: { slug: string } }) {
  const data: any = await strapiFetch(
    `/api/posts?filters[slug][$eq]=${params.slug}&populate=*`
  );
  const post = data.data?.[0];
  if (!post) notFound();

  return (
    <article>
      <h1>{post.attributes.title}</h1>
      {/* render rich content */}
    </article>
  );
}
```

### Why Strapi wins

- **Fully OSS + free** — self-host on a $5 VPS.
- **REST and GraphQL out of the box.**
- **Plugin ecosystem** — i18n, SEO, GraphQL, Cloudinary, Algolia integrations.
- **Admin UI runs on the same server** as the API.
- **Content-type builder UI** — non-technical team members can add fields.

### Why Strapi loses to Payload 3 in 2026

- **Separate process** — Strapi runs as a Node server; Payload 3 runs **inside** your Next.js app.
- **No Local API** — every Strapi read is an HTTP call.
- **TypeScript support is bolted-on** vs Payload's TS-native config.
- **Performance** at large content counts is weaker than Payload + Postgres.
- **Less Next.js-aware** — patterns require manual integration.

If you already run Strapi: keep it. For new projects with Next.js: **Payload 3 is the better fit**.

---

## 7. The "MDX in git" Path — When You Don't Need a CMS

For engineering-led content (changelog, dev blog, technical docs):

### Setup

```bash
npm i next-mdx-remote@5 gray-matter@4 reading-time@1 rehype-pretty-code@0.13
```

```
content/
├── posts/
│   ├── 2026-01-15-launching-v2.mdx
│   ├── 2026-02-03-react-19-deep-dive.mdx
│   └── ...
└── pages/
    └── about.mdx
```

```mdx
---
title: "Launching v2"
slug: launching-v2
publishedAt: 2026-01-15
author: anh
---

# Launching v2

Today we shipped...

<CallToAction href="/signup">Try it</CallToAction>
```

### Render

```tsx
// app/blog/[slug]/page.tsx
import { compileMDX } from 'next-mdx-remote/rsc';
import { promises as fs } from 'fs';
import path from 'path';
import { notFound } from 'next/navigation';

const components = {
  CallToAction: ({ href, children }: any) => <a href={href} className="btn">{children}</a>,
};

export default async function PostPage({ params }: { params: { slug: string } }) {
  let raw: string;
  try {
    raw = await fs.readFile(path.join(process.cwd(), 'content/posts', `${params.slug}.mdx`), 'utf8');
  } catch {
    notFound();
  }

  const { content, frontmatter } = await compileMDX<{ title: string; publishedAt: string }>({
    source: raw,
    options: { parseFrontmatter: true },
    components,
  });

  return (
    <article>
      <h1>{frontmatter.title}</h1>
      {content}
    </article>
  );
}

export async function generateStaticParams() {
  const files = await fs.readdir(path.join(process.cwd(), 'content/posts'));
  return files.map((f) => ({ slug: f.replace(/\.mdx$/, '') }));
}
```

### Why MDX-in-git wins for engineering content

- **Zero infrastructure** — no CMS, no admin UI.
- **Versioned with code** — same git history.
- **Inline React components** in markdown — design system primitives just work.
- **PR review for content** — same code review process.
- **Free, fast, simple.**

### Tools for MDX-driven content sites

| Tool | Purpose |
|------|---------|
| `next-mdx-remote@5` | The standard MDX renderer for App Router |
| `contentlayer2` (community fork) | Type-safe MDX content with frontmatter validation |
| `velite@0.2` | Build-time MDX → typed content (lighter than contentlayer) |
| `nextra@3` | Doc-site framework on top of Next.js |
| `fumadocs@13` | Modern docs framework (Next.js + MDX + great DX) |
| `markdoc@0.4` | Stripe's safer alternative to MDX (no JSX in content) |

For a docs site or engineer blog: **`fumadocs` or `nextra` + MDX in git**. Don't add a CMS.

---

## 8. Real-World Patterns

### Pattern A — On-publish revalidation

When content is published, the page must update without a redeploy.

**Payload 3** — built-in afterChange hook → `revalidatePath`:

```ts
hooks: {
  afterChange: [({ doc }) => revalidatePath(`/blog/${doc.slug}`)],
}
```

**Sanity 3** — Sanity webhooks → Next.js Route Handler:

```ts
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';
import { isValidSignature, SIGNATURE_HEADER_NAME } from '@sanity/webhook';

export async function POST(req: Request) {
  const sig = req.headers.get(SIGNATURE_HEADER_NAME);
  const body = await req.text();
  const isValid = await isValidSignature(body, sig!, process.env.SANITY_WEBHOOK_SECRET!);
  if (!isValid) return new Response('Invalid signature', { status: 401 });

  const { _type, slug } = JSON.parse(body);
  revalidateTag(_type);
  if (slug?.current) revalidatePath(`/blog/${slug.current}`);
  return Response.json({ revalidated: true });
}
```

In Sanity dashboard → API → Webhooks → add `https://yourapp.com/api/revalidate`.

**Contentful** — same pattern via Contentful webhooks.

### Pattern B — Image optimization

CMSes ship images at huge resolutions. Always pipe through `next/image`:

```tsx
import Image from 'next/image';
<Image src={post.cover.url} width={1200} height={630} alt={post.title} />
```

For Sanity images, use `@sanity/image-url@1` to generate optimized variants:

```ts
import imageUrlBuilder from '@sanity/image-url';
const builder = imageUrlBuilder(sanity);
export const urlFor = (source: any) => builder.image(source);

<Image src={urlFor(post.cover).width(1200).format('webp').url()} ... />
```

For Payload + Vercel Blob / S3: Payload generates resize URLs automatically.

### Pattern C — Multi-locale

**Sanity** — first-class via `@sanity/document-internationalization@3`.
**Contentful** — built into the schema.
**Payload 3** — `localized: true` field flag generates per-locale variants.

```ts
{
  name: 'title',
  type: 'text',
  localized: true,
}
```

```tsx
import { getPayload } from 'payload';
const payload = await getPayload({ config });
const post = await payload.find({ collection: 'posts', locale: 'fr', where: { slug: { equals: params.slug } } });
```

Pair with `next-intl@3` (covered in `10.production/05-i18n.md`) for the Next.js routing layer.

### Pattern D — Preview drafts before publish

Draft mode lets editors see unpublished content on the live site at a different URL.

**Next.js draft mode**:

```ts
// app/api/draft/route.ts
import { draftMode } from 'next/headers';

export async function GET(req: Request) {
  // Verify the user is an editor (Auth.js session check — covered in 07.nextjs/04-auth.md)
  // ...
  (await draftMode()).enable();
  const slug = new URL(req.url).searchParams.get('slug');
  return Response.redirect(new URL(`/blog/${slug}`, req.url));
}
```

In your page:

```tsx
import { draftMode } from 'next/headers';

const isDraft = (await draftMode()).isEnabled;
const post = await payload.find({
  collection: 'posts',
  draft: isDraft,                                   // includes drafts when in draft mode
  where: { slug: { equals: params.slug } },
});
```

The editor adds a "Preview" button in Studio/Admin pointing at `/api/draft?slug=foo`. They see the unpublished version; everyone else sees the published one.

---

## 9. Authentication for the CMS

| CMS | Auth model |
|-----|------------|
| **Payload 3** | Built-in users collection; uses bcrypt + cookies. Pluggable to OAuth providers via `@payloadcms/plugin-oauth2`. |
| **Sanity** | Sanity manages editor auth via Studio login (Google, GitHub, email). |
| **Contentful** | Contentful manages editor auth + SSO on enterprise. |
| **Strapi 5** | Built-in admin users + role-based permissions; OAuth via `@strapi/plugin-users-permissions`. |

For app users (your end customers, not editors): keep using **Auth.js v5** (covered in `07.nextjs/04-auth.md`) — separate from CMS editor auth.

---

## 10. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Page doesn't update after publish | Add a webhook → `revalidatePath` route handler |
| Image transforms via CMS slow first paint | Pre-resize via `next/image`; cache aggressively |
| GROQ query times out on large datasets | Add Sanity indexes (`indexed: true` on schema fields) |
| Editor accidentally publishes a draft | Enable scheduled publish + approval workflows |
| Multi-locale fields show as "missing" in Sanity | Use a localization plugin; default to source language fallback |
| Strapi server overloaded by reads | Cache at the CDN; use Strapi's response cache plugin |
| Payload `/admin` slow on Vercel | Increase the function's memory; `/admin` does heavy work on first load |
| Sanity API requests rack up cost | Use `next: { revalidate: ... }` to cache; `useCdn: true` in production |
| Long-form content shows raw HTML in some places | CMS rich-text → React component renderer (PortableText for Sanity, RichText for Payload) |
| Visual editing breaks on dynamic routes | Use Sanity's `defineLive` + `next-sanity@9.8`'s draft mode |
| Editors see fields they shouldn't | Field-level access control (Payload + Sanity both support) |
| Migration between CMSes is painful | Keep schemas in code; export → transform → import scripts |
| Engineers writing content directly via SQL | Use the API/admin UI; it triggers hooks that revalidate, send notifications, etc. |
| Search across CMS content is slow | Index into MeiliSearch / Algolia (covered in `13.service-stack/04-search.md`) |

---

## 11. Decision Tree

```
Who edits the content?
│
├── Engineers only → MDX in git (no CMS)
├── Designers + marketers → Sanity 3 (best UX)
├── Non-technical editorial team → Contentful or Sanity
├── Product managers + occasional engineering → Payload 3 (engineering-led)
└── A non-technical content team using Notion already → Notion as CMS for tiny sites

What's the deploy story?
│
├── Self-hosted Next.js + want one process → Payload 3 (lives inside Next.js)
├── Self-hosted Strapi already running → keep it; or migrate to Payload over time
├── Managed CMS, no servers to run → Sanity / Contentful / Storyblok
└── Static MDX in git → no extra infrastructure

What's the scale and budget?
│
├── < 10K docs, indie team, $0 budget → Payload 3 self-hosted on a $5 VPS
├── 10K–100K docs, design-led → Sanity 3 free / Pro tier
├── Enterprise + workflow + SLA → Contentful
└── Marketing + visual editing → Storyblok or Sanity with Visual Editing

Real-time / collaborative editing?
│
├── Sanity Studio has it built in
├── Payload 3 single-user-locking only
└── For full CRDT collaboration → use a different layer (covered in 09.beyond-web/03-realtime.md)

i18n?
│
├── Built-in Sanity / Contentful / Payload via field-level localization
└── Then map to Next.js via next-intl 3 (covered in 10.production/05-i18n.md)
```

---

## 12. What This Topic Connects To

- **`07.nextjs/01-fundamentals.md`** — Server Components fetching CMS content.
- **`07.nextjs/03-data-fetching.md`** — `revalidate`, `revalidatePath`, draft mode.
- **`07.nextjs/07-seo-metadata.md`** — generating per-post metadata from CMS data.
- **`10.production/01-rich-text-editors.md`** — Lexical / Tiptap / Portable Text for the in-CMS editor.
- **`10.production/05-i18n.md`** — pairing CMS multi-locale with `next-intl@3`.
- **`13.service-stack/04-search.md`** — indexing CMS content into MeiliSearch / Algolia.
- **`11.capabilities/02-file-uploads.md`** — image / asset storage on R2 / S3 / Vercel Blob.

---

## 13. Summary

| Tool | Pick when |
|------|-----------|
| **Payload 3** + Postgres | Self-hosted Next.js + engineering-led + want Local API |
| **Sanity 3** + GROQ | Marketing/content sites + best-in-class editor UX |
| **Contentful** | Enterprise + workflow + multi-locale + SLAs |
| **Strapi 5** | OSS-first + plugin ecosystem; older Node CMS shops |
| **Storyblok 3** | Visual editing for non-tech editors |
| **MDX in git + `fumadocs@13` / `nextra@3`** | Engineer-edited content; docs sites; blogs |
| **Notion as CMS** | Tiny sites where Notion is already in the team's workflow |

| Rule | Why |
|------|-----|
| Don't add a CMS until non-engineers will edit content regularly | Static MDX is faster and simpler |
| Always use `revalidatePath` on publish | Avoid full redeploys for content changes |
| Index CMS content into a search engine | CMSes aren't search engines (covered in `13.service-stack/04-search.md`) |
| Use `next/image` for CMS images | Bandwidth + LCP wins |
| Field-level access control | Editors shouldn't see internal-only fields |
| Schema in code, not in admin UI | Versioned, reviewed, migratable |
| Separate editor auth from end-user auth | Different lifecycles, different security boundaries |

---

## 14. Further reading

- [Payload 3 docs](https://payloadcms.com/docs)
- [Sanity 3 docs](https://www.sanity.io/docs)
- [Contentful docs](https://www.contentful.com/developers/docs/)
- [Strapi 5 docs](https://docs.strapi.io/)
- [Storyblok docs](https://www.storyblok.com/docs)
- [Hygraph docs](https://hygraph.com/docs)
- [`next-mdx-remote@5`](https://github.com/hashicorp/next-mdx-remote)
- [`fumadocs@13`](https://fumadocs.vercel.app/)
- [`nextra@3`](https://nextra.site/)
- [Markdoc (Stripe's MDX alternative)](https://markdoc.dev/)
