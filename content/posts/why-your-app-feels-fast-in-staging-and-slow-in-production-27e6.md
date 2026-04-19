---
dev_to_url: https://dev.to/nosyos/why-your-app-feels-fast-in-staging-and-slow-in-production-27e6
title: Why Your App Feels Fast in Staging and Slow in Production
date: 2026-04-07
tags:
  - webdev
  - react
  - performance
  - javascript
slug: why-your-app-feels-fast-in-staging-and-slow-in-production-27e6
description: "Staging hides real performance — fast hardware, no third-party scripts, warm caches. Why production is always slower and how to close the gap."
draft: false
---

A Lighthouse score of 95 on staging doesn't mean your users will see that. It means your machine, on your network, with your warm cache, hit that number once.

The gap between staging and production isn't random bad luck. It has predictable causes that almost every team hits in the same order.

---

## You're not testing on anything like a real device

The biggest one. Most web developers work on hardware that's two to three times faster than the median device visiting their app. React component trees that reconcile in 40ms on a MacBook Pro take 180ms on a mid-range Android phone from 2022. That's not a small difference — it crosses the line between "feels fast" and "feels like something is wrong."

CPU throttling in DevTools gets you closer. It's not the same. A simulated 6x slowdown doesn't capture memory pressure, thermal behavior, or how the GPU pipeline behaves on constrained hardware. **Test on a physical mid-range Android device at least once per feature that touches render-heavy components.** This is the most reliable signal you have. Everything else is an approximation.

BrowserStack works if you don't have a device. It's slower to iterate but it's still a real device.

---

## Cold cache is a different product

When you're iterating on staging, you've hit that URL dozens of times. The browser cache is warm. The CDN has every asset hot. Your service worker is running.

Your users don't have any of that on their first visit. The first visit is what determines whether they stay or leave, and it's exactly the scenario you never test.

Cold cache isn't just slower — the loading sequence is different. Resources that appear instant in your workflow take seconds the first time. Font requests that look like they resolve immediately actually block layout. Preconnect hints that feel redundant do real work on a cold visit.

Run your staging tests in an incognito window with cache disabled. It's not a substitute for real-user data but it surfaces the worst offenders immediately.

---

## Staging data doesn't tell you how your components scale

Staging databases are seeded for developer convenience: enough data to see the UI, not enough to stress it. A list component that renders 50 items smoothly in staging might be rendering 5,000 in production, and nobody noticed because the test data never revealed it.

React re-renders scale with data. A component tree that's fine at 50 records creates Long Tasks at 500. You don't need to copy production data — synthetic data at realistic scale is enough. But it has to be realistic scale.

---

## Third-party scripts you forgot about

Analytics, chat widgets, A/B testing tools, tag managers. In staging they're often disabled, sandboxed, or absent entirely. In production they load fully, compete for main thread time, and contribute to LCP delays in ways that never show up in local testing.

Run your production URL through WebPageTest with mobile throttling enabled and look at the waterfall. You'll see scripts you forgot were there. For each one, the question is simple: what breaks if this doesn't load? If the answer is "nothing visible to users," question whether it belongs.

---

## Measure from real browsers, not synthetic tests

This is where most teams underinvest.

Lighthouse runs in a controlled environment on a single configuration. It's useful for catching regressions in a CI pipeline. It is not telling you what your actual users experience.

`PerformanceObserver` runs in your users' browsers and gives you the real distribution:

```typescript
new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lcp = entries[entries.length - 1].startTime;

  fetch('/api/metrics', {
    method: 'POST',
    body: JSON.stringify({ metric: 'LCP', value: lcp, page: location.pathname }),
  });
}).observe({ type: 'largest-contentful-paint', buffered: true });
```

Add this to production. Not staging. The data you want is from real users on real devices on real networks. Once you have it, performance stops being a feeling and becomes something you can track across deploys.

The deploy window is where this matters most. A CSS change that pushes your LCP element below the fold, or a new image that wasn't optimized, can move your p75 LCP from 1.8s to 3.5s overnight. If you're only checking periodically, you'll find out from a user complaint. If you're watching the real-user data, you'll know within an hour of deploying.

The PerformanceObserver approach above works if you have somewhere to send the data and something watching the thresholds. If you'd rather not build and maintain that alerting layer yourself, [RPAlert](https://rpalert.dev) does exactly this for React apps — install the SDK, wrap your component, set your LCP threshold, and it posts to Slack or Discord within 60 seconds of a regression. There's a free tier if you want to try it on a single app first.

---

## Three things worth doing this week

Check your LCP element on your main pages. Verify it's WebP, has `fetchpriority="high"`, and has explicit dimensions. Twenty minutes, fixes the most common issue.

Add the `PerformanceObserver` snippet to production and log to your existing analytics. Just having the data changes what gets prioritized in your next sprint.

Run your production URL through WebPageTest once with a mobile throttling preset. Look at what loads, in what order, and what you've forgotten about.

---

The compounding problem with performance is that each change seems fine in isolation, in the environment where it was built. Production is the only place where all of it adds up at once. Measuring there isn't an advanced optimization — it's the baseline for knowing what's actually happening.
