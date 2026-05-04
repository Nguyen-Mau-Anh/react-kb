# Next.js — 05. Deployment: Vercel vs Docker on Railway/Fly.io vs Self-Host

> **What / Why / How** — Vercel is the easy mode. Docker on a small PaaS is the cheap mode. Self-host only when you must.

---

## 1. The Real Hosting Options for Next.js 14 in 2026

| Platform | Pricing model | Best for | Real teams using it |
|----------|---------------|----------|---------------------|
| **Vercel** | Pay per usage (compute, bandwidth, cache reads) | The default. Built by the Next.js team; supports every feature on day one. | Linear, Netflix Jobs, Loom marketing, Replit, Notion's marketing site |
| **Railway** | $5/mo base + per-resource | Docker-friendly PaaS, hands-off Postgres + Redis | Many indie SaaS, side projects |
| **Fly.io** | Per-second compute, regional | Multi-region apps, long-running connections, low-cost VPS feel | Litestream demos, Resend transactional infra examples |
| **AWS / GCP / Azure with Docker** | Variable | Enterprise compliance, existing cloud account, internal apps | Most Fortune-500 companies running Next.js internally |
| **OpenNext on Cloudflare / AWS Lambda** | Per-request, free tier generous | Zero-cost hobby, Cloudflare-native | OpenNext-deployed apps (`opennextjs/opennext`) |
| **Self-host on a VPS (Hetzner, DigitalOcean droplet)** | $5–20/mo flat | Full control, predictable cost | Indie devs, internal tools |

This file walks through the three picks you'll actually use: **Vercel**, **Docker on Railway/Fly.io**, **self-hosted with `docker compose`**.

---

## 2. Vercel — The Easy Mode

### Why Vercel wins by default

Vercel is built by the Next.js team. Every feature ships there first:
- React Server Components ✅
- Server Actions with full caching ✅
- ISR with on-demand revalidation ✅
- Edge Functions / Middleware ✅
- Image optimization ✅
- `next/font` self-hosted Google Fonts ✅
- Preview deployments per PR ✅
- Branch deployments with custom domains ✅
- Analytics, Speed Insights, Logs ✅

On any other platform you may pay extra setup work for some of these (cache invalidation, ISR, image optimization). On Vercel, all of it just works.

### Setup — minutes, not hours

```bash
npm i -g vercel@34
vercel login
vercel --prod
```

Or connect a GitHub repo via the dashboard at vercel.com. Every push to `main` deploys to production; every PR gets a unique preview URL.

### Required environment variables

Set in **Project → Settings → Environment Variables**, scoped per environment (Production / Preview / Development):

```
DATABASE_URL=...
AUTH_SECRET=...
AUTH_GITHUB_ID=...
AUTH_GITHUB_SECRET=...
NEXT_PUBLIC_API_URL=...    # only NEXT_PUBLIC_* are exposed to the browser
```

Pull them locally:
```bash
vercel env pull .env.local
```

### `vercel.json` — when you need it (rarely)

```json
{
  "regions": ["iad1"],
  "functions": {
    "app/api/**/route.ts": { "maxDuration": 30 }
  },
  "headers": [
    { "source": "/(.*)", "headers": [{ "key": "X-Frame-Options", "value": "DENY" }] }
  ]
}
```

Most projects don't need a `vercel.json` at all. Default behavior is correct.

### Cost reality check

Free tier (Hobby) is generous: 100 GB bandwidth, 100 GB-hours compute. Real-world ranges for a small SaaS:

| App profile | Tier | Monthly |
|-------------|------|---------|
| Personal site, < 1K visits/mo | Hobby | $0 |
| Indie SaaS, ~50K monthly visits | Pro | $20 + small overage |
| Production app with heavy ISR + Edge Middleware | Pro | $20–200 |
| High-traffic e-commerce or social app | Enterprise | Custom (often 4–6 figures) |

The two surprise-bill culprits: **bandwidth** (large images / videos served through Next.js) and **edge middleware invocations** (every request runs middleware, including bots). Mitigate with `images.remotePatterns` for an image CDN, and a tight middleware `matcher` config (covered in `07.nextjs/02-routing.md`).

