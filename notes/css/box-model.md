# Box model

> **In one line:** Every element is a rectangle made of four layers - content, padding, border and margin - and `box-sizing` decides whether `width` means just the content or content plus padding plus border.

## Key points
- **Content** is where text and children go. **Padding** is space inside the border. **Border** wraps the padding. **Margin** is space outside the border that pushes other boxes away.
- By default (`box-sizing: content-box`), `width` sets only the content. Padding and border are added on top, so a `width: 200px` box with `padding: 20px` and `border: 1px` is really 242px wide.
- With `box-sizing: border-box`, `width` includes padding and border. The content shrinks to fit. This is what most teams set globally because it makes sizes predictable.
- **Margin collapsing:** vertical margins of block elements can merge into one margin equal to the larger value (not the sum). It never happens to horizontal margins, and never inside flex or grid containers.
- Background colour paints under content and padding (and under the border by default). Margin is always transparent.

See [MDN: The box model](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Styling_basics/Box_model).

## Example

```css
/* The usual global reset: make every box use border-box */
*,
*::before,
*::after {
  box-sizing: border-box;
}

.card {
  width: 300px;          /* border-box: total visible width is exactly 300px */
  padding: 16px;         /* content width becomes 300 - 16*2 - 1*2 = 266px */
  border: 1px solid #ddd;
  margin: 24px auto;     /* 24px top/bottom, auto left/right centres a block */
}
```

The maths for both modes, checked with node:

```js
const w = 200, padding = 20, border = 1;
console.log('content-box total width:', w + 2 * padding + 2 * border);
console.log('border-box content width:', w - 2 * padding - 2 * border);
```

```text
content-box total width: 242
border-box content width: 158
```

## When to use it
- Set `border-box` globally at the start of every project, so a price card or order-form input set to `width: 100%` with padding does not overflow its parent.
- When a layout "jumps" by a few pixels, or a column overflows, the box model is the first thing to check in DevTools (the Computed tab shows the box diagram).
- Use `gap` in flex/grid instead of margins between items. Gap never collapses and does not add space at the edges.

## Likely questions

### Explain the CSS box model.
Every element renders as a box with four layers. From inside out: content, padding, border, margin. Padding and border are part of the element itself, so they get its background. Margin is outside and just pushes neighbours away. The final size depends on `box-sizing`.

### What does `box-sizing: border-box` do and why do people use it?
It makes `width` and `height` include padding and border. So if I say `width: 300px`, the box is 300px on screen no matter how much padding I add; the content area shrinks instead. The default `content-box` adds padding and border on top, which makes things like `width: 100%` plus padding overflow the parent. That is why almost every reset sets `border-box` on all elements and pseudo-elements.

### What is margin collapsing?
When two vertical margins of block-level boxes touch, the browser uses only the bigger one instead of adding them. It happens in three cases:
1. **Adjacent siblings:** a `<p>` with `margin-bottom: 20px` followed by one with `margin-top: 30px` gives a 30px gap, not 50px.
2. **Parent and first/last child:** if the parent has no border, padding or inline content separating them, the child's top margin "leaks" out of the parent.
3. **Empty blocks:** an empty block with no height, padding or border has its own top and bottom margins merge.

If one margin is negative, the negative one is subtracted from the largest positive one.

```css
.a { margin-bottom: 20px; }
.b { margin-top: 30px; }
/* gap between .a and .b = max(20, 30) = 30px, not 50px */
```

### How do you stop margin collapsing?
Anything that separates the margins or creates a new block formatting context: add `padding` or `border` to the parent, use `display: flow-root` on the parent, or make the parent a flex or grid container (margins never collapse inside those). Using `gap` instead of margins avoids the problem entirely.

### Does margin collapsing happen horizontally or with flex items?
No. Only vertical (block-direction) margins in normal block flow collapse. Flex items, grid items, floats, absolutely positioned elements and inline-block elements never collapse their margins.

### What is the difference between `offsetWidth`, `clientWidth` and `scrollWidth`?
`offsetWidth` is content + padding + border (+ scrollbar). `clientWidth` is content + padding, without border and scrollbar. `scrollWidth` is the full width of the content including the part hidden by overflow. None of them include margin. [javascript.info: Element size and scrolling](https://javascript.info/size-and-scroll) has a good diagram.

### What happens with `margin: auto`?
On a block element with a set width, `margin-left: auto; margin-right: auto` splits the leftover space equally, so it centres horizontally. Vertical `auto` margins compute to 0 in normal flow, but inside a flex or grid container `margin: auto` centres in both directions.

## Common mistakes
- Forgetting `*::before, *::after` in the `border-box` reset, so pseudo-elements still use `content-box`.
- Expecting `margin-top: 20px` + `margin-bottom: 20px` to give 40px between two paragraphs.
- A child's `margin-top` pushing the whole parent down (parent-child collapse) and trying to "fix" it with more margin. Use `padding` or `display: flow-root` on the parent.
- Thinking `outline` or `box-shadow` take up space. They do not affect layout.

## Resources
- [MDN: The box model](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Styling_basics/Box_model) - beginner-friendly guide with diagrams
- [MDN: box-sizing](https://developer.mozilla.org/en-US/docs/Web/CSS/box-sizing) - exact reference
- [MDN: Mastering margin collapsing](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_box_model/Mastering_margin_collapsing) - all three collapsing cases
- [MDN: Block formatting context](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_display/Block_formatting_context) - why `flow-root` stops collapsing
- [web.dev Learn CSS: Box model](https://web.dev/learn/css/box-model) - short course chapter
