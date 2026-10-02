# Payments work: integration flow, failures, timeouts, double submits

> **In one line:** "In payments the frontend's job is to never charge twice, never show a wrong status, and always give the user a clear next step, so I treat every payment as a state machine with an idempotency key and the server as the source of truth."

## What the interviewer is really checking

- Do you understand the **full flow** of a payment, not only the button?
- Can you handle the **unhappy paths**: network failure, timeout, user closes the app, double click, bank delay?
- Do you know the **safety tools**: idempotency keys, server-side verification, polling or webhooks, retries with backoff?
- Can you explain **your part** in a wallet or third-party integration clearly?
- For a trading company this maps directly to **placing orders**: same risks (double orders, unknown status), same fixes.

## Key points

- **Idempotency key:** a unique id (for example `crypto.randomUUID()`) sent with the payment request. If the same request arrives twice, the server returns the first result instead of charging again. See [MDN: Idempotent](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent).
- **Timeout does not mean failed.** The money may have moved. Show "processing", then poll the server for the final status.
- **Double submit:** disable the button while submitting, guard in code, and still rely on the idempotency key (the UI guard alone is not enough).
- **Server is the truth.** The client callback can be lost or faked; the backend confirms via the provider's webhook or a status API and verifies the signature.
- **Amounts in the smallest unit** (paise, cents) as integers, never floats.

## The typical flow (generic wallet / third-party payment)

