# Passive event listeners

> **In one line:** `{ passive: true }` is a promise that your touch or wheel handler will never call `preventDefault()`, so the browser can start scrolling right away instead of waiting for your JavaScript to finish.

## Key points
- For `touchstart`, `touchmove` and `wheel`, the browser does not know if your handler will cancel the scroll with `preventDefault()`. So by default it must **wait for your handler to run** before it scrolls.
- If the main thread is busy (parsing JSON, rendering a chart), that wait makes scrolling feel stuck. On mobile this is very visible.
- `passive: true` removes the wait. Scrolling happens on the compositor thread, in parallel with your JS.
- If you call `preventDefault()` inside a passive listener, it is ignored and the console shows a warning.
- Chrome and Firefox already treat `touchstart`, `touchmove`, `wheel` and `mousewheel` listeners on `window`, `document` and `body` as passive by default. On other elements you should set it yourself.

## Example
```js
// BAD on mobile: browser waits for this handler before scrolling
document.querySelector('.feed').addEventListener('touchmove', (e) => {
  trackScroll(e);
});

// GOOD: scrolling starts immediately, handler runs alongside
document.querySelector('.feed').addEventListener('touchmove', (e) => {
  trackScroll(e);
}, { passive: true });

// When you REALLY need to block scrolling (custom swipe-to-delete, chart pan)
chartEl.addEventListener('touchmove', (e) => {
  e.preventDefault();   // works only because passive is false
  panChart(e.touches[0]);
}, { passive: false });
```

A better way to stop scrolling on an element is often CSS, with no JS at all:

```css
/* Let the browser know this element handles horizontal pans itself */
.price-chart {
  touch-action: pan-y; /* page can scroll vertically; horizontal swipes go to the chart */
}
```

In Svelte 5 you can pass options with the `on` function from `svelte/events`, or use an action. Svelte already makes `ontouchstart` and `ontouchmove` attributes passive:

```svelte
<script>
  import { on } from 'svelte/events';

  function track(node) {
    const off = on(node, 'wheel', () => console.log('wheel'), { passive: true });
    return { destroy: off };
  }
</script>

<div use:track class="feed">...</div>
```

## When to use it
- Any scroll analytics, "scroll to top" button logic, sticky headers or parallax that reads touch or wheel events but does not block them.
- Use `passive: false` only for custom gestures: a draggable chart, a bottom sheet, pinch zoom on a candlestick chart.

## Likely questions
### Why does `{ passive: true }` improve scroll performance?
Because the browser no longer has to wait for your handler before scrolling. Without it, every touchmove or wheel event has to go to the main thread, run your JS, and check if `preventDefault()` was called. If the main thread is busy, scrolling stalls. With passive, the compositor thread scrolls right away and your handler runs when the main thread is free.

### What happens if I call preventDefault in a passive listener?
Nothing happens to the scroll; the call is ignored, and the browser logs a console warning. If you need to block scrolling, set `passive: false` explicitly, or better, use CSS `touch-action`.

### Which events does this matter for?
Mostly `touchstart`, `touchmove`, `wheel` and `mousewheel`, because those can cancel scrolling. The `scroll` event itself cannot be cancelled, so `passive` makes no difference for it.

### How do you detect if passive is supported?
Old trick: pass an options object with a getter for `passive`, and see if the browser reads it. Today every modern browser supports it, so this is rarely needed.

## Common mistakes
- Putting `passive: true` on `scroll` and thinking it helps; scroll is already non-blocking.
- Forgetting that the defaults only apply to window/document/body. A listener on a scroll container still blocks unless you mark it passive.
- Using JS `preventDefault()` when `touch-action` or `overscroll-behavior` CSS would do the job.

## Resources
- [MDN: addEventListener options (passive)](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener) - the options object and passive defaults
- [MDN: touch-action](https://developer.mozilla.org/en-US/docs/Web/CSS/touch-action) - CSS way to control gestures
- [developer.chrome.com: Making touch scrolling fast by default](https://developer.chrome.com/blog/scrolling-intervention) - why Chrome made them passive by default
