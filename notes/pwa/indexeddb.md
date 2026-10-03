# IndexedDB

> **In one line:** IndexedDB is the browser's built-in asynchronous database: it stores large amounts of structured data (objects, Blobs, typed arrays) in object stores, lets you query them through indexes, groups reads and writes in transactions, and works in workers and service workers, which is why I pick it over `localStorage` for anything bigger than a few small settings.

## Key points
- **vs localStorage:** [`localStorage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage) is synchronous (blocks the main thread), strings only, around 5 MB, and not available in workers. [IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) is async, stores real JS objects and binary data, can hold hundreds of MB or more (quota-based), and works in Web Workers and Service Workers.
- **Object stores:** like tables. Each holds records with a key, either a `keyPath` from the object (`'id'`) or an auto-increment key.
- **Indexes:** a second way to look records up, by another field (for example `symbol` or `createdAt`). Can be `unique` and support range queries with `IDBKeyRange`.
- **Transactions:** every read or write runs inside a transaction (`readonly` or `readwrite`) over one or more stores. It is all-or-nothing: if one step fails, the whole transaction rolls back. It auto-commits when no requests are pending.
- **Versions and schema:** `indexedDB.open(name, version)`. Stores and indexes can only be created or changed in the `upgradeneeded` event, which runs when the version number goes up.
- **Wrapper libraries:** the raw API uses events (`onsuccess`, `onerror`), which is clumsy. [`idb`](https://github.com/jakearchibald/idb) by Jake Archibald is a tiny wrapper that turns it into Promises. Dexie is a bigger, query-friendly option.

## Example
Raw API, to show the shape:

```js
const req = indexedDB.open('trading', 1);
req.onupgradeneeded = () => {
  const db = req.result;
  const orders = db.createObjectStore('orders', { keyPath: 'id' });
  orders.createIndex('bySymbol', 'symbol');
};
req.onsuccess = () => {
  const db = req.result;
  const tx = db.transaction('orders', 'readwrite');
  tx.objectStore('orders').put({ id: 'o1', symbol: 'INFY', qty: 10, status: 'pending' });
  tx.oncomplete = () => console.log('saved');
  tx.onerror = () => console.error(tx.error);
};
```

Same thing with `idb` (what I would use in a real app):

```js
// db.js
import { openDB } from 'idb';

export const dbPromise = openDB('trading', 2, {
  upgrade(db, oldVersion) {
    if (oldVersion < 1) {
      const orders = db.createObjectStore('orders', { keyPath: 'id' });
      orders.createIndex('bySymbol', 'symbol');
    }
    if (oldVersion < 2) {
      // added in v2: cache of candles, key = [symbol, time]
      db.createObjectStore('candles', { keyPath: ['symbol', 't'] });
    }
  },
});

export async function saveOrder(order) {
  const db = await dbPromise;
  await db.put('orders', order);                       // single-op transaction
}

export async function ordersFor(symbol) {
  const db = await dbPromise;
  return db.getAllFromIndex('orders', 'bySymbol', symbol);
}

export async function saveCandles(candles) {
  const db = await dbPromise;
  const tx = db.transaction('candles', 'readwrite');
  // queue all puts in one transaction, then wait for it to finish
  await Promise.all([...candles.map((c) => tx.store.put(c)), tx.done]);
}

export async function candlesBetween(symbol, from, to) {
  const db = await dbPromise;
  const range = IDBKeyRange.bound([symbol, from], [symbol, to]);
  return db.getAll('candles', range);
}
```

## When to use it
- Offline-first apps: cache the watchlist, instruments list, recent orders, and chart candles so the app opens instantly and works offline.
- An outbox of user actions (place alert, edit watchlist) to replay when back online, read by the service worker.
- Any big or binary data: downloaded PDFs of statements, images.
- Keep `localStorage` for tiny, sync-friendly prefs like theme or last selected tab.

## Likely questions
### When would you use IndexedDB over localStorage?
When the data is big, structured, needed in a worker, or must not block the UI. `localStorage` is sync, so a big `JSON.parse` on every read blocks the main thread, it only stores strings, has about 5 MB, and is not available in service workers. IndexedDB is async, stores objects and Blobs directly, has a much bigger quota, supports indexes and transactions, and works in workers. For a theme setting, `localStorage` is fine.

### Why is the API asynchronous?
Disk I/O can be slow, and a sync API would freeze the page while it waits. So every operation returns a request (or a Promise with `idb`) and you get the result later, keeping the main thread free.

### What are object stores and indexes?
An object store is like a table that holds records by a primary key. An index is an extra lookup on another property, so I can ask "all orders for INFY" without scanning every record. Indexes can also be compound (an array key path) and queried with ranges.

### How do transactions work?
I open a transaction on the stores I need, with a mode `readonly` or `readwrite`. All requests inside it either all succeed or all roll back. It commits automatically once no more requests are queued, so I must not `await` unrelated things (like a `fetch`) in the middle, or the transaction closes and later calls throw `TransactionInactiveError`.

### How do you change the schema?
Bump the version in `open`. The `upgradeneeded` (or `idb`'s `upgrade`) callback gets the old version, and I run migrations step by step with `if (oldVersion < N)`. If another tab has the old version open, it gets a `versionchange` event and should close its connection, otherwise the upgrade is blocked.

### Why use a wrapper like idb?
The raw API is event-based and verbose. `idb` is about 1 KB and just adds Promises, so I can write `await db.get('orders', id)`. It keeps the real IndexedDB model, so knowledge transfers. Dexie adds more on top, like a query builder and live queries.

## Common mistakes
- Awaiting a network call inside a transaction, so it auto-commits early.
- Trying to create a store outside `upgradeneeded`.
- Storing secrets or auth tokens: any script on the origin (XSS) can read IndexedDB.
- Forgetting that the browser can evict data under storage pressure (see Storage limits and eviction).
- Not handling `blocked` / `versionchange` when users have two tabs open.

## Resources
- [MDN: Using IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API/Using_IndexedDB) - complete walkthrough of the raw API
- [javascript.info: IndexedDB](https://javascript.info/indexeddb) - clear explanation of stores, indexes and transactions
- [web.dev: Work with IndexedDB](https://web.dev/articles/indexeddb) - uses the `idb` wrapper
- [web.dev: Storage for the web](https://web.dev/articles/storage-for-the-web) - which storage to choose
