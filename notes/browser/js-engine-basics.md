# JS engine basics

> **In one line:** A JS engine like V8 first runs your code quickly in an interpreter, watches which functions are "hot", and compiles those to fast machine code (JIT), using hidden classes and inline caches to make property access fast as long as your objects keep the same shape.

## Key points
- [**V8**](https://v8.dev/docs) is the engine in Chrome, Edge and Node.js. Firefox uses SpiderMonkey, Safari uses JavaScriptCore.
- **Pipeline:** source code is parsed into an AST (a tree), turned into bytecode, and run by the **Ignition** interpreter. Hot code moves up tiers: Sparkplug (fast baseline compiler), Maglev (mid-tier optimizer), **TurboFan** (top optimizer).
- **JIT** (just-in-time) compilation means compiling while the program runs, using real type information it has seen. If an assumption breaks (a number becomes a string), the engine **deoptimizes** and falls back to slower code.
- **Hidden classes** (V8 calls them "maps" or shapes): objects created with the same properties in the same order share one hidden class, so the engine knows each property's offset in memory.
- **Inline caching:** at each property access, the engine remembers the hidden class it saw and where the property was. Same shape next time = very fast lookup. One shape is monomorphic (fastest), a few is polymorphic, many is megamorphic (slow).

## Example
```js
// Same shape: both objects share one hidden class -> fast
function Quote(symbol, price) {
  this.symbol = symbol;
  this.price = price;
}
const a = new Quote('AAPL', 227.5);
const b = new Quote('TSLA', 251.1);

function value(q) {
  return q.price * 10;   // inline cache sees one shape -> monomorphic
}

// Shape changes: avoid these in hot code
const c = { price: 1, symbol: 'X' }; // different property ORDER -> different hidden class
b.volume = 1000;                     // adding a property later -> new hidden class
delete a.price;                      // delete often drops the object to slow "dictionary" mode
```

## When to use it
- In hot paths that run on every price tick (parsing, formatting, sorting thousands of rows), create objects with all fields up front and keep types stable.
- Mostly this is background knowledge. Measure with the Performance panel before micro-optimising.

## Likely questions
### What is V8 and how does it run code?
V8 is Google's JS engine. It parses code into an AST, compiles it to bytecode and runs it in the Ignition interpreter so startup is fast. While running, it collects type feedback. Functions that run often get compiled to optimized machine code by TurboFan (with Sparkplug and Maglev as quicker middle tiers).

### What is JIT compilation?
Compiling code to machine code while the program is running instead of before. The engine can make guesses based on what it has seen, like "this argument is always a number". That makes code very fast, but if a guess becomes wrong, it deoptimizes and goes back to slower code.

### What are hidden classes and inline caching?
JS objects have no fixed layout, so V8 assigns a hidden class describing the object's shape. Objects with the same properties added in the same order share it. Inline caching stores, at each property access site, the shape seen and the property's location, so the next access with the same shape skips the lookup. Keep shapes consistent: initialise all fields in the constructor, same order, and avoid `delete`.

### What causes deoptimization?
Changing types at a call site (passing a string where numbers were always passed), changing object shapes, or using many different shapes at one spot. The engine throws away the optimized code and returns to the interpreter or baseline code.

## Resources
- [V8 docs](https://v8.dev/docs) - official overview of the engine
- [V8 blog: Fast properties](https://v8.dev/blog/fast-properties) - hidden classes and property storage
- [MDN: JavaScript execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model) - how engines run JS at a high level
