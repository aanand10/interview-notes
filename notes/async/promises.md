# Promises

> **In one line:** A Promise is an object for a value that will be ready later; it starts **pending** and settles once, either **fulfilled** with a value or **rejected** with an error, and `.then` chains let you run steps in order.

## Key points
- **Three states**: `pending`, `fulfilled`, `rejected`. "Settled" means fulfilled or rejected. Once settled, it never changes again ([MDN: Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)).
- **Every `.then` returns a new promise.** What you return inside the callback decides that new promise:
  - return a plain value -> next `.then` gets that value
  - return a promise -> the chain **waits** for it and gets its result
  - return nothing -> next `.then` gets `undefined`
  - throw -> the new promise is rejected
- **Errors skip ahead** to the nearest `.catch` (or `.then`'s second argument). After a `.catch` returns a value, the chain is back to "success".
- **`.finally(fn)`** runs on both success and failure, gets **no arguments**, and passes the original value or error through. Good for "hide the spinner".
- `.then` callbacks always run **asynchronously** as microtasks, even if the promise is already resolved.

## Example
```js
const wait = (ms, v) => new Promise((resolve) => setTimeout(() => resolve(v), ms));

// 1. Forgetting `return`
Promise.resolve(1)
  .then((x) => { wait(50, x * 2); })     // no return!
  .then((v) => console.log('forgot return ->', v));

// 2. Returning the promise
Promise.resolve(1)
  .then((x) => wait(50, x * 2))          // arrow without braces returns it
  .then((v) => console.log('with return ->', v));

// 3. Error propagation and recovery
Promise.resolve()
  .then(() => { throw new Error('boom'); })
  .then(() => console.log('skipped'))
  .catch((e) => { console.log('caught:', e.message); return 'recovered'; })
  .then((v) => console.log('after catch:', v))
  .finally(() => console.log('finally runs (gets no args)'));

// 4. finally does not change the value
Promise.resolve('price')
  .finally(() => 'ignored')
  .then((v) => console.log('finally passthrough:', v));
```
Real output (node):
```text
forgot return -> undefined
caught: boom
after catch: recovered
finally passthrough: price
finally runs (gets no args)
with return -> 2
```
- `forgot return` prints `undefined` immediately: the chain did not wait for `wait(50)`.
- `skipped` never prints: the error jumps to `.catch`.
- `with return` prints last because the chain really waited 50 ms.

## When to use it
- Wrapping a callback API, like a WebSocket "first message" or `setTimeout`, into something you can `await`.
- Chaining "load account -> load positions -> render portfolio", where each step needs the previous result.
- `.finally` to reset `isSubmitting = false` on an order form whether the order succeeded or not.

## Likely questions
### What are the states of a promise?
Pending, fulfilled and rejected. It starts pending. `resolve(value)` makes it fulfilled, `reject(err)` or a thrown error makes it rejected. It can settle only once; later `resolve` or `reject` calls are ignored.

### What are the chaining rules?
Each `.then` returns a new promise. If the callback returns a value, the next step gets that value. If it returns a promise (or any "thenable"), the chain waits and passes on its result. If it throws, the chain becomes rejected and skips to the next `.catch`.

### What happens if you forget `return` inside `.then`?
The next `.then` runs right away with `undefined`, and it does not wait for the inner async work. Worse, if that inner promise rejects, nobody catches it, so you get an "unhandled rejection". This is the most common promise bug. With braces `{ }` you must write `return`; an arrow without braces returns automatically.

### How does error propagation work?
A rejection or thrown error skips every `.then` success handler until it finds a `.catch`. A `.catch` that returns normally "recovers" the chain. If you want the error to keep going, re-throw it inside `.catch`. One `.catch` at the end handles errors from every step above it.

### What does `.finally` do and what does it receive?
It runs when the promise settles either way. It gets no arguments, because it does not know if it was success or failure. Its return value is ignored and the original result passes through. Exception: if `finally` throws or returns a rejected promise, that error replaces the result.

### What is the difference between `.then(ok, fail)` and `.then(ok).catch(fail)`?
With `.then(ok, fail)`, `fail` does not catch errors thrown inside `ok`. With `.then(ok).catch(fail)`, it does. The second form is usually what you want.

### How do you make a promise from a callback API?
```js
const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
```
The function passed to `new Promise` (the executor) runs synchronously, right away.

## Common mistakes
- Missing `return` in `.then` (see above).
- Nesting `.then` inside `.then` ("promise hell") instead of returning and chaining flat.
- No `.catch` at the end, causing unhandled rejections.
- Wrapping an existing promise in `new Promise(...)` for no reason (the "explicit construction" anti-pattern).

## Resources
- [MDN: Using promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises) - chaining, error handling and common mistakes
- [javascript.info: Promise chaining](https://javascript.info/promise-chaining) - return values vs returned promises
- [javascript.info: Error handling with promises](https://javascript.info/promise-error-handling) - how errors skip to `.catch`
- [MDN: Promise.prototype.finally](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally) - exact `finally` behaviour
