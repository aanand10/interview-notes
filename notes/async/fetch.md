# fetch

> **In one line:** `fetch(url, options)` returns a promise of a `Response`; it only rejects on network-level failures (offline, DNS, CORS, abort), so for HTTP errors like 404 or 500 you must check `res.ok` yourself.

## Key points
- **Two steps**: `await fetch()` gives you the headers and status. Then `await res.json()` or `await res.text()` reads the body. Both steps are async ([MDN: Using Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)).
- **No reject on 404/500.** The server did answer, so from fetch's view the request worked. Check [`res.ok`](https://developer.mozilla.org/en-US/docs/Web/API/Response/ok) (true for status 200-299) or `res.status`.
- **Rejects with** `TypeError` on network failure or CORS block, and with `AbortError` / `TimeoutError` when aborted.
- **POST JSON**: `method: 'POST'`, header `Content-Type: application/json`, `body: JSON.stringify(data)`.
- **Body can be read once.** Calling `res.json()` then `res.text()` throws. Use `res.clone()` if you need both.

## Example
A reusable helper you can say out loud in an interview:

```ts
class HttpError extends Error {
  constructor(public status: number, public body: string) {
    super(`HTTP ${status}`);
  }
}

async function request<T>(url: string, options: RequestInit = {}): Promise<T> {
  const res = await fetch(url, {
    ...options,
    headers: { Accept: 'application/json', ...options.headers }
  });

  if (!res.ok) {
    // Error bodies are often plain text or HTML, so read as text.
    throw new HttpError(res.status, await res.text());
  }
  if (res.status === 204) return undefined as T;   // No Content: nothing to parse
  return res.json() as Promise<T>;
}

// POST with a JSON body
const order = await request<{ id: string }>('/api/orders', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ symbol: 'AAPL', side: 'BUY', qty: 10 })
});
```

Proof that a 404 does not reject (node, against a local test server):
```text
404 resolved? ok = false status = 404 text = not found
network error: TypeError fetch failed
```
The 404 came back as a normal response with `ok = false`. Only the unreachable server made fetch reject (browsers say `TypeError: Failed to fetch`).

## When to use it
- Any REST call in a Svelte app: quotes, placing orders, loading the watchlist.
- In SvelteKit, use the `fetch` passed to `load` functions; it works on the server and avoids a second request on hydration ([SvelteKit: loading data](https://svelte.dev/docs/kit/load)).

## Likely questions
### Why does fetch not reject on a 404?
Because fetch only rejects when it could not get a response at all: no network, DNS failure, CORS block, or abort. A 404 or 500 is a valid HTTP response from the server, so the promise fulfills. You have to check `res.ok` and throw yourself. Libraries like axios do reject on non-2xx, which is why people get confused.

### How do you check for errors properly?
```js
const res = await fetch('/api/quote/AAPL');
if (!res.ok) throw new Error(`Quote failed: ${res.status}`);
const quote = await res.json();
```
And wrap the whole thing in `try/catch` so you catch both network errors and your own HTTP errors.

### How do you send a POST with JSON?
Set `method: 'POST'`, set the `Content-Type: application/json` header so the server knows how to parse it, and pass `body: JSON.stringify(obj)`. If you forget `JSON.stringify`, the body becomes the string `"[object Object]"`. If you send `FormData`, do **not** set `Content-Type` yourself; the browser adds it with the multipart boundary.

### Text vs JSON: how do you read the body?
`res.json()` reads the body and parses it as JSON. If the body is not valid JSON (for example an HTML error page), it rejects with a `SyntaxError`. `res.text()` returns the raw string and never fails on format. Other readers: `res.blob()` for files, `res.arrayBuffer()`, `res.formData()`. A safe pattern when the server is unreliable:
```js
const text = await res.text();
const data = text ? JSON.parse(text) : null;
```

### Can you read the body twice?
No. The body is a stream, so after the first read `res.bodyUsed` is true and a second read throws `TypeError`. Use `res.clone()` before reading if two consumers need it.

### Does fetch send cookies?
Same-origin requests send cookies by default (`credentials: 'same-origin'`). For a cross-origin API that uses cookies, you need `credentials: 'include'` and the server must allow it with CORS headers.

### How do you set a timeout?
fetch has no timeout option. Pass a signal: `fetch(url, { signal: AbortSignal.timeout(5000) })`. See the AbortController note.

## Common mistakes
- Not checking `res.ok`, then crashing on `res.json()` of an HTML error page.
- Forgetting `await` on `res.json()`.
- Forgetting `JSON.stringify` or the `Content-Type` header on POST.
- Reading the body twice.

## Resources
- [MDN: Using the Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch) - full guide including errors and bodies
- [MDN: Response.ok](https://developer.mozilla.org/en-US/docs/Web/API/Response/ok) - what counts as ok
- [javascript.info: Fetch](https://javascript.info/fetch) - simple examples of GET, POST and body readers
- [SvelteKit: Loading data](https://svelte.dev/docs/kit/load) - using fetch inside load functions
