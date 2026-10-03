# Script loading

> **In one line:** A normal `<script>` stops HTML parsing until it downloads and runs; `defer` downloads in parallel and runs in order after parsing; `async` downloads in parallel and runs as soon as it arrives, in any order.

## Key points
- **Normal `<script src>`:** parser stops, script downloads, script runs, parser continues. Slow and blocks the page.
- **`defer`:** downloads while the HTML keeps parsing. Runs after parsing ends, just before `DOMContentLoaded`, **in the order written**. Best for app code that needs the DOM.
- **`async`:** downloads while parsing. Runs the moment it is ready (which may pause parsing for a moment), **in no fixed order**. Best for independent scripts like analytics.
- **`type="module"`:** deferred by default. Supports `import`/`export`, runs in strict mode, and is only run once even if included twice. Add `async` to make a module run as soon as it is ready.
- **Resource hints:** `preload` = "I need this file for this page, fetch it now". `prefetch` = "the next page will probably need this, fetch it when idle". `preconnect` = "open the connection (DNS + TCP + TLS) to this server early".

## Example
```html
<head>
  <!-- Open the connection to the price API early: saves DNS + TCP + TLS time later -->
  <link rel="preconnect" href="https://api.example-broker.com" crossorigin />

  <!-- This font is needed for the first paint, fetch it at high priority now -->
  <link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin />

  <!-- The user will likely open the order page next, fetch it at low priority -->
  <link rel="prefetch" href="/js/order-page.js" />

  <!-- Blocks parsing: avoid in <head> -->
  <script src="/legacy.js"></script>

  <!-- Runs after parsing, in order: vendor.js first, then app.js -->
  <script src="/vendor.js" defer></script>
  <script src="/app.js" defer></script>

  <!-- Runs whenever it is downloaded; does not depend on anything -->
  <script src="https://analytics.example.com/a.js" async></script>

  <!-- Module: deferred automatically -->
  <script type="module" src="/main.js"></script>
</head>
```

Timeline picture (`|` = parser running, `D` = download, `R` = run):

```bash
normal:  |||||| DDDD RR ||||||||          parser waits for download + run
defer:   |||||||||||||||||| RR            download in parallel, run at the end
async:   ||||||||| RR |||||||||           download in parallel, run as soon as ready
```

## When to use it
- In a trading dashboard: main app bundle with `defer` or `type="module"`, analytics and chat widgets with `async`, `preconnect` to the market-data and WebSocket host, `preload` for the hero font or the critical chart library.
- SvelteKit already emits `type="module"` scripts and `modulepreload` links for the JS chunks of the current page, so you rarely add these by hand there.

## Likely questions
### What is the difference between async and defer?
Both download without blocking the parser. `defer` waits until parsing is done and runs scripts in the order they appear, before `DOMContentLoaded`. `async` runs each script as soon as it finishes downloading, so the order is not guaranteed and it may run before the DOM is complete. Use `defer` for your own code that depends on other scripts or the DOM, and `async` for independent third-party scripts.

### What happens with a normal script tag?
The HTML parser stops at the tag. It waits for the file to download and run, then continues. That is why old advice was to put scripts at the end of `<body>`. Today `defer` in the `<head>` is better, because the download starts earlier.

### How are module scripts loaded?
`<script type="module">` behaves like `defer` by default. It also loads its `import`s, runs in strict mode, has its own scope (top-level variables are not global), and needs CORS for cross-origin files. Old browsers skip it, so `<script nomodule>` was used as a fallback. `<link rel="modulepreload">` preloads a module and its dependency.

### preload vs prefetch vs preconnect?
`preload` fetches a resource for the **current** page at high priority; you must give `as` (script, style, font, image) and you should use it soon, or Chrome warns in the console. `prefetch` fetches something for a **future** navigation at low priority and puts it in the HTTP cache. `preconnect` does not fetch a file at all; it only sets up the connection to another origin. `dns-prefetch` is a lighter version that only does the DNS lookup.

### Do async and defer work on inline scripts?
No. They only work on scripts with a `src`. An inline classic script always runs right away. The exception is an inline `type="module"` script, which is deferred, and can take `async`.

## Common mistakes
- Using `async` for scripts that depend on each other (for example a library and the code using it). They can run in the wrong order.
- Preloading too many files. Preload competes for bandwidth with things the page really needs first.
- Preloading a font without `crossorigin`. Fonts are fetched in CORS mode, so the preload is not reused and the font downloads twice.
- Using `preconnect` for many origins. Each open connection costs CPU and is closed if unused for a few seconds. Keep it to 2 or 3 key origins.

## Resources
- [MDN: The script element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script) - async, defer, type=module details
- [javascript.info: Scripts async, defer](https://javascript.info/script-async-defer) - clear diagrams and examples
- [MDN: rel=preload](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preload) - how preload and `as` work
- [web.dev: Preconnect and dns-prefetch](https://web.dev/articles/preconnect-and-dns-prefetch) - when early connections help
