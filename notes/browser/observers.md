# Observers

> **In one line:** Observers let the browser tell you when something changes, instead of you checking again and again: IntersectionObserver for "is this element on screen", MutationObserver for "did the DOM change", and ResizeObserver for "did this element's size change".

## Key points
- [**IntersectionObserver**](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API): fires when an element enters or leaves the viewport (or a scroll container). Used for lazy loading, infinite scroll, "seen" analytics, pausing off-screen charts. Replaces scroll listeners plus `getBoundingClientRect()`, which are costly.
- [**MutationObserver**](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver): fires when child nodes, attributes or text change. Used to react to DOM changes you do not control, like a third-party widget.
- [**ResizeObserver**](https://developer.mozilla.org/en-US/docs/Web/API/ResizeObserver): fires when an element's size changes, not just the window. Used for charts that must redraw to fit their container.
- All three are **asynchronous and batched**: you get a list of entries in one callback. Always call `disconnect()` (or `unobserve`) when done to avoid memory leaks.

## Example
Infinite scroll for order history with IntersectionObserver:

```js
const sentinel = document.querySelector('#load-more'); // empty div at the end of the list
let loading = false;

const io = new IntersectionObserver(async (entries) => {
  if (!entries[0].isIntersecting || loading) return;
  loading = true;
  await loadNextPage();             // fetch + append rows
  loading = false;
}, { rootMargin: '200px' });        // start loading 200px before the user reaches the end

io.observe(sentinel);
```

Lazy loading images:

```js
const imgObserver = new IntersectionObserver((entries, obs) => {
  for (const e of entries) {
    if (e.isIntersecting) {
      e.target.src = e.target.dataset.src; // swap in the real URL
      obs.unobserve(e.target);             // load once, then stop watching
    }
  }
});
document.querySelectorAll('img[data-src]').forEach((img) => imgObserver.observe(img));
// For plain images, <img loading="lazy"> does this natively.
```

MutationObserver and ResizeObserver:

```js
const mo = new MutationObserver((mutations) => {
  for (const m of mutations) console.log(m.type, m.addedNodes.length);
});
mo.observe(document.querySelector('#chat-widget'), { childList: true, subtree: true });

const ro = new ResizeObserver((entries) => {
  const { width, height } = entries[0].contentRect;
  chart.resize(width, height);    // redraw the price chart to fit
});
ro.observe(document.querySelector('#chart-container'));
```

In Svelte 5, an attachment or action wraps this neatly and cleans up on destroy:

```svelte
<script>
  let visible = $state(false);

  function inView(node) {
    const io = new IntersectionObserver(([e]) => (visible = e.isIntersecting));
    io.observe(node);
    return { destroy: () => io.disconnect() };
  }
</script>

<div use:inView>{visible ? 'Live chart' : 'Paused'}</div>
```

## When to use it
- Pause live price updates or chart animation for widgets that are scrolled out of view, to save CPU and battery.
- Infinite scroll on trade history or news feed.
- ResizeObserver for charts inside resizable panels or a collapsible sidebar, where `window.resize` never fires.

## Likely questions
### Why is IntersectionObserver better than a scroll listener?
A scroll listener fires many times per second on the main thread, and calling `getBoundingClientRect()` inside it forces layout. IntersectionObserver lets the browser compute visibility itself, off your critical path, and only calls you when the visibility crosses a threshold. Less code, less jank.

### What do root, rootMargin and threshold mean?
`root` is the scroll container to check against (default: the viewport). `rootMargin` grows or shrinks that box, like `'200px'` to trigger early. `threshold` is how much of the element must be visible to fire: `0` means any pixel, `1` means fully visible, and an array like `[0, 0.5, 1]` fires at each step.

### When would you use MutationObserver?
When the DOM changes from code you do not own, such as a third-party script injecting a banner, or to watch an attribute like `class` or `aria-expanded`. Its callback runs as a microtask after the changes, with all mutations batched. Avoid changing the same nodes inside the callback, or you can create an infinite loop.

### Why not just listen to window resize?
`window.resize` only fires when the window changes. An element can change size when a sidebar collapses, content loads, or a flex parent changes. ResizeObserver catches all of those, per element.

## Common mistakes
- Forgetting `disconnect()` in a component, which keeps nodes and callbacks alive (a memory leak).
- Not guarding against double loads in infinite scroll (the `loading` flag above).
- Changing an element's size inside its own ResizeObserver callback, which can cause a loop; browsers report "ResizeObserver loop completed with undelivered notifications".

## Resources
- [MDN: Intersection Observer API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) - options and examples
- [MDN: MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver) - observe options and records
- [MDN: ResizeObserver](https://developer.mozilla.org/en-US/docs/Web/API/ResizeObserver) - entries and box sizes
- [web.dev: Browser-level image lazy loading](https://web.dev/articles/browser-level-image-lazy-loading) - native `loading="lazy"`
