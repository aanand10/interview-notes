# Reflow vs repaint

> **In one line:** A reflow (layout) means the browser has to work out the size and position of elements again, a repaint means it only redraws pixels, and the cheapest changes are `transform` and `opacity`, which can skip both and go straight to compositing.

## Key points
- **[Reflow](https://developer.mozilla.org/en-US/docs/Glossary/Reflow) (layout):** caused by changes to geometry, such as width, height, margin, padding, border, font size, `top`/`left`, adding or removing elements, changing text, or resizing the window. It is the most expensive kind of change, because it can affect parents, children and siblings.
- **[Repaint](https://developer.mozilla.org/en-US/docs/Glossary/Repaint):** caused by changes to how things look but not where they are, such as `color`, `background-color`, `box-shadow`, `outline` or `visibility`. Layout is skipped, but pixels are redrawn.
- **Composite only:** changes to `transform` and `opacity` on an element that has its own layer. The compositor thread just moves or fades a layer it already painted, so the main thread is not involved.
- Every reflow is followed by a repaint, but a repaint does not need a reflow.
- **Reading** layout values (`offsetHeight`, `getBoundingClientRect()`, `scrollTop`, `getComputedStyle`) can force a reflow right away if the styles were changed just before. That is the cause of layout thrashing.

## Example
```css
/* Bad: animating `left` causes layout + paint + composite on every frame */
.toast-bad {
  position: fixed;
  left: -300px;
  transition: left 300ms ease;
}
.toast-bad.open { left: 16px; }

/* Good: transform and opacity only need composite, so it stays smooth on low-end phones */
.toast {
  position: fixed;
  left: 16px;
  transform: translateX(-120%);
  opacity: 0;
  transition: transform 300ms ease, opacity 300ms ease;
}
.toast.open {
  transform: translateX(0);
  opacity: 1;
}
```

```js
const row = document.querySelector('.price-row');

row.style.width = '300px';           // reflow + repaint (geometry changed)
row.style.backgroundColor = 'green'; // repaint only (looks different, same place)
row.style.transform = 'scale(1.02)'; // composite only (if the row is on its own layer)
row.style.opacity = '0.8';           // composite only
```

## When to use it
- **Price flash on a watchlist:** to flash a row green or red when the price ticks, animate `opacity` on an overlay element instead of changing `background-color` on hundreds of rows. Do not change padding or font weight on update, because the row size would change and cause reflow.
- **Sliding panels, modals, toasts, order-ticket drawers:** move them with `transform: translateX/Y`, not `left`, `top` or `width`.
- **Live ticker:** use `transform: translateX()` for the scrolling strip, not `margin-left`.

## Likely questions

### What is the difference between reflow and repaint?
Reflow means the browser has to recompute the geometry: where boxes are and how big they are. Repaint means it only redraws pixels for boxes whose position has not changed. Reflow is more expensive, because one change can move many other elements, and it always ends in a repaint too. A repaint alone is cheaper but still runs on the main thread.

### What triggers a reflow?
- Changing size or position styles: `width`, `height`, `padding`, `margin`, `border`, `top`, `left`, `font-size`, `line-height`, `display`.
- Adding, removing or moving DOM nodes, or changing text content.
- Resizing the window, changing the font when a web font loads, or scrolling in some layouts.
- Reading layout values after a write: `offsetWidth/Height`, `clientWidth`, `scrollTop`, `getBoundingClientRect()`, `getComputedStyle()`, `innerText`. The browser must flush the pending layout to give you a correct number.

### Which CSS properties are cheap to animate, and why?
`transform` (translate, scale, rotate) and `opacity`. When the element is on its own compositor layer, the browser does not need layout or paint. The compositor thread just moves, scales or fades the already-painted layer on the GPU. Because that thread is separate from the main thread, the animation stays smooth even while JavaScript is busy. In Chrome, `filter` can often be composited as well.

### What does `will-change` do? Should I put it everywhere?
[`will-change: transform`](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change) tells the browser that an element is about to be animated, so it can give it its own layer ahead of time. That avoids a stutter on the first frame. But every layer costs GPU memory, so adding it to many elements can make things slower and crash low-end phones. Add it only to elements that really animate, and ideally only just before the animation starts.

### How would you find reflow problems?
In the Chrome DevTools Performance panel, record while you use the page. Look for purple "Layout" blocks, and especially for "Forced reflow" warnings, which link to the line of JS that caused them. The Rendering tab has "Paint flashing" (green boxes show areas being repainted) and "Layer borders".

### How can you limit how far a reflow spreads?
CSS [`contain: layout paint`](https://developer.mozilla.org/en-US/docs/Web/CSS/contain) tells the browser that what happens inside an element does not affect the outside. That keeps a widget's layout work local. [`content-visibility: auto`](https://developer.mozilla.org/en-US/docs/Web/CSS/content-visibility) skips layout and paint for off-screen sections completely.

## Common mistakes
- Animating `width`, `height`, `top` or `left` for slide or expand effects.
- Saying "`transform` never repaints". It skips layout, and it skips paint only when the element has its own layer.
- Adding `will-change` to everything, which wastes memory.
- Changing a class on `<body>` that restyles the whole page, when you could change a class on the small component.

## Resources
- [web.dev: Stick to compositor-only properties](https://web.dev/articles/stick-to-compositor-only-properties-and-manage-layer-count) - transform/opacity and layer count
- [web.dev: Rendering performance](https://web.dev/articles/rendering-performance) - the pixel pipeline diagram
- [web.dev: How to create high-performance CSS animations](https://web.dev/articles/animations-guide) - practical examples
- [MDN: Reflow](https://developer.mozilla.org/en-US/docs/Glossary/Reflow) - short definition
- [MDN: will-change](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change) - when and when not to use it
