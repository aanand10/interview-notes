# Project deep dive

> **In one line:** "Let me walk you through [fill in: project] in the order problem, constraints, options, decision, build, impact, and what I would do differently, and I will be clear about which parts were mine."

## What the interviewer is really checking

- **Depth:** do you understand the system below the surface, or did you only touch one screen?
- **Ownership:** can you separate "I" from "we" honestly? SDE 2 means you drove at least one meaningful piece.
- **Judgement:** did you weigh options and trade-offs, or just do the first thing?
- **Impact:** can you tie the work to a number (conversion, latency, errors, dev time)?
- **Reflection:** do you know what you would improve? This shows growth, not weakness.

## Key points

- Pick ONE project you can defend for 20 minutes of follow-up questions. Big and vague is worse than medium and deep.
- Use the 7-step framework below. Spend about 30% of the time on the decision and implementation.
- Say "I" for your work and "the team" for shared work. Never claim the whole thing.
- Prepare a simple whiteboard sketch: boxes for UI, state, API, third parties.
- Have numbers ready, and know how each number was measured.

## Answer framework

| Step | What to say | Time |
|---|---|---|
| 1. Problem | Who was hurting and why it mattered to the business | 20 s |
| 2. Constraints | Deadline, team size, legacy code, compliance, devices, browsers | 20 s |
| 3. Options considered | 2 to 3 real options with a pro and con each | 40 s |
| 4. Decision | What you picked and the main reason | 20 s |
| 5. Implementation | Architecture, the hardest part, how you tested and rolled out | 60 s |
| 6. Impact | Metrics before and after, plus a qualitative win | 20 s |
| 7. What I would improve | One honest technical change and one process change | 20 s |

Keep the first pass to about 3 minutes. Then let the interviewer pull on threads.

## Fill-in template

```text
Project: [fill in: name it generically, e.g. "the checkout page rewrite"]
Problem: [fill in: user/business pain, ideally with a number]
Constraints: [fill in: deadline, team size, tech limits, compliance]
Options:
  A) [fill in] - pro: [fill in] / con: [fill in]
  B) [fill in] - pro: [fill in] / con: [fill in]
  C) [fill in: "do nothing" or a smaller fix is a fine option]
Decision: [fill in: chosen option + main reason]
My part: [fill in: 2-3 concrete things only you did]
Team's part: [fill in: what others built, who you worked with]
Hardest technical decision: [fill in: the trade-off, what you gave up]
Implementation details: [fill in: state model, API contract, testing, rollout plan]
Impact: [fill in: metric before -> after, how measured, over what time]
What I would improve: [fill in: tech change] + [fill in: process change]
```

## Example answer (generic, for illustration only)

> EXAMPLE, not a real story. Replace with your own.

"**Problem:** our checkout page was a large legacy bundle. It loaded slowly on low-end Android phones and about [X]% of users dropped before paying.
**Constraints:** two frontend engineers, one quarter, no downtime allowed, and the page was embedded in many merchant sites, so we could not break their integrations.
**Options:** a full rewrite in a new framework, an incremental rewrite screen by screen, or only optimising the existing bundle. A full rewrite was too risky for the timeline. Pure optimisation would not fix the messy state logic.
**Decision:** incremental rewrite behind a feature flag, starting with the payment-method screen, which had the most traffic.
**Implementation:** I designed the state model as a small state machine (idle, submitting, pending, success, failed), wrote the shared form components, and set up the A/B rollout. A teammate owned the API changes. We rolled out at 1%, 10%, 50%, 100%, watching error rate and conversion at each step.
**Impact:** bundle size dropped by [X]%, LCP improved from [X] to [Y] seconds at p75, and conversion went up [X] points.
**Improve:** I would add end-to-end tests earlier; we found two regressions late. And I would write the design doc before coding, not halfway through."

## Likely questions

### Walk me through your biggest project end to end.
Use the 7 steps. Open with the problem and why it mattered, not with the tech stack. Draw the architecture if there is a whiteboard. Finish with impact and one improvement. Stop at about 3 minutes and ask, "Which part would you like me to go deeper on?"

### What was your individual contribution versus the team's?
Be specific and honest: "I owned X and Y end to end. Z was built by a teammate, and I reviewed it. The API was the backend team's; I wrote the contract with them." Interviewers often probe this twice. If your answer changes, it hurts trust. A good SDE 2 signal is "I led the design, split the work, and unblocked others."

### What was the hardest technical decision?
Pick a real trade-off, not "which library to use". Good examples: consistency vs speed, rewrite vs refactor, client-side vs server-side state, build vs buy. Say what you gave up and how you reduced the risk (feature flag, fallback, monitoring). End with whether it turned out right and what data told you so.

### If you had to do it again, what would you change?
Name one technical thing and one process thing. Show that you learned something specific, for example "I would define success metrics and dashboards before launch, not after."

### How did you test and roll it out?
Unit tests for logic, component tests for UI states, a few end-to-end tests for the money path, then a staged rollout behind a feature flag with a kill switch. Mention what you watched during rollout (errors, latency, conversion).

### What would break first if traffic went up 10 times?
Shows system thinking. Talk about API rate limits, cache hit rate, bundle size on slow networks, and third-party SDK limits. Say how you would detect it (dashboards, alerts) before users report it.

## Common mistakes

- Starting with "We used React/Svelte, Redux, Tailwind..." instead of the problem.
- Saying "we" for everything, so the interviewer cannot see your part.
- No numbers, or numbers you cannot explain ("how did you measure 30%?").
- Picking a project so big you only know your corner of it.
- Hiding failures. A small honest mistake with a lesson is a strong signal.
- Talking for 10 minutes without a check-in.
- Naming confidential details (internal tool names, client names, revenue). Keep it generic.

## Resources

- [web.dev: User-centric performance metrics](https://web.dev/articles/user-centric-performance-metrics) - vocabulary for talking about impact numbers
- [web.dev: Web Vitals](https://web.dev/articles/vitals) - standard metrics to quote (LCP, INP, CLS)
- [MDN: Performance API](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API) - how numbers like render time are actually measured
