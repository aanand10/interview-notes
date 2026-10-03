# Making a React app load faster

> **In one line:** I first measure (Lighthouse, Performance panel, Web Vitals), then ship less JavaScript (code splitting, tree shaking, smaller libraries), render earlier (SSR/SSG), and serve everything fast and cached from a CDN with optimized images and fonts.

## Key points
- **Measure first**: [Core Web Vitals](https://web.dev/articles/vitals) are LCP (main content shows, good under 2.5s), INP (responds to input, under 200ms) and CLS (layout stays stable, under 0.1).
- **Ship less JS**: split code by route, lazy-load heavy widgets, tree-shake, replace big libraries.
- **Render earlier**: SSR or SSG sends real HTML, so the user sees content before all JS runs.
- **Deliver faster**: CDN, compression (Brotli), long cache for hashed files, preload the critical resources.
- **Assets**: modern image formats, correct sizes, lazy-load below-the-fold images, fast fonts.

## Example
Route-level code splitting with [`React.lazy`](https://react.dev/reference/react/lazy) and `Suspense`. The chart library is only downloaded when the user opens the chart page.

```tsx
import { lazy, Suspense } from "react";
import { Routes, Route } from "react-router";

const Dashboard = lazy(() => import("./pages/Dashboard"));
const ChartPage = lazy(() => import("./pages/ChartPage")); // pulls in a heavy charting lib

export function App() {
  return (
    <Suspense fallback={<div className="skeleton" aria-busy="true" />}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/chart/:symbol" element={<ChartPage />} />
      </Routes>
    </Suspense>
  );
}
```

## When to use it
A trading web app: the login and watchlist must appear fast on a phone over 4G. The advanced chart, options chain and reports are big and used by fewer people, so they are split out and loaded on demand.

## Likely questions

### How do you make a React application load faster?
I start by measuring so I fix the real problem. Then I work in layers:
1. **Less JS**: route-based code splitting, lazy-load heavy components, tree shaking, drop unused libraries and polyfills.
2. **Faster first paint**: SSR or SSG so HTML arrives with content; inline critical CSS.
3. **Faster network**: CDN, Brotli/gzip, HTTP/2 or 3, long caching of hashed files, `preconnect` to the API domain.
4. **Assets**: compressed, right-sized images; preload the LCP image; `font-display: swap`.
5. **Less work on start**: avoid big synchronous work in the first render, fetch data in parallel, not in waterfalls.

### What is code splitting?
Code splitting means breaking one big JavaScript bundle into smaller chunks that load when needed. The bundler (Vite, webpack) creates a new chunk at every dynamic `import()`. The most common split is per route, so the login page doesn't download the code for the chart page.

### What is lazy loading?
Lazy loading means loading something only when it's needed, not upfront. In React, `React.lazy(() => import("./X"))` plus `<Suspense>` loads a component the first time it renders. For images, `<img loading="lazy">` waits until the image is near the viewport. You can also lazy-load on interaction, for example import the export-to-PDF code only when the user clicks "Export".

### What is tree shaking?
[Tree shaking](https://developer.mozilla.org/en-US/docs/Glossary/Tree_shaking) is when the bundler removes code you import but never use. It works with **ES modules** (`import`/`export`), because those are static and the bundler can see exactly what is used. It doesn't work well with CommonJS `require`. A module must also be **side-effect free** to be safely dropped; libraries say so with `"sideEffects": false` in `package.json` (or list the files that do have side effects, like CSS).

```tsx
import _ from "lodash";             // pulls in all of lodash (CommonJS, ~70KB min)
import { debounce } from "lodash-es"; // ES modules: only debounce and its helpers are bundled
```

### How do you reduce the initial JavaScript bundle size?
- **See what's in it**: a bundle analyzer (`rollup-plugin-visualizer` for Vite, `webpack-bundle-analyzer`) shows the biggest modules.
- **Dynamic imports** for routes, modals, charts, rich text editors.
- **Smaller libraries**: `date-fns` or `Intl` instead of moment, native `fetch` instead of axios if not needed.
- **Remove polyfills** you don't need by targeting modern browsers (browserslist).
- **Compression**: Brotli on the server or CDN, and minification in the build.
- Use Chrome DevTools [Coverage](https://developer.chrome.com/docs/devtools/coverage) to find code that loads but never runs.

### What are SSR and SSG (and CSR, ISR)?
- **CSR (client-side rendering)**: server sends an almost empty HTML and a JS bundle; the browser builds the page. Cheap to host, but slow LCP and weaker SEO.
- **SSR (server-side rendering)**: the server renders HTML on each request. Content shows quickly and is SEO-friendly, but TTFB (time to first byte) is higher because the server does work, and the page still needs **hydration** (React attaching event handlers) before it's interactive.
- **SSG (static site generation)**: HTML is built once at build time and served from a CDN. Fastest TTFB and LCP, but the data can be stale. Good for marketing and help pages.
- **ISR (incremental static regeneration)**: SSG pages that are rebuilt in the background after a time limit (a Next.js feature). Fast like SSG, fresher than SSG.

For a trading app: SSG for landing and pricing pages, SSR for SEO pages like a public stock page, and CSR with live WebSocket data for the logged-in dashboard.

### How do caching and a CDN improve load performance?
A [CDN](https://developer.mozilla.org/en-US/docs/Glossary/CDN) keeps copies of your files on servers close to the user, so each request travels less distance. For caching, build tools put a content hash in file names, like `app.3f9a2c.js`. Because the name changes whenever the content changes, you can cache these forever: `Cache-Control: public, max-age=31536000, immutable`. The `index.html` uses `no-cache` so the browser always checks for a new version, which then points to new hashed files. Repeat visits load almost nothing from the network. CDNs can also cache HTML or API responses at the edge for a short time.

### How do you optimize images and static assets?
- Use modern formats: **AVIF or WebP**, with fallback via `<picture>`. SVG for icons and logos.
- Serve the right size with [`srcset` and `sizes`](https://developer.mozilla.org/en-US/docs/Web/HTML/Guides/Responsive_images), so phones don't download desktop images.
- `loading="lazy"` for below-the-fold images, but **never** for the LCP image.
- Preload the LCP image and give it `fetchpriority="high"`.
- Always set `width` and `height` to avoid layout shift (CLS).
- Fonts: WOFF2, subset to needed characters, `font-display: swap`, preload the main font.

```tsx
<img
  src="/hero-800.webp"
  srcSet="/hero-400.webp 400w, /hero-800.webp 800w, /hero-1600.webp 1600w"
  sizes="(max-width: 600px) 100vw, 800px"
  width={800} height={450}
  fetchPriority="high"
  alt="Portfolio overview"
/>
```

### The page takes 5 seconds to load. How do you investigate, step by step?
1. **Reproduce like a user**: Chrome incognito, DevTools with "Slow 4G" and CPU throttling, cache disabled. Also check real-user data (field data) if we collect Web Vitals.
2. **Lighthouse**: gives LCP, TBT, CLS and a list of opportunities. Tells me which metric is bad.
3. **Network waterfall**: is TTFB slow (server/backend problem)? Is the JS bundle huge? Are there request chains (HTML, then JS, then API, then another API)? Are files uncompressed or uncached?
4. **Performance panel**: record the load. Look for long tasks (over 50ms) on the main thread, slow hydration, heavy scripts.
5. **Coverage tab**: how much loaded JS/CSS is unused on this page.
6. **Bundle analyzer**: which libraries are big.
7. **Fix the biggest cause first, measure again**, and add monitoring (the [`web-vitals`](https://web.dev/articles/vitals) library, performance budgets in CI) so it doesn't regress.

### What is the difference between preload, prefetch and preconnect?
`preload` fetches a resource needed for **this** page, early and with high priority (LCP image, main font). `prefetch` fetches something likely needed for the **next** navigation, at low priority. `preconnect` opens the connection (DNS, TCP, TLS) to another origin early, like your API or CDN domain.

## Common mistakes
- Lazy-loading the LCP hero image, which makes LCP worse.
- Splitting into too many tiny chunks, creating long request chains.
- Optimizing without measuring, or only measuring on a fast laptop.
- Forgetting that SSR still needs hydration: a big bundle still hurts interactivity (INP).
- Caching `index.html` for a long time, so users never get the new release.

## Resources
- [web.dev: Web Vitals](https://web.dev/articles/vitals) - what LCP, INP and CLS mean and the targets
- [react.dev: lazy](https://react.dev/reference/react/lazy) - code splitting components with Suspense
- [web.dev: Optimize LCP](https://web.dev/articles/optimize-lcp) - step-by-step fixes for slow main content
- [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching) - Cache-Control and long-lived hashed assets
