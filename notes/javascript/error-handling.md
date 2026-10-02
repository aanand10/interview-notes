# Error handling

> **In one line:** I wrap risky code in `try/catch`, use `finally` for cleanup that must always run, throw custom `Error` subclasses so callers can tell error types apart, and in async code I `await` inside `try` (or use `.catch`) because a `try` cannot catch errors from code that runs later.

## Key points
- [`try...catch...finally`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch): `catch` runs if `try` throws; `finally` always runs, even after `return`. A `return` inside `finally` overrides the earlier return or throw, so avoid it.
- `catch` can skip the variable: `catch { ... }` (ES2019). The caught value can be anything, not only an `Error`, so check it.
- **Custom errors**: `class ApiError extends Error`, set `this.name`, add fields like `status`. Use the [`cause`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/cause) option to wrap a low-level error without losing it.
- **Async**: a rejected promise is the async version of `throw`. With `async/await`, a normal `try/catch` works, but only if you `await` the promise inside the `try`. Errors thrown inside `setTimeout` or event callbacks are not caught by an outer `try`.
- Unhandled rejections fire the [`unhandledrejection`](https://developer.mozilla.org/en-US/docs/Web/API/Window/unhandledrejection_event) event in browsers (and crash the process by default in Node 15+). Use it for logging, not as the main handler.

## Example
```js
class ApiError extends Error {
  constructor(message, { status, cause } = {}) {
    super(message, { cause }); // Error supports { cause } since ES2022
    this.name = 'ApiError';
    this.status = status;
  }
}

async function placeOrder(order) {
  let res;
  try {
    res = await fetch('/api/orders', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(order),
    });
  } catch (err) {
    // fetch only rejects on network failure, not on 4xx/5xx
    throw new ApiError('Network error, please retry', { cause: err });
  }
  if (!res.ok) throw new ApiError('Order rejected', { status: res.status });
  return res.json();
}

let submitting = true;
try {
  await placeOrder({ sym: 'AAPL', qty: 10 });
} catch (e) {
  if (e instanceof ApiError && e.status === 422) {
    console.log('Show validation message');
  } else {
    throw e; // don't swallow errors you don't understand
  }
} finally {
  submitting = false; // always re-enable the button
}
```

Verified edge cases (Node 24):

```js
function f() { try { return 'try'; } finally { console.log('finally runs'); } }
f(); // logs 'finally runs', returns 'try'

function g() { try { return 'try'; } finally { return 'finally'; } }
g(); // 'finally'  (finally's return wins)

function noAwait() {
  try { return placeThatRejects(); }  // returned without await
  catch { console.log('never'); }     // NOT run: the rejection happens later
}
```

## When to use it
- **Order placement:** show a clear message for a known `422` (bad quantity), retry for network errors, and send unknown errors to monitoring (for example Sentry).
- **Loading states:** clear the spinner in `finally`, so it never gets stuck.
- **Dashboards:** load positions, news and charts with `Promise.allSettled`, so one failed widget doesn't blank the whole page.
- **SvelteKit:** use `error(404, 'Not found')` from `@sveltejs/kit` in load functions and `+error.svelte` pages for UI; Svelte 5 also has `<svelte:boundary>` for component errors.

## Likely questions
### How does `try/catch/finally` work?
Code in `try` runs; if anything throws, control jumps to `catch` with the thrown value. `finally` runs no matter what: after a normal finish, after a catch, or even after a `return` in `try`. I use `finally` for cleanup like hiding a loader or closing a connection. One trap: a `return` in `finally` replaces the earlier result and can hide errors.

### How and why do you create a custom error class?
I extend `Error`, call `super(message, { cause })`, set `this.name` and add useful fields like `status` or `code`. Callers can then use `instanceof` to handle each kind differently, and logs show a clear name. `cause` keeps the original low-level error for debugging.

```js
class InsufficientFundsError extends Error { name = 'InsufficientFundsError'; }
```

### How do you handle errors in async code?
With `async/await`, wrap `await` calls in `try/catch`. With raw promises, add `.catch()` at the end of the chain. The key rule is that `try` only catches what happens synchronously or what you `await`. If you return a promise without `await` in the `try`, or throw inside a `setTimeout` callback, the outer `try` won't see it. For many tasks, `Promise.all` rejects on the first failure, `Promise.allSettled` gives you every result, and `Promise.any` rejects with an [`AggregateError`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/AggregateError) only if all fail.

### Does `fetch` throw on a 404 or 500?
No. `fetch` only rejects on network failure or abort. For HTTP errors you must check `res.ok` or `res.status` and throw yourself.

### What happens to an unhandled promise rejection?
The browser fires `unhandledrejection` on `window` and logs it in the console. Node 15+ crashes the process by default. I add a global listener to report them, but every promise should still have its own handling.

### Should you catch every error?
No. Catch only what you can handle (show a message, retry, fall back). Re-throw the rest so it reaches a boundary or global handler. Empty catch blocks hide bugs.

## Common mistakes
- Forgetting `await` inside `try`, so the rejection escapes.
- Assuming `fetch` rejects on 4xx/5xx.
- Using `forEach` with async callbacks; errors and timing are lost.
- Throwing strings (`throw 'failed'`): no stack trace, no `instanceof`.
- Swallowing errors with an empty `catch`.

## Resources
- [javascript.info: Error handling, try...catch](https://javascript.info/try-catch) - basics including finally
- [javascript.info: Custom errors](https://javascript.info/custom-errors) - extending Error and wrapping
- [javascript.info: Error handling with promises](https://javascript.info/promise-error-handling) - async errors and unhandled rejections
- [MDN: Error](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error) - built-in error types and `cause`
