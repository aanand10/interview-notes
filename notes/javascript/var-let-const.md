# var, let, const

> **In one line:** `var` is function-scoped and can be redeclared, while `let` and `const` are block-scoped, sit in the temporal dead zone until their line runs, and `const` also cannot be reassigned.

## Key points
- **Scope:** [`var`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var) lives in the whole function. [`let`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let) and [`const`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const) live only inside the nearest `{ }` block.
- **Hoisting:** all three are hoisted (the engine knows about them before the code runs). `var` starts as `undefined`. `let`/`const` stay uninitialised, so reading them early throws a `ReferenceError`. That gap is the temporal dead zone (TDZ).
- **Reassignment:** `var` and `let` can be reassigned. `const` cannot. But `const` only locks the *binding*, not the value: you can still change an object's properties.
- **Redeclaration:** `var x` twice in the same scope is allowed. `let`/`const` twice in the same scope is a `SyntaxError`.
- **Global object:** a top-level `var` in a classic script becomes `window.x`. Top-level `let`/`const` do not.

| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | function | block | block |
| Hoisted value | `undefined` | TDZ (error) | TDZ (error) |
| Reassign | yes | yes | no |
| Redeclare in same scope | yes | no | no |
| Must initialise | no | no | yes |

## Example
```js
function demo() {
  if (true) {
    var a = 1;   // function-scoped: visible in all of demo()
    let b = 2;   // block-scoped: only inside this if-block
    const c = 3; // block-scoped and cannot be reassigned
  }
  console.log(a);          // 1
  console.log(typeof b);   // "undefined" (b does not exist out here)
}
demo();

const order = { qty: 10 };
order.qty = 20;            // fine: we changed a property, not the binding
// order = {};             // TypeError: Assignment to constant variable.
```

## When to use it
- Default to `const` (for example `const priceStore = ...`, `const API_URL = ...`). It tells the reader "this name will not point to something else".
- Use `let` when the value really changes, like a loop counter or a retry count in a WebSocket reconnect.
- Avoid `var` in new code. You still need to know it because interview output questions and old code use it.

## Likely questions

### What is the difference between var, let and const?
`var` is function-scoped, hoisted with the value `undefined`, and can be redeclared. `let` and `const` are block-scoped and hoisted but stay in the temporal dead zone until their declaration runs, so using them early throws. `let` can be reassigned, `const` cannot. `const` does not make objects immutable; use `Object.freeze` for that.

### What does this loop print with var vs let?
```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log('var', i), 0);
for (let j = 0; j < 3; j++) setTimeout(() => console.log('let', j), 0);
```
Output (verified with node):
```text
var 3
var 3
var 3
let 0
let 1
let 2
```
- `var 3` x3: there is only one `i` for the whole function. The callbacks run after the loop ends, when `i` is already 3.
- `let 0 1 2`: `let` in a `for` head creates a **new binding per iteration**, so each callback closes over its own copy.

### How do you fix the var version without changing it to let?
Wrap the body in an IIFE (an immediately invoked function) so each callback gets its own parameter, or pass the value as the third argument of `setTimeout`.
```js
for (var i = 0; i < 3; i++) {
  ((n) => setTimeout(() => console.log(n), 0))(i); // 0 1 2
}
for (var k = 0; k < 3; k++) setTimeout((n) => console.log(n), 0, k); // 0 1 2
```

### Can you change a const object or array?
Yes. `const` stops you from pointing the name at a new value. It does not freeze what it points to. `const list = []; list.push(1)` works. `Object.freeze(list)` makes it shallowly read-only.

### Why is redeclaring with var dangerous?
In a long file two people can both write `var total` and silently overwrite each other. With `let`, the second declaration is a `SyntaxError`, so the bug is caught before the code even runs.

### Does a top-level var become a property of window?
In a classic browser script, yes: `var x = 1` gives `window.x === 1`. `let`/`const` at the top level do not. In ES modules (which Svelte/Vite use), nothing at the top level leaks onto `window`, because each module has its own scope.

## Common mistakes
- Saying `let`/`const` are "not hoisted". They are hoisted; they just are not initialised (TDZ).
- Thinking `const` makes an object immutable.
- `const x;` with no value is a `SyntaxError`.
- `typeof` is not safe in the TDZ: `typeof x; let x;` throws `ReferenceError`, while `typeof undeclaredName` returns `"undefined"`.

## Resources
- [MDN: var](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var) - function scope and hoisting rules
- [MDN: let](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let) - block scope and the TDZ section
- [MDN: const](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const) - binding vs value
- [javascript.info: The old "var"](https://javascript.info/var) - clear side-by-side with let
