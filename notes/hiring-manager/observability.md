# Observability

> **In one line:** "Observability means I can answer 'is it broken, for whom, and why' from data, without asking users, so on the frontend I track errors, performance, and the success rate of key user journeys, with alerts on the ones that cost money."

## What the interviewer is really checking

- Do you think beyond "it works on my machine"? Real users have slow phones, bad networks, and old WebViews.
- Can you name **concrete tools** and **concrete metrics**, not just "we had monitoring"?
- Do you alert on **user impact** (order success rate dropped) and not only on noise (one console error)?
- Do you act on the data: dashboards that drive fixes, not dashboards nobody opens.

## Key points

- Three pillars: **logs** (what happened), **metrics** (numbers over time), **traces** (one request across services). On the frontend we add **error tracking** and **RUM**.
- **Error tracking** (Sentry is the common choice): catches JS exceptions with stack trace, browser, release version, and user breadcrumbs (the clicks before the error).
- **RUM (Real User Monitoring)** (Datadog RUM, Sentry Performance, or your own beacon): Core Web Vitals ([LCP, INP, CLS](https://web.dev/articles/vitals)) from real devices, sliced by device, network, and release.
- **Journey metrics** (funnels): for each critical flow, track step events and the success rate, for example "order form opened -> order submitted -> order confirmed".
- **Alerts** on symptoms users feel (success rate, error rate, p75 latency), routed to on-call with a **runbook** (a short "what to check, what to do" page).

## What I log and monitor on a trading / payments frontend

| Area | What | Example tool |
|---|---|---|
| JS errors | Uncaught errors, unhandled promise rejections, framework error boundaries | Sentry |
| API health | Status codes, latency, timeouts per endpoint, seen from the client | Sentry / Datadog RUM |
| Performance | LCP, INP, CLS at p75; JS bundle size; long tasks | web-vitals library -> Datadog / Grafana |
| Real-time | WebSocket disconnects, reconnect count, message lag (time from server tick to render) | Custom metrics -> Grafana |
| Journeys | Funnel step events and success rate for login, place order, add funds, checkout | Analytics + Datadog / Grafana |
| Releases | Release version tagged on every event, so a spike maps to a deploy | Sentry releases |

What I do NOT log: passwords, OTPs, full card numbers, tokens, PAN or other personal data. Scrub them before sending (see the [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)).

## Example: capturing errors and Web Vitals

```ts
// src/hooks.client.ts (SvelteKit): runs for unexpected errors during client-side load/render
import type { HandleClientError } from '@sveltejs/kit';
import * as Sentry from '@sentry/sveltekit';

export const handleError: HandleClientError = ({ error, event, status, message }) => {
  const errorId = crypto.randomUUID();
  Sentry.captureException(error, { extra: { errorId, route: event.route.id, status } });
  // safe message for the user; the errorId helps support find the event
  return { message: 'Something went wrong', errorId };
};
```

```ts
// src/lib/vitals.ts: send Core Web Vitals from real users
import { onLCP, onINP, onCLS } from 'web-vitals';

// __APP_VERSION__ is a build-time constant injected with Vite's `define` option
function send(metric: { name: string; value: number; id: string; rating: string }) {
  const body = JSON.stringify({ ...metric, page: location.pathname, release: __APP_VERSION__ });
  // sendBeacon still delivers when the page is closing
  navigator.sendBeacon('/rum', body);
}

onLCP(send);
onINP(send);
onCLS(send);
```

```ts
// a journey metric: measure "submit order" success and time
async function submitOrder(order: Order) {
  const start = performance.now();
  track('order_submit_clicked', { type: order.type });
  try {
    const res = await api.placeOrder(order);
    track('order_submit_success', { ms: Math.round(performance.now() - start) });
    return res;
  } catch (err) {
    track('order_submit_failed', { reason: classify(err) }); // timeout | 4xx | 5xx | network
    throw err;
  }
}
```

## Alerts and dashboards

- **Alert on symptoms, with thresholds and a time window.** Example: "order success rate below [fill in: 98]% for 5 minutes" or "JS error rate 2x the 7-day baseline after a release".
- **Every alert links to a runbook and a dashboard.** If nobody would act on an alert, delete it (alert fatigue).
- **One dashboard per journey:** top row is success rate and volume, then latency p50/p75/p95, then errors by type, then split by app version, device, and network.
- Use **percentiles** (p75, p95), not averages. An average hides the slow users.

## When to use it

From day one of a feature: decide the success metric and the events in the design doc, build the dashboard before launch, and watch it during the staged rollout.

## Fill-in template

```text
Tools I used: [fill in: e.g. Sentry, Datadog, Grafana, in-house analytics]
What I tracked: [fill in: 2-3 errors/metrics/journeys]
An alert I set up or improved: [fill in: condition, window, who got paged]
A time monitoring caught something before users reported it: [fill in]
A dashboard I built and what decision it drove: [fill in]
```

## Example answer (generic, for illustration only)

> EXAMPLE, not a real story. Replace with your own.

"On checkout I tracked three things: Sentry for JS errors tagged with the release, Web Vitals from real users, and a funnel from 'payment page opened' to 'payment success'. We had an alert if success rate dropped more than [X] points for 10 minutes. Once, right after a release, the alert fired; the Sentry breakdown showed the errors only on one old Android WebView version because of an unsupported JS method. We turned the flag off in minutes and shipped a polyfill."

## Likely questions

### What do you monitor on the frontend?
Errors (Sentry), performance from real users (LCP, INP, CLS at p75), API latency and failures seen from the client, WebSocket health for live prices, and success rates of key journeys like placing an order. Everything tagged with release, device, and network.

### How do you know a release broke something?
Errors and metrics are tagged with the release version, so I compare new vs old version side by side. During staged rollout I watch the error rate and journey success rate at each step, and alerts compare against a baseline.

### How do you avoid alert fatigue?
Alert only on user-facing symptoms, use time windows so one blip does not page anyone, group duplicate errors, and review alerts monthly: if one fired and nobody needed to act, change or remove it.

### What is the difference between monitoring and observability?
Monitoring tells you something is wrong (a known check failed). Observability lets you ask new questions about why, because you have rich enough data: tags, traces, breadcrumbs. In practice I mean both.

### How would you measure the latency of live price updates?
Have the server include a timestamp in each tick, and on the client record the difference between that timestamp and when the value is painted, then report p75/p95. Clock skew between server and device adds noise, so I compare trends, not absolute numbers.

## Common mistakes

- Logging personal or payment data.
- Only lab numbers (Lighthouse) and no field data (RUM).
- Averages instead of percentiles.
- Alerts without an owner or runbook.
- No source maps uploaded, so Sentry stack traces are unreadable minified code.

## Resources

- [web.dev: Web Vitals](https://web.dev/articles/vitals) - the core user-experience metrics to track
- [MDN: Navigator.sendBeacon()](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon) - reliable way to send metrics as the page unloads
- [SvelteKit: Hooks (handleError)](https://svelte.dev/docs/kit/hooks) - central place to catch unexpected errors
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) - what not to log
