# Output questions: scope, hoisting, closures

> **In one line:** JavaScript sets up every declaration before running a scope, but only `var` and function declarations are usable early. `let`/`const` sit in a "dead zone" until their line runs, and closures remember variables, not values.

## Key points
- **Hoisting** means declarations are registered before the code in a scope runs. `var` starts as `undefined`. A function declaration starts as the full function. See [MDN: Hoisting](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting).
- `let`, `const` and `class` are hoisted too, but reading them before their line throws a `ReferenceError`. That gap is the **TDZ** (temporal dead zone). See [MDN: let](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let#temporal_dead_zone_tdz).
- `var` is function-scoped. `let`/`const` are block-scoped (`{ }`, loop bodies, `if` blocks).
- A **closure** is a function plus the variables it can see where it was created. It holds a live link to those variables, not a copy. See [javascript.info: Closures](https://javascript.info/closure).
- `for (let i ...)` creates a new `i` for every loop turn. `for (var i ...)` shares one `i` across all turns.

How to solve these fast: (1) find each scope, (2) hoist the declarations to the top of it, (3) run line by line. All outputs below were run with Node.

## Drills

### Q1. var before assignment
```js
console.log(a);
var a = 1;
console.log(a);
```
**Answer:**
```text
undefined
1
```
`var a` is hoisted and set to `undefined`. The value `1` is assigned only when line 2 runs.

### Q2. let before declaration
```js
console.log(b);
let b = 2;
```
**Answer:**
```text
ReferenceError: Cannot access 'b' before initialization
```
`b` exists (it is hoisted) but is in the TDZ until `let b` runs. That error message is different from "b is not defined", which means the name does not exist at all.

### Q3. Function declaration called early
```js
console.log(typeof greet);
greet();
function greet() {
  console.log("hi");
}
```
**Answer:**
```text
function
hi
```
Function declarations are hoisted with their body, so they work before their line.

### Q4. Function expression called early
```js
console.log(typeof bar);
bar();
var bar = function () {
  console.log("bar");
};
```
**Answer:**
```text
undefined
TypeError: bar is not a function
```
Only `var bar` is hoisted (as `undefined`). The function is assigned later, so calling `undefined()` gives a TypeError. With `const bar` it would be a ReferenceError (TDZ).

### Q5. Shadowing with var inside a function
```js
var x = 1;
function f() {
  console.log(x);
  var x = 2;
}
f();
```
**Answer:**
```text
undefined
```
The inner `var x` is hoisted to the top of `f` and **shadows** (hides) the outer `x`. At the log line it is still `undefined`.

### Q6. Shadowing with let inside a block
```js
let price = 100;
{
  console.log(price);
  let price = 200;
}
```
**Answer:**
```text
ReferenceError: Cannot access 'price' before initialization
```
The block has its own `price`, which shadows the outer one from the start of the block. Reading it before `let price` is a TDZ error. It does not fall back to the outer `100`.

### Q7. Closures in a loop with var
```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```
**Answer:**
```text
3
3
3
```
There is one shared `i`. The timers run after the loop ends, when `i` is already `3`.

### Q8. Closures in a loop with let
```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```
**Answer:**
```text
0
1
2
```
`let` makes a fresh `i` for each turn, so each callback closes over its own copy.

### Q9. IIFE fix (the pre-ES6 way)
```js
for (var i = 0; i < 3; i++) {
  (function (j) {
    setTimeout(() => console.log(j), 0);
  })(i);
}
```
**Answer:**
```text
0
1
2
```
The **IIFE** (Immediately Invoked Function Expression) runs right away and gets `i` as a new parameter `j` each turn. Each callback closes over its own `j`.

### Q10. Two counters
```js
function makeCounter() {
  let count = 0;
  return () => ++count;
}
const c1 = makeCounter();
const c2 = makeCounter();
console.log(c1(), c1(), c2());
```
**Answer:**
```text
1 2 1
```
Each call to `makeCounter` creates a new `count`. `c1` and `c2` have separate private state.

### Q11. var and function with the same name
```js
var foo = 1;
function foo() {}
console.log(typeof foo);
```
**Answer:**
```text
number
```
The function declaration is hoisted first, then the assignment `foo = 1` runs and overwrites it.

### Q12. Mutating a const object
```js
const order = { qty: 1 };
order.qty = 5;
console.log(order.qty);
order = { qty: 10 };
```
**Answer:**
```text
5
TypeError: Assignment to constant variable.
```
`const` locks the **binding** (the variable), not the object. You can change properties, but you cannot point `order` at a new object.

### Q13. typeof and the TDZ
```js
console.log(typeof notDeclared);
console.log(typeof later);
let later = 1;
```
**Answer:**
```text
undefined
ReferenceError: Cannot access 'later' before initialization
```
`typeof` is safe for names that do not exist at all. It is not safe for a `let` in its TDZ.

### Q14. Accidental global
```js
function setTotal() {
  total = 50; // no var/let/const
}
setTotal();
console.log(total);
```
**Answer:**
```text
50
```
In sloppy (non-strict) mode, assigning to an undeclared name creates a global. In strict mode, and in ES modules (which is what Svelte/Vite code is), this throws `ReferenceError: total is not defined`.

### Q15. IIFE module pattern
```js
const result = (function () {
  var secret = "s3cr3t";
  return { reveal: () => secret.length };
})();
console.log(result.reveal());
console.log(typeof secret);
```
**Answer:**
```text
6
undefined
```
`secret` lives only inside the IIFE. The returned arrow keeps it alive through a closure, but outside code cannot see it.

### Q16. Closure over a shared outer variable
```js
let count = 0;
const fns = [];
for (let i = 0; i < 3; i++) {
  count++;
  fns.push(() => count);
}
console.log(fns.map((fn) => fn()));
```
**Answer:**
```text
[ 3, 3, 3 ]
```
`let i` is per-turn, but `count` is one variable outside the loop. All three closures read its latest value. Closures capture variables, not snapshots.

## When it matters in real code
- A Svelte list of stock rows that attaches handlers in a loop: with `let` (or `{#each}`) each handler sees its own row.
- Polling or retry code with `setTimeout` inside loops.
- Factory functions (like `makeCounter`) for private state, such as a per-widget request counter.

## Common mistakes
- Saying `let` is "not hoisted". It is hoisted. It is just not initialised (TDZ).
- Thinking `const` makes an object immutable. Use `Object.freeze` for that (shallow only).
- Mixing up the errors: "is not defined" (no such name) vs "Cannot access before initialization" (TDZ) vs "is not a function" (`var` holding `undefined`).

## Resources
- [MDN: Hoisting](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting) - short and exact definition
- [javascript.info: Variable scope, closure](https://javascript.info/closure) - clear walk-through of lexical environments
- [javascript.info: The old "var"](https://javascript.info/var) - var vs let, and IIFEs
- [MDN: Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures) - loop closure pitfall and fixes
