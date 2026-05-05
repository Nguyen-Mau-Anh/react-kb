# Next.js — 07. SEO & Metadata: Metadata API, Open Graph, JSON-LD, Sitemap, Robots

> **What / Why / How** — Next.js 14 ships a typed metadata API. Use it. Skip `react-helmet`. Generate OG images dynamically.

---

## 1. What "SEO" Actually Means for a React App

Not just "rank on Google." The full surface area:

| Surface | What it cares about |
|---------|----------------------|
| **Google / Bing search results** | `<title>`, `<meta name="description">`, headings, server-rendered HTML, structured data (JSON-LD), canonical URLs |
| **Social previews (Twitter/X, LinkedIn, Slack, iMessage)** | Open Graph tags (`og:title`, `og:description`, `og:image`), Twitter Card tags |
| **Browser tab + history** | `<title>`, `<link rel="icon">`, `theme-color` |
| **Crawlers & search infra** | `robots.txt`, `sitemap.xml`, `Last-Modified` headers, response status codes |
| **AI / RAG indexers (Perplexity, ChatGPT browse, Claude search)** | Same as Google — clean SSR HTML + JSON-LD wins |

Next.js 14 has a built-in API for every one of these. **Don't add `react-helmet@6`** — that's a Pages Router / Vite era library; the App Router has a better, server-rendered, type-safe replacement.

---

## 2. The Metadata API — Static Form

### What

Export a `metadata` object (or function) from a `layout.tsx` or `page.tsx`. Next.js renders the corresponding `<head>` tags during SSR. Metadata exports must live in **Server Components** (which is the default — see `07.nextjs/01-fundamentals.md`).

```tsx
// app/layout.tsx — root metadata applies to every page unless overridden
import type { Metadata } from 'next';

export const metadata: Metadata = {
  metadataBase: new URL('https://example.com'),  // resolves relative URLs in OG images, canonicals
  title: {
    default: 'My App',
    template: '%s | My App',                     // child pages: "About | My App"
  },
  description: 'The example app for the React KB',
  applicationName: 'My App',
  authors: [{ name: 'Anh Nguyen', url: 'https://example.com' }],
  keywords: ['react', 'next.js', 'kb'],
  creator: 'Anh Nguyen',
  publisher: 'Anh Nguyen',
  formatDetection: { email: false, address: false, telephone: false },
};
```

```tsx
// app/about/page.tsx — overrides only title and description
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'About',                  // becomes "About | My App" via the parent template
  description: 'Who built this and why.',
};

export default function AboutPage() { return <h1>About</h1>; }
```

The framework merges parent metadata with child metadata. Children only need to declare what they override.

### Why this beats `react-helmet@6`

| | `react-helmet@6` | Next.js Metadata API |
|--|-------------------|----------------------|
| Renders during SSR | Yes (with extra setup) | Yes — built-in |
| Type-safe (`Metadata` interface) | No | Yes |
| Async metadata (DB lookup) | Awkward | Native — `generateMetadata` is `async` |
| Conflicts when nested deeply | Possible | Resolved by framework merge rules |
| Extra runtime cost | Hooks + library | Zero |
| Maintained for App Router | No | Yes (Vercel team) |

---

## 3. The Metadata API — Dynamic Form (`generateMetadata`)

For metadata that depends on route params, search params, or fetched data, export an `async function` instead.

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata, ResolvingMetadata } from 'next';
import { notFound } from 'next/navigation';

type Props = { params: { slug: string } };

export async function generateMetadata(
  { params }: Props,
  parent: ResolvingMetadata
): Promise<Metadata> {
  const post = await db.post.findUnique({ where: { slug: params.slug } });
  if (!post) return {};

  // Compose from the parent's openGraph.images so you don't lose the default
  const previousImages = (await parent).openGraph?.images ?? [];

  return {
    title: post.title,
    description: post.excerpt,
    alternates: {
      canonical: `/blog/${params.slug}`,
    },
    openGraph: {
      title: post.title,
      description: post.excerpt,
      type: 'article',
      publishedTime: post.publishedAt.toISOString(),
      authors: [post.author.name],
      images: [
        { url: `/blog/${params.slug}/opengraph-image`, width: 1200, height: 630 },
        ...previousImages,
      ],
    },
    twitter: {
      card: 'summary_large_image',
      title: post.title,
      description: post.excerpt,
      images: [`/blog/${params.slug}/opengraph-image`],
    },
  };
}

