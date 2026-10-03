# Component design patterns

> **In one line:** In React you build flexible UI by composing small components (passing children and props) instead of inheritance, and patterns like compound components, container/presentational and controlled/uncontrolled help you design reusable pieces.

## Key points
- **Composition over inheritance:** React never uses class inheritance between components. You pass `children` or other elements as props to customise.
- **Compound components:** a parent shares state with its children through [context](https://react.dev/learn/passing-data-deeply-with-context), so users write `<Tabs><Tabs.Tab/>...</Tabs>`, like `<select>` and `<option>`.
- **Container / presentational:** one part fetches data and holds logic, the other only renders props. Today a custom hook often plays the "container" role.
- **Controlled vs uncontrolled:** controlled means the parent owns the value through props (`value` + `onChange`); uncontrolled means the component keeps its own state (often with a `defaultValue`).
- **Prop getters:** a hook returns functions like `getButtonProps()` that give you all the right props (aria, handlers) to spread, letting you own the markup.

## Example
Compound Tabs component.

```tsx
import { createContext, useContext, useState, type ReactNode } from "react";

const TabsCtx = createContext<{ active: string; setActive: (id: string) => void } | null>(null);

function Tabs({ defaultTab, children }: { defaultTab: string; children: ReactNode }) {
  const [active, setActive] = useState(defaultTab); // uncontrolled: owns its state
  return <TabsCtx.Provider value={{ active, setActive }}>{children}</TabsCtx.Provider>;
}

function Tab({ id, children }: { id: string; children: ReactNode }) {
  const ctx = useContext(TabsCtx)!;
  return (
    <button role="tab" aria-selected={ctx.active === id} onClick={() => ctx.setActive(id)}>
      {children}
    </button>
  );
}

function Panel({ id, children }: { id: string; children: ReactNode }) {
  const ctx = useContext(TabsCtx)!;
  return ctx.active === id ? <div role="tabpanel">{children}</div> : null;
}

Tabs.Tab = Tab;
Tabs.Panel = Panel;

// Usage: caller controls layout and order freely.
<Tabs defaultTab="open">
  <div role="tablist">
    <Tabs.Tab id="open">Open orders</Tabs.Tab>
    <Tabs.Tab id="history">History</Tabs.Tab>
  </div>
  <Tabs.Panel id="open"><OpenOrders /></Tabs.Panel>
  <Tabs.Panel id="history"><OrderHistory /></Tabs.Panel>
</Tabs>
```

## When to use it
- Compound components: Tabs, Accordion, Menu, Select in a design system.
- Controlled: when the parent must react to every change (validated order quantity, a modal opened from a URL param).
- Svelte 5 equivalent: context with `setContext` / `getContext`, and snippets for composition.

## Likely questions
### What does "composition over inheritance" mean in React?
Instead of `class FancyButton extends Button`, you make a `Button` that accepts `children` or props like `icon`, and wrap it. A `Card` that takes `header` and `children` can be reused for a stock card or a news card without subclassing.

### Controlled vs uncontrolled: how would you design a reusable Modal or Select?
Support both. If the caller passes `open` (or `value`), use it and call `onOpenChange` (or `onChange`) on changes: controlled. If not, keep internal state starting from `defaultOpen`: uncontrolled. Never switch between the two during a component's life; React warns about this for inputs.

```tsx
function Modal({ open, defaultOpen = false, onOpenChange, children }: ModalProps) {
  const [inner, setInner] = useState(defaultOpen);
  const isControlled = open !== undefined;
  const isOpen = isControlled ? open : inner;
  const setOpen = (v: boolean) => { if (!isControlled) setInner(v); onOpenChange?.(v); };
  // ...render with isOpen and setOpen
}
```

### What are prop getters?
A headless hook returns functions such as `getToggleProps()` that return the correct `aria-*` attributes and event handlers. You spread them on your own elements, and can pass your own handlers in to merge. Libraries like Downshift use this. It gives full markup control while the hook handles accessibility and behaviour.

### Is container/presentational still relevant?
The idea is: keep pure display components easy to test and reuse. With hooks, the "container" is often a custom hook like `useOrders()`, and the component that renders is presentational.

## Resources
- [react.dev: Passing data deeply with context](https://react.dev/learn/passing-data-deeply-with-context) - basis of compound components
- [react.dev: Sharing state between components](https://react.dev/learn/sharing-state-between-components) - controlled vs uncontrolled explained
- [react.dev: Passing props to a component](https://react.dev/learn/passing-props-to-a-component) - children and composition
