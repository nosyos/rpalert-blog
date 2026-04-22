---
title: The Scripts You Didn't Write Are Slowing Down Your App
date: 2026-04-21
tags:
  - webdev
  - react
  - performance
  - javascript
slug: the-scripts-you-didnt-write-are-slowing-down-your-app
description: Third-party scripts — analytics, chat widgets, tag managers — compete with your React app for CPU time. How to measure their cost and take back control.
draft: false
---

I once audited a page where nearly 40% of the main thread blocking time came from a tag manager firing scripts that the engineering team didn't know were still active. Analytics from a vendor they'd switched away from. A heatmap tool from a trial nobody cancelled. A pixel for an ad campaign that ended months ago.

Nobody wrote those scripts. They accumulated.

---

## What third-party scripts actually cost you

The performance impact happens in two places: network and main thread.

On the network side, each script is an additional HTTP request, often to a slow external domain with no SLA on response time. A single chat widget might make 4–6 requests before it's ready. On a slow connection, this shows up in your waterfall as a long chain of blocking or near-blocking resources.

On the main thread, third-party scripts run JavaScript. That JavaScript competes with your React app for CPU time. A script that takes 80ms to parse and execute on a fast development machine might take 350ms on a mid-range Android phone. Every millisecond it holds the main thread is a millisecond your app can't respond to user input or complete a render.

The combination of late network requests and CPU-heavy execution is why third-party scripts are so good at pushing LCP. The browser is waiting on resources it didn't know it needed, while the main thread is occupied with someone else's code.

---

## Find out what's actually running

Before you optimize anything, run your production URL through WebPageTest with a mobile throttling preset and look at the waterfall. Sort by domain. You'll see every request, grouped by origin.

The question to ask for each third-party domain: does the engineering team know this is here, and what breaks if it doesn't load?

Chrome's Coverage tab gives you the JavaScript utilization angle — how much of each loaded script is actually executed on the page. A script that's 90% unused is paying full network and parse cost for very little value.

The surprises are usually in the tag manager. If your site uses Google Tag Manager or a similar tool, open it and look at what's configured. Marketing and analytics teams often have direct access and add tags without engineering review. The list is rarely what anyone expects.

---

## Stop loading scripts at the worst possible time

Most third-party scripts don't need to be ready before your app is interactive. Analytics doesn't need to fire before the user can click anything. Chat widgets don't need to be loaded before the hero image is painted.

The default behavior — scripts in `<head>` without `async` or `defer` — blocks HTML parsing entirely. This is almost never what you want for third-party code.

`async` loads the script in parallel with parsing, but executes it as soon as it downloads, which can still interrupt parsing at a bad moment. `defer` loads in parallel and waits until parsing is complete before executing. For most analytics and tracking scripts, `defer` is the right default.

For scripts that are truly non-critical — chat widgets, feedback tools, anything that doesn't affect the initial render — load them after hydration:

```tsx
useEffect(() => {
  const script = document.createElement('script');
  script.src = 'https://third-party-widget.com/widget.js';
  script.async = true;
  document.body.appendChild(script);
}, []);
```

This pushes execution entirely past React's initial render and hydration cycle. The widget loads when it loads. Your LCP doesn't wait for it.

In Next.js, the `Script` component handles this with the `strategy` prop:

```tsx
import Script from 'next/script';

// afterInteractive: loads after hydration, good for analytics
<Script src="https://analytics.example.com/script.js" strategy="afterInteractive" />

// lazyOnload: loads during browser idle time, good for chat widgets
<Script src="https://widget.example.com/chat.js" strategy="lazyOnload" />
```

`beforeInteractive` exists for scripts that genuinely need to be ready before the page is usable. For third-party code, that's almost never true.

---

## Tag managers are the hard part

A tag manager with unrestricted access is effectively a way for non-engineers to inject arbitrary JavaScript into production. The scripts themselves might be fine individually. The problem is the total: 8 tags that each take 50ms to execute is 400ms of main thread time that engineering had no visibility into.

**Audit the tag manager on a regular schedule.** Not annually — quarterly at minimum. For each tag: who owns it, what it does, and what happens if it's removed. Treat it like a dependency review. Tags accumulate the same way npm packages do, and they're harder to spot because they're not in the codebase.

Two practical rules that help: require engineering sign-off before any new tag is added, and set a network budget threshold that triggers a review if total third-party bytes cross it. Neither is bureaucratic overhead — they're the minimum to prevent the page you ship from drifting away from the page you tested.

---

## The problem doesn't stay solved

You optimize the loading strategy, audit the tag manager, remove the stale scripts. A month later, marketing adds a new analytics tool. Another month, a new A/B testing SDK. Each addition seems small in isolation.

Measuring this from real users catches it before it accumulates. Adding the `PerformanceObserver` for Long Tasks gives you a signal when a new script is hitting the main thread harder than expected:

```typescript
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 100) {
      sendMetric({ metric: 'LongTask', duration: entry.duration, page: location.pathname });
    }
  }
}).observe({ type: 'longtask', buffered: true });
```

If you want that signal to reach you automatically when a new script causes a regression — without manually checking dashboards — I built [RPAlert](https://rpalert.dev) to handle this. It monitors LCP and Long Tasks from real browsers and sends a Slack or Discord alert when thresholds are crossed. It's caught more than a few cases where a new marketing tag quietly pushed LCP past the threshold right after it was deployed.

---

The engineering team usually gets blamed when the app is slow. The scripts that actually caused it were added by someone else, through a tool that didn't require a code review. Getting visibility into that layer is half the work.
