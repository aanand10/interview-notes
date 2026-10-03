# AbortController and timeouts

> **In one line:** An `AbortController` gives you a `signal` to pass into `fetch` (or any API that accepts one) and an `abort()` button; calling it cancels the request and makes the promise reject with an `AbortError`, which is how we add timeouts, cancel stale searches and clean up when a component unmounts.

## Key points
- `const c = new AbortController()` -> pass `c.signal` to `fetch(url, { signal })` -> call `c.abort()` to cancel ([MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)).
- Aborting rejects the fetch promise with a `DOMException` named `AbortError` (or the custom reason you pass to `abort(reason)`).
- A controller is **single-use**. Once aborted, it stays aborted; create a new one per request.
- Built-in helpers: [`AbortSignal.timeout(ms)`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static) rejects with `TimeoutError`; [`AbortSignal.any([s1, s2])`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/any_static) aborts when any input aborts.
- Signals also work with `addEventListener(type, fn, { signal })`, which removes listeners on abort. Handy for cleanup.

## Example: implement `fetchWithTimeout`
```js
async function fetchWithTimeout(url, { timeout = 5000, signal, ...options } = {}) {
  const controller = new AbortController();
  const timer = setTimeout(
    () => controller.abort(new DOMException('Request timed out', 'TimeoutError')),
    timeout
  );
  // Also respect a signal from the caller (for example a component unmount).
  const onAbort = () => controller.abort(signal.reason);
  if (signal) {
    if (signal.aborted) controller.abort(signal.reason);
    else signal.addEventListener('abort', onAbort, { once: true });
  }
  try {
    return await fetch(url, { ...options, signal: controller.signal });
  } finally {
    clearTimeout(timer);                            // no leaking timers
    signal?.removeEventListener('abort', onAbort);
  }
}
```
Tested in node against a local server (one slow route, one fast):
```text
slow: TimeoutError - Request timed out
fast: 200 { q: '/fast?delay=10' }
AbortSignal.timeout: TimeoutError
manual abort: AbortError
```
- `slow`: took 300 ms, timeout 100 ms, so it was aborted.
- `fast`: finished in time, timer was cleared.
- The modern one-liner gives the same result:

```js
const signal = AbortSignal.any([userSignal, AbortSignal.timeout(5000)]);
const res = await fetch(url, { signal });
```
Note: my version clears the timer once the response headers arrive, so a slow body download (`await res.json()`) is not covered by the timeout. With `AbortSignal.timeout`, the timer keeps running, so the body read is covered too. Both are fine for small JSON APIs; say which one you chose.

## Example: cancel stale autocomplete requests (Svelte 5)
Each keystroke aborts the previous request, so only the latest search can update the list.

```svelte
<script lang="ts">
  let query = $state('');
  let results = $state<string[]>([]);
  let error = $state('');

  $effect(() => {
    const q = query.trim();          // read state BEFORE any await so the effect tracks it
    if (!q) { results = []; return; }

    const controller = new AbortController();
    const timer = setTimeout(async () => {         // small debounce
      try {
        const res = await fetch(`/api/search?q=${encodeURIComponent(q)}`, {
          signal: controller.signal
        });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        results = await res.json();
        error = '';
      } catch (e) {
        if ((e as Error).name !== 'AbortError') error = 'Search failed';
      }
    }, 250);

    // Runs before the effect re-runs (new query) AND when the component unmounts.
    return () => {
      clearTimeout(timer);
      controller.abort();
    };
  });
</script>

<input bind:value={query} placeholder="Search symbol" aria-label="Search symbol" />
{#if error}<p role="alert">{error}</p>{/if}
<ul>
  {#each results as r (r)}<li>{r}</li>{/each}
</ul>
```

## When to use it
- Symbol search / autocomplete on a trading app.
- Switching tabs between "Stocks" and "Options" quickly: abort the old tab's load.
- Leaving a page while a big report or chart history is still downloading.
- Any call that must not hang forever: placing an order should time out and show "please check order status".

## Likely questions
### Implement fetchWithTimeout.
See the code above. Create a controller, `setTimeout` that calls `abort`, pass the signal to fetch, and clear the timer in `finally`. Bonus points: accept an outside signal too, and mention that `AbortSignal.timeout(ms)` plus `AbortSignal.any` now do this natively.

### How do you cancel stale autocomplete requests?
Keep the controller of the last request. When the user types again, call `abort()` on it before starting a new one. In Svelte 5 the cleanup function returned from `$effect` is the natural place, because it runs right before the effect re-runs with the new query. Ignore `AbortError` in the `catch` so cancelling does not show an error.

### How do you cancel on component unmount?
Return a cleanup from `$effect` that calls `controller.abort()`; Svelte runs it when the component is destroyed ([Svelte docs: $effect](https://svelte.dev/docs/svelte/$effect)). In React the same idea is the `useEffect` cleanup. Without this, a late response may set state on a dead component or waste bandwidth.

### Does abort stop the server from processing?
Not necessarily. The browser closes the connection and stops waiting, but the server may already be working. For non-idempotent actions like placing an order, a timeout does not mean "not placed". Use an idempotency key and check order status.

### What is `signal.reason` and `throwIfAborted`?
`abort(reason)` stores the reason on `signal.reason`, and fetch rejects with it. `signal.throwIfAborted()` throws that reason, useful inside your own long async loops to stop early.

## Common mistakes
- Reusing an already-aborted controller for the next request (it fails immediately).
- Showing "Something went wrong" for an `AbortError` that you caused on purpose.
- Forgetting `clearTimeout`, leaving timers alive.
- In Svelte 5, reading `query` only after an `await` inside `$effect`, so the effect does not re-run when it changes.

## Resources
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) - API and fetch cancel example
- [MDN: AbortSignal.timeout()](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static) - built-in timeout signal and TimeoutError
- [javascript.info: Fetch: Abort](https://javascript.info/fetch-abort) - simple walkthrough of aborting fetch
- [Svelte docs: $effect](https://svelte.dev/docs/svelte/$effect) - effect cleanup on re-run and destroy
