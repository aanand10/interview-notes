# Workbox

> **In one line:** Workbox is Google's set of libraries for service workers that gives you precaching, routing and ready-made caching strategies with expiration, so you write a few lines of config instead of hand-written cache code.

## Key points
- **What it simplifies:** cache versioning, cleanup, revision tracking, strategies, expiry, update prompts and offline retry. Hand-written service workers get these wrong easily.
- **Precaching (`workbox-precaching`):** at build time a plugin generates a list of files with a revision hash (the "precache manifest"). At install Workbox downloads them, and on the next deploy it only re-downloads the files whose hash changed.
- **Runtime routing (`workbox-routing`):** `registerRoute(match, strategy)` sends requests that match a URL pattern to a strategy, for requests that happen while the app runs.
- **Strategies (`workbox-strategies`):** `CacheFirst`, `NetworkFirst`, `StaleWhileRevalidate`, `NetworkOnly`, `CacheOnly`.
- **Plugins:** `ExpirationPlugin` (max entries, max age), `CacheableResponsePlugin` (which statuses to cache), `BackgroundSyncPlugin` (retry failed POSTs later).

## Example
```js
// src/sw.js (injectManifest mode: you write the SW, the build injects the file list)
import { precacheAndRoute, cleanupOutdatedCaches, createHandlerBoundToURL } from 'workbox-precaching';
import { registerRoute, NavigationRoute } from 'workbox-routing';
import { CacheFirst, NetworkFirst, StaleWhileRevalidate, NetworkOnly } from 'workbox-strategies';
import { ExpirationPlugin } from 'workbox-expiration';
import { CacheableResponsePlugin } from 'workbox-cacheable-response';
import { BackgroundSyncPlugin } from 'workbox-background-sync';

// 1. Precache the app shell. The build replaces self.__WB_MANIFEST with
//    [{ url: '/_app/app.3f9a1c.js', revision: null }, { url: '/index.html', revision: 'a1b2' }, ...]
precacheAndRoute(self.__WB_MANIFEST);
cleanupOutdatedCaches(); // delete precaches made by older Workbox versions

// 2. SPA navigations -> cached index.html (works offline)
registerRoute(new NavigationRoute(createHandlerBoundToURL('/index.html'), {
  denylist: [/^\/api\//],
}));

// 3. Images and logos: cache first, keep 100 for 30 days
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [
      new CacheableResponsePlugin({ statuses: [0, 200] }),
      new ExpirationPlugin({ maxEntries: 100, maxAgeSeconds: 30 * 24 * 60 * 60, purgeOnQuotaError: true }),
    ],
  }),
);

// 4. Reference data: stale-while-revalidate
registerRoute(({ url }) => url.pathname.startsWith('/api/instruments'),
  new StaleWhileRevalidate({ cacheName: 'instruments',
    plugins: [new ExpirationPlugin({ maxAgeSeconds: 24 * 60 * 60 })] }));

// 5. Portfolio: network first, fall back to cache after 3 s
registerRoute(({ url }) => url.pathname.startsWith('/api/portfolio'),
  new NetworkFirst({ cacheName: 'portfolio', networkTimeoutSeconds: 3,
    plugins: [new ExpirationPlugin({ maxEntries: 20 })] }));

// 6. Live quotes: never cached
registerRoute(({ url }) => url.pathname.startsWith('/api/quotes'), new NetworkOnly());

// 7. Non-critical POSTs (analytics, watchlist edits) retried when back online
registerRoute(({ url }) => url.pathname === '/api/watchlist',
  new NetworkOnly({ plugins: [new BackgroundSyncPlugin('watchlist-queue', { maxRetentionTime: 24 * 60 })] }),
  'POST');

self.addEventListener('message', (e) => { if (e.data?.type === 'SKIP_WAITING') self.skipWaiting(); });
```

```js
// main.js - update prompt with workbox-window
import { Workbox } from 'workbox-window';
const wb = new Workbox('/sw.js');
wb.addEventListener('waiting', () => {
  showToast('New version available', { action: 'Refresh', onAction: () => {
    wb.addEventListener('controlling', () => location.reload());
    wb.messageSkipWaiting();
  } });
});
wb.register();
```

## When to use it
- Any production PWA. In a Vite + Svelte app, the community plugin `vite-plugin-pwa` uses Workbox under the hood (`generateSW` creates the whole worker from config, `injectManifest` lets you write your own like above).
- In SvelteKit you can use the built-in `$service-worker` module (it gives you `build`, `files` and `version`) for simple cases, or Workbox for routes, expiry and background sync.

## Likely questions
### What does Workbox simplify?
Without it I would write my own cache names, version bumps, cleanup in activate, every strategy, expiration with timestamps in IndexedDB, and retry queues. Workbox gives tested versions of all of these. The two big wins are precaching with per-file revisions, so a deploy downloads only changed files, and declarative runtime routes like "images: cache first, max 100 entries".

### How does precaching work?
At build time `workbox-build` or a bundler plugin scans the output and creates a list of `{ url, revision }`. Hashed files get `revision: null` because the URL is already unique. The list is injected into the service worker, so any file change also changes the worker bytes and triggers an update. At install Workbox fetches new or changed entries; at activate it deletes entries no longer in the list. `precacheAndRoute` also adds a route so those URLs are served cache first.

### What is the difference between precaching and runtime caching?
Precaching happens at install time for a known list of files, the app shell, so the app opens offline even on first repeat visit. Runtime caching happens as the user browses, for requests you cannot list at build time, like API responses and user images. Precache is versioned by the build; runtime caches need an expiration policy.

### How does cache expiration work?
`ExpirationPlugin` stores timestamps for each entry in IndexedDB. `maxEntries` deletes the least recently used entries above the limit. `maxAgeSeconds` ignores and deletes entries older than that. `purgeOnQuotaError: true` lets the browser wipe that cache if storage runs out. Expiry runs after a response is cached, so the cache stays bounded.

### Why `CacheableResponsePlugin({ statuses: [0, 200] })`?
Cross-origin requests without CORS give **opaque** responses with status 0. You cannot see if they are errors, and Chrome counts each one as several MB of quota. By default `CacheFirst` only caches status 200 (so a hidden error is not stuck in cache forever), while `NetworkFirst` and `StaleWhileRevalidate` also cache opaque ones. Adding the plugin makes the rule explicit, and I pair it with a small `maxEntries`.

## Common mistakes
- Precaching everything, including large images and every route chunk. First visit becomes heavy. Precache the shell, runtime-cache the rest.
- Using `NetworkFirst` without `networkTimeoutSeconds`, so slow networks feel broken.
- Forgetting `cleanupOutdatedCaches()` after upgrading Workbox major versions.
- Using `generateSW` with `skipWaiting: true` and `clientsClaim: true` without thinking about open tabs mid-order.

## Resources
- [Chrome for Developers: Workbox](https://developer.chrome.com/docs/workbox) - official docs home
- [workbox-precaching](https://developer.chrome.com/docs/workbox/modules/workbox-precaching) - how revisions and precache work
- [workbox-strategies](https://developer.chrome.com/docs/workbox/modules/workbox-strategies) - every built-in strategy
- [workbox-expiration](https://developer.chrome.com/docs/workbox/modules/workbox-expiration) - maxEntries and maxAgeSeconds
- [web.dev: Learn PWA - Workbox](https://web.dev/learn/pwa/workbox) - short intro inside the PWA course
- [SvelteKit: Service workers](https://svelte.dev/docs/kit/service-workers) - the built-in `$service-worker` module
