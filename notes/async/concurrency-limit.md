# Concurrency limit (promise pool)

> **In one line:** A promise pool runs many async tasks but never more than K at the same time: K "workers" each pick the next task from a shared list as soon as they finish the previous one.

## Key points
- `Promise.all(urls.map(fetch))` starts **everything at once**. With 500 requests that can hit browser connection limits, server rate limits (HTTP 429) and memory.
- A pool takes **task functions** (`() => fetch(url)`), not promises. A promise has already started; a function only starts when the pool calls it.
- Simple design: start K workers. Each worker loops: take the next index, `await` that task, store the result at the same index, repeat until no tasks are left.
- Results keep the **input order** because we write to `results[i]`, not `push`.
- Decide the error policy: fail fast (like `Promise.all`) or collect every result (like [`Promise.allSettled`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)).

## Example
Tested with `node`.

```js
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function promisePool(tasks, limit) {
  const results = new Array(tasks.length);
  let next = 0;                         // shared index; safe because JS is single-threaded

  async function worker() {
    while (next < tasks.length) {
      const i = next++;                 // grab a task synchronously
      results[i] = await tasks[i]();    // run it; keep input order
    }
  }

  const workers = Array.from({ length: Math.min(limit, tasks.length) }, worker);
  await Promise.all(workers);           // done when every worker runs out of work
  return results;
}

// Test: 5 tasks, limit 2. Track how many run at once.
let running = 0, maxRunning = 0;
const task = (id, ms) => async () => {
  running++; maxRunning = Math.max(maxRunning, running);
  await sleep(ms);
  running--;
  return id;
};

const t0 = Date.now();
const res = await promisePool(
  [task('a', 300), task('b', 100), task('c', 100), task('d', 100), task('e', 100)], 2);
console.log(res, 'max parallel', maxRunning, '~ms', Math.round((Date.now() - t0) / 100) * 100);
console.log(await promisePool([], 3));
```

Output:

```text
[ 'a', 'b', 'c', 'd', 'e' ] max parallel 2 ~ms 400
[]
```

Why 400ms: worker 1 spends 300ms on `a`. Worker 2 does `b`, `c`, `d`, `e` back to back (4 x 100ms). Order is still a, b, c, d, e.

"Settled" version that never throws, so one bad symbol does not lose the rest:

```js
const settled = await promisePool(
  tasks.map((t) => () => t().then(
    (value) => ({ status: 'fulfilled', value }),
    (reason) => ({ status: 'rejected', reason }))),
  3);
```

## When to use it
- Fetching charts or fundamentals for 200 symbols in a watchlist, 5 at a time, to stay under the API rate limit.
- Uploading many KYC documents or files with at most 3 uploads in parallel.
- Prefetching images or routes without blocking the user's real requests.

## Likely questions
### Run N async tasks with at most K in parallel.
I start K workers with `Array.from({ length: K }, worker)`. Each worker is an async loop that takes the next index from a shared counter, awaits that task and saves the result at that index. `Promise.all` on the workers resolves when all tasks are done. The counter is safe because `next++` runs synchronously and JavaScript runs one piece of code at a time.

### Why pass functions instead of promises?
A promise starts working the moment it is created. If I pass `urls.map(fetch)`, all requests have already started and the "limit" does nothing. Passing `() => fetch(url)` lets the pool decide when each one starts.

### What happens if one task fails?
In my version the worker's `await` throws, that worker's promise rejects, and `Promise.all` rejects with that error (fail fast). The other workers keep running in the background though, because promises cannot be cancelled. If I want every result, I wrap each task to return `{ status, value | reason }`, or pass an `AbortSignal` to stop the rest.

### Other ways to build it?
A queue-based version: keep a `running` count; when a task finishes, `running--` and start the next from the queue. Libraries like `p-limit` work this way. It also lets you add tasks over time instead of passing a fixed list.

### Why not chunk into batches of K?
Batching (`for` chunk of K, `await Promise.all(chunk)`) waits for the slowest task in each batch, so slots stay idle. The pool refills a slot as soon as any task finishes, so it is faster.

## Common mistakes
- Passing promises instead of functions.
- Using `results.push()`, which gives results in finish order, not input order.
- Starting `limit` workers when there are fewer tasks than the limit (harmless here thanks to `Math.min`, but worth saying).
- Forgetting the error policy and losing every result because of one failure.

## Resources
- [MDN: Promise.all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) - fail-fast behaviour of the base tool
- [MDN: Promise.allSettled](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled) - the "collect everything" result shape
- [javascript.info: Promise API](https://javascript.info/promise-api) - all, allSettled, race, any side by side
