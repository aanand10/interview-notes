# Race conditions

> **In one line:** In the frontend, a race condition usually means two async requests finish in a different order than they started, so an older, slower response overwrites the newer result; we fix it by aborting old requests, tagging requests with an id, or only accepting the latest one.

## Key points
- Network time is random. Request 1 (`"ap"`) can take 200 ms and request 2 (`"apple"`) 50 ms, so request 1 lands **last** and wins.
- JS is single-threaded, so this is not about two threads touching memory. It is about **callbacks arriving out of order** after an `await`.
- **Fix 1: abort** the previous request with [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController). Best: saves bandwidth too.
- **Fix 2: request id / latest-only**: give each request an increasing id, and ignore a response whose id is not the latest.
- **Fix 3: compare the input**: when the response arrives, check it still matches the current query.
- Same bug shows up in: tab switching, pagination, filters, double-clicked "Buy" buttons, and live price updates arriving out of order.

## Example
The bug and the request-id fix, side by side:

```js
const fakeSearch = (q, ms) =>
  new Promise((r) => setTimeout(() => r(`results for "${q}"`), ms));

// Buggy: whoever finishes last wins
let shown = '';
async function naive(q, ms) { shown = await fakeSearch(q, ms); }
naive('ap', 200);     // older, slow
naive('apple', 50);   // newer, fast
setTimeout(() => console.log('naive shows:', shown), 300);

// Fixed: latest-only with a request id
let latestId = 0, shown2 = '';
async function latestOnly(q, ms) {
  const id = ++latestId;              // remember which request this is
  const data = await fakeSearch(q, ms);
  if (id !== latestId) return console.log('  ignored stale:', q);
  shown2 = data;
}
latestOnly('ap', 200);
latestOnly('apple', 50);
setTimeout(() => console.log('request-id shows:', shown2), 300);
```
Real output (node):
```text
  ignored stale: ap
naive shows: results for "ap"
request-id shows: results for "apple"
```
- The naive version shows results for `"ap"` even though the user typed `"apple"`. That is the bug.
- The id version throws away the stale `"ap"` response.

## Fix with abort (Svelte 5)
```svelte
<script lang="ts">
  let query = $state('');
  let results = $state<string[]>([]);
  let controller: AbortController | null = null;

  async function search(q: string) {
    controller?.abort();                      // cancel the older request
    controller = new AbortController();
    try {
      const res = await fetch(`/api/search?q=${encodeURIComponent(q)}`, {
        signal: controller.signal
      });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      results = await res.json();
    } catch (e) {
      if ((e as Error).name !== 'AbortError') console.error(e);
    }
  }
</script>

<input bind:value={query} oninput={() => search(query)} aria-label="Search symbol" />
```
In real code combine this with a debounce (~250 ms) so you do not fire a request on every key. The `$effect` cleanup version is in the AbortController note.

## When to use it
- Symbol search box on a trading app (the classic case).
- Switching between "1D / 1W / 1M" chart ranges quickly: the 1M data must not replace the 1D chart the user picked last.
- Live prices over WebSocket: compare a sequence number or timestamp and drop older ticks.
- Order buttons: disable while submitting, and send an idempotency key so a double click does not place two orders.

## Likely questions
### The user types fast and sometimes sees results for an older query. Why, and how do you fix it?
Each keystroke sends a request, and responses can come back in any order. If an older, slower one comes last, it overwrites the newer results. I would fix it in layers: debounce the input to send fewer requests, abort the previous request with `AbortController`, and as a safety net only apply a response if it belongs to the latest request (request id or matching query).

### Abort vs request id: which is better?
Abort is better when you control the call, because it also frees the network and the server may stop early. A request id or "latest-only" check works for anything async, even things you cannot cancel (a third-party SDK, a Web Worker reply). I often use both: abort for efficiency, id check for correctness.

### How would you write a reusable "latest only" wrapper?
```js
function latestOnly(fn) {
  let lastId = 0;
  return async (...args) => {
    const id = ++lastId;
    const result = await fn(...args);
    if (id !== lastId) throw new DOMException('Stale result', 'AbortError');
    return result;
  };
}
const searchLatest = latestOnly(searchApi);
```
Callers treat the stale case like an abort and ignore it.

### Does debounce alone fix it?
No. Debounce reduces how many requests you send, but two requests can still overlap if the user pauses, types, and pauses again while the first is slow. You still need abort or an id check.

### How do you handle out-of-order live price updates?
Each message should carry a sequence number or server timestamp. Keep the last applied one per symbol and ignore anything older. Never trust arrival order on its own.

### What about a double-submitted order?
Disable the button while a request is in flight and, on the server side, use an idempotency key so the same order is not created twice. This is a race between two user actions rather than two responses.

## Common mistakes
- Only debouncing and calling it fixed.
- Showing an error toast for your own `AbortError`.
- Setting `loading = false` from a stale request, which hides the spinner while the latest request is still running. Only the latest request should change loading state.
- Trusting message arrival order for prices.

## Resources
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) - cancel the previous request
- [javascript.info: Fetch: Abort](https://javascript.info/fetch-abort) - abort pattern step by step
- [Svelte docs: $effect](https://svelte.dev/docs/svelte/$effect) - cleanup that runs before the next search
- [react.dev: Fetching data in effects (race conditions)](https://react.dev/learn/synchronizing-with-effects#fetching-data) - the "ignore" flag fix, same idea in React
