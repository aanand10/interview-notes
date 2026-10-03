# Monitoring performance in production

> **In one line:** Lab tools like Lighthouse tell me how the page performs in one controlled run, but real user monitoring tells me what actual users feel, so I collect Core Web Vitals from real users with the `web-vitals` library and guard against regressions with performance budgets in CI.

## Key points
- **Lab data**: a test in a fixed setup (Lighthouse, DevTools, WebPageTest). Repeatable, good for debugging, but it is one device and one network.
- **Field data / RUM** (Real User Monitoring): metrics collected from real users' browsers. Shows the true spread across devices, networks and countries. Google's CrUX report is field data too.
- **Core Web Vitals**: **LCP** (Largest Contentful Paint, loading, good is 2.5 s or less), **INP** (Interaction to Next Paint, responsiveness, good is 200 ms or less), **CLS** (Cumulative Layout Shift, visual stability, good is 0.1 or less). Judged at the **75th percentile** of users.
- The [`web-vitals`](https://github.com/GoogleChrome/web-vitals) library is a small script that measures these the same way Chrome does and gives you callbacks to send them to your backend.
- **Performance budgets**: limits like "main bundle under 170 KB gzipped" or "LCP under 2.5 s in Lighthouse". CI fails the build when a change breaks the budget.

## Example
Send Web Vitals from real users:

```js
import { onLCP, onINP, onCLS, onTTFB } from 'web-vitals';

function sendToAnalytics(metric) {
  const body = JSON.stringify({
    name: metric.name,       // 'LCP' | 'INP' | 'CLS' | 'TTFB'
    value: metric.value,
    rating: metric.rating,   // 'good' | 'needs-improvement' | 'poor'
    id: metric.id,
    page: location.pathname,
  });
  // sendBeacon still works when the page is being closed
  navigator.sendBeacon?.('/analytics', body) ||
    fetch('/analytics', { method: 'POST', body, keepalive: true });
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
onTTFB(sendToAnalytics);
```

A Lighthouse CI budget check (`lighthouserc.json`):

```json
{
  "ci": {
    "collect": { "url": ["http://localhost:4173/"], "numberOfRuns": 3 },
    "assert": {
      "assertions": {
        "largest-contentful-paint": ["error", { "maxNumericValue": 2500 }],
        "cumulative-layout-shift": ["error", { "maxNumericValue": 0.1 }],
        "total-byte-weight": ["warn", { "maxNumericValue": 400000 }]
      }
    }
  }
}
```

## When to use it
- A trading dashboard: track INP on the "place order" button and LCP on the watchlist page, split by device and network type.
- After a release, a dashboard shows if p75 LCP got worse, so you can roll back fast.
- e.g. in my last project I confirmed a ~300 ms page-load win with measurements before and after, not just a single Lighthouse run.

## Likely questions
### What is the difference between RUM and lab data?
Lab data comes from a controlled test like Lighthouse: same device, same network, easy to repeat and debug. RUM comes from real users, so it includes slow phones, bad networks and real interactions. Lab is for finding and fixing problems; RUM is the truth about user experience. INP in particular needs real interactions, so it is mainly a field metric (Lighthouse uses Total Blocking Time as a lab proxy).

### How would you monitor performance in production?
Add the `web-vitals` library, send LCP, INP, CLS and TTFB to an analytics endpoint with `sendBeacon`, and attach context like page, device type and app version. Then build dashboards on the 75th percentile and set alerts on regressions. I can also use the attribution build of `web-vitals` to learn which element caused the LCP or which interaction was slow.

### What is a performance budget and how do you enforce it?
A budget is a limit you agree on, for bundle size, request count, or a metric like LCP. I enforce it in CI with Lighthouse CI assertions or bundle-size checks, so a pull request that adds a heavy library fails before it ships.

### Why the 75th percentile and not the average?
The average hides the slow users. p75 means 75% of visits were at least this good, which is a fair target that still includes many slower devices.

## Common mistakes
- Trusting one Lighthouse score on a fast laptop as "our performance".
- Collecting metrics but sending them with plain `fetch` on unload, so many are lost.
- Budgets that are never enforced in CI.

## Resources
- [web.dev: Web Vitals](https://web.dev/articles/vitals) - the metrics and thresholds
- [web.dev: Lab and field data differences](https://web.dev/articles/lab-and-field-data-differences) - why the numbers differ
- [web-vitals library](https://github.com/GoogleChrome/web-vitals) - API and attribution build
- [web.dev: Performance budgets 101](https://web.dev/articles/performance-budgets-101) - how to set budgets
