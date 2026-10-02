# `debounce`

> **In one line:** Debounce waits until calls stop for `wait` ms and then runs the function once, so a burst of events (like typing) becomes a single call.

## Requirements

What a [debounce](https://developer.mozilla.org/en-US/docs/Glossary/Debounce) (like lodash's) should do:

- `debounce(fn, wait)` returns a new function. Every call **resets the timer**. `fn` runs only after `wait` ms of silence.
- **Trailing** (default): run at the **end** of the burst with the **latest** arguments.
- **Leading** (option): run **immediately** on the first call of a burst, then ignore calls until things are quiet for `wait` ms.
- Leading + trailing together: run at the start, and again at the end only if more calls came in during the burst.
- Preserve **`this`** and **arguments** of the last call, so it works as an object method or event handler.
- `cancel()` drops any pending call. `flush()` (bonus) runs the pending call right now.

## Implementation

```js
function debounce(fn, wait = 300, { leading = false, trailing = true } = {}) {
  let timerId = null;
  let lastArgs = null;
  let lastThis = null;

  function invoke() {
    const args = lastArgs, ctx = lastThis;
    lastArgs = lastThis = null;            // mark "nothing pending"
    fn.apply(ctx, args);
  }

  function debounced(...args) {            // normal function, so it receives `this`
    lastArgs = args;
    lastThis = this;
    const isFirstCallOfBurst = timerId === null;

    clearTimeout(timerId);                 // every call restarts the wait
    timerId = setTimeout(() => {
      timerId = null;                      // burst is over
      if (trailing && lastArgs) invoke();  // only if a call is still pending
    }, wait);

    if (leading && isFirstCallOfBurst) invoke(); // fire right away on the first call
  }

  debounced.cancel = () => {
    clearTimeout(timerId);
    timerId = lastArgs = lastThis = null;
  };

  debounced.flush = () => {                // run the pending call now
    if (timerId !== null) {
      clearTimeout(timerId);
      timerId = null;
      if (trailing && lastArgs) invoke();
    }
  };

  return debounced;
}
```

The simplest interview version (trailing only), if you are asked to write it in 2 minutes:

```js
function debounce(fn, wait) {
  let timerId;
  return function (...args) {
    clearTimeout(timerId);
    timerId = setTimeout(() => fn.apply(this, args), wait); // arrow keeps outer `this`
  };
}
```

## Edge cases

- **Rapid calls** - `clearTimeout` + new `setTimeout` each time, so only the last call survives.
- **Latest arguments win** - `lastArgs` is overwritten on every call, so the trailing call uses the newest search text.
- **`this`** - `debounced` is a normal function, so it gets the caller's `this`; we save it and use `fn.apply(ctx, args)`.
- **Leading + trailing with one call** - after the leading call, `invoke()` clears `lastArgs`, so the timer finds nothing pending and does not call `fn` twice.
- **`cancel()`** - clears the timer and the pending args. Nothing runs later.
- **Component unmount** - call `cancel()` in cleanup so a late call doesn't update a destroyed component.
- **Return value** - a debounced function can't return `fn`'s result synchronously (it runs later). If you need the result, have `fn` set state or return a promise.

## Test it

```js
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

const search = debounce((q) => console.log('trailing', q), 100);
search('r'); search('re'); search('rel');
await sleep(150);                       // trailing rel          (one call, last args)

const lead = debounce((q) => console.log('leading', q), 100, { leading: true, trailing: false });
lead('a'); lead('b'); lead('c');
await sleep(150);                       // leading a             (first call only)

const both = debounce((n) => console.log('both', n), 100, { leading: true, trailing: true });
both(1); both(2); both(3);
await sleep(150);                       // both 1 ... then both 3
both(4);
await sleep(150);                       // both 4                (single call: no duplicate)

const c = debounce(() => console.log('should not print'), 100);
c(); c.cancel();
await sleep(150);                       // (nothing)

const form = {
  symbol: 'TCS',
  onType: debounce(function (q) { console.log('this.symbol =', this.symbol, 'q =', q); }, 50),
};
form.onType('buy');
await sleep(100);                       // this.symbol = TCS q = buy
```

Checked with Node 24 (wrapped in an async IIFE). Output order: `trailing rel`, `leading a`, `both 1`, `both 3`, `both 4`, `this.symbol = TCS q = buy`.

## When to use it

- **Search box / symbol lookup**: call the search API only after the user stops typing for ~300 ms. Fewer requests, less flicker.
- Auto-saving a draft order form, validating a field, resizing a chart **after** the user finishes resizing.
- A "Place order" button with `leading: true` so a double-click sends only one order (but also disable the button and use an idempotency key on the server).

```svelte
<script>
  import { debounce } from '$lib/debounce';
  let query = $state('');
  let results = $state([]);

  const search = debounce(async (q) => {
    const res = await fetch(`/api/symbols?q=${encodeURIComponent(q)}`);
    results = await res.json();
  }, 300);

  $effect(() => {
    if (query.trim()) search(query);   // re-runs when query changes
    return () => search.cancel();      // cleanup on change or unmount
  });
</script>

<input bind:value={query} placeholder="Search symbol" />
```

Note: debounce does not stop **old responses arriving late**. Pair it with an `AbortController` or check that the response matches the current query.

## Likely follow-ups

### What is the difference between leading and trailing debounce?
Trailing waits for the user to stop and then fires once with the last value, which is what you want for search. Leading fires on the first event straight away and then ignores the rest of the burst, which is good for buttons where you want an instant response but no duplicates.

### Why do you use `fn.apply(this, args)`?
So the original function gets the same `this` and arguments it would have had without debounce. If `debounced` is used as an object method or a DOM handler, `this` is the object or element. If I used an arrow for `debounced`, I would lose that `this`.

### Why does the trailing call use the latest arguments?
Because I overwrite `lastArgs` on every call. For search, that means we query `"reliance"`, not `"r"`.

### How would you add `cancel()`?
Functions are objects, so I attach a method: `debounced.cancel = () => { clearTimeout(timerId); ... }`. Use it on unmount or when the user clears the input.

### Debounce vs throttle?
Debounce waits for quiet and then fires once. Throttle fires at most once every `wait` ms while events keep coming. Search input: debounce. Scroll position or live price redraw: throttle.

## Common mistakes

- Returning an arrow function for `debounced`, so `this` is lost.
- Creating the debounced function inside a render or event handler, so a new timer is made every time and nothing is debounced.
- Forgetting `clearTimeout`, which just delays every call instead of collapsing them.
- Firing twice (leading and trailing) for a single call.

## Resources

- [MDN Glossary: Debounce](https://developer.mozilla.org/en-US/docs/Glossary/Debounce) - short definition, leading vs trailing
- [javascript.info: Decorators and forwarding](https://javascript.info/call-apply-decorators) - has debounce and throttle exercises with solutions
- [MDN: setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout) - timer details and the `this` problem
- [LeetCode 2627: Debounce](https://leetcode.com/problems/debounce/) - practice problem
