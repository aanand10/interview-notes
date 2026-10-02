# Offline-first architecture

> **In one line:** Offline-first means the UI reads from local storage first and treats the network as a sync mechanism, so the app works instantly without a connection, queues user actions, and syncs and resolves conflicts when it comes back online.

## Key points
- **Read local first:** render from IndexedDB (or the Cache API) straight away, then fetch fresh data and update both the store and the UI. The user never stares at a spinner because of a slow network.
- **Write to a local outbox:** user actions go into a queue in IndexedDB with a unique id, the UI updates optimistically, and a sync step sends them when online (on `online` event, app start, or Background Sync).
- **Conflicts:** when the server and the device both changed the same thing, pick a rule: last-write-wins with timestamps, version numbers (server rejects stale versions), field-level merge, or ask the user.
- **Idempotency:** each queued action carries an **idempotency key** so a retry never runs twice on the server.
- **Some data must never be served stale:** live prices, order status, balances / buying power, margin. Show them with a timestamp and a "stale" badge, and block actions that depend on them.

## Example
```ts
// Local-first data layer (uses the idb wrapper)
import { openDB } from 'idb';

const db = await openDB('trade-app', 1, {
  upgrade(db) {
    db.createObjectStore('watchlist', { keyPath: 'symbol' });
    db.createObjectStore('outbox', { keyPath: 'id' });
  },
});

// 1. Read local first, then refresh from network
export async function loadWatchlist(onData: (rows: any[]) => void) {
  onData(await db.getAll('watchlist'));                 // instant, works offline
  try {
    const res = await fetch('/api/watchlist');
    const fresh = await res.json();
    const tx = db.transaction('watchlist', 'readwrite');
    await tx.store.clear();
    await Promise.all(fresh.map((r) => tx.store.put(r)));
    await tx.done;
    onData(fresh);
  } catch { /* offline: keep showing local data */ }
}

// 2. Queue a change while offline (optimistic update)
export async function addToWatchlist(symbol: string) {
  await db.put('watchlist', { symbol, pending: true });
  await db.put('outbox', {
    id: crypto.randomUUID(),            // doubles as idempotency key
    type: 'WATCHLIST_ADD', payload: { symbol }, createdAt: Date.now(),
  });
  void flushOutbox();
}

// 3. Sync the queue in order when online
export async function flushOutbox() {
  if (!navigator.onLine) return;
  for (const item of await db.getAll('outbox')) {
    const res = await fetch('/api/watchlist', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Idempotency-Key': item.id },
      body: JSON.stringify(item.payload),
    }).catch(() => null);
    if (!res) return;                                   // still offline, try later
    if (res.ok || res.status === 409) await db.delete('outbox', item.id); // 409: conflict, server wins
  }
}
window.addEventListener('online', flushOutbox);
```

```svelte
<!-- Price cell: never pretend stale data is live -->
<script>
  let { quote } = $props(); // { price, ts }
  let now = $state(Date.now());
  $effect(() => { const t = setInterval(() => (now = Date.now()), 1000); return () => clearInterval(t); });
  let stale = $derived(now - quote.ts > 5000);
</script>

<span class:stale>{quote.price.toFixed(2)}</span>
{#if stale}<small>Delayed, last update {new Date(quote.ts).toLocaleTimeString()}</small>{/if}
```

## When to use it
- **Good for offline-first:** watchlists, notes, alerts setup, research and charts history, profile settings, draft orders.
- **Must be online:** placing or cancelling an order, live prices, current order status, balances, KYC and payments. For these, offline-first means "show the last known value clearly marked, and disable the action".

## Likely questions
### What does offline-first mean, compared to "offline support"?
Offline support usually means "show an offline page when the network fails". Offline-first flips the model: local data is the source the UI reads, and the network only syncs it. The app opens instantly every time, not just when offline, because the first paint never waits for the network.

### How do you queue actions offline?
I store each action in an IndexedDB "outbox" with a UUID, a type, a payload and a timestamp, and update the UI optimistically with a "pending" state. When the app is back online, I send them in order. Each request sends the UUID as an idempotency key, so if the response was lost and I retry, the server ignores the duplicate. On Chromium I can also use Background Sync so the retry happens even if the tab is closed.

### How do you handle conflicts?
First I decide per data type. For simple personal settings, last-write-wins is fine. For shared or important data, each record has a version number; the client sends the version it edited, and the server returns 409 if it changed. Then I either merge field by field, let the server win and refresh, or show the user both versions. For lists like a watchlist, operations ("add AAPL", "remove TSLA") merge better than sending the whole list.

### Should a trading app queue orders offline?
No. A market order queued while offline could execute minutes later at a very different price. That is a real money risk and often a compliance issue. I would block order submission when offline, keep the draft in local storage, and let the user re-submit when back online after seeing the live price.

### What must never be served stale, and how do you show it?
Live prices, order status, balances, buying power and margin. They use network only, often WebSocket streams. If the connection drops I keep the last value but mark it with a timestamp and a "delayed" style, and I disable buy and sell. After reconnect I refetch a snapshot before trusting the stream again.

### How do you detect online and offline?
`navigator.onLine` and the `online` / `offline` events only tell you if there is a network interface, not if the server is reachable. So I also treat failed fetches and WebSocket heartbeats as signals. A "connection" store in Svelte can combine both.

## Common mistakes
- Trusting `navigator.onLine === true` as "the API is reachable".
- Retrying POSTs without idempotency keys, causing duplicate actions.
- Showing cached prices with no timestamp or stale marker.
- Forgetting to clear local data on logout, leaking one user's data to the next on a shared device.
- Not handling storage eviction: local data is a cache, the server is the source of truth.

## Resources
- [web.dev: Learn PWA - Offline data](https://web.dev/learn/pwa/offline-data) - IndexedDB and Cache API for offline
- [web.dev: The Offline Cookbook](https://web.dev/articles/offline-cookbook) - patterns for offline responses and updates
- [MDN: Offline and background operation](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Offline_and_background_operation) - sync, background fetch and push
- [MDN: Navigator.onLine](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/onLine) - what online detection really means
- [MDN: Background Synchronization API](https://developer.mozilla.org/en-US/docs/Web/API/Background_Synchronization_API) - retry when back online
