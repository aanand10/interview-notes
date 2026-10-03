# Callbacks

> **In one line:** A callback is a function you pass to another function so it can call you back later, and it was JavaScript's first way to handle async work, but deep nesting and handing control to other code led us to promises.

## Key points
- A **callback** is just a function passed as an argument: `button.addEventListener('click', onClick)`, `setTimeout(fn, 100)`, `arr.map(fn)`. Some run now (sync, like `map`), some run later (async, like `setTimeout`).
- Node's style is the **error-first callback**: `cb(err, result)`. If `err` is not `null`, something failed.
- **Callback hell** (the "pyramid of doom"): each async step needs the result of the one before, so code nests deeper and deeper, and error handling is repeated at every level.
- **Inversion of control**: when you pass a callback to someone else's code, *they* decide when and how often it runs. It might be called twice, never, too early (sync), or have its errors swallowed. Promises fix this: a promise settles only once and `.then` always runs async.
- **Promisify** = wrap a callback API in a function that returns a promise, so you can use `async/await`.

## Example
Callback hell vs promises:

```js
// Callback style: nested, error check at every level
getUser(id, (err, user) => {
  if (err) return showError(err);
  getAccount(user.accountId, (err, account) => {
    if (err) return showError(err);
    getHoldings(account.id, (err, holdings) => {
      if (err) return showError(err);
      render(holdings);
    });
  });
});

// Same flow with promises + async/await: flat, one catch
try {
  const user = await getUserAsync(id);
  const account = await getAccountAsync(user.accountId);
  render(await getHoldingsAsync(account.id));
} catch (err) {
  showError(err);
}
```

Promisify, tested with `node`:

```js
function promisify(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      // add our own error-first callback as the last argument
      fn.call(this, ...args, (err, value) => (err ? reject(err) : resolve(value)));
    });
  };
}

// An old callback API
function getPrice(symbol, cb) {
  setTimeout(() => (symbol ? cb(null, { symbol, price: 101.5 }) : cb(new Error('no symbol'))), 10);
}

const getPriceAsync = promisify(getPrice);
console.log(await getPriceAsync('AAPL'));
try { await getPriceAsync(''); } catch (e) { console.log('error:', e.message); }
```

Output:

```text
{ symbol: 'AAPL', price: 101.5 }
error: no symbol
```

Wrapping an event-style API (not error-first) by hand:

```js
const loadImage = (src) => new Promise((resolve, reject) => {
  const img = new Image();
  img.onload = () => resolve(img);
  img.onerror = () => reject(new Error(`Failed to load ${src}`));
  img.src = src;
});
```

## When to use it
- Callbacks are still right for **events that happen many times**: clicks, WebSocket `message`, price ticks. A promise can only resolve once.
- Use promises / `async/await` for **one-off results**: a fetch, a file read, a timer you await.
- Promisify old SDKs (some charting or payment SDKs still take callbacks) so the rest of the Svelte app can `await` them.

## Likely questions
### What is callback hell and how do you fix it?
It is when each async step is nested inside the previous one's callback, so code drifts right and every level repeats error handling. Fixes: name and split the functions, or better, use promises so steps chain flat with `.then`, or use `async/await` with one `try/catch`.

### What is inversion of control?
When I pass my callback to a third-party library, I hand over control of *my* code to them. I trust they call it exactly once, at the right time, with the right arguments, and do not swallow errors. Promises give control back: I get a promise object, it can only settle once, and I decide what to do with it.

### Write `promisify`.
Return a new function that takes the args and returns a `new Promise`. Inside, call the original function with the args plus one extra callback `(err, value) => err ? reject(err) : resolve(value)`. Use `fn.call(this, ...)` so methods keep their `this`. Node has a built-in `util.promisify`, and many Node APIs have promise versions such as `fs/promises`.

### Can a callback be sync or async? Why does it matter?
Both. `[1,2].forEach(cb)` calls it right now; `setTimeout(cb)` calls it later. An API that is sometimes sync and sometimes async (for example, sync when cached) causes order bugs, often called "releasing Zalgo". Promises avoid this because `.then` callbacks always run later, as microtasks.

## Common mistakes
- Forgetting `return` after calling `cb(err)`, so the success path also runs.
- Calling the callback twice (once in `try`, again in `catch`).
- Throwing inside an async callback and expecting an outer `try/catch` to catch it. It will not; the outer code already finished.
- Promisifying something that fires many times (an event stream); use an async iterator or keep the callback.

## Resources
- [javascript.info: Callbacks](https://javascript.info/callbacks) - pyramid of doom with clear examples
- [javascript.info: Promisification](https://javascript.info/promisify) - writing promisify step by step
- [MDN: Callback function](https://developer.mozilla.org/en-US/docs/Glossary/Callback_function) - sync vs async callbacks
- [MDN: Using promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises) - why promises replace callback pyramids
