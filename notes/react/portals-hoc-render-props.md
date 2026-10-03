# Portals, HOCs and render props

> **In one line:** A portal renders children into a different DOM node (like `document.body`) while keeping them in the same React tree, and HOCs and render props are older patterns for sharing logic that custom hooks have mostly replaced.

## Key points
- [`createPortal(children, domNode)`](https://react.dev/reference/react-dom/createPortal) from `react-dom` puts a modal or toast outside a parent with `overflow: hidden` or a low `z-index`.
- Events from a portal **bubble through the React tree**, not the DOM tree. A click inside a portal modal reaches the `onClick` of its React parent. Context also still works.
- **HOC (higher-order component):** a function that takes a component and returns a new one with extra props, like `withAuth(Page)`.
- **Render props:** a component takes a function as a prop (often `children`) and calls it with data, like `<MousePosition>{(pos) => ...}</MousePosition>`.
- **Hooks replaced most of both** because they share logic without wrapping components, so there is no "wrapper hell" and no prop name clashes.

## Example
A toast rendered with a portal.

```tsx
import { createPortal } from "react-dom";

function OrderToast({ message, onClose }: { message: string; onClose: () => void }) {
  return createPortal(
    <div role="status" className="toast">
      {message} <button onClick={onClose}>Dismiss</button>
    </div>,
    document.body // rendered at the end of body, escapes overflow/z-index of parents
  );
}

function OrderPanel() {
  const [msg, setMsg] = useState<string | null>(null);
  return (
    // This onClick ALSO fires for clicks inside the toast (React-tree bubbling).
    <section onClick={() => console.log("panel clicked")}>
      <button onClick={() => setMsg("Order placed: 10 AAPL")}>Buy</button>
      {msg && <OrderToast message={msg} onClose={() => setMsg(null)} />}
    </section>
  );
}
```

HOC vs render prop vs hook for the same "live price" logic:

```tsx
// HOC
const withLivePrice = (Comp) => (props) => <Comp {...props} price={useLivePrice(props.symbol)} />;

// Render prop
<LivePrice symbol="AAPL">{(price) => <span>{price}</span>}</LivePrice>

// Hook (modern)
const price = useLivePrice("AAPL");
```

## When to use it
- Portals: modals, toasts, tooltips, dropdown menus that must sit above everything.
- HOCs: still seen in older code and some libraries (`connect` from old react-redux, `withRouter`). Render props still useful when a component must control *what* to render, like a virtualized list's row renderer.
- Svelte 5 equivalent: no built-in portal, you move the node with an action/attachment; snippets play the role of render props.

## Likely questions
### How does event bubbling work with portals?
React events follow the component tree. So a click in a portal modal bubbles to the modal's React parents, even though in the DOM the modal is under `body`. That is useful, but it can surprise you: a parent's "click outside to close" handler might fire. Call `e.stopPropagation()` inside the portal if needed.

### What is an HOC, and its problems?
It is a function `withX(Component)` that returns a wrapped component with extra props. Problems: many wrappers nest deeply in DevTools, props can clash when two HOCs inject the same name, it is unclear where a prop came from, and typing them in TypeScript is awkward.

### Why did hooks replace HOCs and render props?
Hooks let you reuse stateful logic by just calling a function, `useLivePrice(symbol)`, inside any component. No extra components in the tree, clear data source, easy to combine, and good TypeScript inference.

## Resources
- [react.dev: createPortal](https://react.dev/reference/react-dom/createPortal) - API and event bubbling notes
- [react.dev: Reusing logic with custom hooks](https://react.dev/learn/reusing-logic-with-custom-hooks) - the modern replacement
- [legacy.reactjs.org: Higher-order components](https://legacy.reactjs.org/docs/higher-order-components.html) - the classic HOC pattern
- [legacy.reactjs.org: Render props](https://legacy.reactjs.org/docs/render-props.html) - the classic render prop pattern
