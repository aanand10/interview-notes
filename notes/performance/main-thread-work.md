# Main thread work

> **In one line:** The browser has one main thread for JavaScript, layout and painting, so any task that runs longer than 50 ms blocks clicks and frames; I measure it, then split it into chunks or move it to a Web Worker.

## Key points
- The **main thread** runs your JS, handles input events, does style/layout/paint. While JS runs, nothing else can happen there.
- A [long task](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceLongTaskTiming) is any task over **50 ms**. Long tasks hurt **INP** (Interaction to Next Paint, the Core Web Vital for responsiveness) and make the page feel frozen.
- **Chunking**: break a big loop into small pieces and *yield* back to the browser between pieces so it can handle input and paint.
- **Web Workers** run JS on a separate thread. Good for pure computation (parsing, sorting, indicators). Workers cannot touch the DOM; they talk with `postMessage`.
- Always measure first: the Chrome DevTools [Performance panel](https://developer.chrome.com/docs/devtools/performance) shows long tasks as grey blocks with a red corner.

## Example
Chunking a big job and yielding between chunks:

```js
// Yield to the browser so it can handle clicks and paint.
function yieldToMain() {
  // scheduler.yield() is newer (Chromium); fall back to setTimeout elsewhere.
  if (globalThis.scheduler?.yield) return scheduler.yield();
  return new Promise((resolve) => setTimeout(resolve, 0));
}

async function processTrades(trades) {
  const CHUNK = 500;
  for (let i = 0; i < trades.length; i += CHUNK) {
    trades.slice(i, i + CHUNK).forEach(computePnL); // small piece of work
    await yieldToMain();                             // let the browser breathe
  }
}
```

Moving heavy math to a Web Worker:

```js
// worker.js
self.onmessage = (e) => {
  const candles = e.data;
  const sma = movingAverage(candles, 20); // heavy calculation, off the main thread
  self.postMessage(sma);
};

// main.js (module-friendly syntax used by Vite / SvelteKit)
const worker = new Worker(new URL('./worker.js', import.meta.url), { type: 'module' });
worker.onmessage = (e) => drawIndicator(e.data);
worker.postMessage(candles);
```

## When to use it
- A trading screen computing indicators (SMA, RSI) on 10k candles: do it in a worker so the order form stays responsive.
- Rendering a 5,000-row portfolio table: chunk the work or, better, virtualise the list (render only visible rows).
- Parsing a large CSV export of transactions: worker.
- e.g. in my last project I found long tasks in the Performance panel, split the work and removed wasted renders, which cut render time by about 40%.

## Likely questions
### What is a long task and why does it matter?
A long task is any main-thread task over 50 ms. During that time the browser cannot respond to a click or paint a frame, so the user sees lag. It directly hurts INP. I find them in the Performance panel or with a `PerformanceObserver` watching `longtask` entries.

### How do you break up a long task?
I split the work into chunks and yield between them with `await scheduler.yield()` where supported, or `setTimeout(0)` as a fallback. Yielding lets the browser run pending input handlers and paint. `requestIdleCallback` is an option for low-priority work like analytics.

### When would you use a Web Worker instead of chunking?
When the work is pure computation and does not need the DOM, and it is big enough that even chunking would make it slow overall. A worker runs truly in parallel on another thread. The cost is copying data through `postMessage` (structured clone), so for very large data I use transferable objects like `ArrayBuffer`.

### What can a Web Worker NOT do?
It cannot access the DOM, `window` or `document`. It communicates only through messages. It can use `fetch`, `WebSocket`, timers and `IndexedDB`.

### How would you detect long tasks in production?
```js
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log('Long task', Math.round(entry.duration), 'ms');
  }
}).observe({ type: 'longtask', buffered: true });
```
Send these to analytics, together with INP from the `web-vitals` library.

## Common mistakes
- Using `async` and thinking the work no longer blocks. An `async` function still runs on the main thread; only the `await` points give control back.
- Sending huge objects to a worker every frame; the copy cost can be worse than the work.
- Optimising without measuring first.

## Resources
- [web.dev: Optimize long tasks](https://web.dev/articles/optimize-long-tasks) - chunking and yielding explained
- [MDN: Using Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers) - full worker guide
- [Chrome DevTools: Performance panel](https://developer.chrome.com/docs/devtools/performance) - how to record and find long tasks
- [web.dev: Interaction to Next Paint](https://web.dev/articles/inp) - the metric long tasks hurt
