# Garbage collection

> **In one line:** JavaScript frees memory for you: the garbage collector starts from the "roots" (globals, the current call stack), marks everything it can reach, and sweeps away the rest, and V8 makes this fast by splitting the heap into a young generation and an old generation.

## Key points
- **Reachability:** a value stays alive while something can reach it from a root. Roots are global variables, the local variables and parameters of functions currently on the call stack, and internal browser references (for example, live DOM nodes and active listeners). Anything you cannot reach can be collected.
- **Mark-and-sweep:** (1) Mark: start at the roots, follow every reference, and mark each object you visit. (2) Sweep: free every object that was not marked. Often there is a third step, compact, which moves live objects together so free memory is not split into small pieces.
- **Cycles are fine:** two objects that point only at each other, with nothing else pointing at them, are unreachable, so they get collected. Old reference counting could not handle this. Mark-and-sweep can.
- **Generational GC:** most objects die young (temporary arrays, event objects). So V8 has a small **young generation**, cleaned often and quickly by a copying "scavenger", and a big **old generation** for objects that survived a couple of young-generation cleanups, cleaned less often with mark-sweep-compact.
- **No pauses you can control:** GC runs when the engine decides. V8 does much of the marking in small steps (incremental) and on background threads (concurrent/parallel), so the main thread pauses less. You cannot force GC from page code.

## Example
Run with `node --expose-gc` (only so we can call `gc()` for the demo):

```js
// 1. Still reachable through another reference
let user = { name: 'Asha' };
const ref = new WeakRef(user);   // WeakRef does not keep the object alive
const admin = user;              // second, strong reference
user = null;
globalThis.gc();
console.log('after user=null:', ref.deref()?.name);
```

```bash
after user=null: Asha
```
- `admin` still points at the object, so it is reachable and is not collected.

```js
// 2. A cycle with no outside references
function makeCycle() {
  const a = {}; const b = {};
  a.b = b; b.a = a;              // a and b point at each other
  return new WeakRef(a);         // only a weak link leaves the function
}
const ref = makeCycle();
setTimeout(() => { globalThis.gc(); console.log('cycle collected:', ref.deref() === undefined); }, 0);
```

```bash
cycle collected: true
```
- After `makeCycle` returns, nothing reachable points at `a` or `b`. Mark-and-sweep never marks them, so they are freed even though they reference each other.

## When to use it
- **Long-running trading sessions:** a dashboard that stays open for 8 hours with prices streaming in creates lots of short-lived objects (one per tick). Generational GC handles those cheaply. What hurts is objects that stay reachable by accident, such as an ever-growing tick history array. That is a memory leak, not a GC problem.
- **Reduce GC pressure in hot paths:** in a 60 fps chart, avoid making new arrays or objects every frame where you can reuse a buffer. Less garbage means fewer and shorter GC pauses (less jank).
- **Caches keyed by objects:** use `WeakMap` so an entry goes away automatically when its key object is no longer used anywhere else.

## Likely questions

### How does garbage collection work in JavaScript?
The engine finds out which objects are still reachable. It starts from roots like global variables and the current call stack, follows every reference, and marks what it finds. Anything left unmarked cannot be used by the program again, so its memory is freed. That is mark-and-sweep. Modern engines add compaction (moving live objects together) and do most of the work in steps or on background threads.

### What does "reachable" mean? Give an example.
A value is reachable if you can get to it through a chain of references starting from a root. If `window.store` points to an object that holds an array of orders, all those orders are reachable. If you set `window.store = null` and nothing else points at them, the whole group becomes unreachable at once and can be collected.

### What is generational garbage collection?
It is based on the observation that most objects die young. V8 splits the heap. New objects go into a small young generation, which is cleaned very often by a scavenger: it copies the few surviving objects to a new space and throws the rest away in one go. Objects that survive a couple of these cleanups are promoted to the old generation. That area is cleaned less often with a full mark-sweep-compact, mostly concurrently. This keeps the common case (short-lived garbage) very cheap.

### Mark-and-sweep vs reference counting?
Reference counting keeps a count of how many references point at each object and frees it when the count hits zero. It fails on cycles: two objects pointing at each other never reach zero, so they leak. Mark-and-sweep works from reachability, so cycles with no outside link are collected. All modern JS engines use tracing (mark-based) GC.

### What are WeakMap and WeakRef for?
A [`WeakMap`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap) holds its keys weakly. If the key object is not reachable anywhere else, the entry can be collected. That is useful for attaching metadata to DOM nodes or objects without leaking. A [`WeakRef`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef) lets you hold an object without keeping it alive. GC timing is not predictable, so do not build program logic that depends on exactly when something is collected.

### Can you trigger GC manually?
Not from normal page code. In DevTools you can click the "Collect garbage" button in the Memory or Performance panel, and Node has `--expose-gc` for testing. In production you only control what stays reachable.

## Common mistakes
- Saying "setting a variable to `null` deletes the object". It only removes one reference. The object is freed only when no references are left.
- Saying "JS uses reference counting". Modern engines use tracing/mark-based GC.
- Thinking closures always leak. A closure keeps its outer variables alive only while the closure itself is reachable.
- Relying on `FinalizationRegistry` callbacks for important cleanup. They might run late or not at all.

## Resources
- [javascript.info: Garbage collection](https://javascript.info/garbage-collection) - reachability explained with diagrams
- [MDN: Memory management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Memory_management) - mark-and-sweep vs reference counting
- [V8 blog: Trash talk (Orinoco GC)](https://v8.dev/blog/trash-talk) - V8's generational, parallel and concurrent GC from the engine team
- [MDN: WeakMap](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap) - weakly held keys
