# Promise combinators

> **In one line:** `Promise.all` waits for all and fails fast on the first error, `allSettled` waits for all and never fails, `race` settles with whichever finishes first (success or error), and `any` gives the first success and only fails if everything fails.

## Key points
| Method | Resolves when | Rejects when | Result |
| --- | --- | --- | --- |
| [`Promise.all`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) | all fulfill | **any one** rejects (fail-fast) | array of values, in input order |
| [`Promise.allSettled`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled) | all settle | never | array of `{status, value}` / `{status, reason}` |
| [`Promise.race`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/race) | first to settle fulfills | first to settle rejects | that one value or error |
| [`Promise.any`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/any) | first fulfills | **all** reject | first value, or `AggregateError` |

- **Fail-fast** means `Promise.all` rejects as soon as one input rejects. The others **keep running**; their results are just ignored. Promises cannot be cancelled by `all` (use `AbortController` for that).
- Results keep **input order**, not finish order.
- Empty input: `all([])` and `allSettled([])` resolve to `[]`; `any([])` rejects with `AggregateError`; `race([])` stays pending forever.

## Example
```js
const wait = (ms, v, fail) =>
  new Promise((res, rej) => setTimeout(() => (fail ? rej(new Error(v)) : res(v)), ms));

(async () => {
  try { await Promise.all([wait(100, 'A'), wait(50, 'B', true), wait(200, 'C')]); }
  catch (e) { console.log('all rejected with:', e.message); }

  const s = await Promise.allSettled([wait(100, 'A'), wait(50, 'B', true)]);
  console.log('allSettled:', s.map((r) => r.status === 'fulfilled' ? r.value : 'rejected: ' + r.reason.message));

  console.log('race:', await Promise.race([wait(100, 'slow'), wait(50, 'fast')]));
  try { await Promise.race([wait(100, 'ok'), wait(50, 'race err', true)]); }
  catch (e) { console.log('race rejected:', e.message); }

  console.log('any:', await Promise.any([wait(50, 'x', true), wait(100, 'mirror-2')]));
  try { await Promise.any([wait(10, 'e1', true), wait(20, 'e2', true)]); }
  catch (e) { console.log('any all failed:', e.constructor.name, e.errors.map((x) => x.message)); }

  console.log('all([]):', await Promise.all([]));
})();
```
Real output (node):
```text
all rejected with: B
allSettled: [ 'A', 'rejected: B' ]
race: fast
race rejected: race err
any: mirror-2
any all failed: AggregateError [ 'e1', 'e2' ]
all([]): []
```
- `all` rejects at 50 ms with B, without waiting for C.
- `race` takes the first settled, even if it is an error.
- `any` ignores the failure and waits for the first success.

## When to use it (real use cases)
- **`all`**: a dashboard needs account, positions and balance before it can render anything. If one fails, show an error state.
- **`allSettled`**: refresh 20 watchlist quotes; show the ones that worked and a "stale" badge on the ones that failed. Also "send analytics to 3 endpoints, do not care if one fails".
- **`race`**: a timeout. `Promise.race([fetchQuote(), timeoutAfter(3000)])`. (Today `AbortSignal.timeout` is better for fetch because it also cancels the request.)
- **`any`**: fetch the same price from two mirror servers or CDNs and use whichever answers first successfully.

## Likely questions
### What is the difference between all, allSettled, race and any?
`all` needs everyone to succeed and fails on the first error. `allSettled` waits for everyone and reports each result, it never rejects. `race` follows whoever finishes first, good or bad. `any` follows the first success and only rejects with an `AggregateError` when all fail.

### What does "fail-fast" mean for Promise.all? Do the other requests stop?
`Promise.all` rejects as soon as one promise rejects, so your `catch` runs early. But the other requests do not stop. Promises have no built-in cancel. If you want to stop them, share an `AbortController` and call `abort()` in the `catch`.

### How do you get partial results if one call fails?
Use `allSettled` and filter:
```js
const results = await Promise.allSettled(symbols.map(getQuote));
const ok = results.filter((r) => r.status === 'fulfilled').map((r) => r.value);
```
Or, with `all`, add `.catch(() => null)` to each promise so none of them reject.

### race vs any?
`race` settles on the first promise to settle, even if it rejected. `any` skips rejections and waits for the first fulfillment. For "fastest mirror", `any` is right; with `race` one fast failure would break it.

### Implement a timeout with race.
```js
const timeout = (ms) => new Promise((_, rej) => setTimeout(() => rej(new Error('Timeout')), ms));
const quote = await Promise.race([getQuote('AAPL'), timeout(3000)]);
```
Mention the weakness: the slow request keeps running. Prefer `fetch(url, { signal: AbortSignal.timeout(3000) })`.

## Common mistakes
- Thinking `Promise.all` cancels the rest on failure.
- Using `race` when you meant `any`.
- Passing functions instead of promises: `Promise.all([getA, getB])` must be `Promise.all([getA(), getB()])`.
- Firing 500 requests at once with `all`; limit concurrency instead.

## Resources
- [javascript.info: Promise API](https://javascript.info/promise-api) - all four combinators with examples
- [MDN: Promise.all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) - fail-fast and ordering rules
- [MDN: Promise.allSettled](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled) - result object shape
- [MDN: Promise.any](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/any) - first success and AggregateError
