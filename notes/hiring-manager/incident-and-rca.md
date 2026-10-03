# Incident and RCA

> **In one line:** "In an incident I first stop the bleeding, then find the root cause, then make sure it cannot happen the same way again, and I run the RCA blameless so we fix the system, not blame a person."

## What the interviewer is really checking

- **Calm under pressure:** do you mitigate first (rollback, flag off) before debugging?
- **Clear thinking:** can you separate detection, impact, mitigation, root cause, fix, and prevention?
- **Leadership:** the job asks you to **lead RCA** (Root Cause Analysis). Can you run the meeting, write the doc, and drive action items to done?
- **Honesty:** do you own your part without blaming others?

## Key points

- **Mitigate before you fix.** Rolling back or turning a feature flag off is usually faster than a hotfix.
- **Communicate early:** an incident channel, one owner (incident commander), regular status updates to support and stakeholders.
- **Root cause is rarely "a developer made a typo".** Ask why the system allowed it: missing test, missing alert, risky deploy process.
- **Blameless postmortem:** focus on what happened and why, not who. People share the truth only when they feel safe.
- **Action items** have owners and dates, and are tracked to done. An RCA without follow-through is just a story.

## Incident flow

| Phase | What I do |
|---|---|
| Detect | Alert fires (error rate, success rate), or support/users report it. Note the time. |
| Triage | How many users, which platforms, money at risk? Set severity (SEV1 to SEV3). |
| Mitigate | Flag off, rollback, failover, or a banner telling users. Goal: stop the impact. |
| Communicate | Incident channel, status updates every [fill in: 15-30] min, inform support. |
| Root cause | Logs, Sentry, dashboards, diff between releases, reproduce. |
| Fix | Proper fix with a test that would have caught it. |
| Prevent | Postmortem, action items: tests, alerts, runbooks, process changes. |

Key times to record: **time to detect** (TTD) and **time to mitigate/resolve** (MTTR, mean time to recovery). Good RCAs ask how to shrink both.

## The 5 Whys technique

Keep asking "why" until you reach a cause you can fix in the system.

```text
Problem: users were charged but saw "payment failed".
1. Why? The UI showed failure when the status API timed out.
2. Why? The client treated timeout as failure instead of "unknown".
3. Why? The state model had only success and failure, no "pending" state.
4. Why? The design did not cover slow bank responses; tests mocked only fast responses.
5. Why? We had no checklist for failure modes on money flows.
Root causes: missing "pending" state + no failure-mode review.
Actions: add pending + polling/webhook reconciliation, add timeout tests, add a failure-mode checklist to design docs.
```

## Example: the code fix for the incident above

```ts
type PaymentStatus = 'idle' | 'submitting' | 'pending' | 'success' | 'failed';

async function confirmPayment(id: string): Promise<PaymentStatus> {
  const ctrl = new AbortController();
  const timer = setTimeout(() => ctrl.abort(), 10_000);
  try {
    const res = await fetch(`/api/payments/${id}`, { signal: ctrl.signal });
    if (!res.ok) return 'pending'; // server trouble: do not claim failure
    const { status } = await res.json();
    return status; // 'success' | 'failed' | 'pending' from the source of truth
  } catch {
    // timeout or network error: the outcome is UNKNOWN, not failed
    return 'pending'; // UI shows "confirming..." and keeps polling the status
  } finally {
    clearTimeout(timer);
  }
}
```

## Postmortem doc template (blameless)

```text
Title, date, severity, author
Summary: 2-3 lines, what users saw
Impact: users affected, duration, money/orders affected
Timeline: detection -> mitigation -> resolution, with times
Root cause: 5 Whys result
What went well / what went badly / where we got lucky
Action items: [action] - [owner] - [due date] - [ticket]
```

## Fill-in template (your story)

```text
Incident: [fill in: generic description, no company or product names]
Detection: [fill in: alert / user report / you noticed; time to detect]
Impact: [fill in: % users, duration, business effect]
Mitigation: [fill in: rollback / flag off / hotfix; time to mitigate]
Root cause: [fill in: the 5 Whys chain in one line]
Fix: [fill in: code + test]
Prevention: [fill in: alert, test, process change you drove]
My role: [fill in: did you detect, lead, write the RCA, own actions?]
What I learned: [fill in]
```

## Example answer (generic, for illustration only)

> EXAMPLE, not a real story. Replace with your own.

"After a release, our success-rate alert fired: about [X]% of users on one browser could not submit the checkout form. I was on call. Within 10 minutes I confirmed in Sentry that all errors came from the new release, and we turned the feature flag off; the success rate recovered. Root cause: a new date API we used was not supported in an older WebView, and our test matrix did not include that version. I fixed it with a fallback and a test, and I led the RCA: we added that WebView to the device matrix, added a lint rule for unsupported APIs based on our browser targets, and added a per-browser breakdown to the alert. Detection took 8 minutes, mitigation 10 minutes."

## Likely questions

### Tell me about a production incident you handled.
Use the flow: detection, impact, mitigation, root cause, fix, prevention, with times and numbers. Be clear about your role. Spend most of the time on mitigation and prevention.

### Rollback or hotfix?
Rollback (or flag off) first if it is safe, because it is fast and well tested. Hotfix forward only when rollback is not possible, for example a database migration already ran, or the old version is also broken.

### How do you lead an RCA meeting?
Share the timeline draft before the meeting. In the meeting, walk the timeline, run 5 Whys, keep it blameless ("the deploy process allowed..." not "X pushed..."), and agree on action items with owners and dates. After it, I track the items in our tracker and report when they are done.

### How do you prevent the same incident from happening again?
Add the test that would have caught it, an alert that would have detected it faster, and fix the process gap (checklist, review rule, staged rollout). Update the runbook.

### What if the root cause was your own mistake?
Say it plainly, show the mitigation and the system fix, and what you changed in your own habits. Owning it is a strong signal.

## Common mistakes

- Debugging for an hour while users are still affected.
- "Root cause: human error" with no system fix.
- No numbers (how many users, how long).
- Blaming another team or a colleague.
- Action items with no owner, never finished.

## Resources

- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) - timeouts on fetch, common in payment incidents
- [web.dev: Web Vitals](https://web.dev/articles/vitals) - performance regressions are incidents too
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) - logging that helps investigations without leaking data
