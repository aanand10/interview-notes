# Browser storage overview

> **In one line:** Cookies are tiny and sent to the server with every request; localStorage and sessionStorage are small, synchronous key-value stores that stay in the browser; IndexedDB and the Cache API are large, asynchronous stores for real data and offline files.

## Key points
- **Cookies:** about 4 KB each. Sent automatically with matching HTTP requests. Can be `HttpOnly` (JS cannot read them), `Secure` (HTTPS only), `SameSite` (controls cross-site sending). Best for session tokens.
- **localStorage:** about 5 MB per origin. Strings only. **Synchronous** (blocks the main thread). Lives until cleared. Shared by all tabs of the same origin.
- **sessionStorage:** same API and size, but scoped to **one tab** and cleared when that tab closes.
- **IndexedDB:** a real database in the browser. **Asynchronous**, stores objects, blobs and files, has indexes and transactions. Size is a share of free disk (often hundreds of MB or more).
- **Cache API:** stores `Request`/`Response` pairs. Asynchronous. Used by service workers for offline pages and assets. Same large quota as IndexedDB.

| | Size (rough) | Sync / async | Sent to server | Lifetime | Readable in a service worker |
|---|---|---|---|---|---|
| Cookies | ~4 KB per cookie | sync (`document.cookie`) | Yes, every matching request | Until `Expires`/`Max-Age`, or session | No (only via request headers) |
| localStorage | ~5 MB per origin | sync | No | Until cleared | No |
| sessionStorage | ~5 MB per origin | sync | No | Until the tab closes | No |
| IndexedDB | Large, quota-based | async | No | Until cleared or evicted | Yes |
| Cache API | Large, quota-based | async | No | Until cleared or evicted | Yes |

## Example
```js
// Cookie (set by JS here; a session cookie should be set by the server with HttpOnly)
document.cookie = 'theme=dark; Max-Age=31536000; Path=/; Secure; SameSite=Lax';

// localStorage: strings only, so serialize objects
localStorage.setItem('watchlist', JSON.stringify(['AAPL', 'TSLA']));
const list = JSON.parse(localStorage.getItem('watchlist') ?? '[]');

// sessionStorage: survives reload, not a new tab
sessionStorage.setItem('orderDraft', JSON.stringify({ symbol: 'AAPL', qty: 10 }));

// Cache API: store a response for offline use
const cache = await caches.open('static-v1');
await cache.add('/offline.html');
const res = await cache.match('/offline.html');
```

IndexedDB (raw API is event-based; libraries like `idb` wrap it in promises):

```js
const req = indexedDB.open('trading', 1);
req.onupgradeneeded = () => {
  req.result.createObjectStore('candles', { keyPath: 'id' });
};
req.onsuccess = () => {
  const db = req.result;
  const tx = db.transaction('candles', 'readwrite');
  tx.objectStore('candles').put({ id: 'AAPL-1d', data: [/* ...thousands of candles */] });
};
```

Server-set session cookie (the safe way to keep a login):

```bash
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/
```

## When to use it
- **Cookie:** auth session (HttpOnly, Secure, SameSite). Never store tokens in localStorage in a trading app, since any XSS can read it.
- **localStorage:** small UI preferences: theme, last selected tab, column order of the watchlist.
- **sessionStorage:** per-tab state like a half-filled order form or wizard step.
- **IndexedDB:** large or structured data: cached historical candles, offline transaction list.
- **Cache API:** app shell, JS/CSS, offline fallback page in a PWA.

## Likely questions
### Compare cookies, localStorage, sessionStorage, IndexedDB and Cache API.
Cookies are small and the only one sent to the server automatically, so they suit auth. localStorage and sessionStorage are simple sync string stores of about 5 MB; local lasts forever, session lasts for one tab. IndexedDB is an async database for big structured data. Cache API is an async store of HTTP responses, mainly for service workers. Only IndexedDB and Cache API are usable inside a service worker.

### Why is localStorage being synchronous a problem?
Every read or write blocks the main thread while the browser reads from disk. Small values are fine, but storing a big JSON blob or reading it on every tick can cause jank. For large data use IndexedDB, which is async.

### Where should I store an auth token?
Best is an `HttpOnly; Secure; SameSite` cookie set by the server: JS cannot read it, so XSS cannot steal it. Protect against CSRF with `SameSite` and a CSRF token. localStorage is readable by any script on the page, so a single XSS leaks the token.

### Is storage shared across tabs?
localStorage, IndexedDB, Cache API and cookies are shared by all tabs on the same origin. sessionStorage is per tab. The `storage` event fires in other tabs when localStorage changes, which you can use to sync logout across tabs.

### Can the browser delete my data?
Yes. Under storage pressure the browser can evict "best-effort" data for an origin. Call `navigator.storage.persist()` to ask for persistent storage, and `navigator.storage.estimate()` to see usage and quota. Safari can also clear script-written storage for sites the user has not visited in a while.

## Common mistakes
- Storing objects in localStorage without `JSON.stringify`; you get `"[object Object]"`.
- Not wrapping storage in try/catch: it can throw in private mode or when full (`QuotaExceededError`).
- Putting large data in cookies, which bloats every request.

## Resources
- [MDN: Web Storage API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API) - localStorage and sessionStorage
- [MDN: Using HTTP cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) - HttpOnly, Secure, SameSite
- [MDN: IndexedDB API](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) - async database basics
- [web.dev: Storage for the web](https://web.dev/articles/storage-for-the-web) - quotas, eviction, which store to choose
