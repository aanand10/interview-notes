# Application performance optimisation (the full answer)

> **In one line:** I split performance into load (ship less, ship it sooner, cache it), runtime (keep the main thread free, render less) and perceived speed (show something useful fast), measure each with Core Web Vitals and traces, fix the biggest bottleneck first, and guard it with budgets and production monitoring.

## Key points
Use this as the map when someone asks "How do you optimise a frontend application?". Go layer by layer and give one concrete example per layer.

| Layer | Goal | Techniques |
| ----- | ---- | ---------- |
| **Measure** | Know what is slow | Lighthouse, DevTools Performance trace, Web Vitals (LCP, INP, CLS) from real users, bundle analyzer |
| **Network** | Fewer, smaller, closer bytes | HTTP/2/3, CDN, Brotli, `preconnect`, `preload` the LCP image/font, fewer third-party scripts |
| **JavaScript size** | Ship less JS | Code splitting per route, `import()` for heavy widgets, tree shaking, replace big libraries, remove polyfills modern browsers do not need |
| **CSS** | Do not block paint | Critical CSS inline, async the rest, remove unused CSS, split per route |
| **Images / fonts** | Smaller media | AVIF/WebP, `srcset` + `sizes`, `loading="lazy"` below the fold, `fetchpriority="high"` on the LCP image, `font-display: swap`, subset fonts |
| **Caching** | Do not download twice | Hashed file names + `immutable`, `stale-while-revalidate`, service worker, HTTP caching for API, client cache (TanStack Query, SWR, LRU) |
| **Rendering strategy** | HTML with content early | SSR / SSG / streaming, partial hydration, islands, server components |
| **Main thread** | Stay responsive (INP) | Break long tasks (`scheduler.yield()`, `setTimeout` chunks), debounce/throttle, move heavy work to Web Workers |
| **DOM / rendering** | Less work per frame | Virtualise long lists, avoid layout thrashing, animate `transform`/`opacity`, `content-visibility`, fewer DOM nodes |
| **Framework** | Fewer re-renders | React: `memo`, `useMemo`, `useCallback`, stable keys, split context, `useTransition`. Svelte/Solid: fine-grained updates by default, keep `$derived` cheap |
| **Memory** | No slow-down over time | Clean up listeners, timers, sockets, observers; cap caches |
| **Perceived** | Feels fast | Skeletons, optimistic UI, prefetch on hover, instant feedback on click |
| **Keep it fast** | No regressions | Performance budgets in CI (Lighthouse CI, `size-limit`), RUM dashboards and alerts |

## Example
A few high-impact changes in code:

```js
// 1. Route-level code splitting (React)
const Reports = lazy(() => import('./pages/Reports'));

// 2. Load a heavy library only when it is needed
async function exportToExcel(rows) {
  const { utils, writeFile } = await import('xlsx'); // not in the main bundle
  const sheet = utils.json_to_sheet(rows);
  /* ... */
}

// 3. Break a long task so clicks are handled in between (better INP)
async function processAll(rows) {
  for (let i = 0; i < rows.length; i++) {
    process(rows[i]);
    if (i % 500 === 0) {
      // give the browser a chance to handle input and paint
      await (globalThis.scheduler?.yield?.() ?? new Promise((r) => setTimeout(r, 0)));
    }
  }
}

// 4. Heavy computation off the main thread
const worker = new Worker(new URL('./indicators.worker.js', import.meta.url), { type: 'module' });
worker.postMessage({ candles });
worker.onmessage = (e) => drawChart(e.data);

// 5. Prefetch the next page on hover / when visible
link.addEventListener('pointerenter', () => import('./pages/OrderDetails'), { once: true });
```

```html
<!-- 6. LCP image: discover early, high priority, right size -->
<link rel="preload" as="image" href="/hero-800.avif" fetchpriority="high">
<img src="/hero-800.avif"
     srcset="/hero-400.avif 400w, /hero-800.avif 800w, /hero-1600.avif 1600w"
     sizes="(max-width: 600px) 100vw, 800px"
     width="800" height="450" alt="Portfolio overview" fetchpriority="high">

<!-- 7. Below-the-fold images -->
<img src="/chart-thumb.webp" loading="lazy" decoding="async" width="320" height="180" alt="">
```

