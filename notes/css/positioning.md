# Positioning

> **In one line:** The `position` property decides whether an element stays in normal flow and how `top`/`right`/`bottom`/`left` move it - `relative` nudges it, `absolute` places it against the nearest positioned ancestor, `fixed` places it against the viewport, and `sticky` acts normal until you scroll past a threshold and then sticks.

## Key points
- **`static`** (default): normal flow. `top`/`left` and `z-index` do nothing.
- **`relative`:** stays in flow and keeps its original space; offsets move it visually. Its main job is to become the **containing block** (the reference box) for absolute children.
- **`absolute`:** removed from flow (others act as if it is not there). Positioned against the nearest ancestor whose `position` is not `static`; if none, against the initial containing block (roughly the first screen of the page).
- **`fixed`:** removed from flow, positioned against the viewport, so it does not move on scroll. Gotcha: an ancestor with `transform`, `filter`, `perspective` or `contain: paint` becomes its containing block instead.
- **`sticky`:** stays in flow like `relative`, then sticks once you scroll it to the offset you set (e.g. `top: 0`). It only sticks within its parent box, and it sticks relative to the nearest scrolling ancestor.

See [MDN: position](https://developer.mozilla.org/en-US/docs/Web/CSS/position) and [MDN: Containing block](https://developer.mozilla.org/en-US/docs/Web/CSS/Containing_block).

## Example
A badge on a card (relative + absolute) and a sticky table header for an order book.

```css
/* 1. Badge in the corner of a card */
.card { position: relative; }          /* becomes the reference box */
.card .badge {
  position: absolute;
  top: 0.5rem;
  right: 0.5rem;                       /* inset: 0.5rem 0.5rem auto auto; is the shorthand */
}

/* 2. Sticky table header inside a scrolling panel */
.table-wrap {
  max-height: 24rem;
  overflow: auto;                      /* this is the scroll container sticky uses */
}
.orders thead th {
  position: sticky;
  top: 0;                              /* REQUIRED: sticky does nothing without an offset */
  z-index: 1;                          /* stay above the rows scrolling under it */
  background: var(--surface);          /* otherwise rows show through */
}

/* Sticky first column too (symbol column) */
.orders td:first-child,
.orders th:first-child {
  position: sticky;
  left: 0;
  background: var(--surface);
}
.orders thead th:first-child { z-index: 2; } /* corner cell above both */
```

```html
<div class="table-wrap">
  <table class="orders">
    <thead><tr><th>Symbol</th><th>Qty</th><th>Price</th></tr></thead>
    <tbody><!-- many rows --></tbody>
  </table>
</div>
```

## When to use it
- **relative + absolute:** badges, close buttons, tooltips and dropdowns anchored to a trigger.
- **fixed:** toast notifications, a "Buy/Sell" bar pinned to the bottom on mobile, modal backdrops.
- **sticky:** table headers in an order book or holdings table, section headers in a long watchlist, a sidebar that follows you while scrolling.

## Likely questions

### Explain the difference between relative, absolute, fixed and sticky.
`relative` keeps the element in flow and lets you nudge it; the space it left stays reserved. `absolute` takes it out of flow and positions it against the nearest positioned ancestor. `fixed` is also out of flow but positioned against the viewport, so it stays put when you scroll. `sticky` is a hybrid: it behaves like `relative` until scrolling would push it past its offset, then it sticks like `fixed`, but only while its parent is still on screen.

### How does an absolutely positioned element choose what it is positioned against?
It walks up the tree and uses the first ancestor whose `position` is anything but `static`. Also, an ancestor with `transform`, `filter`, `perspective`, `contain: layout/paint` or `will-change: transform` becomes the containing block. If nothing matches, it uses the initial containing block. That is why we add `position: relative` to the parent.

### How do you make a sticky table header?
Put `position: sticky; top: 0` on the `th` cells (not on `thead` or `tr` - cells are the most reliable), give them a background so rows don't show through, and a `z-index` so they sit above the body cells. The table must be inside the element that scrolls. If the scroll happens on the page, it sticks to the viewport; if it is in a wrapper with `overflow: auto` and a `max-height`, it sticks to the top of that wrapper.

### Why is my `position: sticky` not working?
The usual reasons:
1. No offset set (`top`, `bottom`, etc.). Sticky needs one.
2. An ancestor between the element and the real scroller has `overflow: hidden`, `auto` or `scroll`. That ancestor becomes the scroll container, and it is not scrolling, so nothing sticks.
3. The parent is the same height as the sticky element, so there is no room to stick (common in a flex row where the item stretches - add `align-self: flex-start`).
4. The `display: table` parts like `thead` in some older browsers; put it on `th`.

### Why is my `position: fixed` element scrolling with the page?
An ancestor has a `transform`, `filter`, `perspective` or similar. That ancestor becomes the containing block, so "fixed" is now fixed to it. The fix is to render the element outside that ancestor - for modals and toasts I render them at the end of `body` (a portal).

### Does `position: absolute` keep the element's space in the layout?
No. Siblings move up as if it is not there. `relative` keeps the space.

## Common mistakes
- Forgetting `position: relative` on the parent, so the badge flies to the corner of the page.
- Adding `overflow: hidden` to a wrapper (for rounded corners) and silently breaking sticky.
- Not giving sticky headers a background.
- Using `fixed` for something that should be `sticky`, then adding manual `padding-top` hacks to avoid overlap.

## Resources
- [MDN: position](https://developer.mozilla.org/en-US/docs/Web/CSS/position) - all values with live examples
- [MDN: Positioning guide](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/CSS_layout/Positioning) - step-by-step tutorial
- [MDN: Containing block](https://developer.mozilla.org/en-US/docs/Web/CSS/Containing_block) - exact rules for what absolute/fixed attach to
- [MDN: inset](https://developer.mozilla.org/en-US/docs/Web/CSS/inset) - the shorthand for top/right/bottom/left
