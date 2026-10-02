# Event Loop

JavaScript runs on a **single thread**. The event loop is what lets it handle async work without blocking.

## The pieces

| Piece | What it holds |
| --- | --- |
| Call stack | Currently executing functions |
| Microtask queue | Promise callbacks, `queueMicrotask`, `MutationObserver` |
| Macrotask (task) queue | `setTimeout`, `setInterval`, I/O, UI events |

## The algorithm

1. Run the current task until the call stack is empty.
2. Run **all** microtasks (including ones queued while running microtasks).
3. Render if needed.
4. Take the next macrotask and repeat.

## Predict the output

```js
console.log('A');
setTimeout(() => console.log('B'), 0);
Promise.resolve().then(() => console.log('C'));
console.log('D');
```

Answer: `A D C B`. Sync code first, then microtasks (`C`), then the timer (`B`).

## Interview tips

- `setTimeout(fn, 0)` means "at least 0 ms, after the current task and all microtasks".
- An infinite chain of microtasks starves rendering; an infinite chain of `setTimeout` does not.