export default async function BlogPost({ params }: Props) {
  const post = await db.post.findUnique({ where: { slug: params.slug } });
  if (!post) notFound();
  return <article>{post.body}</article>;
}
```

Key facts:
- `generateMetadata` and the page's data fetch are **deduplicated** — Next.js shares the fetch result if you use `cache()` or the same fetch URL.
- Returning `{}` on missing data is fine — `notFound()` in the page component still triggers the 404 response.
- `metadataBase` (set in the root layout) makes relative URLs in `openGraph.images` resolve correctly.

---

## 4. Open Graph & Twitter Cards — What Actually Renders Where

| Tag | Used by |
|-----|---------|
| `<meta property="og:title">`, `og:description`, `og:image`, `og:url`, `og:type` | LinkedIn, Slack, Discord, iMessage, Facebook, Telegram, Pinterest |
| `<meta name="twitter:card">`, `twitter:image`, `twitter:title`, `twitter:description`, `twitter:creator` | X / Twitter |
| `og:image` minimum dimensions | 1200×630 (Twitter `summary_large_image`, Open Graph standard) |
| File size | Keep < 8 MB (Twitter limit); aim for < 1 MB for fast preview |
| Format | PNG or JPEG. Avoid SVG (Twitter ignores). Avoid WebP/AVIF (Slack/iMessage support inconsistent). |

### The single image that covers everything

Twitter falls back to `og:image` if `twitter:image` is absent. So you can ship one URL:

```ts
openGraph: {
  images: [{ url: '/og.png', width: 1200, height: 630 }],
},
twitter: { card: 'summary_large_image' },
```

---

## 5. Dynamic OG Images — `opengraph-image` File Convention

This is one of the most underused Next.js 14 features. A file named `opengraph-image.tsx` next to a `page.tsx` becomes a **per-route generated PNG** — rendered as JSX, served as an image.

```tsx
// app/blog/[slug]/opengraph-image.tsx
import { ImageResponse } from 'next/og';

export const runtime = 'edge';
export const size = { width: 1200, height: 630 };
export const contentType = 'image/png';

