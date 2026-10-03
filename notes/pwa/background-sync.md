# Background sync

> **In one line:** Background Sync lets a service worker retry a failed request later, when the connection is back, even if the user has closed the tab: the page saves the request in IndexedDB and registers a sync tag, and the browser fires a `sync` event in the service worker once it is online.

## Key points
- **The idea:** "do this when online". The user acts offline, the app queues the work, and the browser replays it at a good time.
- **Flow:** page stores the payload in IndexedDB, calls `registration.sync.register('tag')`. When the device has connectivity, the browser starts the service worker and fires `sync` with that tag. You replay the queue inside `event.waitUntil(...)`.
- **Retries:** if the promise in `waitUntil` rejects, the browser retries later with backoff, a limited number of times. `event.lastChance` is `true` on the final try.
- **Support:** the [Background Synchronization API](https://developer.mozilla.org/en-US/docs/Web/API/Background_Synchronization_API) is Chromium-only (Chrome, Edge, Android). Firefox and Safari do not support it, so always have a fallback: replay the queue on app start and on the `online` event.
- **Periodic Background Sync** is a different API (refresh content on a schedule), also Chromium-only and only for installed apps.

## Example
```js
// main.js - user adds a stock to the watchlist while offline
async function addToWatchlist(symbol) {
  const item = { id: crypto.randomUUID(), symbol, at: Date.now() };  // id makes replay safe
  await outbox.add(item);                                            // IndexedDB queue (idb)

  const reg = await navigator.serviceWorker.ready;
  if ('sync' in reg) {
    await reg.sync.register('sync-watchlist');   // browser fires `sync` when online
  } else {
    window.addEventListener('online', flushOutbox, { once: true });  // fallback
  }
}
```

```js
// sw.js
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-watchlist') event.waitUntil(flushOutbox());
});

async function flushOutbox() {
  for (const item of await outbox.getAll()) {
    const res = await fetch('/api/watchlist', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Idempotency-Key': item.id },
      body: JSON.stringify(item),
    });
    if (!res.ok) throw new Error('retry later');  // reject = browser retries
    await outbox.delete(item.id);
  }
}
```

Workbox wraps this whole pattern in [`BackgroundSyncPlugin`](https://developer.chrome.com/docs/workbox/modules/workbox-background-sync), which queues failed requests and replays them for you, with a fallback for browsers without the API.

## When to use it
- Safe, non-urgent writes: watchlist edits, price alert setup, saved notes, analytics.
- Not for placing a market order. Prices move, so an order sent minutes later at a different price is dangerous. For orders, block when offline and tell the user clearly.

## Likely questions
### How do you retry failed requests when the user is back online?
Save the request data in IndexedDB, then register a sync tag. When the browser sees connectivity, it wakes the service worker and fires `sync`. I read the queue, send each item, delete it on success, and throw if it fails so the browser retries later. I give each item a unique id and send it as an idempotency key, so a retry after a timeout does not create duplicates on the server.

### What if the browser does not support Background Sync?
Feature-detect `'sync' in registration`. Without it, I flush the queue when the app starts and when the window fires `online`. It only runs while a tab is open, but the data is not lost. Workbox's plugin does this automatically.

### How is it different from just listening to `online`?
The `online` event only fires in an open page. Background Sync runs in the service worker, so it can finish even after the user closes the tab, and the browser handles retry timing.

## Common mistakes
- Queuing in memory instead of IndexedDB, so a closed tab loses the data.
- No idempotency, so retries create duplicate records.
- Using it for time-sensitive actions like trades.
- Relying on it in Safari or Firefox without a fallback.

## Resources
- [MDN: Background Synchronization API](https://developer.mozilla.org/en-US/docs/Web/API/Background_Synchronization_API) - API and support
- [Chrome for Developers: workbox-background-sync](https://developer.chrome.com/docs/workbox/modules/workbox-background-sync) - ready-made queue and replay
- [web.dev: Learn PWA - Service workers](https://web.dev/learn/pwa/service-workers) - where `sync` fits among worker events
