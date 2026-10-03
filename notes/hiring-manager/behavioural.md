# Behavioural

> **In one line:** "I answer behavioural questions with STAR: a short situation, my task, the specific actions I took, and a result with a number and a lesson."

## What the interviewer is really checking

- **Self-awareness:** can you admit mistakes and show what you learned?
- **Collaboration:** do you disagree respectfully and commit to the team decision?
- **Judgement under pressure:** what do you cut and what do you protect when time is short?
- **Motivation:** why you are leaving and why this company. They want "running towards something", not "running away".

## STAR framework

| Part | What to say | Share of time |
|---|---|---|
| Situation | Context in 1-2 lines | 10% |
| Task | Your responsibility | 10% |
| Action | What YOU did, 2-3 concrete steps | 60% |
| Result | Outcome with a number, plus a lesson | 20% |

Prepare 5-6 stories and reuse them for many questions. One story can show ownership, conflict, and learning.

## The questions, with a template and a generic example each

### Tell me about a disagreement with a teammate or manager.
```text
[fill in: topic, their view, my view, how we resolved it (data/prototype/call), outcome, what I learned]
```
> EXAMPLE: "A teammate wanted a global store for all server data; I preferred per-page loading. Instead of arguing in comments, I built two small prototypes and we compared stale-data bugs and code size. We picked per-page loading for most data and a store only for live prices. I learned to bring a prototype instead of opinions."

### Tell me about a tight deadline.
```text
[fill in: deadline and why fixed, how I split must-have vs nice-to-have, what I told stakeholders, result, what I would do differently]
```
> EXAMPLE: "A regulatory change had a fixed date. I listed must-haves with Product, cut two nice-to-haves to phase 2, shipped behind a flag, and kept tests on the core logic. We went live a day early and shipped phase 2 a week later."

### Tell me about a mistake you made.
```text
[fill in: the mistake (real, not fake-humble), impact, how I fixed it fast, the system change so it cannot repeat, personal habit change]
```
> EXAMPLE: "I merged a change that broke a form on an older browser because I only tested on my laptop. I rolled it back within [X] minutes, fixed it, and added that browser to our test matrix and a lint rule for unsupported APIs. Now I check our browser targets before using a new API."

### Tell me about a time you learned something fast.
```text
[fill in: what I had to learn, why fast, how (docs, small prototype, asking experts), what I shipped, how long]
```
> EXAMPLE: "I had to work on a codebase in a framework I had not used. I did the official tutorial in a day, built a tiny clone of one screen, and asked for early review on my first PR. I shipped my first feature in [X] weeks." (For Svelte, mention the official tutorial and Svelte 5 runes.)

### Why are you leaving your current company?
Keep it positive and forward-looking. Never criticise your current employer.
```text
[fill in: what I learned there] + [fill in: what I want next: e.g. a product used daily by millions, real-time UI challenges, larger ownership] + [fill in: why that is not available in my current role]
```
> EXAMPLE: "I have learned a lot about payments and reliability. Now I want to work on a real-time, high-traffic consumer product where frontend performance directly affects users' money decisions, and to grow into owning larger areas."

### Why this company?
Be specific to the product and team, not "it is a big brand".
```text
[fill in: the product and why I care (I use it? the scale?)] + [fill in: the tech: Svelte, real-time data, WebView + PWA] + [fill in: how my experience fits: payments, reliability, performance]
```
> EXAMPLE: "Trading apps are one of the hardest frontend problems: live data, low latency, trust, and huge spikes at market open. The team uses Svelte, which I enjoy, and my background in money flows where correctness matters maps directly to order placement."

## Other common ones (prepare a story ID for each)

- Time you received hard feedback: [fill in]
- Time you helped a teammate: [fill in] (see the Mentoring note)
- Time you took initiative: [fill in] (see the Ownership note)
- Time you failed to meet a deadline: [fill in]
- Biggest achievement: [fill in] (see the Project deep dive note)

## Common mistakes

- Saying "we" for all actions.
- Fake weaknesses ("I work too hard").
- Bad-mouthing your current company or manager.
- Stories with no result or lesson.
- Rambling past 2-3 minutes. Stop and let them ask.
- Mentioning confidential names or numbers. Keep it generic.

## Resources

- [Svelte tutorial](https://svelte.dev/tutorial) - quick, concrete proof for "learning fast" stories about Svelte
- [Svelte 5 migration guide](https://svelte.dev/docs/svelte/v5-migration-guide) - useful context if you talk about adapting to Svelte 5
- [web.dev: Web Vitals](https://web.dev/articles/vitals) - the vocabulary for result numbers in performance stories
