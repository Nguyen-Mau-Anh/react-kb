# Capabilities — 04. PWA: Serwist 9, Install Prompts, Push Notifications, Offline-First

> **What / Why / How** — pick **`@serwist/next@9`** for Next.js 14+ (Workbox's modern successor). PWAs in 2026 work great on Android + desktop, **partially on iOS** (notifications via PWA shipped in iOS 16.4 but require home-screen install). Build offline-first if your app's value depends on it; otherwise focus on installability + push.

---

## 1. What a PWA Actually Is in 2026

A **Progressive Web App** is just a website with three opt-in capabilities:

1. **Web App Manifest** (`manifest.json`) — tells the OS the app's name, icons, theme color, and that it's installable.
2. **Service Worker** — a script that runs in a separate thread, intercepts network requests, caches responses, and enables offline + push.
3. **HTTPS** — required for both of the above (except `localhost`).

That's it. There's no "PWA framework," no special build target. Any Next.js / Vite / Remix app can become a PWA by adding a manifest and service worker.

### What PWAs unlock

| Capability | Browser support in 2026 |
|------------|-------------------------|
| Install to home screen / dock | ✅ All browsers + iOS/Android/macOS/Windows |
| Run offline (cached shell + cached data) | ✅ All |
| Background sync (defer API calls until online) | ✅ Chrome, Edge; partial Safari |
| Push notifications | ✅ Chrome, Firefox, Edge, Safari 16.4+ on iOS (after install) |
| Periodic background sync | Chrome only |
| File handling (open `.foo` files in your PWA) | Chrome, Edge |
| Window Controls Overlay (custom title bar) | Chrome, Edge |
| WebRTC, WebGPU, Web Bluetooth, etc. | Same as the underlying browser |

### When to actually go PWA

- Your app has **offline value** (note-taking, drawing, expense tracker, drafts).
- You want **push notifications** without paying for native iOS/Android development.
- You want users to **launch from a home screen icon** as a standalone window.
- You want to **install on Windows/macOS/ChromeOS** as a "native-feeling" app.

### When PWA is overkill

- Marketing site or blog → just static HTML.
- B2B SaaS dashboard that's always online → service worker complexity outweighs benefit.
- Heavy-graphics game → use a real native build (Tauri / Electron).
- iOS-first audience that won't install — iOS PWAs require home-screen install for push.

---

## 2. The Real Choices in 2026

| Library | npm | Engine | Best for |
|---------|-----|--------|----------|
| **Serwist 9** | `@serwist/next@9.0`, `@serwist/sw@9.0`, `@serwist/window@9.0` | Modern Workbox successor | **Default for Next.js 14+ in 2026** |
| **next-pwa 5.6** | `next-pwa@5.6` | Workbox-based | Maintenance only — author moved to Serwist |
| **Vite PWA** | `vite-plugin-pwa@0.20` | Workbox-based | Vite + React SPA |
| **Workbox 7** | `workbox-build@7`, `workbox-window@7` | Google's library | Lower-level, framework-agnostic |
| **Raw Service Worker** | hand-written | — | Tiny apps with simple caching |
| **next-offline** | `next-offline@5` | Older | Don't use |

`next-pwa@5.6` was *the* Next.js PWA library 2020–2023. Its author started **Serwist** in 2024 as a from-scratch rewrite for the App Router era. **In 2026, Serwist is the clear default for Next.js**. `next-pwa` still works but is maintenance-only — don't pick it for new code.

For Vite + React: **`vite-plugin-pwa@0.20`** is the equivalent default, also Workbox-based.

---

## 3. The Web App Manifest — Step One

The manifest is a JSON file telling browsers your site is installable. In Next.js 14, generate it via `app/manifest.ts`:

```ts
// app/manifest.ts
import type { MetadataRoute } from 'next';

export default function manifest(): MetadataRoute.Manifest {
  return {
    name: 'Acme Tasks',
    short_name: 'Acme',                    // shown under the home-screen icon
    description: 'A simple task manager',
    start_url: '/',
    scope: '/',                            // pages outside this scope open in browser
    display: 'standalone',                 // 'standalone' | 'minimal-ui' | 'fullscreen' | 'browser'
    background_color: '#ffffff',           // splash-screen color
    theme_color: '#2563eb',                // status-bar color on Android
    orientation: 'portrait',
    categories: ['productivity'],

    icons: [
      { src: '/icon-192.png', sizes: '192x192', type: 'image/png' },
      { src: '/icon-512.png', sizes: '512x512', type: 'image/png' },
      { src: '/icon-512-maskable.png', sizes: '512x512', type: 'image/png', purpose: 'maskable' },
    ],

    shortcuts: [
      { name: 'New task',     short_name: 'New',     url: '/tasks/new',     icons: [{ src: '/icon-new.png', sizes: '96x96' }] },
      { name: 'Today',        short_name: 'Today',   url: '/today' },
    ],

    screenshots: [
      { src: '/screen-mobile.png',  sizes: '390x844',   type: 'image/png', form_factor: 'narrow' },
      { src: '/screen-desktop.png', sizes: '1280x720',  type: 'image/png', form_factor: 'wide' },
    ],
  };
}
```

That serves `/manifest.webmanifest` automatically. Reference it from your root layout via the Metadata API:

```tsx
// app/layout.tsx
export const metadata = {
  manifest: '/manifest.webmanifest',
  themeColor: '#2563eb',
  appleWebApp: {
    capable: true,
    statusBarStyle: 'default',
    title: 'Acme Tasks',
  },
};
```

### What each field actually does

- **`display: 'standalone'`** — when launched from the home screen, opens without a URL bar. The single biggest "looks like an app" lever.
- **`maskable` icons** — Android adaptive icons; the OS crops them to the system shape. Always include alongside regular icons.
- **`shortcuts`** — long-press the home-screen icon to see these as a context menu (Android, Chrome on desktop).
- **`screenshots`** — show up in Chrome's install prompt and in `chrome://apps`. Make them real product screenshots, not marketing assets.
- **`scope`** — defines which URLs open in the standalone window. Set to `/` unless you're partitioning a multi-section site.

### Required icons

At minimum:
- `icon-192.png` — 192×192, purpose `any`.
- `icon-512.png` — 512×512, purpose `any`.
- `icon-512-maskable.png` — 512×512, purpose `maskable`. Logo must fit inside the **safe zone** (center 80%) since the OS can crop the corners.

Use `pwa-asset-generator@6` to generate the full set from one source SVG:

```bash
npx pwa-asset-generator@6 logo.svg public/ --icon-only --type png --background "#fff"
```

---

## 4. Serwist 9 — Service Worker for Next.js 14+

### What

`@serwist/next@9.0` is a Next.js plugin that:
1. Compiles a custom service worker file (e.g., `app/sw.ts`) into the build output.
2. Generates a **precache manifest** of all static assets at build time.
3. Sets caching strategies for runtime fetches (HTML, CSS, JS, images).
4. Wires up registration via `@serwist/window@9.0` on the client.

### Setup

```bash
npm i @serwist/next@9.0
npm i -D serwist@9.0
```

```ts
// next.config.ts
import withSerwistInit from '@serwist/next';

const withSerwist = withSerwistInit({
  swSrc: 'app/sw.ts',         // your service worker source
  swDest: 'public/sw.js',     // compiled output
  cacheOnNavigation: true,    // navigations use the SW cache too
  reloadOnOnline: true,       // page auto-reloads when network returns
  disable: process.env.NODE_ENV === 'development',
});

export default withSerwist({
  // ...the rest of your next config
});
```

```ts
// app/sw.ts
import { defaultCache } from '@serwist/next/worker';
import type { PrecacheEntry, SerwistGlobalConfig } from 'serwist';
import { Serwist } from 'serwist';

declare global {
  interface WorkerGlobalScope extends SerwistGlobalConfig {
    __SW_MANIFEST: (PrecacheEntry | string)[] | undefined;
  }
}
declare const self: ServiceWorkerGlobalScope;

const serwist = new Serwist({
  precacheEntries: self.__SW_MANIFEST,
  skipWaiting: true,
  clientsClaim: true,
  navigationPreload: true,
  runtimeCaching: defaultCache,
  fallbacks: {
    entries: [
      {
        url: '/offline',                       // fallback HTML for offline navigations
        matcher: ({ request }) => request.destination === 'document',
      },
    ],
  },
});

serwist.addEventListeners();
```

### What's happening

- **`self.__SW_MANIFEST`** — Serwist injects the build-time list of static assets here. They're precached on first install.
- **`defaultCache`** — a sensible Workbox-flavored cache config: HTML (NetworkFirst), CSS/JS (StaleWhileRevalidate), images (CacheFirst with 30-day expiry), Google Fonts (CacheFirst), etc.
- **`skipWaiting` + `clientsClaim`** — new SW activates immediately on next page load (vs the default "waiting for all tabs to close").
- **`navigationPreload: true`** — the browser fires the network request in parallel with SW startup. Faster first response on slow devices.
- **`fallbacks`** — when the user is offline AND the page isn't in cache, serve `/offline` instead of the browser's "no internet" error.

### The offline page

```tsx
// app/offline/page.tsx
export default function OfflinePage() {
  return (
    <main className="grid min-h-screen place-items-center p-6 text-center">
      <div>
        <h1 className="text-3xl font-bold">You&apos;re offline</h1>
        <p className="mt-2 text-muted-foreground">
          Reconnect to load this page. Your unsaved changes are still safe.
        </p>
      </div>
    </main>
  );
}
```

This page itself must be precached. Add it to the precache list in Serwist's config or set it as a static export so it ends up in `__SW_MANIFEST` automatically.

### Register the SW from the client

```tsx
// app/components/sw-register.tsx
'use client';
import { useEffect } from 'react';

export function SWRegister() {
  useEffect(() => {
    if ('serviceWorker' in navigator && process.env.NODE_ENV === 'production') {
      navigator.serviceWorker.register('/sw.js', { scope: '/' });
    }
  }, []);
  return null;
}
```

```tsx
// app/layout.tsx
import { SWRegister } from './components/sw-register';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <SWRegister />
      </body>
    </html>
  );
}
```

In Vite + React: use `vite-plugin-pwa@0.20` and its `useRegisterSW` hook from `virtual:pwa-register/react`. Same idea, different import path.

---

## 5. Caching Strategies — The Four That Matter

Workbox/Serwist define four canonical strategies. Pick by request type:

| Strategy | Behavior | When to use |
|----------|----------|-------------|
| **CacheFirst** | Look in cache; if hit, return. Fall back to network on miss. | Versioned static assets (CSS, JS, fonts) |
| **NetworkFirst** | Try network; on failure, fall back to cache. | HTML pages, JSON API responses where freshness > offline |
| **StaleWhileRevalidate** | Return cache (if any) immediately, fetch fresh in background, update cache. | Images, content that's "good enough if a few seconds stale" |
| **NetworkOnly** | Always fetch from network. | Form POSTs, mutations, anything user-data-changing |
| **CacheOnly** | Never go to network. | Pre-installed assets you've explicitly precached |

### Real per-resource config

```ts
// Inside Serwist runtimeCaching, or override defaultCache:
import { CacheFirst, NetworkFirst, StaleWhileRevalidate } from 'serwist';

runtimeCaching: [
  // Google Fonts CSS — long-lived
  {
    matcher: ({ url }) => url.hostname === 'fonts.googleapis.com',
    handler: new StaleWhileRevalidate({ cacheName: 'google-fonts-css' }),
  },
  // Google Fonts files — versioned, cache aggressively
  {
    matcher: ({ url }) => url.hostname === 'fonts.gstatic.com',
    handler: new CacheFirst({ cacheName: 'google-fonts', plugins: [/* expiry */] }),
  },
  // Images — stale-while-revalidate
  {
    matcher: ({ request }) => request.destination === 'image',
    handler: new StaleWhileRevalidate({ cacheName: 'images' }),
  },
  // API GETs — network-first with offline fallback
  {
    matcher: ({ url, request }) => url.pathname.startsWith('/api/') && request.method === 'GET',
    handler: new NetworkFirst({ cacheName: 'api', networkTimeoutSeconds: 3 }),
  },
],
```

**Most important rule: never cache POST/PUT/DELETE.** Mutations must always hit the network. The cache strategy applies only to GET.

### Cache expiry — critical

Without expiry, caches grow forever. Add `ExpirationPlugin`:

```ts
import { ExpirationPlugin } from 'serwist';

new CacheFirst({
  cacheName: 'images',
  plugins: [
    new ExpirationPlugin({
      maxEntries: 100,
      maxAgeSeconds: 30 * 24 * 60 * 60,   // 30 days
      purgeOnQuotaError: true,
    }),
  ],
}),
```

`purgeOnQuotaError: true` lets the cache self-evict when storage quota is hit (otherwise the SW throws and stops caching anything).

---

## 6. Install Prompts — `beforeinstallprompt`

Browsers show their own install UI (Chrome's omnibar +, Safari's "Add to Home Screen"). Some teams want a custom in-app prompt:

```tsx
'use client';
import { useEffect, useState } from 'react';

type BeforeInstallPromptEvent = Event & {
  prompt: () => Promise<void>;
  userChoice: Promise<{ outcome: 'accepted' | 'dismissed' }>;
};

export function InstallButton() {
  const [deferredPrompt, setDeferredPrompt] = useState<BeforeInstallPromptEvent | null>(null);
  const [installed, setInstalled] = useState(false);

  useEffect(() => {
    const handler = (e: Event) => {
      e.preventDefault();                                       // don't auto-show Chrome's bar
      setDeferredPrompt(e as BeforeInstallPromptEvent);
    };
    window.addEventListener('beforeinstallprompt', handler);
    window.addEventListener('appinstalled', () => setInstalled(true));
    return () => window.removeEventListener('beforeinstallprompt', handler);
  }, []);

  if (installed || !deferredPrompt) return null;

  async function handleInstall() {
    await deferredPrompt!.prompt();
    const { outcome } = await deferredPrompt!.userChoice;
    if (outcome === 'accepted') setInstalled(true);
    setDeferredPrompt(null);
  }

  return (
    <button onClick={handleInstall} className="rounded bg-blue-600 px-3 py-1.5 text-white">
      Install Acme
    </button>
  );
}
```

### iOS install — different model

Safari **doesn't fire `beforeinstallprompt`**. iOS requires the user to tap the share button → "Add to Home Screen" manually. Provide an **on-page tutorial** for iOS users:

```tsx
function isIos() {
  if (typeof window === 'undefined') return false;
  return /iPad|iPhone|iPod/.test(navigator.userAgent) && !('MSStream' in window);
}

function isInStandaloneMode() {
  if (typeof window === 'undefined') return false;
  return ('standalone' in window.navigator && (window.navigator as any).standalone)
    || window.matchMedia('(display-mode: standalone)').matches;
}

export function IosInstallTip() {
  if (!isIos() || isInStandaloneMode()) return null;
  return (
    <div className="rounded-md border bg-blue-50 p-3 text-sm">
      Install: tap <ShareIcon /> then "Add to Home Screen".
    </div>
  );
}
```

Show this once per user (cookie or `localStorage` flag). Don't be annoying about it — show it on day 2 of usage, dismissible, never again.

### Engagement heuristics matter

Browsers only fire `beforeinstallprompt` when the user has shown **engagement**:

- Visited the site at least twice with at least 5 minutes between visits (Chrome's "engagement score" threshold).
- Had a successful interaction (click, form submit).

A brand-new visitor doesn't see the install prompt no matter what. The platform decides; you can only handle the event when it fires.

---

## 7. Push Notifications — The 2026 Reality

### iOS support

**Web Push on iOS Safari shipped in iOS 16.4 (March 2023).** Critical caveats:

- Only works if the user has **installed the PWA to home screen** first.
- macOS Safari supports it without install (16.4+).
- Notification permission UX is identical to native.

So: on iOS, the install prompt is your gating step before push.

### Setup — `web-push@3` server, native APIs client

```bash
npm i web-push@3
```

#### Server: VAPID keys + storing subscriptions

```ts
// scripts/generate-vapid-keys.ts (run once)
import webpush from 'web-push';
const keys = webpush.generateVAPIDKeys();
console.log(keys); // save publicKey + privateKey to env
```

```ts
// lib/push.ts
import webpush from 'web-push';
import { db } from '@/server/db';

webpush.setVapidDetails(
  'mailto:hi@example.com',
  process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!,
  process.env.VAPID_PRIVATE_KEY!
);

export async function sendPushTo(userId: string, payload: { title: string; body: string; url?: string }) {
  const subs = await db.pushSubscription.findMany({ where: { userId } });
  await Promise.allSettled(
    subs.map(async (sub) => {
      try {
        await webpush.sendNotification(
          { endpoint: sub.endpoint, keys: { p256dh: sub.p256dh, auth: sub.auth } },
          JSON.stringify(payload)
        );
      } catch (err: any) {
        if (err.statusCode === 410 || err.statusCode === 404) {
          // Subscription expired — remove it
          await db.pushSubscription.delete({ where: { id: sub.id } });
        }
        throw err;
      }
    })
  );
}
```

```prisma
// prisma/schema.prisma
model PushSubscription {
  id        String   @id @default(cuid())
  userId    String
  endpoint  String   @unique
  p256dh    String
  auth      String
  createdAt DateTime @default(now())
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
}
```

#### Client: subscribe to push

```tsx
'use client';
import { useState } from 'react';

function urlBase64ToUint8Array(base64: string) {
  const padding = '='.repeat((4 - (base64.length % 4)) % 4);
  const b64 = (base64 + padding).replace(/-/g, '+').replace(/_/g, '/');
  const raw = atob(b64);
  return new Uint8Array([...raw].map((c) => c.charCodeAt(0)));
}

export function EnableNotifications() {
  const [status, setStatus] = useState<'idle' | 'subscribed' | 'error'>('idle');

  async function subscribe() {
    if (!('serviceWorker' in navigator) || !('PushManager' in window)) {
      setStatus('error');
      return;
    }

    const registration = await navigator.serviceWorker.ready;
    const permission = await Notification.requestPermission();
    if (permission !== 'granted') return;

    const subscription = await registration.pushManager.subscribe({
      userVisibleOnly: true,                           // required by browsers
      applicationServerKey: urlBase64ToUint8Array(process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!),
    });

    await fetch('/api/push/subscribe', {
      method: 'POST',
      body: JSON.stringify(subscription),
    });
    setStatus('subscribed');
  }

  return <button onClick={subscribe} disabled={status === 'subscribed'}>Enable notifications</button>;
}
```

#### Service worker: receive and display

```ts
// app/sw.ts (add to your existing Serwist file)
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? { title: 'Notification', body: '' };
  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icon-192.png',
      badge: '/badge-72.png',                          // monochrome silhouette for status bar
      data: { url: data.url ?? '/' },
    })
  );
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  event.waitUntil(
    clients.matchAll({ type: 'window' }).then((wins) => {
      for (const win of wins) {
        if (win.url.includes(event.notification.data.url) && 'focus' in win) return win.focus();
      }
      return clients.openWindow(event.notification.data.url);
    })
  );
});
```

`userVisibleOnly: true` is required by every browser — you must show a visible notification when push fires; silent push isn't allowed.

### When to ask for notification permission

**Never on first page load.** A cold permission prompt has < 5% acceptance and primes users to deny on every future visit.

The pattern that works:
1. User signs up.
2. User completes a meaningful action (creates first task, connects an integration).
3. Show an in-app card explaining why notifications matter (e.g., "Get reminders when tasks are due").
4. **The card's button** triggers `Notification.requestPermission()`.

This staged ask gets 30–60% acceptance on real apps vs the 2–5% of cold prompts.

### Alternatives to web push

If web push's complexity isn't worth it:

| Tool | Best for | Notes |
|------|----------|-------|
| **OneSignal** | Cross-platform push (web + iOS native + Android native) | Free up to 10K MAU |
| **Knock** | Notification orchestration (push + email + Slack + in-app) | Per-MAU pricing |
| **Resend Audiences + email** | Email instead of push | Simpler; covered in `11.capabilities/03-email.md` |
| **Pusher Beams** | Managed web push | Free up to 1K subscribers |

For a SaaS where push isn't core: **email or OneSignal** is usually less work than rolling your own VAPID setup.

---

## 8. Offline-First Patterns — Beyond "Cache the Page"

True offline-first means the app **functions** offline, not just renders.

### Pattern A — Read-only offline (most apps)

Cache the shell + recent data. If the user's offline, they can browse what they've already loaded but can't mutate. This is what `defaultCache` gives you.

### Pattern B — Optimistic write + sync when online

The user creates/edits while offline. The mutation is queued in IndexedDB. When online, the queue replays.

```ts
import { BackgroundSyncPlugin } from 'serwist';

new NetworkOnly({
  plugins: [
    new BackgroundSyncPlugin('mutations-queue', {
      maxRetentionTime: 24 * 60,                       // try for 24 hours
    }),
  ],
}),
```

This wraps failed POST/PUT/DELETE in a queue; the SW retries them when network returns. Browsers without Background Sync (Safari) just lose the request — handle that case explicitly with TanStack Query's offline queue or your own IndexedDB store.

### Pattern C — Full local-first (Linear, Notion-style)

The whole DB lives in the client (IndexedDB / SQLite-WASM). The server is the sync target, not the source of truth.

| Tool | npm | Best for |
|------|-----|----------|
| **TanStack DB** | `@tanstack/react-db@0.x` | Local-first reactive queries from TanStack |
| **Replicache** | `replicache@15` | Battle-tested, used by Reflect/Linear |
| **PowerSync** | `@powersync/web@1` | SQLite-WASM with Postgres sync |
| **Dexie 4** | `dexie@4`, `dexie-react-hooks@1` | Lightweight IndexedDB wrapper, manual sync |
| **electric-sql** | `electric-sql@0.12` | Postgres → client SQLite sync |
| **RxDB 15** | `rxdb@15` | Rx-based reactive DB with multiple backends |

Local-first is its own discipline; covered briefly here. For a real implementation, pick Replicache or PowerSync and dedicate weeks to it. Most apps don't need it — Pattern A with TanStack Query is enough.

### IndexedDB the modern way — Dexie 4

For ad-hoc client storage:

```bash
npm i dexie@4 dexie-react-hooks@1
```

```ts
import Dexie, { Table } from 'dexie';

interface DraftTask {
  id?: number;
  title: string;
  body: string;
  createdAt: Date;
}

class AppDB extends Dexie {
  drafts!: Table<DraftTask, number>;
  constructor() {
    super('acme-app');
    this.version(1).stores({ drafts: '++id, createdAt' });
  }
}

export const db = new AppDB();
```

```tsx
'use client';
import { useLiveQuery } from 'dexie-react-hooks';
import { db } from '@/lib/dexie';

function Drafts() {
  const drafts = useLiveQuery(() => db.drafts.orderBy('createdAt').reverse().toArray());
  if (!drafts) return null;
  return <ul>{drafts.map((d) => <li key={d.id}>{d.title}</li>)}</ul>;
}
```

`useLiveQuery` re-renders when underlying IndexedDB rows change. ~6 KB gzipped. The 2026 default for "I just need to persist this in the browser."

---

## 9. Background Sync & Periodic Sync

| API | Purpose | Browser support |
|-----|---------|-----------------|
| **Background Sync** | Replay failed mutations when online | Chrome, Edge, Firefox-experimental |
| **Periodic Background Sync** | Wake the SW on a schedule (hourly+) to fetch updates | Chrome only |

```ts
// In a Client Component, register a sync tag
const reg = await navigator.serviceWorker.ready;
await reg.sync.register('sync-tasks');

// In sw.ts
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-tasks') {
    event.waitUntil(replayFailedMutations());
  }
});
```

For Periodic Sync (Chrome desktop only, requires the user to have the PWA installed):

```ts
const status = await navigator.permissions.query({ name: 'periodic-background-sync' as any });
if (status.state === 'granted') {
  await reg.periodicSync.register('refresh-data', { minInterval: 60 * 60 * 1000 });
}
```

In `sw.ts`:
```ts
self.addEventListener('periodicsync', (event) => {
  if (event.tag === 'refresh-data') event.waitUntil(refreshDataInCache());
});
```

Browser support is narrow. Treat periodic sync as a **bonus optimization**, not a feature you depend on.

---

## 10. Updating the App — The Killer Detail

A long-standing PWA gotcha: the user has the SW installed, you deploy a new version, **the old SW keeps serving cached HTML for hours or days** until they close every tab.

### Force update on next reload

Serwist's `skipWaiting: true` + `clientsClaim: true` (set in §4) handles this — new SW activates the moment the page reloads.

### Show an "update available" banner

Detect the new SW and prompt the user:

```tsx
'use client';
import { useEffect, useState } from 'react';

export function UpdateBanner() {
  const [update, setUpdate] = useState<ServiceWorkerRegistration | null>(null);

  useEffect(() => {
    if (!('serviceWorker' in navigator)) return;
    navigator.serviceWorker.ready.then((reg) => {
      reg.addEventListener('updatefound', () => {
        const newWorker = reg.installing;
        if (!newWorker) return;
        newWorker.addEventListener('statechange', () => {
          if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
            setUpdate(reg);
          }
        });
      });
    });
  }, []);

  if (!update) return null;
  return (
    <div className="fixed bottom-4 left-1/2 -translate-x-1/2 rounded-md border bg-background p-3 shadow-lg">
      <span>New version available.</span>
      <button onClick={() => { update.waiting?.postMessage({ type: 'SKIP_WAITING' }); window.location.reload(); }}>
        Update
      </button>
    </div>
  );
}
```

```ts
// Add to sw.ts
self.addEventListener('message', (event) => {
  if (event.data?.type === 'SKIP_WAITING') self.skipWaiting();
});
```

Without this, your "fixed bug 2 weeks ago" is still live for thousands of returning users.

---

## 11. Testing PWAs

### Lighthouse PWA audit

```bash
# Chrome DevTools → Lighthouse → Mobile + PWA category → Generate report
```

Lighthouse reports installability score (0–100) and lists specific failures. **Aim for "Installable" badge in PWA category before launch.** Real launchpad: Lighthouse score ≥ 90 in Performance, Accessibility, Best Practices, AND PWA installable.

### Real-device testing

- **Android**: Chrome on a real device. Simulators differ subtly.
- **iOS**: Safari on a real iPhone. Simulators don't support push.
- **Desktop**: Chrome on Windows/macOS install behavior differs by ~5%; test both.

### Service worker debugging

Chrome DevTools → Application → Service Workers:
- **"Update on reload"** — every page reload re-registers the SW. Essential during development.
- **"Bypass for network"** — disables the SW entirely. Toggle when SW caching is masking a real bug.
- **"Clear storage"** — wipes everything; restart from a clean slate.

`chrome://serviceworker-internals/` shows every registered SW system-wide.

---

## 12. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| SW cache won't update after deploy | `skipWaiting: true` + `clientsClaim: true` in Serwist config; show update banner |
| `manifest.webmanifest` 404s | Make sure `app/manifest.ts` exports default; check the URL — Next.js serves it at `/manifest.webmanifest` |
| Install button never shows on Chrome | Engagement threshold not met (visit twice with 5+ min gap), OR manifest invalid, OR SW not registered |
| iOS doesn't show push opt-in | User hasn't installed to home screen — required on iOS 16.4+ |
| Push notifications fire but no notification appears | `userVisibleOnly: true` not set, OR icon path wrong, OR `showNotification` not in `event.waitUntil` |
| Service worker breaks `localhost:3000` dev | Set `disable: process.env.NODE_ENV === 'development'` in Serwist config |
| Cached HTML shows old layout for hours | Use NetworkFirst for HTML, not CacheFirst; OR ship cache-busting hashes in the manifest |
| 410 errors when sending push | Subscription expired (user unsubscribed); delete from DB on `webpush.sendNotification` failure |
| App crashes when offline because of font CDN | Cache the fonts via runtime cache OR self-host with `next/font` (covered in `07.nextjs/01-fundamentals.md`) |
| `beforeinstallprompt` not firing locally | Only fires on HTTPS or `localhost`; manifest must validate; engagement criteria met |
| New SW immediately controls page but UI breaks | Probably `clientsClaim` mid-session race. Show update banner instead of auto-claim |
| IndexedDB quota errors on old devices | `purgeOnQuotaError: true` on cache plugins; cap cache sizes |

---

## 13. Decision Tree

```
Are you building a PWA?
│
├── Static marketing site / blog → NO. Don't add SW complexity.
├── Always-online B2B SaaS dashboard → MAYBE just for installability + push, not offline
└── Offline-relevant app (notes, drawing, drafts, expense tracker) → YES, full PWA

What framework are you on?
├── Next.js 14+ App Router → @serwist/next@9 (default 2026)
├── Next.js Pages Router → next-pwa@5.6 (still works) OR migrate to Serwist
├── Vite + React → vite-plugin-pwa@0.20
└── Hand-rolled SW → only if you have very specific needs

Need push notifications?
├── Want managed cross-platform → OneSignal (web + native)
├── Web-only, full control → web-push@3 + VAPID + Serwist push handler
├── Notification orchestration (push + email + Slack) → Knock
└── Skip and use email → Resend + react-email (covered in 11.capabilities/03-email.md)

Need true offline-first (mutations work offline)?
├── Read-only with optimistic UI → TanStack Query + BackgroundSyncPlugin
├── Light client storage → Dexie 4 + manual sync
├── Full local-first → Replicache or PowerSync (weeks of setup)
└── Most apps don't need this. Don't over-engineer.
```

---

## 14. What This Topic Connects To

- **`07.nextjs/01-fundamentals.md`** — `next/font` and `next/image` integrate with PWA caching strategies.
- **`07.nextjs/07-seo-metadata.md`** — `app/manifest.ts` lives next to `app/sitemap.ts` and `app/robots.ts`.
- **`04.state-and-data/03-data-fetching.md`** — TanStack Query pairs with offline cache for read-only patterns.
- **`10.production/04-observability.md`** — Track SW install rate, push subscription rate, offline-fallback hits as real metrics.
- **`08.ecosystem/04-real-project.md`** — Adding installability + push reminders to the Task Manager is a 1-day PR.

---

## Summary

| Tool | Pick when |
|------|-----------|
| `@serwist/next@9.0` | PWA on Next.js 14+ in 2026 |
| `vite-plugin-pwa@0.20` | PWA on Vite + React |
| `next-pwa@5.6` | Maintaining existing project; don't pick for new code |
| `web-push@3` + VAPID | DIY web push notifications |
| OneSignal / Knock / Pusher Beams | Managed push without VAPID setup |
| Dexie 4 | Client-side persistence with reactive queries |
| Replicache / PowerSync | Full local-first apps |
| `pwa-asset-generator@6` | Generate the full icon set from one SVG |

| Rule | Why |
|------|-----|
| Add `app/manifest.ts` and SW from day one if going PWA | Adding offline retroactively is much harder |
| Set `display: 'standalone'` and maskable icons | The biggest "feels like an app" levers |
| `skipWaiting + clientsClaim + update banner` | Stale SWs are the #1 PWA pain point |
| Stage push permission requests | Cold prompts get < 5% acceptance |
| Never cache POST/PUT/DELETE | Mutations must always hit the network |
| Add expiry to all runtime caches | Storage fills up over time without it |
| Provide an `/offline` fallback page | Browser's "no internet" page is hostile |
| iOS push requires home-screen install | Plan the install funnel before promising notifications |

---

## Further reading

- [Serwist docs](https://serwist.pages.dev/)
- [`@serwist/next` reference](https://serwist.pages.dev/docs/next)
- [Workbox docs](https://developer.chrome.com/docs/workbox)
- [Web App Manifest spec (MDN)](https://developer.mozilla.org/en-US/docs/Web/Manifest)
- [web.dev — Push Notifications](https://web.dev/articles/push-notifications-overview)
- [web.dev — App-like](https://web.dev/explore/app-like)
- [WebKit blog — Web Push for iOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)
- [Replicache docs](https://doc.replicache.dev/)
- [Dexie docs](https://dexie.org/docs/)
- [Next.js — PWA example with Serwist](https://github.com/vercel/next.js/tree/canary/examples/progressive-web-app)
