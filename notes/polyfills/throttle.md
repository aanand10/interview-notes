# `throttle`

> **In one line:** Throttle lets a function run at most once every `wait` ms, no matter how often it is called, so a stream of events (scroll, resize, price ticks) becomes a steady, limited rate of calls.

## Requirements

What a [throttle](https://developer.mozilla.org/en-US/docs/Glossary/Throttle) (like lodash's) should do:

- `throttle(fn, wait)` returns a new function. While calls keep coming, `fn` runs **at most once per `wait` ms**.
- **Leading** (default `true`): run on the **first** call right away.
- **Trailing** (default `true`): if calls came in during the window, run once more **at the end of the window** with the **latest** arguments. This makes sure the final value (last scroll position, last price) is never lost.
- Both `false` is pointless; usually one of them is on.
- Preserve **`this`** and **arguments**. Provide `cancel()`.

## Implementation

```js
function throttle(fn, wait = 100, { leading = true, trailing = true } = {}) {
  let lastCallTime = 0;      // when fn last ran (0 = never)
  let timerId = null;
  let lastArgs = null;
  let lastThis = null;

  function invoke() {
    lastCallTime = Date.now();
    const args = lastArgs, ctx = lastThis;
    lastArgs = lastThis = null;
    fn.apply(ctx, args);
  }

  function throttled(...args) {
    const now = Date.now();
    if (!lastCallTime && !leading) lastCallTime = now;  // start the window without calling
    const remaining = wait - (now - lastCallTime);
    lastArgs = args;          // always remember the newest call
    lastThis = this;

    if (remaining <= 0) {
      // Window is open: run now
      clearTimeout(timerId);
      timerId = null;
      invoke();
    } else if (!timerId && trailing) {
      // Inside the window: schedule ONE trailing call for when it closes
      timerId = setTimeout(() => {
        timerId = null;
        lastCallTime = leading ? Date.now() : 0;
        if (lastArgs) invoke();
      }, remaining);
    }
  }

  throttled.cancel = () => {
    clearTimeout(timerId);
    timerId = lastArgs = lastThis = null;
    lastCallTime = 0;
  };

  return throttled;
}
```

Simplest interview version (leading only, a "cooldown flag"):

```js
function throttle(fn, wait) {
  let cooling = false;
  return function (...args) {
    if (cooling) return;               // drop calls during the cooldown
    fn.apply(this, args);
    cooling = true;
    setTimeout(() => { cooling = false; }, wait);
  };
}
```

Its weakness: the **last** event in a burst can be dropped, so the UI may show a stale scroll position or price. That is why the trailing option exists.

## Edge cases

- **Burst of calls** - the first runs now (leading), the rest only update `lastArgs`; one timer fires at the end of the window with the newest args (trailing).
- **Only one timer at a time** - the `!timerId` check stops us from scheduling many trailing calls.
- **`leading: false`** - on the first call we set `lastCallTime = now`, so `remaining` is positive and only the trailing timer runs.
- **`trailing: false`** - we never schedule a timer; calls inside the window are dropped.
- **`this` and args** - saved on each call and passed through `fn.apply`.
- **`cancel()`** - clears the timer and state; good for component cleanup.
- **Timer accuracy** - `setTimeout` can fire a few ms late, so the exact times drift a little. That is normal.

## Test it

```js
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
async function burst(label, opts) {
  const t0 = Date.now();
  const calls = [];
  const t = throttle((v) => calls.push(`${v}@${Math.round((Date.now() - t0) / 10) * 10}`), 100, opts);
  for (let i = 1; i <= 10; i++) { t(i); await sleep(25); }  // a call every 25 ms for 250 ms
  await sleep(200);
  console.log(label.padEnd(18), calls.join('  '));
}
await burst('default', {});
await burst('leading only', { trailing: false });
await burst('trailing only', { leading: false });
```

Real output from Node 24 (value@ms):

```bash
default            1@0  4@100  8@200  10@300
leading only       1@0  5@100  9@210
trailing only      4@100  8@200  10@300
```

- **default**: first call instantly, then one call per 100 ms window, and the last value `10` is delivered at the end.
- **leading only**: calls at the start of each window; the final value `10` is lost.
- **trailing only**: nothing at time 0; each window ends with its latest value.

## Debounce vs throttle

| | Debounce | Throttle |
|---|---|---|
| Fires | After calls **stop** for `wait` ms | At most once **every** `wait` ms while calls continue |
| During a constant stream | May never fire (keeps resetting) | Fires regularly |
| Best for | Search input, autosave, validation | Scroll, resize, mousemove, live price redraw |

## When to use it

- **Live prices over WebSocket**: ticks may arrive 50 times a second. Throttle the UI update to every 100-250 ms so the watchlist stays smooth, but keep trailing on so the latest price always shows.
- **Scroll**: infinite-scroll checks or "back to top" button visibility on the [scroll event](https://developer.mozilla.org/en-US/docs/Web/API/Element/scroll_event).
- **Resize**: re-layout a chart while the [window resizes](https://developer.mozilla.org/en-US/docs/Web/API/Window/resize_event).
- For pure visual updates, [`requestAnimationFrame`](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) is a natural throttle to the screen refresh rate.

## Likely follow-ups

### What's the difference between throttle and debounce?
Debounce waits for a pause and then fires once. Throttle fires at a fixed maximum rate while events keep coming. If a user scrolls non-stop for 5 seconds, a debounced handler runs once at the end, but a 100 ms throttled handler runs about 50 times.

### Why do you need the trailing call?
Without it, the last event in a burst can be dropped. For a price feed that means the screen might show an old price until the next tick. Trailing makes sure the final state is always rendered.

### How would you throttle with requestAnimationFrame?
Store the latest args, and if no frame is scheduled, call `requestAnimationFrame` to run `fn` with the stored args. This limits updates to once per frame (about 60 per second) and syncs with painting.

```js
function rafThrottle(fn) {
  let frame = null, lastArgs;
  return function (...args) {
    lastArgs = args;
    if (frame === null) {
      frame = requestAnimationFrame(() => { frame = null; fn.apply(this, lastArgs); });
    }
  };
}
```

### Why `Date.now()` and not just a boolean flag?
The flag version can only do leading calls. Tracking the last call time lets me compute how long is left in the window, so I can schedule the trailing call at exactly the right moment.

## Common mistakes

- Scheduling a new trailing timer on every call (no `!timerId` check), which fires many calls at once.
- Using the first call's arguments for the trailing call instead of the latest.
- Losing `this` by returning an arrow function.
- Using debounce for a live price feed: under constant ticks it may never update the screen.

## Resources

- [MDN Glossary: Throttle](https://developer.mozilla.org/en-US/docs/Glossary/Throttle) - short definition and use cases
- [MDN Glossary: Debounce](https://developer.mozilla.org/en-US/docs/Glossary/Debounce) - compare the two
- [javascript.info: Decorators and forwarding](https://javascript.info/call-apply-decorators) - throttle decorator exercise with solution
- [MDN: requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) - frame-based throttling for visual updates