---

## 3. Docker on Railway / Fly.io / Render — The Cheap Mode

### Why pick this over Vercel

- **Predictable monthly cost.** $5–25/mo regardless of traffic.
- **Long-running processes.** Background workers, websockets, persistent connections — Vercel's serverless model isn't ideal for these.
- **Supabase / external DB on the same network.** Railway's "Postgres + app in same project" pattern keeps DB latency sub-ms.
- **No vendor-specific knobs.** A Dockerfile runs on any Docker host; you can move providers in a day.

### What you give up

- ISR with on-demand revalidation needs extra work — Next.js's default cache is filesystem-based per server. With one container, fine. Multi-region or autoscaling → need a shared cache (Redis with `@neshca/cache-handler@1` or Upstash with `next-cache-handler-redis`).
- Image optimization on a tiny VPS can spike CPU. Either bake images at build time or front it with Cloudflare Images / `imgproxy`.

### Real Dockerfile for Next.js 14 standalone output

Next.js supports a "standalone" output that copies only the runtime files needed — under 100 MB image typical:

```js
// next.config.js
module.exports = {
  output: 'standalone',
};
```

```dockerfile
# Dockerfile — multi-stage, ~80MB final image
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
RUN addgroup -g 1001 nodejs && adduser -S nextjs -u 1001
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder --chown=nextjs:nodejs /app/public ./public
USER nextjs
EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME=0.0.0.0
CMD ["node", "server.js"]
```

