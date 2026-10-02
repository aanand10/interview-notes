# Reconciliation & keys

**Reconciliation** is how React decides what changed between two renders and what to update in the DOM.

## Two heuristics

1. Elements of **different types** produce different trees, so React tears down the old subtree.
2. **Keys** tell React which children are the same across renders.

## Why index keys are risky

```jsx
{items.map((item, index) => (
	<TodoRow key={index} item={item} />
))}
```

If you insert an item at the start, every index shifts. React reuses the wrong component instances, and local state (like an `<input>` value) ends up attached to the wrong row.

Prefer a stable id:

```jsx
{items.map((item) => (
	<TodoRow key={item.id} item={item} />
))}
```

## Resetting state with a key

Changing a component's `key` forces React to remount it, which is a clean way to reset a form when switching between records.

```jsx
<ProfileForm key={userId} userId={userId} />
```
