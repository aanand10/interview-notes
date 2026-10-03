# Virtual DOM and reconciliation

> **In one line:** React keeps a lightweight JavaScript copy of the UI (the virtual DOM), and on every state change it builds a new copy, compares it with the old one (reconciliation), and applies only the differences to the real DOM (commit).

## Key points
- The **virtual DOM** is just a tree of plain JS objects (React elements) that describe what the UI should look like. Creating objects is cheap; touching the real [DOM](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model) is expensive.
- **Reconciliation** (the "render phase") is the diffing step. React uses two cheap rules instead of a perfect tree diff: different element type means "throw away and rebuild", and **keys** identify list items.
- **Commit phase** is when React actually writes to the DOM, attaches refs and then runs effects. Render can be paused or thrown away; commit is always synchronous.
- **Fiber** is React's internal engine (since React 16) that splits rendering into small units of work so it can pause, prioritise and resume.
- **Svelte** has no virtual DOM. Its compiler turns components into code that updates exactly the DOM nodes that depend on a changed value.

## Example
```tsx
type Order = { id: string; symbol: string; qty: number };

function OrderList({ orders }: { orders: Order[] }) {
  return (
    <ul>
      {orders.map((o) => (
        // Stable, unique key from the data. React uses it to match
        // "the same item" between the old and new tree.
        <li key={o.id}>
          {o.symbol} x {o.qty}
          <input placeholder="note" /> {/* uncontrolled, keeps its own DOM state */}
        </li>
      ))}
    </ul>
  );
}
```

What React does when `orders` changes:
1. Calls `OrderList` again and gets a new element tree.
2. Compares it with the previous tree. `<ul>` is still `<ul>`, so it keeps that DOM node.
3. For the children, matches old and new `<li>` by `key`. Same key = update in place. New key = create. Missing key = remove.
4. In commit, it applies just those DOM changes.

## When to use it
You don't "use" the virtual DOM directly, but knowing it explains real bugs: in a watchlist that can be sorted or filtered, wrong keys make input text, focus or animations jump to the wrong row. It also explains why React re-runs the whole component function on each update while Svelte does not.

## Likely questions

### What is the virtual DOM and why does React use it?
It is an in-memory tree of JS objects describing the UI. When state changes, React re-runs the component, gets a new tree, diffs it against the old tree, and updates only what changed in the real DOM. The main benefit is a simple model: you describe the UI for the current state and React figures out the DOM operations. It is not "faster than the DOM"; it is a way to make declarative UI fast enough.

### How does the diffing algorithm work?
A general tree diff is O(n^3), which is too slow, so React uses an O(n) heuristic with two assumptions. First, if two elements have **different types** (for example `<div>` to `<span>`, or `<Chart>` to `<Table>`), React destroys the old subtree, including its state, and builds a new one. Second, for lists, the developer gives **keys** so React can tell which child is which across renders. Same type and same position (or same key) means React keeps the DOM node or component instance and only updates changed props.

### Why do keys matter? What is the index-as-key bug?
Keys tell React "this is the same item as before". If you use the array index as the key and then insert an item at the top, every item's index shifts. React thinks item 0 is still item 0, so it keeps the old component state and DOM (like typed input text or a checkbox) and attaches it to the wrong data.

```tsx
// Bug: add a stock to the top and the "note" text stays on row 0,
// now showing next to a different stock.
{stocks.map((s, i) => <Row key={i} stock={s} />)}

// Fix: a stable id from the data
{stocks.map((s) => <Row key={s.symbol} stock={s} />)}
```
Index keys are fine only if the list never reorders, filters or inserts, and the items have no state. Also never use `Math.random()` as a key: it remounts every row on every render.

### How does React compare element types?
For host elements it compares the tag string (`'div'` vs `'span'`). For components it compares the **function or class reference**. That is why you must never define a component inside another component: each render creates a new function, so the type is "different" every time and React remounts the child and loses its state.

```tsx
function Parent() {
  // Bad: new Child type every render -> remount, state lost, input loses focus
  const Child = () => <input />;
  return <Child />;
}
```
You can also use this on purpose: changing a `key` (for example `<OrderForm key={symbol} />`) forces a fresh component with reset state.

### What is the difference between the render phase and the commit phase?
In the **render phase** React calls your components and diffs the trees. It must be pure (no side effects) because React may run it more than once, pause it or throw it away. In the **commit phase** React applies DOM changes, sets refs, runs `useLayoutEffect` synchronously, and then schedules `useEffect`. Commit cannot be interrupted, so the user never sees a half-updated UI.

### What is Fiber?
Fiber is the reimplementation of React's core from React 16. Each component instance gets a "fiber" object, which is a unit of work in a linked tree. Because work is split into units, React can stop in the middle of rendering a big tree, handle something urgent like a keystroke, and then continue or restart. This is what makes concurrent features like `useTransition`, `useDeferredValue` and Suspense possible. React also keeps two trees (current and work-in-progress) and swaps them at commit.

### How is Svelte different?
Svelte is a compiler. At build time it knows which DOM nodes read which state, so it generates direct update code like "set this text node when `price` changes". There is no tree to rebuild and diff at runtime. In Svelte 5, runes (`$state`, `$derived`) use fine-grained signals, so only the parts that depend on a changed value re-run. React re-runs the whole component function and diffs; Svelte updates the exact node. Svelte still uses keys in `{#each items as item (item.id)}` for the same reason React does.

## Common mistakes
- Using index or random values as keys in lists that change.
- Declaring components inside components (new type every render).
- Thinking the virtual DOM makes React "always faster". Extra re-renders still cost CPU.
- Doing side effects in the render body; render can run more than once.
- Conditionally swapping wrapper types (`isMobile ? <div> : <section>`) and being surprised that child state resets.

## Resources
- [react.dev: Preserving and resetting state](https://react.dev/learn/preserving-and-resetting-state) - how position, type and key decide if state is kept
- [react.dev: Rendering lists](https://react.dev/learn/rendering-lists) - rules for keys and why index keys break
- [react.dev: Render and commit](https://react.dev/learn/render-and-commit) - the render vs commit phases explained simply
- [Svelte docs: What are runes?](https://svelte.dev/docs/svelte/what-are-runes) - how Svelte 5 tracks changes without a virtual DOM