export default async function OGImage({ params }: { params: { slug: string } }) {
  const post = await db.post.findUnique({
    where: { slug: params.slug },
    select: { title: true, author: { select: { name: true } } },
  });

  return new ImageResponse(
    (
      <div
        style={{
          width: '100%',
          height: '100%',
          display: 'flex',
          flexDirection: 'column',
          justifyContent: 'space-between',
          padding: 80,
          background: 'linear-gradient(135deg, #0f172a, #1e293b)',
          color: 'white',
          fontFamily: 'Inter',
        }}
      >
        <div style={{ fontSize: 28, opacity: 0.7 }}>example.com / blog</div>
        <div style={{ fontSize: 64, fontWeight: 700, lineHeight: 1.1 }}>
          {post?.title ?? 'Untitled'}
        </div>
        <div style={{ fontSize: 28, opacity: 0.85 }}>
          by {post?.author.name}
        </div>
      </div>
    ),
    { ...size }
  );
}
```

This is a Server Component running on the Edge runtime that returns a PNG. The image URL is `/blog/<slug>/opengraph-image` — Next auto-injects it into Open Graph and Twitter metadata for that route.

### Why this is huge

- **No external service** (no Cloudinary, no Bannerbear) for per-route social previews.
- **Per-locale, per-author, per-product** OG images — generated on demand, cached per route.
- Same JSX you already know — no Photoshop pipeline.
- Fonts via `fetch` to a `.ttf` (the `ImageResponse` constructor accepts a `fonts` array).

### Sibling files for the rest of the icon family

```
app/icon.png              → /icon.png
app/apple-icon.png        → /apple-icon.png
app/manifest.ts           → /manifest.json (PWA)
app/robots.ts             → /robots.txt (next section)
app/sitemap.ts            → /sitemap.xml (next section)
```

You can also generate any of them dynamically via `.tsx` versions (e.g., `app/icon.tsx` returning an `ImageResponse`).

---

## 6. `robots.txt` — Two Real Approaches

### Static — `app/robots.ts`

```ts
// app/robots.ts
import type { MetadataRoute } from 'next';

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      { userAgent: '*', allow: '/', disallow: ['/api/', '/admin/'] },
      { userAgent: 'GPTBot', disallow: '/' },                  // block OpenAI's crawler
      { userAgent: 'CCBot', disallow: '/' },                   // block Common Crawl
      { userAgent: 'anthropic-ai', disallow: '/' },            // block Anthropic's crawler
    ],
    sitemap: 'https://example.com/sitemap.xml',
    host: 'https://example.com',
  };
}
```

The route `/robots.txt` is generated automatically. You don't need `next-sitemap@4` or any plugin.

### Static file — when you want full control

Drop a literal `public/robots.txt` if your config has hand-tuned `Crawl-delay`, complex `Allow`/`Disallow` ordering, or vendor-specific directives that don't fit the typed `MetadataRoute.Robots` shape.

---

## 7. Sitemap — Three Real Patterns

### Pattern A: Static — `app/sitemap.ts`

```ts
// app/sitemap.ts
import type { MetadataRoute } from 'next';

export default function sitemap(): MetadataRoute.Sitemap {
  return [
    { url: 'https://example.com',         lastModified: new Date(), changeFrequency: 'daily',   priority: 1.0 },
    { url: 'https://example.com/about',   lastModified: new Date(), changeFrequency: 'monthly', priority: 0.5 },
    { url: 'https://example.com/pricing', lastModified: new Date(), changeFrequency: 'monthly', priority: 0.7 },
  ];
}
```

### Pattern B: Dynamic from a database

```ts
// app/sitemap.ts
import type { MetadataRoute } from 'next';

export const revalidate = 3600; // regenerate every hour

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await prisma.post.findMany({
    where: { published: true },
    select: { slug: true, updatedAt: true },
  });
  return [
    { url: 'https://example.com', lastModified: new Date(), priority: 1.0 },
    ...posts.map(p => ({
      url: `https://example.com/blog/${p.slug}`,
      lastModified: p.updatedAt,
      changeFrequency: 'weekly' as const,
      priority: 0.8,
    })),
  ];
}
```

### Pattern C: Multiple sitemaps (>50,000 URLs — Google's per-file limit)

For very large sites: sitemap **index** with multiple sitemap files. Next.js 14 supports it via `generateSitemaps`:

```ts
// app/sitemap.ts
export async function generateSitemaps() {
  const total = await prisma.post.count();
  const pages = Math.ceil(total / 50_000);
  return Array.from({ length: pages }, (_, i) => ({ id: i }));
}

