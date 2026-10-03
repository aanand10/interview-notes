# ES6 classes

> **In one line:** Classes are cleaner syntax over JavaScript's existing prototype system: a class is still a function, and its methods live on the prototype, but you get `extends`, `super`, static members, getters/setters and true private `#fields`.

## Key points
- **Syntax sugar over prototypes**: `class Stock {}` creates a constructor function; methods go on `Stock.prototype`; instances link to it through the prototype chain. `typeof Stock` is `"function"`.
- Classes are **not exactly the same** as old constructor functions: you must call them with `new`, their body always runs in strict mode, methods are non-enumerable, and class declarations are in the TDZ (temporal dead zone) until defined.
- **`extends` and `super`**: a child class must call `super(...)` in its constructor before using `this`. `super.method()` calls the parent's version.
- **`static`** members belong to the class itself, not instances: `Order.fromJSON(...)`, counters, helpers.
- **Private fields** `#name` are truly private (a syntax error to access from outside). [Getters and setters](https://javascript.info/class#getters-setters) let you run code when a property is read or written.

## Example
```js
class Order {
  static count = 0;          // static field: shared, on the class
  #status = 'PENDING';       // private field: only code inside the class can see it

  constructor(symbol, qty, price) {
    this.symbol = symbol;
    this.qty = qty;
    this.price = price;
    Order.count++;
  }

  get total() {              // getter: read like a property
    return this.qty * this.price;
  }

  set quantity(value) {      // setter: validate on write
    if (value <= 0) throw new Error('Quantity must be positive');
    this.qty = value;
  }

  get status() { return this.#status; }
  fill() { this.#status = 'FILLED'; }

  static fromJSON(json) {    // static factory method
    const { symbol, qty, price } = JSON.parse(json);
    return new Order(symbol, qty, price);
  }
}

class LimitOrder extends Order {
  constructor(symbol, qty, price, limit) {
    super(symbol, qty, price); // must run before `this`
    this.limit = limit;
  }
  describe() {
    return `${this.symbol} x${this.qty} @ limit ${this.limit}`;
  }
}

const o = new LimitOrder('INFY', 10, 1500, 1490);
console.log(o.total);          // 15000
console.log(o.describe());     // INFY x10 @ limit 1490
o.fill();
console.log(o.status);         // FILLED
console.log(Order.count);      // 1
// console.log(o.#status);     // SyntaxError: private field
```

The same thing without classes (what classes do under the hood):

```js
function OrderFn(symbol, qty) {
  this.symbol = symbol;
  this.qty = qty;
}
OrderFn.prototype.describe = function () {
  return `${this.symbol} x${this.qty}`;
};
```

## When to use it
- Modelling things with state and behaviour: a `WebSocketClient` that reconnects, a `PriceFeed`, an `OrderBook`.
- Svelte 5 lets you put `$state` fields in classes, which is a neat way to build reactive stores:
  ```js
  // watchlist.svelte.js
  export class Watchlist {
    items = $state([]);
    count = $derived(this.items.length);
    add(symbol) { this.items.push(symbol); }
  }
  ```
- For small UI logic, plain functions and closures are often simpler.

## Likely questions
### Are classes just syntax sugar?
Mostly yes. A class creates a constructor function, and methods are placed on its prototype, so inheritance still works through the prototype chain. But there are real differences: classes must be called with `new`, the body is strict mode, methods are non-enumerable, and private `#fields` cannot be done with plain prototypes at all.

### What does `super` do?
In a constructor, `super(...)` calls the parent constructor and sets up `this`; you must call it before touching `this`, or you get a ReferenceError. In a method, `super.method()` calls the parent's version, which is useful when you override a method but want to extend it.

### What are static methods used for?
They belong to the class, not to instances, so you call `Order.fromJSON()` not `order.fromJSON()`. Common uses are factory methods, helpers and shared counters or caches. Static members are also inherited: `LimitOrder.fromJSON` works.

### How are `#private` fields different from `_private` or closures?
`_name` is only a naming convention; anyone can still read it. `#name` is enforced by the language: accessing it outside the class is a syntax error, and it does not show up in `Object.keys` or JSON. You can check if an object has it with `#name in obj`. Closures also give privacy, but each instance creates its own function copies, while class methods are shared on the prototype.

### Why use getters and setters?
A getter computes a value when read (`order.total`), so it is always up to date. A setter can validate or transform on write. They make the API look like plain properties.

### What does `this` refer to in a class method passed as a callback?
It loses its object, so `this` is `undefined` (classes are strict mode). Fix with an arrow function class field (`handleClick = () => {...}`) or `.bind(this)`.

## Common mistakes
- Using `this` before `super()` in a child constructor.
- Passing `obj.method` as a callback and losing `this`.
- Expecting class declarations to be usable before the line they are defined (TDZ).

## Resources
- [MDN: Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes) - full reference
- [javascript.info: Class basic syntax](https://javascript.info/class) - classes vs functions, getters and setters
- [javascript.info: Class inheritance](https://javascript.info/class-inheritance) - `extends` and `super` in depth
- [MDN: Private properties](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_properties) - `#fields` and `#x in obj`