## When to use it
- **"How would you optimise our app?"** Walk the table top to bottom, but say you start by measuring and pick the top 2-3 items based on data.
- **"Our page loads slowly"** -> network, JS size, CSS, images, caching, rendering strategy.
- **"The UI is laggy / typing is slow"** -> main thread, DOM, framework re-renders (this is INP).
- **"It gets slower after an hour"** -> memory leaks, growing caches, too many DOM nodes.

## Likely questions
### How do you optimise the performance of a web application?
I start by measuring: Lighthouse and a Performance trace in the lab, Web Vitals from real users. That tells me whether the problem is loading (LCP), responsiveness (INP) or layout stability (CLS). For loading, I reduce what we ship (code splitting, tree shaking, removing unused CSS, image formats), make the critical path short (preload the LCP resource, critical CSS, `defer` scripts) and cache aggressively with hashed assets and a CDN. For responsiveness, I break up long tasks, move heavy work to workers, virtualise long lists and stop unnecessary re-renders. For CLS, I give images and ads fixed dimensions. Then I measure again and add budgets so it does not regress.

### What is the first thing you check?
The biggest numbers in the data. Usually that is JavaScript bundle size (DevTools Coverage + bundle analyzer) or the LCP element (what it is and why it shows late). These two explain most slow loads.

### How do you reduce JavaScript bundle size?
Analyse it first (`rollup-plugin-visualizer`, `webpack-bundle-analyzer`). Then split by route, lazy-load heavy components (charts, editors, PDF/Excel export), use ESM imports that tree-shake (`lodash-es` per-function imports instead of all of `lodash`), swap heavy libraries for lighter ones (`date-fns` or `Intl` instead of `moment`), drop legacy polyfills with a modern browserslist, and audit third-party scripts.

### How do you improve INP (interaction speed)?
Find long tasks in the trace around the interaction. Do less work in the event handler (show feedback first, then do the heavy part), split long tasks with `scheduler.yield()`, debounce input handlers, move computation to a worker, use `startTransition` in React for non-urgent updates, and reduce the amount of DOM that changes (virtualisation, memoisation).

### SSR vs CSR for performance?
SSR (or SSG) sends HTML with content, so first paint and LCP are faster and it is better for SEO, but the server does more work and the page is not interactive until hydration. CSR has a slower first paint (blank page until JS runs) but simple hosting. Modern answers: SSR/SSG for the first page, then client navigation; streaming and server components to send less JS.

### How do you make sure performance does not regress?
Budgets in CI: fail the build when the main bundle grows past X KB (`size-limit`) or when Lighthouse CI scores drop. Real-user monitoring with Web Vitals sent to analytics, dashboards by page and device, and alerts on the 75th percentile. Review performance in code review for new dependencies.

## Common mistakes
- Listing 20 techniques without saying how you would measure or which matters most.
- Adding `useMemo` everywhere "for performance" without a profile.
- Optimising Lighthouse score on a fast laptop while real users are on mid-range Android over 4G.
- Lazy-loading the LCP image (it should be eager and high priority).
- Forgetting third-party scripts (chat widgets, tag managers), which are often the biggest cost.

## Resources
- [web.dev: Learn Performance](https://web.dev/learn/performance) - free course, covers every layer here
- [web.dev: Optimize LCP](https://web.dev/articles/optimize-lcp) - break LCP into parts and fix each
- [web.dev: Optimize INP](https://web.dev/articles/optimize-inp) - long tasks, yielding, input delay
- [web.dev: Optimize long tasks](https://web.dev/articles/optimize-long-tasks) - `scheduler.yield()` and chunking
- [patterns.dev: Performance patterns](https://www.patterns.dev/vanilla/import-on-interaction) - import on interaction, visibility, prefetching
