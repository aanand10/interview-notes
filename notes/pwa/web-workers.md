# Web Workers

> **In one line:** A Web Worker runs JavaScript on a separate background thread, so heavy work like crunching thousands of price ticks or computing chart indicators does not freeze the UI; the page and the worker talk only by sending messages with `postMessage`.

## Key points
- **Why:** JavaScript on the page runs on one **main thread**. That same thread handles clicks, layout, painting and your code. A 300 ms loop there means 300 ms where buttons do not respond and animations stutter. A [Web Worker](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers) moves that loop to another thread.
- **Messages, not shared variables:** the page calls `worker.postMessage(data)` and listens with `worker.onmessage`. Inside the worker it is `self.postMessage` and `self.onmessage`. Data is copied using the [structured clone algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm) (objects, arrays, Maps, Dates, typed arrays work; functions and DOM nodes do not).
- **No DOM access:** a worker has no `document`, no `window`, and cannot touch elements. It does have `fetch`, `setTimeout`, `WebSocket`, `IndexedDB`, `crypto` and `importScripts` / ES module imports. It sends results back, and the main thread updates the DOM.
- **Transferable objects:** big binary data like an `ArrayBuffer` can be **transferred** instead of copied. Ownership moves to the worker in near-zero time and the sender's copy becomes empty (length 0). Great for large datasets.
- **Types:** a dedicated `Worker` (one page owns it), a [`SharedWorker`](https://developer.mozilla.org/en-US/docs/Web/API/SharedWorker) (shared by tabs of the same origin), and Service Workers (a different thing: a network proxy, see the Service Worker vs Web Worker note).

## Example
Compute a moving average for a chart without blocking the UI.

```js
// sma.worker.js
self.onmessage = (event) => {
  const { prices, period } = event.data;     // prices is a Float64Array
  const out = new Float64Array(prices.length);
  let sum = 0;
  for (let i = 0; i < prices.length; i++) {
    sum += prices[i];
    if (i >= period) sum -= prices[i - period];
    out[i] = i >= period - 1 ? sum / period : NaN;
  }
  // Transfer the result buffer back: no copy
  self.postMessage({ sma: out }, [out.buffer]);
};
```

```svelte
<!-- Chart.svelte -->
<script>
  let { prices } = $props();      // Float64Array of closing prices
  let sma = $state(null);

  $effect(() => {
    // Vite bundles this as a module worker
    const worker = new Worker(new URL('./sma.worker.js', import.meta.url), { type: 'module' });
    worker.onmessage = (e) => (sma = e.data.sma);
    worker.onerror = (e) => console.error('worker failed', e.message);

    const copy = prices.slice();                         // keep our own array usable
    worker.postMessage({ prices: copy, period: 20 }, [copy.buffer]); // transfer
    // copy.length is now 0 here: ownership moved to the worker

    return () => worker.terminate();                     // clean up on unmount
  });
</script>

{#if sma}<p>Last SMA(20): {sma.at(-1).toFixed(2)}</p>{:else}<p>Calculating...</p>{/if}
```

## When to use it
- **Chart calculations:** indicators (SMA, EMA, RSI, Bollinger bands) over years of candles.
- **Data processing:** parsing a big CSV of trade history, sorting or filtering 50,000 rows, decoding a binary market-data feed.
- **Heavy formatting or search:** building a search index for all instruments.
- Not worth it for small, quick work: creating a worker and copying data has its own cost.

## Likely questions
### Why do we need Web Workers?
The browser runs the page's JavaScript, event handling, style, layout and paint on one main thread. If my code takes long, nothing else can run, so the page feels frozen and Interaction to Next Paint gets bad. A worker gives me a second thread for CPU-heavy work. The main thread stays free to respond to clicks and paint frames.

### How do the page and worker communicate?
Only through messages. The page does `worker.postMessage(data)` and gets replies with `worker.onmessage`; the worker uses `self.onmessage` and `self.postMessage`. Data is copied with structured clone, so each side has its own copy and there is no shared state to race on. For request/response patterns I add an id to each message, or use a library like Comlink to make it look like a function call.

### Can a worker access the DOM?
No. There is no `document` or `window` in a worker, because the DOM is not thread-safe and only the main thread may touch it. The worker computes and posts the result; the main thread renders it. If I need to draw from a worker, I can use [`OffscreenCanvas`](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas), which a canvas can hand over to a worker.

### What are transferable objects?
Normally `postMessage` copies the data, which is slow for big buffers. If I pass a second argument, the transfer list, objects like `ArrayBuffer`, `MessagePort`, `ImageBitmap` and `OffscreenCanvas` are moved instead of copied. It is very fast, but the sender loses access: the buffer becomes detached with `byteLength` 0.

```js
const buf = new ArrayBuffer(8 * 1_000_000);
worker.postMessage(buf, [buf]);
console.log(buf.byteLength); // 0, it now belongs to the worker
```

### How do you handle errors and cleanup?
Listen to `worker.onerror` (uncaught errors) and `onmessageerror` (data that could not be cloned). Wrap worker code in try/catch and post back `{ error }` for expected failures. Call `worker.terminate()` when the component unmounts, or the thread keeps running and holding memory.

### What about SharedArrayBuffer?
[`SharedArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer) lets threads share the same memory, with `Atomics` for safe updates. It needs the page to be cross-origin isolated (COOP and COEP headers). Most apps do not need it; message passing is simpler and safer.

## Common mistakes
- Trying to use `document` or `localStorage` inside a worker (neither exists there; use IndexedDB).
- Sending huge objects back and forth on every tick, so the copy cost is bigger than the work saved.
- Using a transferred buffer afterwards and getting an empty array.
- Creating a new worker per message instead of reusing one.
- Forgetting `terminate()` on unmount.

## Resources
- [MDN: Using Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers) - full guide with examples
- [MDN: Transferable objects](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects) - which objects can move and how
- [web.dev: Use web workers to run JavaScript off the browser's main thread](https://web.dev/articles/off-main-thread) - why off-main-thread matters
- [javascript.info: Event loop](https://javascript.info/event-loop) - why one long task blocks the page
