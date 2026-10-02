# `Array.prototype.map` / `filter` / `forEach`

> **In one line:** All three loop over the array with a callback `(value, index, array)`, skip holes, and accept an optional `thisArg`; `map` returns a new array of the same length, `filter` returns a new array of the items that passed, and `forEach` returns `undefined`.

## Requirements

What the real methods do ([map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map), [filter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter), [forEach](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach)):

- Signature: `arr.method(callback, thisArg?)`. The callback is called as `callback.call(thisArg, value, index, array)`.
- Inside the polyfill, `this` is the array the method was called on. That is why the polyfill must be a normal `function`, not an arrow.
- **Holes are skipped** (checked with `i in arr`). `map` keeps the hole in the output at the same index; `filter` just leaves it out.
- **Never mutate the original.** `map` and `filter` build a brand new array. (The callback itself could still mutate, that is the caller's choice.)
- The length is read once at the start, so items pushed during the loop are not visited.
- Throws `TypeError` if the callback is not a function.
- `forEach` always returns `undefined` and cannot be stopped early except by throwing.

## Implementation

```js
function checkArgs(self, callback) {
  if (self == null) throw new TypeError('called on null or undefined');
  if (typeof callback !== 'function') throw new TypeError(callback + ' is not a function');
}

Array.prototype.myMap = function (callback, thisArg) {
  checkArgs(this, callback);
  const arr = Object(this);              // `this` is the array (or array-like)
  const len = arr.length >>> 0;          // read length once
  const result = new Array(len);         // same length, holes stay holes
  for (let i = 0; i < len; i++) {
    if (i in arr) {                      // skip holes
      result[i] = callback.call(thisArg, arr[i], i, arr);
    }
  }
  return result;                         // new array, original untouched
};

Array.prototype.myFilter = function (callback, thisArg) {
  checkArgs(this, callback);
  const arr = Object(this);
  const len = arr.length >>> 0;
  const result = [];
  for (let i = 0; i < len; i++) {
    if (i in arr) {
      const value = arr[i];              // read before the callback runs
      if (callback.call(thisArg, value, i, arr)) result.push(value);
    }
  }
  return result;
};

Array.prototype.myForEach = function (callback, thisArg) {
  checkArgs(this, callback);
  const arr = Object(this);
  const len = arr.length >>> 0;
  for (let i = 0; i < len; i++) {
    if (i in arr) callback.call(thisArg, arr[i], i, arr);
  }
  // no return: always undefined
};
```

## Edge cases

- **`this` is the array** - we use a normal `function` so `this` points at the array the method is called on. `Object(this)` also lets it work on strings and array-likes via `.call`.
- **Holes** - `i in arr` skips them. `myMap` uses `new Array(len)` so the hole stays a hole in the output, like native (`[ 10, <1 empty item>, 30 ]`).
- **`thisArg`** - we pass it as the first argument of `callback.call`. If the callback is an arrow function, `thisArg` is ignored, because arrows use the `this` from where they were written.
- **Not mutating** - we only read from `arr` and write into `result`.
- **Array grows during the loop** - `len` was saved before the loop, so new items are not visited.
- **Falsy values** - `filter` keeps `undefined` or `null` items if the callback returns truthy for them. We test the callback's return value, not the item.
- **Non-function callback** - throws `TypeError` like native.

## Test it

```js
const prices = [100, 250, 75];
console.log(prices.myMap((p) => p * 2), prices);
// [ 200, 500, 150 ] [ 100, 250, 75 ]   -> original not mutated

console.log([1, , 3].myMap((x) => x * 10));   // [ 10, <1 empty item>, 30 ] (same as native)
console.log([1, , 3, 4].myFilter((x) => x > 0)); // [ 1, 3, 4 ]

const fx = { rate: 83 };
console.log([1, 2].myMap(function (usd) { return usd * this.rate; }, fx)); // [ 83, 166 ]
console.log([1, 2].myMap((usd) => usd * this?.rate, fx));                 // [ NaN, NaN ] arrow ignores thisArg

let visits = 0;
[1, , 3].myForEach(() => visits++);
console.log(visits, [1, 2].myForEach((x) => x)); // 2 undefined

const live = [1, 2, 3];
live.myForEach((v, i, a) => { if (i === 0) a.push(99); });
console.log(live); // [ 1, 2, 3, 99 ]  (99 added but not visited)

console.log(Array.prototype.myMap.call('abc', (c) => c.toUpperCase())); // [ 'A', 'B', 'C' ]
try { [1].myMap(null); } catch (e) { console.log(e.name + ': ' + e.message); }
// TypeError: null is not a function
```

Outputs checked with Node 24.

## When to use it

- `map`: turn API quote objects into view rows, e.g. `quotes.map(q => ({ ...q, change: q.ltp - q.prevClose }))`.
- `filter`: show only the watchlist symbols that match a search or only open orders.
- `forEach`: side effects only, like sending each order to an analytics queue. If you need a result, use `map`/`filter`/`reduce` instead.
- In Svelte 5, a `$derived(watchlist.filter(...).map(...))` gives a new array each time, which is exactly the non-mutating style reactivity likes.

## Likely follow-ups

### Why can't you write the polyfill as an arrow function?
Arrow functions don't get their own `this`. They take `this` from the outer scope, so inside the polyfill `this` would not be the array. A normal `function` gets `this` set to whatever is left of the dot: `prices.myMap(...)` makes `this === prices`.

### What is `thisArg` and how do you support it?
It is an optional second argument that becomes `this` inside the callback. I support it with `callback.call(thisArg, value, i, arr)`. It only works for normal functions; arrow callbacks ignore it.

### How do you handle sparse arrays?
I check `i in arr` before calling the callback. For `map`, I create the result with `new Array(len)` and only assign present indexes, so the hole is kept at the same position. For `filter`, the hole is simply never pushed.

### Does map mutate the original array?
No. It returns a new array of the same length. But if the items are objects and the callback mutates them (`item.price = 0`), those shared objects change. `map` makes a new array, not deep copies.

### How do you break out of forEach?
You can't with `break` or `return`. `return` only exits the current callback. Use `for...of`, `some`, or `every` if you need to stop early.

### What does `['1', '2', '3'].map(parseInt)` return?
`[1, NaN, NaN]`. `map` passes `(value, index)` and `parseInt` reads the index as the radix: `parseInt('2', 1)` and `parseInt('3', 2)` are `NaN`.

## Common mistakes

- Returning the original array from `myMap` or pushing into `this`.
- Using `result.push` in `map`: it closes the holes and shifts indexes.
- Calling `callback(arr[i], i, arr)` without `.call(thisArg, ...)`, so `thisArg` is lost.
- Forgetting the `index` and `array` arguments.

## Resources

- [MDN: Array.prototype.map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map) - thisArg, sparse arrays, the parseInt trap
- [MDN: Array.prototype.filter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter) - exact behaviour
- [MDN: Array.prototype.forEach](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach) - why you can't break, return value
- [LeetCode 2635: Apply Transform Over Each Element](https://leetcode.com/problems/apply-transform-over-each-element-in-array/) - map practice
- [LeetCode 2634: Filter Elements from Array](https://leetcode.com/problems/filter-elements-from-array/) - filter practice
