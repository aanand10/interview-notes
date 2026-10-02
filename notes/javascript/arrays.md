# Arrays

> **In one line:** I pick array methods by what they return and whether they mutate: `map` returns a new array while `forEach` returns nothing, `slice` copies while `splice` changes the original, and `sort` sorts as strings unless I pass a comparator.

## Key points
- [`map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map) returns a new array of the same length. [`forEach`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach) returns `undefined` and is only for side effects. Neither can be stopped early with `break` (use `for...of`, `some` or `find` for that).
- [`slice(start, end)`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/slice) returns a copy of a part and does **not** change the array. [`splice(start, deleteCount, ...items)`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/splice) **mutates** the array and returns the removed items.
- [`sort()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort) with no comparator converts items to strings, so `[100, 25, 3]` sorts as `[100, 25, 3]`. Always pass `(a, b) => a - b` for numbers. `sort` also mutates and returns the same array.
- ES2023 added non-mutating copies: [`toSorted`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/toSorted), `toReversed`, `toSpliced` and [`with`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/with).
- Remove duplicates with `[...new Set(arr)]`. Group with [`Object.groupBy`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/groupBy) / [`Map.groupBy`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map/groupBy) (ES2024) or [`reduce`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce).

## Example
```js
const trades = [
  { sym: 'AAPL', side: 'BUY', qty: 10 },
  { sym: 'TSLA', side: 'SELL', qty: 5 },
  { sym: 'AAPL', side: 'SELL', qty: 3 },
];

// map vs forEach
[1, 2, 3].map(x => x * 2);     // [2, 4, 6]
[1, 2, 3].forEach(x => x * 2); // undefined

// slice vs splice
const arr = [1, 2, 3, 4, 5];
arr.slice(1, 3);   // [2, 3]   arr is still [1, 2, 3, 4, 5]
arr.splice(1, 2);  // [2, 3]   arr is now [1, 4, 5]

// sort trap
const prices = [100, 25, 3, 1000];
prices.toSorted();                // [100, 1000, 25, 3]  (string order!)
prices.toSorted((a, b) => a - b); // [3, 25, 100, 1000]
// prices itself is unchanged: [100, 25, 3, 1000]
['b', 'a', 'C'].sort();                               // ['C', 'a', 'b']
['b', 'a', 'C'].sort((a, b) => a.localeCompare(b));   // ['a', 'b', 'C']

// remove duplicates
[...new Set([3, 1, 3, 2, 1])];                        // [3, 1, 2]
[1, 2, 1, 3].filter((x, i, a) => a.indexOf(x) === i); // [1, 2, 3]  (O(n^2), older way)
// unique objects by key (last one wins)
[...new Map(trades.map(t => [t.sym, t])).values()];
// [{ sym:'AAPL', side:'SELL', qty:3 }, { sym:'TSLA', side:'SELL', qty:5 }]

// group by
Object.groupBy(trades, t => t.sym);
// { AAPL: [ {..BUY..}, {..SELL..} ], TSLA: [ {..} ] }   (null-prototype object)

// group by with reduce (works everywhere)
trades.reduce((acc, t) => {
  (acc[t.side] ??= []).push(t.sym);
  return acc;
}, {});
// { BUY: ['AAPL'], SELL: ['TSLA', 'AAPL'] }
```

## When to use it
- **Watchlist:** `map` to render rows, `filter` for search, `toSorted` for "sort by % change" without mutating state.
- **Order history:** `Object.groupBy(orders, o => o.status)` for "Open / Filled / Cancelled" tabs.
- **Deduping live ticks** by symbol: a `Map` keyed by symbol keeps only the latest tick.
- **Svelte 5:** mutating methods like `push` and `splice` on a `$state` array are tracked, so they update the UI. With `$state.raw`, you must replace the array (`list = list.toSorted(...)`).

## Likely questions
### What is the difference between `map` and `forEach`?
`map` builds and returns a new array from what your callback returns, so it is for transforming data. `forEach` just runs the callback and returns `undefined`, so it is for side effects like logging. Using `map` without using its result is a code smell. Neither supports `break` or works well with `await` inside.

### What is the difference between `slice` and `splice`?
`slice` is read-only: it returns a shallow copy of a range and leaves the original alone. `splice` edits the original in place: it can remove, insert or replace items, and returns the removed ones. If I need a non-mutating version of `splice`, I use `toSpliced`.

### Why does `[10, 1, 5, 100].sort()` give `[1, 10, 100, 5]`?
Without a comparator, `sort` converts each item to a string and compares UTF-16 code units, so `'100'` comes before `'5'`. Pass `(a, b) => a - b`: a negative result puts `a` first, positive puts `b` first, zero keeps the order. Sort is stable since ES2019, so equal items keep their original order.

### How do you remove duplicates from an array?
For primitives, `[...new Set(arr)]` is O(n) and keeps first-seen order. For objects, `Set` compares references, so I dedupe by a key: put items in a `Map` keyed by `id`, or use `filter` with a `Set` of seen ids.

```js
const seen = new Set();
const unique = trades.filter(t => !seen.has(t.sym) && seen.add(t.sym));
```

### How do you group an array by a key?
Modern: `Object.groupBy(items, fn)` returns an object of arrays, and `Map.groupBy` returns a `Map` (useful when keys are objects). Before ES2024, I write a `reduce` that pushes each item into `acc[key]`. Mention that `Object.groupBy` returns a null-prototype object, so it has no `hasOwnProperty` method.

### What does `['1', '2', '3'].map(parseInt)` return?
`[1, NaN, NaN]`. `map` passes `(value, index)`, so it calls `parseInt('2', 1)` and `parseInt('3', 2)`. Radix 1 is invalid and `'3'` is not a binary digit. Fix: `.map(Number)` or `.map(s => parseInt(s, 10))`.

## Common mistakes
- Forgetting that `sort`, `reverse` and `splice` mutate, which breaks state updates that rely on a new reference.
- Calling `sort()` on numbers with no comparator.
- Using `forEach` with `async` callbacks and expecting it to wait. Use `for...of` or `Promise.all(arr.map(...))`.
- Using `delete arr[i]`, which leaves a hole. Use `splice` or `filter`.

## Resources
- [javascript.info: Array methods](https://javascript.info/array-methods) - all common methods with tasks
- [MDN: Array.prototype.sort](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort) - default string order and comparator rules
- [MDN: Array.prototype.splice](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/splice) - all the argument forms
- [MDN: Object.groupBy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/groupBy) - modern grouping
