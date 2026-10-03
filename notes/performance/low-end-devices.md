# Low-end devices

> **In one line:** Many of our users are on budget Android phones with slow CPUs and patchy networks, so I ship less JavaScript, keep animations light, and test with CPU and network throttling instead of only on my fast laptop.

## Key points
- On low-end phones the **CPU** is the bottleneck: the same JS can take several times longer to parse and run than on a laptop. Bytes of JS cost more than bytes of images because JS must be parsed and executed.
- **Smaller JS**: code-split by route, lazy-load heavy widgets (charts, editors), remove unused libraries, check bundle size. Svelte helps because it compiles away most framework code.
- **Fewer, cheaper animations**: only `transform`/`opacity`, and respect [`prefers-reduced-motion`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion).
- **Test realistically**: Chrome DevTools Performance panel lets you throttle the CPU (4x or 6x slowdown) and the network (e.g. "Slow 4G"). Better still, test on a real cheap Android phone via remote debugging.
- Also watch memory: big lists, many chart instances and leaks crash tabs on 2-3 GB RAM phones.

## Example
Lazy-load a heavy chart only when needed (Svelte 5):

```svelte
<script>
  let { symbol } = $props();
  let showChart = $state(false);
  let Chart = $state(null);

  async function openChart() {
    showChart = true;
    // The chart library is in a separate chunk; low-end users who never open it never download it.
    Chart = (await import('./PriceChart.svelte')).default;
  }
</script>

<button onclick={openChart}>Show chart</button>
{#if showChart && Chart}
  <Chart {symbol} />
{/if}
```

Reduce motion for users who ask for it (and save CPU):

```css
.price-flash { transition: background-color 200ms; }
@media (prefers-reduced-motion: reduce) {
  .price-flash { transition: none; }
}
```

Optional: adapt to the device ([`navigator.hardwareConcurrency`](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/hardwareConcurrency) is widely supported; `navigator.deviceMemory` is Chromium-only):

```js
const lowEnd = (navigator.deviceMemory ?? 8) <= 2 || navigator.hardwareConcurrency <= 4;
const tickThrottleMs = lowEnd ? 1000 : 250; // update the watchlist less often on slow phones
```

## When to use it
- Watchlist with live prices: batch WebSocket updates and repaint at most a few times per second on slow devices.
- Landing and login pages: keep them tiny so first load on 3G/4G is fast.
- e.g. in my last project trimming and splitting bundles helped improve page load by about 300 ms.

## Likely questions
### How do you make an app fast on low-end devices?
First I measure on a throttled or real low-end device. Then I reduce JavaScript: route-based code splitting, lazy-load heavy parts, drop unused dependencies. I keep the main thread free (chunk work, Web Workers), keep animations to `transform`/`opacity`, virtualise long lists, and optimise images (right size, modern formats, lazy-loading).

### How do you test for low-end devices?
In Chrome DevTools I use CPU throttling (4x or 6x slowdown) and network throttling, and run Lighthouse in mobile mode, which simulates a mid-range phone. I also check field data (real-user Web Vitals) split by device type, because lab tests do not show the real spread of devices.

### Why is JS more expensive than an image of the same size?
An image is decoded, often off the main thread. JavaScript must be downloaded, parsed, compiled and executed on the main thread, and it blocks interaction while running. So 200 KB of JS hurts much more than 200 KB of image on a slow CPU.

### Why does this matter for India?
A large share of users in India use budget Android phones on mobile data that can be slow or costly. A heavy app loses those users. Budgets, small bundles and offline-friendly behaviour directly affect conversion and retention.

## Common mistakes
- Only testing on a MacBook and a flagship phone.
- Adding a big library (date, chart, utility) for one small function.
- Many simultaneous animations or blur/shadow effects on scroll.

## Resources
- [web.dev: Reduce JavaScript payloads with code splitting](https://web.dev/articles/reduce-javascript-payloads-with-code-splitting) - how to ship less JS
- [Chrome DevTools: Performance panel](https://developer.chrome.com/docs/devtools/performance) - CPU and network throttling
- [MDN: prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion) - respecting motion settings
- [Chrome: Remote debug Android devices](https://developer.chrome.com/docs/devtools/remote-debugging) - test on a real cheap phone
