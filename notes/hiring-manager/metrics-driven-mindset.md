# Metrics-driven mindset

> **In one line:** "Before I build, I agree on the metric that defines success and capture a baseline; after release I compare against it, and I let the data decide what to do next, even when it says my idea did not work."

## What the interviewer is really checking

- Do you connect frontend work to **business and user outcomes** (conversion, orders placed, drop-off, support tickets)?
- Do you know **how** a number was measured, and can you defend it?
- Can you tell correlation from causation (did your change cause it, or did a sale event)?
- Will you change your mind when data disagrees with you?

## Key points

- **Pick the metric first:** one primary metric (for example payment success rate) plus **guardrail** metrics that must not get worse (error rate, INP, crash rate).
- **Baseline:** measure before the change, for long enough to cover normal weekly patterns.
- **Compare fairly:** A/B test (two random groups, one gets the change) is best. If not possible, a staged rollout comparing flag-on vs flag-off users at the same time.
- **Use percentiles** for performance (p75, p95) and **rates** for funnels (success / attempts), not raw counts.
- **Segment:** device, network, app version, platform (WebView vs browser). Averages hide the users who suffer most.

## Example: calculating a funnel and comparing two variants

```js
// events per user session, from analytics
const sessions = [
  { variant: 'A', steps: ['open', 'select_method', 'pay_click', 'success'] },
  { variant: 'A', steps: ['open', 'select_method'] },
  { variant: 'B', steps: ['open', 'select_method', 'pay_click', 'success'] },
  { variant: 'B', steps: ['open', 'select_method', 'pay_click', 'success'] },
];

function successRate(variant) {
  const group = sessions.filter((s) => s.variant === variant);
  const opened = group.filter((s) => s.steps.includes('open')).length;
  const paid = group.filter((s) => s.steps.includes('success')).length;
  return opened ? paid / opened : 0;
}

console.log(successRate('A')); // 0.5
console.log(successRate('B')); // 1
// In real life: thousands of sessions, and a significance check before deciding.
```

## Fill-in template

```text
Decision: [fill in: what you decided]
Data used: [fill in: metric, source, tool, segment]
Baseline: [fill in: number + period]
Change: [fill in: what you shipped, A/B or staged]
Result: [fill in: before -> after, guardrails OK?]
How I knew it was my change: [fill in: A/B, flag-on vs off, timing]
What I would measure differently: [fill in]
```

## Example answer (generic, for illustration only)

> EXAMPLE, not a real story. Replace with your own.

"Our dashboard showed checkout drop-off was highest on the payment-method step, and RUM showed p75 INP of [X] ms on low-end Android, mostly from a heavy third-party script loaded upfront. My hypothesis was that slow taps caused drop-off. I lazy-loaded the script only after the user picked that method, and we ran it as a 50/50 test for two weeks. p75 INP went from [X] to [Y] ms, and step completion went up [Z] points, with no rise in errors. Because it was a randomised test over two full weeks, I was confident the change caused it."

## Likely questions

### Tell me about a decision you made based on data.
Hypothesis, data that pointed to it, change, measurement method, result, and next step. Mention a guardrail metric.

### How did you measure success of your feature?
Name the primary metric, the baseline, the comparison method, and the time window. Mention how the number was collected (analytics events, RUM, server logs).

### What if the data showed your feature did not help?
Say it honestly, dig into segments to understand why, then iterate or roll back. Removing a feature that does not help is a good outcome.

### How do you avoid fooling yourself with metrics?
Use control groups, enough data and time, look at guardrails, watch for outside events (sales, market holidays, outages), and decide the success metric before seeing results.

### Which frontend metrics matter for a trading app?
Time to interactive on app open, INP on the order form, price update latency, WebSocket reconnect rate, order placement success rate and time, crash/error rate per app version.

## Common mistakes

- Numbers with no baseline or no way they were measured.
- Taking credit for a jump that happened during a big sale.
- Only vanity metrics (page views) instead of outcomes (orders placed).
- Averages for latency.

## Resources

- [web.dev: User-centric performance metrics](https://web.dev/articles/user-centric-performance-metrics) - choosing metrics that reflect real user experience
- [web.dev: Web Vitals](https://web.dev/articles/vitals) - standard metrics and p75 thresholds
- [MDN: Performance API](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API) - measuring custom timings in the browser
