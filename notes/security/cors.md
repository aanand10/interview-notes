# CORS (Cross-Origin Resource Sharing)

> **In one line:** CORS is how a server tells the browser "it's OK for pages from this other origin to read my responses"; without it, the same-origin policy blocks the page from reading cross-origin responses.

## Key points
- An **origin** is scheme + host + port. `https://app.example` and `https://api.example` are different origins.
- The **[same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)** is the browser rule; CORS is the opt-in relaxation. It protects the **user's** data (cookies, intranet), not the server.
- **Simple requests** are sent directly; the browser then checks the response headers before giving it to JS.
- **Preflight:** for "non-simple" requests, the browser first sends an `OPTIONS` request asking permission, and only sends the real request if the answer allows it.
- The **server** sets `Access-Control-Allow-*` headers. The frontend cannot "fix" CORS by itself.

## Why the browser blocks it
If any site could `fetch('https://bank.example/api/balance', { credentials: 'include' })` and read the result, every site you visit could read your bank data using your cookies. So by default the browser lets the request go out (sometimes) but **hides the response** from the calling page unless the server opts in.

Note: CORS is a browser thing. `curl`, Postman, and server-to-server calls ignore it.

## Simple vs preflighted

A request is **simple** (no preflight) when all of these are true:
- Method is `GET`, `HEAD`, or `POST`.
- Only safelisted headers (like `Accept`, `Accept-Language`, `Content-Language`, `Content-Type`).
- `Content-Type` is only `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`.

Anything else triggers a **preflight**: `PUT`/`DELETE`/`PATCH`, `Content-Type: application/json`, or a custom header like `Authorization` or `X-Request-Id`.

## Example

```js
// Page on https://app.example calls API on https://api.example
const res = await fetch('https://api.example/orders', {
  method: 'POST',
  credentials: 'include',                       // send cookies
  headers: { 'Content-Type': 'application/json' }, // JSON -> preflight needed
  body: JSON.stringify({ symbol: 'TCS', qty: 5 }),
});
```

What happens on the wire:

```bash
# 1. Preflight sent by the browser automatically
OPTIONS /orders HTTP/1.1
Origin: https://app.example
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type

# 2. Server answers
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example
Access-Control-Allow-Methods: GET, POST, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 600
Vary: Origin

# 3. Real request goes out; the actual response must ALSO include
Access-Control-Allow-Origin: https://app.example
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: X-RateLimit-Remaining
```

## Headers the server sets

| Header | Meaning |
|---|---|
| `Access-Control-Allow-Origin` | Which origin may read the response. `*` or one exact origin (not a list). |
| `Access-Control-Allow-Methods` | Methods allowed (preflight response). |
| `Access-Control-Allow-Headers` | Request headers allowed (preflight response). |
| `Access-Control-Allow-Credentials` | `true` lets cookies/auth be used and the response be read. |
| `Access-Control-Max-Age` | How many seconds the browser may cache the preflight result. |
| `Access-Control-Expose-Headers` | Response headers JS may read beyond the basic safe ones. |
| `Vary: Origin` | Tells caches the response depends on the Origin, when you echo it back. |

## When to use it
- Frontend on `app.example`, API on `api.example`: configure CORS on the API with an **allow-list** of origins.
- Avoid it where you can: in SvelteKit, call the API from a `+page.server.ts` load or a same-origin `/api` route (backend-for-frontend), so the browser only talks to its own origin.
- In dev, a Vite proxy (`server.proxy`) makes API calls same-origin.

## Likely questions

### Why does the browser block my request when Postman works?
Because CORS is enforced by the browser to protect the user. Postman has no user cookies to protect, so it does not check. The fix is on the server: return the right `Access-Control-Allow-Origin` (and friends) for my frontend's origin.

### What is a preflight request?
An `OPTIONS` request the browser sends before a non-simple request, with `Origin`, `Access-Control-Request-Method` and `Access-Control-Request-Headers`. The server replies with what it allows. If the reply matches, the browser sends the real request; if not, it fails with a CORS error and the real request is never sent. `Access-Control-Max-Age` caches the answer so we do not pay an extra round trip every time.

### Can I use `Access-Control-Allow-Origin: *` with cookies?
No. With `credentials: 'include'`, the browser requires an exact origin and `Access-Control-Allow-Credentials: true`. Also never blindly echo back any `Origin` with credentials; that is the same as `*` and lets any site read user data. Check the origin against an allow-list.

### Is CORS a security feature for my server?
Not really. It does not stop requests from reaching the server; simple requests are sent and processed even when the response is blocked. The server still needs authentication, authorization and CSRF protection.

### What is `mode: 'no-cors'`?
It lets you send a request without CORS, but you get an "opaque" response you cannot read. It does not bypass CORS; it is mainly for things like caching third-party resources in a service worker.

## Common mistakes
- Trying to fix CORS by adding `Access-Control-Allow-Origin` on the **request** from the client.
- Forgetting the server must answer `OPTIONS` (some frameworks route it to auth and return 401).
- Missing CORS headers on error responses, so a 500 shows up as a confusing CORS error.
- Echoing the `Origin` header without validation, or allowing `null` origin.

## Resources
- [MDN: CORS guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS) - simple vs preflight, all headers
- [MDN: Same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) - what an origin is and what is blocked
- [web.dev: Cross-Origin Resource Sharing](https://web.dev/articles/cross-origin-resource-sharing) - short practical explanation
- [javascript.info: Fetch Cross-Origin Requests](https://javascript.info/fetch-crossorigin) - clear walkthrough with examples
