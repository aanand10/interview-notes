# XSS (Cross-Site Scripting)

> **In one line:** XSS is when an attacker gets their own JavaScript to run on my page, so it runs with my user's session and can read data or act as them.

## Key points
- **Three types:** stored (saved in the database), reflected (bounced back from the URL or request), and DOM-based (my own frontend JS puts untrusted data into a dangerous place).
- **Svelte escapes by default.** `{value}` in markup is inserted as text, so `<script>` shows up as plain characters and never runs.
- **`{@html}` turns escaping off.** Whatever string you pass is parsed as real HTML. Only use it with trusted or sanitised content.
- **Sanitise** untrusted HTML with a library like [DOMPurify](https://github.com/cure53/DOMPurify) before rendering it.
- **[Content Security Policy (CSP)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP)** is the second wall: even if a payload gets in, the browser refuses to run scripts that are not allowed.

## The three types, simply

| Type | Where the payload lives | Example |
|---|---|---|
| Stored | Server / database | A user saves a "nickname" of `<img src=x onerror=alert(1)>`; every viewer of the leaderboard runs it. |
| Reflected | The request (URL, form) | `/search?q=<script>...</script>` and the server echoes `q` into the HTML without escaping. Attacker sends the link to a victim. |
| DOM-based | Only in the browser | Frontend code does `el.innerHTML = location.hash.slice(1)`. The server never sees the payload. |

## Example

```svelte
<script>
  import DOMPurify from 'dompurify';

  // Pretend this came from an API (user-written note on a stock)
  let note = $state('<img src=x onerror="alert(document.cookie)"> Buy below 200');

  // Safe: sanitise once, keep only harmless tags
  let safeNote = $derived(
    DOMPurify.sanitize(note, { ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'] })
  );
</script>

<!-- 1. Safe: Svelte escapes this. The user sees the raw text "<img ...>" -->
<p>{note}</p>

<!-- 2. DANGEROUS: the onerror handler runs -->
<!-- <p>{@html note}</p> -->

<!-- 3. Safe enough: DOMPurify removed the onerror attribute and the <img> -->
<p>{@html safeNote}</p>
```

Other DOM-XSS sinks to watch in plain JS:

```js
el.innerHTML = userInput;          // dangerous: parses HTML
el.textContent = userInput;        // safe: plain text
location.href = userInput;         // dangerous if it can be "javascript:..."
const ok = new URL(userInput, location.origin);
if (ok.protocol === 'https:') location.href = ok.href; // allow-list the scheme
```

A strict CSP header (nonce-based) looks like this:

```bash
Content-Security-Policy: script-src 'nonce-r4nd0m' 'strict-dynamic'; object-src 'none'; base-uri 'none'
```

Only `<script nonce="r4nd0m">` tags (and scripts they load) can run. An injected `<script>` or inline `onerror=` has no nonce, so it is blocked.

## When to use it
- **Checkout / payment page:** the page shows merchant name, item names, and notes that come from other systems. Render them with `{value}`, never `{@html}`.
- **Rich text** like research notes or news summaries: sanitise with DOMPurify, ideally on the server too.
- **Every production app:** ship a CSP. Start with `Content-Security-Policy-Report-Only` to collect violations, then enforce.

## Likely questions

### What are the types of XSS?
Stored XSS is saved on the server and served to every user who views it, like a malicious comment. Reflected XSS comes from the request, like a search term in the URL that the server prints back unescaped; the attacker tricks the victim into clicking the link. DOM-based XSS happens fully in the browser, when my JS takes something like `location.hash` and writes it with `innerHTML` or `eval`. The fix for all three is the same idea: treat data as text, not code.

### How does Svelte protect you from XSS?
When I write `{name}` in a Svelte template, the compiler sets it as a text node, so HTML characters are shown, not parsed. Attribute values are also set safely. So the default path is safe. The escape hatch is `{@html}`, which inserts raw HTML, and Svelte does no sanitising there. Also, an `href={userUrl}` can still be `javascript:alert(1)`, so I validate URL schemes myself.

### When would you use `{@html}` and how do you make it safe?
Only when I truly need HTML, like a CMS article or a formatted help text. I sanitise with DOMPurify using an allow-list of tags, and I prefer sanitising on the server too, so the stored data is clean. If the content is from my own team (a static translation string), it is trusted, but I still keep user values out of it.

### What is DOMPurify doing?
It parses the HTML in an inert way, walks the tree, and removes anything not on its allow-list: `<script>`, event handler attributes like `onerror`, `javascript:` URLs, and so on. It returns a clean string. It is a browser library, so on the server you run it with a DOM implementation like jsdom.

### What is CSP and how does it help with XSS?
CSP is an HTTP response header that tells the browser which sources of script, style, images, and frames are allowed. A strict CSP with nonces or hashes means injected inline scripts will not run, even if my escaping fails somewhere. It is defence in depth, not a replacement for escaping. I avoid `'unsafe-inline'` and `'unsafe-eval'` because they cancel most of the protection. SvelteKit can add nonces or hashes for you through the [`csp` config](https://svelte.dev/docs/kit/configuration#csp).

### Can an HttpOnly cookie stop XSS?
No. It stops the script from *reading* the cookie, so the token is harder to steal. But the attacker's script can still call my APIs from the page, and the browser attaches the cookie. So HttpOnly limits damage; it does not prevent XSS.

### What are Trusted Types?
A browser feature you turn on with CSP (`require-trusted-types-for 'script'`). After that, dangerous sinks like `innerHTML` only accept special typed objects made by a policy you write, not plain strings. It forces all HTML writes through one reviewed sanitiser. Support is best in Chromium browsers, so I treat it as an extra layer.

## Common mistakes
- Thinking "React/Svelte escape, so I am safe" while using `{@html}` or `innerHTML` with API data.
- Validating input only on the client. The server must also escape or sanitise.
- Putting user data into `href`, `src`, inline `<script>` JSON, or `style` without checking. Each context needs its own encoding.
- CSP with `'unsafe-inline'`: looks secure, blocks almost nothing.
- Sanitising, then changing the string afterwards (for example, string-replacing in new HTML), which can re-open the hole.

## Resources
- [OWASP: XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) - the rules for each output context
- [MDN: Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP) - how CSP works with examples
- [Svelte docs: {@html}](https://svelte.dev/docs/svelte/@html) - what it does and the warning about sanitising
- [web.dev: Strict CSP](https://web.dev/articles/strict-csp) - nonce and hash based CSP step by step
