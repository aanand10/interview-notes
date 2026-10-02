# Closures

A **closure** is a function bundled together with references to the variables of the scope it was created in. The function "remembers" those variables even after the outer function has returned.

## The one-sentence answer

> A closure lets an inner function access variables from an outer function's scope, even after the outer function has finished executing.

## Example: a counter

```js
function createCounter() {
	let count = 0; // private: nothing outside can touch it directly
	return {
		increment: () => ++count,
		get: () => count
	};
}

const counter = createCounter();
counter.increment();
counter.increment();
console.log(counter.get()); // 2
```

## Classic interview trap: `var` in loops

```js
for (var i = 0; i < 3; i++) {
	setTimeout(() => console.log(i), 0);
}
// 3, 3, 3: every callback closes over the SAME `i`

for (let i = 0; i < 3; i++) {
	setTimeout(() => console.log(i), 0);
}
// 0, 1, 2: `let` creates a new binding per iteration
```

## Where closures show up

- Data privacy / module pattern
- Function factories and partial application
- Event handlers and callbacks
- Memoization caches
- React hooks (stale closures in `useEffect`!)

## Gotchas

1. **Memory**: closures keep their captured variables alive. Large objects captured by long-lived callbacks won't be garbage collected.
2. **Stale values**: a closure captures the *variable*, not a snapshot. But in React each render creates a new scope, so old callbacks see old state.
