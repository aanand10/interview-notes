# Rendering pipeline

> **In one line:** The browser turns HTML into the DOM and CSS into the CSSOM, joins them into a render tree, then runs layout, paint and composite to get pixels on screen, and CSS plus synchronous scripts are what block that first render.

## Key points
- **Parse:** HTML is parsed into the [DOM](https://developer.mozilla.org/en-US/docs/Glossary/DOM) (a tree of nodes). CSS is parsed into the [CSSOM](https://developer.mozilla.org/en-US/docs/Glossary/CSSOM) (a tree of style rules). Both are built step by step as bytes arrive.
- **Style + render tree:** the browser matches CSS rules to DOM nodes and computes the final style for each one. The render tree only holds things that will be drawn. `display: none` elements and `<head>` are left out. `visibility: hidden` elements stay in, because they still take up space.
- **Layout (also called reflow):** works out the exact size and position of every box, based on the viewport size.
- **Paint:** fills in pixels for text, colours, borders, shadows and images, often into several layers.
- **Composite:** the compositor thread puts the layers together in the right order and sends them to the GPU. Changes to `transform` and `opacity` can skip straight to this step, which is why they are cheap.

## Example
```html
<!doctype html>
<html>
  <head>
    <!-- Render-blocking: nothing is painted until this CSS is downloaded and parsed -->
    <link rel="stylesheet" href="/app.css" />

    <!-- Parser-blocking: HTML parsing stops here until the script downloads and runs.
         It also waits for app.css first, because the script might read styles. -->
    <script src="/legacy-analytics.js"></script>

    <!-- Not blocking: downloads in parallel, runs after the HTML is parsed -->
    <script src="/app.js" defer></script>

    <!-- Print CSS does not block the first screen render -->
    <link rel="stylesheet" href="/print.css" media="print" />
  </head>
  <body>
    <h1>Watchlist</h1>
    <div id="prices"></div>
  </body>
</html>
```

The order of work for this page:

```bash
bytes -> tokens -> DOM nodes          # HTML parser (pauses at legacy-analytics.js)
app.css -> CSSOM                      # runs in parallel with the HTML download
DOM + CSSOM -> style -> render tree   # only visible nodes
layout -> paint -> composite          # first pixels on screen (first paint)
```

## When to use it
- **Fast first paint for a trading dashboard:** inline the small "above the fold" CSS (the header, the ticker strip), load the rest without blocking, and put every script on `defer` or `type="module"`. Users see the price grid sooner.
- **Debugging a slow page:** the Chrome DevTools Performance panel shows these exact steps (Parse HTML, Recalculate Style, Layout, Paint, Composite Layers). Knowing the pipeline tells you which one to fix.
- **Svelte:** SvelteKit does server-side rendering (SSR) by default, so the HTML shows up with content already in it. The browser can paint before any JS runs, and hydration (attaching the JS to the existing HTML) happens after.

## Likely questions

### Walk me through how the browser renders a page.
The HTML parser reads bytes, turns them into tokens and builds the DOM. When it finds a stylesheet it fetches it and builds the CSSOM. Then the browser combines them: for every visible node it works out the computed style, and that gives the render tree. Layout works out where each box goes and how big it is. Paint draws the pixels into layers. Composite stacks the layers and shows them on screen. After the first render, any change to the DOM or styles runs some part of this pipeline again.

### Why is CSS render-blocking?
If the browser painted before the CSS arrived, users would see unstyled content that then jumps around. This is called a "flash of unstyled content". So the browser waits for all CSS in the `<head>` that applies to the current media before it paints anything. That is why CSS should be small and come early. A stylesheet with a `media` query that does not match, like `media="print"`, still downloads but does not block rendering.

### Why does a normal script block parsing?
A classic `<script>` with no `async` or `defer` can call `document.write` or change the DOM. So the parser must stop, download the script and run it before it goes on. Also, the script might read styles (for example `getComputedStyle`), so it has to wait for any CSS above it to finish. That means CSS can end up blocking JavaScript, and JavaScript blocks the HTML parser. The fix is `defer`, `async` or `type="module"`, or putting scripts at the end of `<body>`.

### What is the difference between the DOM and the render tree?
The DOM holds every node in the document, including `<head>`, `<script>` and hidden elements. The render tree only holds what will be drawn, with computed styles attached. `display: none` removes a node from the render tree, while `visibility: hidden` keeps it (it still takes space, it just is not painted).

### What is the preload scanner?
While the main parser is blocked on a script, a second lightweight parser called the preload scanner looks ahead in the raw HTML. It starts downloading images, CSS and scripts it finds. This is why resources written in the HTML load faster than ones added later by JS. Hiding a hero image inside a JS-only component means the scanner cannot find it early.

### What is the critical rendering path, and how do you shorten it?
It is the minimum set of steps and resources the browser needs before it can paint the first screen. To shorten it: send fewer critical bytes (minify CSS, inline only the critical CSS), have fewer render-blocking resources (`defer` scripts, use `media` on non-critical CSS), and cut round trips (`preconnect` to the API or CDN origin, use HTTP/2).

### What happens on the screen after the first paint, when something changes?
JS or CSS changes start the pipeline again from the earliest step that is affected. If you change geometry, you get style, then layout, then paint, then composite. If you change a colour, it skips layout and goes to paint. If you change `transform` or `opacity`, the browser can often do composite only. See the reflow vs repaint note.

## Common mistakes
- Saying "JavaScript blocks rendering" with no detail. To be precise: sync scripts block the parser, and CSS blocks rendering (and blocks script execution).
- Thinking `visibility: hidden` and `display: none` are the same thing for rendering.
- Putting `@import` inside CSS files. Each import is another round trip, discovered late.
- Forgetting that web fonts can delay text. Use `font-display: swap` so fallback text shows first.

## Resources
- [web.dev: Critical rendering path](https://web.dev/articles/critical-rendering-path) - the official step-by-step series
- [web.dev: Render-blocking CSS](https://web.dev/articles/critical-rendering-path/render-blocking-css) - why CSS blocks paint, plus media queries
- [MDN: How browsers work](https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/How_browsers_work) - the whole pipeline in one page
- [Chrome: Inside look at a modern browser, part 3](https://developer.chrome.com/blog/inside-browser-part3) - the renderer process, with diagrams
- [MDN: Critical rendering path](https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/Critical_rendering_path) - short reference
