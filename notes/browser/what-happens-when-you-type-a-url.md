# What happens when you type a URL

> **In one line:** The browser turns the name into an IP address with DNS, opens a secure connection with TCP and TLS, sends an HTTP request, then parses the HTML, CSS and JS it gets back and turns them into pixels through the rendering pipeline.

## Key points
- **Find the server (DNS):** the domain name (`app.example.com`) is turned into an IP address. Caches are checked first: browser, OS, router, then a recursive DNS resolver that asks the root, `.com` and finally the domain's own name server.
- **Connect (TCP + TLS):** a TCP handshake (SYN, SYN-ACK, ACK) opens a reliable connection. Then a [TLS](https://developer.mozilla.org/en-US/docs/Glossary/TLS) handshake checks the server's certificate and agrees on encryption keys. HTTP/3 replaces TCP with QUIC (over UDP) and merges these steps.
- **Ask (HTTP):** the browser sends `GET /` with headers (cookies, `Accept`, `User-Agent`). The server replies with a status (200, 301, 304...), headers (`Cache-Control`, `Content-Type`, `Set-Cookie`) and the HTML body.
- **Build (parse):** HTML becomes the [DOM](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model), CSS becomes the CSSOM. Together they make the render tree.
- **Draw (render):** layout (sizes and positions), paint (pixels into layers), composite (layers put together on the GPU). Scripts run along the way and can pause HTML parsing.

## Example
The whole flow, step by step, for `https://trade.example.com/watchlist`:

```bash
# 0. Browser checks: is this a URL or a search term? Is the page in cache?
#    HSTS list: if the site is HSTS, http:// is upgraded to https:// before any request.

# 1. DNS lookup
#    browser cache -> OS cache -> resolver -> root (.) -> .com -> example.com's name server
#    trade.example.com -> 203.0.113.10

# 2. TCP handshake (1 round trip)
#    client: SYN  ->  server: SYN-ACK  ->  client: ACK

# 3. TLS 1.3 handshake (1 more round trip)
#    ClientHello (ciphers, key share) -> ServerHello + certificate -> keys agreed
#    Browser checks the certificate is valid and signed by a trusted CA.

# 4. HTTP request
GET /watchlist HTTP/2
Host: trade.example.com
Cookie: session=abc123
Accept: text/html

# 5. HTTP response
HTTP/2 200
content-type: text/html; charset=utf-8
cache-control: no-cache
```

What the browser does with the HTML:

```html
<!doctype html>
<html>
  <head>
    <!-- CSS is render-blocking: nothing is painted until the CSSOM is ready -->
    <link rel="stylesheet" href="/app.css" />
    <!-- defer: download in parallel, run after parsing finishes -->
    <script src="/app.js" defer></script>
  </head>
  <body>
    <h1>Watchlist</h1>
    <!-- A normal <script> here would STOP the parser until it downloads and runs -->
    <img src="/logo.png" alt="Logo" />
  </body>
</html>
```

Order of events you can observe in JS:

```js
document.addEventListener('DOMContentLoaded', () => {
  // HTML fully parsed, DOM is ready, deferred scripts have run
  console.log('DOM ready');
});

window.addEventListener('load', () => {
  // Everything else done too: images, stylesheets, iframes
  console.log('page fully loaded');
});
```

## When to use it
- This is the base for any performance talk. Each step has a fix: DNS and connection cost is cut with `preconnect`, request cost with HTTP caching and a CDN, parse cost with `defer`, render cost with smaller CSS and fewer layout changes.
- For a trading app, the first load decides how fast a user sees prices. Server-side rendering in SvelteKit sends ready HTML so the user sees the watchlist before JS finishes loading.

## Likely questions
### Walk me through what happens when I type a URL and press Enter.
First the browser checks if the input is a URL and if it has a cached copy. Then DNS turns the domain into an IP, checking browser, OS and resolver caches before asking root, TLD and the domain's name server. Then it opens a TCP connection and does a TLS handshake for HTTPS. It sends the HTTP request, gets HTML back, and starts parsing it into the DOM while downloading CSS and JS. Once DOM and CSSOM are ready it builds the render tree, does layout, paint and composite, and the user sees the page.

### How does DNS resolution work?
The browser asks its own cache, then the OS cache (which includes the hosts file). If nobody knows, the request goes to a recursive resolver (usually your ISP or something like 1.1.1.1). That resolver asks a root server, which points to the `.com` server, which points to the domain's authoritative name server, which gives the IP. Each answer has a TTL, which says how long it can be cached.

### What is the TCP handshake and the TLS handshake?
TCP handshake is three messages: SYN, SYN-ACK, ACK. After it both sides know the other is ready, and TCP gives ordered, reliable delivery. TLS runs on top: the client sends supported ciphers and a key share, the server sends its certificate and its key share, and both derive the same secret key. In TLS 1.3 this takes one round trip. The certificate proves the server really owns the domain.

### How does the browser parse HTML, and what blocks it?
The parser reads HTML top to bottom and builds DOM nodes. A normal `<script>` stops the parser: the browser must download and run it first, because the script could change the DOM with `document.write`. CSS does not stop parsing, but it blocks rendering, and it also blocks script execution because scripts may read styles. A "preload scanner" looks ahead in the HTML to start downloading images and scripts early, even while the main parser is blocked.

### What is the critical rendering path?
It is the steps from bytes to pixels: DOM, CSSOM, render tree, layout, paint, composite. "Critical" means the resources needed for the first paint. To speed it up: inline small critical CSS, `defer` scripts, keep the HTML small, and avoid large render-blocking files.

### What changes with HTTP/2 and HTTP/3?
HTTP/2 sends many requests over one TCP connection at the same time (multiplexing) and compresses headers. HTTP/3 runs over QUIC, which is built on UDP. It has TLS built in, so the setup takes fewer round trips, and one lost packet does not block every other stream (no TCP head-of-line blocking).

### What is the difference between DOMContentLoaded and load?
`DOMContentLoaded` fires when the HTML is fully parsed and deferred scripts have run. `load` fires later, when images, stylesheets and iframes are also done. Start your app logic on `DOMContentLoaded`, not `load`.

## Common mistakes
- Forgetting that CSS is render-blocking. A huge stylesheet delays first paint even if the HTML arrived fast.
- Saying "the browser renders after downloading everything". It is incremental: it parses and can paint parts while still streaming.
- Mixing up DNS caching with HTTP caching. DNS caches the IP; HTTP caching stores the response.
- Forgetting redirects: `http://` to `https://` or `example.com` to `www.example.com` each cost a full extra round trip.

## Resources
- [MDN: How browsers work](https://developer.mozilla.org/en-US/docs/Web/Performance/How_browsers_work) - the full journey from navigation to paint
- [MDN: Critical rendering path](https://developer.mozilla.org/en-US/docs/Web/Performance/Critical_rendering_path) - DOM, CSSOM, render tree, layout, paint
- [web.dev: Critical rendering path](https://web.dev/articles/critical-rendering-path) - how to optimise each step
- [MDN: An overview of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview) - requests, responses and headers
