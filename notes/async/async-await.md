# async/await

> **In one line:** `async/await` is nicer syntax on top of promises: an `async` function always returns a promise, and `await` pauses only that function until a promise settles, so async code reads top to bottom like normal code.

## Key points
- **An `async` function always returns a promise.** `return 42` becomes a promise fulfilled with 42. `throw err` becomes a rejected promise ([MDN: async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)).
- **`await p`** pauses the function, lets other code run, and resumes later as a **microtask** with the value. If `p` rejects, `await` **throws** at that line.
- **Error handling**: use normal `try/catch/finally` around `await`. Or call `.catch()` on the returned promise.
- **Code before the first `await` runs synchronously.** Everything after it runs later.
- **Top-level await**: `await` at the top of an **ES module** (not inside a function). Works in `<script type="module">`, `.mjs` files and bundlers; not in classic scripts or CommonJS ([MDN: await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await#top_level_await)).

## Example
```js
async function f() { return 42; }
const p = f();
console.log(p instanceof Promise, p);

async function g() { throw new Error('nope'); }
g().catch((e) => console.log('g rejected:', e.message));

async function h() {
  try {
    await Promise.reject(new Error('api down'));   // await throws here
  } catch (e) {
    console.log('caught in try/catch:', e.message);
    return 'fallback';
  } finally {
    console.log('cleanup');
  }
}
h().then((v) => console.log('h ->', v));
```
Real output (node):
```text
true Promise { 42 }
g rejected: nope
caught in try/catch: api down
cleanup
h -> fallback
```
- `f()` returns a promise, not 42.
- The thrown error in `g` becomes a rejection, caught by `.catch`.
- In `h`, the rejected `await` throws, `catch` returns a fallback, `finally` always runs.

## When to use it
Almost everywhere you would write a `.then` chain. For example, an order form in Svelte 5:

```svelte
<script lang="ts">
  let qty = $state(1);
  let status = $state<'idle' | 'sending' | 'done' | 'error'>('idle');

  async function placeOrder() {
    status = 'sending';
    try {
      const res = await fetch('/api/orders', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ symbol: 'AAPL', qty })
      });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      status = 'done';
    } catch {
      status = 'error';
    }
  }
</script>

<button onclick={placeOrder} disabled={status === 'sending'}>Buy</button>
```

## Likely questions
### How does async/await relate to promises?
It is built on promises; it is not a new async system. `await x` is roughly `x.then(rest of the function)`. The async function's return value is a promise you can `.then`, `await`, or pass to `Promise.all`. You still need promises for things like combinators.

### What does an async function return?
Always a promise. Return a value and the promise fulfills with it. Return a promise and it adopts that promise's result. Throw and it rejects. Even an async function with no `return` gives `Promise<undefined>`.

### How do you handle errors?
Wrap the `await` lines in `try/catch`. A rejected promise makes `await` throw, so `catch` gets the error, just like sync code. Use `finally` for cleanup like hiding a spinner. If you do not catch, the returned promise rejects, so the caller must handle it.

### What is top-level await and where can you use it?
It lets you `await` at the top level of a module, for example `const config = await fetch('/config.json').then(r => r.json());`. It only works in ES modules. Any module that imports this one waits until it finishes, so a slow top-level await delays app start. In SvelteKit you would usually load data in a `load` function instead.

### What happens if you `await` a non-promise?
It is wrapped in a resolved promise, so `await 5` gives 5, but the rest of the function still runs later as a microtask.

### Does `await` inside `forEach` work?
No. `forEach` ignores the returned promises, so it does not wait. Use `for...of` with `await` for one-by-one, or `Promise.all(items.map(...))` for parallel.

### Is `return await` needed?
Usually not. But inside `try/catch`, `return await p` is needed if you want the `catch` to handle `p`'s rejection. Plain `return p` lets the error escape the `try`.

## Common mistakes
- Forgetting `await`, so you get a `Promise` object instead of data (`if (promise)` is always true).
- Awaiting independent calls one after another, which makes them slow (see the sequential vs parallel note).
- Using `await` in `forEach`.
- Not catching errors, which gives unhandled promise rejections.

## Resources
- [javascript.info: Async/await](https://javascript.info/async-await) - simple walkthrough with exercises
- [MDN: async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) - exact return and throw behaviour
- [MDN: await (top-level await)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await) - await rules including modules
- [MDN: Promises (Learn)](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Async_JS/Promises) - beginner guide linking promises and async/await
