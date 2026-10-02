# Layout thrashing

> **In one line:** Layout thrashing happens when code mixes DOM reads and writes in a loop, so the browser is forced to recalculate layout again and again inside one frame; the fix is to do all the reads first, then all the writes, and schedule the writes with `requestAnimationFrame`.

## Key points
- Normally the browser is lazy. When you change a style, it marks the layout as "dirty" and recalculates it once, just before the next frame is painted.
- **Reading** a layout value (`offsetHeight`, `getBoundingClientRect()`, `scrollTop`, `clientWidth`, `getComputedStyle()`) while the layout is dirty forces the browser to do layout **right now**, so it can give you a correct number. This is called a forced synchronous layout (or forced reflow).
- Read, write, read, write in a loop means one forced layout per loop pass. With 500 rows, that is 500 layouts in one frame instead of 1, and the page freezes.
- **The fix:** do every read first, then every write. Or move the writes into `requestAnimationFrame` (rAF), which runs just before the next paint.

## Example
Making every card in a watchlist as tall as the tallest one:

```js
const cards = [...document.querySelectorAll('.stock-card')];

// BAD: read and write in the same loop
// Pass 1 reads (fine), then writes (layout is now dirty).
// Pass 2 reads offsetHeight -> forced layout. This repeats for every card.
function equalizeBad() {
  for (const card of cards) {
    const h = card.offsetHeight;        // READ (forces layout if dirty)
    card.style.minHeight = `${h + 8}px`; // WRITE (makes layout dirty again)
  }
}

// GOOD: batch every read, then batch every write -> only 1 layout
function equalizeGood() {
  const heights = cards.map((card) => card.offsetHeight); // all READS
  const max = Math.max(...heights);
  for (const card of cards) {
    card.style.minHeight = `${max}px`;                      // all WRITES
  }
}
```

Using `requestAnimationFrame` to keep reads and writes in separate phases, for example in a scroll handler:

```js
let ticking = false;
let lastScrollY = 0;

window.addEventListener('scroll', () => {
  lastScrollY = window.scrollY;      // READ in the event handler
  if (ticking) return;               // at most one rAF per frame
  ticking = true;
  requestAnimationFrame(() => {
    header.classList.toggle('compact', lastScrollY > 80); // WRITE right before paint
    ticking = false;
  });
}, { passive: true });
```

A tiny read/write scheduler (the idea behind the `fastdom` library):

```js
const reads = [];
const writes = [];
let scheduled = false;

function flush() {
  reads.splice(0).forEach((fn) => fn());   // all reads first
  writes.splice(0).forEach((fn) => fn());  // then all writes
  scheduled = false;
}
function schedule() {
  if (!scheduled) { scheduled = true; requestAnimationFrame(flush); }
}
export const measure = (fn) => { reads.push(fn); schedule(); };
export const mutate  = (fn) => { writes.push(fn); schedule(); };
```

## When to use it
- **Live order book or watchlist:** when many rows update their widths or heights (depth bars, flashing cells), measure everything once, then apply all the style changes.
- **Auto-sizing tables, sticky headers, tooltips placed next to a cell:** these need `getBoundingClientRect()`. Read it once per frame, not once per row.
- **Svelte:** `bind:clientWidth` / `bind:clientHeight` use a `ResizeObserver` under the hood, so you get sizes without forcing layout yourself. In an `$effect`, read all the measurements first, then write.

## Likely questions

### What is layout thrashing? Show it in code.
It is when JavaScript writes a style and then reads a layout property again and again in a loop. Each read after a write forces the browser to recalculate layout synchronously, instead of once per frame. The `equalizeBad` loop above is the classic example: `offsetHeight` (read) and then `style.minHeight` (write) on every pass. In the DevTools Performance panel you see many purple "Layout" blocks with a "Forced reflow" warning.

### How do you fix it?
Split the work into two phases. First read everything you need into variables. Then do all the writes. That gives one layout instead of N. If reads and writes come from different parts of the code, schedule the writes in `requestAnimationFrame`, or use a read/write queue like fastdom, so they all run together just before the paint.

### Why does `requestAnimationFrame` help?
rAF callbacks run once per frame, right before the browser does style, layout and paint. If you do your DOM writes there, they all land together, and the browser does one layout for the frame. It also ties visual updates to the screen's refresh rate, so you do not do work the user will never see. Watch out: if you read a layout value inside rAF **after** writing, you still force a layout.

### Which properties or methods force a layout?
The common ones: `offsetTop/Left/Width/Height`, `clientTop/Left/Width/Height`, `scrollTop/Left/Width/Height`, `getBoundingClientRect()`, `getClientRects()`, `getComputedStyle()` (for layout values), `innerText`, `focus()`, and `scrollIntoView()`. They only cost a lot when the layout is dirty, which means you made a change just before.

### Is reading `offsetHeight` always slow?
No. If nothing has changed since the last layout, the browser returns the cached value cheaply. The cost only comes when a write makes the layout dirty and then a read forces it to recalculate. That is why the order of reads and writes matters, not just how many there are.

## Common mistakes
- Thinking that wrapping the bad loop in one `requestAnimationFrame` fixes it. It does not: the reads and writes are still mixed inside the callback.
- Using `setTimeout(fn, 0)` for visual updates. It is not lined up with frames, so you can get extra or missed frames.
- Reading `scrollTop` or `getBoundingClientRect()` on every scroll or mousemove event without throttling. Use rAF, `IntersectionObserver` or `ResizeObserver` instead.
- Changing a style just to "measure, then change it back" inside a loop.

## Resources
- [web.dev: Avoid large, complex layouts and layout thrashing](https://web.dev/articles/avoid-large-complex-layouts-and-layout-thrashing) - the canonical explanation with examples
- [MDN: requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) - API details
- [MDN: getBoundingClientRect](https://developer.mozilla.org/en-US/docs/Web/API/Element/getBoundingClientRect) - a common forced-layout read
- [svelte.dev: bind](https://svelte.dev/docs/svelte/bind) - dimension bindings (`bind:clientWidth`) without manual reads
