# Service worker lifecycle

> **In one line:** A service worker is registered by the page, then goes through install (cache files), waiting, and activate (clean up), and after that it controls pages and handles their `fetch` events; a new version waits until no tab uses the old one, unless you call `skipWaiting`.

## Key points
- **Register:** the page calls `navigator.serviceWorker.register('/sw.js')`. The browser downloads it and starts the lifecycle. The file location sets the default **scope** (which URLs it controls).
- **Install:** runs once per version. You precache the app shell here inside `event.waitUntil(...)`. If any promise rejects, install fails and this version is thrown away.
- **Waiting:** if an older service worker still controls open tabs, the new one stays in **waiting**. Only one version controls a page at a time, so old pages are never mixed with new files.
- **Activate:** runs when the old version is gone. Delete old caches here. After activate, the worker gets `fetch`, `push`, `sync` and `message` events.
- **Not controlled on first load:** the page that registered the worker is not controlled until reload, unless you call `clients.claim()` in activate.

## Example
```js
// sw.js
const VERSION = 'v3';
const SHELL_CACHE = `shell-${VERSION}`;
const SHELL = ['/', '/app.css', '/app.js', '/offline.html'];

self.addEventListener('install', (event) => {
  // Keep the worker in "installing" until the shell is cached
  event.waitUntil(caches.open(SHELL_CACHE).then((c) => c.addAll(SHELL)));
  // Do NOT call skipWaiting here if you want a "new version" prompt
});

self.addEventListener('activate', (event) => {
  event.waitUntil((async () => {
    const keys = await caches.keys();
    await Promise.all(keys.filter((k) => k !== SHELL_CACHE).map((k) => caches.delete(k)));
    await self.clients.claim(); // take control of already-open tabs now
  })());
});

self.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(fetch(event.request).catch(() => caches.match('/offline.html')));
  }
});

// Page asks the waiting worker to activate
self.addEventListener('message', (event) => {
  if (event.data?.type === 'SKIP_WAITING') self.skipWaiting();
});
```

```js
// main.js - show "New version available" and reload on accept
const reg = await navigator.serviceWorker.register('/sw.js');

function promptUser(worker) {
  showToast('New version available', {
    action: 'Refresh',
    onAction: () => worker.postMessage({ type: 'SKIP_WAITING' }),
  });
}

if (reg.waiting) promptUser(reg.waiting);           // update already waiting
reg.addEventListener('updatefound', () => {
  const nw = reg.installing;
  nw.addEventListener('statechange', () => {
    // installed + an existing controller = this is an update, not first install
    if (nw.state === 'installed' && navigator.serviceWorker.controller) promptUser(nw);
  });
});

let reloaded = false;
navigator.serviceWorker.addEventListener('controllerchange', () => {
  if (reloaded) return;
  reloaded = true;
  location.reload(); // new worker now controls the page, load fresh files
});
```

## When to use it
- A trading PWA deploys several times a day. You want users to get fixes fast, but you must not swap JS under a user who is halfway through placing an order. So you show a "New version, refresh" toast and let the user choose, or auto-reload only on safe screens.

## Likely questions
### Walk me through the lifecycle.
The page registers the script. The browser downloads it and fires `install`, where I precache the app shell. Then it goes to `waiting` if an old version controls any tab, or straight to `activate` if not. In `activate` I delete old caches. After that the worker is "activated" and handles `fetch` for pages in its scope. The browser stops the worker when idle and wakes it up for the next event, so I never keep state in global variables.

### Why does a new version wait?
Because the old version may still be serving open tabs, and those tabs were built against the old cached files. If the new worker took over and deleted the old cache, an old tab could request a file that no longer exists. So the browser keeps the new version waiting until all tabs using the old one are closed. A simple reload is not enough when only one tab is open, because the old page stays alive during navigation; the user must close the tab or you use `skipWaiting`.

### What do `skipWaiting` and `clients.claim` do?
[`self.skipWaiting()`](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerGlobalScope/skipWaiting) tells the waiting worker to activate right away, even if old tabs are open. [`clients.claim()`](https://developer.mozilla.org/en-US/docs/Web/API/Clients/claim) makes an active worker take control of open pages that are not controlled yet, for example the very first page load. Together they mean "new version takes over immediately". The risk is that an open page now runs old JS against a new worker and new caches, which can break lazy-loaded chunks.

### How do you prompt users to update?
Do not call `skipWaiting` automatically. Detect a waiting worker (`reg.waiting` or `updatefound` then state `installed` while a controller exists). Show a toast "New version available, Refresh". When clicked, `postMessage({type: 'SKIP_WAITING'})` to the waiting worker; it calls `skipWaiting()`. Listen for `controllerchange` on the page and reload once. [Workbox Window](https://developer.chrome.com/docs/workbox/modules/workbox-window) wraps this in a `waiting` event and `messageSkipWaiting()`.

### When does the browser check for updates?
On navigation to a page in scope, on functional events like `push` or `sync` if no check happened in the last 24 hours, and when you call [`registration.update()`](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration/update). It compares the script byte by byte; any change makes it a new version. By default the `sw.js` file itself bypasses the HTTP cache for update checks (`updateViaCache: 'imports'`), but still serve it with `Cache-Control: no-cache` to be safe.

### What is scope?
Scope is the URL path a worker controls. `/sw.js` controls the whole origin; `/app/sw.js` controls only `/app/`. You can narrow it with `register(url, { scope })`. Widening it beyond the script folder needs the `Service-Worker-Allowed` header.

## Common mistakes
- Calling `skipWaiting()` in install by default, then getting `ChunkLoadError` in old tabs after deploy.
- Forgetting `event.waitUntil`, so the browser thinks install is done before caching finishes.
- Storing state in global variables in the worker. It is killed when idle and loses them.
- Reloading on `controllerchange` without a guard, causing a reload loop in DevTools "Update on reload" mode.
- Caching `sw.js` with a long `max-age` on the CDN, so updates are delayed.

## Resources
- [web.dev: The service worker lifecycle](https://web.dev/articles/service-worker-lifecycle) - the classic deep dive by Jake Archibald
- [web.dev: Learn PWA - Update](https://web.dev/learn/pwa/update) - update flow and prompting
- [Chrome for Developers: Handling service worker updates](https://developer.chrome.com/docs/workbox/handling-service-worker-updates) - update prompt with workbox-window
- [MDN: Using Service Workers](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers) - API walkthrough
- [MDN: ExtendableEvent.waitUntil()](https://developer.mozilla.org/en-US/docs/Web/API/ExtendableEvent/waitUntil) - how install and activate wait for promises
