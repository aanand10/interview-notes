# Same-origin policy

> **In one line:** The same-origin policy stops a page from reading data from a different origin (scheme + host + port), and CORS is how a server opts in to let a specific other origin read its responses.

## Key points
- An **origin** is `scheme://host:port`. All three must match exactly to be the same origin.
- The policy blocks **reading** cross-origin responses, DOM of cross-origin iframes, and cross-origin storage. It does not block **embedding**: `<img>`, `<script src>`, `<link>` and forms can still load or send to other origins.
- [**CORS**](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) (Cross-Origin Resource Sharing): the server sends headers like `Access-Control-Allow-Origin` to say "this origin may read my response". The browser enforces it; the server only declares it.
- Non-simple requests (e.g. `PUT`, JSON `Content-Type`, custom headers like `Authorization`) trigger a **preflight**: an `OPTIONS` request asking permission first.
- To send cookies cross-origin you need `credentials: 'include'` on the client, plus `Access-Control-Allow-Credentials: true` and an exact origin (not `*`) on the server.

## Example
Same origin or not, compared with `https://app.broker.com/page`:

| URL | Same origin? | Why |
|---|---|---|
| `https://app.broker.com/orders` | Yes | Only the path differs |
| `http://app.broker.com` | No | Scheme differs |
| `https://api.broker.com` | No | Host differs (subdomain counts) |
| `https://app.broker.com:8443` | No | Port differs (default is 443) |

A cross-origin call with cookies from `https://app.broker.com` to `https://api.broker.com`:

```js
const res = await fetch('https://api.broker.com/orders', {
  method: 'POST',
  credentials: 'include',                       // send cookies
  headers: { 'Content-Type': 'application/json' }, // JSON -> preflight needed
  body: JSON.stringify({ symbol: 'AAPL', qty: 5 }),
});
```

What happens on the wire:

```bash
# 1. Preflight (browser sends automatically)
OPTIONS /orders
Origin: https://app.broker.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type

# Server answers
Access-Control-Allow-Origin: https://app.broker.com
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: content-type
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 600        # cache this preflight for 10 minutes

# 2. Real POST is sent, response must again include
Access-Control-Allow-Origin: https://app.broker.com
Access-Control-Allow-Credentials: true
```

## When to use it
- Your SvelteKit app on `app.broker.com` calls an API on `api.broker.com`: the API must send CORS headers. Or avoid CORS by calling the API through SvelteKit server routes on the same origin.
- In dev, use a Vite proxy so the browser only talks to `localhost:5173`.

## Likely questions
### What counts as the same origin?
Scheme, host and port must all be the same. `https://a.com` and `https://a.com:443` are the same because 443 is the default. `https://a.com` and `https://www.a.com` are different. Path and query do not matter.

### How does CORS relax the same-origin policy?
The browser adds an `Origin` header to the cross-origin request. If the server's response has `Access-Control-Allow-Origin` matching that origin (or `*`), the browser lets your JS read the response; otherwise it blocks the read and you see a CORS error. For non-simple requests, the browser first sends an `OPTIONS` preflight and only sends the real request if the server approves.

### Does CORS protect the server?
No. CORS is about protecting the user's data in the browser. A simple request (like a form-style POST) may still reach the server even if the response is blocked. Tools like curl ignore CORS completely. The server still needs auth and CSRF protection.

### What is a simple request?
`GET`, `HEAD` or `POST` with only safe headers, and a `Content-Type` of `text/plain`, `multipart/form-data` or `application/x-www-form-urlencoded`. These skip the preflight. Anything else, like `application/json` or an `Authorization` header, gets a preflight.

### Origin vs site?
Site is wider: scheme plus the registrable domain (like `broker.com`), ignoring subdomains and port. `app.broker.com` and `api.broker.com` are same-site but cross-origin. `SameSite` cookies use the site idea; CORS and storage use origin.

## Common mistakes
- `Access-Control-Allow-Origin: *` with credentials. The browser rejects it; you must echo the exact origin (and add `Vary: Origin`).
- Trying to fix CORS in frontend code. It is always a server (or proxy) setting.
- Using `mode: 'no-cors'` to "fix" the error; you get an opaque response you cannot read.

## Resources
- [MDN: Same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) - definition and examples
- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) - headers, preflight, credentials
- [javascript.info: Fetch Cross-Origin Requests](https://javascript.info/fetch-crossorigin) - simple vs preflighted requests explained
- [web.dev: Same-site and same-origin](https://web.dev/articles/same-site-same-origin) - site vs origin
