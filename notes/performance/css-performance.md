# CSS performance

> **In one line:** CSS gets slow when it forces the browser to redo layout or paint too often, so I animate only `transform` and `opacity`, keep selectors simple, and use `contain` and `content-visibility` to limit how much of the page the browser has to recalculate.

## Key points
- The [rendering pipeline](https://web.dev/articles/rendering-performance) is: **Style -> Layout -> Paint -> Composite**. The earlier the step you trigger, the more work follows it.
- **Layout properties** (`width`, `height`, `top`, `left`, `margin`) trigger layout on every frame when animated. **`transform` and `opacity`** can be done by the compositor (often on the GPU) and skip layout and paint.
- **Selectors**: modern browsers match selectors very fast. Selector cost only matters on huge DOMs with very frequent style changes. Deep, broad selectors like `.page div * span` are the ones to avoid.
- [`will-change`](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change) hints that an element will animate, so the browser can promote it to its own layer early. Use it sparingly.
- [`contain`](https://developer.mozilla.org/en-US/docs/Web/CSS/contain) tells the browser a box is independent, so changes inside it do not cause layout of the whole page. [`content-visibility: auto`](https://developer.mozilla.org/en-US/docs/Web/CSS/content-visibility) skips rendering of off-screen sections entirely.

## Example
```css
/* Bad: animates layout every frame */
.toast-bad { transition: top 300ms; }
.toast-bad.show { top: 20px; }

/* Good: compositor-only animation */
.toast { transform: translateY(-100%); opacity: 0; transition: transform 300ms, opacity 300ms; }
.toast.show { transform: translateY(0); opacity: 1; }

/* Hint before the animation starts, not on everything */
.drawer.opening { will-change: transform; }

/* A price ticker widget that updates every second:
   keep its layout and paint work inside its own box */
.ticker { contain: layout paint; }

/* Long news feed or order history: skip rendering off-screen sections */
.history-month {
  content-visibility: auto;
  contain-intrinsic-size: auto 600px; /* placeholder height so the scrollbar does not jump */
}
```

## When to use it
- Live price cells flashing green/red: change `background-color` or `opacity`, not `width`, and put `contain` on the row or widget.
- Slide-in order panel: animate `transform`.
- Long transaction history page: `content-visibility: auto` on each month block can cut initial render time a lot.

## Likely questions
### Are some CSS selectors expensive?
In theory yes: browsers match selectors right to left, so `.list *` checks every element. In practice, selector matching is rarely the bottleneck today. The bigger costs are large DOMs, frequent style recalculation and layout. I would only look at selectors if the Performance panel shows long "Recalculate Style" blocks.

### Why should you not animate `width` or `top`?
They change geometry, so the browser must redo layout, then paint, then composite, on every frame. On a slow phone that drops frames. `transform` and `opacity` usually only need compositing, so they stay smooth at 60 fps.

### What does `will-change` do, and why not put it everywhere?
It tells the browser ahead of time that a property will change, so it can create a separate layer before the animation. Each layer uses memory (GPU memory especially), so using it on many elements can make things slower. Add it just before an animation, or on a few elements that animate often, and remove it after.

### What is `contain`?
It promises the browser that an element's insides do not affect the outside. `contain: layout` means layout changes stay inside; `paint` means children do not draw outside the box; `size` means its size does not depend on children; `content` is shorthand for `layout paint` (plus style). The browser can then limit recalculation to that box.

### What is `content-visibility: auto`?
The browser skips layout and paint for the element while it is off-screen and renders it when it comes near the viewport. Pair it with `contain-intrinsic-size` so the page has a placeholder height and the scrollbar does not jump.

### What is layout thrashing?
Reading a layout value (like `offsetHeight`) right after writing a style, in a loop. Each read forces a synchronous layout. Fix: do all reads first, then all writes, or batch writes in `requestAnimationFrame`.

## Common mistakes
- `will-change: transform` on every card in a list.
- `transition: all`, which can animate layout properties by accident.
- Forgetting `contain-intrinsic-size`, so the scrollbar jumps as sections render.

## Resources
- [web.dev: Rendering performance](https://web.dev/articles/rendering-performance) - the pixel pipeline explained
- [web.dev: content-visibility](https://web.dev/articles/content-visibility) - real numbers and usage
- [MDN: contain](https://developer.mozilla.org/en-US/docs/Web/CSS/contain) - every value explained
- [MDN: will-change](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change) - when and when not to use it
