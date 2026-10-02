# Arrow functions

> **In one line:** Arrow functions are a short function syntax that do not have their own `this`, `arguments`, `super` or `prototype`, so they take `this` from the surrounding code and cannot be used as constructors.

## Key points
- **Lexical `this`:** an [arrow function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions) uses the `this` of the place it was written. `call`, `apply` and `bind` cannot change it.
- **No `arguments` object:** inside an arrow, `arguments` refers to the outer normal function's `arguments`. Use rest parameters `(...args)` instead.
- **Not a constructor:** `new arrowFn()` throws `TypeError: ... is not a constructor`. Arrows have no `prototype` property.
- **Short syntax:** `x => x * 2` has an implicit return. To return an object literal, wrap it in parentheses: `() => ({ id: 1 })`.
- Also no own `super` or `new.target`, and they cannot be generators (`yield`).

## Example
Output verified with node (ES module, strict mode).
```js
const Arrow = () => {};
new Arrow();                 // TypeError: Arrow is not a constructor
console.log(Arrow.prototype); // undefined

function regular() {
  const inner = () => arguments[0]; // reads regular's arguments
  return inner('ignored');
}
console.log(regular('outer-arg'));   // outer-arg

const count = (...args) => args.length; // rest params instead of arguments
console.log(count(1, 2, 3));         // 3

const make = () => ({ id: 1 });      // { id: 1 }
const wrong = () => { id: 1 };       // undefined: braces are a block, "id:" is a label
```

## When to use it
- **Callbacks** inside methods where you want the outer `this`: `setInterval(() => this.refreshPrices(), 5000)`.
- **Array helpers:** `holdings.filter(h => h.qty > 0).map(h => h.symbol)`.
- **Svelte event handlers:** `onclick={() => placeOrder(stock)}`.
- **Class fields that are passed around:** `handleTick = (tick) => { this.price = tick.price; }` so `this` stays correct when the socket calls it.

## Likely questions

### What does "lexical this" mean?
The arrow function does not get a `this` of its own when called. It looks up `this` like any other variable, in the scope where it was defined. So inside a method, an arrow sees the method's `this`; at the top of an ES module it sees `undefined`.
```js
const fn = () => this;
fn.call({ a: 1 }); // undefined in a module -> call cannot set an arrow's this
```

### Why don't arrow functions have arguments?
They were designed to be lightweight and to blend into the outer function, so `arguments` (like `this`) comes from the enclosing normal function. In modern code use rest params, which give a real array: `(...args) => args.reduce(...)`.

### Why can't you use new with an arrow function?
Arrow functions have no internal `[[Construct]]` method and no `prototype` property, so there is nothing for `new` to link the new object to. The engine throws a `TypeError`. Use a class or a normal function for constructors.

### When should you NOT use arrow functions?
1. **Object methods** that need the object as `this`:
```js
const counter = {
  count: 0,
  incArrow: () => this.count++, // this is NOT counter
  inc() { return ++this.count; }, // correct: method shorthand
};
```
2. **Prototype methods**, for the same reason:
```js
function Stock(sym) { this.sym = sym; }
Stock.prototype.label = () => this.sym;              // broken: this is outer scope
Stock.prototype.label2 = function () { return this.sym; }; // works: "TCS"
```
3. **Constructors** (cannot be `new`-ed).
4. **DOM handlers that rely on `this` being the element.** Use `event.currentTarget` or a normal function.
5. **When you need `arguments`** or a generator.

### Arrow class field vs normal method: what is the trade-off?
An arrow field (`onClick = () => {}`) is created for each instance and keeps `this` bound, so it is safe as a callback. A normal method lives once on the prototype (less memory, can be overridden and called with `super.method()`), but loses `this` when extracted. For a few UI objects the arrow field is fine; for thousands of objects (say, one per order book row) prefer prototype methods.

### Are arrow functions hoisted?
No more than the variable that holds them. `const f = () => {}` is in the TDZ before that line, so calling `f()` early throws a `ReferenceError`.

## Common mistakes
- Returning an object without parentheses: `() => { id: 1 }` returns `undefined`.
- Using arrows for methods in object literals or Vue/Svelte-style option objects that rely on `this`.
- Trying `arrow.bind(obj)` to fix `this`. It silently does nothing to `this`.
- A line break between params and `=>` is a syntax error.

## Resources
- [MDN: Arrow function expressions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions) - every limitation listed
- [javascript.info: Arrow functions revisited](https://javascript.info/arrow-functions) - this and arguments in depth
- [javascript.info: Arrow functions, the basics](https://javascript.info/arrow-functions-basics) - syntax refresher
- [MDN: arguments](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/arguments) - what arrows do not have
