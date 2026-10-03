# Storage limits and eviction

> **In one line:** Each origin gets a storage quota shared by IndexedDB, the Cache API and similar storage; I can check usage with `navigator.storage.estimate()`, and because "best-effort" data can be evicted when the device is low on space, I ask for persistent storage with `navigator.storage.persist()` when losing data would hurt.

## Key points
- **One shared quota per origin:** IndexedDB, Cache API, OPFS (Origin Private File System) and service worker registrations all count toward the same [quota](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria). `localStorage` is separate and small (about 5 MB).
- **Quotas are big but differ by browser** (rough numbers, they change over time): Chrome lets an origin use up to about 60% of total disk; Firefox about 10% of disk (up to 10 GiB) in best-effort mode, more when persistent; Safari (17+) about 60% of disk for browser apps, less for apps inside other apps' WebViews. Never hard-code these; ask the browser.
- **Check usage:** [`navigator.storage.estimate()`](https://developer.mozilla.org/en-US/docs/Web/API/StorageManager/estimate) returns `{ usage, quota }` in bytes. They are estimates (padded for privacy), not exact numbers.
- **Two modes, best-effort and persistent:** by default storage is **best-effort**: when the disk is low, the browser evicts whole origins, least recently used first, without asking. [`navigator.storage.persist()`](https://developer.mozilla.org/en-US/docs/Web/API/StorageManager/persist) asks to make it **persistent**, so it is only removed if the user clears it.
- **Eviction is all or nothing per origin:** the browser does not delete one IndexedDB record; it clears all of the origin's data together. Safari also has a 7-day rule: script-written storage of a site the user has not visited (interacted with) in 7 days of browser use can be deleted, though installed home-screen web apps are exempt.

## Example
```js
// storage.js
export async function storageReport() {
  if (!navigator.storage?.estimate) return null;
  const { usage, quota } = await navigator.storage.estimate();
  const mb = (b) => (b / 1024 / 1024).toFixed(1);
  return { usedMB: mb(usage), quotaMB: mb(quota), percent: ((usage / quota) * 100).toFixed(1) };
}

export async function ensurePersistent() {
  if (!navigator.storage?.persist) return false;
  if (await navigator.storage.persisted()) return true;   // already granted
  return navigator.storage.persist();                     // true if granted
}

// Before saving 200 MB of historical candles for offline charts:
const { usage, quota } = await navigator.storage.estimate();
if (quota - usage < 250 * 1024 * 1024) {
  // not enough room: save less history, or trim old data first
}
```

```js
// Writes can still fail; handle the quota error
try {
  await db.put('candles', bigBatch);
} catch (err) {
  if (err.name === 'QuotaExceededError') {
    await trimOldCandles();   // free space, then retry or tell the user
  } else throw err;
}
```

## When to use it
- A trading PWA that keeps offline chart history and an outbox of unsent actions. Show "Offline data: 120 MB" in settings with a "Clear" button, request persistence after the user installs the app or turns on offline mode, and trim old candles when close to the quota.

## Likely questions
### How much can I store in the browser?
It depends on the browser and the free disk space, and it is shared by IndexedDB, the Cache API and OPFS. Modern browsers give a lot, often many GB on a desktop. I do not guess: I call `navigator.storage.estimate()` to get `usage` and `quota`, and I handle `QuotaExceededError` on writes.

### What does `navigator.storage.estimate()` return?
A Promise of an object with `usage` and `quota` in bytes. Both are estimates. Browsers round or pad them so sites cannot fingerprint users, so use them for decisions like "do I have room for this download", not exact accounting.

### What is eviction and when does it happen?
Eviction is the browser deleting stored data to free space. For best-effort storage it happens when the device is low on disk, and it removes whole origins, least recently used first. Safari can also clear data for sites not used for a while. So data in the browser should be treated as a cache that I can rebuild from the server, unless it is persistent.

### How do you request persistent storage?
Call `navigator.storage.persist()`. It returns a Promise of `true` or `false`. Chrome decides on its own without a prompt, based on signals like site engagement, the app being installed or bookmarked, or notification permission. Firefox shows the user a prompt. I check `navigator.storage.persisted()` first, and I ask at a meaningful moment, like when the user enables offline mode, not on first load.

### If storage is persistent, is it safe forever?
It is safe from automatic eviction, but the user can still clear site data, uninstall, or use private browsing. So important data, like an order that has not reached the server, must sync to the server as soon as possible.

## Common mistakes
- Treating browser storage as the source of truth for important data.
- Assuming `localStorage` limits apply to IndexedDB (they do not).
- Asking for `persist()` on page load with no context.
- Not catching `QuotaExceededError`.

## Resources
- [MDN: Storage quotas and eviction criteria](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria) - per-browser limits and eviction rules
- [web.dev: Storage for the web](https://web.dev/articles/storage-for-the-web) - quotas, estimate and persist explained
- [web.dev: Persistent storage](https://web.dev/articles/persistent-storage) - how browsers grant persistence
- [MDN: StorageManager.estimate()](https://developer.mozilla.org/en-US/docs/Web/API/StorageManager/estimate) - API reference
