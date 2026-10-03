# Higher-order functions

> **In one line:** A higher-order function is a function that takes another function as an argument or returns a function, like `map`, `filter` and `debounce`, and it works because functions in JavaScript are values.

## Key points
- JavaScript has **first-class functions**: you can store them in variables, pass them in, and return them.
- **Takes a function**: `map`, `filter`, `reduce`, `forEach`, `sort`, `addEventListener`, `setTimeout`.
- **Returns a function**: `debounce`, `throttle`, `memoize`, `curry`, `bind`, `compose`, `pipe`.
- **`compose(f, g)(x)` = `f(g(x))`** runs right to left. **`pipe(f, g)(x)` = `g(f(x))`** runs left to right (easier to read).
- They make code declarative ("what", not "how") and reusable.

## Example
```js
const holdings = [
  { symbol: 'TCS', qty: 5, price: 4000 },
  { symbol: 'INFY', qty: 10, price: 1500 },
  { symbol: 'WIPRO', qty: 0, price: 500 },
];

const value = holdings
  .filter((h) => h.qty > 0)            // takes a function
  .map((h) => h.qty * h.price)          // takes a function
  .reduce((sum, v) => sum + v, 0);      // takes a function

console.log(value); // 35000
```

A function that returns a function:
```js
function debounce(fn, delay) {
  let timer;
  return function (...args) {           // returns a new function
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}
const searchSymbol = debounce((q) => fetch(`/api/search?q=${q}`), 300);
```

## When to use it
- Data transforms on watchlists and order history (`filter` / `map` / `reduce`).
- Wrapping behaviour around a function without changing it: `debounce` a search input, `throttle` a scroll handler, `memoize` a fee calculation, add logging or retries.
- Building a data pipeline with `pipe`, e.g. clean -> normalise -> format a price.

## Likely questions
### What is a higher-order function? Give examples.
A function that takes a function as input or returns one. `map` and `filter` take a callback; `debounce` takes a function and returns a new, wrapped function. They work because functions are values in JavaScript.

### Write `compose` and `pipe`.
```js
const compose = (...fns) => (x) => fns.reduceRight((acc, fn) => fn(acc), x);
const pipe = (...fns) => (x) => fns.reduce((acc, fn) => fn(acc), x);

const inc = (x) => x + 1;
const double = (x) => x * 2;

console.log(compose(inc, double)(5)); // 11 -> double first (10), then inc
console.log(pipe(inc, double)(5));    // 12 -> inc first (6), then double
```
Output checked with node: `11` and `12`. `compose` uses `reduceRight` so the last function runs first; `pipe` uses `reduce` so it runs in reading order.

A real pipe:
```js
const formatPrice = pipe(
  (s) => s.trim(),
  Number,
  (n) => n.toFixed(2),
  (s) => `₹${s}`,
);
formatPrice(' 1499.5 '); // '₹1499.50'
```

### How would you support async functions in `pipe`?
Reduce over a promise:
```js
const pipeAsync = (...fns) => (x) => fns.reduce((p, fn) => p.then(fn), Promise.resolve(x));
```

### Is `forEach` a higher-order function? Difference from `map`?
Yes, it takes a callback. `forEach` returns `undefined` and is for side effects; `map` returns a new array of the same length. Use `map` when you need the result.

### Why are higher-order functions useful?
They remove repeated loop code, separate "what to do" from "how to loop", and let you add behaviour (debounce, cache, retry, logging) by wrapping instead of editing the original function.

## Common mistakes
- Using `map` just to loop without using the result.
- Mixing up order: `compose` is right-to-left, `pipe` is left-to-right.
- Losing `this` inside wrappers like `debounce`; use `fn.apply(this, args)`.

## Resources
- [MDN: Array.prototype.map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map) - the classic example
- [MDN: Array.prototype.reduce](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce) - the core of `compose` and `pipe`
- [javascript.info: Decorators and forwarding](https://javascript.info/call-apply-decorators) - functions that wrap functions
- [MDN: First-class function](https://developer.mozilla.org/en-US/docs/Glossary/First-class_Function) - why this is possible
