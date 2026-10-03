# Timers

> **In one line:** `setTimeout` runs a function once after *at least* a delay, `setInterval` runs it repeatedly, and neither is exact: the delay is a minimum, they drift, and browsers slow them down in background tabs.

## Key points
- The delay is a **minimum, not a promise**. The callback is a task (macrotask). It runs only after the delay *and* after the call stack and all microtasks are empty. A long-running script delays every timer.
- **Clearing:** both return an id. `clearTimeout(id)` / `clearInterval(id)` cancel them. Always clear in cleanup (component destroy, route change), or you leak work and memory.
- **Nested minimum:** after 5 levels of nested timers, browsers clamp the delay to at least **4ms** (HTML spec).
- **Background tabs:** browsers throttle timers in hidden tabs to at most about **once per second**, and Chrome can throttle further (about once per minute) for pages hidden a long time ("intensive throttling"). Never rely on timers for accurate time in the background.
- **Drift:** `setInterval(fn, 1000)` does not mean "exactly every second". Delays add up, and if `fn` is slow, runs can bunch up or be skipped.

## Example
Recursive `setTimeout` vs `setInterval`:

```js
// setInterval: fires every 1000ms whether or not the last poll has finished.
// If the API takes 1500ms, requests overlap.
const id = setInterval(pollQuotes, 1000);
// later
clearInterval(id);

// Recursive setTimeout: schedule the next run only after this one finishes.
// Guarantees a gap between runs and never overlaps.
let timer;
async function loop() {
  try {
    await pollQuotes();
  } finally {
    timer = setTimeout(loop, 1000);
  }
}
loop();
// later
clearTimeout(timer);
```

Drift-corrected clock (for a countdown such as "order expires in 30s"):

```js
function startCountdown(endAt, onTick) {
  let timer;
  function tick() {
    const left = Math.max(0, endAt - Date.now());   // use real time, not a counter
    onTick(Math.ceil(left / 1000));
    if (left > 0) timer = setTimeout(tick, 1000 - (Date.now() % 1000)); // align to next second
  }
  tick();
  return () => clearTimeout(timer);                  // cleanup function
}
```

Svelte 5 cleanup: return a function from `$effect`.

```svelte
<script>
  let now = $state(new Date());
  $effect(() => {
    const id = setInterval(() => (now = new Date()), 1000);
    return () => clearInterval(id);   // runs when the component is destroyed
  });
</script>

<p>Market time: {now.toLocaleTimeString()}</p>
```

Delay is a minimum, output order:

```js
setTimeout(() => console.log('timeout 0'), 0);
Promise.resolve().then(() => console.log('microtask'));
console.log('sync');
// sync        -> runs on the current call stack
// microtask   -> microtasks run before any timer
// timeout 0   -> timer task runs last, even with delay 0
```

## When to use it
- Recursive `setTimeout` for **polling** an order status or fallback prices when WebSocket is down. Add backoff if it errors.
- `setTimeout` for **debounce** (search a symbol after the user stops typing) and toasts that auto-hide.
- For countdowns and session timeouts, compute from `Date.now()` (or `performance.now()`) each tick; do not count ticks.
- For animations, use [`requestAnimationFrame`](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame), not timers.

## Likely questions
### `setTimeout` vs `setInterval`, and what is drift?
`setTimeout` runs once; `setInterval` repeats. Neither is exact because the callback waits for the event loop to be free. Over time, small delays add up, so a "1 second" counter falls behind the real clock. Fix by reading the real time each tick and correcting the next delay.

### Why prefer recursive `setTimeout`?
It schedules the next run only after the current one finishes, so runs never overlap and you can change the delay each time (for example backoff on errors). `setInterval` keeps firing on a fixed schedule even if the previous async work is still running.

### How do you clear timers, and what happens if you forget?
Keep the id and call `clearTimeout` / `clearInterval`. In Svelte, return a cleanup from `$effect`; in React, from `useEffect`. If you forget, the timer keeps running after the component is gone, updates dead state, makes extra network calls and leaks memory.

### What is the minimum delay in background tabs?
Browsers throttle hidden tabs so timers fire at most about once per second. Chrome adds intensive throttling for pages hidden for a while (timers grouped to about once per minute in some cases). For live prices, listen to [`visibilitychange`](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event), pause polling when hidden, and refresh data when the tab becomes visible again.

### Does `setTimeout(fn, 0)` run immediately?
No. It runs after the current script and all microtasks (promise callbacks) finish, and maybe after rendering. So `Promise.then` callbacks always run before it.

## Common mistakes
- Passing `setTimeout(fn(), 100)` (calls `fn` now) instead of `setTimeout(fn, 100)`.
- Using a tick counter for elapsed time.
- Forgetting to clear timers on destroy.
- Losing `this` inside a timer callback (use arrow functions).

## Resources
- [MDN: setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout) - delays, the 4ms clamp, background throttling
- [javascript.info: Scheduling](https://javascript.info/settimeout-setinterval) - recursive setTimeout vs setInterval
- [developer.chrome.com: Timer throttling in Chrome 88](https://developer.chrome.com/blog/timer-throttling-in-chrome-88) - intensive throttling in hidden tabs
- [MDN: Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API) - pause work when the tab is hidden
