# Hoisting and TDZ

> **In one line:** Before running a scope, JavaScript registers all its declarations, so they are "hoisted"; function declarations are fully ready, `var` is set to `undefined`, and `let`/`const`/`class` exist but cannot be touched until their line runs, which is the temporal dead zone.

## Key points
- [Hoisting](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting) is not code moving. The engine does a setup pass over each scope and creates the variables first, then runs the code top to bottom.
- **Function declarations** are hoisted with their body, so you can call them before the line they are written on.
- **`var`** is hoisted and set to `undefined`. Reading it early gives `undefined`, not an error.
- **`let`, `const`, `class`** are hoisted but uninitialised. Reading them before the declaration throws `ReferenceError: Cannot access 'x' before initialization`. The time between the start of the block and the declaration is the [temporal dead zone (TDZ)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let#temporal_dead_zone_tdz).
- **Function expressions and arrow functions** follow the rules of the variable that holds them (`var` gives `undefined`, `let`/`const` give TDZ).

## Example
```js
console.log(typeof a, a);  // undefined undefined   -> var is hoisted as undefined
var a = 5;

console.log(b);            // ReferenceError: Cannot access 'b' before initialization (TDZ)
let b = 1;

console.log(sayHi());      // hi   -> function declaration is fully hoisted
function sayHi() { return 'hi'; }

greet();                   // TypeError: greet is not a function
var greet = function () { return 'hello'; }; // greet is undefined at call time

arrow();                   // ReferenceError: Cannot access 'arrow' before initialization
const arrow = () => 1;
```
Each error line was run separately in node to confirm the message.

## When to use it
- Use function declarations for helper functions at the bottom of a file (for example `formatPrice`, `calcPnL`) and call them from the top. Hoisting makes that readable.
- Rely on the TDZ as a safety net: with `const`/`let`, using a config value before it is set fails loudly instead of quietly being `undefined` (which could send a wrong order quantity).

## Likely questions

### What prints when a variable is used before it is declared?
It depends on the keyword. With `var` you get `undefined`. With `let` or `const` you get a `ReferenceError` because of the TDZ. With no declaration at all you get `ReferenceError: x is not defined`.

### Function declaration vs function expression: how does hoisting differ?
A function declaration (`function foo() {}`) is hoisted with its body, so calling it early works. A function expression (`var foo = function () {}`) only hoists the variable. With `var`, `foo` is `undefined`, so calling it gives `TypeError: foo is not a function`. With `const`, it is in the TDZ, so you get a `ReferenceError`.

### What is the temporal dead zone?
It is the period from the start of a block until the `let`/`const`/`class` line runs. The variable already exists (the engine reserved it), but any read or write throws. It is called "temporal" because it depends on time of execution, not position in code:
```js
function show() { return value; } // fine to reference here
const value = 10;
show(); // 10 -> called after the declaration ran, so no TDZ
```

### Output question: shadowing plus hoisting
```js
var p = 1;
function sh() {
  console.log(p); // undefined -> local var p is hoisted and shadows the global one
  var p = 2;
  console.log(p); // 2
}
sh();
```
Swap the inner `var` with `let` and the first log throws a `ReferenceError`, not `1`. The inner `p` still shadows the outer one; it is just in its TDZ.

### What happens when a function and a var have the same name?
The function declaration wins during setup. Then, when the code runs, any assignment to the var overwrites it.
```js
console.log(typeof x); // "function"
var x = 1;
function x() {}
console.log(typeof x); // "number"
```

### Are classes hoisted?
Yes, but like `let`, they are in the TDZ. `new Foo(); class Foo {}` throws `ReferenceError: Cannot access 'Foo' before initialization`.

## Common mistakes
- Saying `let`/`const` are not hoisted. They are; the TDZ is the proof (otherwise the inner `p` above would read the outer one).
- Thinking `typeof` is always safe. `typeof x` inside x's TDZ still throws.
- Default parameters have a TDZ too: `function f(a = b, b = 1) {}` throws when `a` uses the default.
- Function declarations inside blocks (`if (x) { function f() {} }`) behave differently in sloppy vs strict mode. Avoid them; use `const f = () => {}`.

## Resources
- [MDN: Hoisting](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting) - the four kinds of hoisting
- [MDN: let and the TDZ](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let#temporal_dead_zone_tdz) - exact TDZ rules
- [MDN: function declaration](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function) - hoisting of declarations
- [javascript.info: The old "var"](https://javascript.info/var) - var hoisting with examples
