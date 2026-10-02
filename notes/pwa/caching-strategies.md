# Caching strategies

> **In one line:** A caching strategy is the rule a service worker follows for a request (cache first, network first, stale-while-revalidate, cache only or network only), and you pick it per type of data based on how fresh it must be.

## Key points
- **Cache first:** answer from cache; go to network only if missing. Best for files that never change at the same URL (hashed JS/CSS, fonts, logos).
- **Network first:** try network; if it fails (or times out), use cache. Best for HTML pages and data that should be fresh but can fall back when offline (portfolio, order history).
- **Stale-while-revalidate (SWR):** answer from cache right away, and fetch an update in the background for next time. Best for "fresh enough" data (avatars, news, company profiles, watchlist names).
- **Cache only:** only the cache, never network. For precached app shell files.
- **Network only:** always network, never cached. For **live prices, order placement, balances, auth** and any `POST`.

## Example
```js
// sw.js - hand-written strategies (Workbox gives you these ready-made)
const RUNTIME = 'runtime-v1';

async function cacheFirst(req) {
  const cached = await caches.match(req);
  if (cached) return cached;
  const res = await fetch(req);
  if (res.ok) (await caches.open(RUNTIME)).put(req, res.clone()); // a body can be read once, so clone
  return res;
}

async function networkFirst(req, timeoutMs = 3000) {
  const cache = await caches.open(RUNTIME);
  try {
    const res = await Promise.race([
      fetch(req),
      new Promise((_, rej) => setTimeout(() => rej(new Error('timeout')), timeoutMs)),
    ]);
    if (res.ok) cache.put(req, res.clone());
    return res;
  } catch {
    return (await cache.match(req)) ?? Response.error();
  }
}

async function staleWhileRevalidate(event) {
  const cache = await caches.open(RUNTIME);
  const cached = await cache.match(event.request);
  const update = fetch(event.request).then((res) => {
    if (res.ok) cache.put(event.request, res.clone());
    return res;
  });
  event.waitUntil(update.catch(() => {})); // keep the worker alive for the refresh
  return cached ?? update;
}

self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);
  if (request.method !== 'GET') return;                       // network only (browser default)
  if (url.pathname.startsWith('/api/quotes')) return;         // live prices: network only
  if (url.pathname.startsWith('/api/orders')) return;         // order status: network only
  if (request.mode === 'navigate') return event.respondWith(networkFirst(request));
  if (url.pathname.startsWith('/_app/immutable/')) return event.respondWith(cacheFirst(request));
  if (url.pathname.startsWith('/api/instruments')) return event.respondWith(staleWhileRevalidate(event));
});
```

How SWR behaves, simulated in Node with a Map as the cache (verified output):

```js
const cache = new Map();
let version = 1;
const network = async (url) => { await new Promise(r => setTimeout(r, 10)); return `${url} v${version++}`; };
async function swr(url) {
  const cached = cache.get(url);
  const update = network(url).then((res) => { cache.set(url, res); return res; });
  return cached ?? update;
}
console.log(await swr('/api/watchlist'));
console.log(await swr('/api/watchlist'));
await new Promise(r => setTimeout(r, 20));
console.log(await swr('/api/watchlist'));
```

```text
/api/watchlist v1   <- nothing cached, so we wait for the network
/api/watchlist v1   <- served from cache instantly; v2 is fetched in the background
/api/watchlist v2   <- the background refresh is now in the cache
```

## When to use it
| Request | Strategy | Why |
| --- | --- | --- |
| Hashed JS/CSS, fonts, icons | Cache first (or precache) | URL changes when content changes, so cache never goes stale |
| HTML navigation | Network first + offline fallback | Get the latest deploy, still open offline |
| Instrument list, logos, news | Stale-while-revalidate | Instant, slightly old is fine |
| Portfolio snapshot, order history | Network first with timeout | Fresh when online, labelled "last updated" when offline |
| Live prices, quotes, order book | Network only (and WebSocket, which SW cannot intercept) | Stale price can cause a wrong trade |
| Place order, login, payments | Network only | Never cache POST or sensitive data |

## Likely questions
### Explain the five strategies.
Cache first reads the cache and only falls back to network. Network first tries the network and falls back to the cache. Stale-while-revalidate returns the cached copy at once and refreshes the cache in the background. Cache only never touches the network, and network only never touches the cache. The choice is a trade-off between speed and freshness.

### Which one for static assets?
Cache first, or precache them at install. Modern build tools put a content hash in the file name, like `app.3f9a1c.js`, so the same URL always has the same content. That makes cache first safe forever.

### Which one for API data?
It depends on how fresh it must be. Reference data like the list of instruments or a company profile can use stale-while-revalidate. User data like holdings can use network first with a short timeout so it still opens offline, and the UI shows "last updated at 10:42". Anything about money that is acted on right now is network only.

### Which one for live prices?
Network only, and usually not even through the service worker, because live prices come over a WebSocket or SSE stream. A service worker cannot intercept WebSocket traffic. If the network is down, I show the last price clearly marked as stale (greyed out with a timestamp) and disable the Buy/Sell button, rather than serving it from cache as if it were live.

### What is the risk of stale-while-revalidate?
The user always sees the previous version first, so after a change they see old data once. It also costs a network request every time. If the cached data is wrong for business reasons, like an old price, SWR is the wrong choice.

### Why do you need `res.clone()`?
A Response body is a stream and can be read only once. If I put the same object in the cache and also return it to the page, one of them would get an empty body. Cloning gives two independent copies.

## Common mistakes
- Using cache first for HTML. Users get stuck on an old deploy.
- Caching error responses (404, 500) or opaque responses without checking `res.ok`.
- Caching personalised API responses, then a different user logs in on the same device and sees them.
- Network first with no timeout. On a slow 2G "lie-fi" connection, the user waits 30 seconds before the cache is used.
- Unlimited runtime caches. Add expiration (max entries, max age).

## Resources
- [web.dev: The Offline Cookbook](https://web.dev/articles/offline-cookbook) - every strategy with code
- [Chrome for Developers: Strategies for service worker caching](https://developer.chrome.com/docs/workbox/caching-strategies-overview) - clear diagrams of each strategy
- [web.dev: Learn PWA - Serving](https://web.dev/learn/pwa/serving) - strategies inside the PWA course
- [MDN: PWA caching guide](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Caching) - choosing a strategy
- [MDN: FetchEvent.respondWith()](https://developer.mozilla.org/en-US/docs/Web/API/FetchEvent/respondWith) - how the worker answers a request
