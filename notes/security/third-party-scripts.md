# Third-party scripts

> **In one line:** A third-party script runs with full power on my page, so I load as few as possible, pin them with Subresource Integrity, load them async so they don't block rendering, and keep them away from sensitive pages.

## Key points
- **Risks:** the script can read the DOM (including form fields), make requests with the user's session, and change the page. If the vendor or its CDN is hacked, so am I (supply-chain attack, like card-skimming code injected into payment pages).
- **Performance risk too:** a slow or blocking script hurts load time and responsiveness.
- **[Subresource Integrity (SRI)](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity):** add `integrity="sha384-..."`. The browser hashes the downloaded file and refuses to run it if the file changed.
- **Load async:** `async` (run as soon as downloaded, order not kept) or `defer` (run after HTML parsing, in order). Or inject the script after the page is interactive.
- **Limit damage:** CSP `script-src` allow-list, sandboxed iframes for widgets, and no analytics or chat widgets on checkout pages.

## Example

```html
<!-- Pinned version + SRI. crossorigin is needed so the browser can check the hash of a cross-origin file -->
<script
  src="https://cdn.jsdelivr.net/npm/dayjs@1.11.13/dayjs.min.js"
  integrity="sha384-REPLACE_WITH_REAL_HASH"
  crossorigin="anonymous"
  defer
></script>

<!-- Analytics: does not need to block anything -->
<script src="https://analytics.example/tag.js" async></script>
```

Generate the hash:

```bash
curl -s https://cdn.jsdelivr.net/npm/dayjs@1.11.13/dayjs.min.js \
  | openssl dgst -sha384 -binary | openssl base64 -A
```

Load a chat widget only when needed, in Svelte 5:

```svelte
<script>
  let loaded = $state(false);

  function loadChat() {
    if (loaded) return;
    const s = document.createElement('script');
    s.src = 'https://chat.example/widget.js';
    s.async = true;
    document.head.append(s);
    loaded = true;
  }
</script>

<button onclick={loadChat}>Need help?</button>
```

| Attribute | Downloads | Runs | Order kept? |
|---|---|---|---|
| none | Blocks parsing | Immediately | Yes |
| `async` | In parallel | As soon as ready | No |
| `defer` | In parallel | After HTML parsed | Yes |

## When to use it
- **Checkout page:** keep only what is required (payment SDK). No ad tags or heatmap tools that can read card inputs. Use CSP `script-src` to block anything unexpected.
- **Marketing pages:** analytics with `async`, chat widget on interaction, consent banner first.
- **Libraries from a CDN:** pin an exact version and add SRI, or self-host the file.

## Likely questions

### What are the risks of third-party scripts?
They run with the same rights as my own code: they can read inputs, cookies that are not HttpOnly, and localStorage, and they can call my APIs. If the vendor is compromised, attackers get that power on my site. They can also slow the page or break it if the vendor is down.

### What is SRI and when does it not work?
SRI is a hash in the `integrity` attribute; the browser refuses to execute the file if its hash does not match. It only works for files that do not change, so it fits pinned versions. It does not fit "always latest" tags like most analytics loaders, which change often and also load more scripts dynamically. For those I rely on CSP, vendor review, and keeping them off sensitive pages.

### async vs defer?
Both download without blocking HTML parsing. `async` runs as soon as it arrives, in any order, which suits independent scripts like analytics. `defer` waits until the HTML is parsed and keeps order, which suits scripts that depend on each other or on the DOM.

### How else can you isolate a widget?
Put it in a sandboxed iframe (`sandbox="allow-scripts"` without `allow-same-origin`) on another origin, and talk to it with `postMessage`, checking `event.origin`. Or run heavy third-party code in a web worker using tools like Partytown.

## Common mistakes
- Using SRI without `crossorigin="anonymous"`, so the check fails.
- Loading scripts from a URL with no version (it changes under you).
- Adding tag-manager access to the checkout page, letting marketing add any script.
- Trusting `postMessage` data without checking `event.origin`.

## Resources
- [MDN: Subresource Integrity](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity) - how to use and generate hashes
- [MDN: The script element](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/script) - async, defer, crossorigin
- [web.dev: Efficiently load third-party JavaScript](https://web.dev/articles/efficiently-load-third-party-javascript) - performance techniques
- [OWASP: Third Party JavaScript Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Third_Party_Javascript_Management_Cheat_Sheet.html) - security controls
