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

## Event loop architecture (one full turn)
What the [HTML spec's processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model) does on every turn, in simple steps:

```text
 1. Pick ONE task from a task queue        (setTimeout, click, message, network callback)
    -> run it to completion on the call stack
 2. Microtask checkpoint                   (promise.then, await, queueMicrotask, MutationObserver)
    -> run ALL microtasks, including ones queued while draining
 3. Update the rendering (if it is time, ~every 16.7 ms at 60 Hz, and the tab is visible)
    a. resize / scroll events
    b. requestAnimationFrame callbacks
    c. style -> layout -> paint (IntersectionObserver / ResizeObserver also run here)
 4. If there is spare time before the next frame: requestIdleCallback
 5. Repeat
```

- There is **more than one task queue** (user input, timers, network). The browser picks which queue to serve, and usually gives user input higher priority. Only the **order inside one queue** is guaranteed.
- **Rendering is not after every task.** It happens at the screen's refresh rate. Two `setTimeout`s can run in the same frame, and `requestAnimationFrame` runs once per frame, right before paint.
- **Microtasks can starve rendering.** A microtask that queues another microtask forever freezes the page (no render, no clicks), while a `setTimeout` loop does not.
- **Where it lives:** each renderer process has a **main thread** that runs this loop (JS, style, layout, paint). The **compositor** and **raster** threads handle scrolling and `transform`/`opacity` animations, which is why those stay smooth when JS is busy. Network, timers and file reading run on other threads and only post callbacks back.
- **Workers** each have their own event loop and call stack (no DOM, no rendering step).
- **Node.js** (libuv) has phases instead of a rendering step: timers -> pending callbacks -> poll (I/O) -> check (`setImmediate`) -> close callbacks. `process.nextTick` runs before promise microtasks. See [Node docs: event loop](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick).

```js
// rAF vs setTimeout vs microtask in one frame
setTimeout(() => console.log('timeout'), 0);
requestAnimationFrame(() => console.log('rAF (before paint)'));
Promise.resolve().then(() => console.log('microtask'));
console.log('sync');
// sync, microtask, then usually timeout before rAF,
// but rAF can come first if a frame is due; only "sync, microtask" first is guaranteed
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

### Explain the architecture of the event loop. Where does rendering fit?
Each turn the loop takes one task, runs it to completion, then drains the whole microtask queue. Then, if a frame is due (about every 16 ms) the browser runs `requestAnimationFrame` callbacks and does style, layout and paint. Spare time goes to `requestIdleCallback`. There are several task queues with different priorities (input is usually first). All of this runs on the renderer's main thread, while the compositor thread can scroll and run `transform` animations independently. So a long task delays clicks and paints, and an endless chain of microtasks blocks rendering completely.

### Browser event loop vs Node.js event loop?
Both run one task then drain microtasks. The browser has task queues plus a rendering step. Node (libuv) has fixed phases: timers, pending callbacks, poll for I/O, check (`setImmediate`), close callbacks, and drains `process.nextTick` and promise microtasks between callbacks. `process.nextTick` runs before promise callbacks. In Node, `setTimeout(0)` vs `setImmediate` order is not fixed from the main module, but inside an I/O callback `setImmediate` always runs first.

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
