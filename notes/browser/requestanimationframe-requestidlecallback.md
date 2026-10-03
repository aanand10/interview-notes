# requestAnimationFrame / requestIdleCallback

> **In one line:** `requestAnimationFrame` runs your code right before the next paint, so it is for visual updates and animations; `requestIdleCallback` runs your code when the browser has spare time, so it is for low-priority background work.

## Key points
- [`requestAnimationFrame(cb)`](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) (rAF): called once before the next repaint, usually 60 times a second (more on 120Hz screens). `cb` gets a high-precision timestamp. It is paused in background tabs, which saves battery.
- Use rAF for anything the user sees moving: JS animations, canvas charts, batching DOM writes from fast events (scroll, WebSocket ticks) into one update per frame.
- [`requestIdleCallback(cb, { timeout })`](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestIdleCallback) (rIC): called when the frame has finished its work and there is idle time left. `cb` gets a `deadline` object; `deadline.timeRemaining()` tells you how many ms are left (at most 50).
- Use rIC for work that can wait: sending analytics, prefetching, warming a cache, indexing data for search. Do not touch the DOM in it, because the frame may already be laid out.
- Safari support for rIC has historically been missing or behind a flag (check [caniuse](https://caniuse.com/requestidlecallback)), so feature-detect it and fall back to `setTimeout`. `timeout` forces the callback to run after that many ms even if the browser never gets idle.

## Example
Smooth price-flash animation, and batching many WebSocket ticks into one paint:

```js
// Many ticks per frame, but we only want to touch the DOM once per frame
const pending = new Map();
let scheduled = false;

socket.onmessage = (msg) => {
  const { symbol, price } = JSON.parse(msg.data);
  pending.set(symbol, price);            // keep only the latest price
  if (!scheduled) {
    scheduled = true;
    requestAnimationFrame(flush);        // run once, right before paint
  }
};

function flush() {
  for (const [symbol, price] of pending) {
    document.getElementById(symbol).textContent = price.toFixed(2);
  }
  pending.clear();
  scheduled = false;
}
```

A simple animation loop that uses the timestamp, so speed does not depend on frame rate:

```js
let start;
function step(ts) {
  start ??= ts;
  const progress = Math.min((ts - start) / 300, 1);   // 300ms animation
  bar.style.transform = `scaleX(${progress})`;
  if (progress < 1) requestAnimationFrame(step);
}
requestAnimationFrame(step);
```

Background work in small chunks with rIC:

```js
const ric = window.requestIdleCallback ?? ((cb) => setTimeout(() => cb({ timeRemaining: () => 5, didTimeout: false }), 1));

const queue = [...analyticsEvents];
function sendBatch(deadline) {
  while (deadline.timeRemaining() > 1 && queue.length) {
    navigator.sendBeacon('/analytics', JSON.stringify(queue.shift()));
  }
  if (queue.length) ric(sendBatch, { timeout: 2000 });
}
ric(sendBatch, { timeout: 2000 });
```

## When to use it
- rAF: live price tickers, sparkline and candlestick redraws, smooth scroll effects, drag handles.
- rIC: analytics, prefetching the next page's data, building a search index of all symbols after first load.
- Prefer CSS transitions and animations for simple effects (`transform`, `opacity`); they can run on the compositor without JS.

## Likely questions
### When would you use requestAnimationFrame instead of setTimeout?
`setTimeout(fn, 16)` is not synced with the screen. It can fire mid-frame or twice in one frame, causing jank or wasted work, and it keeps running in background tabs. rAF is called exactly once per frame just before paint, matches the screen refresh rate, and is paused when the tab is hidden. So for anything visual, rAF.

### When would you use requestIdleCallback?
For non-urgent work that should not slow down input or animation, like analytics or prefetching. The browser runs it only in idle periods and tells you how much time you have. Keep each chunk small, check `timeRemaining()`, and reschedule the rest. Add a `timeout` if the work must happen eventually.

### Can you update the DOM inside requestIdleCallback?
You should not. The idle period comes after layout and paint, so a DOM change there forces new style and layout work and can push into the next frame. If idle work produces a DOM change, schedule that part with rAF.

### Does rAF run if the tab is in the background?
No, browsers pause it in hidden tabs. If you need prices to stay updated, keep the data in state and let rAF paint when the user comes back. For timing-critical background logic, do not rely on rAF.

### What is the difference between rAF and a microtask?
A microtask (`queueMicrotask`, promise callback) runs right after the current script, before the browser can render. rAF waits for the next frame. Microtasks are for "finish this logic now", rAF is for "draw this on the next frame".

## Common mistakes
- Assuming 60fps and moving a fixed number of pixels per frame; on a 120Hz screen the animation runs twice as fast. Use the timestamp.
- Calling `requestAnimationFrame` many times per event without a flag, which queues many callbacks in one frame.
- Using rIC for work that must finish soon (like saving an order draft); it might wait a long time on a busy page.

## Resources
- [MDN: requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) - timing and background behaviour
- [MDN: requestIdleCallback](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestIdleCallback) - deadline and timeout options
- [javascript.info: Event loop](https://javascript.info/event-loop) - where rendering fits between tasks and microtasks
- [web.dev: Optimize long tasks](https://web.dev/articles/optimize-long-tasks) - yielding to keep the page responsive
