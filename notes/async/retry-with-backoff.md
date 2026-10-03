# Retry with backoff

> **In one line:** Retry with backoff means "if an async call fails, try again, but wait a little longer each time, plus a bit of randomness", so a struggling server gets breathing room instead of a flood of retries.

## Key points
- **Retry** only things that can fail for a short time: network errors, timeouts, HTTP 429 (too many requests) and 5xx. Do not retry a 400 or 401; it will fail the same way again.
- **Exponential backoff** means the wait doubles every attempt: 100ms, 200ms, 400ms, 800ms... Cap it with a `maxDelay` so it never grows forever.
- **Jitter** means adding randomness to the wait. Without it, 10,000 clients that failed at the same second all retry at the same second again (the "thundering herd"). "Full jitter" picks a random wait between 0 and the backoff value.
- Always have a **maximum number of attempts**, and throw the last error when you give up.
- Only auto-retry **idempotent** requests (safe to repeat, like a GET). Repeating a "place order" POST can create two orders unless the server supports an idempotency key.

## Example
Tested with `node`.

```js
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

/**
 * Calls fn() up to `attempts` times.
 * Wait before retry i (0-based) = random(0, min(maxDelay, delay * 2^i))  -> full jitter
 */
async function retry(fn, attempts = 3, delay = 100, { maxDelay = 5000, shouldRetry = () => true } = {}) {
  let lastError;
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn(i);            // success: return straight away
    } catch (err) {
      lastError = err;
      const isLast = i === attempts - 1;
      if (isLast || !shouldRetry(err)) break;   // give up early on non-retryable errors
      const backoff = Math.min(maxDelay, delay * 2 ** i); // 100, 200, 400, ...
      const wait = Math.random() * backoff;               // jitter
      await sleep(wait);
    }
  }
  throw lastError;                   // all attempts used: surface the real error
}

// Test: fails twice, then works
let calls = 0;
const flaky = async () => {
  calls++;
  if (calls < 3) throw new Error(`fail ${calls}`);
  return 'ok';
};
console.log(await retry(flaky, 5, 50), 'after', calls, 'calls');

// Test: never retry a 4xx
let c2 = 0;
try {
  await retry(async () => { c2++; const e = new Error('400'); e.status = 400; throw e; },
    5, 10, { shouldRetry: (e) => e.status >= 500 });
} catch { console.log('no retry on 400, calls =', c2); }
```

Output:

```text
ok after 3 calls
no retry on 400, calls = 1
```

Using it with `fetch` (note: `fetch` does not reject on 500, so we throw ourselves):

```js
const quote = await retry(async () => {
  const res = await fetch('/api/quote/AAPL');
  if (!res.ok) {
    const err = new Error(`HTTP ${res.status}`);
    err.status = res.status;
    throw err;
  }
  return res.json();
}, 4, 200, { shouldRetry: (e) => !e.status || e.status === 429 || e.status >= 500 });
```

## When to use it
- Loading a watchlist or portfolio when the network is flaky (mobile users on trains).
- Reconnecting a WebSocket for live prices: wait 1s, 2s, 4s... up to 30s, with jitter so all users do not reconnect together after an outage.
- Not for placing orders, unless the API takes an idempotency key so a repeat is safe.

## Likely questions
### Implement `retry(fn, attempts, delay)` with exponential backoff and jitter.
Write a loop from 0 to `attempts - 1`. In each turn, `try` to `await fn()` and return on success. In the `catch`, save the error; if this was the last attempt, break; otherwise wait `random(0, delay * 2 ** i)` (capped) and loop again. After the loop, throw the last error. The code above is the full answer.

### Why add jitter? Isn't backoff enough?
Backoff spreads retries over time for one client, but all clients still use the same schedule. If a server goes down for everyone at 10:00:00, all clients retry at exactly 10:00:00.1, then 10:00:00.3, and so on, in waves. Jitter makes each client pick a random time, which turns the waves into a smooth trickle.

### Which errors should you retry?
Network failures, timeouts, 408, 429 and 5xx. Not 400, 401, 403, 404 or validation errors, because the same request will fail the same way. For a 429, respect the `Retry-After` header if the server sends one.

### How would you make it cancellable?
Accept an [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal). Check `signal.aborted` before each attempt, pass the signal to `fetch`, and make the sleep reject when the signal fires. Useful when the user leaves the page or switches stock.

### Recursive version?
`return fn().catch(err => attempts <= 1 ? Promise.reject(err) : sleep(d).then(() => retry(fn, attempts - 1, delay * 2)))`. Same idea; the loop version is easier to read and to add options to.

## Common mistakes
- Forgetting `await` inside `try` (`return fn()` without `await`), so the rejection escapes the `catch` and no retry happens.
- Sleeping after the last attempt (wasted wait before throwing).
- Swallowing the error and returning `undefined` instead of throwing the last error.
- Retrying non-idempotent POSTs (double orders, double payments).
- No cap on delay or attempts.

## Resources
- [MDN: Using promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises) - promise chaining and error handling basics
- [javascript.info: Async/await](https://javascript.info/async-await) - try/catch with await, the core of the loop
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) - cancelling retries and fetches
- [MDN: 429 Too Many Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429) - Retry-After and rate limits
