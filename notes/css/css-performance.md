# CSS performance

> **In one line:** Make the browser do the least work per frame - animate only `transform` and `opacity` so it can skip layout and paint, avoid changing layout properties in hot paths, limit work with `contain` and `content-visibility`, and ship critical CSS first so the page paints fast.

## Key points
- **The rendering pipeline:** Style -> **Layout** (sizes and positions) -> **Paint** (pixels) -> **Composite** (combine layers on the GPU). Changing a property re-runs its step **and every step after it**.
- **Layout-triggering** properties: `width`, `height`, `top`, `left`, `margin`, `padding`, `font-size`, `border-width`. **Paint-only:** `color`, `background`, `box-shadow`. **Composite-only:** `transform`, `opacity`. So `transform` and `opacity` are the cheapest to animate. See [web.dev: compositor-only properties](https://web.dev/articles/stick-to-compositor-only-properties-and-manage-layer-count).
- **[`contain`](https://developer.mozilla.org/en-US/docs/Web/CSS/contain)** tells the browser a subtree is independent, so a change inside it does not force layout or paint of the rest of the page. **[`content-visibility: auto`](https://developer.mozilla.org/en-US/docs/Web/CSS/content-visibility)** skips rendering off-screen sections completely.
- **Critical CSS:** CSS is **render-blocking**. Inline the small CSS needed for the first screen in `<head>`, load the rest without blocking.
- `will-change: transform` promotes an element to its own GPU layer. Use it sparingly; every layer costs memory.

## Example
```css
/* BAD: animates left -> layout + paint every frame */
.toast-bad { position: fixed; left: -300px; transition: left 300ms; }
.toast-bad.show { left: 16px; }

/* GOOD: transform + opacity -> composite only, runs smoothly at 60fps */
.toast {
  position: fixed;
  left: 16px;
  transform: translateX(-120%);
  opacity: 0;
  transition: transform 300ms ease, opacity 300ms ease;
}
.toast.show { transform: translateX(0); opacity: 1; }

/* Price flash: animate opacity of an overlay, not background-color of every row */
.row { position: relative; }
.row::after {
  content: ""; position: absolute; inset: 0;
  background: var(--up-green); opacity: 0;
  transition: opacity 400ms;
}
.row.flash::after { opacity: 0.25; }

/* Contain each widget so a tick in one does not re-layout the page */
.widget { contain: layout paint; }       /* or contain: content */

/* Skip rendering long off-screen sections (news feed, history) */
.history-section {
  content-visibility: auto;
  contain-intrinsic-size: auto 600px;   /* placeholder height so the scrollbar does not jump */
}
```

```html
<!-- Critical CSS inline, rest loaded without blocking first paint -->
<head>
  <style>/* header, layout grid, above-the-fold styles only */</style>
  <link rel="preload" href="/app.css" as="style" onload="this.rel='stylesheet'">
  <noscript><link rel="stylesheet" href="/app.css"></noscript>
</head>
```

## When to use it
- **Live prices:** hundreds of rows updating every second. Use `transform`/`opacity` for flashes, `contain` on rows or widgets, and virtualise long lists.
- **Drawers, toasts, modals:** slide with `transform`, fade with `opacity`.
- **Landing and marketing pages:** critical CSS for a fast [LCP](https://web.dev/articles/lcp). SvelteKit can inline small CSS files with the `kit.inlineStyleThreshold` config option.

## Likely questions
### Why animate `transform` and `opacity` instead of `top` or `width`?
Changing `top` or `width` changes geometry, so the browser must redo layout, then paint, then composite, every frame. `transform` and `opacity` do not change layout, so the compositor can apply them on the GPU, often even when the main thread is busy with JavaScript. That keeps animations at 60fps.

### What is layout thrashing?
It is when JavaScript writes a style and then reads a layout value like `offsetHeight` in a loop. Each read forces the browser to do layout right now ("forced synchronous layout"). The fix is to batch all reads first, then all writes, or use `requestAnimationFrame`.

### What does `contain` do?
It promises the browser that an element's insides do not affect the outside. `contain: layout` means internal layout changes do not move anything outside. `contain: paint` means children do not draw outside the box. `contain: content` is `layout paint style`. `contain: strict` adds `size` too. The browser can then limit re-layout and re-paint to that box.

### What is critical CSS?
The minimum CSS needed to render what the user sees first. Because external CSS blocks rendering, you inline that small part in `<head>` and load the full stylesheet asynchronously. It improves First Contentful Paint and LCP. The trade-off is that inlined CSS is not cached separately.

## Common mistakes
- Putting `will-change` on everything. Too many layers use a lot of GPU memory and can make things slower.
- Using `transition: all`, which can animate layout properties by accident.
- Large unused CSS bundles. Tools like Tailwind already generate only used classes; check Coverage in Chrome DevTools.

## Resources
- [web.dev: Stick to compositor-only properties](https://web.dev/articles/stick-to-compositor-only-properties-and-manage-layer-count) - why transform and opacity are cheap
- [web.dev: Extract critical CSS](https://web.dev/articles/extract-critical-css) - how and why to inline critical CSS
- [MDN: contain](https://developer.mozilla.org/en-US/docs/Web/CSS/contain) - all containment values
- [web.dev: content-visibility](https://web.dev/articles/content-visibility) - skip rendering off-screen content
