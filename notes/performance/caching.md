# Caching

> **In one line:** Caching means not fetching the same thing twice: HTTP cache headers and a CDN for static files, a service worker for offline and instant repeat loads, and an in-memory cache with a short expiry for API data.

## Key points
- **HTTP cache headers:** `Cache-Control` decides who can store a response and for how long. Hashed static assets (`app.3f9a1c.js`) get `public, max-age=31536000, immutable`. HTML gets `no-cache` (store it, but check with the server before using). See [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching).
- **Validation:** `ETag` / `Last-Modified` let the browser ask "has this changed?" and get a tiny `304 Not Modified` instead of the full file.
- **CDN:** copies of static files on servers close to users (edge locations), so a user in Pune does not wait on a server in Virginia. `s-maxage` sets the CDN's cache time separately.
- **Service worker caching:** a script that intercepts requests and can serve from the Cache Storage API with strategies like cache-first, network-first and stale-while-revalidate. Workbox makes this easy.
- **In-memory API caching:** keep recent API responses in a `Map` with a time-to-live (TTL) and dedupe in-flight requests. Never cache live prices or order status for long.

## Example
Typical headers:

```bash
# Hashed JS/CSS from the build: cache "forever", the file name changes on deploy
Cache-Control: public, max-age=31536000, immutable

# HTML: always revalidate so users get the new deploy
Cache-Control: no-cache
ETag: "v42"

# Personal API data (portfolio): never store in shared caches
Cache-Control: private, no-store

# Semi-static API (list of instruments): CDN for 5 min, serve stale while refreshing
Cache-Control: public, max-age=60, s-maxage=300, stale-while-revalidate=600
```

In-memory cache with TTL and request de-duplication (verified with node):

```js
const cache = new Map();
const inflight = new Map();

async function cachedFetch(key, loader, ttlMs = 30_000) {
  const hit = cache.get(key);
  if (hit && Date.now() - hit.time < ttlMs) return hit.data; // fresh hit
  if (inflight.has(key)) return inflight.get(key);           // same request already running
  const p = loader()
    .then((data) => { cache.set(key, { data, time: Date.now() }); return data; })
    .finally(() => inflight.delete(key));
  inflight.set(key, p);
  return p;
}

let calls = 0;
const loader = async () => { calls++; return { symbol: 'INFY', pe: 24 }; };
await Promise.all([cachedFetch('INFY', loader), cachedFetch('INFY', loader)]);
await cachedFetch('INFY', loader);
console.log('network calls:', calls);
await cachedFetch('INFY', loader, 0);
console.log('after expiry:', calls);
```

```bash
network calls: 1   # two parallel calls shared one request, third was a cache hit
after expiry: 2    # TTL of 0 means stale, so it fetched again
```

## When to use it
Static assets and fonts: long HTTP cache plus CDN. Company fundamentals, instrument lists and news: short TTL in memory or `stale-while-revalidate`. App shell: service worker so the app opens instantly on repeat visits and shows something offline. Live prices, balances and order status: no long caching; show them fresh, or show cached values clearly marked as stale.

## Likely questions
### Explain the main Cache-Control directives.
`max-age=N` means fresh for N seconds. `no-cache` means you may store it but must revalidate before each use. `no-store` means do not store at all, used for sensitive data. `private` means only the browser may cache, not CDNs; `public` allows shared caches. `immutable` tells the browser not to revalidate during `max-age`. `s-maxage` overrides `max-age` for shared caches like CDNs.

### How do you cache JS forever but still ship updates?
Content hashing: the build puts a hash of the file content in the name, like `app.3f9a1c.js`. The HTML, which is not cached long, points to the new file names after each deploy. So old files can be cached for a year safely. SvelteKit and Vite do this by default.

### What service worker caching strategies do you know?
Cache-first: use the cache, go to network only if missing; good for fonts and hashed assets. Network-first: try the network, fall back to cache when offline; good for HTML and API data that should be fresh. Stale-while-revalidate: return the cached version immediately and update the cache in the background; good for avatars and semi-static data. See [Workbox caching strategies](https://developer.chrome.com/docs/workbox/caching-strategies-overview).

### Service worker cache vs HTTP cache?
The HTTP cache is controlled by server headers and the browser decides. The service worker cache is controlled by your own code, so you can pre-cache the app shell, work offline and pick strategies per route. The service worker's `fetch` still goes through the HTTP cache underneath. See [web.dev: SW caching and HTTP caching](https://web.dev/articles/service-worker-caching-and-http-caching).

### What are the risks of caching in a trading app?
Showing stale prices or balances as if they were live, and leaking personal data through a shared cache. So user-specific endpoints use `private` or `no-store`, prices come from the live stream, and anything cached shows its age. Also clear in-memory and service worker caches of user data on logout.

## Common mistakes
- Long `max-age` on HTML, so users get stuck on an old version.
- Confusing `no-cache` (revalidate) with `no-store` (do not store).
- Caching authenticated API responses at the CDN.
- A service worker that serves an old app forever because it never updates.

## Resources
- [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching) - full guide to cache behaviour
- [web.dev: Prevent unnecessary network requests with the HTTP Cache](https://web.dev/articles/http-cache) - practical header choices
- [Workbox: Caching strategies overview](https://developer.chrome.com/docs/workbox/caching-strategies-overview) - service worker strategies
- [SvelteKit: Service workers](https://svelte.dev/docs/kit/service-workers) - built-in service worker support
- [web.dev: stale-while-revalidate](https://web.dev/articles/stale-while-revalidate) - fast and fresh at the same time
