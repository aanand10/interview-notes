# Closures

> **In one line:** A closure is a function that remembers the variables from the place where it was created, even after that outer function has finished running.

## Key points
- Every function in JavaScript keeps a link to its **lexical scope**: the variables around it where it was *written*, not where it is called.
- If an inner function uses an outer variable and the inner function lives on (returned, stored, passed as a callback), that variable stays alive too. That pair is the [closure](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures).
- A closure captures the **variable itself**, not a copy of its value. If the variable changes later, the closure sees the new value.
- Closures give you **private state** without classes: code outside cannot reach the variable, only the functions you return.
- They power everyday tools: event handlers, `debounce`, `memoize`, the module pattern, React hooks, Svelte stores.

## Example
Counter with a private variable:

```js
function createCounter() {
  let count = 0; // private: only the functions below can see it

  return {
    increment: () => ++count,
    decrement: () => --count,
    get value() { return count; },
  };
}

const counter = createCounter();
counter.increment();
counter.increment();
console.log(counter.value); // 2
console.log(counter.count); // undefined - no direct access

const other = createCounter(); // separate closure, separate count
console.log(other.value);   // 0
```

## When to use it
- **Debounce** a search box for stock symbols (the timer id lives in a closure).
- **Memoize** an expensive calculation like a brokerage fee or an indicator.
- **Factory functions**: `createPriceFormatter('INR')` returns a formatter that remembers the currency.
- Event handlers in a component that need the current item, e.g. `onclick={() => buy(stock.id)}`.

## Likely questions
### What is a closure? Give an example.
A closure is when a function remembers the variables of its outer scope even after the outer function has returned. For example, `createCounter` returns `increment`, and `increment` can still read and change `count` later, even though `createCounter` already finished. Each call to `createCounter` makes a new, separate `count`.

### How do you make a private variable with closures?
Declare the variable inside a function and return only the functions that should use it, like the counter above. Nothing outside can read or change `count` directly. Before `#private` class fields existed, this was the main way to get privacy in JavaScript.

### What does this print, and how do you fix it?
```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```
Output (checked with node):
```text
3
3
3
```
- `var` is function-scoped, so there is only **one** `i` shared by all three callbacks.
- The callbacks run after the loop ends, when `i` is already 3.

Fix 1: use `let`, which creates a new `i` for each loop iteration.
```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 0, 1, 2
}
```
Fix 2 (old-school): wrap in an IIFE so each callback gets its own copy.
```js
for (var i = 0; i < 3; i++) {
  ((j) => setTimeout(() => console.log(j), 0))(i); // 0, 1, 2
}
```
Fix 3: `setTimeout(console.log, 0, i)` passes the current value as an argument.

### Where are closures used in real code?
**Debounce**: the timer id is kept in a closure between calls.
```js
function debounce(fn, delay) {
  let timer; // remembered between calls
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}
const search = debounce((q) => fetch(`/api/search?q=${q}`), 300);
```
**Memoize**: the cache lives in a closure.
```js
function memoize(fn) {
  const cache = new Map();
  return (arg) => {
    if (cache.has(arg)) return cache.get(arg);
    const result = fn(arg);
    cache.set(arg, result);
    return result;
  };
}
```
**Module pattern**: an IIFE returns a public API and hides the rest.
```js
const portfolio = (() => {
  const holdings = []; // private
  return {
    add: (h) => holdings.push(h),
    total: () => holdings.reduce((sum, h) => sum + h.qty * h.price, 0),
  };
})();
```
Today ES modules give the same privacy (anything not exported is private), but the idea is the same.

### What is the memory impact of closures?
As long as a closure is reachable, every variable it uses stays in memory and cannot be garbage collected. That is fine for small state, but it causes leaks when a long-lived callback (a global event listener, an interval, a WebSocket handler) captures something big, like a large array of candles or a DOM node that was removed. The fix is to clean up: remove listeners, clear intervals, close sockets when a component is destroyed (in Svelte 5, return a cleanup function from `$effect`), and avoid capturing large objects you do not need. A memoize cache in a closure can also grow forever, so bound its size.

### What is a stale closure?
A callback that captured an old value and keeps using it. In React, a `useEffect` or `setInterval` callback can see the state from the render it was created in. The fix is correct dependencies, functional updates, or reading from a ref.

## Common mistakes
- Thinking a closure stores a snapshot of the value. It stores a reference to the variable.
- Using `var` in loops with async callbacks.
- Forgetting cleanup, so closures keep big data or detached DOM nodes alive.

## Resources
- [MDN: Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures) - definition and classic examples
- [javascript.info: Variable scope, closure](https://javascript.info/closure) - how lexical environments work, step by step
- [javascript.info: Decorators and forwarding](https://javascript.info/call-apply-decorators) - caching wrapper and debounce-style patterns
- [MDN: Memory management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_management) - how garbage collection decides what stays alive