The official Next.js Dockerfile reference is at [vercel/next.js — examples/with-docker](https://github.com/vercel/next.js/tree/canary/examples/with-docker).

### Railway — the simplest "Vercel alternative"

```bash
npm i -g @railway/cli
railway login
railway init
railway up
```

Railway auto-detects the Dockerfile, builds it, deploys it, and gives you a public URL. Add Postgres with a click; the connection string is injected as `DATABASE_URL` automatically.

Pricing as of 2026: $5/mo flat per service + actual resource usage. For a Next.js app + Postgres at small scale: ~$10–15/mo total.

### Fly.io — multi-region, more control

```bash
brew install flyctl
fly launch
fly deploy
```

`fly launch` reads your Dockerfile and asks where to deploy. To go multi-region:

```bash
fly regions add lhr fra hkg
fly scale count 1 --region iad,lhr,fra,hkg
```

Now requests are served from the nearest region. Postgres can be regional too (with read replicas via `fly-replay` headers).

Fly.io pricing: per-second compute. A `shared-cpu-1x` machine running 24/7 is ~$1.94/mo per region.

---

## 4. Self-Hosting on a VPS — Cheap and Controlled

### When this makes sense

- Internal tools.
- Side projects you want to keep cheap forever.
- Compliance constraints requiring a specific region or provider you control.

### Stack: Hetzner CPX11 ($4.50/mo) + `docker compose` + Caddy 2 + Postgres

```yaml
# docker-compose.yml
services:
  app:
    build: .
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://app:${POSTGRES_PASSWORD}@db:5432/app
      AUTH_SECRET: ${AUTH_SECRET}
      NODE_ENV: production
    depends_on: [db]

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data

  caddy:
    image: caddy:2.8-alpine
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config

volumes:
  pgdata:
  caddy_data:
  caddy_config:
```

```caddyfile
# Caddyfile — automatic HTTPS via Let's Encrypt
yourdomain.com {
  reverse_proxy app:3000
}
```

Deploy:
```bash
ssh root@vps
git clone <repo> && cd <repo>
docker compose up -d --build
```

Caddy 2 fetches Let's Encrypt certs automatically on first boot. **Total time from a fresh Hetzner VPS to HTTPS-served Next.js: ~10 minutes.**

### Adding zero-downtime deploys

```bash
docker compose pull
docker compose up -d --no-deps --build app
```

For real zero-downtime: front with Caddy or Traefik 3 doing rolling deploys, or use `coolify@4` (open-source PaaS that wraps `docker compose`) or `dokku@0.34` (Heroku-style git-push deploys).

---

## 5. The Database Question — Where Do You Put Postgres?

| Option | $/mo at small scale | Notes |
|--------|---------------------|-------|
| **Neon** (`neon.tech`) | $0 free tier, $19+ pro | Serverless Postgres, instant branching for previews. Works great with Vercel preview deployments. |
| **Supabase** (`supabase.com`) | $0 free tier, $25+ pro | Postgres + auth + storage + realtime. Good if using Supabase Auth. |
| **Railway Postgres** | ~$5/mo | Co-located with Railway app — sub-ms latency. |
| **Fly Postgres** | $1.94/mo per replica | Co-located with Fly app, regional read replicas possible. |
| **PlanetScale** | $39+/mo | MySQL, not Postgres. Branching is excellent but pricier. |
| **Self-hosted on the same VPS** | $0 incremental | Cheapest. Backups become your responsibility. |

For the typical "Next.js + Prisma + Auth.js" stack in 2026: **Neon** with Vercel, or **Railway Postgres** with a Docker app. Both keep latency low and simplify connection-string management.

### Neon's preview-branch trick

In Vercel, install the Neon integration → every PR preview gets its own Postgres branch (a copy-on-write fork of production). Each preview is a full isolated DB — no fixture drift, no shared-DB collisions.

---

## 6. ISR / On-Demand Revalidation Across Multiple Servers

If you have one server (single Docker container, single Vercel function), Next.js's filesystem cache works fine.

The moment you have **two or more instances** behind a load balancer, ISR breaks: instance A revalidates `/products`, instance B still serves the stale page. The fix: a shared cache.

### The standard solutions

| Setup | Solution |
|-------|----------|
| Vercel | Built-in. No action needed. |
| Single Docker | Default filesystem cache works. |
| Multi-instance Docker on same host | Shared volume for `.next/cache`. Hacky. |
| Multi-instance / autoscaling / Kubernetes | Redis-backed cache handler |

The Redis cache handler config (Next.js 14):

```js
// next.config.js
module.exports = {
  cacheHandler: require.resolve('./cache-handler.js'),
  cacheMaxMemorySize: 0, // disable in-memory cache
};
```

```js
// cache-handler.js — using @neshca/cache-handler@1.4
const { CacheHandler } = require('@neshca/cache-handler');
const createRedisCache = require('@neshca/cache-handler/redis-strings').default;
const Redis = require('ioredis');

CacheHandler.onCreation(async () => ({
  handlers: [await createRedisCache({ client: new Redis(process.env.REDIS_URL) })],
}));

module.exports = CacheHandler;
```

For Upstash (serverless Redis): use `@neshca/cache-handler/redis-stack` or the official Upstash integration.

---

## 7. CI/CD — GitHub Actions Recipes

### Vercel: nothing required

Vercel watches your GitHub repo. Every push deploys. **You don't need a workflow file.** If you want CI checks before merge, run them separately:

```yaml
# .github/workflows/ci.yml
name: CI
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm run lint
      - run: npm run test -- --run
      - run: npm run build  # catches build-time errors before Vercel does
```

### Docker push → Railway / Fly / GHCR

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
      # then trigger your platform — examples:
      # Railway:
      - run: |
          curl -X POST https://backboard.railway.app/graphql/v2 \
            -H "Authorization: Bearer ${{ secrets.RAILWAY_TOKEN }}" \
            -d '{"query":"mutation { serviceInstanceRedeploy(...) { id } }"}'
      # Fly.io:
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - run: flyctl deploy --remote-only
        env: { FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }} }
