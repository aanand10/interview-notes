# Flexbox vs Grid

> **In one line:** Flexbox lays things out in one direction (a row or a column) and lets content decide the sizes; Grid lays things out in two directions (rows and columns together) and lets the container decide the sizes.

## Key points
- **Flexbox is one-dimensional.** Items flow along a main axis. Great for toolbars, nav bars, button groups, a row of chips, or "push this to the right".
- **Grid is two-dimensional.** You define rows and columns on the parent, then place children into cells or named areas. Great for page shells, dashboards, card galleries and forms with aligned labels.
- **Content-out vs layout-in:** flex starts from the size of the items and distributes leftover space. Grid starts from a track plan you define and fits items into it.
- They are not rivals. A normal app uses Grid for the page skeleton and Flexbox inside each piece (header contents, card footer).
- Both support `gap`, the alignment properties (`justify-*`, `align-*`) and `margin: auto` tricks.

Read more: [web.dev Learn CSS: Flexbox](https://web.dev/learn/css/flexbox) and [Grid](https://web.dev/learn/css/grid).

## Example
The classic live-coding task: header, sidebar, content, footer. Grid with named areas, collapsing to one column on mobile.

```html
<div class="layout">
  <header class="header">
    <span class="logo">TradeApp</span>
    <nav class="nav"><a href="/">Markets</a><a href="/orders">Orders</a></nav>
  </header>
  <aside class="sidebar">Watchlist</aside>
  <main class="content">Chart and order form</main>
  <footer class="footer">Market data delayed 15 min</footer>
</div>
```

```css
.layout {
  display: grid;
  min-height: 100dvh;                  /* fill the screen; footer sits at the bottom */
  grid-template-columns: 1fr;          /* mobile first: one column */
  grid-template-rows: auto auto 1fr auto;
  grid-template-areas:
    "header"
    "sidebar"
    "content"
    "footer";
  gap: 1rem;
}

.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.content { grid-area: content; min-width: 0; } /* lets wide tables/charts shrink instead of overflowing */
.footer  { grid-area: footer; }

@media (min-width: 48rem) {
  .layout {
    grid-template-columns: 16rem 1fr;  /* fixed sidebar, flexible content */
    grid-template-rows: auto 1fr auto;
    grid-template-areas:
      "header  header"
      "sidebar content"
      "footer  footer";
  }
}

/* Flexbox INSIDE the header: one row, logo left, nav right */
.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}
.nav { display: flex; gap: 0.75rem; }
```

Same shell with Flexbox only (to show you know the difference - it needs an extra wrapper):

```css
.page  { display: flex; flex-direction: column; min-height: 100dvh; }
.middle { display: flex; flex: 1; }          /* wrapper around sidebar + main */
.sidebar { flex: 0 0 16rem; }                /* don't grow, don't shrink, 16rem wide */
.content { flex: 1; min-width: 0; }
```

## When to use it
- **Grid:** the app shell (header/sidebar/main), a dashboard of widgets, a responsive card grid with `repeat(auto-fill, minmax(14rem, 1fr))`, an order form where labels and inputs line up in columns.
- **Flexbox:** the row inside a watchlist item (symbol, price, change pushed right), a toolbar of buttons, centring an icon in a button, a wrapping list of filter chips.

## Likely questions

### When would you use Flexbox and when Grid?
If I'm laying things out in one line and want the items to size themselves, I use Flexbox - a nav bar, a button row, a list item with a price on the right. If I need rows and columns to line up with each other, or I want the parent to control the structure, I use Grid - the page layout, a dashboard, a card gallery. In practice I use both: Grid for the skeleton, Flexbox inside the components.

### Build a header, sidebar, content, footer layout.
I'd use Grid with `grid-template-areas` because it reads like a picture of the layout and is easy to change at a breakpoint. Mobile first: one column with all four areas stacked. At a wider breakpoint: two columns, header and footer spanning both. `min-height: 100dvh` plus a `1fr` middle row keeps the footer at the bottom on short pages. (See the example above.)

### What does `flex: 1` mean?
It is shorthand for `flex-grow: 1; flex-shrink: 1; flex-basis: 0%`. The item starts from zero size and takes an equal share of free space. `flex: auto` is `1 1 auto`, so it starts from its content size first. `flex: none` is `0 0 auto`, a rigid item. See [MDN: flex](https://developer.mozilla.org/en-US/docs/Web/CSS/flex).

### What is the `fr` unit?
A fraction of the free space in a grid container. `grid-template-columns: 2fr 1fr` gives the first column two thirds of the leftover space after fixed tracks and gaps are taken out.

### How do you make a responsive card grid without media queries?
```css
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(14rem, 1fr));
  gap: 1rem;
}
```
Each card is at least 14rem; the browser fits as many columns as possible. `auto-fill` keeps empty tracks; `auto-fit` collapses empty tracks so a few cards stretch to fill the row.

### Why does my flex item overflow instead of shrinking?
Flex and grid items have `min-width: auto` by default, which means "no smaller than my content". A long symbol name or a wide table will push out. Fix it with `min-width: 0` on the item (or `overflow: hidden`).

### What is the difference between `justify-content` and `align-items`?
`justify-content` aligns along the main axis (horizontal in a row). `align-items` aligns along the cross axis (vertical in a row). If you switch to `flex-direction: column`, the axes swap. In Grid, `justify-*` is always the inline (row) direction and `align-*` the block (column) direction.

## Common mistakes
- Using Flexbox with percentage widths and `flex-wrap` to fake a grid, then fighting the last row alignment. Use Grid.
- Forgetting `min-width: 0` on a flex/grid child that holds a chart or long text.
- Mixing up `auto-fill` and `auto-fit`.
- Reordering with `order` or grid placement so visual order differs from DOM order. Keyboard and screen reader users follow the DOM, so tab order looks random.

## Resources
- [web.dev Learn CSS: Flexbox](https://web.dev/learn/css/flexbox) - clear walkthrough of every flex property
- [web.dev Learn CSS: Grid](https://web.dev/learn/css/grid) - tracks, `fr`, areas, auto-fill
- [MDN: grid-template-areas](https://developer.mozilla.org/en-US/docs/Web/CSS/grid-template-areas) - named area syntax
- [MDN: Relationship of grid to other layout methods](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_grid_layout/Relationship_of_grid_layout_with_other_layout_methods) - the official "flex vs grid" comparison
