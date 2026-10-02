# Real-time data

> **In one line:** For live prices over a WebSocket, I never update the UI on every message: I buffer ticks, keep only the latest price per symbol, flush to the UI on a fixed rhythm (like every 100 ms or each animation frame), and update only the rows that changed.

## Key points
- **Ticks arrive faster than humans can read.** A busy market can send hundreds of messages per second. Rendering each one wastes CPU, drains battery and makes the page janky.
- **Per-item updates:** store prices by symbol (a `Map` or object keyed by symbol) so one tick touches one row, not the whole list. Svelte's fine-grained reactivity makes this natural.
- **Throttled UI flushes (batching):** collect ticks in a buffer, overwrite older ticks for the same symbol (conflation), then apply the batch every 100 to 250 ms or on `requestAnimationFrame`.
- **Avoid full re-renders:** do not replace the whole watchlist array on every tick. Do not re-sort the list on every tick either; re-sort on a slower timer or only when the user asks.
- **Be a good client:** subscribe only to symbols on screen, unsubscribe when they leave, pause when the tab is hidden, and reconnect with exponential backoff. See [MDN: WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket).

## Example
A tick buffer with conflation, verified with node:

```js
function createTickBuffer(flush, intervalMs = 100) {
  let pending = new Map();
  let timer = null;
  return {
    push(tick) {
      pending.set(tick.symbol, tick); // newer tick overwrites the older one
      if (!timer) {
        timer = setTimeout(() => {
          const batch = pending;
          pending = new Map();
          timer = null;
          flush(batch);
        }, intervalMs);
      }
    },
  };
}

const buf = createTickBuffer((batch) => {
  console.log('flush', batch.size, 'symbols:',
    [...batch.values()].map((t) => `${t.symbol}=${t.price}`).join(', '));
});
for (let i = 1; i <= 1000; i++) {
  buf.push({ symbol: ['INFY', 'TCS', 'RELIANCE'][i % 3], price: 100 + i });
}
setTimeout(() => buf.push({ symbol: 'TCS', price: 2000 }), 150);
```

```bash
flush 3 symbols: TCS=1100, RELIANCE=1098, INFY=1099   # 1000 ticks became 3 updates
flush 1 symbols: TCS=2000                             # later tick, next window
```

Wiring it into Svelte 5 so each row updates on its own:

```svelte
<script>
  import { SvelteMap } from 'svelte/reactivity';
  import { createTickBuffer } from '$lib/tickBuffer.js'; // the function above
  let { symbols } = $props();

  // Reactive Map: setting one key only updates components reading that key
  const prices = new SvelteMap();

  $effect(() => {
    const list = [...symbols]; // read synchronously so the effect re-runs when symbols change
    const ws = new WebSocket('wss://stream.example.com/prices');
    ws.onopen = () => ws.send(JSON.stringify({ subscribe: list }));

    const buffer = createTickBuffer((batch) => {
      for (const [symbol, tick] of batch) prices.set(symbol, tick.price);
    }, 100);

    ws.onmessage = (e) => buffer.push(JSON.parse(e.data));
    return () => ws.close(); // cleanup when component unmounts or symbols change
  });
</script>

{#each symbols as symbol (symbol)}
  <div class="row">{symbol}: {prices.get(symbol) ?? '--'}</div>
{/each}
```

## When to use it
Watchlists, order books, live P&L, index tickers, option chains: basically every screen in a trading app. Also live order status updates and notifications. The same buffering idea works for Server-Sent Events or polling.

## Likely questions
### How would you show live prices for 500 stocks without the UI lagging?
I would keep a WebSocket connection, subscribe only to the symbols visible on screen, and push incoming ticks into a buffer keyed by symbol so only the latest price per symbol is kept. Every 100 ms or so I flush the buffer into reactive state; in Svelte I use a `SvelteMap` or per-row state so only the changed rows update. The list itself is virtualized if long, and sorting by price change runs on a slower timer, not every tick.

### Throttle vs debounce for live data?
Throttle, or rather batching on a fixed interval. Debounce waits for a quiet period, and in a busy market there may never be one, so the UI would freeze on stale prices. Throttling guarantees a steady update rate, for example 10 times a second, which still looks "live" to a human.

### Why not update on every message?
Each update means reactive work, DOM writes and possibly layout and paint. At hundreds of ticks per second, the main thread gets busy, clicks on "Buy" feel slow (bad INP), and on low-end phones frames drop. Humans cannot read faster than a few updates per second per number anyway.

### `requestAnimationFrame` or `setTimeout` for flushing?
`requestAnimationFrame` lines up with the browser's paint, so you never do two flushes per frame, and it pauses automatically in background tabs. A `setTimeout` of 100 to 250 ms gives an even lower, calmer update rate. I often use a timer for the data flush and let price-flash animations use CSS. See [MDN: requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame).

### How do you handle disconnects and reconnects?
Listen for `close` and `error`, then reconnect with exponential backoff and jitter (1 s, 2 s, 4 s, capped). Show a "reconnecting" badge so users know prices may be stale. After reconnecting, re-subscribe and fetch a fresh snapshot over HTTP, because ticks were missed. Use a heartbeat (ping/pong) to detect dead connections.

### What if parsing the messages is itself expensive?
Move the WebSocket and parsing into a Web Worker. The worker parses, conflates and sends one small batch per interval to the main thread with `postMessage`. This keeps the main thread free for user interactions.

### What about when the tab is hidden?
On `visibilitychange` to hidden, pause UI flushes or unsubscribe from non-critical streams to save battery and data. When the tab becomes visible again, fetch a fresh snapshot so the screen is correct immediately.

## Common mistakes
- Replacing the whole array on every tick (`list = list.map(...)`), which updates every row.
- Re-sorting a big list on every tick, causing rows to jump while the user tries to click.
- Forgetting to close the socket on unmount, leaking connections and duplicate handlers.
- Using debounce for a continuous stream.
- Not showing staleness: a frozen price that looks live is dangerous in finance.

## Resources
- [MDN: WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket) - events, methods, close codes
- [javascript.info: WebSocket](https://javascript.info/websocket) - easy intro with examples
- [Svelte docs: svelte/reactivity (SvelteMap)](https://svelte.dev/docs/svelte/svelte-reactivity) - reactive Map for per-key updates
- [web.dev: Optimize INP](https://web.dev/articles/optimize-inp) - why busy main threads hurt clicks
- [MDN: Using Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers) - move parsing off the main thread
