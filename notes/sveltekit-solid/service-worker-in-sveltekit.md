# Service worker in SvelteKit

> **In one line:** If you add `src/service-worker.js` (or `.ts`), SvelteKit bundles and registers it for you, and the `$service-worker` module gives you the list of built files, static files and a `version` string so you can cache the app for offline use.

## Key points
- A [service worker](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) is a script that sits between your page and the network. It can cache files and answer requests even when offline.
- **Auto-registered.** SvelteKit registers it for you. Turn this off with `kit.serviceWorker.register: false` if you want to register it yourself ([service workers docs](https://svelte.dev/docs/kit/service-workers)).
- **`$service-worker` module** (only importable inside the service worker) ([reference](https://svelte.dev/docs/kit/$service-worker)):
  - `build`: URLs of files Vite generated (JS and CSS chunks).
  - `files`: URLs of files in your `static` folder.
  - `prerendered`: paths of prerendered pages.
  - `version`: a string that changes on each build. Use it in the cache name.
  - `base`: the app's base path.
- **Lifecycle:** `install` (pre-cache files), `activate` (delete old caches), `fetch` (decide: cache or network).
- It is only bundled for production builds. In dev it works only in browsers that support module service workers.

## Example
```ts
// src/service-worker.ts
/// <reference types="@sveltejs/kit" />
/// <reference no-default-lib="true"/>
/// <reference lib="esnext" />
/// <reference lib="webworker" />
import { build, files, version } from '$service-worker';

const sw = self as unknown as ServiceWorkerGlobalScope;
const CACHE = `cache-${version}`;
const ASSETS = [...build, ...files];

sw.addEventListener('install', (event) => {
  event.waitUntil(caches.open(CACHE).then((cache) => cache.addAll(ASSETS)));
});

sw.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(keys.filter((k) => k !== CACHE).map((k) => caches.delete(k)))
    )
  );
});

sw.addEventListener('fetch', (event) => {
  if (event.request.method !== 'GET') return;
  const url = new URL(event.request.url);

  // app shell files: cache first (they have hashed names, safe forever)
  if (ASSETS.includes(url.pathname)) {
    event.respondWith(caches.match(url.pathname).then((r) => r ?? fetch(event.request)));
    return;
  }
  // everything else (prices, orders, API): always network, never cache
});
```

## When to use it
- Make the app shell (JS, CSS, icons, fonts) load instantly on repeat visits and work on bad mobile networks.
- Show an "You are offline" page instead of the browser's error page.
- In a trading app, do **not** cache live prices, balances or order responses. Stale financial data is worse than an error.

## Likely questions
### How do you add a service worker in SvelteKit?
Create `src/service-worker.ts`. SvelteKit bundles it and registers it. Inside it I import `build`, `files` and `version` from `$service-worker`, pre-cache `build` and `files` on `install`, delete old caches on `activate`, and handle `fetch`.

### What is `version` for?
It changes with each deploy. I name the cache `cache-${version}`, so a new deploy creates a new cache and `activate` deletes the old one. That avoids users stuck on an old app.

### Difference between `build` and `files`?
`build` is what Vite produced from your code, with hashed file names. `files` is whatever you put in `static/`, like `favicon.png` and `manifest.json`.

### Which caching strategy would you use?
Cache first for hashed static assets, network first (with cache fallback) for pages, and network only for live or private data like quotes and orders.

## Common mistakes
- Caching API responses with user data, which can then show to the next user on a shared device.
- Not cleaning old caches, so storage keeps growing.
- Forgetting a service worker only works over HTTPS (or `localhost`).

## Resources
- [SvelteKit: Service workers](https://svelte.dev/docs/kit/service-workers) - setup and full example
- [SvelteKit: $service-worker](https://svelte.dev/docs/kit/$service-worker) - `build`, `files`, `prerendered`, `version`
- [MDN: Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) - lifecycle and events
- [web.dev: Service workers](https://web.dev/learn/pwa/service-workers) - caching strategies explained
