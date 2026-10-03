# Microtasks vs macrotasks

> **In one line:** Microtasks (promise callbacks, `await`, `queueMicrotask`) always run before the next macrotask (`setTimeout`, events), because after every task the event loop empties the entire microtask queue first.

## Key points
- **Macrotask (task)**: one unit of work the loop picks per turn. Sources: the main script, `setTimeout`, `setInterval`, user events, `MessageChannel`, I/O in Node.
- **Microtask**: a small "do this right after the current code" job. Sources: `Promise.then/catch/finally`, code after `await`, [`queueMicrotask`](https://developer.mozilla.org/en-US/docs/Web/API/Window/queueMicrotask), `MutationObserver`.
- **Order per loop turn**: run one task, then run **all** microtasks (including new ones added while draining), then the browser may render, then the next task.
- **`requestAnimationFrame`** is neither. It runs in the **rendering step**, just before the browser paints, about once per screen refresh (~16.7 ms at 60 Hz). It is paused in background tabs.
- **Why microtasks first?** Promises promise consistency: a `.then` callback runs as soon as possible after the current code, before any outside event can change state in between.

## Example
```js
console.log('sync');
setTimeout(() => console.log('setTimeout 0'), 0);
queueMicrotask(() => console.log('queueMicrotask'));
Promise.resolve().then(() => console.log('promise.then'));
console.log('sync end');
```
Real output (node):
```text
sync
sync end
queueMicrotask
promise.then
setTimeout 0
```
- `sync`, `sync end`: the current task (the script).
- `queueMicrotask` and `promise.then` share one microtask queue, so they run in the order they were added.
- `setTimeout 0`: next macrotask.

## Where each API fits
| API | Queue | When it runs |
| --- | --- | --- |
| `Promise.then`, `await` | microtask | right after the current code, before rendering |
| `queueMicrotask(fn)` | microtask | same as above; use it to defer without making a promise |
| `setTimeout(fn, 0)` | macrotask | a later loop turn; min ~0-4 ms; lets the browser paint first |
| `requestAnimationFrame(fn)` | rendering step | right before the next paint, once per frame |
| `MessageChannel.postMessage` | macrotask | a later turn, no 4 ms clamp (used by schedulers) |

## When to use it
- **`queueMicrotask`**: batch several state changes and flush once at the end of the current code (this is how frameworks like Svelte batch DOM updates).
- **`setTimeout(0)`** or [`scheduler.yield()`](https://developer.chrome.com/blog/use-scheduler-yield) (Chromium; check support): split a long job, for example formatting 20,000 order-book rows in chunks, so clicks and paint happen in between.
- **`requestAnimationFrame`**: animate a price ticker flash or a chart, and read layout then write styles once per frame.

## Likely questions
### Which runs first, a microtask or a macrotask, and why?
A microtask. The spec says after each task finishes, the event loop must run every microtask in the queue before taking the next task or rendering. So even `setTimeout(fn, 0)` waits behind every pending `.then`.

### Where does `requestAnimationFrame` fit?
It runs in the render step, after microtasks and before paint. So in a single turn the order is: current task, all microtasks, then (if a frame is due) rAF callbacks, then style, layout, paint. Compared to `setTimeout(0)`, rAF may run before or after it depending on when the next frame is due, so do not rely on their order.

### Can microtasks starve the UI?
Yes. If a microtask keeps queuing another microtask, the queue never empties, so no task runs and the browser never paints. The page freezes like an infinite loop.
```js
setTimeout(() => console.log('timeout (had to wait)'), 0);
let n = 0;
function loop() {
  if (++n < 1_000_000) queueMicrotask(loop);
  else console.log('microtasks done:', n);
}
queueMicrotask(loop);
```
Real output (node):
```text
microtasks done: 1000000
timeout (had to wait)
```
The timeout had to wait for one million microtasks. If the condition never ended, it would never run. Macrotasks cannot starve the UI in the same way, because the browser can render between them.

### How would you break up a long job so the UI stays responsive?
Process a chunk, then yield with a macrotask (`await new Promise(r => setTimeout(r, 0))`) or `scheduler.yield()`. Yielding with `await Promise.resolve()` does **not** help, because that is a microtask and the browser still cannot paint.

### Are there other queues?
In Node, `process.nextTick` runs even before promise microtasks, and `setImmediate` runs in the "check" phase after I/O. In browsers there can be several task queues (for example input events may get priority), but microtasks always come first.

## Common mistakes
- Using `await Promise.resolve()` to "yield to the browser". It does not let it paint.
- Calling `setTimeout(0)` a microtask.
- Thinking rAF runs at a fixed time after `setTimeout`. It is tied to the display frame.

## Resources
- [MDN: Using microtasks (Microtask guide)](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide) - official explanation and `queueMicrotask` use cases
- [javascript.info: Microtasks](https://javascript.info/microtask-queue) - short examples of the promise job queue
- [MDN: requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) - frame timing and background-tab behaviour
- [web.dev: Optimize long tasks](https://web.dev/articles/optimize-long-tasks) - yielding strategies to keep the UI responsive
