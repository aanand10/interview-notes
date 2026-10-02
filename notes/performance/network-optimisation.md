# Network optimisation

> **In one line:** Make fewer and smaller requests, compress everything text-based with Brotli or gzip, use HTTP/2 or HTTP/3, warm up connections early with `preconnect`, and prefetch the next route before the user clicks.

## Key points
- **Fewer requests:** combine API calls the page needs (or use one backend endpoint per screen), avoid request waterfalls (A waits for B waits for C), and remove unused third-party scripts.
- **Compression:** the server compresses text files (HTML, JS, CSS, JSON, SVG). Brotli (`Content-Encoding: br`) is usually smaller than gzip. Do not compress images that are already compressed. See [MDN: Compression](https://developer.mozilla.org/en-US/docs/Web/HTTP/Compression).
- **HTTP/2:** many requests share one connection at the same time (multiplexing), with header compression. HTTP/3 runs over QUIC (UDP) and handles lossy mobile networks better. See [MDN: HTTP/2](https://developer.mozilla.org/en-US/docs/Glossary/HTTP_2).
- **Resource hints:** `preconnect` does DNS + TCP + TLS early for an important other origin; `preload` fetches a critical file for this page early; `prefetch` fetches something for the next page at low priority.
- **Prefetch next route:** SvelteKit can preload a route's code and data when the user hovers or taps a link.

## Example
```html
<!-- Warm up the connection to the API and price stream origins -->
<link rel="preconnect" href="https://api.example.com" />
<link rel="dns-prefetch" href="https://stream.example.com" />

<!-- Fetch a critical file for THIS page early -->
<link rel="preload" href="/fonts/inter-latin-600.woff2" as="font" type="font/woff2" crossorigin />

<!-- Low-priority fetch for the page the user will probably visit next -->
<link rel="prefetch" href="/orders" />
```

SvelteKit prefetching with link options:

```svelte
<!-- Preload code + data when the user hovers or touches the link -->
<a href="/stocks/INFY" data-sveltekit-preload-data="hover">Infosys</a>

<!-- Only preload code (not data) as soon as the link is visible -->
<a href="/reports" data-sveltekit-preload-code="viewport">Reports</a>
```

Avoid a waterfall by running independent requests in parallel:

```js
// Slow: 3 round trips one after another
// const profile = await getJson('/api/profile');
// const holdings = await getJson('/api/holdings');
// const watchlist = await getJson('/api/watchlist');

// Fast: all three at once
const [profile, holdings, watchlist] = await Promise.all([
  fetch('/api/profile').then((r) => r.json()),
  fetch('/api/holdings').then((r) => r.json()),
  fetch('/api/watchlist').then((r) => r.json()),
]);
```

## When to use it
Users on Indian mobile networks often have high latency, so round trips matter more than raw bandwidth. Preconnect to the API and WebSocket origins on app start. Prefetch the stock detail page when the user hovers a watchlist row. Combine the dashboard's data into one or two calls. In SvelteKit, load data in `+page.js`/`+page.server.js` so it starts in parallel with the code, not after the component mounts. See [SvelteKit: Link options](https://svelte.dev/docs/kit/link-options).

## Likely questions
### How do you reduce the number of requests?
Bundle code sensibly (not hundreds of tiny modules), inline tiny critical CSS, use SVG sprites or icon components, and on the API side make a screen-level endpoint or use GraphQL so one screen does not need ten calls. Also cache responses so repeat visits need fewer requests, and remove third-party scripts that are not worth their cost.

### gzip vs Brotli?
Both are lossless compression for text. Brotli usually produces smaller files, especially for JS and CSS, and every modern browser supports it over HTTPS. The browser sends `Accept-Encoding: gzip, br` and the server picks one. For static assets, pre-compress at build time at the highest level; for dynamic responses use a faster level.

### What does HTTP/2 change for frontend developers?
With HTTP/1.1 browsers opened about 6 connections per host, so we used domain sharding and huge bundles to reduce requests. HTTP/2 multiplexes many requests over one connection, so many small files are cheaper and domain sharding actually hurts. Bundling is still useful for compression and fewer module waterfalls, but extreme concatenation is no longer needed.

### preconnect vs preload vs prefetch?
`preconnect` sets up the connection to another origin early but downloads nothing; use it for 1 to 3 critical origins. `preload` downloads a specific resource needed for the current page with high priority, like the LCP image or a font. `prefetch` downloads something likely needed for a future navigation at low priority. Overusing preload hurts because it steals bandwidth from more important files. See [web.dev: preconnect](https://web.dev/articles/preconnect-and-dns-prefetch).

### How does route prefetching work in SvelteKit?
SvelteKit watches links. With `data-sveltekit-preload-data="hover"` it starts loading the next page's JS and runs its `load` function when the pointer hovers or the user starts a tap. By the time the click finishes, the data is often already there, so navigation feels instant. The default project template puts this on `<body>`.

## Common mistakes
- Fetching data in `onMount` or `$effect` after the component renders, creating a code-then-data waterfall.
- Preloading too many things, which slows down the truly critical ones.
- Preconnecting to many origins; each open connection costs CPU and it expires if unused.
- Forgetting `crossorigin` on preconnect for origins fetched with CORS (fonts, fetch).

## Resources
- [MDN: HTTP compression](https://developer.mozilla.org/en-US/docs/Web/HTTP/Compression) - how compression is negotiated
- [web.dev: Establish network connections early](https://web.dev/articles/preconnect-and-dns-prefetch) - preconnect and dns-prefetch
- [web.dev: Prefetch resources](https://web.dev/articles/link-prefetch) - speeding up future navigations
- [SvelteKit: Link options](https://svelte.dev/docs/kit/link-options) - preload-data and preload-code
- [web.dev: Introduction to HTTP/2](https://web.dev/articles/performance-http2) - multiplexing and why it matters
