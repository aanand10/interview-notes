# Testing React components

> **In one line:** I test components the way a user uses them, with React Testing Library: find elements by role and label, interact with `user-event`, mock the network with MSW, and assert on what appears on screen, not on internal state.

## Key points
- [React Testing Library](https://testing-library.com/docs/react-testing-library/intro) (RTL) philosophy: "The more your tests resemble the way your software is used, the more confidence they can give you." Test **behaviour**, not implementation details like state or hook calls.
- **Query priority:** `getByRole` first (also checks accessibility), then `getByLabelText`, `getByText`. `getByTestId` is the last resort. See [query priority](https://testing-library.com/docs/queries/about#priority).
- **get / query / find:** `getBy` throws if missing, `queryBy` returns `null` (use to assert something is *not* there), `findBy` returns a promise and waits (use for async UI).
- [`user-event`](https://testing-library.com/docs/user-event/intro) simulates real typing and clicking (focus, key events) better than `fireEvent`. Always `await` it.
- [MSW](https://mswjs.io/docs/) (Mock Service Worker) mocks the **network**, so your real `fetch` code runs and tests do not depend on how you fetch.

## Example
A login form test with Vitest, RTL, user-event and MSW.

```tsx
// LoginForm.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { http, HttpResponse } from "msw";
import { setupServer } from "msw/node";
import { beforeAll, afterEach, afterAll, test, expect } from "vitest";
import { LoginForm } from "./LoginForm";

const server = setupServer(
  http.post("/api/login", async ({ request }) => {
    const { password } = (await request.json()) as { email: string; password: string };
    return password === "correct"
      ? HttpResponse.json({ name: "Asha" })
      : HttpResponse.json({ message: "Invalid credentials" }, { status: 401 });
  })
);
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test("logs in with valid credentials", async () => {
  const user = userEvent.setup();
  render(<LoginForm />);

  await user.type(screen.getByLabelText(/email/i), "asha@example.com");
  await user.type(screen.getByLabelText(/password/i), "correct");
  await user.click(screen.getByRole("button", { name: /log in/i }));

  expect(await screen.findByText(/welcome, asha/i)).toBeInTheDocument(); // waits for fetch
});

test("shows an error for a wrong password", async () => {
  const user = userEvent.setup();
  render(<LoginForm />);

  await user.type(screen.getByLabelText(/email/i), "asha@example.com");
  await user.type(screen.getByLabelText(/password/i), "wrong");
  await user.click(screen.getByRole("button", { name: /log in/i }));

  expect(await screen.findByRole("alert")).toHaveTextContent(/invalid credentials/i);
});

test("submit is disabled until both fields are filled", async () => {
  render(<LoginForm />);
  expect(screen.getByRole("button", { name: /log in/i })).toBeDisabled();
});
```

## When to use it
- Unit and integration tests for forms (login, order entry, KYC), lists and error states.
- MSW handlers can be shared between tests, Storybook and local dev.
- Svelte equivalent: `@testing-library/svelte` has the same queries, so the skills transfer directly.

## Likely questions
### What is the RTL philosophy?
Test what the user sees and does. A user does not know your component has a `useState` called `isOpen`; they see a button and a dialog. So I query by role and accessible name, click like a user, and assert on visible output. Refactoring the inside then does not break tests, and querying by role also catches accessibility problems, like a button with no label.

### Why user-event over fireEvent?
`fireEvent.change` sets a value and fires one event. `user.type` fires focus, keydown, keypress, input and keyup for every character, like a real user, so it catches bugs such as a `maxLength` or an `onKeyDown` handler. Since v14, call `userEvent.setup()` first and `await` every action.

### How do you mock fetch?
I use MSW. It intercepts requests at the network level, so my component's real `fetch` or TanStack Query code runs. I override a handler inside one test with `server.use(...)` to test the error path. This is better than `vi.fn()` on `fetch`, which ties tests to how I fetch.

### How do you test a custom hook?
Prefer testing it through a small component that uses it. If the hook is a reusable library piece, use `renderHook` from `@testing-library/react` and wrap state changes in `act`.

```tsx
import { renderHook, act } from "@testing-library/react";

test("useCounter increments", () => {
  const { result } = renderHook(() => useCounter(0));
  act(() => result.current.increment());
  expect(result.current.count).toBe(1);
});
```

### When do you use snapshot tests?
Rarely. Big snapshots get approved without reading and break on every markup change, so they give little confidence. I use small inline snapshots for stable output, like a formatted price string or a serialized object. For visual changes, screenshot tests in Playwright are more useful.

## Common mistakes
- Using `getBy` to assert absence. It throws; use `expect(screen.queryByText(...)).not.toBeInTheDocument()`.
- Forgetting `await` on `user-event` calls or on `findBy`, which causes flaky tests and "not wrapped in act" warnings.
- Testing internal state or class names instead of behaviour.

## Resources
- [Testing Library: React intro](https://testing-library.com/docs/react-testing-library/intro) - setup and philosophy
- [Testing Library: Query priority](https://testing-library.com/docs/queries/about#priority) - which query to use
- [Testing Library: user-event](https://testing-library.com/docs/user-event/intro) - realistic interactions
- [MSW docs](https://mswjs.io/docs/) - network-level API mocking
