# Core Web Vitals

> **In one line:** Core Web Vitals are Google's three user-focused metrics: LCP for loading, INP for responsiveness, and CLS for visual stability, each judged at the 75th percentile of real users.

## Key points
- **LCP (Largest Contentful Paint):** time until the biggest image or text block in the viewport is painted. Good is **2.5 s or less**, poor is over 4 s. See [LCP](https://web.dev/articles/lcp).
- **INP (Interaction to Next Paint):** how long the page takes to visually respond to clicks, taps and key presses, looking at (roughly) the worst interaction of the visit. Good is **200 ms or less**, poor is over 500 ms. It replaced FID (First Input Delay) in March 2024. See [INP](https://web.dev/articles/inp).
- **CLS (Cumulative Layout Shift):** how much visible content jumps around unexpectedly. It is a unitless score. Good is **0.1 or less**, poor is over 0.25. See [CLS](https://web.dev/articles/cls).
- A page "passes" when **75% of page visits** meet the good threshold for all three. This is field data from real users, not one Lighthouse run.
- Supporting metrics help diagnose: TTFB (server response), FCP (first content shown), TBT (Total Blocking Time, the lab stand-in for INP).

| Metric | Measures | Good | Needs improvement | Poor |
| --- | --- | --- | --- | --- |
| LCP | Loading | 2.5 s or less | 2.5 s to 4 s | over 4 s |
| INP | Responsiveness | 200 ms or less | 200 ms to 500 ms | over 500 ms |
| CLS | Visual stability | 0.1 or less | 0.1 to 0.25 | over 0.25 |

## Example
Measuring all three in production with Google's `web-vitals` library:

```js
import { onLCP, onINP, onCLS } from 'web-vitals';

function sendToAnalytics(metric) {
  // metric.name: 'LCP' | 'INP' | 'CLS', metric.rating: 'good' | 'needs-improvement' | 'poor'
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    page: location.pathname,
  });
  // sendBeacon survives the page being closed
  navigator.sendBeacon('/analytics/vitals', body);
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
```

Common fixes in HTML and CSS:

```html
<!-- LCP: the hero image loads early with high priority, and is NOT lazy -->
<img src="/hero.avif" width="1200" height="600" fetchpriority="high" alt="Markets today" />

<!-- CLS: width/height (or aspect-ratio) reserve space before the image loads -->
<img src="/logo.webp" width="120" height="40" alt="Company logo" loading="lazy" />
```

```css
/* CLS: reserve space for a live price ticker that loads later */
.ticker-slot { min-height: 48px; }
```

## When to use it
In a trading app: LCP is how fast the portfolio value or chart appears. INP is how fast the "Buy" button or a watchlist tab responds; a laggy order button feels risky to users. CLS matters a lot: if the layout shifts just as a user taps, they might hit "Sell" instead of "Buy". Core Web Vitals also affect Google search ranking for public pages like stock detail pages.

## Likely questions
### What are the Core Web Vitals and what do they measure?
There are three. LCP measures loading: when the largest visible content element is painted. INP measures responsiveness: the delay from a user interaction until the next frame is painted. CLS measures visual stability: how much the layout jumps unexpectedly. Thresholds are 2.5 s, 200 ms and 0.1, measured at the 75th percentile of real users.

### How do you improve LCP?
LCP is made of four parts: server time (TTFB), the delay before the LCP resource starts loading, its download time, and render delay. So I make the server and CDN fast, make the LCP image discoverable in the HTML (not injected by JS), add `fetchpriority="high"` or a preload, never lazy-load it, compress it (AVIF/WebP, right size), and remove render-blocking CSS and JS. Server-side rendering helps because text is in the first HTML. See [Optimize LCP](https://web.dev/articles/optimize-lcp).

### How do you improve INP?
An interaction has three parts: input delay (main thread busy with something else), processing time (our event handlers), and presentation delay (rendering the next frame). I break up long tasks, keep handlers small and push non-urgent work later, avoid huge DOM updates, debounce expensive input work, and move heavy computation to a Web Worker. Showing instant feedback first (like a pressed state or spinner) and then doing the work also helps. See [Optimize INP](https://web.dev/articles/optimize-inp).

### How do you improve CLS?
Always set `width` and `height` or `aspect-ratio` on images, videos and iframes. Reserve space for ads, banners and late-loading widgets. Do not insert content above existing content unless it is a response to a user action (shifts within 500 ms of an input do not count). Use `font-display` with a matching fallback font to reduce font-swap shifts. Animate with `transform` instead of `top` or `height`. See [Optimize CLS](https://web.dev/articles/optimize-cls).

### Why did INP replace FID?
FID only measured the input delay of the first interaction, so a page could pass FID and still feel slow later. INP looks at all interactions during the visit and includes the full time until the next paint, so it reflects real responsiveness much better. It became a Core Web Vital in March 2024.

### Can Lighthouse measure INP?
A normal Lighthouse page-load run has no real user clicks, so it cannot measure INP. It reports TBT (Total Blocking Time) as a lab proxy: lots of long tasks during load usually means poor INP. To see INP you need field data (web-vitals library, CrUX) or you record yourself clicking in the Performance panel.

### Why the 75th percentile?
It means most users (3 out of 4 visits) get a good experience, but it is not so strict that a few users on terrible devices or networks make it impossible to pass. Read [how the thresholds were defined](https://web.dev/articles/defining-core-web-vitals-thresholds).

## Common mistakes
- Lazy-loading the hero image. That delays LCP.
- Thinking a 100 Lighthouse score means you pass Core Web Vitals. Passing is decided by field data.
- Still talking about FID as a Core Web Vital.
- Forgetting that CLS also counts shifts after load, like a banner injected after 10 seconds.
- Reporting averages instead of the 75th percentile.

## Resources
- [web.dev: Web Vitals](https://web.dev/articles/vitals) - official definitions and thresholds
- [web.dev: Optimize LCP](https://web.dev/articles/optimize-lcp) - the four LCP sub-parts and fixes
- [web.dev: Optimize INP](https://web.dev/articles/optimize-inp) - input delay, processing, presentation
- [web.dev: Optimize CLS](https://web.dev/articles/optimize-cls) - common causes of layout shifts
- [web.dev: Most effective ways to improve Core Web Vitals](https://web.dev/articles/top-cwv) - prioritised checklist
