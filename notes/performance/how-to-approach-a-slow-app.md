# How to approach a slow app

> **In one line:** I never guess. I measure first, find which part is slow (network, JavaScript, or rendering), fix the biggest bottleneck, and then measure again to prove the fix worked.

## Key points
- **Measure first.** Use [Lighthouse](https://developer.chrome.com/docs/lighthouse/overview) for a quick lab report, the Chrome DevTools [Performance panel](https://developer.chrome.com/docs/devtools/performance) for a detailed timeline, and [Core Web Vitals](https://web.dev/articles/vitals) (LCP, INP, CLS) for what real users feel.
- **Define "slow".** Slow to load? Slow to respond to clicks? Janky scrolling? Each points to a different cause.
- **Find the bottleneck category:** network (big files, many requests, slow server), JavaScript (long tasks blocking the main thread), or rendering (too many DOM updates, layout thrashing, heavy paint).
- **Fix the biggest win first**, one change at a time, so you know what helped.
- **Measure again and keep it fixed.** Compare before/after numbers, then add monitoring or a performance budget so it does not regress.

## Example
A simple workflow I follow, written as a checklist:

```bash
# 1. Reproduce on a realistic setup
#    DevTools > Performance > CPU: 4x slowdown, Network: Fast 4G
# 2. Lab check
npx lighthouse https://app.example.com/portfolio --view
# 3. Record a trace of the slow action (e.g. opening the watchlist)
#    DevTools > Performance > Record > do the action > Stop
# 4. Read the trace:
#    - Network track: what loads, how big, what blocks first paint
#    - Main track: long tasks (red corner = over 50 ms), which function
#    - Purple (Rendering/Layout) and green (Paint) blocks
# 5. Fix one thing, re-record, compare numbers
```

And a tiny code-level way to measure one specific operation:

```js
// Measure a suspect function with the User Timing API
performance.mark('sort-start');
const sorted = rows.toSorted((a, b) => b.change - a.change);
performance.mark('sort-end');
performance.measure('sort-watchlist', 'sort-start', 'sort-end');

// The measure shows up in the DevTools Performance "Timings" track too
const [m] = performance.getEntriesByName('sort-watchlist');
console.log(`sort took ${m.duration.toFixed(1)} ms`);
```

## When to use it
Any time someone says "the app feels slow". For a trading app, the common complaints are: the dashboard takes too long to show (load), the order form lags when typing (interaction), or the watchlist stutters during live price updates (rendering). E.g. in my last project I followed exactly this loop: profiled first, found the expensive re-renders, fixed them, and cut render time by about 40% and page load by about 300 ms. The numbers are what made the change convincing to the team.

## Likely questions
### A user says our app is slow. What do you do?
First I ask what "slow" means: slow first load, slow after a click, or janky scrolling. Then I reproduce it, ideally with CPU and network throttling because our users are often on mid-range phones. I run Lighthouse for a quick overview and record a Performance panel trace of the exact slow action. I also check real-user data (Web Vitals from production) to see if it is widespread or one device type. Once I know the bottleneck, I fix the biggest one and measure again.

### How do you tell if the problem is network, JavaScript or rendering?
In the Performance panel, the Network track shows if we are waiting on big or late files, or a slow server response (high TTFB, time to first byte). The Main thread track shows yellow scripting blocks; long tasks over 50 ms mean JavaScript is blocking. Purple blocks are style and layout, green is paint; if those dominate, it is a rendering problem, for example too many DOM nodes or reading layout in a loop. The bottom-up and call tree views tell me which function is to blame.

### What is the difference between lab data and field data?
Lab data comes from a controlled test like Lighthouse on one machine. It is repeatable and good for debugging. Field data, also called RUM (real user monitoring), comes from real users on real devices and networks, for example via the `web-vitals` library or the [Chrome UX Report](https://developer.chrome.com/docs/crux). Field data tells you what users actually feel; lab data helps you find why. They can disagree, and field data wins when deciding priorities. See [lab vs field data](https://web.dev/articles/lab-and-field-data-differences).

### What tools do you use?
Lighthouse for a score and a list of opportunities. The Performance panel for detailed traces. The Network panel for sizes, caching and waterfalls. The [Coverage tab](https://developer.chrome.com/docs/devtools/coverage) to find unused JavaScript and CSS. A bundle analyzer to see what is inside our JS. And the `web-vitals` library in production for real-user numbers.

### Tell me about a performance improvement you made.
I use a STAR-style answer: the problem, how I measured, what I changed, and the result in numbers. For example: "Our dashboard was slow on lower-end laptops. I profiled it and found that components were re-rendering on every data change. I fixed the update pattern and split the heavy code, and measured a roughly 40% cut in render time and about 300 ms faster page load." Always end with the measured result.

### How do you make sure it does not get slow again?
I add guards: a performance budget in CI (for example Lighthouse CI that fails the build if bundle size or LCP crosses a limit), and production monitoring of Web Vitals with alerts. Performance regresses quietly, one dependency at a time, so automation matters more than one-off fixes.

## Common mistakes
- Optimising before measuring, for example adding memoization everywhere without proof it helps.
- Testing only on a fast MacBook with fast Wi-Fi. Always throttle.
- Changing five things at once, so you cannot tell which change helped.
- Trusting one Lighthouse run. Scores vary; run it a few times and look at the median.
- Forgetting to re-measure and report the result in numbers.

## Resources
- [web.dev: Learn Performance](https://web.dev/learn/performance) - a full free course on web performance
- [Chrome DevTools: Analyze runtime performance](https://developer.chrome.com/docs/devtools/performance) - step-by-step Performance panel tutorial
- [Lighthouse overview](https://developer.chrome.com/docs/lighthouse/overview) - how to run it and read the report
- [web.dev: Why lab and field data differ](https://web.dev/articles/lab-and-field-data-differences) - when to trust which numbers
- [MDN: Web performance](https://developer.mozilla.org/en-US/docs/Web/Performance) - reference hub for all performance topics
