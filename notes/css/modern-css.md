# Modern CSS

> **In one line:** Modern CSS lets components adapt to their own container with container queries, lets a parent react to its children with `:has()`, supports Sass-like nesting natively, and uses logical properties so layouts work in both left-to-right and right-to-left languages.

## Key points
- **[Container queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_containment/Container_size_and_style_queries):** style an element based on the size of its **container**, not the whole viewport. Mark a parent with `container-type: inline-size`, then use `@container (min-width: 400px)`.
- **[`:has()`](https://developer.mozilla.org/en-US/docs/Web/CSS/:has):** the "parent selector". `.card:has(img)` matches a card that contains an image. It can also look at siblings and form state.
- **[Nesting](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_nesting):** write child rules inside the parent rule, with `&` for the parent. No Sass needed.
- **[Logical properties](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_logical_properties_and_values):** `margin-inline-start` instead of `margin-left`, `padding-block` instead of top and bottom. They flip automatically for right-to-left text.
- All four work in current Chrome, Firefox and Safari.

## Example
```css
/* 1. Container query: the same widget is compact in a sidebar, wide in the main area */
.widget-slot { container-type: inline-size; container-name: slot; }

.quote { display: grid; gap: 0.5rem; }
@container slot (min-width: 420px) {
  .quote { grid-template-columns: 1fr auto auto; } /* symbol | price | change in one row */
}

/* 2. :has() - parent reacts to child state */
.field:has(input:invalid) { border-color: var(--down); }   /* highlight the whole field */
.order-form:has(#type-limit:checked) .limit-price { display: block; }
.card:has(> img) { padding-top: 0; }

/* 3. Native nesting */
.watchlist {
  padding: 1rem;

  & .row {
    display: flex;
    &:hover { background: var(--surface-hover); }
  }

  @media (width < 600px) {
    padding: 0.5rem;
  }
}

/* 4. Logical properties */
.toast {
  inset-inline-end: 1rem;     /* right in LTR, left in RTL */
  inset-block-end: 1rem;      /* bottom */
  padding-inline: 1rem;       /* left + right */
  margin-block: 0.5rem;       /* top + bottom */
  inline-size: 20rem;         /* width in horizontal writing */
  border-inline-start: 4px solid var(--brand);
}
```

## When to use it
- **Container queries:** dashboard widgets (chart, quote, news) that users can place in narrow or wide panels.
- **`:has()`:** show the "limit price" input only when "Limit" is selected, style a form group with an error, all without JavaScript.
- **Logical properties:** apps that support Arabic or Hebrew.

## Likely questions
### Media query vs container query?
A media query looks at the viewport, so a component cannot know if it sits in a narrow sidebar on a wide screen. A container query looks at the size of a parent you mark with `container-type`, so the component is truly reusable. I use media queries for page layout and container queries for components.

### What is `:has()` and why is it a big deal?
It lets you select an element based on what it contains or what follows it, which CSS could never do before. Things like "style the label when its checkbox is checked" or "change the form when an input is invalid" no longer need JavaScript. Keep the selector inside `:has()` simple, since a very broad one can be slower to match.

### Is native CSS nesting the same as Sass nesting?
Mostly. One difference: in native CSS you cannot glue strings together like Sass `&__title` to build BEM names. `&` is a real selector, so `&__title` is not supported.

## Resources
- [MDN: Container queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_containment/Container_size_and_style_queries) - syntax and units
- [MDN: :has()](https://developer.mozilla.org/en-US/docs/Web/CSS/:has) - the parent selector
- [MDN: CSS nesting](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_nesting) - native nesting rules
- [MDN: Logical properties](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_logical_properties_and_values) - RTL-friendly layout
