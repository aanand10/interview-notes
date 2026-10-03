# Output questions: this and binding

> **In one line:** For a normal function, `this` is decided by **how the function is called**, not where it is written. Arrow functions have no `this` of their own and use the `this` of the code around them.

## Key points
- Four rules for normal functions, strongest first: **`new`** (fresh object) > **explicit** (`call`/`apply`/`bind`) > **implicit** (`obj.method()` gives `obj`) > **default** (plain `fn()` gives `undefined` in strict mode, `globalThis` in sloppy mode). See [MDN: this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this).
- **Arrow functions** take `this` from the surrounding scope when they are created. `call`, `apply` and `bind` cannot change it. See [MDN: Arrow functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions).
- Pulling a method off its object (`const f = obj.m`, passing `obj.m` as a callback) loses the object. That is the most common `this` bug.
- `bind` returns a new function with `this` locked forever. A second `bind` does nothing. Only `new` overrides it.
- ES modules and class bodies are always **strict mode**, so lost `this` is `undefined`, not `window`. Svelte/Vite code is modules.

How to solve: look at the **call site**. Is there a `new`? A `call/apply/bind`? A `something.` before the call? If none, it is the default rule. If the function is an arrow, ignore the call site and look outward.

All snippets below are run as ES modules (`.mjs`, strict mode) with Node unless stated. I use `this?.label` in some places so a lost `this` prints `undefined` instead of crashing.

## Drills

### Q1. Method call vs extracted function
```js
const user = {
  label: "Asha",
  show() {
    console.log(this?.label);
  },
};
user.show();
const show = user.show;
show();
```
**Answer:**
```text
Asha
undefined
```
`user.show()` has `user.` before the call, so `this = user`. `show()` is a plain call, so `this = undefined` in strict mode.

### Q2. Same thing without optional chaining
```js
const user = {
  label: "Asha",
  show() {
    console.log(this.label);
  },
};
const show = user.show;
show();
```
**Answer:**
```text
TypeError: Cannot read properties of undefined (reading 'label')
```
`this` is `undefined`, so reading `.label` on it throws. In a sloppy-mode browser script it would read `window.label` instead.

### Q3. Normal function nested in a method
```js
const stock = {
  label: "TCS",
  outer() {
    function inner() {
      console.log(this?.label);
    }
    inner();
  },
};
stock.outer();
```
**Answer:**
```text
undefined
```
`inner()` is a plain call. It does not inherit `this` from `outer`. Every normal function gets its own `this`.

### Q4. Arrow nested in a method
```js
const stock = {
  label: "TCS",
  outer() {
    const inner = () => console.log(this.label);
    inner();
  },
};
stock.outer();
```
**Answer:**
```text
TCS
```
The arrow has no `this`, so it uses `outer`'s `this`, which is `stock`.

### Q5. Arrow as an object-literal method
```js
const stock = {
  label: "INFY",
  show: () => console.log(this?.label),
};
stock.show();
```
**Answer:**
```text
undefined
```
An object literal `{ }` is not a scope. The arrow takes `this` from the module top level, which is `undefined`. Never use arrows as object methods that need `this`.

### Q6. Callbacks: normal function vs arrow
```js
const list = {
  label: "Watchlist",
  items: ["A", "B"],
  printRegular() {
    this.items.forEach(function (item) {
      console.log(this?.label, item);
    });
  },
  printArrow() {
    this.items.forEach((item) => console.log(this.label, item));
  },
};
list.printRegular();
list.printArrow();
```
**Answer:**
```text
undefined A
undefined B
Watchlist A
Watchlist B
```
`forEach` calls the normal callback as a plain function, so `this` is lost. The arrow keeps the method's `this`. (`forEach` also takes a second `thisArg` argument as an old-style fix.)

### Q7. setTimeout with a method
```js
const user = {
  label: "Asha",
  show() {
    console.log(this?.label);
  },
};
setTimeout(user.show, 0);
setTimeout(() => user.show(), 0);
```
**Answer:**
```text
undefined
Asha
```
Passing `user.show` passes only the function. The timer calls it without `user` (in browsers `this` is `window`, in Node it is a Timeout object; neither has `label`). The arrow wrapper calls `user.show()` properly. `setTimeout(user.show.bind(user), 0)` also works.

