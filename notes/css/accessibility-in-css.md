# Accessibility in CSS

> **In one line:** CSS affects accessibility directly - keep a clearly visible focus style for keyboard users, respect `prefers-reduced-motion`, meet colour-contrast ratios, never show meaning by colour alone, and hide things carefully so screen readers still get what they need.

## Key points
- **Focus styles:** never remove `outline` without a replacement. Use [`:focus-visible`](https://developer.mozilla.org/en-US/docs/Web/CSS/:focus-visible) so the ring shows for keyboard users but not on every mouse click.
- **Reduced motion:** some users get dizzy from animation. [`prefers-reduced-motion: reduce`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion) tells you to remove or soften big movements.
- **Contrast (WCAG AA):** at least **4.5:1** for normal text, **3:1** for large text (about 24px, or about 18.5px bold), and **3:1** for UI parts like borders of inputs, icons and focus rings.
- **Hiding:** `display: none` and `visibility: hidden` hide from **everyone**, screen readers included. A **visually-hidden** class hides from eyes but keeps it for screen readers. `aria-hidden="true"` does the opposite (visible but silent).
- Do not use colour alone for meaning. Red/green for price down/up also needs an arrow or a minus/plus sign.

## Example
```css
/* 1. Focus ring for keyboard users only */
button:focus { outline: none; }               /* fine ONLY because of the next rule */
button:focus-visible {
  outline: 2px solid var(--focus, #2563eb);
  outline-offset: 2px;
}

/* 2. Reduced motion: tone down globally */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}

/* 3. Visually hidden, still read by screen readers */
.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip-path: inset(50%);
  white-space: nowrap;
  border: 0;
}

/* 4. Forced colours (Windows High Contrast): keep borders visible */
@media (forced-colors: active) {
  .chip { border: 1px solid CanvasText; }
}
```

```svelte
<!-- PriceChange.svelte: meaning not by colour alone + hidden text for screen readers -->
<script lang="ts">
  import { prefersReducedMotion } from 'svelte/motion';
  let { change }: { change: number } = $props();
  let up = $derived(change >= 0);
</script>

<span class:up class:down={!up} class:animate={!prefersReducedMotion.current}>
  <span aria-hidden="true">{up ? '▲' : '▼'}</span>
  {Math.abs(change).toFixed(2)}%
  <span class="visually-hidden">{up ? 'up' : 'down'}</span>
</span>
```

## When to use it
- **Order forms:** strong focus rings so keyboard traders can tab through quantity, price and "Place order" quickly.
- **Live tickers and flashing prices:** turn off flashes and auto-scrolling marquees under reduced motion.
- **Icon-only buttons** (close, refresh): add visually hidden text or `aria-label`.
- "Skip to content" link: visually hidden until focused.

## Likely questions
### Is `outline: none` bad?
Only if you do not replace it. Keyboard users need to see where focus is (WCAG 2.4.7 Focus Visible). I remove the default only together with a custom `:focus-visible` style with at least 3:1 contrast against the background.

### How do you handle `prefers-reduced-motion`?
I wrap non-essential animations in a media query, or write motion only inside `@media (prefers-reduced-motion: no-preference)` so the safe default is no motion. Reduced motion does not always mean zero motion: I swap big slides and parallax for simple fades. In JS I can check `matchMedia('(prefers-reduced-motion: reduce)')`, and Svelte has `prefersReducedMotion` in `svelte/motion`.

### What are the colour contrast rules?
WCAG AA asks for 4.5:1 for body text and 3:1 for large text and for UI parts and graphics. AAA asks for 7:1 for body text. I check them in Chrome DevTools (the colour picker shows the ratio) or with Lighthouse.

### How do you hide something visually but keep it for screen readers?
Use a visually-hidden utility: position absolute, 1px by 1px, `overflow: hidden`, clipped. Do not use `display: none` or `visibility: hidden`, because those remove it from the accessibility tree too. Tailwind has this built in as `sr-only`.

## Common mistakes
- Using `display: none` for "screen reader only" text.
- Focus ring hidden behind a sticky header or clipped by `overflow: hidden`.
- Light grey placeholder text that fails contrast.
- Changing the visual order with `order` or `flex-direction: row-reverse`, so tab order no longer matches what users see.

## Resources
- [MDN: :focus-visible](https://developer.mozilla.org/en-US/docs/Web/CSS/:focus-visible) - keyboard-only focus styles
- [MDN: prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion) - reduced motion media query
- [web.dev: Color and contrast accessibility](https://web.dev/articles/color-and-contrast-accessibility) - contrast ratios explained
- [web.dev: Learn Accessibility - Focus](https://web.dev/learn/accessibility/focus) - focus management and styling
