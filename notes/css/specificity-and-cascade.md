# Specificity and cascade

> **In one line:** When two rules set the same property, the cascade picks the winner by checking, in order: origin and `!important`, cascade layers, specificity, and finally which rule comes last in the source.

## Key points
- **Cascade order (simplified):** importance and origin (user-agent, user, author) first, then context (like scoped styles), then **layers**, then **specificity**, then **source order** (later wins).
- **Specificity** is a three-part score `(IDs, classes, elements)`, compared left to right. One ID beats any number of classes.
  - IDs `#buy` = (1,0,0)
  - Classes `.btn`, attributes `[type="text"]`, pseudo-classes `:hover` = (0,1,0)
  - Elements `button`, pseudo-elements `::before` = (0,0,1)
  - `*`, combinators (`>`, `+`, space) and `:where()` = 0. `:is()`, `:not()`, `:has()` take the specificity of their most specific argument.
- **Inline styles** (`style="..."`) beat any selector. **`!important`** beats inline styles (unless the inline one is also `!important`).
- **Cascade layers** (`@layer`) let you group CSS into ordered buckets. A later layer wins over an earlier layer **regardless of specificity**. Unlayered CSS beats all layers.

See [MDN: Specificity](https://developer.mozilla.org/en-US/docs/Web/CSS/Specificity) and [MDN: Introduction to the cascade](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascade/Cascade).

## Example

```css
/* Specificity scores */
button              { color: black; }  /* (0,0,1) */
.btn                { color: blue; }   /* (0,1,0) */
.form .btn:hover    { color: green; }  /* (0,3,0) - two classes + one pseudo-class */
#buy                { color: red; }    /* (1,0,0) - beats everything above */
:where(#buy)        { color: gray; }   /* (0,0,0) - :where always adds zero */

/* Cascade layers: order is declared once, up front */
@layer reset, base, components, utilities;

@layer components {
  #order-form .submit { background: navy; }  /* high specificity, but in an earlier layer */
}
@layer utilities {
  .bg-green { background: green; }           /* WINS: utilities layer is later */
}
```

```html
<form id="order-form"><button class="submit bg-green">Buy</button></form>
<!-- background is green, because layer order beats specificity -->
```

## When to use it
- Debugging "why is my style not applying?" - DevTools Styles panel shows crossed-out rules and the winner.
- Layers let a design system be overridable: put the library in `@layer vendor` and your own styles always win without specificity wars. Tailwind v4 itself outputs its CSS into `theme`, `base`, `components` and `utilities` layers.
- In Svelte, component styles are scoped by adding a hash class like `.svelte-xyz123`, which adds (0,1,0) to each selector - worth knowing when a global style is not overriding a component.

## Likely questions

### How is specificity calculated?
Count three buckets: IDs, then classes/attributes/pseudo-classes, then elements/pseudo-elements. Compare left to right, like version numbers. `#nav a` is (1,0,1), `.nav .link.active` is (0,3,0), so the ID one wins even though the other has more selectors. If specificity ties, the rule written later wins.

### What does `!important` do, and when is it OK?
It moves the declaration into a higher-priority group that beats normal declarations, including inline styles. Among `!important` rules, specificity and order apply again. One twist: for `!important`, layer order is **reversed**, so an important rule in an early layer beats one in a later layer. I avoid it in app code because the only way to override it is another `!important`. Fair uses: utility classes meant to always win, overriding third-party inline styles, and accessibility user stylesheets.

### What are cascade layers?
`@layer` lets me name buckets of CSS and set their priority once: `@layer reset, base, components, utilities;`. Rules in a later layer win over earlier layers no matter how specific the earlier selector is. Styles not in any layer beat all layered styles. It solves the classic problem of a third-party library having very specific selectors that are hard to override - put the library in a low layer. See [MDN: @layer](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer).

### What is the difference between `:is()` and `:where()`?
Both match any selector in the list. `:is()` takes the specificity of its most specific argument. `:where()` always has zero specificity, so it is perfect for default styles in a library that users should override easily. See [MDN: :where()](https://developer.mozilla.org/en-US/docs/Web/CSS/:where).

### Do inherited styles have specificity?
No. Inherited values have no specificity at all. Any rule that targets the element directly, even `*`, beats a value inherited from the parent. That is why setting `color` on a `div` does not change a link inside it if the browser has `a { color: ... }`.

### Is `.a.b` the same as `.a .b`?
No. `.a.b` (no space) is one element with both classes. `.a .b` is a `.b` inside an `.a`. Both happen to be (0,2,0).

## Common mistakes
- Thinking 11 classes beat 1 ID (they do not - buckets never overflow).
- Fixing a conflict by adding an ID or `!important`, starting a specificity war.
- Forgetting that unlayered CSS beats layered CSS, so a reset outside a layer overrides your components.
- Thinking `*` or combinators add specificity.

## Resources
- [MDN: Specificity](https://developer.mozilla.org/en-US/docs/Web/CSS/Specificity) - exact rules with examples
- [MDN: Handling conflicts](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Styling_basics/Handling_conflicts) - beginner guide to cascade, specificity, inheritance
- [MDN: @layer](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer) - cascade layers reference
- [web.dev Learn CSS: The cascade](https://web.dev/learn/css/the-cascade) - clear walkthrough of the cascade order
- [web.dev Learn CSS: Specificity](https://web.dev/learn/css/specificity) - scoring exercises
