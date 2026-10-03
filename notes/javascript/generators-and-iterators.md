# Generators and iterators

> **In one line:** An iterator is any object with a `next()` method that returns `{ value, done }`, and a generator (`function*`) is an easy way to build one, because `yield` pauses the function and resumes it on the next `next()` call.

## Key points
- **Iterator protocol**: an object with `next()` returning `{ value, done }`.
- **Iterable protocol**: an object with a `[Symbol.iterator]()` method that returns an iterator. Arrays, strings, Maps, Sets are iterable, which is why `for...of`, spread `[...x]` and destructuring work on them.
- **Generator**: `function*` returns a generator object (which is both an iterator and iterable). Each `yield` pauses and hands out a value; the function's local state is kept between calls.
- Generators are **lazy**: they compute values only when asked, so they can even be infinite.
- `next(value)` can send a value *into* the generator; it becomes the result of the paused `yield`.

## Example
Make a custom class iterable with a generator (checked with node):

```js
class PriceRange {
  constructor(from, to) { this.from = from; this.to = to; }
  *[Symbol.iterator]() {
    for (let p = this.from; p <= this.to; p++) yield p;
  }
}
console.log([...new PriceRange(1, 4)]); // [ 1, 2, 3, 4 ]
```

How `yield` pauses and resumes (checked with node):

```js
function* demo() {
  const x = yield 1;        // pause here, hand out 1
  console.log('got', x);    // resumes with the value passed to next()
  return 5;
}
const g = demo();
console.log(g.next());      // { value: 1, done: false }
console.log(g.next('hi'));  // logs "got hi", then { value: 5, done: true }
console.log(g.next());      // { value: undefined, done: true }
```

## When to use it
- **Infinite or lazy sequences**: an id generator `function* ids() { let i = 1; while (true) yield i++; }`.
- **Paginated APIs** with async generators and `for await...of`:
  ```js
  async function* fetchAllTrades() {
    let url = '/api/trades?page=1';
    while (url) {
      const res = await fetch(url);
      const { items, next } = await res.json();
      yield* items;   // hand out each trade one by one
      url = next;
    }
  }
  for await (const trade of fetchAllTrades()) console.log(trade.id);
  ```
- Making your own data structures (order book levels, linked lists) work with `for...of`.
- Historically, libraries like redux-saga and co used generators for async flow; `async/await` replaced most of that.

## Likely questions
### What is the difference between an iterator and an iterable?
An iterable has a `[Symbol.iterator]()` method; calling it gives you an iterator. The iterator has `next()` which returns `{ value, done }`. `for...of` asks the iterable for an iterator and keeps calling `next()` until `done` is true.

### Write an iterator without a generator.
```js
const countdown = {
  from: 3,
  [Symbol.iterator]() {
    let n = this.from;
    return { next: () => (n > 0 ? { value: n--, done: false } : { value: undefined, done: true }) };
  },
};
console.log([...countdown]); // [3, 2, 1]
```

### What does `yield*` do?
It delegates to another iterable, yielding each of its values one by one, e.g. `yield* [1, 2, 3]` or `yield* otherGenerator()`.

### Are plain objects iterable?
No. `for...of` on `{a: 1}` throws a TypeError. Use `Object.entries(obj)` (an array) or `for...in`.

## Resources
- [javascript.info: Generators](https://javascript.info/generators) - `yield`, `next(value)`, iterables
- [javascript.info: Iterables](https://javascript.info/iterable) - `Symbol.iterator` step by step
- [MDN: Iteration protocols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols) - exact rules
- [javascript.info: Async iteration and generators](https://javascript.info/async-iterators-generators) - `for await` and paginated data
