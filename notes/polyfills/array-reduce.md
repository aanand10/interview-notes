# `Array.prototype.reduce`

> **In one line:** `reduce` walks the array once and folds every element into a single "accumulator" value, using an initial value if you give one, or the first real element if you don't.

## Requirements

What the real [`Array.prototype.reduce`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce) does:

- Signature: `arr.reduce(callback, initialValue?)`.
- The callback gets 4 arguments: `(accumulator, currentValue, currentIndex, array)`.
- **With** an initial value: the accumulator starts as that value and the loop starts at index 0.
- **Without** an initial value: the accumulator starts as the first element and the loop starts at the next index.
- Empty array and no initial value: throws `TypeError: Reduce of empty array with no initial value`.
- It **skips holes** in sparse arrays (an index that was never assigned). It checks this with the [`in` operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/in).
- Throws `TypeError` if the callback is not a function.
- The length is read once at the start, so items pushed during the loop are not visited.
- It does not mutate the array by itself.

## Implementation

```js
Array.prototype.myReduce = function (callback, ...rest) {
  if (this == null) {
    throw new TypeError('Array.prototype.myReduce called on null or undefined');
  }
  if (typeof callback !== 'function') {
    throw new TypeError(callback + ' is not a function');
  }

  const arr = Object(this);          // works on array-likes too
  const len = arr.length >>> 0;      // safe unsigned length, read once
  let i = 0;
  let acc;

  if (rest.length > 0) {
    // An initial value was passed (even if it is undefined)
    acc = rest[0];
  } else {
    // No initial value: find the first real element (skip holes)
    while (i < len && !(i in arr)) i++;
    if (i >= len) {
      throw new TypeError('Reduce of empty array with no initial value');
    }
    acc = arr[i];
    i++;
  }

  for (; i < len; i++) {
    if (i in arr) {                   // skip holes in sparse arrays
      acc = callback(acc, arr[i], i, arr);
    }
  }
  return acc;
};
```

Why `...rest` and not `initialValue === undefined`? Because `[1, 2].reduce(fn, undefined)` is a real call **with** an initial value. Checking `rest.length` (or `arguments.length > 1`) tells "not passed" apart from "passed as `undefined`".

## Edge cases

- **No initial value** - we take the first present element as the accumulator and start the loop from the next index.
- **Empty array, no initial value** - the `while` loop finds nothing, so we throw the same `TypeError` as the native method.
- **Empty array with initial value** - the loop never runs, we return the initial value. No error.
- **Single element, no initial value** - we return that element; the callback is never called.
- **Sparse arrays** like `[1, , 3]` - `i in arr` is `false` for the hole, so the callback is skipped. Holes before the first element are also skipped when picking the starting accumulator.
- **`undefined` passed as initial value** - treated as "given", same as native.
- **Array-likes** (`{ length: 2, 0: 'a', 1: 'b' }`) - `Object(this)` plus `length >>> 0` makes `Array.prototype.myReduce.call(obj, fn)` work.
- **Bad callback** - throws `TypeError` before doing anything.

## Test it

```js
console.log([1, 2, 3, 4].myReduce((a, b) => a + b));      // 10
console.log([1, 2, 3, 4].myReduce((a, b) => a + b, 10));  // 20
console.log([].myReduce((a, b) => a + b, 0));             // 0
try { [].myReduce((a, b) => a + b); }
catch (e) { console.log(e.name + ': ' + e.message); }
// TypeError: Reduce of empty array with no initial value

console.log([, , 5].myReduce((a, b) => a + b));           // 5  (holes skipped)
console.log([1, , 3].myReduce((acc, v, i) => { acc.push(i); return acc; }, []));
// [ 0, 2 ]  (index 1 is a hole, never visited - same as native)

console.log([5].myReduce(() => 'never called'));          // 5
console.log([1, 2].myReduce((acc) => acc, undefined));    // undefined
console.log(Array.prototype.myReduce.call({ length: 2, 0: 'a', 1: 'b' }, (a, b) => a + b)); // ab

['x', 'y'].myReduce((acc, value, index, array) => {
  console.log(acc, value, index, array);                  // x y 1 [ 'x', 'y' ]
  return acc + value;
});
```

All outputs above were checked with Node 24, and they match the native `reduce`.

## When to use it

- Summing a portfolio: `holdings.reduce((sum, h) => sum + h.qty * h.price, 0)`.
- Grouping orders by status into an object, or building a lookup map `{ [symbol]: quote }` from an array of quotes.
- Always pass an initial value in real code. It makes the type of the accumulator clear and avoids the empty-array crash when a user's watchlist is empty.

## Likely follow-ups

### What happens if you call reduce on an empty array without an initial value?
It throws `TypeError: Reduce of empty array with no initial value`. There is no first element to use as the accumulator, so the spec says throw. If you pass an initial value, it just returns that value and never calls the callback.

### What arguments does the callback get?
Four: the accumulator, the current value, the current index, and the original array. Most people only use the first two. The index is handy for things like "skip the first row", and the array lets you look at neighbours.

### How does your polyfill handle sparse arrays?
I check `i in arr` before calling the callback. A hole has no property at that index, so `in` returns `false` and I skip it. A slot that holds `undefined` explicitly is not a hole, so it is visited. This matches native `reduce`.

### Why not just check `initialValue === undefined`?
Because someone can pass `undefined` on purpose. The native method looks at how many arguments were passed, not at the value. I use a rest parameter and check `rest.length > 0`.

### Implement `map` or `filter` using `reduce`.
```js
const mapViaReduce = (arr, fn) => arr.reduce((out, v, i, a) => { out.push(fn(v, i, a)); return out; }, []);
const filterViaReduce = (arr, fn) => arr.reduce((out, v, i, a) => (fn(v, i, a) && out.push(v), out), []);
```

### What is `reduceRight`?
Same idea but walks from the last index to the first. See [MDN reduceRight](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduceRight). It is the natural way to write `compose`.

## Common mistakes

- Using an arrow function for the polyfill. Arrow functions don't have their own `this`, so `this` will not be the array.
- Starting the loop at index 0 when no initial value is given, which processes the first element twice.
- Forgetting to `return acc` inside the callback, so the accumulator becomes `undefined` on the next step.
- Using `for...of` or `forEach` inside the polyfill: `for...of` visits holes as `undefined`, so you lose the sparse-array behaviour.

## Resources

- [MDN: Array.prototype.reduce](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce) - exact behaviour and edge cases
- [MDN: Sparse arrays (Indexed collections)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Indexed_collections) - what holes are and which methods skip them
- [javascript.info: Array methods](https://javascript.info/array-methods) - friendly walk-through of reduce
- [LeetCode 2626: Array Reduce Transformation](https://leetcode.com/problems/array-reduce-transformation/) - practice problem
