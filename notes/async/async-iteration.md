# Async iteration

> **In one line:** `for await...of` loops over values that arrive over time, one at a time, and an async generator (`async function*`) is the easiest way to produce such values, for example chunks of a streaming response.

## Key points
- An **async iterable** has a `[Symbol.asyncIterator]()` method whose `next()` returns a promise of `{ value, done }`.
- `for await (const x of source)` awaits each `next()` before running the loop body. It works only inside an `async` function or a module.
- **Async generator:** `async function*` can both `await` and `yield`. Each `yield` hands one value to the loop and pauses until the loop asks for the next one (this is called back-pressure: the producer waits for the consumer).
- `break` or `return` inside `for await` calls the iterator's `return()`, so cleanup in a `finally` block runs.
- `fetch` response bodies are streams (`response.body` is a [`ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream)). `ReadableStream` is async iterable in Node, Firefox and recent Chrome (Chrome 124+), but Safari support is newer, so the `getReader()` loop is the safe pattern.

## Example
A small async generator (tested with `node`):

```js
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function* ticks(n) {
  for (let i = 1; i <= n; i++) {
    await sleep(10);   // wait for "data"
    yield i;           // hand one value to the loop
  }
}

for await (const t of ticks(3)) console.log('tick', t);
```

Output:

```text
tick 1
tick 2
tick 3
```

Streaming a response line by line (newline-delimited JSON, like an order history export or AI chat stream):

```js
async function* readLines(response) {
  const reader = response.body.pipeThrough(new TextDecoderStream()).getReader();
  let buffer = '';
  try {
    while (true) {
      const { value, done } = await reader.read();
      if (done) break;
      buffer += value;
      const lines = buffer.split('\n');
      buffer = lines.pop();          // keep the unfinished last line
      for (const line of lines) if (line) yield JSON.parse(line);
    }
    if (buffer) yield JSON.parse(buffer);
  } finally {
    reader.releaseLock();            // runs even if the consumer breaks early
  }
}

const res = await fetch('/api/trades/stream');
for await (const trade of readLines(res)) {
  trades.push(trade);                // e.g. a Svelte $state array, UI updates as data arrives
  if (trades.length >= 500) break;   // stop early; finally still runs
}
```

Paginated API as one stream:

```js
async function* allOrders() {
  let cursor = null;
  do {
    const res = await fetch(`/api/orders?cursor=${cursor ?? ''}`);
    const page = await res.json();
    yield* page.items;               // yield each order of this page
    cursor = page.nextCursor;
  } while (cursor);
}
```

## When to use it
- Showing streamed data as it arrives (large CSV export, chat or AI answers, trade history).
- Walking a paginated API without loading every page up front.
- Not for many independent requests you want in parallel: `for await` is one at a time. Use `Promise.all` or a pool for that.

## Likely questions
### What is `for await...of`?
It is a loop for async iterables. On each turn it calls `next()`, awaits the promise, and runs the body with the value. It also works on a normal array of promises, awaiting them in order, though `Promise.all` is usually better there.

### What is an async generator?
A function written `async function*`. It returns an async iterator. Inside you can `await` and `yield`. It is lazy: code runs only when the consumer asks for the next value, so it naturally pauses a fast producer.

### How do you read a streaming `fetch` response?
`response.body` is a `ReadableStream` of bytes. Decode with `TextDecoderStream`, read chunks with `getReader().read()` in a loop, or `for await (const chunk of response.body)` where supported. Chunks do not line up with lines or JSON objects, so buffer and split yourself.

### `for await` vs `Promise.all`?
`for await` is sequential and handles values as they come. `Promise.all` starts everything in parallel and gives all results at once.

## Resources
- [MDN: for await...of](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of) - syntax and cleanup behaviour
- [javascript.info: Async iteration and generators](https://javascript.info/async-iterators-generators) - async generators and pagination example
- [MDN: Using readable streams](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Using_readable_streams) - reading fetch bodies chunk by chunk
