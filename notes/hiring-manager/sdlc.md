# SDLC

> **In one line:** "For me a feature goes requirement, design, build, review, test, release behind a flag, monitor, and iterate, and at each step I check one question: is this still solving the user's problem safely?"

## What the interviewer is really checking

- **Process maturity:** do you know the full path to production, or only "I write code and someone deploys it"?
- **Risk thinking:** in a trading or payments app a bad release can cost real money. Do you ship safely (flags, staged rollout, rollback)?
- **Ownership after merge:** do you watch the feature in production, or stop caring when the PR is merged?
- **Collaboration:** where do Product, Design, QA and Backend come in, and how early?

## Key points

- SDLC means **Software Development Life Cycle**: the steps a feature goes through from idea to production and back.
- Make the "left" side strong: clear requirements and a short design doc remove most bugs before code exists.
- Make the "right" side safe: feature flags, staged rollout, monitoring, and a fast rollback.
- Each stage has an **exit check** (a "definition of done"), not just a calendar date.
- Close the loop: production data and user feedback decide the next iteration.

## The stages, with what I actually do

| Stage | What happens | My exit check |
|---|---|---|
| 1. Requirement | PRD (product requirement doc) from Product, designs from Design | Acceptance criteria are written, edge cases listed, success metric agreed |
| 2. Design | Short tech design doc: components, state, API contract, error states, rollout plan | Reviewed by a peer and backend; API shape agreed |
| 3. Development | Small PRs, behind a feature flag, tests with the code | CI green: lint, type check, unit tests, build |
| 4. Code review | 1-2 reviewers, focus on correctness, edge cases, a11y, security | Approved, all comments resolved |
| 5. Testing | Unit + component tests, a few end-to-end tests on the money path, QA on real devices | QA sign-off, no P0/P1 bugs open |
| 6. Deployment | Merge to main, deploy to staging, then production with the flag off; turn flag on 1% -> 10% -> 50% -> 100% | Error rate and key metric stable at each step |
| 7. Monitoring | Sentry errors, RUM dashboards (Web Vitals), funnel metrics, alerts | No regression for [fill in: e.g. 48 hours] at 100% |
| 8. Iteration | Read data and feedback, fix papercuts, remove the flag and dead code | Flag cleaned up, follow-up tickets created |

Terms used above:
- **Feature flag:** a runtime switch (for example from LaunchDarkly, Unleash, or an in-house config service) that turns code on or off without a new deploy. It is also your **kill switch**.
- **Staged (canary) rollout:** release to a small percent of users first, watch, then widen.
- **RUM (Real User Monitoring):** performance and errors measured on real users' devices, not in the lab.

## Example: a feature flag guard in Svelte 5

```svelte
<script lang="ts">
  // flags come from a config service, loaded once at app start
  import { flags } from '$lib/flags.svelte';
  import NewOrderForm from './NewOrderForm.svelte';
  import OldOrderForm from './OldOrderForm.svelte';

  // $derived re-computes if the flag value changes at runtime (kill switch)
  let useNewForm = $derived(flags.value['new-order-form'] === true);
</script>

{#if useNewForm}
  <NewOrderForm />
{:else}
  <OldOrderForm />
{/if}
```

Why it matters: if errors spike after release, we switch the flag off in seconds. No rollback deploy, no app store review for a WebView app.

## A simple CI pipeline (what "CI green" means)

```bash
# runs on every pull request
npm ci                 # clean install from lockfile
npm run lint           # ESLint + Prettier check
npm run check          # svelte-check / tsc type check
npm run test -- --run  # Vitest unit + component tests
npm run build          # production build must succeed
npx playwright test    # a few E2E tests on critical flows (login, place order, pay)
```

## When to use it

Every feature, but scale the process to the risk. A copy change needs a PR and a review. A change to the order placement or payment flow needs a design doc, E2E tests, a flag, a staged rollout, and a dashboard ready before launch.

## Fill-in template

```text
Feature: [fill in: generic name, e.g. "new payment method selector"]
Requirement: [fill in: who asked, the user problem, success metric]
Design: [fill in: did you write a design doc? what did it cover? who reviewed?]
Build: [fill in: how you split PRs, flag name style, tests you wrote]
Review: [fill in: who reviewed, one thing review caught]
Testing: [fill in: unit / E2E / QA devices]
Release: [fill in: rollout steps and what you watched]
Monitoring: [fill in: dashboard, alerts, how long you watched]
Iteration: [fill in: what changed after launch based on data]
```

## Example answer (generic, for illustration only)

> EXAMPLE, not a real story. Replace with your own.

"Take adding a new payment method to checkout. Product shared a PRD; I first listed edge cases they had missed, like what happens on timeout or if the user presses back mid-payment, and we added them as acceptance criteria. I wrote a one-page design doc: a small state machine for the payment status, the API contract with backend, and the rollout plan. I built it in four small PRs behind a flag. Each PR had unit tests, and I added one Playwright test for the happy path and one for the failure path. QA tested on low-end Android and iOS Safari. We released at 1%, watched Sentry and the success-rate dashboard for a day, then went to 10, 50 and 100%. After two weeks I removed the flag and the old code."

## Likely questions

### Walk me through how a feature goes from requirement to production in your team.
Use the table above in your own words, about 90 seconds. Mention at least: acceptance criteria, design doc, small PRs, CI, review, QA, feature flag, staged rollout, dashboards. End with "and I keep watching it after release".

### How do you handle a requirement that is unclear?
I write down my understanding as acceptance criteria, list open questions, and get Product to confirm in writing before I build. For small unknowns I build the clear part first and keep the unclear part behind a flag.

### How do you make sure you do not break production?
Several layers: types and tests in CI, careful review, QA on real devices, feature flags, staged rollout, and alerts on error rate and key business metrics. And a rollback plan written before release, not during the incident.

### How do you decide what to test?
Test the logic that can lose money or trust most: price calculation, order and payment states, validation. Unit-test pure logic, component-test UI states (loading, error, empty), and keep E2E tests few and focused on critical journeys, because they are slow and flaky.

### What is the difference between deploy and release?
Deploy means the code is on the servers. Release means users can see it. Feature flags separate the two, so we can deploy any time and release when ready.

### What do you do after a feature is at 100%?
Watch it for a few days, compare the success metric to the baseline, clean up the flag and old code, and write follow-up tickets for anything we cut.

## Common mistakes

- Describing only the coding step.
- No mention of rollback or flags in a fintech context.
- Saying "QA will catch it". The engineer owns quality, QA is a second net.
- Leaving feature flags in the code forever (flag debt).
- Big-bang PRs of 2,000 lines that nobody can review well.

## Resources

- [web.dev: Web Vitals](https://web.dev/articles/vitals) - metrics to watch after release
- [Playwright: Getting started](https://playwright.dev/docs/intro) - E2E tests for critical flows
- [Vitest guide](https://vitest.dev/guide/) - unit and component tests in a Svelte/Vite project
- [OWASP Top Ten](https://owasp.org/www-project-top-ten/) - security checks to keep in the review and test stages
