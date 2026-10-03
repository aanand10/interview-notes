# Stacking context

> **In one line:** A stacking context is a "layer group" in the page - `z-index` only compares elements inside the same group, so a child can never climb above something outside its parent's group, no matter how big its `z-index` is.

## Key points
- `z-index` decides which element is drawn on top along the z-axis (towards the user). It only works on **positioned** elements (`position` is not `static`) and on **flex or grid children**.
- A **stacking context** is like a folder. Everything inside is stacked together, then the whole folder is placed as **one unit** in its parent's stack. A child with `z-index: 9999` inside a folder with `z-index: 1` is still below a sibling folder with `z-index: 2`.
- Many properties create a new stacking context, not just `z-index`. The common surprise ones are `opacity` below 1, `transform`, `filter` and `will-change`.
- `isolation: isolate` creates a stacking context on purpose, with no other side effects. It is the clean fix when you want to "contain" z-index inside a component.
- Elements in the **top layer** (an open modal `<dialog>`, a `popover`, fullscreen) sit above everything, so you do not need `z-index` for them.

See [MDN: Stacking context](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_positioned_layout/Stacking_context) and [MDN: z-index](https://developer.mozilla.org/en-US/docs/Web/CSS/z-index).

## What creates a stacking context
- The root element (`<html>`).
- `position: absolute` or `relative` **with** a `z-index` other than `auto`.
- `position: fixed` or `sticky` (always, even without `z-index`).
- A flex or grid child with a `z-index` other than `auto`.
- `opacity` less than 1.
- `transform`, `filter`, `backdrop-filter`, `perspective`, `clip-path`, `mask` set to anything other than `none`.
- `mix-blend-mode` other than `normal`.
- `isolation: isolate`.
- `will-change` naming any of the properties above (e.g. `will-change: transform`).
- `contain: layout`, `paint`, `strict` or `content`, and `container-type: size` or `inline-size`.

## Example
The classic bug: a dropdown inside a card is hidden behind the next card.

```html
<div class="card animated">
  <div class="dropdown">Order type: Limit / Market / Stop</div>
</div>
<div class="card">Next card</div>
```

```css
.card { position: relative; background: white; }

/* transform creates a stacking context on the FIRST card */
.card.animated { transform: translateY(0); }

.dropdown {
  position: absolute;
  top: 100%;
  z-index: 9999; /* only competes INSIDE .card.animated */
}

/* The second card comes later in the HTML, so it paints on top of
   the whole first card, dropdown included. 9999 does not help. */

/* Fix 1: lift the whole first card (the context), not the child */
.card.animated { z-index: 2; }

/* Fix 2: remove the transform if it is not needed */
/* Fix 3: render the dropdown somewhere else (a portal, or the top layer with popover) */
```

## When to use it
- **Dropdowns, tooltips, toasts and modals** in a trading app: the order-type dropdown hidden behind the chart is almost always a stacking-context bug.
- Put `isolation: isolate` on reusable components (a price card, a chart widget) so their internal `z-index` values cannot leak out and fight the rest of the page.
- Keep a small **z-index scale** as tokens (e.g. `--z-dropdown: 10; --z-sticky: 20; --z-modal: 100; --z-toast: 200`) instead of random numbers.

## Likely questions
### I set `z-index: 9999` and the element is still behind. Why?
Two common reasons. First, the element may be `position: static`, so `z-index` is ignored. Second, and more often, one of its ancestors creates a stacking context, for example with `transform`, `opacity` or `filter`. Then the element can only be on top **within that ancestor**. I would find that ancestor in DevTools and raise the ancestor's `z-index`, or remove the property that creates the context.

### What creates a stacking context?
The root, positioned elements with a `z-index` that is not `auto`, `fixed` and `sticky` elements, flex or grid children with a `z-index`, and visual properties like `opacity < 1`, `transform`, `filter`, `clip-path`, `mix-blend-mode`, `will-change`, `contain` and `isolation: isolate`.

### What is the painting order inside one stacking context?
From back to front: the context's own background and borders, then children with negative `z-index`, then normal block boxes, then floats, then inline content, then positioned children with `z-index: auto` or `0`, then positive `z-index` children. If two elements have the same level, the one later in the HTML wins.

### How do you put a modal above everything without z-index fights?
Use `<dialog>` with `showModal()` or the `popover` attribute. Both put the element in the **top layer**, which is above every stacking context in the page. Otherwise, render the modal as a direct child of `<body>` (a "portal") with a high `z-index`.

## Common mistakes
- Thinking a bigger number always wins. It only wins against siblings in the same stacking context.
- Adding `will-change: transform` or a `transform` for an animation and accidentally creating a context that hides dropdowns.
- Forgetting that `position: fixed` inside a `transform`ed parent is positioned against that parent, not the viewport.

## Resources
- [MDN: Stacking context](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_positioned_layout/Stacking_context) - full list of what creates one
- [MDN: z-index](https://developer.mozilla.org/en-US/docs/Web/CSS/z-index) - how z-index values work
- [MDN: isolation](https://developer.mozilla.org/en-US/docs/Web/CSS/isolation) - the clean way to create a context
- [MDN: Top layer](https://developer.mozilla.org/en-US/docs/Glossary/Top_layer) - why dialogs and popovers sit above everything
