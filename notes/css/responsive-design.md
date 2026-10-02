# Responsive design

> **In one line:** Responsive design means one codebase that adapts to any screen - I write the mobile layout first, add `min-width` media queries to enhance it for bigger screens, and use relative units and `clamp()` so sizes flow smoothly instead of jumping.

## Key points
- **Mobile-first:** base styles are for small screens; `@media (min-width: ...)` adds layout for larger ones. Small devices download and apply the simplest CSS, and you add complexity only when there is room.
- **Media queries** react to the viewport (width, orientation, hover ability, user preferences like dark mode or reduced motion). **Container queries** react to the size of a parent, which is better for reusable components.
- **Relative units** scale with something else: `rem` (root font size), `em` (the element's own font size), `%` (the parent), `vw`/`vh` (viewport), `dvh` (the viewport height that changes as mobile browser bars show and hide).
- **Fluid typography with `clamp(min, preferred, max)`** lets text grow with the viewport but never below a minimum or above a maximum.
- Do not forget the viewport meta tag: `<meta name="viewport" content="width=device-width, initial-scale=1">`. Without it, mobile browsers render at about 980px and zoom out.

See [MDN: Responsive design](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/CSS_layout/Responsive_Design) and [web.dev Learn Responsive Design](https://web.dev/learn/design).

## Example

```css
/* Mobile first: one column, stacked */
.dashboard {
  display: grid;
  gap: 1rem;
  padding-inline: 1rem;
}

/* Tablet and up: two columns */
@media (min-width: 48rem) {          /* 48rem = 768px at default font size */
  .dashboard { grid-template-columns: 2fr 1fr; }
}

/* Desktop: three columns */
@media (min-width: 75rem) {
  .dashboard { grid-template-columns: 16rem 2fr 1fr; }
}

/* Fluid heading: 1rem at a 320px screen, 1.5rem at 1280px, smooth in between */
h1 {
  font-size: clamp(1rem, 0.8333rem + 0.8333vw, 1.5rem);
}

/* Full-height mobile panel that is not hidden behind the browser toolbar */
.sheet {
  min-height: 100vh;     /* fallback for old browsers */
  min-height: 100dvh;
}

/* Only add hover effects on devices that can hover */
@media (hover: hover) {
  .row:hover { background: var(--row-hover); }
}
```

How that `clamp()` was calculated (checked with node):

```js
const minPx = 16, maxPx = 24, minVw = 320, maxVw = 1280;
const slope = (maxPx - minPx) / (maxVw - minVw);
const vw = +(slope * 100).toFixed(4);
const interceptRem = +((minPx - slope * minVw) / 16).toFixed(4);
console.log(`clamp(1rem, ${interceptRem}rem + ${vw}vw, 1.5rem)`);
const size = (w) => Math.min(maxPx, Math.max(minPx, interceptRem * 16 + (vw / 100) * w));
for (const w of [320, 800, 1280, 1600]) console.log(w, 'px wide ->', +size(w).toFixed(2), 'px');
```

```text
clamp(1rem, 0.8333rem + 0.8333vw, 1.5rem)
320 px wide -> 16 px
800 px wide -> 20 px
1280 px wide -> 24 px
1600 px wide -> 24 px
```

## When to use it
- A trading app is used on phones a lot. The watchlist, order ticket and chart must work at 360px wide, then gain side-by-side panels on desktop.
- `dvh` for a bottom sheet order form on mobile so the Submit button is not hidden behind the browser bar.
- `rem` for font sizes and spacing, so if a user raises their browser font size for readability, the whole UI scales.
- `clamp()` for headings and hero numbers like portfolio value.

## Likely questions

### What does mobile-first mean and why do it?
I write the default CSS for the smallest screen and then use `min-width` media queries to add layout as space grows. The mobile CSS is usually simpler (single column), so building up is easier than overriding a desktop layout down. It also forces you to prioritise content, and the base CSS works even if a query fails.

### Explain `rem`, `em`, `%`, `vw` and `dvh`.
- `rem` is relative to the root (`html`) font size, usually 16px. Predictable; use it for font sizes and spacing.
- `em` is relative to the element's own font size (for `font-size` itself, the parent's). It compounds when nested, so a nested list at `1.2em` keeps growing. Useful for padding on a button that should scale with its text.
- `%` is relative to the parent - for `width` it is the parent's content width; for `padding` and `margin` it is also the parent's **width**, even for top/bottom.
- `vw`/`vh` are 1% of the viewport width/height.
- `dvh` is 1% of the *dynamic* viewport height, which updates when mobile browser bars appear or hide. `svh` is the smallest (bars shown), `lvh` the largest (bars hidden). `100vh` on mobile is the large one, which is why it overflows. See [MDN: length](https://developer.mozilla.org/en-US/docs/Web/CSS/length).

### How does `clamp()` work?
`clamp(MIN, PREFERRED, MAX)` returns the preferred value but never below MIN or above MAX. It is the same as `max(MIN, min(PREFERRED, MAX))`. For fluid type I use a preferred value like `0.83rem + 0.83vw`: the `rem` part keeps it respecting user zoom, the `vw` part makes it grow with the screen. Using only `vw` is bad for accessibility because text would not grow when the user zooms. See [MDN: clamp()](https://developer.mozilla.org/en-US/docs/Web/CSS/clamp).

### Why use `rem` instead of `px` in media queries?
`em`/`rem` breakpoints respect the user's browser font size setting. If someone sets a larger default font, an `em`-based breakpoint switches to the simpler layout earlier, which is what they need. In media queries, `rem` and `em` both use the browser's initial font size, not your `html` font size.

### Media queries vs container queries?
Media queries ask "how wide is the screen?". Container queries ask "how wide is the box I'm in?". A price card might sit in a narrow sidebar on desktop and full-width on mobile; only a container query handles both correctly. I use media queries for page layout and container queries for components.

### How do you make images responsive?
`max-width: 100%; height: auto;` so they never overflow. Use `srcset` and `sizes` (or `<picture>`) to ship smaller files to small screens, and set `width`/`height` attributes or `aspect-ratio` to avoid layout shift.

### How do you test responsive layouts?
DevTools device mode, real devices (especially iOS Safari for viewport quirks), zoom to 200% and 400% for accessibility, and Playwright tests at a few viewport sizes.

## Common mistakes
- Using `max-width` queries for a mobile-first codebase, or mixing both directions so rules fight.
- `100vh` on mobile hiding content under the browser toolbar - use `dvh` with a `vh` fallback.
- Pure `vw` font sizes that ignore user zoom.
- Breakpoints chosen by device names ("iPhone") instead of where the content breaks.
- Missing viewport meta tag.

## Resources
- [MDN: Responsive design](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/CSS_layout/Responsive_Design) - the full overview
- [MDN: Using media queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_media_queries/Using_media_queries) - syntax including range syntax like `(width >= 48rem)`
- [MDN: clamp()](https://developer.mozilla.org/en-US/docs/Web/CSS/clamp) - fluid sizing
- [MDN: CSS length units](https://developer.mozilla.org/en-US/docs/Web/CSS/length) - `rem`, `em`, `vw`, `dvh`, `svh`, `lvh`
- [web.dev Learn Responsive Design](https://web.dev/learn/design) - free course