1. User taps Pay. Frontend asks the backend to **create an order** (amount, currency, order id). The amount is decided by the server, never trusted from the client.
2. Frontend checks if the wallet or method is **available** on this device and browser (for browser wallets, the [Payment Request API](https://developer.mozilla.org/en-US/docs/Web/API/Payment_Request_API) has `canMakePayment()`). Hide the button if not.
3. Frontend opens the wallet sheet or redirects to the app (for UPI-style apps, an intent or deep link on mobile, or a collect request on desktop).
4. User approves. The wallet returns a **token** or a result to the frontend.
5. Frontend sends the token to the backend. Backend calls the payment provider and **verifies** the result.
6. Backend also gets a **webhook** from the provider. This is the final word.
7. Frontend shows success, failure, or "processing" and polls until final.

## Example

```svelte
<script lang="ts">
  let { orderId }: { orderId: string } = $props();

  type Status = 'idle' | 'submitting' | 'pending' | 'success' | 'failed';
  let status = $state<Status>('idle');
  // One key per payment attempt. Reused on retry, so the server never charges twice.
  const idempotencyKey = crypto.randomUUID();

  async function pay() {
    if (status === 'submitting' || status === 'pending') return; // code guard, not only disabled
    status = 'submitting';
    try {
      const res = await fetch('/api/payments', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'Idempotency-Key': idempotencyKey },
        body: JSON.stringify({ orderId }), // amount comes from the server-side order
        signal: AbortSignal.timeout(15_000), // throws a TimeoutError after 15s
      });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      const { state } = await res.json(); // 'captured' | 'pending' | 'failed'
      status = state === 'captured' ? 'success' : state === 'failed' ? 'failed' : 'pending';
    } catch (err) {
      // Timeout or network drop: we do NOT know the result. Never show "failed" here.
      status = (err as Error).name === 'TimeoutError' ? 'pending' : 'failed';
    }
  }
</script>

<button onclick={pay} disabled={status === 'submitting' || status === 'pending'} aria-busy={status === 'submitting'}>
  {status === 'submitting' ? 'Processing...' : 'Pay'}
</button>
<p role="status">
  {#if status === 'pending'}Payment is processing. Do not pay again; we will update you.{/if}
  {#if status === 'failed'}Payment failed. No money was taken. Try again.{/if}
</p>
```

Polling the final status with backoff (verified with Node):

```js
async function pollPaymentStatus(paymentId, { getStatus, maxWaitMs = 60_000 } = {}) {
  const start = Date.now();
  let delay = 1000;
  while (Date.now() - start < maxWaitMs) {
    const status = await getStatus(paymentId); // 'pending' | 'captured' | 'failed'
    if (status !== 'pending') return status;
    await new Promise((r) => setTimeout(r, delay));
    delay = Math.min(delay * 2, 8000); // 1s, 2s, 4s, 8s, 8s...
  }
  return 'unknown'; // show "we will notify you", never "failed"
}
// With a fake server that returns pending, pending, captured -> prints "captured" after 3 calls
```

## When to use it

Any flow where repeating an action costs money: checkout, adding funds, **placing a buy or sell order**, withdrawing. In a trading app, a timed-out order request must show "order status unknown, checking", then reconcile with the order book, never auto-resubmit.

## Answer framework

Problem, constraints, options, decision, implementation, impact, improve. For payment stories, constraints are usually: compliance (no card data on your servers), third-party SDK limits, many devices and browsers, and "zero double charges".

## Fill-in template

```text
Integration: [fill in: generic name, e.g. "a wallet payment method on checkout"]
The flow I built: [fill in: steps 1-7 above, in your words]
My part vs team: [fill in: e.g. "I built the frontend flow and availability check; backend team owned token verification"]
Hardest failure case: [fill in: timeout / app switch / double tap / bank delay]
How I handled it: [fill in: state machine, idempotency, polling, copy for users]
How we tested: [fill in: sandbox, real devices, failure injection]
Impact: [fill in: success rate, adoption, drop in support tickets]
What I would improve: [fill in]
```

## Example answer (generic, for illustration only)

> EXAMPLE, not a real story.

"I built the frontend for a new wallet payment method on checkout. **My part** was the availability check, the button and sheet, the result handling, and the error states. The backend team owned verification with the provider. **The hardest part** was the user switching to the wallet app and coming back: sometimes the page was reloaded and the in-memory state was lost. I stored the pending payment id in session storage, and on load the page checked the server for the status of that id. For **timeouts** we showed 'processing' and polled with backoff. For **double submits** we disabled the button and sent an idempotency key. **Impact:** [X]% of eligible users chose the new method in the first month and the 'paid twice' support tickets went to near zero."

## Likely questions

### Walk me through a wallet or third-party payment integration flow.
Use the 7 steps above. Stress three things: the server creates the order and amount, the client only collects a token, and the backend verifies and trusts the webhook. Mention the availability check so you never show a method that cannot work.

### What exactly was your part?
Say the pieces you owned in concrete words: "the availability check, the UI states, the retry and polling logic, and the analytics events." Name what other teams owned. This is about honesty, not size.

### How do you handle a payment timeout?
A timeout means the result is unknown. Show a "processing" state, do not let the user pay again, and poll the status API with exponential backoff. If still unknown after a limit, tell the user we will notify them, and let the backend reconcile with the webhook. Never show "failed" unless the server says failed.

### How do you prevent double submits?
Three layers. UI: disable the button and show a spinner. Code: a guard that returns early if a request is in flight. Server: an idempotency key so even two real requests create only one charge. The server layer is the one that actually guarantees safety.

### What if the user refreshes or closes the tab mid-payment?
Save the pending payment or order id (session storage or URL). On return, ask the server for its status instead of starting fresh. Optionally warn with a [`beforeunload`](https://developer.mozilla.org/en-US/docs/Web/API/Window/beforeunload_event) prompt while a payment is in flight.

### Should you retry failed requests automatically?
Retry only when it is safe: idempotent requests, network errors, or 5xx and 429 (respect [`Retry-After`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After)). Use the same idempotency key, a small retry limit, and backoff with jitter. Never retry a 4xx like "insufficient funds".

### How does this apply to placing a trade order?
Same pattern: client order id as the idempotency key, a clear "order pending" state, reconcile with the live order status (often over a WebSocket), and never auto-resubmit on timeout because a duplicate order is real money.

## Common mistakes

- Showing "Payment failed" on a timeout, so the user pays again.
- Only disabling the button and calling it "double-submit protection".
- Generating a new idempotency key on every retry (defeats the purpose).
- Trusting the amount or success status sent from the browser.
- Using floats for money (`0.1 + 0.2 !== 0.3`).
- Logging card numbers, tokens, or personal data in analytics or error tools.

## Resources

- [MDN: Payment Request API](https://developer.mozilla.org/en-US/docs/Web/API/Payment_Request_API) - browser-native wallet and payment sheet
- [MDN: AbortSignal.timeout()](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static) - clean request timeouts
- [MDN: Idempotent](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent) - why repeating a request must be safe
- [MDN: crypto.randomUUID()](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID) - generating idempotency keys
