# Cache API

> **In one line:** The Cache API is a storage area where my code saves `Request` to `Response` pairs and reads them back later; it is mostly used inside a service worker to serve files offline, and unlike the HTTP cache, I decide exactly what goes in, when it is used and when it is deleted.

## Key points
- **Request/response pairs:** a [`Cache`](https://developer.mozilla.org/en-US/docs/Web/API/Cache) maps a request (URL plus method) to a full response (status, headers, body). You get named caches from the global [`caches`](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage) object.
- **Main methods:** `caches.open(name)`, `cache.addAll(urls)` (fetch and store), `cache.put(request, response)` (store something you already have), `cache.match(request)` / `caches.match(request)` (look up), `cache.delete()`, `caches.keys()` / `caches.delete(name)`.
- **Used inside service workers:** the `fetch` handler checks the cache and returns a response with `event.respondWith`. But `caches` is also available on the page and in Web Workers.
- **Vs HTTP cache:** the HTTP cache is automatic and controlled by server headers (`Cache-Control`, `ETag`), and the browser may drop entries any time. The Cache API is fully manual: entries never expire on their own, headers like `max-age` are ignored, and they stay until your code deletes them (or the browser evicts the whole origin's storage).
- **GET only:** `put` only accepts GET requests. A response body can be read once, so use `response.clone()` when you both cache and return it.

## Example
```js
// sw.js - stale-while-revalidate for the instruments list
const RUNTIME = 'runtime-v1';

self.addEventListener('fetch', (event) => {
  const { request } = event;
  if (request.method !== 'GET' || !request.url.includes('/api/instruments')) return;

  event.respondWith((async () => {
    const cache = await caches.open(RUNTIME);
    const cached = await cache.match(request);         // look up the saved pair

    const network = fetch(request).then((res) => {
      if (res.ok) cache.put(request, res.clone());     // clone: body is read once
      return res;
    });

    // Return cached copy instantly if we have one; refresh in the background
    if (cached) {
      event.waitUntil(network.catch(() => {}));
      return cached;
    }
    return network;
  })());
});
```

```js
// On the page: check what is cached (handy in DevTools console too)
const cache = await caches.open('runtime-v1');
const keys = await cache.keys();
console.log(keys.map((r) => r.url));
```

## When to use it
- Precaching the app shell (HTML, JS, CSS, fonts, logo) so the trading app opens offline.
- Runtime caching of GET API responses that are fine to be a bit old, like the instruments list or company profiles.
- Not for live prices or order status: those should go to the network, and stale data there is dangerous. Not for structured app data you query: use IndexedDB.

## Likely questions
### What is the Cache API?
It is a key-value store where the key is a `Request` and the value is a `Response`. I open a named cache, put responses in, and later `match` a request to get the response back. It is the building block for offline support: the service worker's `fetch` handler serves from it.

### How is it different from the HTTP cache?
The HTTP cache is managed by the browser using response headers; I can only influence it from the server, and it is shared by everything that loads the URL. The Cache API is controlled by my JavaScript: I choose what to store and when to serve it, it ignores `Cache-Control`, and nothing expires unless I delete it. That is also the downside: I must version caches and clean up, or old files pile up. Also, a request from a service worker can still hit the HTTP cache first before reaching the network, so both layers can be involved.

### Why do you call `response.clone()`?
A response body is a stream that can be read only once. If I put it in the cache and also return it to the page, one of them would get an empty body. Cloning gives two independent copies.

### Can you cache POST requests?
No, `cache.put` throws for non-GET requests. For offline POSTs (like placing a watchlist change), store the payload in IndexedDB and replay it later, for example with Background Sync.

### Where does the Cache API data live and can it be lost?
It counts toward the origin's storage quota along with IndexedDB. Under storage pressure the browser can evict the whole origin's data unless persistent storage was granted, so treat it as a cache, not as the source of truth.

## Common mistakes
- Caching error responses (404, 500) because of not checking `res.ok`. Note opaque cross-origin responses (`no-cors`) have status 0 and you cannot tell if they failed.
- Forgetting `clone()`.
- Never deleting old caches.
- Caching authenticated, user-specific API data and serving it after logout. Clear caches on logout.

## Resources
- [MDN: Cache](https://developer.mozilla.org/en-US/docs/Web/API/Cache) - all methods with examples
- [MDN: CacheStorage](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage) - the global `caches` object
- [web.dev: The Cache API: A quick guide](https://web.dev/articles/cache-api-quick-guide) - practical usage
- [web.dev: Learn PWA - Caching](https://web.dev/learn/pwa/caching) - how it fits in a PWA
