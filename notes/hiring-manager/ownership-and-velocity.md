# Ownership and velocity

> **In one line:** "Ownership for me means I take a problem from 'unclear idea' to 'working in production and measured', and I keep speed safe with small PRs, feature flags, staged rollouts, and good tests on the risky parts."

## What the interviewer is really checking

- The company stresses "own it, drive it". Do you act without being chased?
- Did you own something **end to end**: requirements, design, build, release, monitoring, follow-up?
- Can you move fast **without** breaking production? Speed with no safety is a red flag in trading and payments.
- Do you unblock yourself and others (chase dependencies, make decisions, escalate early)?

## Key points

- **End to end** means beyond your code: you chase the backend contract, align Design, plan the rollout, watch the dashboards, and clean up.
- **Velocity comes from small batches:** small PRs, frequent merges, flags so unfinished work can ship dark (deployed but hidden).
- **Safety nets let you go fast:** types, tests on money logic, CI, staged rollout, kill switch, alerts.
- **Reduce waiting:** agree on API contracts early and build against a mock (for example MSW, Mock Service Worker, or a typed fake) while backend is building.
- **Escalate early:** "this is at risk, here are options" on day 2 is ownership; a surprise on day 10 is not.

## Example: building against an agreed API contract (no waiting on backend)

```ts
// shared contract agreed with backend on day 1
export interface PlaceOrderRequest {
  symbol: string;
  side: 'BUY' | 'SELL';
  qty: number;
  type: 'MARKET' | 'LIMIT';
  limitPrice?: number;
  idempotencyKey: string; // server dedupes retries / double taps
}
export interface PlaceOrderResponse { orderId: string; status: 'PENDING' | 'REJECTED' }

// a fake used in dev and tests until the real endpoint is ready
export async function placeOrderFake(req: PlaceOrderRequest): Promise<PlaceOrderResponse> {
  await new Promise((r) => setTimeout(r, 300)); // simulate latency
  if (req.qty <= 0) return { orderId: '', status: 'REJECTED' };
  return { orderId: crypto.randomUUID(), status: 'PENDING' };
}
```

## Fill-in template

```text
What I owned end to end: [fill in: generic name]
Why it was mine: [fill in: assigned / I volunteered / I spotted the problem]
Hard parts I drove beyond code: [fill in: dependency, alignment, decision]
How I kept it fast: [fill in: small PRs, flags, mocks, parallel work]
How I kept it safe: [fill in: tests, rollout, monitoring]
Result: [fill in: shipped date vs plan, metric]
After launch: [fill in: what you watched, fixed, cleaned up]
```

## Example answer (generic, for illustration only)

> EXAMPLE, not a real story. Replace with your own.

"I noticed our support team got many tickets about failed payments where money was actually deducted. Nobody owned it, so I took it. I dug into the logs with backend, found the client showed 'failed' on timeouts, and wrote a short proposal for a pending state with status polling. I agreed the status API contract with backend, built the UI against a fake, and shipped in small PRs behind a flag. We rolled out 5% -> 100% over a week while I watched the dashboard. Related support tickets dropped by about [X]%, and I wrote a runbook for the support team."

## Likely questions

### Tell me about something you owned end to end.
Use the 7-step project framework (problem, constraints, options, decision, implementation, impact, improve). Highlight the non-coding parts you drove.

### How do you ship fast without breaking things?
Small PRs, trunk-based merges behind flags, CI with types and tests, tests focused on money logic, staged rollout with alerts, and a kill switch. Speed comes from removing waiting and big-bang releases, not from skipping checks.

### What do you do when you are blocked by another team?
Unblock what I can (mock the API, build the UI states), raise it early with a clear ask and a date, offer help (draft the contract, write the PR for their side if allowed), and escalate to managers with options if the date is at risk.

### Tell me about a time you went beyond your role.
A gap nobody owned (flaky tests, missing alert, bad onboarding) that you fixed, with the result.

### When is it OK to cut corners?
Prototypes and experiments behind a flag, with the debt written down as a ticket. Never on money paths, security, or accessibility basics.

## Common mistakes

- "Ownership" stories that are just "I finished my ticket".
- Speed stories that ended in an incident without a lesson.
- Hero stories (working nights alone) instead of good process.
- Not mentioning what happened after launch.

## Resources

- [MDN: Crypto.randomUUID()](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID) - generating idempotency keys on the client
- [Playwright: Getting started](https://playwright.dev/docs/intro) - E2E safety net for fast releases
- [web.dev: Web Vitals](https://web.dev/articles/vitals) - metrics to watch during a rollout
