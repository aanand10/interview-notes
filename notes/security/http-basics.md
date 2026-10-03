# HTTP basics

> **In one line:** HTTP is the request-response protocol of the web: the client sends a method, URL, headers and maybe a body, and the server returns a status code, headers and a body.

## Key points
- **Methods** say what you want to do: `GET` read, `POST` create/act, `PUT` replace, `PATCH` partly update, `DELETE` remove.
- **Status codes** say what happened: 2xx success, 3xx redirect, 4xx client mistake, 5xx server failure.
- **Headers** carry metadata: content type, auth, caching, cookies, CORS.
- **Idempotent** means doing it twice has the same effect as doing it once. This decides what is safe to retry.
- **HTTP/2** multiplexes many requests on one TCP connection; **HTTP/3** does the same over QUIC (UDP), avoiding TCP head-of-line blocking.

## Methods

| Method | Use | Safe (no side effect)? | Idempotent? |
|---|---|---|---|
| GET | Read data | Yes | Yes |
| HEAD | Like GET, headers only | Yes | Yes |
| OPTIONS | Ask what is allowed (CORS preflight) | Yes | Yes |
| POST | Create / perform an action (place order) | No | No |
| PUT | Replace a resource fully | No | Yes |
| PATCH | Partial update | No | Not guaranteed |
| DELETE | Remove | No | Yes |

## Status codes worth knowing

| Code | Meaning | Frontend reaction |
|---|---|---|
| 200 OK / 201 Created / 204 No Content | Success | Update UI |
| 301 / 302 / 307 / 308 | Redirects (307/308 keep the method) | Browser follows |
| 304 Not Modified | Cached copy is still fresh | Browser uses cache |
| 400 Bad Request / 422 Unprocessable | Invalid input | Show field errors |
| 401 Unauthorized | Not logged in / token expired | Refresh token or go to login |
| 403 Forbidden | Logged in but not allowed | Show "no access" |
| 404 Not Found | Missing | Not-found page |
| 409 Conflict | State conflict (duplicate order) | Show message, refetch |
| 429 Too Many Requests | Rate limited | Back off, respect `Retry-After` |
| 500 / 502 / 503 / 504 | Server or gateway problem | Retry idempotent calls with backoff |

## Example: idempotent retry for a payment

```ts
// POST is not idempotent, so a blind retry could charge twice.
// An Idempotency-Key lets the server recognise the retry and return the first result.
async function pay(amount: number) {
  const key = crypto.randomUUID(); // one key per user action, reused on retries
  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      const res = await fetch('/api/payments', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'Idempotency-Key': key },
        body: JSON.stringify({ amount }),
      });
      if (res.status < 500) return res;        // success or client error: do not retry
    } catch { /* network error: safe to retry because of the key */ }
    await new Promise((r) => setTimeout(r, 2 ** attempt * 500));
  }
  throw new Error('Payment status unknown, check order history');
}
```

## Common headers

| Header | Purpose |
|---|---|
| `Content-Type` / `Accept` | Format of the body / format I want back |
| `Authorization` | `Bearer <token>` |
| `Cookie` / `Set-Cookie` | Session cookies |
| `Cache-Control`, `ETag`, `If-None-Match` | Caching and revalidation (304) |
| `Origin`, `Access-Control-Allow-*` | CORS |
| `Content-Security-Policy`, `Strict-Transport-Security` | Security policies |

## HTTP/1.1 vs HTTP/2 vs HTTP/3

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP (with TLS in browsers) | QUIC over UDP (TLS 1.3 built in) |
| Requests per connection | One at a time (browsers open ~6 connections per host) | Many in parallel (multiplexing) | Many in parallel |
| Headers | Plain text, repeated | Binary, compressed (HPACK) | Compressed (QPACK) |
| Head-of-line blocking | Yes, per connection | Fixed at HTTP level, but one lost TCP packet still stalls all streams | Fixed: a lost packet only stalls its own stream |
| Setup | TCP + TLS handshakes | Same | Faster handshake; connection survives network change (Wi-Fi to 4G) |

## When to use it
- Mobile users on flaky networks: HTTP/3 helps because of faster setup and connection migration.
- With HTTP/2+, old tricks like domain sharding and sprite sheets matter less; many small files are fine.
- Order placement: POST + idempotency key; reading the order book: GET with caching rules.

## Likely questions

### What is idempotency and why does it matter?
An operation is idempotent if repeating it leaves the server in the same state as doing it once. GET, PUT and DELETE are idempotent; POST is not. It matters for retries: on a timeout I can safely retry a GET, but retrying a "place order" POST could create two orders. So for payments and orders we send an idempotency key so the server de-duplicates.

### 401 vs 403?
401 means "I don't know who you are": missing or expired credentials, so try refreshing or log in. 403 means "I know who you are, but you are not allowed". Retrying with the same credentials will not help a 403.

### PUT vs PATCH vs POST?
PUT replaces the whole resource at a known URL, and is idempotent. PATCH changes some fields. POST creates something new or triggers an action, where the server decides the result, and it is not idempotent.

### What did HTTP/2 and HTTP/3 change?
HTTP/2 sends many requests in parallel over one connection with compressed binary headers, so we no longer need many connections. But it still runs on TCP, so one lost packet blocks all streams. HTTP/3 runs on QUIC over UDP, where each stream is independent, the handshake is quicker, and the connection survives switching networks, which is great on mobile.

## Resources
- [MDN: HTTP request methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods) - every method with safe/idempotent flags
- [MDN: HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status) - full list with meaning
- [MDN: Evolution of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Evolution_of_HTTP) - HTTP/1.1 to HTTP/3 history
- [MDN: Idempotent](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent) - short definition with examples
