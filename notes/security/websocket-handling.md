# WebSocket handling

> **In one line:** A production WebSocket client needs three things beyond `new WebSocket()`: reconnect with backoff, a fresh snapshot after reconnect so no data is stale, and subscriptions that follow what the user can actually see.

## Key points
- **Reconnect with exponential backoff + jitter:** 1s, 2s, 4s, ... capped at ~30s, with randomness so all clients do not hit the server at once. Reset after a successful connect.
- **Stale data after reconnect:** messages sent while you were disconnected are gone. Re-subscribe, then fetch a **snapshot** (REST) or ask the server to replay from a sequence number. Mark data as stale in the UI until fresh.
- **Subscribe only to visible symbols:** use [`IntersectionObserver`](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) on watchlist rows, and send subscribe / unsubscribe diffs. Saves bandwidth, battery and server fan-out.
- **Heartbeat** to detect dead connections; **pause** when the tab is hidden or offline.
- **Batch UI updates** (once per animation frame) so 100 ticks per second do not cause 100 renders.

## Example: a Svelte 5 price store

```ts
// prices.svelte.ts
export const prices = $state<Record<string, { ltp: number; seq: number; stale: boolean }>>({});

const wanted = new Map<string, number>(); // symbol -> how many rows want it
let ws: WebSocket | null = null;
let attempt = 0;
let pending: Record<string, any> = {};
let frame = 0;

function send(msg: object) {
  if (ws?.readyState === WebSocket.OPEN) ws.send(JSON.stringify(msg));
}

function connect() {
  ws = new WebSocket('wss://stream.example/prices');

  ws.onopen = async () => {
    attempt = 0;
    const symbols = [...wanted.keys()];
    if (!symbols.length) return;
    send({ type: 'subscribe', symbols });
    // Fill the gap: snapshot replaces stale values
    const snap = await fetch(`/api/quotes?symbols=${symbols.join(',')}`).then((r) => r.json());
    for (const q of snap) applyTick(q);
  };

  ws.onmessage = (e) => {
    const t = JSON.parse(e.data); // { symbol, ltp, seq }
    pending[t.symbol] = t;        // keep only the latest per symbol
    frame ||= requestAnimationFrame(flush);
  };

  ws.onclose = () => {
    for (const s in prices) prices[s].stale = true; // show grey "stale" state
    if (!navigator.onLine) return;                  // wait for "online" event
    const delay = Math.min(30_000, 1000 * 2 ** attempt++) * (0.5 + Math.random() / 2);
    setTimeout(connect, delay);
  };
}

function flush() {
  frame = 0;
  for (const t of Object.values(pending)) applyTick(t);
  pending = {};
}

function applyTick(t: { symbol: string; ltp: number; seq: number }) {
  const cur = prices[t.symbol];
  if (cur && t.seq <= cur.seq && !cur.stale) return; // ignore out-of-order / old ticks
  prices[t.symbol] = { ltp: t.ltp, seq: t.seq, stale: false };
}

export function watch(symbol: string) {
  const n = wanted.get(symbol) ?? 0;
  wanted.set(symbol, n + 1);
  if (n === 0) send({ type: 'subscribe', symbols: [symbol] });
}

export function unwatch(symbol: string) {
  const n = (wanted.get(symbol) ?? 1) - 1;
  if (n > 0) return void wanted.set(symbol, n);
  wanted.delete(symbol);
  send({ type: 'unsubscribe', symbols: [symbol] });
}

addEventListener('online', () => { if (ws?.readyState !== WebSocket.OPEN) connect(); });
connect();
```

A row subscribes only while on screen, using a Svelte action:

```svelte
<script>
  import { prices, watch, unwatch } from './prices.svelte.ts';
  let { symbol } = $props();

  function visible(node) {
    let watching = false;
    const io = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting && !watching) { watch(symbol); watching = true; }
      else if (!entry.isIntersecting && watching) { unwatch(symbol); watching = false; }
    });
    io.observe(node);
    return {
      destroy() {
        io.disconnect();
        if (watching) unwatch(symbol); // clean up on unmount
      },
    };
  }
</script>

<li use:visible class:stale={prices[symbol]?.stale}>
  {symbol}: {prices[symbol]?.ltp ?? '--'}
</li>
```

## When to use it
- **Watchlist with 500 symbols:** only the ~20 visible rows are subscribed. Scrolling sends small diffs.
- **Order updates:** after reconnect, refetch open orders by REST, because a "FILLED" event may have been missed.
- **Mobile / WebView:** the OS often kills sockets when the app goes to background. On `visibilitychange` to visible, check the socket and reconnect right away.

## Likely questions

### How do you reconnect safely?
Exponential backoff with a cap and jitter, reset the counter on success, and do not retry while `navigator.onLine` is false; instead reconnect on the `online` event. I also distinguish intentional close (code 1000 on logout) from errors, so logout does not trigger a reconnect.

### After reconnecting, how do you avoid showing stale data?
I mark all prices as stale the moment the socket closes, so the UI greys them out. On reconnect I re-subscribe and fetch a snapshot. If the server sends sequence numbers, I drop any tick whose sequence is older than what I have, and for order streams I can ask the server to replay from the last sequence I saw.

### How do you subscribe only to visible symbols?
IntersectionObserver on each row calls `watch` or `unwatch`. A reference count handles the same symbol appearing in two places, like the watchlist and an open order ticket. I send subscribe/unsubscribe diffs instead of the whole list. Some teams debounce this during fast scrolling.

### How do you stop the UI from lagging with very fast ticks?
Buffer messages and flush once per `requestAnimationFrame`, keeping only the latest tick per symbol. With Svelte 5 fine-grained reactivity, only the rows whose values changed update. For very heavy parsing, move the socket into a Web Worker or SharedWorker (one socket shared by all tabs).

## Common mistakes
- Reconnecting immediately in a loop when the server is down.
- Forgetting to re-subscribe after reconnect, so the socket is open but silent.
- Subscribing to every symbol in a long list.
- Leaking sockets or observers when a component unmounts.

## Resources
- [MDN: Writing WebSocket client applications](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_client_applications) - client basics
- [MDN: IntersectionObserver](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) - visibility detection
- [MDN: Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API) - pause and resume on tab hide
- [javascript.info: WebSocket](https://javascript.info/websocket) - events, close codes, bufferedAmount
