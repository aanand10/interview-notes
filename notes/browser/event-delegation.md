# Event delegation

> **In one line:** Event delegation means putting one listener on a common parent instead of one on every child, and using `event.target.closest()` to work out which child was hit, which works because most events bubble up.

## Key points
- **Why:** 10,000 rows with a listener each means 10,000 function objects and registrations, so more memory and slower setup. One listener on the parent costs the same however many rows there are.
- **Dynamic items for free:** rows added later (new symbols in a watchlist) work right away with no new wiring. Rows that are removed leave no listeners behind to leak.
- **How:** listen on the container, find the item with `e.target.closest('[data-id]')`, check it is inside the container, then read `data-*` attributes to know what to do.
- **Limits:** it only works for events that bubble. `focus`/`blur`, `mouseenter`/`mouseleave`, `load`/`error` (on images and similar) and `scroll` on elements do not bubble. Use `focusin`/`focusout` and `mouseover`/`mouseout` instead, or listen in the capture phase.

## Example
A watchlist of 10,000 symbols with one click listener (run with Node + jsdom):

```js
const list = document.getElementById('watchlist');

// Build 10k rows in one DOM insertion with a DocumentFragment
const frag = document.createDocumentFragment();
for (let i = 0; i < 10000; i++) {
  const li = document.createElement('li');
  li.dataset.symbol = `SYM${i}`;
  li.innerHTML = `<span class="name">SYM${i}</span> <button class="remove" type="button">Remove</button>`;
  frag.appendChild(li);
}
list.appendChild(frag);

// ONE listener for all rows, now and in the future
list.addEventListener('click', (e) => {
  const btn = e.target.closest('button.remove');
  if (btn && list.contains(btn)) {
    const li = btn.closest('li');
    console.log('remove', li.dataset.symbol);
    li.remove();
    return;
  }
  const row = e.target.closest('li');
  if (row) console.log('open', row.dataset.symbol);
});

list.children[42].querySelector('.name').click();
list.children[7].querySelector('.remove').click();
console.log('items left:', list.children.length);

const li = document.createElement('li');            // added after setup
li.dataset.symbol = 'NEW'; li.textContent = 'NEW';
list.append(li);
li.click();
```

```bash
open SYM42
remove SYM7
items left: 9999
open NEW
```
- Clicking the inner `<span>` bubbles to the `<ul>`. `closest('li')` finds the row.
- The remove branch returns early, so the row is not also "opened".
- The new row works with no extra listener.

Svelte 5 version, with one handler on the list:

```svelte
<script>
  let { symbols } = $props();   // e.g. ['INFY', 'TCS', ...]
  let selected = $state(null);

  function onListClick(e) {
    const row = e.target.closest('li[data-symbol]');
    if (!row) return;
    if (e.target.closest('.remove')) symbols = symbols.filter((s) => s !== row.dataset.symbol);
    else selected = row.dataset.symbol;
  }
</script>

<ul onclick={onListClick}>
  {#each symbols as s (s)}
    <li data-symbol={s}>{s} <button class="remove" type="button">Remove</button></li>
  {/each}
</ul>
```
Note: Svelte 5 already delegates common events like `click` to the app root for you. So writing `onclick` on every `<li>` is also cheap in Svelte. Knowing the manual pattern still matters for vanilla JS and interviews.

## When to use it
- Long lists or tables: watchlist, order history, option chain, where each row has buttons (buy, sell, remove).
- Content that changes often: rows inserted from WebSocket updates.
- Menus, toolbars, tab bars: one listener on the bar.
- For 10,000 rows, also **virtualize**: render only the ~30 visible rows. Delegation fixes the listener cost, but 10,000 DOM nodes still cost memory and layout time.

## Likely questions

### What is event delegation and why use it?
Instead of attaching a handler to each child, you attach one to a parent, and let events bubble up to it. In the handler you work out which child was the source using `event.target`. You get less memory use, faster setup, automatic support for children added later, and fewer listeners to clean up, so fewer leaks.

### Implement it for a list of 10,000 items.
Build the items in a `DocumentFragment` and append once. Store an id on each item in a `data-` attribute. Add a single `click` listener on the `<ul>`. Inside it, do `const li = e.target.closest('li')`, check that `li` exists and `list.contains(li)`, then act on `li.dataset.id`. The code above does exactly this. Mention that at 10k items you would also use virtual scrolling.

### Why use `closest()` instead of checking `e.target.tagName === 'LI'`?
The click usually lands on a child, like a `<span>` or an icon inside the row or button. Then `target` is not the `<li>`. `closest(selector)` walks up from the target to the nearest matching ancestor, or the element itself, so it works however deep the click lands.

### Which events do not bubble, and how do you delegate them anyway?
`focus`, `blur`, `mouseenter`, `mouseleave`, `load`, `error` and `scroll` on elements. For focus, use the bubbling versions `focusin`/`focusout`. For hover, use `mouseover`/`mouseout`, or `pointerover`/`pointerout`. Or add the listener with `{ capture: true }`: the capture phase still passes through the parent even when the event does not bubble.

### Any downsides?
The handler runs for every click inside the container, so keep it cheap and return early. If some inner code calls `stopPropagation`, the delegated handler never gets the event. You also have to handle nested matching carefully (a button inside a row). And `currentTarget` is the container, not the item.

## Common mistakes
- Using `e.target` directly when the click hit a child element.
- Not checking that `closest()` found something inside **this** container (nested lists).
- Expecting `focus` or `mouseenter` to reach the parent.
- Thinking delegation alone makes 10k DOM rows fast. It only fixes the listeners. Virtualize the list too.

## Resources
- [javascript.info: Event delegation](https://javascript.info/event-delegation) - the pattern with exercises
- [MDN: Element.closest()](https://developer.mozilla.org/en-US/docs/Web/API/Element/closest) - finding the matching ancestor
- [MDN: focusin event](https://developer.mozilla.org/en-US/docs/Web/API/Element/focusin_event) - the bubbling alternative to focus
- [svelte.dev: Basic markup (event delegation)](https://svelte.dev/docs/svelte/basic-markup) - how Svelte 5 delegates events
