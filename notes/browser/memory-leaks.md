# Memory leaks

> **In one line:** A memory leak in the browser is memory you no longer need but that is still reachable, so the garbage collector cannot free it; the usual causes are listeners and timers you never removed, detached DOM nodes, closures holding big data, caches that only grow, and accidental globals, and you find them with DevTools heap snapshots and the allocation timeline.

## Key points
- GC frees only **unreachable** memory (see the garbage collection note). A leak is always "something still holds a reference".
- **The usual suspects:**
  - Event listeners on `window`, `document` or a WebSocket that are never removed.
  - `setInterval` / `setTimeout` that are never cleared, and subscriptions that are never closed.
  - **Detached DOM nodes:** removed from the page but still referenced from JS (an array, a Map, a closure).
  - **Closures** that capture a large object even though they only need one field.
  - **Caches and arrays that only grow:** a Map with no size limit, or a tick history array with no cap.
  - **Accidental globals:** assigning to an undeclared variable in sloppy mode, or `window.foo = bigThing`.
- In single-page apps (SPAs) the page never reloads, so leaks add up every time you navigate between routes. Always clean up on unmount.
- **How to find them:** watch memory over time (Performance panel with "Memory" ticked, or the Performance monitor), then take heap snapshots, compare them, and look at the retainers (the chain of references keeping an object alive).

## Example
A live price widget that leaks in four ways, then the fixed Svelte 5 version.

```js
// LEAKY vanilla component
const history = [];                         // grows forever
export function mountTicker(el, socket) {
  const big = new Array(100_000).fill(0);   // large data captured by the closure below

  function onMessage(e) {
    const tick = JSON.parse(e.data);
    history.push(tick);                     // (1) unbounded array
    el.textContent = `${tick.symbol} ${tick.price} ${big.length}`;
  }
  socket.addEventListener('message', onMessage); // (2) listener never removed
  setInterval(() => el.classList.toggle('blink'), 1000); // (3) timer never cleared
  window.lastTickerEl = el;                 // (4) global keeps a detached node alive
}
```

```svelte
<!-- FIXED: Svelte 5. The $effect cleanup runs when the component is destroyed
     (and before the effect re-runs if `socket` changes) -->
<script>
  let { socket } = $props();
  const MAX = 500;
  let ticks = $state([]);

  $effect(() => {
    const controller = new AbortController();
    socket.addEventListener('message', (e) => {
      ticks.push(JSON.parse(e.data));
      if (ticks.length > MAX) ticks.shift();          // bounded history
    }, { signal: controller.signal });

    const id = setInterval(() => { /* blink */ }, 1000);

    return () => {
      controller.abort();                             // removes every listener using this signal
      clearInterval(id);
    };
  });
</script>

<p>{ticks.at(-1)?.symbol} {ticks.at(-1)?.price}</p>
```

A growing cache, measured in Node (`node --expose-gc`):

```js
const cache = new Map();
function getQuote(symbol, i) {
  const key = `${symbol}-${i}`;           // unique key every time -> never reused
  if (!cache.has(key)) cache.set(key, { symbol, data: new Array(1000).fill(i) });
  return cache.get(key);
}
for (let i = 0; i < 20000; i++) getQuote('INFY', i);
globalThis.gc();
console.log('Map size:', cache.size, 'heapUsed MB:', Math.round(process.memoryUsage().heapUsed / 1e6));
```

```bash
Map size: 20000 heapUsed MB: 167
```
- Even after a forced GC, all 20,000 entries stay, because the Map still references them. The fix is a size limit (LRU), a TTL (time to live), or a `WeakMap` when the keys are objects.

## When to use it
- A trading terminal that stays open all day, getting thousands of WebSocket ticks per minute. A small leak per tick turns into hundreds of MB and the tab crashes.
- Route changes in SvelteKit: chart libraries (they need `chart.destroy()`), WebSocket subscriptions and `ResizeObserver`s must be cleaned up in the `$effect` teardown, or in an attachment's returned cleanup function.

## Likely questions

### What are the common causes of memory leaks in frontend apps?
Listeners added to long-lived objects (`window`, `document`, sockets) and never removed. Timers and intervals never cleared. Detached DOM nodes still referenced from JS. Closures that keep big objects alive. Caches and arrays with no limit. Accidental globals. In frameworks, it is usually missing cleanup when a component unmounts.

### What is a detached DOM node?
It is an element removed from the document but still referenced from JavaScript, for example kept in an array or in a closure. It cannot be collected, and it keeps its whole subtree alive too. In a heap snapshot, type "Detached" in the class filter to find them, then check the retainers panel to see which variable is holding them.

### How do closures cause leaks?
A closure keeps its outer variables alive as long as the closure itself is reachable. If a long-lived listener closes over a huge array, that array lives as long as the listener does. Fix it by removing the listener, or by copying only the small value you need into a local variable before you create the closure.

### How do you find a memory leak with DevTools?
1. Reproduce the suspected action (for example, open and close the order modal 10 times).
2. **Heap snapshots** (Memory panel): take one snapshot before, do the action several times, force GC with the trash-can icon, take another snapshot, and use the "Comparison" view. Objects whose count keeps growing are suspects. Click one and read its **Retainers** to see who holds it.
3. **Allocation instrumentation on timeline:** records allocations over time. Blue bars that stay blue (not grey) are objects that were never freed.
4. The **Performance panel** with Memory ticked shows a JS heap line that only goes up, like a saw with a rising floor, plus the count of DOM nodes and listeners.

### How do you prevent leaks with event listeners?
Always pair `addEventListener` with `removeEventListener`, using the same function reference. Or pass `{ signal }` from an `AbortController` and call `abort()` once to remove them all. Use `{ once: true }` for one-shot events. Use event delegation so there are fewer listeners. In Svelte, `onclick={...}` on an element is cleaned up automatically. Manual listeners belong in an `$effect` that returns a cleanup function.

### When would you use WeakMap for a cache?
When the key is an object (a DOM node, a model object) and the cached data should go away with it. A `WeakMap` does not keep its keys alive. For string keys like a stock symbol you cannot use a WeakMap, so cap the size with an LRU or add a TTL.

## Common mistakes
- `removeEventListener` with a new arrow function. It does not match the original, so nothing is removed.
- Clearing a timer in one place but starting a second one somewhere else (the effect re-runs and starts another interval).
- Keeping full order or tick history in reactive state "for later" with no cap.
- Mistaking a normal sawtooth memory graph (it goes up, then GC drops it) for a leak. A leak is when the low points keep rising.
- Leaving `console.log(bigObject)` in code. DevTools keeps logged objects alive while the console is open.

## Resources
- [Chrome DevTools: Fix memory problems](https://developer.chrome.com/docs/devtools/memory-problems) - the full workflow for finding leaks
- [Chrome DevTools: Heap snapshots](https://developer.chrome.com/docs/devtools/memory-problems/heap-snapshots) - comparison view, retainers, detached nodes
- [Chrome DevTools: Allocation timeline](https://developer.chrome.com/docs/devtools/memory-problems/allocation-profiler) - finding allocations that are never freed
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) - removing listeners and cancelling fetches together
- [svelte.dev: $effect](https://svelte.dev/docs/svelte/$effect) - teardown functions in Svelte 5
