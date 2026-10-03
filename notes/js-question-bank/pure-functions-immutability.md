# Pure functions and immutability

> **In one line:** A pure function returns the same output for the same input and changes nothing outside itself. Immutability means you create a new object instead of editing the old one, so it is cheap and reliable to tell that something changed.

## Key points
- **Pure function**: same inputs give the same output, and no **side effects**. A side effect is anything visible outside the function: changing a variable or argument, a network call, writing to the DOM or `localStorage`, logging, reading the clock or `Math.random()`.- **Why immutability helps UI**: many frameworks detect change by comparing references (`prev === next`, or `Object.is`). React state, React `memo`, `$state.raw` in Svelte 5 and Redux all depend on getting a new object when data changes.
- **Spread** (`{...obj}`, `[...arr]`) is a **shallow** copy: nested objects are still shared. **`structuredClone`** makes a deep copy (handles Dates, Maps, Sets, cycles, but not functions or DOM nodes). See [MDN: structuredClone](https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone).
- **`Object.freeze`** blocks changes to the top level only. Nested objects stay mutable. Writes fail silently in sloppy mode and throw in strict mode.
- For nested updates, copy only the path you change and reuse the rest (**structural sharing**). Unchanged items keep their identity, so their components can skip re-rendering.

## Example: pure vs impure
```js
let taxRate = 0.18;
const impureTotal = (price) => price + price * taxRate; // reads outside state
const pureTotal = (price, rate) => price + price * rate; // only uses inputs

console.log(impureTotal(100));
taxRate = 0.2;
console.log(impureTotal(100), pureTotal(100, 0.18), pureTotal(100, 0.18));
```
Output (run with Node):
```text
118
120 118 118
```
Same call, different answer: `impureTotal` depends on hidden state. The pure one is easy to test and safe to cache (**memoize**).

## Example: spread vs structuredClone vs freeze
```js
const watch = { id: 1, sym: "TCS", quote: { ltp: 3500 } };
const shallow = { ...watch };
shallow.sym = "INFY";
shallow.quote.ltp = 1; // nested object is shared!
console.log(watch.sym, watch.quote.ltp, shallow.quote === watch.quote);

const w2 = { quote: { ltp: 3500 }, at: new Date(0), tags: new Set(["it"]) };
const deep = structuredClone(w2);
deep.quote.ltp = 1;
console.log(w2.quote.ltp, deep.at instanceof Date, deep.tags instanceof Set);
try { structuredClone({ fn: () => 1 }); } catch (e) { console.log(e.name + ": " + e.message); }

// sloppy-mode script
const cfg = Object.freeze({ theme: "dark", limits: { maxQty: 100 } });
cfg.theme = "light";     // silently ignored
cfg.limits.maxQty = 5;   // works: freeze is shallow
console.log(cfg.theme, cfg.limits.maxQty, Object.isFrozen(cfg), Object.isFrozen(cfg.limits));
```
Output:
```text
TCS 1 true
3500 true true
DataCloneError: () => 1 could not be cloned.
dark 5 true false
```
In an ES module (strict mode) the frozen write throws instead: `TypeError: Cannot assign to read only property 'theme' of object '#<Object>'`.

## Example: immutable watchlist updates
```js
const watchlist = [
  { id: 1, sym: "TCS", ltp: 3500, alerts: [3400] },
  { id: 2, sym: "INFY", ltp: 1500, alerts: [] },
];

// Update one field of one item: new array, new item, other items reused
const updateLtp = (list, id, ltp) =>
  list.map((item) => (item.id === id ? { ...item, ltp } : item));

// Update a nested array: copy each level on the path
const addAlert = (list, id, price) =>
  list.map((item) => (item.id === id ? { ...item, alerts: [...item.alerts, price] } : item));

const removeItem = (list, id) => list.filter((item) => item.id !== id);

const next = updateLtp(watchlist, 1, 3550);
console.log(watchlist[0].ltp, next[0].ltp, next === watchlist, next[0] === watchlist[0], next[1] === watchlist[1]);
const next2 = addAlert(next, 1, 3600);
console.log(next[0].alerts, next2[0].alerts, removeItem(next2, 1).map((i) => i.sym), watchlist.length);

// Non-mutating array helpers (ES2023) and a nested object update
const prices = [30, 10, 20];
console.log(prices, prices.toSorted((a, b) => a - b), prices.with(0, 99), prices.toReversed());
const state = { user: { prefs: { theme: "dark", lang: "en" } } };
const ns = { ...state, user: { ...state.user, prefs: { ...state.user.prefs, theme: "light" } } };
console.log(state.user.prefs.theme, ns.user.prefs.theme, ns.user.prefs.lang);
```
Output:
```text
3500 3550 false false true
[ 3400 ] [ 3400, 3600 ] [ 'INFY' ] 2
[ 30, 10, 20 ] [ 10, 20, 30 ] [ 99, 10, 20 ] [ 20, 10, 30 ]
dark light en
```
`next[1] === watchlist[1]` is `true`: the INFY row was not touched, so a keyed list can skip re-rendering it. The original `watchlist` never changed.

## When to use it
- Reducers, selectors and formatters (`formatPrice`, `computePnL`) should be pure: easy to unit test, safe to memoize.
- Svelte 5: `$state` is a deep **proxy**, so `item.ltp = 3550` is tracked and fine. But with `$state.raw` (good for large, frequently replaced data like a 5,000-row order book) you must assign a new object, so the immutable helpers above are exactly what you need. `$derived` expressions should be pure. See [Svelte docs: $state](https://svelte.dev/docs/svelte/$state).
- React: state must be replaced, never mutated, because it compares with `Object.is`.
- Undo/redo and "time travel" debugging work only if old snapshots are not mutated.

## Likely questions
### What is a pure function? Give an example of a side effect.
A function whose output depends only on its inputs and that changes nothing outside. `(a, b) => a + b` is pure. A function that calls `fetch`, writes to `localStorage`, mutates its argument, or uses `Date.now()` is not. Apps need side effects; the goal is to keep them at the edges (event handlers, `$effect`) and keep the core logic pure.

### Why does immutability matter for UI state?
Change detection becomes a cheap reference check: a new object means "changed", the same object means "skip". If you mutate in place, the reference is the same, so React or a `$state.raw` value will not notice and the UI goes stale. It also prevents bugs where two components share one object and one silently changes it.

### Spread vs structuredClone vs JSON clone?
Spread is shallow and fast, ideal for immutable updates where you copy only the changed path. `structuredClone` is a true deep copy and keeps Dates, Maps and Sets, but throws on functions. `JSON.parse(JSON.stringify())` loses `undefined`, Dates and Maps, so avoid it.

### Is Object.freeze enough for immutability?
No. It is shallow and checks only at runtime. You would need a recursive "deep freeze", and it has a small cost. In TypeScript, `readonly` and `as const` give compile-time protection at zero runtime cost.

## Common mistakes
- Thinking spread deep-copies. Nested objects are shared.
- Using `sort`, `reverse` or `splice` on state; they mutate. Use `toSorted`, `toReversed`, `toSpliced`, `with`.
- Deep-cloning the whole state on every update. It is slow and breaks identity for unchanged items.

## Resources
- [MDN: structuredClone](https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone) - deep copy rules and what cannot be cloned
- [MDN: Object.freeze](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze) - shallow freeze and strict-mode behaviour
- [javascript.info: Object references and copying](https://javascript.info/object-copy) - shallow vs deep copy explained simply
- [Svelte docs: $state](https://svelte.dev/docs/svelte/$state) - deep proxies vs `$state.raw`
