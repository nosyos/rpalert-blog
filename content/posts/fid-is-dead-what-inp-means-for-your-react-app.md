---
title: FID Is Dead. What INP Means for Your React App
date: 2026-05-05
tags:
  - webdev
  - react
  - performance
  - javascript
slug: fid-is-dead-what-inp-means-for-your-react-app
description: "Google replaced FID with INP as a Core Web Vital. Most React apps that passed before are now failing — here's why and what to fix."
series: React Core Web Vitals in Depth
draft: false
---

In March 2024, Google replaced First Input Delay with Interaction to Next Paint as an official Core Web Vital. FID is gone. INP is what matters now — and most React apps that were passing before are failing under the new standard without anyone realizing it.

---

## What was wrong with FID

FID measured how long the browser took to respond to the very first user interaction on a page. Click a button, FID measures the delay before the browser started processing that click. Just the first one. Just the start of processing, not the time until anything actually happened on screen.

In practice, FID was easy to pass and bad at catching real responsiveness problems. A page could have an excellent FID score while every subsequent click — after the page was fully loaded and the user was actually using it — took 600ms to respond. FID had nothing to say about that.

---

## What INP actually measures

INP measures the full interaction latency for all interactions throughout the entire page session, not just the first. It captures the delay from when a user interacts (click, tap, keyboard input) to when the browser finishes rendering the visual response.

The threshold: Good is under 200ms. Needs Improvement is 200–500ms. Poor is over 500ms.

The shift matters because it exposes a category of problems FID never touched. Long-running JavaScript that doesn't affect first-load responsiveness but blocks the main thread during normal usage. React state updates that trigger expensive re-renders mid-session. Event handlers that do too much synchronous work before returning control to the browser.

A React app with heavy component trees, lots of context consumers, and synchronous state updates can have a perfectly acceptable LCP and a terrible INP. Under FID, nobody would have noticed.

---

## Where React apps tend to fail INP

The most common causes in React specifically:

**Synchronous state updates that cascade.** A click handler updates state, which triggers a re-render of a large subtree, which blocks the main thread while React reconciles. If that reconciliation takes 300ms, the user sees a 300ms delay before anything on screen changes. The React Compiler helps here by reducing unnecessary re-renders, but it doesn't reduce the cost of renders that need to happen.

**Unoptimized event handlers.** An `onClick` that does validation, makes a synchronous API call via a cached store, updates multiple pieces of state, and then re-renders — all before returning — is an INP problem waiting to be found.

`useTransition` is the right tool for expensive updates that don't need to block the interaction response:

```tsx
const [isPending, startTransition] = useTransition();

function handleClick() {
  // This part runs immediately — updates the UI to acknowledge the interaction
  setButtonState('loading');

  // This part is deferred — React schedules it without blocking
  startTransition(() => {
    setFilteredResults(computeExpensiveFilter(data));
  });
}
```

The interaction response — acknowledging that something happened — is immediate. The expensive computation happens without blocking the input.

**Third-party scripts running during interactions.** An analytics script that fires on every click event and does synchronous work is adding to your INP whether you wrote it or not.

---

## The Long Animation Frames API

Alongside INP, browsers shipped the Long Animation Frames API (LoAF) as a more precise replacement for Long Tasks.

Long Tasks measured any main thread task over 50ms. LoAF measures animation frames that take over 50ms to render, with richer attribution — it tells you which scripts, which event handlers, and which rendering work contributed to the slow frame.

```typescript
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    // entry.scripts shows which scripts contributed to the slow frame
    console.log('Slow frame:', entry.duration, 'ms');
    console.log('Scripts:', entry.scripts);
  }
}).observe({ type: 'long-animation-frame', buffered: true });
```

The `scripts` array in each entry is the part that changes the debugging workflow. Instead of knowing "something ran too long on the main thread," you know exactly which function in which file was responsible. This is significantly faster to diagnose than working backward from a Long Tasks timeline in the Performance panel.

---

## Speculation Rules: prefetching gets declarative

The Speculation Rules API lets you declare prefetch and prerender rules directly in HTML, without JavaScript:

```html
<script type="speculationrules">
{
  "prerender": [
    { "where": { "href_matches": "/product/*" }, "eagerness": "moderate" }
  ]
}
</script>
```

`prerender` goes further than `prefetch` — it fully renders the page in a hidden tab, so navigation is instant. `eagerness: "moderate"` triggers prerendering when the user holds the pointer over a matching link for 200ms (or on pointerdown if that happens sooner), not immediately on page load.

For Next.js apps, the router already handles prefetching via `<Link>`, but Speculation Rules gives you declarative control for non-Next.js apps or edge cases the router doesn't cover.

---

## Measuring INP from real users

INP can't be meaningfully measured with synthetic tests. Lighthouse doesn't measure it in a way that reflects real interaction patterns — it simulates a few interactions, not the full session. The only honest INP measurement is from actual users doing actual things.

```typescript
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.entryType === 'event' && entry.duration > 200) {
      sendMetric({
        metric: 'INP_candidate',
        value: entry.duration,
        target: (entry.target as Element)?.tagName,
        page: location.pathname,
      });
    }
  }
}).observe({ type: 'event', durationThreshold: 100, buffered: true });
```

This captures interaction events over 100ms — the candidates that might become your INP score. Logging the target element tells you which UI components are the source of slow interactions, which is where the optimization work starts.

If you're already monitoring LCP in production, adding INP measurement to the same pipeline is straightforward. I built [RPAlert](https://rpalert.dev) to handle this for React apps — it tracks Web Vitals including INP from real browsers and alerts via Slack or Discord when thresholds are crossed. Given that INP is now an official ranking signal and most React apps haven't tuned for it, catching regressions as you work toward improvement is worth having in place.

---

## What to prioritize now

**Measure your INP first.** Add the `PerformanceObserver` above to production and collect a week of data. Look at which interactions are the worst offenders and which pages they're on. The distribution will tell you where to focus.

**Audit your event handlers.** Find click and input handlers that do significant synchronous work and identify which ones can be wrapped in `useTransition`.

**Switch from Long Tasks to LoAF** in your monitoring if you're already collecting Long Task data. The attribution is better and the LoAF data supersedes Long Tasks for debugging.

**Check your INP score in Chrome UX Report** (via PageSpeed Insights or Search Console) for a baseline. If you're in the "Needs Improvement" or "Poor" range, it's affecting your search ranking today.

---

The FID-to-INP transition isn't just a metric rename. It's a change in what "responsive" means for a passing grade. A React app that passed Core Web Vitals before March 2024 might be failing now without any code having changed. Worth checking.
