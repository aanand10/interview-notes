# Event loop

> **In one line:** JavaScript runs one thing at a time on a single call stack, and the event loop decides what runs next: first all synchronous code, then every microtask (promise callbacks), then one macrotask (like a `setTimeout` callback), and repeat.

## Key points
- **Call stack**: where functions run right now. JS has one, so only one piece of code runs at a time ("single-threaded").
- **Web APIs** (browser) or **libuv** (Node): timers, `fetch`, DOM events. These do the waiting outside the stack, then hand a callback back.
- **Task queue** (also called macrotask queue): callbacks from `setTimeout`, `setInterval`, events, message channels. The loop takes **one** task per turn.
- **Microtask queue**: `Promise.then/catch/finally`, code after `await`, `queueMicrotask`. After each task (and after the main script), the loop **empties the whole microtask queue** before doing anything else.
- The browser can **render** (style, layout, paint, plus `requestAnimationFrame`) only between tasks, after microtasks are done. A long task blocks clicks and paint. See [MDN: execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model).

## The picture (draw this on the whiteboard)
```text
            +------------------+        +------------------------+
 your code  |    CALL STACK    | -----> |  Web APIs / Node APIs  |
 runs here  |  (one at a time) |  start |  timers, fetch, events |
            +------------------+  work  +------------------------+
                    ^                          |  when done, queue the callback
                    |                          v
                    |            +-----------------------------+
                    |            | MICROTASK QUEUE (high prio) |  <- promise.then, await, queueMicrotask
                    |            +-----------------------------+
                    |            | TASK QUEUE (macrotasks)     |  <- setTimeout, setInterval, click, message
                    |            +-----------------------------+
                    |                          |
                    +------- EVENT LOOP -------+
   Loop: stack empty? -> run ALL microtasks -> (maybe render) -> take ONE task -> repeat
```

## Example
The classic output question. Run with `node` (same order in browsers):

```js
console.log('1: sync start');
setTimeout(() => console.log('2: setTimeout'), 0);
Promise.resolve().then(() => console.log('3: promise.then'));
async function run() {
  console.log('4: inside async fn (before await)');
  await null;
  console.log('5: after await');
}
run();
queueMicrotask(() => console.log('6: queueMicrotask'));
console.log('7: sync end');
```

Real output:
```text
1: sync start
4: inside async fn (before await)
7: sync end
3: promise.then
5: after await
6: queueMicrotask
2: setTimeout
```
- `1`, `4`, `7`: synchronous. The body of an async function runs synchronously **until the first `await`**.
- `3`, `5`, `6`: microtasks, run in the order they were queued, once the stack is empty.
- `2`: a macrotask. Even with 0 ms it waits until all microtasks are done.

## When to use it
- Explaining why a UI freezes: a big loop over 50,000 trade rows blocks the stack, so clicks and paints wait.
- Knowing that `await` gives control back, so a live price socket message can be handled while you wait for `fetch`.
- In Svelte, DOM updates are batched and applied in a microtask; `await tick()` waits for them ([Svelte docs: tick](https://svelte.dev/docs/svelte/lifecycle-hooks#tick)).

## Likely questions
### Explain the event loop in simple words.
JS has one call stack. Slow work like timers or network is handed to the browser's Web APIs. When that work finishes, its callback goes into a queue. The event loop waits for the stack to be empty, then runs all microtasks, lets the browser paint if needed, then takes the next macrotask. That is how a single thread stays responsive.

### What is the output? (async function + promise chain)
```js
async function a1() { console.log('a1 start'); await a2(); console.log('a1 end'); }
async function a2() { console.log('a2'); }
console.log('script start');
setTimeout(() => console.log('setTimeout'), 0);
a1();
new Promise((resolve) => { console.log('promise executor'); resolve(); })
  .then(() => console.log('then 1'))
  .then(() => console.log('then 2'));
console.log('script end');
```
Real output:
```text
script start
a1 start
a2
promise executor
script end
a1 end
then 1
then 2
setTimeout
```
The promise executor runs synchronously. `a1 end` is queued first (when `await a2()` settles), then `then 1`. `then 2` is only queued after `then 1` runs. `setTimeout` is a task, so it is last.

### What about microtasks queued inside a timeout?
```js
setTimeout(() => {
  console.log('timeout 1');
  Promise.resolve().then(() => console.log('micro inside timeout 1'));
}, 0);
setTimeout(() => console.log('timeout 2'), 0);
Promise.resolve().then(() => {
  console.log('micro 1');
  setTimeout(() => console.log('timeout from micro'), 0);
});
```
Real output:
```text
micro 1
timeout 1
micro inside timeout 1
timeout 2
timeout from micro
```
After each single task, the microtask queue is drained. So the microtask from `timeout 1` runs before `timeout 2`. The timeout queued from `micro 1` joins the end of the task queue.

### Does `setTimeout(fn, 0)` run after 0 ms?
No. 0 is the minimum delay, not a promise. It runs after the current script and all microtasks, and only when the stack is free. Browsers also clamp nested timers to at least 4 ms after 5 levels of nesting ([MDN: setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout)).

### Is JavaScript single-threaded? Then how does fetch run in parallel?
Your JS code runs on one thread. But the browser itself is multi-threaded: network, timers and decoding happen elsewhere. Only the callback comes back to your thread. For real parallel JS you use Web Workers.

## Common mistakes
- Thinking `await` blocks the whole program. It only pauses that async function.
- Forgetting the code before the first `await` runs synchronously.
- Saying `setTimeout(0)` runs "immediately" or before promises.
- Forgetting the Promise executor (`new Promise(fn)`) runs synchronously.

## Resources
- [javascript.info: Event loop](https://javascript.info/event-loop) - clear diagrams and macrotask vs microtask examples
- [MDN: JavaScript execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model) - official model of stack, queues and agents
- [MDN: Microtask guide](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide) - when microtasks run and why
- [web.dev: Optimize long tasks](https://web.dev/articles/optimize-long-tasks) - why blocking the main thread hurts the UI