export default async function sitemap({ id }: { id: number }): Promise<MetadataRoute.Sitemap> {
  const posts = await prisma.post.findMany({ skip: id * 50_000, take: 50_000, select: { slug: true, updatedAt: true } });
  return posts.map(p => ({ url: `https://example.com/blog/${p.slug}`, lastModified: p.updatedAt }));
}
```

Routes generated: `/sitemap/0.xml`, `/sitemap/1.xml`, ... plus the index at `/sitemap.xml`.

### Where `next-sitemap@4` still wins

The `next-sitemap@4` package generates static `.xml` files at build time. Real reasons to still pick it:
- You're on a static export (`output: 'export'`) and want `.xml` written to `out/`.
- You want the legacy Pages Router setup to keep working.
- You want fancy features like priority maps with regex patterns.

For App Router with SSR or ISR, the built-in `app/sitemap.ts` is the right answer.

---

## 8. Canonical URLs

Set on individual pages or in metadata:

```ts
export const metadata: Metadata = {
  alternates: {
    canonical: '/blog/my-post',
    languages: {
      'en-US': '/en-US/blog/my-post',
      'fr-FR': '/fr-FR/blog/my-post',
    },
  },
};
```

`metadataBase` (set in root layout) prefixes the path automatically. Canonical URLs are critical for:
- Pages reachable via multiple URLs (with/without trailing slash, with/without query).
- Internationalized routes (Google needs to know which is the "primary" version).
- Pagination — `?page=2` should canonical to either itself or page 1, depending on your strategy.

---

## 9. JSON-LD — Structured Data for Rich Results

JSON-LD is the format Google reads to surface rich results: stars on product pages, FAQ accordions, author bylines, breadcrumbs, etc. Put a `<script type="application/ld+json">` in your page.

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next';

export default async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await db.post.findUnique({ where: { slug: params.slug } });
  if (!post) notFound();

  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'BlogPosting',
    headline: post.title,
    image: [`https://example.com/blog/${post.slug}/opengraph-image`],
    datePublished: post.publishedAt.toISOString(),
    dateModified: post.updatedAt.toISOString(),
    author: { '@type': 'Person', name: post.author.name, url: `https://example.com/author/${post.author.handle}` },
    publisher: { '@type': 'Organization', name: 'Example', logo: { '@type': 'ImageObject', url: 'https://example.com/logo.png' } },
    description: post.excerpt,
    mainEntityOfPage: `https://example.com/blog/${post.slug}`,
  };

  return (
    <article>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
      />
      <h1>{post.title}</h1>
      {/* ... */}
    </article>
  );
}
```

### Real schema types you'll use

| Type | When |
|------|------|
| `BlogPosting` / `Article` / `NewsArticle` | Articles, blog posts |
| `Product` + `Offer` + `AggregateRating` | E-commerce product pages |
| `FAQPage` + `Question` / `Answer` | FAQs (lets Google show accordion in results) |
| `BreadcrumbList` | Site hierarchy (Home > Blog > Post) |
| `Organization` + `Person` | Author bylines, company info |
| `Recipe` | Cooking content |
| `Course` / `Event` / `JobPosting` | Schema.org types Google has rich snippets for |

Reference all schemas at [schema.org/docs/schemas.html](https://schema.org/docs/schemas.html). Verify your output with [Google's Rich Results Test](https://search.google.com/test/rich-results).

### Why `dangerouslySetInnerHTML` here

JSON-LD is text content, not HTML, but React escapes inline content by default. `dangerouslySetInnerHTML` is the official recommended pattern for JSON-LD in React (the [Next.js docs themselves use it](https://nextjs.org/docs/app/building-your-application/optimizing/metadata#json-ld)).

---

## 10. Real SEO Checklist for Launch

- [ ] `metadataBase` set in `app/layout.tsx`
- [ ] Default `title` template configured in root metadata
- [ ] Every public page has a real `description` (not the default)
- [ ] Open Graph + Twitter Card metadata on every page that gets shared
- [ ] `opengraph-image.tsx` per content type (blog post, product, profile)
- [ ] `app/icon.png` and `apple-icon.png` (or `.tsx` generated equivalents)
- [ ] `app/manifest.ts` for PWA installability + theme color
- [ ] `app/sitemap.ts` covering all crawlable routes
- [ ] `app/robots.ts` with `Sitemap:` line + crawler allowlist
- [ ] Canonical URLs on every page (especially international and paginated)
- [ ] JSON-LD on content pages where rich results are valuable
- [ ] Server-rendered HTML for crawler-visible content (no client-only-rendered text)
- [ ] HTTPS only, redirects 301'd to canonical host (`www.` vs no-`www.` settled)
- [ ] `next/image` `alt` text on every meaningful image
- [ ] Headings hierarchy: one `<h1>` per page, semantic order
- [ ] Verified ownership in Google Search Console + Bing Webmaster
- [ ] Monitor `core-web-vitals` (covered in `06.testing-perf/02-performance.md`)

---

## 11. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `og:image` shows a broken URL on social previews | Set `metadataBase` in root layout — relative URLs need it |
| OG image looks blurry on Twitter | Use `summary_large_image` card type and 1200×630 PNG |
| `metadata` export from a Client Component does nothing | Metadata can only be exported from Server Components; remove `'use client'` from that file |
| `<title>` is the literal `%s` template string | Child page didn't override `title`; the template only kicks in when `title` is set in a child |
| Sitemap doesn't include database content | Add `revalidate` to `app/sitemap.ts` so it re-renders periodically |
| `react-helmet` warnings in App Router | Don't use `react-helmet@6` here; use the Metadata API |
| OG image route has stale content | `opengraph-image.tsx` is cached forever by default; export `revalidate` if your data changes |
| Multiple `<h1>` tags on a page | Pick one. Sub-sections use `<h2>`, never another `<h1>` |
| `description` truncated in Google SERPs | Keep under ~160 characters; ~50 chars for `title` |
| Search Console says "URL not in sitemap" | Confirm it's actually included in `app/sitemap.ts` and resubmit the sitemap |

---

## 12. Decision Tree

```
Does the metadata depend on route params or fetched data?
│   YES → generateMetadata (async)
│   NO  → metadata (static const)

