# Centering

> **In one line:** Today I centre almost everything with flexbox (`justify-content` + `align-items: center`) or grid (`place-items: center`); for horizontal-only I use `margin-inline: auto`, and for overlays I use `position: absolute` with `inset: 0; margin: auto` or the `translate(-50%, -50%)` trick.

## Key points
- **Horizontal only:** `text-align: center` for inline content (text, inline images), `margin-inline: auto` for a block that has a width.
- **Both axes, modern:** flexbox or grid on the **parent**. These are the default answers in an interview.
- **Both axes, out of flow:** absolute positioning for overlays, badges, spinners on top of content.
- Vertical centering needs the parent to have a **height** (or `min-height`). Without extra height there is nothing to centre inside.
- Know at least one legacy way (`translate(-50%, -50%)`, `line-height`) because interviewers like to ask "what else?".

See [MDN: Center an element](https://developer.mozilla.org/en-US/docs/Web/CSS/Layout_cookbook/Center_an_element).

## Example
Six ways, from most to least common.

```css
/* 1. Flexbox: centre child in both axes */
.flex-center {
  display: flex;
  justify-content: center; /* main axis (horizontal by default) */
  align-items: center;     /* cross axis (vertical by default) */
  min-height: 100dvh;      /* parent needs height */
}

/* 2. Grid: shortest version */
.grid-center {
  display: grid;
  place-items: center;     /* align-items + justify-items */
  min-height: 100dvh;
}

/* 3. Flex or grid parent + margin: auto on the child */
.parent { display: flex; min-height: 20rem; }
.parent > .child { margin: auto; } /* auto margins eat the free space on all sides */

/* 4. Horizontal only: block with a width */
.container { max-width: 60rem; margin-inline: auto; }

/* 5. Absolute + transform: size of child is unknown */
.overlay-parent { position: relative; }
.spinner {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%); /* move back by half its own size */
}

/* 6. Absolute + inset + margin auto: child has a known size */
.modal {
  position: fixed;
  inset: 0;          /* top/right/bottom/left: 0 */
  margin: auto;
  width: 30rem;
  height: fit-content;
}

/* Bonus: single line of text vertically centred in a fixed-height pill */
.pill { height: 2rem; line-height: 2rem; text-align: center; }
```

## When to use it
- **Flex:** a loading spinner in an empty watchlist, an icon and label in a "Buy" button.
- **Grid `place-items`:** empty states, login card in the middle of the screen.
- **`margin-inline: auto`:** the main page container with a `max-width`.
- **Absolute/fixed:** a spinner over a chart while it loads, a confirmation modal (or just use `<dialog>`, which the browser centres for you).

## Likely questions
### Give me multiple ways to centre a div horizontally and vertically.
Flexbox with `justify-content: center` and `align-items: center` on the parent. Grid with `place-items: center`. A flex or grid parent with `margin: auto` on the child. Absolute positioning with `top: 50%; left: 50%; transform: translate(-50%, -50%)`. Or absolute with `inset: 0; margin: auto` when the child has a set size. I default to grid or flex because they need no magic numbers.

### Why does `margin: 0 auto` not centre vertically?
In normal block layout, `auto` vertical margins compute to 0. Auto margins only absorb vertical space inside a flex or grid container, or for absolutely positioned elements with `inset` set.

### Why use `translate(-50%, -50%)` instead of negative margins?
`top: 50%; left: 50%` puts the child's **top-left corner** in the centre. Percentages in `translate` are relative to the element's **own** size, so `-50%` pulls it back by half its width and height, even if the size is unknown. Negative margins only work when you know the exact size.

### What is the difference between `justify-content` and `align-items`?
`justify-content` works on the main axis, `align-items` on the cross axis. With `flex-direction: column` the axes swap, so `justify-content` becomes vertical.

## Common mistakes
- Putting `justify-content` on the child instead of the parent.
- Forgetting the parent height, so vertical centering "does nothing".
- `transform` centering can give blurry text at half pixels and also creates a stacking context.

## Resources
- [MDN: Center an element](https://developer.mozilla.org/en-US/docs/Web/CSS/Layout_cookbook/Center_an_element) - cookbook recipe
- [web.dev: Centering in CSS](https://web.dev/articles/centering-in-css) - compares 5 techniques with trade-offs
- [MDN: place-items](https://developer.mozilla.org/en-US/docs/Web/CSS/place-items) - grid shorthand
