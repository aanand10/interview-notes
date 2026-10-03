# Output questions: promises and event loop

> **In one line:** Synchronous code runs first, then **all** microtasks (promise callbacks, `await` continuations, `queueMicrotask`), then **one** macrotask (like a `setTimeout` callback), then all microtasks again, and so on.

## Key points
- **Call stack**: the code running right now. Nothing else runs until it is empty.
- **Microtask queue**: `.then/.catch/.finally` callbacks, the code after an `await`, and `queueMicrotask`. It is drained completely after each task, including microtasks added while draining. See [MDN: Using microtasks](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide).
- **Task (macrotask) queue**: `setTimeout`, `setInterval`, I/O, UI events. One task per loop turn. See [javascript.info: Event loop](https://javascript.info/event-loop).
- The function you pass to `new Promise(fn)` runs **synchronously**. Only the `.then` callbacks are async.
- An `async` function runs synchronously until its first `await`. Everything after the `await` is a microtask, even `await null` or `await 5`.

How to solve: write three columns on paper: **Sync log**, **Microtasks**, **Tasks**. Walk the code top to bottom, filling queues. Then print sync, drain micro, take one task, drain micro, repeat.

All outputs below were run with Node 20+. Browsers give the same order for these snippets.

## Drills

### Q1. The basic four
```js
console.log(1);
setTimeout(() => console.log(2), 0);
Promise.resolve().then(() => console.log(3));
console.log(4);
```
**Answer:**
```text
1
4
3
2
```
Sync: 1, 4. Then microtasks: 3. Then the timer task: 2. A 0 ms timer still waits for the microtask queue.

### Q2. The executor is synchronous
```js
new Promise((resolve) => {
  console.log("A");
  resolve("B");
  console.log("C");
}).then((v) => console.log(v));
console.log("D");
```
**Answer:**
```text
A
C
D
B
```
The executor runs right away, so A and C print. `resolve` does not stop the function. The `.then` callback is a microtask, so B prints after sync D.

### Q3. queueMicrotask vs then
```js
Promise.resolve().then(() => console.log("then 1"));
queueMicrotask(() => console.log("micro"));
Promise.resolve().then(() => console.log("then 2"));
setTimeout(() => console.log("timeout"), 0);
```
**Answer:**
```text
then 1
micro
then 2
timeout
```
They share one microtask queue and run in the order they were queued (FIFO). The timer comes last.

### Q4. await of a non-promise
```js
async function load() {
  console.log("load start");
  await null;
  console.log("load end");
}
console.log("start");
load();
console.log("end");
```
**Answer:**
```text
start
load start
end
load end
```
`load` runs sync until `await`. `await null` wraps `null` in a resolved promise, so the rest becomes a microtask, after sync `end`.

### Q5. Two chains interleave
```js
Promise.resolve()
  .then(() => console.log("a1"))
  .then(() => console.log("a2"));
Promise.resolve()
  .then(() => console.log("b1"))
  .then(() => console.log("b2"));
```
**Answer:**
```text
a1
b1
a2
b2
```
Only the first `.then` of each chain is queued at the start. `a2` is queued only when `a1` finishes, which is after `b1` is already in the queue.

### Q6. Timers and promises inside each other
```js
setTimeout(() => {
  console.log("t1");
  Promise.resolve().then(() => console.log("p inside t1"));
}, 0);
setTimeout(() => console.log("t2"), 0);
Promise.resolve().then(() => {
  console.log("p1");
  setTimeout(() => console.log("t inside p1"), 0);
});
```
**Answer:**
```text
p1
t1
p inside t1
t2
t inside p1
```
Step by step: no sync logs. Microtask p1 runs and adds a third timer at the back. Task t1 runs and queues a microtask, which drains before the next task (t2). The timer added by p1 goes last.

### Q7. Error in then, caught later
```js
Promise.resolve(1)
  .then((v) => {
    throw new Error("boom " + v);
  })
  .then(() => console.log("skipped"))
  .catch((e) => {
    console.log("caught:", e.message);
    return 2;
  })
  .then((v) => console.log("after catch:", v));
```
**Answer:**
```text
caught: boom 1
after catch: 2
```
A throw turns the chain rejected, so the next `.then` is skipped. `.catch` handles it and returns a normal value, so the chain is fulfilled again.

### Q8. finally passes the value through
```js
Promise.resolve("ok")
  .finally(() => {
    console.log("finally 1");
    return "ignored";
  })
  .then((v) => console.log("then:", v));
Promise.reject(new Error("bad"))
  .finally(() => console.log("finally 2"))
  .catch((e) => console.log("catch:", e.message));
```
**Answer:**
```text
finally 1
finally 2
then: ok
catch: bad
```
`finally` gets no argument and its return value is ignored. The original value or error passes through. The two chains interleave like Q5.

### Q9. finally that throws
```js
Promise.resolve("ok")
  .finally(() => {
    throw new Error("from finally");
  })
  .then((v) => console.log("then:", v))
  .catch((e) => console.log("catch:", e.message));
```
**Answer:**
```text
catch: from finally
```
Exception to Q8: if `finally` throws (or returns a rejected promise), that error replaces the original result.

### Q10. An async function always returns a promise
```js
async function getPrice() {
  return 100;
}
const p = getPrice();
console.log(p);
p.then((v) => console.log(v));
console.log(p instanceof Promise);
```
**Answer:**
```text
Promise { 100 }
true
100
```
`return 100` inside `async` resolves the returned promise. Node prints it as `Promise { 100 }` (a browser console shows `Promise {<fulfilled>: 100}`). The value comes out only through `.then` or `await`.

### Q11. Returning a promise from an async function costs extra ticks
```js
async function returnsPromise() {
  return Promise.resolve("A");
}
async function returnsValue() {
  return "B";
}
returnsPromise().then(console.log);
returnsValue().then(console.log);
```
**Answer:**
```text
B
A
```
When you resolve with a promise (a **thenable**), the engine must "unwrap" it, which takes 2 more microtask ticks. A plain value resolves at once, so B wins even though it was called second.

### Q12. try/catch/finally with await
```js
async function placeOrder() {
  try {
    await Promise.reject(new Error("insufficient funds"));
    console.log("placed");
  } catch (e) {
    console.log("error:", e.message);
  } finally {
    console.log("cleanup");
  }
  return "done";
}
placeOrder().then(console.log);
console.log("sync end");
```
**Answer:**
```text
sync end
error: insufficient funds
cleanup
done
```
`await` of a rejected promise throws at that line, so "placed" never prints and `catch` runs. All of it happens after the sync log.

### Q13. The classic interview puzzle
```js
async function async1() {
  console.log("async1 start");
  await async2();
  console.log("async1 end");
}
async function async2() {
  console.log("async2");
}
console.log("script start");
setTimeout(() => console.log("setTimeout"), 0);
async1();
new Promise((resolve) => {
  console.log("promise1");
  resolve();
}).then(() => console.log("promise2"));
console.log("script end");
```
**Answer:**
```text
script start
async1 start
async2
promise1
script end
async1 end
promise2
setTimeout
```
Sync: script start, async1 start, async2 (called sync), promise1 (executor), script end. Microtasks in queue order: "async1 end" was queued first by the `await`, then promise2. The timer is last.

### Q14. A promise settles only once
```js
const p = new Promise((resolve, reject) => {
  resolve("first");
  resolve("second");
  reject(new Error("third"));
});
p.then((v) => console.log(v)).catch((e) => console.log("never", e));
```
**Answer:**
```text
first
```
The first `resolve` or `reject` wins. Later calls are silently ignored.

### Q15. Resolving with a nested promise
```js
const outer = new Promise((resolve) => {
  resolve(Promise.resolve("inner value"));
});
outer.then((v) => console.log("outer got:", v));
Promise.resolve()
  .then(() => console.log("tick 1"))
  .then(() => console.log("tick 2"))
  .then(() => console.log("tick 3"));
```
**Answer:**
```text
tick 1
tick 2
outer got: inner value
tick 3
```
Promises flatten: `outer` gets the inner value, not a promise. But adopting the inner promise takes 2 extra ticks (same reason as Q11), so it lands between tick 2 and tick 3.

### Q16. Real delays
```js
setTimeout(() => console.log("10ms"), 10);
setTimeout(() => console.log("0ms"), 0);
(async () => {
  console.log("iife");
  await 1;
  console.log("after await 1");
  await new Promise((r) => setTimeout(r, 5));
  console.log("after 5ms wait");
})();
```
**Answer:**
```text
iife
after await 1
0ms
after 5ms wait
10ms
```
`await 1` is just a microtask, so it beats every timer. Then timers fire by their due time: 0 ms, then the 5 ms one (which resumes the async function), then 10 ms. Timer delays are a minimum, not a promise of exact timing.

## When it matters in real code
- Svelte batches DOM updates in a microtask. That is why you `await tick()` after changing `$state` before measuring the DOM.
- A long sync loop (for example parsing a huge order book snapshot) blocks rendering and clicks. Split it up or move it to a Web Worker.
- Retry/backoff for a price API uses `await new Promise(r => setTimeout(r, ms))`.

## Common mistakes
- Thinking `setTimeout(fn, 0)` runs "immediately". It waits for all sync code and all microtasks.
- Forgetting the `new Promise` executor is sync.
- Forgetting that `.catch` returns a fulfilled promise unless it rethrows.
- An infinite chain of microtasks starves timers and rendering, because the microtask queue must empty first.

## Resources
- [javascript.info: Event loop: microtasks and macrotasks](https://javascript.info/event-loop) - the clearest diagram of the loop
- [javascript.info: Microtasks](https://javascript.info/microtask-queue) - why `.then` is always async
- [MDN: Using microtasks](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide) - `queueMicrotask` and ordering rules
- [MDN: Promise.prototype.finally](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally) - how finally passes values and errors through
