# Scope and scope chain

> **In one line:** Scope is where a variable can be seen; JavaScript uses lexical scope, so a function looks up variables in the place it was *written*, walking outward through parent scopes (the scope chain) until it finds the name or reaches global.

## Key points
- **Kinds of [scope](https://developer.mozilla.org/en-US/docs/Glossary/Scope):** global, module (each ES module file), function, and block (`{ }` with `let`/`const`/`class`).
- **Lexical scope** means scope is decided by where code sits in the source, not by who calls the function. "Lexical" just means "based on the written text".
- **Scope chain:** when a name is not found in the current scope, the engine checks the parent, then the grandparent, up to global. If it is still missing, you get `ReferenceError: x is not defined`.
- **Block vs function scope:** `var` ignores blocks and belongs to the function. `let`/`const` belong to the nearest [block](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/block).
- **Shadowing:** an inner variable with the same name hides the outer one inside that inner scope. The outer one is untouched.

## Example
```js
const fee = 20;                 // global (or module) scope

function placeOrder(qty) {      // function scope: qty, total
  const total = qty * 100;
  if (total > 1000) {           // block scope
    const fee = 0;              // shadows the outer fee inside this block only
    console.log('fee', fee);    // fee 0
  }
  return total + fee;           // uses the outer fee = 20 (lookup walks up the chain)
}
console.log(placeOrder(20));    // logs "fee 0", then 2020
```

## When to use it
- Keep variables in the smallest scope that works. In a Svelte component, a value used only inside one handler should be declared inside that handler, not at the top of the `<script>`.
- Modules give you file-level privacy: a `const` in `priceFeed.js` that is not exported cannot be touched by other files.
- Understanding the chain explains closures, which power debounced search, stores and event handlers.

## Likely questions

### What is lexical scope?
The scope of a variable is fixed by where it is written in the code. A function can see variables from the function or block it was defined inside, no matter where it is called from later.
```js
let name = 'A';
function show() { console.log(name); }
function run() { let name = 'B'; show(); }
run(); // A -> show was written next to the outer name, not inside run
```
If JavaScript used dynamic scope, this would print `B`. It does not.

### Block scope vs function scope: show the difference
```js
function f1() {
  if (true) { var v = 1; let l = 2; }
  console.log(v); // 1 -> var belongs to the whole function
  console.log(l); // ReferenceError: l is not defined -> let stayed in the if-block
}
```

### What is shadowing? Is it a problem?
Shadowing is declaring an inner variable with the same name as an outer one. Inside the inner scope, the inner one wins.
```js
let count = 10;
{ let count = 20; console.log(count); } // 20
console.log(count);                     // 10
```
It is legal and sometimes useful, but it hurts readability, so linters can warn about it (`no-shadow`). One illegal case: shadowing a `let` with a `var` in an inner block of the same function, for example `let a; { var a; }`, is a `SyntaxError`, because `var` would hoist into the same scope as `let a`.

### Output question: nested functions
```js
var x = 'global';
function outer() {
  var x = 'outer';
  function inner() { console.log(x); }
  return inner;
}
outer()(); // outer

function a() {
  var y = 1;
  function b() {
    var z = 2;
    function c() { console.log(y + z); } // looks up z in b, y in a
    c();
  }
  b();
}
a(); // 3
```
`inner` is called from the global scope, but it still prints `outer`, because lookup follows where it was defined.

### What happens if you assign to a variable you never declared?
In sloppy (non-strict) mode, `leaked = 5` inside a function creates a global variable by accident. In [strict mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode), and in all ES modules and class bodies, it throws `ReferenceError`. This is one reason modern code (Svelte, Vite) is always strict.

### How is the scope chain related to closures?
A closure is just a function that keeps a reference to its scope chain after the outer function returns. The same lookup rules apply; the outer variables simply stay alive.

## Common mistakes
- Thinking scope depends on the call site. That is how `this` works, not scope.
- Expecting `var` inside an `if` or `for` to stay in that block.
- Accidentally reading an outer variable because of a typo in the inner name (shadowing in reverse).
- Forgetting that each ES module has its own top-level scope, so top-level `const` is not global.

## Resources
- [MDN: Scope](https://developer.mozilla.org/en-US/docs/Glossary/Scope) - the kinds of scope in one page
- [javascript.info: Variable scope, closure](https://javascript.info/closure) - lexical environment explained step by step
- [MDN: block statement](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/block) - block scoping of let/const vs var
- [MDN: Strict mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode) - why undeclared assignment throws
