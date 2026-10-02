# Testing pyramid

> **In one line:** The testing pyramid says write many fast, cheap unit tests at the bottom, fewer integration tests in the middle, and only a handful of slow, expensive end-to-end tests at the top for the flows that make money.

## Key points
- **Unit tests** check one function or module alone (for example `toPaise('19.9')`). They run in milliseconds and pinpoint the bug.
- **Integration tests** check a few pieces working together, like a Svelte component plus its store plus a mocked API. For frontend, component tests live here.
- **End-to-end (E2E) tests** drive a real browser through the real app, like "log in, search TCS, place a buy order". They give the most confidence but are slow and can be flaky (pass or fail randomly).
- Going up the pyramid: **confidence goes up, but speed goes down and cost goes up** (time to write, time to run, time to debug, CI minutes).
- Many frontend teams use the **testing trophy** variant: static checks (TypeScript, ESLint) at the base and most effort on integration tests, because they test how users actually use the UI. See [Kent C. Dodds on the testing trophy](https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications).

## Example
One feature, the order form, tested at each level:

```js
// 1. Unit (Vitest): pure logic, no DOM, runs in ~1ms
import { expect, it } from 'vitest';
import { toPaise } from './money.js';

it('converts rupees to paise without float errors', () => {
  expect(toPaise('19.9')).toBe(1990);
});
```

```js
// 2. Integration / component (Vitest + Testing Library): component + fake API
import { render, screen } from '@testing-library/svelte';
import userEvent from '@testing-library/user-event';
import { expect, it, vi } from 'vitest';
import OrderForm from './OrderForm.svelte';

it('shows server error on failed order', async () => {
  const user = userEvent.setup();
  const placeOrder = vi.fn().mockRejectedValue(new Error('Insufficient funds'));
  render(OrderForm, { symbol: 'TCS', placeOrder });

  await user.type(screen.getByLabelText('Quantity'), '2');
  await user.click(screen.getByRole('button', { name: 'Buy TCS' }));
  expect(await screen.findByRole('alert')).toHaveTextContent('Insufficient funds');
});
```

```ts
// 3. E2E (Playwright): real browser, real routing, real (test) backend
import { test, expect } from '@playwright/test';

test('user can place a buy order', async ({ page }) => {
  await page.goto('/stocks/TCS');
  await page.getByLabel('Quantity').fill('1');
  await page.getByRole('button', { name: 'Buy TCS' }).click();
  await expect(page.getByRole('status')).toHaveText('Order placed');
});
```

| Level | Tool | Speed | What it catches | What it misses |
| --- | --- | --- | --- | --- |
| Static | TypeScript, ESLint | Instant | Typos, wrong types, banned patterns | Wrong logic |
| Unit | Vitest | ms | Logic bugs in pure code | Wiring between parts |
| Integration | Vitest + Testing Library | 10s of ms | UI behaviour, state, error states | Real network, real browser quirks |
| E2E | Playwright | seconds | Routing, auth, real API contract | Hard to cover every edge case |

## When to use it
- **Unit:** money math, price formatting, P&L calculation, order validation rules, reducers, utility functions like debounce.
- **Integration:** order form, watchlist add/remove, login form errors, a live-price row that turns green or red.
- **E2E:** a short list of critical journeys only: login, place order, add funds / checkout, KYC step. If these break, the business loses money, so they deserve the slow, high-confidence test.

## Likely questions
### What is the difference between unit, integration and E2E tests?
A unit test checks one small piece in isolation, with its dependencies faked. An integration test checks that a few real pieces work together, like a component with its store, with only the network mocked. An E2E test runs the whole app in a real browser like a user would. Each step up gives more confidence but is slower and harder to debug.

### What would you test at each level for a stock trading app?
Unit: pure logic like converting rupees to paise, computing order value, validating quantity against lot size. Integration: the order form shows errors, disables the button while submitting, and shows the server error. E2E: just the money paths, like login and place a buy order, plus maybe add funds. I keep E2E few because each one costs seconds and can be flaky.

### Why not just write E2E tests for everything? They test the real thing.
Because they are slow, so CI takes forever and developers stop running them. They also fail for reasons unrelated to my code, like a slow test server or animation timing, so people start ignoring red builds. And when one fails, it is hard to know which part broke. Lower-level tests fail fast and point at the exact line.

### What are the cost and speed trade-offs?
Unit tests are cheap to write, run in milliseconds and rarely break for wrong reasons, but they can pass while the app is broken because the pieces do not fit. E2E tests catch wiring and contract bugs but cost the most to write, run and maintain. So the goal is the best confidence per minute of CI time: lots at the bottom, few at the top.

### Have you heard of the testing trophy? Which do you prefer?
Yes. The trophy puts static analysis at the bottom and the biggest slice on integration tests. For UI code I lean that way, because a component test that clicks a button and checks the screen tests real behaviour, while a unit test of a tiny internal function often just tests implementation. I still unit test pure logic like money math heavily.

### What is code coverage and is 100% a good goal?
Coverage shows which lines ran during tests. It is useful to find untested areas, but 100% is not a good target, because a line can run without its result being checked. I prefer a sensible threshold in CI plus near-full coverage of risky code like payments and order validation.

## Common mistakes
- An "ice cream cone": mostly manual and E2E tests, very few unit tests. Slow and flaky.
- Testing implementation details (internal state, private functions) so tests break on every refactor.
- Mocking so much in an "integration" test that it no longer tests any integration.
- Chasing a coverage number instead of testing the risky paths.

## Resources
- [Martin Fowler: The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) - the classic long-form explanation
- [web.dev: Testing strategies](https://web.dev/articles/ta-strategies) - pyramid, trophy and other shapes compared
- [web.dev: Types of tests](https://web.dev/articles/ta-types) - clear definitions of each level
- [Testing trophy by Kent C. Dodds](https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications) - the frontend-friendly variant
