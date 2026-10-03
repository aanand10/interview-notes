# Your questions for them

> **In one line:** "Ask 2-3 sharp questions that show you already think like a member of the team: about their Svelte migration, their WebView and PWA setup, how they watch performance, and how the team works."

## What the interviewer is really checking

- Curiosity and preparation: did you think about their product and stack?
- Priorities: do you care about engineering quality, users, and growth, or only perks?
- Your questions also help YOU decide if the team is right for you.

## Key points

- Prepare 8-10, ask 2-3 (more if time). Pick ones that were not already answered.
- Ask open "how" questions, not yes/no.
- Listen and follow up on the answer; that is where the real conversation happens.
- Do not ask about salary or leave in a hiring manager round unless they bring it up.
- Hiring manager round: lean towards team, expectations, and growth. Tech round: lean towards stack and architecture.

## Tech stack and architecture

### Svelte 4 to 5 migration
- "Where are you on the Svelte 5 migration? Are new components written with runes while old ones stay on Svelte 4 syntax?"
- "What was hardest in the migration so far: stores vs runes, third-party libraries, or testing?"
- "Do you have shared guidelines for when to use `$state` in a `.svelte.ts` module versus stores?"

Why it is smart: Svelte 5 supports mixing old and new components during migration (see the [migration guide](https://svelte.dev/docs/svelte/v5-migration-guide)), so most large apps do it in steps. Shows you know the real-world path.

### WebView vs PWA split
- "Which parts of the product run inside the native apps' WebView and which are served as a PWA or web app? How do you share code between them?"
- "How do the WebView pages talk to the native app: a JS bridge, deep links, or postMessage?"
- "What is your minimum supported WebView or browser version, and how do you test on low-end Android devices?"

Background: a [WebView](https://developer.android.com/develop/ui/views/layout/webapps/webview) shows web content inside a native app; a [PWA](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps) is a web app that can be installed and work offline with a service worker.

### Performance monitoring
- "How do you monitor frontend performance in production? Do you track Core Web Vitals from real users, and per device tier?"
- "Do you have performance budgets (bundle size, INP) enforced in CI?"
- "How do you handle the traffic spike at market open: for example WebSocket fan-out, throttled rendering, or degraded modes?"
- "How do you measure price-update latency from server to screen?"

### Reliability and release
- "How do releases work: how often, feature flags, staged rollout?"
- "What did your last frontend incident look like, and what changed after the RCA?"

## Team structure and ways of working

- "How is the web team structured: by product area (orders, portfolio, onboarding) or by platform?"
- "How do frontend, backend, design and QA work together on a feature? Who writes the design doc?"
- "What is the ratio of new features to tech debt and platform work?"
- "Is there a shared design system, and who owns it?"

## Expectations and growth

- "What would a great first 90 days look like for this role?"
- "What separates an SDE 2 who does well here from one who struggles?"
- "What is the biggest challenge the web team faces in the next 6-12 months?"
- "How are engineers given ownership: do SDE 2s lead projects or RCAs end to end?"
- "How does the path from SDE 2 to senior look here?"

## Template: your shortlist for the day

```text
Hiring manager round:
1. [fill in: pick from Expectations]
2. [fill in: pick from Team structure]
3. [fill in: a follow-up on something they said earlier]
Tech round:
1. [fill in: Svelte 5 migration]
2. [fill in: WebView vs PWA]
3. [fill in: performance monitoring]
```

## Common mistakes

- "No, I have no questions." Always ask at least one.
- Asking things easily found on their website.
- Only asking about perks, remote work, or salary.
- Asking a question that sounds like a test or a criticism ("Why is your app so slow?").
- Not listening to the answer.

## Resources

- [Svelte 5 migration guide](https://svelte.dev/docs/svelte/v5-migration-guide) - background for the migration questions
- [MDN: Progressive web apps](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps) - background for WebView vs PWA questions
- [Android Developers: WebView](https://developer.android.com/develop/ui/views/layout/webapps/webview) - how web content runs inside the native app
- [web.dev: Web Vitals](https://web.dev/articles/vitals) - vocabulary for performance-monitoring questions
