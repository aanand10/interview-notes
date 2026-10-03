# Service Worker vs Web Worker

> **In one line:** Both run JavaScript off the main thread and neither can touch the DOM, but a Web Worker is a helper thread owned by one page for heavy computation, while a Service Worker is an event-driven network proxy for the whole origin that intercepts requests, enables offline and receives push, and lives independently of any tab.

## Key points
- **Purpose:** a [Web Worker](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) does CPU work for a page (parse, sort, calculate). A [Service Worker](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) sits between the app and the network: it handles `fetch`, caching, offline, `push` and background `sync`.
- **Lifetime:** a Web Worker is created with `new Worker()` and lives as long as the page that made it (or until `terminate()`). A Service Worker is registered once, then the browser starts it when an event arrives and kills it when idle, even with no tab open. So never keep state in its global variables.
- **Network interception:** only the Service Worker can intercept requests, through `event.respondWith()` in the `fetch` event. A Web Worker can call `fetch`, but it cannot see or change the page's requests.
- **DOM access:** neither has it. No `document`, no `window`. Both talk to pages with `postMessage`.
- **Scope and count:** a Web Worker belongs to one page, and a page can create many. A Service Worker controls every page in its scope (for example the whole origin), and only one version is active per scope. Service Workers also need HTTPS (localhost is allowed).

## Example
```js
// Web Worker: page creates it, asks a question, gets an answer
const worker = new Worker(new URL('./rsi.worker.js', import.meta.url), { type: 'module' });
worker.postMessage({ closes: [101.2, 102.5, 101.9 /* ... */] });
worker.onmessage = (e) => console.log('RSI', e.data);
```

```js
// Service Worker: page only registers it; the browser runs it on events
// main.js
navigator.serviceWorker.register('/sw.js');

// sw.js
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);
  if (url.pathname.startsWith('/api/instruments')) {
    // network first, fall back to cache when offline
    event.respondWith(
      fetch(event.request)
        .then(async (res) => {
          const cache = await caches.open('api-v1');
          cache.put(event.request, res.clone());
          return res;
        })
        .catch(() => caches.match(event.request))
    );
  }
});
```

| | Web Worker | Service Worker |
|---|---|---|
| Main job | Heavy computation for a page | Network proxy, offline, push, sync |
| Created by | `new Worker(url)` | `navigator.serviceWorker.register(url)` |
| Lifetime | Tied to the page; ends with it | Event-driven; starts and stops on its own, outlives tabs |
| Intercepts requests | No | Yes, `fetch` event + `respondWith` |
| DOM access | No | No |
| Pages served | The one page that created it | All pages in its scope |
| HTTPS required | No | Yes (except localhost) |
| Storage it can use | IndexedDB, Cache API | IndexedDB, Cache API |
| Talks to page via | `postMessage` | `postMessage` (via `clients` / `controller`) |

## When to use it
- **Web Worker:** computing indicators for a candlestick chart, filtering 50,000 trades, parsing an uploaded statement.
- **Service Worker:** loading the trading app shell offline, caching the instrument list, showing a price alert push notification, retrying a queued watchlist change when back online.
- In a real app you often use both: the Service Worker for network and offline, a Web Worker for number crunching.

## Likely questions
### What is the difference between a Service Worker and a Web Worker?
Both are background JavaScript without DOM access. A Web Worker is a thread a page creates to offload heavy work, and it dies with the page. A Service Worker is installed for the origin and acts as a proxy between the app and the network; the browser wakes it for events like `fetch`, `push` and `sync`, and it can run when no tab is open. Only the Service Worker can intercept network requests.

### Can either of them access the DOM?
No, neither can. They have no `document` or `window`. They post a message to the page, and the page updates the DOM. A Service Worker can also find open tabs with `clients.matchAll()` and message or focus them.

### Why can't a Service Worker hold state in variables?
Because the browser stops it when it is idle, often after about 30 seconds, and starts a fresh one for the next event. Global variables are lost. Keep state in IndexedDB or the Cache API.

### Can I do heavy computation in a Service Worker?
You could, but it is the wrong tool. It is shared by all tabs, so long work there slows every `fetch` for the whole app, and the browser may kill it if an event runs too long. Use a Web Worker for computation.

### What about Shared Workers?
A [SharedWorker](https://developer.mozilla.org/en-US/docs/Web/API/SharedWorker) is shared between tabs of the same origin, for example to keep one WebSocket for live prices across several open tabs. It still cannot intercept network requests and it lives only while some tab uses it. Safari added support late, so check support before relying on it.

## Common mistakes
- Saying a Service Worker can update the DOM directly.
- Saying a Web Worker can cache or intercept the page's requests.
- Forgetting the Service Worker needs HTTPS and only controls pages after it activates (the first load is not controlled without `clients.claim()`).

## Resources
- [MDN: Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) - overview of worker types
- [MDN: Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) - concepts and lifecycle
- [web.dev: Workers overview](https://web.dev/articles/workers-overview) - when to use which worker
