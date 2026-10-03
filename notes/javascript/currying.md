# Currying

> **In one line:** Currying turns a function that takes many arguments into a chain of functions that each take one argument, so `add(a, b, c)` becomes `add(a)(b)(c)`, and it works because each inner function remembers earlier arguments through closures.

## Key points
- **Currying**: `f(a, b, c)` -> `f(a)(b)(c)`. Each call returns a new function until all arguments are collected.
- It relies on **closures**: each returned function remembers the arguments already given.
- **Partial application** is related but different: you fix *some* arguments now and pass the rest later in one go, e.g. `bind` or a helper.
- A generic `curry(fn)` uses `fn.length` (the number of declared parameters) to know when it has enough arguments.
- Uses: reusable specialised functions (`formatPrice('INR')`), cleaner `map`/`filter` callbacks, function composition.

## Example
Fixed-length currying: `sum(1)(2)(3)(4)`.

```js
const sum = (a) => (b) => (c) => (d) => a + b + c + d;
console.log(sum(1)(2)(3)(4)); // 10
```

## When to use it
- Creating configured helpers once and reusing them:
  ```js
  const formatMoney = (currency) => (amount) =>
    new Intl.NumberFormat('en-IN', { style: 'currency', currency }).format(amount);
  const inr = formatMoney('INR');
  inr(1500); // '₹1,500.00'
  ```
- Event handlers: `const handleSide = (side) => (event) => placeOrder(side, event)` then `onclick={handleSide('BUY')}`.
- Pipelines with `compose`/`pipe`, where each step takes one argument.

## Likely questions
### Implement `sum(1)(2)(3)(4)`.
```js
const sum = (a) => (b) => (c) => (d) => a + b + c + d;
sum(1)(2)(3)(4); // 10
```
Each arrow returns the next function. The last one has all four values thanks to closures.

### Implement infinite currying: `sum(1)(2)(3)()` returns the total.
Keep returning a function; when it is called with no argument, return the running total.
```js
function sum(a) {
  return function (b) {
    if (b === undefined) return a; // empty call: stop and return the total
    return sum(a + b);            // otherwise keep going with the new total
  };
}

console.log(sum(1)(2)(3)()); // 6
console.log(sum(5)());       // 5
```
Output checked with node: `6` and `5`.

A follow-up variant without the empty call uses `valueOf`, so the function turns into a number when used in math:
```js
function sum(...a) {
  const f = (...b) => sum(...a, ...b);
  f.valueOf = () => a.reduce((x, y) => x + y, 0);
  return f;
}
console.log(+sum(1)(2)(3)); // 6
```

### Write a generic `curry(fn)`.
```js
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);      // enough arguments: call the original
    }
    return (...more) => curried.apply(this, [...args, ...more]); // wait for more
  };
}

const add3 = (a, b, c) => a + b + c;
const c = curry(add3);
console.log(c(1)(2)(3)); // 6
console.log(c(1, 2)(3)); // 6
console.log(c(1)(2, 3)); // 6
console.log(c(1, 2, 3)); // 6
```
Output checked with node: all four print `6`. Note that `fn.length` does not count default or rest parameters, so this version does not work for functions like `(a, b = 1) => ...` or `(...nums) => ...`.

### What is the difference between currying and partial application?
Currying always transforms the function into one-argument steps: `f(a)(b)(c)`. Partial application fixes some arguments now and returns a function that takes the *rest* at once: `const addTax = partial(calc, 0.18); addTax(price, qty)`. `fn.bind(null, x)` is built-in partial application. In practice people mix the terms, so I explain both.
```js
const partial = (fn, ...preset) => (...rest) => fn(...preset, ...rest);
const brokerage = (rate, price, qty) => rate * price * qty;
const discountBroker = partial(brokerage, 0.0003);
discountBroker(1500, 10); // 4.5
```

### Why is currying useful?
It lets you build small reusable functions from general ones, avoids repeating the same argument, and fits well with `map` and composition, e.g. `prices.map(applyFee(0.001))`.

## Common mistakes
- Forgetting that `fn.length` ignores default and rest parameters.
- In infinite currying, checking `if (!b)`, which wrongly stops on `sum(1)(0)`. Check `b === undefined`.
- Losing `this` by using arrow functions where the original needs `this` (use `apply(this, ...)`).

## Resources
- [javascript.info: Currying](https://javascript.info/currying-partials) - step-by-step generic `curry`
- [MDN: Function.length](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/length) - how arity is counted
- [MDN: Function.prototype.bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind) - built-in partial application
- [MDN: Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures) - why curried functions remember arguments
