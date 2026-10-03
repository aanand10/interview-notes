# Unhandled rejections

> **In one line:** An unhandled rejection is a promise that failed and nobody attached a `.catch` or `try/catch` to it; the browser logs it and fires an `unhandledrejection` event, which you should listen to so these silent failures reach your error monitoring.

## Key points
- A rejection is "unhandled" if no handler is attached by the time the microtask queue is processed. In the browser the app keeps running, but the error is logged as "Uncaught (in promise)" and the user's action may silently do nothing.
- In **Node.js 15+** the default is to **crash the process** on an unhandled rejection (`--unhandled-rejections=throw`).
- Global handlers: `window.addEventListener('unhandledrejection', e => ...)` gives you `e.reason` and `e.promise`. Calling `e.preventDefault()` stops the default console error.
- If a handler is attached **later**, the browser fires `rejectionhandled`.
- `window.onerror` / the `error` event catch normal thrown errors, **not** promise rejections. You need both listeners.

## Example
```js
// 1. The bug: nobody catches this
async function loadWatchlist() {
  const res = await fetch('/api/watchlist');
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}
loadWatchlist();   // no await, no .catch -> unhandled rejection if it fails

// 2. The global safety net (set up once, at app start)
window.addEventListener('unhandledrejection', (event) => {
  reportError({
    type: 'unhandledrejection',
    message: event.reason?.message ?? String(event.reason),
    stack: event.reason?.stack,
    url: location.href,
  });
  // event.preventDefault();  // optional: hide the default console message
});

window.addEventListener('error', (event) => {
  reportError({ type: 'error', message: event.message, stack: event.error?.stack });
});

function reportError(payload) {
  // sendBeacon still works while the page is closing
  navigator.sendBeacon('/api/client-errors', JSON.stringify(payload));
}
```

The real fix is local handling; the global handler is only the net:

```js
try {
  watchlist = await loadWatchlist();
} catch (err) {
  error = 'Could not load your watchlist. Retry?';
}
```

Svelte / SvelteKit: errors thrown in `load` functions go to the [`handleError` hook](https://svelte.dev/docs/kit/hooks) and the `+error.svelte` page. In Svelte 5 you can also wrap UI in `<svelte:boundary>` to catch render errors.

## When to use it
- Set up the listeners in the app's entry (for SvelteKit, `hooks.client.js` has `handleError`; add the window listeners there or in the root layout).
- Send to Sentry, Datadog, or your own endpoint with release version, user id (no personal data) and route.
- In a trading app this catches "Place order clicked, nothing happened" bugs, which are the most costly and hardest to reproduce.

## Likely questions
### What happens when a promise rejects and nobody handles it?
Nothing stops in the browser: the error is printed as "Uncaught (in promise)" and an `unhandledrejection` event fires on `window`. But whatever the user was doing just fails silently, maybe with a spinner forever. In Node 15 and later the process crashes by default.

### How do you catch them globally?
`window.addEventListener('unhandledrejection', e => log(e.reason))`. In workers, listen on `self`. Pair it with the `error` event for normal exceptions. In Node, use `process.on('unhandledRejection', ...)`.

### Why does it matter for observability?
Observability means being able to see what is going wrong in production from logs and metrics. Unhandled rejections never reach a `catch`, so without a global listener they never reach your logs. You only hear about them from angry users. With the listener you get the error, stack, route and release, and can alert on spikes after a deploy.

### Does `try/catch` around an async call catch it?
Only if you `await` it inside the `try`. `try { doAsync(); } catch {}` without `await` does not catch anything; the function returned a promise and the `try` block finished before it rejected.

### Does `Promise.all` cause unhandled rejections?
If you `await Promise.all(...)` with a `try/catch`, the first rejection is handled, and later rejections from the other promises are ignored quietly (they are treated as handled). The common bug is creating promises early and awaiting them one by one: if the second one rejects while you wait for the first, it is unhandled for a while.

## Common mistakes
- Fire-and-forget calls (`save()` with no `await` or `.catch`) inside click handlers.
- `.then(onSuccess)` with no `.catch`, or a `.catch` that throws again.
- Using the global handler as the only error handling; the user still sees nothing.
- Logging only `event.reason` as a string and losing the stack.

## Resources
- [MDN: unhandledrejection event](https://developer.mozilla.org/en-US/docs/Web/API/Window/unhandledrejection_event) - event shape, reason, preventDefault
- [MDN: rejectionhandled event](https://developer.mozilla.org/en-US/docs/Web/API/Window/rejectionhandled_event) - when a late handler is attached
- [javascript.info: Error handling with promises](https://javascript.info/promise-error-handling) - implicit try/catch and unhandled rejections
- [SvelteKit: Hooks](https://svelte.dev/docs/kit/hooks) - handleError for client and server errors