### Q8. bind is permanent
```js
function show() {
  console.log(this.label);
}
const bound = show.bind({ label: "B" });
bound();
bound.call({ label: "C" });
bound.bind({ label: "D" })();
```
**Answer:**
```text
B
B
B
```
A bound function ignores later `call`, `apply` and `bind`. The first `bind` wins.

### Q9. call vs apply
```js
function greet(greeting, punct) {
  console.log(`${greeting}, ${this.label}${punct}`);
}
greet.call({ label: "Ann" }, "Hi", "!");
greet.apply({ label: "Bo" }, ["Hey", "?"]);
```
**Answer:**
```text
Hi, Ann!
Hey, Bo?
```
Both call the function now with a given `this`. `call` takes arguments one by one, `apply` takes them as an array (memory trick: **A**pply = **A**rray).

### Q10. Class method vs class-field arrow
```js
class Ticker {
  symbol = "TCS";
  print() {
    console.log(this?.symbol);
  }
  printArrow = () => {
    console.log(this.symbol);
  };
}
const t = new Ticker();
const { print, printArrow } = t;
print();
printArrow();
```
**Answer:**
```text
undefined
TCS
```
`print` lives on the prototype and loses `this` when destructured (class bodies are strict). `printArrow` is a class field created per instance in the constructor, so its `this` is the instance forever. Cost: one function per instance.

### Q11. new beats bind
```js
function Stock(sym) {
  this.sym = sym;
}
const BoundStock = Stock.bind({ sym: "IGNORED" });
const s = new BoundStock("HDFC");
console.log(s.sym);
console.log(s instanceof Stock);
```
**Answer:**
```text
HDFC
true
```
`new` creates a fresh object and uses it as `this`, ignoring the bound object. It is the highest-priority rule.

### Q12. Constructor that returns something
```js
function A() {
  this.v = 1;
  return { v: 2 };
}
function B() {
  this.v = 1;
  return 42;
}
console.log(new A().v, new B().v);
```
**Answer:**
```text
2 1
```
If a constructor returns an **object**, `new` gives you that object. If it returns a primitive, the return value is ignored and you get `this`.

### Q13. Arrow with call and bind
```js
const arrow = () => this?.label;
console.log(arrow.call({ label: "Z" }));
console.log(arrow.bind({ label: "Z" })());
```
**Answer:**
```text
undefined
undefined
```
Arrows ignore the `this` you pass. Module top-level `this` is `undefined`.

### Q14. Parentheses and the comma operator
```js
const obj = {
  label: "O",
  get() {
    return this?.label;
  },
};
console.log(obj.get());
console.log((obj.get)());
console.log((0, obj.get)());
```
**Answer:**
```text
O
O
undefined
```
Plain grouping `(obj.get)` keeps the reference to `obj`, so `this` is kept. The comma operator evaluates to just the function value, so the call becomes a plain call.

### Q15. Sloppy mode default this (CommonJS file)
```js
// CommonJS file, sloppy mode (not a module)
function whoAmI() {
  return this === globalThis;
}
console.log(whoAmI());
```
**Answer:**
```text
true
```
Without strict mode, the default rule gives `globalThis` (`window` in a browser). Same function in a module would return `false` because `this` is `undefined`.

## When it matters in real code
- Passing class methods as event handlers or to `setTimeout`/`setInterval` (for example a `PriceFeed` class with `onMessage` passed to a WebSocket). Use an arrow wrapper, a class-field arrow, or `bind`.
- In Svelte 5 you mostly write plain functions and `$state`, so `this` rarely appears. It shows up when you wrap classes, such as a WebSocket client or a chart library instance.

## Common mistakes
- Thinking `this` is where the function is defined. For normal functions it is the call site.
- Using an arrow as an object method and expecting `this` to be the object.
- Thinking `bind` can be re-bound, or that `call` works on arrows.

## Resources
- [MDN: this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this) - full rules, including classes and callbacks
- [javascript.info: Object methods, "this"](https://javascript.info/object-methods) - beginner-friendly explanation
- [javascript.info: Function binding](https://javascript.info/bind) - losing `this` and fixing it with bind
- [MDN: Function.prototype.bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind) - bound functions and `new`