```

---

## 8. Production Checklist

Before pointing real users at it:

- [ ] `AUTH_SECRET` set; rotated from any leaked dev value
- [ ] All `process.env.*` configured per environment; no hardcoded secrets in code
- [ ] `next.config.js` `images.remotePatterns` allowlists every external image host you actually use (otherwise `next/image` errors at runtime)
- [ ] `reactStrictMode: true` left on (default)
- [ ] `output: 'standalone'` if Docker
- [ ] DB connection pool sized correctly — `prisma@5` defaults to one connection per `PrismaClient`; for serverless, use `pgbouncer` or Neon's pooler endpoint
- [ ] Sentry / DataDog wired up (`@sentry/nextjs@8` is the standard) with source maps uploaded
- [ ] `next/font` used everywhere — no `<link rel="stylesheet" href="https://fonts.googleapis.com/...">` left
- [ ] `Cache-Control` headers set on static assets via `next.config.js` `headers()`
- [ ] CSP header configured (start with `Content-Security-Policy-Report-Only` to discover violations before enforcing)
- [ ] HSTS, X-Frame-Options, X-Content-Type-Options headers set
- [ ] `robots.txt` / `sitemap.xml` (covered in `07.nextjs/07-seo-metadata.md`)
- [ ] Lighthouse score in CI: LCP < 2.5s, INP < 200ms, CLS < 0.1
- [ ] Backup strategy: automatic daily DB snapshots, tested restore at least once

---

## 9. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `next/image` returns 400 for an external URL | Add the host to `images.remotePatterns` in `next.config.js` |
| Vercel preview shows old DB schema | Use Neon branching, or run `prisma migrate deploy` in a build hook |
| Docker image is 1GB+ | Use `output: 'standalone'` and the multi-stage Dockerfile above |
| ISR not invalidating across instances | Add Redis-backed cache handler |
| OAuth callback fails on preview deployments (URL mismatch) | Configure provider to accept `https://*.vercel.app/api/auth/callback/{provider}`, or use a wildcard via NextAuth's `trustHost` flag |
| Edge middleware bills exploding | Tighten the `matcher`; exclude static asset paths |
| `AUTH_SECRET` mismatch between preview and prod logs users out | Use the same value; Vercel "Encrypted" env var category preserves it across all environments |
| Build runs out of memory | `NODE_OPTIONS=--max-old-space-size=4096` in CI; Vercel Pro provides 8GB build memory |

---

## 10. Decision Tree

```
Does this app have a budget for hosted infrastructure ($20–200/mo OK)?
│   YES → Vercel (default for any Next.js project where time-to-launch matters)
│   NO  ↓
Need predictable cost (~$5–25/mo) and don't mind a Dockerfile?
│   YES → Railway with Docker + Railway Postgres (simplest cheap path)
│   NO  ↓
Need multi-region with low latency or per-second billing?
│   YES → Fly.io with regional Postgres
│   NO  ↓
Want full control on a single VPS for $5/mo?
│   YES → Hetzner / Hostinger / DigitalOcean droplet + docker compose + Caddy 2
│   NO  ↓
Need to deploy inside an existing AWS/GCP/Azure account?
│   YES → ECS Fargate / Cloud Run / Container Apps with the same standalone Dockerfile
```

---

## Summary

| Platform | Pick when |
|----------|-----------|
| Vercel | You want zero infra work and full Next.js feature parity |
| Railway | Predictable monthly cost, Docker-friendly, Postgres co-located |
| Fly.io | Multi-region, long-running processes, per-second pricing |
| Self-host (Hetzner + Caddy) | $5/mo flat, full control, internal tools |
| AWS/GCP/Azure | Enterprise compliance forces it |

| Rule | Why |
|------|-----|
| Use `output: 'standalone'` for any Docker deploy | 10× smaller image, faster cold start |
| Set `images.remotePatterns` from day one | `next/image` errors at runtime otherwise |
| Use Neon or Railway Postgres for small/mid scale | Co-located with app, cheap, branchable |
| Add Redis cache handler for multi-instance ISR | Default filesystem cache breaks across replicas |
| Wire Sentry from day one | Diagnose production issues without re-deploying |

---

## Further reading

- [Next.js — Deploying](https://nextjs.org/docs/app/building-your-application/deploying)
- [Next.js + Docker example](https://github.com/vercel/next.js/tree/canary/examples/with-docker)
- [OpenNext (deploy to Cloudflare / AWS Lambda / SST)](https://opennext.js.org/)
- [Neon — Vercel integration](https://neon.tech/docs/guides/vercel)
- [Railway docs](https://docs.railway.app/)
- [Fly.io — Run a Next.js App](https://fly.io/docs/js/frameworks/nextjs/)
- [Caddy reverse proxy](https://caddyserver.com/docs/quick-starts/reverse-proxy)
