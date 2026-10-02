# Vitest

> **In one line:** Vitest is a fast, Vite-native test runner with a Jest-compatible API, so I write tests with `describe`, `it`, `expect`, and use `vi` to fake functions, modules and timers.

## Key points
- **Vite-native:** it reuses your `vite.config` (aliases, Svelte plugin, TypeScript), so there is no separate Babel or Jest transform setup. It is the default in a SvelteKit project made with `npx sv create`.
- **Jest-like API:** `describe`, `it`/`test`, `expect`, `beforeEach`, `afterEach`. Mocks live on [`vi`](https://vitest.dev/api/vi) instead of `jest`.
- **Mocking tools:** `vi.fn()` makes a fake function that records calls, `vi.spyOn()` wraps a real method, `vi.mock()` replaces a whole module, `vi.stubGlobal()` replaces globals like `fetch`.
- **Fake timers:** `vi.useFakeTimers()` lets you jump time forward with `vi.advanceTimersByTime(ms)` instead of really waiting. Great for debounce, polling and retries.
- **Environments:** `node` by default, `jsdom` or `happy-dom` for DOM tests, or [Browser Mode](https://vitest.dev/guide/browser/) to run in a real browser through Playwright.

## Example
All code below was run with Vitest and passes.

```js
// src/debounce.test.js - fake timers
import { beforeEach, afterEach, it, expect, vi } from 'vitest';
import { debounce } from './debounce.js';

beforeEach(() => vi.useFakeTimers());
afterEach(() => vi.useRealTimers()); // always restore, or other tests break

it('calls the search only once after typing stops', () => {
  const search = vi.fn();
  const debounced = debounce(search, 300);

  debounced('R'); debounced('RE'); debounced('REL');
  expect(search).not.toHaveBeenCalled();

  vi.advanceTimersByTime(299);
  expect(search).not.toHaveBeenCalled(); // not yet

  vi.advanceTimersByTime(1);
  expect(search).toHaveBeenCalledTimes(1);
  expect(search).toHaveBeenCalledWith('REL'); // only the last value
});
```

```js
// src/portfolio.test.js - mocking a module + async code
import { describe, it, expect, vi } from 'vitest';
import { holdingValue } from './portfolio.js';
import { getQuote } from './api.js';

// vi.mock is hoisted to the top of the file, so portfolio.js gets the fake too
vi.mock('./api.js', () => ({ getQuote: vi.fn() }));

describe('holdingValue', () => {
  it('multiplies live price by quantity', async () => {
    vi.mocked(getQuote).mockResolvedValue({ price: 2500 });

    await expect(holdingValue('TCS', 4)).resolves.toBe(10000);
    expect(getQuote).toHaveBeenCalledWith('TCS');
  });

  it('passes API errors up', async () => {
    vi.mocked(getQuote).mockRejectedValue(new Error('HTTP 503'));
    await expect(holdingValue('TCS', 4)).rejects.toThrow('HTTP 503');
  });
});
```

```js
// src/api.test.js - stubbing fetch
import { afterEach, it, expect, vi } from 'vitest';
import { getQuote } from './api.js';

afterEach(() => vi.unstubAllGlobals());

it('throws on 500', async () => {
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue(new Response('', { status: 500 })));
  await expect(getQuote('INFY')).rejects.toThrow('HTTP 500');
});
```

## When to use it
- Unit tests for pure logic: money conversion (`toPaise`), P&L, order validation, formatters.
- Async code: API clients, retry with backoff, polling a quote every 5 seconds.
- Time-based code: debounced search in a watchlist, session timeout, market open/close banners (`vi.setSystemTime`).
- Component tests together with [Testing Library](https://testing-library.com/docs/svelte-testing-library/intro).

## Likely questions
### How do you write a basic unit test in Vitest?
I import `describe`, `it` and `expect` from `vitest`, call my function with an input, and assert the output with a matcher like `toBe` (same value) or `toEqual` (deep equal for objects). I name tests by behaviour, like "rejects negative quantity", and use `it.each` to run one test over many inputs, like a table of bad amounts.

### What is `vi.fn()` and when do you use it?
`vi.fn()` creates a fake function that remembers every call. I pass it in as a callback or dependency, then assert with `toHaveBeenCalledWith` or `toHaveBeenCalledTimes`. I can control what it returns with `mockReturnValue`, `mockResolvedValue` for promises, or `mockRejectedValue` to simulate a failure. `mockResolvedValueOnce` lets me script a sequence, like "fail, fail, then succeed".

### What is the difference between `vi.fn`, `vi.spyOn` and `vi.mock`?
`vi.fn` makes a brand new fake function. `vi.spyOn(obj, 'method')` wraps a real method on an existing object, so I can watch it or override it and later put it back with `mockRestore`. `vi.mock('./api.js')` replaces a whole module for every file that imports it in that test file. `vi.mock` is hoisted above imports, so if the factory needs a variable I create it with `vi.hoisted`.

### How do you test code that uses `setTimeout` or `setInterval`?
I call `vi.useFakeTimers()` in `beforeEach`, then move time forward with `vi.advanceTimersByTime(ms)` or `vi.runAllTimers()`. If the timer callback awaits promises, I use the async versions like `vi.advanceTimersByTimeAsync`. I always call `vi.useRealTimers()` in `afterEach`. For "is the market open" logic I use `vi.setSystemTime(new Date(...))` to freeze the clock.

```js
it('retries with backoff, then succeeds', async () => {
  vi.useFakeTimers();
  const fn = vi.fn()
    .mockRejectedValueOnce(new Error('timeout'))
    .mockRejectedValueOnce(new Error('timeout'))
    .mockResolvedValue('ok');

  const promise = withRetry(fn, { retries: 2, delayMs: 1000 });
  await vi.advanceTimersByTimeAsync(1000); // 1st backoff
  await vi.advanceTimersByTimeAsync(2000); // 2nd backoff
  await expect(promise).resolves.toBe('ok');
  expect(fn).toHaveBeenCalledTimes(3);
  vi.useRealTimers();
});
```

### How do you test async code?
Make the test `async` and `await` the result, or use `await expect(promise).resolves.toBe(x)` and `.rejects.toThrow(...)`. The key is to always `await` or `return` the expectation, otherwise the test finishes before the promise settles and passes by accident. For UI that updates later, Testing Library's `findBy` queries wait for the element.

### What is the difference between `mockClear`, `mockReset` and `mockRestore`?
`mockClear` wipes the recorded calls but keeps the fake behaviour. `mockReset` wipes calls and any behaviour I set later, going back to the original implementation given to `vi.fn` (I checked: `vi.fn(() => 'orig')` returns `'orig'` again after reset). `mockRestore` does that and also puts the real method back on a `spyOn`. You can turn these on globally with the `clearMocks`, `mockReset` or `restoreMocks` config options so tests do not leak into each other.

### Why Vitest over Jest?
It shares config with Vite, so Svelte, TypeScript and path aliases just work. It is fast thanks to Vite's transform pipeline and a smart watch mode that reruns only affected tests. The API is almost the same as Jest, so moving over is easy.

## Common mistakes
- Forgetting to `await` an async assertion, so the test always passes.
- Not restoring fake timers or stubbed globals, so later tests break in confusing ways.
- Mocking the module you are testing, or mocking so much that the test only checks the mocks.
- Using `toBe` on objects (checks same reference) when you meant `toEqual`.
- Testing floating point money with `toBe(0.3)`; `0.1 + 0.2` is `0.30000000000000004`. Use integers (paise) or `toBeCloseTo`.

## Resources
- [Vitest: Getting started](https://vitest.dev/guide/) - setup and first test
- [Vitest: Mocking](https://vitest.dev/guide/mocking) - functions, modules, globals
- [Vitest: Mocking timers](https://vitest.dev/guide/mocking/timers) - fake timers and system time
- [Vitest: vi API](https://vitest.dev/api/vi) - every `vi.*` helper
- [Svelte docs: Testing](https://svelte.dev/docs/svelte/testing) - Vitest setup for Svelte 5
