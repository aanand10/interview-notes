# REST vs WebSockets vs SSE vs polling

> **In one line:** I use REST for request-response actions like placing an order, WebSockets for high-frequency two-way streams like live prices, SSE for simple one-way server pushes like order status and notifications, and polling only when the data changes slowly or nothing else is available.

## Key points
- **REST (plain HTTP):** client asks, server answers. Great for actions and reading data on demand. No push.
- **Polling:** ask again every N seconds. Simple, works everywhere, but wastes requests and adds delay. **Long polling** holds the request open until there is news.
- **[SSE (Server-Sent Events)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events):** one long HTTP response where the server streams text events. One-way (server to client), **auto-reconnects** and can resume with `Last-Event-ID`.
- **[WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API):** a persistent two-way connection. Lowest overhead per message; best for many fast updates and client subscriptions. You build reconnection and heartbeats yourself.
- **Heartbeat** = small periodic ping/pong to detect a dead connection fast; **reconnection** = retry with exponential backoff plus jitter.

## Comparison

| | REST | Polling | SSE | WebSocket |
|---|---|---|---|---|
| Direction | Client asks | Client asks repeatedly | Server to client | Both ways |
| Latency | On demand | Up to the interval | Near real-time | Near real-time |
| Reconnect | n/a | n/a | Built in | Manual |
| Data | Any | Any | UTF-8 text | Text or binary |
| Works through proxies / HTTP caching | Yes | Yes | Mostly (plain HTTP) | Sometimes blocked by old proxies |
| Best for | Place order, fetch history | Slow data, fallback | Order updates, notifications, AI token streaming | Live prices, order book, chat |

Watch out: on HTTP/1.1 browsers allow only about 6 connections per domain, and each SSE stream uses one. On HTTP/2 this is not a problem.

## Example

```ts
// SSE: order status updates (one-way, auto-reconnect)
const es = new EventSource('/api/orders/stream', { withCredentials: true });
es.addEventListener('order', (e) => {
  const order = JSON.parse(e.data);   // { id, status: 'FILLED' }
  updateOrder(order);
});
es.onerror = () => console.warn('SSE dropped, browser will retry');
// Server sends:  id: 42\nevent: order\ndata: {"id":"o1","status":"FILLED"}\n\n
// On reconnect the browser sends header Last-Event-ID: 42 so the server can resume.
```

```ts
// WebSocket: live prices with heartbeat and backoff reconnect
function connectPrices(symbols: string[], onTick: (t: any) => void) {
  let ws: WebSocket;
  let attempt = 0;
  let heartbeat: ReturnType<typeof setInterval>;
  let lastMessageAt = Date.now();

  function open() {
    ws = new WebSocket('wss://stream.example/prices');
    ws.onopen = () => {
      attempt = 0;
      ws.send(JSON.stringify({ type: 'subscribe', symbols }));
      heartbeat = setInterval(() => {
        if (Date.now() - lastMessageAt > 15_000) ws.close(); // dead: no pong or data
        else ws.send('{"type":"ping"}');
      }, 5_000);
    };
    ws.onmessage = (e) => {
      lastMessageAt = Date.now();
      const msg = JSON.parse(e.data);
      if (msg.type === 'tick') onTick(msg);
    };
    ws.onclose = () => {
      clearInterval(heartbeat);
      const delay = Math.min(30_000, 1000 * 2 ** attempt++) * (0.5 + Math.random() / 2);
      setTimeout(open, delay); // exponential backoff + jitter
    };
  }
  open();
}
```

## When to use it (trading / checkout)
- **Live prices, market depth:** WebSocket. Many updates per second, and the client changes subscriptions as the user scrolls a watchlist.
- **Order updates (placed, partially filled, filled):** WebSocket if one is already open for prices; otherwise SSE is simpler. Always also refetch via REST after reconnect to fix missed events.
- **Notifications / alerts:** SSE, or web push when the tab is closed.
- **Payment status on checkout:** short polling (every 2-3 seconds, with a timeout) is common and robust, especially inside WebViews, plus a server webhook as the source of truth.
- **Placing an order:** REST `POST` with an idempotency key. You want a clear response and status code.

## Likely questions

### What would you use for live stock prices?
WebSockets. Prices update many times per second, the payload is small, and I need to send subscribe/unsubscribe messages as the visible symbols change. I add a heartbeat, reconnect with backoff, and throttle UI updates to animation frames so the page does not re-render on every tick.

### Why SSE instead of WebSockets for notifications?
Notifications only flow server to client, so I do not need two-way. SSE is plain HTTP, so it works with existing auth cookies, load balancers and HTTP/2, and the browser reconnects automatically and resumes with `Last-Event-ID`. Less code, fewer failure modes.

### When is polling fine?
When data changes slowly (portfolio summary every 30 seconds), when I need a fallback because WebSockets are blocked, or for a short-lived wait like payment confirmation. I pause polling when the tab is hidden (`document.visibilityState`) to save battery and server load.

### How do you handle reconnection and heartbeats?
On close or error, retry with exponential backoff and random jitter so thousands of clients do not reconnect at the same moment. A heartbeat (ping every few seconds, expect a pong or any message) detects "half-open" connections, which mobile networks create often, where the socket looks open but nothing arrives. After reconnecting, re-subscribe and refetch a snapshot over REST so missed updates are filled in.

### What about the network going offline?
Listen to `online`/`offline` events and `visibilitychange`. When offline, stop retrying and show a "Reconnecting" banner and mark prices as stale. When back online or visible again, reconnect immediately instead of waiting for the backoff timer.

## Common mistakes
- Using WebSockets for everything, including simple one-off reads.
- No heartbeat, so a dead connection shows frozen prices that look live.
- Reconnecting in a tight loop with no backoff.
- Trusting the stream alone for orders; always reconcile with a REST snapshot.

## Resources
- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events) - event format, `id`, `retry`
- [MDN: WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) - client API and events
- [javascript.info: Long polling](https://javascript.info/long-polling) - how long polling works
- [javascript.info: WebSocket](https://javascript.info/websocket) - handshake, frames, close codes
