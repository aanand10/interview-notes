# Sequential vs parallel

> **In one line:** If calls do not depend on each other, start them all first and then `await Promise.all(...)` so they run in parallel; only await one after another (in series) when each step needs the previous result or the order matters.

## Key points
- A promise starts its work **when you create it**, not when you `await` it. `await` just waits for the result.
- **Sequential**: `const a = await getA(); const b = await getB();` -> total time = A + B.
- **Parallel**: `const [a, b] = await Promise.all([getA(), getB()]);` -> total time = max(A, B).
- **Series on purpose** (one at a time): use `for...of` with `await`, or `reduce` over a promise. Needed when the order matters or the server has rate limits.
- **Middle ground**: limit concurrency (for example 3 at a time) when you have many requests.

## Example
```js
const wait = (ms, v) => new Promise((r) => setTimeout(() => r(v), ms));
const getPrice = (s) => wait(100, `${s}:price`);   // pretend API, 100 ms each

(async () => {
  let t = Date.now();
  const a = await getPrice('AAPL');           // waits 100 ms
  const b = await getPrice('TSLA');           // then another 100 ms
  console.log('sequential', a, b, Date.now() - t, 'ms');

  t = Date.now();
  const [c, d] = await Promise.all([getPrice('AAPL'), getPrice('TSLA')]);  // both start now
  console.log('parallel', c, d, Date.now() - t, 'ms');
})();
```
Real output (node, times rounded):
```text
sequential AAPL:price TSLA:price 200 ms
parallel AAPL:price TSLA:price 100 ms
```

## When to use it
- **Parallel**: a portfolio page needs `/positions`, `/balance` and `/watchlist`. None depend on each other, so fetch together.
- **Sequential**: "get the user, then get their accounts using `user.id`". Step 2 needs step 1.
- **Series with reduce/for...of**: submit a basket of orders in a strict order, or replay queued offline actions one by one.
- **Limited concurrency**: load 200 stock logos, 5 at a time, so you do not flood the network.

## Likely questions
### Rewrite these sequential awaits to run in parallel.
Before:
```js
const user = await getUser();
const news = await getNews();
const quotes = await getQuotes();
```
After:
```js
const [user, news, quotes] = await Promise.all([getUser(), getNews(), getQuotes()]);
```
Another way is to start the promises first and await later:
```js
const userP = getUser();
const newsP = getNews();
const user = await userP;   // both already running
const news = await newsP;
```
Prefer `Promise.all`: if `newsP` rejects while you are still awaiting `userP`, that rejection is briefly unhandled and can log a warning or crash Node.

### Run promises in series with `reduce`.
The trick: the accumulator is a promise. Each step awaits the previous one, then does its own work.
```js
const symbols = ['A', 'B', 'C'];
const results = await symbols.reduce(async (accPromise, s) => {
  const acc = await accPromise;        // wait for all previous steps
  const v = await getPrice(s);         // then run this one
  console.log('  done', s);
  return [...acc, v];
}, Promise.resolve([]));
console.log('reduce series', results);
```
Real output (node, ~300 ms total because each step waits for the one before):
```text
  done A
  done B
  done C
reduce series [ 'A:price', 'B:price', 'C:price' ]
```
Classic `.then` version, often asked:
```js
const runInSeries = (tasks) =>
  tasks.reduce((p, task) => p.then((acc) => task().then((v) => [...acc, v])), Promise.resolve([]));
// tasks is an array of FUNCTIONS that return promises: [() => getPrice('A'), ...]
```
Note: the tasks must be **functions**. If you pass already-created promises, they all started already and it is not really in series.

### Is there a simpler way to run in series?
Yes, `for...of` with `await` is easier to read, and I would use it in real code:
```js
const out = [];
for (const s of symbols) out.push(await getPrice(s));
```

### How do you limit concurrency, say 2 at a time?
Start N "workers" that each pull the next item from a shared index:
```js
async function mapWithLimit(items, limit, fn) {
  const results = new Array(items.length);
  let next = 0;
  async function worker() {
    while (next < items.length) {
      const i = next++;                  // safe: JS is single-threaded
      results[i] = await fn(items[i], i);
    }
  }
  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
  return results;
}
```
With 5 items of 100 ms and limit 2, node printed `[ 'A!', 'B!', 'C!', 'D!', 'E!' ] 300 ms` (3 rounds).

### Why does `await` inside `map` look parallel but `forEach` is broken?
`items.map(async (x) => await f(x))` creates all the promises at once, so it is parallel; wrap it in `Promise.all` to wait for them. `forEach` ignores the promises, so code after it runs before they finish.

## Common mistakes
- Awaiting independent calls one by one ("request waterfall"), which makes the page slow.
- Passing promises (not functions) to a "series" helper, so they already run in parallel.
- `Promise.all` on hundreds of requests with no limit.
- Using `forEach` with `async` callbacks.

## Resources
- [MDN: Using promises (composition)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises#composition) - parallel and sequential composition, including the reduce pattern
- [javascript.info: Promise API](https://javascript.info/promise-api) - Promise.all for parallel work
- [MDN: Array.prototype.reduce](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce) - how the accumulator works
- [javascript.info: Async/await](https://javascript.info/async-await) - await basics before mixing with Promise.all
