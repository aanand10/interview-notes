# Common JS coding questions

> **In one line:** These are short warm-up problems. Interviewers check that you pick the right data structure (usually a `Map`, `Set` or two pointers), state the complexity, and mention edge cases before they ask.

## Key points
- Say the plan out loud first: input, output, one edge case, then the approach and its **Big-O** (how time/space grows with input size `n`).
- Hash maps (`Map` or a plain object) turn "search again" (O(n^2)) into "look up" (O(1)). See [MDN: Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map).
- Iterate strings with `for...of` or `[...str]` so emoji and other multi-unit characters stay whole.
- Know the built-in too (`flat`, `Object.groupBy`, `Set`), but be ready to write it by hand.
- Common edge cases: empty input, one item, duplicates, mixed case, `null`/`undefined`.

All code below was run with Node 22 and the outputs are real.

## Problems

### Q1. Reverse a string
```js
const reverse = (str) => [...str].reverse().join("");
console.log(reverse("hello"));
```
**Answer:**
```text
olleh
```
- Time O(n), space O(n). Strings are immutable, so you need a new array/string.
- Edge case: `"hi😀".split("")` splits the emoji into two broken halves (output `"\ude00\ud83dih"`), while `[..."hi😀"]` keeps it (output `😀ih`). Spread iterates by code point.
- Follow-up: a two-pointer swap on an array is the in-place version.

