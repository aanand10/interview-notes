# Comlink

> **In one line:** Comlink is a tiny library from the Chrome team that turns `postMessage` into RPC (remote procedure call), so I can call a function that lives in a Web Worker as if it were a normal `async` function, `await api.calculate(data)`, instead of writing message handlers by hand.

## Key points
- **Problem it solves:** raw workers only have `postMessage` and `onmessage`. For many functions you end up writing a `switch` on message types, matching request ids to replies, and wiring up errors. That is boilerplate.
- **How it works:** in the worker you `Comlink.expose(obj)`. On the page you `Comlink.wrap(worker)`, which returns a [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy). Every call or property read on the proxy becomes a message; Comlink sends it, waits for the reply and resolves a Promise.
- **Everything becomes async:** even a sync function in the worker returns a Promise on the page, because the answer has to travel back across threads.
- **Errors travel too:** if the worker function throws, the Promise on the page rejects.
- **Extras:** `Comlink.transfer(value, [buffer])` to transfer instead of copy, and `Comlink.proxy(fn)` to pass a callback (for example a progress handler) into the worker. Works with any `postMessage` endpoint: workers, iframes, `MessagePort`.

## Example
Expose a worker function that computes portfolio stats.

```js
// stats.worker.js
import * as Comlink from 'comlink';

const api = {
  // plain function: Comlink makes it callable from the page
  portfolioStats(positions) {
    let value = 0, pnl = 0;
    for (const p of positions) {
      value += p.qty * p.ltp;                 // ltp = last traded price
      pnl += p.qty * (p.ltp - p.avgPrice);
    }
    return { value, pnl };
  },

  // takes a callback that lives on the main thread
  async backtest(candles, onProgress) {
    for (let i = 0; i < candles.length; i++) {
      // ...heavy work...
      if (i % 1000 === 0) await onProgress(i / candles.length);
    }
    return 'done';
  },
};

Comlink.expose(api);
```

```svelte
<!-- Portfolio.svelte -->
<script>
  import * as Comlink from 'comlink';
  let { positions } = $props();
  let stats = $state(null);
  let progress = $state(0);

  const worker = new Worker(new URL('./stats.worker.js', import.meta.url), { type: 'module' });
  const api = Comlink.wrap(worker);          // proxy to the worker's api object

  $effect(() => {
    // Looks like a local call, but runs in the worker and returns a Promise.
    // $state.snapshot gives a plain object (a $state proxy cannot be structured-cloned).
    api.portfolioStats($state.snapshot(positions)).then((s) => (stats = s));
  });

  async function runBacktest(candles) {
    await api.backtest(candles, Comlink.proxy((p) => (progress = p)));
  }

  $effect(() => () => worker.terminate());
</script>

{#if stats}<p>Value: {stats.value.toFixed(2)} | P&L: {stats.pnl.toFixed(2)}</p>{/if}
```

Without Comlink the page side would look like this:

```js
let id = 0;
const pending = new Map();
worker.onmessage = ({ data }) => {
  const { id, result, error } = data;
  error ? pending.get(id).reject(error) : pending.get(id).resolve(result);
  pending.delete(id);
};
function call(method, ...args) {
  return new Promise((resolve, reject) => {
    pending.set(++id, { resolve, reject });
    worker.postMessage({ id, method, args });
  });
}
```

## When to use it
- Any app with more than one or two worker functions: chart indicator calculations, CSV export, search over instruments. It keeps the worker code readable and testable as plain functions.
- Skip it for a single fire-and-forget message, where raw `postMessage` is simple enough.

## Likely questions
### What does Comlink do?
It gives RPC over `postMessage`. The worker exposes an object, and the page gets a proxy for it. When I call a method on the proxy, Comlink serialises the method name and arguments into a message, the worker runs the real function, and the result comes back as a resolved Promise. So I write `await api.fn(x)` instead of managing message ids and handlers.

### Show how to expose a worker function.
In the worker: `Comlink.expose({ add: (a, b) => a + b })`. On the page: `const api = Comlink.wrap(new Worker(url, { type: 'module' }))` and then `await api.add(2, 3)` gives `5`. Note the `await`: every call crosses a thread, so it is always async.

### Does Comlink make data sharing free?
No. Arguments are still copied with structured clone, the same as `postMessage`. For big buffers I wrap them with `Comlink.transfer(data, [data.buffer])` to move them instead. Functions cannot be cloned, so callbacks must be wrapped in `Comlink.proxy(fn)`.

### How big is it and what are the trade-offs?
It is small (about 1 KB gzipped). The trade-off is a little hidden magic: it is easy to forget that each call is a message, and chatty code (thousands of tiny calls) is slow. Batch work into fewer, bigger calls.

## Common mistakes
- Forgetting `await`, then logging a Promise instead of the result.
- Passing a Svelte `$state` proxy or a function directly; snapshot the data and use `Comlink.proxy` for callbacks.
- Calling the worker in a tight loop instead of sending the whole batch once.

## Resources
- [Comlink on GitHub](https://github.com/GoogleChromeLabs/comlink) - official README with the API (`expose`, `wrap`, `transfer`, `proxy`)
- [web.dev: Use web workers to run JavaScript off the browser's main thread](https://web.dev/articles/off-main-thread) - uses Comlink in its examples
- [MDN: Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) - the JS feature Comlink is built on
- [MDN: Worker.postMessage()](https://developer.mozilla.org/en-US/docs/Web/API/Worker/postMessage) - the raw API underneath
