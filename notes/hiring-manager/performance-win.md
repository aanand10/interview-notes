# Performance work: how a render-time improvement was measured and achieved

> **In one line:** "I first measured a clear baseline with the same metric, device and data, found the real bottleneck in a profile, fixed the top causes one at a time, and re-measured the same way to get the [fill in: X]% number."

## What the interviewer is really checking

- **Measurement honesty:** what exactly is "render time"? How, where, on what device, how many runs, which percentile?
- **Debugging method:** did you profile, or guess?
- **Technique depth:** do you know why each fix worked (fewer DOM nodes, less main-thread work, fewer re-renders)?
- **Lasting impact:** did you stop it from regressing (budgets, monitoring)?

## Key points

- Define the metric precisely. "Render time" could be time from data arrival to the list painted, LCP, or a custom `performance.measure()`. Pick one and say it.
- **Lab vs field:** lab = controlled runs in DevTools or Lighthouse; field = real users (RUM). Strong answers have both. See [web.dev: lab vs field](https://web.dev/articles/lab-and-field-data-differences).
- Report a **median or p75 over many runs**, on a throttled or mid-range device, with the same dataset before and after.
- Typical causes: too many DOM nodes, unnecessary re-renders, long tasks over 50 ms, layout thrashing, big bundles, heavy work on the main thread.
- Typical fixes: virtualise long lists, memoise or derive state, batch DOM reads and writes, move heavy work to a [Web Worker](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers), code-split, debounce input.

## Example

Measuring a custom "render time" with the [Performance API](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API):

```js
// 1. Mark when data arrives and when the UI has painted.
performance.mark('table:data-ready');
renderTable(rows); // your framework update
requestAnimationFrame(() => {
  // rAF runs just before the next paint; a setTimeout queued here runs just after it
  setTimeout(() => {
    performance.mark('table:painted');
    performance.measure('table:render', 'table:data-ready', 'table:painted');
  }, 0);
});

// 2. Collect measures and long tasks (main-thread work over 50 ms).
new PerformanceObserver((list) => {
  for (const e of list.getEntries()) {
    console.log(e.entryType, e.name, Math.round(e.duration), 'ms');
    // in production: send to your RUM/analytics endpoint instead
  }
}).observe({ entryTypes: ['measure', 'longtask'] });
```

Then run it, for example, 10 times on a 4x CPU throttled profile before and after, and compare medians.

## When to use it

Dashboards and tables with many rows (a watchlist with hundreds of live-updating symbols, an order history), maps or charts with large datasets, or a slow first load on mid-range phones.

## Answer framework

1. **Problem:** what users felt (janky scroll, slow table, frozen input) and who reported it.
2. **Constraints:** device range, data size, deadline, cannot change the API.
3. **Measure:** metric, tool, device, runs, percentile, baseline number.
4. **Diagnose:** what the profile showed (for example "70% of time in layout from reading `offsetHeight` in a loop").
5. **Options and decision:** which fixes you considered and why you picked these.
6. **Implementation:** the 2 to 3 changes that mattered most.
7. **Impact:** after number, measured the same way. Plus user-facing effect.
8. **Keep it fixed and improve:** performance budget, CI check, RUM alert; what you would do next.

## Fill-in template

```text
What was slow: [fill in: screen/component, what the user felt]
Metric: [fill in: e.g. custom measure "data ready -> painted", or LCP/INP]
How measured: [fill in: tool (DevTools Performance panel / Lighthouse / performance.measure / RUM), device, throttling, number of runs, median or p75]
Baseline: [fill in: X ms]
Bottleneck found: [fill in: what the flame chart showed]
Fixes (in order of impact): 1) [fill in] 2) [fill in] 3) [fill in]
After: [fill in: Y ms] -> [fill in: % from your resume], same method
How we kept it: [fill in: budget, monitoring, review checklist]
Next time: [fill in]
```

## Example answer (generic, for illustration only)

> EXAMPLE, not a real story. Numbers are made up.

"A data table with about 5,000 rows took around 1.2 seconds to render after filtering and the page froze. I defined render time as a custom `performance.measure` from 'data ready' to the next paint, and measured it 10 times on a 4x CPU throttled profile, using the median. The baseline was about 1,200 ms. The Performance panel showed two problems: we rendered every row, and each row read its width in a loop, causing forced layouts. I added list virtualisation so only about 40 visible rows were in the DOM, batched the DOM reads before writes, and memoised the filtered list so typing did not recompute it. The median went to about 700 ms, roughly 40% faster, and long tasks during filtering dropped from four to zero. We added the measure to our RUM so a regression would show on a dashboard."

## Likely questions

### How exactly did you measure the improvement?
Name the metric, tool, device, throttling, number of runs, and percentile. Say the before and after used the same data and the same method. If you also had field data, mention the p75 change for real users. Being precise here is the main thing they are testing.

### How did you find the bottleneck?
Record in the [Chrome DevTools Performance panel](https://developer.chrome.com/docs/devtools/performance), look for long tasks (red corners), then the flame chart to see which functions take time (scripting, style, layout, paint). Check for forced reflows and repeated renders. Confirm the guess by changing one thing and re-measuring.

### Why did virtualisation help?
The browser only creates, styles and lays out the rows you can see, so DOM size goes from thousands of nodes to tens. Less layout and paint work per update. Trade-offs: harder for find-in-page, screen readers and variable row heights, so you handle those deliberately. A CSS-only alternative for some cases is [`content-visibility: auto`](https://web.dev/articles/content-visibility).

### What is layout thrashing?
Reading a layout value (like `offsetHeight`) right after writing styles forces the browser to recalculate layout synchronously. Doing it in a loop repeats that many times. Fix: read everything first, then write, or use `requestAnimationFrame`. See [web.dev: layout thrashing](https://web.dev/articles/avoid-large-complex-layouts-and-layout-thrashing).

### How did you make sure it did not regress?
A performance budget (bundle size, a max for the custom measure) checked in CI, Lighthouse CI on key pages, and RUM dashboards with an alert on p75. Also a code review checklist item for large lists.

### How does this apply to a live trading screen?
Prices tick many times a second. Batch updates per animation frame, only update changed cells, virtualise the watchlist, and keep heavy calculations off the main thread. Watch [INP](https://web.dev/articles/inp) so taps on Buy or Sell still feel instant.

## Common mistakes

- Quoting "40% faster" with no metric, device or method.
- Measuring once on a fast laptop.
- Comparing before and after with different data or network.
- Listing ten optimisations without saying which one mattered.
- Forgetting the trade-offs of the fix (accessibility of virtual lists, memory of caches).

## Resources

- [Chrome DevTools: Performance panel](https://developer.chrome.com/docs/devtools/performance) - profiling and flame charts
- [MDN: Performance API](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API) - `mark`, `measure`, observers
- [web.dev: Optimize long tasks](https://web.dev/articles/optimize-long-tasks) - breaking up main-thread work
- [web.dev: Lab and field data](https://web.dev/articles/lab-and-field-data-differences) - why both matter