### Q2. Palindrome (ignore case and punctuation)
```js
function isPalindrome(str) {
  const s = str.toLowerCase().replace(/[^a-z0-9]/g, "");
  let i = 0, j = s.length - 1;
  while (i < j) {
    if (s[i] !== s[j]) return false;
    i++;
    j--;
  }
  return true;
}
console.log(isPalindrome("A man, a plan, a canal: Panama"), isPalindrome("race a car"), isPalindrome(""));
```
**Answer:**
```text
true false true
```
- **Two pointers** from both ends. Time O(n), space O(n) for the cleaned string (O(1) if you skip non-letters while walking instead).
- Edge cases: empty string is a palindrome; clarify whether case and spaces count. See [LeetCode 125](https://leetcode.com/problems/valid-palindrome/).

### Q3. Anagram check
```js
function isAnagram(a, b) {
  if (a.length !== b.length) return false;
  const count = new Map();
  for (const ch of a) count.set(ch, (count.get(ch) ?? 0) + 1);
  for (const ch of b) {
    const c = count.get(ch);
    if (!c) return false; // missing or already used up
    count.set(ch, c - 1);
  }
  return true;
}
console.log(isAnagram("listen", "silent"), isAnagram("rat", "car"), isAnagram("aab", "abb"));
```
**Answer:**
```text
true false false
```
- Count letters in one string, subtract with the other. Time O(n), space O(k) where k is the number of distinct characters.
- Brute force: sort both and compare, `[...a].sort().join("") === [...b].sort().join("")`, which is O(n log n).
- Edge case: `"aab"` vs `"abb"` has the same letters but different counts, which is why counting beats a `Set`. See [LeetCode 242](https://leetcode.com/problems/valid-anagram/).

### Q4. Remove duplicates
```js
const unique = (arr) => [...new Set(arr)];
console.log(unique([1, 2, 2, "2", NaN, NaN, 3]));

// By a key, keep the first one (e.g. dedupe watchlist items by id)
function uniqueBy(arr, keyFn) {
  const seen = new Set();
  return arr.filter((item) => {
    const k = keyFn(item);
    if (seen.has(k)) return false;
    seen.add(k);
    return true;
  });
}
console.log(uniqueBy([{ id: 1, s: "TCS" }, { id: 2, s: "INFY" }, { id: 1, s: "TCS dup" }], (x) => x.id));
```
**Answer:**
```text
[ 1, 2, '2', NaN, 3 ]
[ { id: 1, s: 'TCS' }, { id: 2, s: 'INFY' } ]
```
- `Set` uses **SameValueZero** equality: `NaN` equals `NaN`, but `2` and `"2"` are different. Order is kept. Time O(n).
- `Set` compares objects by reference, so two equal-looking objects are both kept. That is why `uniqueBy` exists.
- Avoid `arr.filter((x, i) => arr.indexOf(x) === i)`: it is O(n^2) and drops `NaN` entirely.

### Q5. Group by
```js
function groupBy(arr, keyFn) {
  const out = {};
  for (const item of arr) {
    const k = keyFn(item);
    (out[k] ??= []).push(item);
  }
  return out;
}
const trades = [{ sym: "TCS", qty: 5 }, { sym: "INFY", qty: 2 }, { sym: "TCS", qty: 1 }];
console.log(groupBy(trades, (t) => t.sym));
```
**Answer:**
```text
{
  TCS: [ { sym: 'TCS', qty: 5 }, { sym: 'TCS', qty: 1 } ],
  INFY: [ { sym: 'INFY', qty: 2 } ]
}
```
- Time O(n), space O(n). `??=` assigns only if the key is missing.
- Built-in: `Object.groupBy(trades, t => t.sym)` gives the same groups, but the result has a **null prototype** (Node prints `[Object: null prototype]`). See [MDN: Object.groupBy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/groupBy). Use `Map.groupBy` when keys are not strings.

### Q6. Chunk an array
```js
function chunk(arr, size) {
  if (size < 1) throw new RangeError("size must be >= 1");
  const out = [];
  for (let i = 0; i < arr.length; i += size) out.push(arr.slice(i, i + size));
  return out;
}
console.log(chunk([1, 2, 3, 4, 5], 2), chunk([], 3));
```
**Answer:**
```text
[ [ 1, 2 ], [ 3, 4 ], [ 5 ] ] []
```
- Time O(n). The last chunk can be shorter. Guard `size < 1`, or the loop never ends.
- Real use: batching symbol subscriptions, for example 50 symbols per WebSocket message.

### Q7. Flatten a nested array
```js
function flatten(arr, depth = Infinity) {
  const out = [];
  for (const item of arr) {
    if (Array.isArray(item) && depth > 0) out.push(...flatten(item, depth - 1));
    else out.push(item);
  }
  return out;
}
console.log(flatten([1, [2, [3, [4]]], 5]), flatten([1, [2, [3, [4]]]], 1), [1, [2, [3, [4]]]].flat(Infinity));
```
**Answer:**
```text
[ 1, 2, 3, 4, 5 ] [ 1, 2, [ 3, [ 4 ] ] ] [ 1, 2, 3, 4 ]
```
- Recursion with a depth counter, matching `Array.prototype.flat(depth)`. Time O(total items).
- Edge case: very deep nesting can overflow the call stack. Follow-up: an iterative version with an explicit stack. Note `push(...big)` can also hit argument limits on huge arrays.

### Q8. Count character frequency
```js
function charFrequency(str) {
  const freq = {};
  for (const ch of str) freq[ch] = (freq[ch] ?? 0) + 1;
  return freq;
}
console.log(charFrequency("banana"));
```
**Answer:**
```text
{ b: 1, a: 3, n: 2 }
```
- Time O(n), space O(k). Use a `Map` if keys may clash with object keys like `__proto__`, or if you need non-string keys.

### Q9. First non-repeating character
```js
function firstNonRepeating(str) {
  const freq = new Map();
  for (const ch of str) freq.set(ch, (freq.get(ch) ?? 0) + 1);
  for (const ch of str) if (freq.get(ch) === 1) return ch;
  return null;
}
console.log(firstNonRepeating("swiss"), firstNonRepeating("aabb"), firstNonRepeating("leetcode"));
```
**Answer:**
```text
w null l
```
- Two passes: count, then find the first with count 1. Time O(n), space O(k). Brute force with nested loops is O(n^2).
- Edge case: return `null` (or `-1` for the index version) when none exists. See [LeetCode 387](https://leetcode.com/problems/first-unique-character-in-a-string/).

### Q10. Deep get by path ("a.b.c")
```js
function get(obj, path, defaultValue) {
  const keys = Array.isArray(path)
    ? path
    : path.replace(/\[(\w+)\]/g, ".$1").split(".").filter(Boolean); // "a[0].b" -> ["a","0","b"]
  let cur = obj;
  for (const key of keys) {
    if (cur == null) return defaultValue; // null or undefined: stop safely
    cur = cur[key];
  }
  return cur === undefined ? defaultValue : cur;
}
const state = { user: { portfolio: { holdings: [{ sym: "TCS", qty: 0 }] } } };
console.log(
  get(state, "user.portfolio.holdings[0].sym"),
  get(state, "user.portfolio.holdings.0.qty"),
  get(state, "user.address.city", "N/A"),
  get(null, "a.b", "none"),
);
```
**Answer:**
```text
TCS 0 N/A none
```
- Time O(depth). Like lodash `_.get`.
- Edge cases: falsy but valid values like `0` must be returned, not replaced by the default (check `=== undefined`, not `||`). Support bracket indexes. A null in the middle returns the default instead of throwing.
- Modern one-off alternative: optional chaining `state.user?.address?.city ?? "N/A"`. See [MDN: Optional chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining).

### Q11. sleep(ms)
```js
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

(async () => {
  const t = Date.now();
  await sleep(100);
  console.log("slept ~100ms:", Date.now() - t >= 99);
})();
```
**Answer:**
```text
slept ~100ms: true
```
- Wraps `setTimeout` in a promise so you can `await` it. It does not block the thread; other code keeps running.
- The delay is a minimum, not exact. Follow-up: make it cancellable with an `AbortSignal`, or use it for retry with backoff.

### Q12. Random id
```js
const id1 = crypto.randomUUID(); // best: standard, collision-safe
console.log(id1.length, /^[0-9a-f-]{36}$/.test(id1));

const randomId = (len = 8) => Math.random().toString(36).slice(2, 2 + len); // quick, not secure
console.log(typeof randomId(), randomId().length <= 8);

let counter = 0;
const nextId = (prefix = "id") => `${prefix}-${++counter}`; // unique per page session
console.log(nextId(), nextId("order"));
```
**Answer:**
```text
36 true
string true
id-1 order-2
```
- `crypto.randomUUID()` gives a v4 UUID using a secure random source. It needs a **secure context** (HTTPS or localhost) in browsers. See [MDN: randomUUID](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID).
- `Math.random` ids are fine for UI keys but not for anything security-related (tokens, idempotency keys). A counter is simplest for element ids like `aria-describedby` links (in Svelte 5 there is also `$props.id()`).

## When to use these
- Group by and uniqueBy: building a portfolio view from raw trades.
- Chunk: batching API calls or subscriptions.
- Deep get: reading optional fields from nested API responses.
- sleep: polling, retries, and tests.

## Common mistakes
- Forgetting complexity, or saying O(n) for code that hides an `indexOf` inside a loop (that is O(n^2)).
- `split("")` breaking emoji.
- Using `||` for defaults and losing valid `0` or `""` values.

## Resources
- [MDN: Set](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set) - equality rules used for dedupe
- [MDN: Array.prototype.flat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/flat) - the built-in to compare your flatten with
- [MDN: Object.groupBy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/groupBy) - built-in grouping and its null prototype
- [javascript.info: Map and Set](https://javascript.info/map-set) - when to pick Map over an object