Need a unique social preview per page?
│   YES → opengraph-image.tsx with ImageResponse
│   NO  → A single /og.png in public/, referenced from root metadata

Is the site < 1,000 routes and you control all of them?
│   YES → app/sitemap.ts (static array)
│   NO  ↓
Is content driven by a DB and updates often?
│   YES → app/sitemap.ts that fetches from DB + revalidate = 3600
│   NO  ↓
> 50,000 URLs?
│   YES → generateSitemaps for the index pattern
│   NO  → app/sitemap.ts is enough

Does this content type qualify for a Schema.org rich result type?
│   YES → JSON-LD via dangerouslySetInnerHTML on the relevant pages
│   NO  → Skip JSON-LD; standard metadata is enough
```

---

## Summary

| Tool | Use for |
|------|---------|
| `metadata` const export | Static metadata per route |
| `generateMetadata` async export | Metadata derived from params or DB |
| `opengraph-image.tsx` | Per-route generated PNG for social previews |
| `app/icon.tsx`, `app/apple-icon.tsx` | Generated favicons |
| `app/sitemap.ts` (+ `generateSitemaps`) | Sitemap with optional sharding |
| `app/robots.ts` | `robots.txt` with typed rules |
| `app/manifest.ts` | PWA manifest |
| `<script type="application/ld+json">` + `dangerouslySetInnerHTML` | Structured data for rich results |

| Rule | Why |
|------|-----|
| Set `metadataBase` once in the root layout | Relative OG image URLs depend on it |
| Use the title `template` for consistent branding | Every child page automatically gets the suffix |
| Generate dynamic OG images with `ImageResponse` | One JSX file replaces a whole image-rendering service |
| Don't add `react-helmet` to App Router | The built-in API is strictly better |
| Validate JSON-LD with Google's Rich Results Test | Catches schema typos before they cost rankings |

---

## Further reading

- [Next.js — Metadata API](https://nextjs.org/docs/app/building-your-application/optimizing/metadata)
- [Next.js — File-based metadata (opengraph-image, etc.)](https://nextjs.org/docs/app/api-reference/file-conventions/metadata)
- [Next.js — `ImageResponse` (`next/og`)](https://nextjs.org/docs/app/api-reference/functions/image-response)
- [Next.js — `sitemap.xml`](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/sitemap)
- [Next.js — `robots.txt`](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/robots)
- [Schema.org — full vocabulary](https://schema.org/)
- [Google — Rich Results Test](https://search.google.com/test/rich-results)
- [Twitter — Card validator](https://cards-dev.twitter.com/validator)
