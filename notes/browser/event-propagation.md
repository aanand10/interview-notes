# Event propagation

> **In one line:** A DOM event travels in three phases: it goes down from `window` to the target (capturing), fires on the target, then goes back up (bubbling); `stopPropagation` stops that journey, `preventDefault` cancels the browser's default action, `target` is the element where the event started, and `currentTarget` is the element whose listener is running now.

## Key points
- **Three phases:** (1) capture: `window` to `document` to `html` down to the target's parent. (2) target: on the element itself. (3) bubble: back up to `window`. By default, listeners run in the bubble phase. Pass `{ capture: true }` (or `true`) to run in the capture phase instead.
- [`stopPropagation()`](https://developer.mozilla.org/en-US/docs/Web/API/Event/stopPropagation) stops the event from reaching any further elements. Other listeners on the **same** element still run. `stopImmediatePropagation()` stops those too.
- [`preventDefault()`](https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault) cancels the default browser behaviour (following a link, submitting a form, a checkbox toggling, scrolling on wheel or touch). It does **not** stop propagation.
- [`event.target`](https://developer.mozilla.org/en-US/docs/Web/API/Event/target) is the deepest element that started the event (what was actually clicked). [`event.currentTarget`](https://developer.mozilla.org/en-US/docs/Web/API/Event/currentTarget) is the element the running listener is attached to. It is `null` once the handler has finished.
- Not every event bubbles: `focus`, `blur`, `mouseenter`, `mouseleave`, `load` and `scroll` (on elements) do not. Check `event.bubbles`.

## Example
Run with Node and jsdom (the same rules as a browser):

```js
// <div id="outer"><div id="inner"><button id="btn">Buy</button></div></div>
outer.addEventListener('click', () => log('outer capture'), { capture: true });
inner.addEventListener('click', () => log('inner capture'), true);
btn.addEventListener('click', () => log('btn listener'));
inner.addEventListener('click', () => log('inner bubble'));
outer.addEventListener('click', (e) =>
  log(`outer bubble target=${e.target.id} currentTarget=${e.currentTarget.id}`));

btn.click();

// now add a stopper on inner (bubble phase) and click again
inner.addEventListener('click', (e) => e.stopPropagation());
btn.click();
```

```bash
outer capture
inner capture
btn listener
inner bubble
outer bubble target=btn currentTarget=outer
--- stopPropagation at inner bubble ---
outer capture
inner capture
btn listener
inner bubble
```
- Capture listeners run first, from the outside in.
- Then the target's own listener runs.
- Then bubble listeners run from the inside out. `target` is still `btn`, but `currentTarget` is `outer`.
- After `stopPropagation` on `inner`, the event never reaches `outer`'s bubble listener. Other listeners on `inner` (the logger) still ran.

`preventDefault` does not stop bubbling:

```js
// <form id="f"><a id="a" href="#x">link</a></form>
form.addEventListener('click', (e) => console.log('form saw click, defaultPrevented=' + e.defaultPrevented));
a.addEventListener('click', (e) => e.preventDefault());
a.click();
```

```bash
form saw click, defaultPrevented=true
```

## When to use it
- **Order form:** `preventDefault()` on `submit` so you can validate and send with `fetch` instead of a full page reload. Svelte 5 dropped the `|preventDefault` modifier, so you call `e.preventDefault()` inside `onsubmit`.
- **Row with a nested button:** clicking "Remove" in a watchlist row should not also open the stock details. Check `e.target.closest('button')` in the row handler, or use `stopPropagation` on the button (use it carefully).
- **Click outside to close** a dropdown: a listener on `document` checks whether `menu.contains(e.target)`.

## Likely questions

### Explain capturing vs bubbling.
When you click a button, the event first travels down from `window` through every ancestor to the button. That is capturing. Then it fires on the button. Then it travels back up through the ancestors. That is bubbling. `addEventListener(type, fn)` listens during bubbling. `addEventListener(type, fn, { capture: true })` listens during capturing, so an ancestor gets the event before the target does. Most code uses bubbling. Capture is useful when a parent must see the event first, for example for analytics or to intercept clicks.

### `stopPropagation` vs `preventDefault`?
They do different things. `stopPropagation` stops the event from moving to other elements, but the default action (like following a link) still happens. `preventDefault` stops the default action, but the event still bubbles to the parents. If you need both, call both. `stopImmediatePropagation` also stops the other listeners on the same element.

### `event.target` vs `event.currentTarget`?
`target` is where the event started: the innermost element clicked, like the `<span>` inside a button. `currentTarget` is the element whose listener is running now, the one you called `addEventListener` on. In event delegation, you attach to the `<ul>` (`currentTarget`) and use `target.closest('li')` to find which item was clicked. Inside an arrow function, `this` is not the element, so use `currentTarget`.

### Why is `stopPropagation` often a bad idea?
It hides the event from everything above it. That includes "click outside to close" logic, analytics listeners and delegated handlers that other code depends on. Those break in ways that are hard to debug. It is usually better to check `event.target` in the parent handler, or to set a flag, than to stop the event.

### What does `return false` do in a handler?
In an `addEventListener` handler, nothing special. In an inline `onclick="..."` HTML attribute or an `element.onclick` property, returning `false` cancels the default action, but it still does not stop propagation. (jQuery's `return false` did both, which is where the confusion comes from.)

### What does `composedPath()` give you?
An array of the elements the event will pass through, from the target up to `window`. It is useful with Shadow DOM, where `target` is changed to the shadow host when seen from outside the component.

## Common mistakes
- Thinking `preventDefault` stops bubbling, or that `stopPropagation` stops the default action.
- Reading `e.currentTarget` after an `await`. By then it is `null`, so store it in a variable first.
- Using `e.target` when the click landed on a child icon inside the button. Use `e.target.closest('button')`.
- Calling `preventDefault` inside a passive listener. It is ignored, and the browser logs a warning.

## Resources
- [javascript.info: Bubbling and capturing](https://javascript.info/bubbling-and-capturing) - clear diagrams of the three phases
- [MDN: Event bubbling](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Scripting/Event_bubbling) - beginner-friendly guide
- [MDN: Event.currentTarget](https://developer.mozilla.org/en-US/docs/Web/API/Event/currentTarget) - target vs currentTarget
- [javascript.info: Browser default actions](https://javascript.info/default-browser-action) - preventDefault in depth
