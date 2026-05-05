---
title: Testing on Fast Wi-Fi Is Not a Performance Test
date: 2026-04-23
tags:
  - webdev
  - react
  - performance
  - javascript
slug: testing-on-fast-wi-fi-is-not-a-performance-test
description: "Your dev machine hides real-world performance problems — fast CPU, warm cache, stable network. Here's what you're missing and how to test properly."
draft: false
---

Most performance testing happens on a MacBook Pro, over a fast home or office connection, with a warm browser cache. Then you deploy and wonder why the numbers are different in production.

The gap isn't mysterious. You were never testing what your users experience.

---

## What your local setup hides from you

Three things consistently make local testing optimistic:

Your CPU is fast. A React component tree that reconciles in 30ms on your development machine can take 150ms on a mid-range Android phone from two years ago. JavaScript execution time scales with CPU speed, not network speed. Throttling your network doesn't help here.

Your cache is warm. You've loaded the page dozens of times during development. The browser has cached your fonts, your CSS, your JS bundles. A first-time visitor has none of that. Cold cache loads can be 2–3x slower than what you see after the third reload.

Your connection is fast and stable. Office and home wifi is typically 50–200Mbps with low latency. A user on 4G in a building with poor signal might be getting 5Mbps with 200ms latency. The same 300KB JavaScript bundle takes 0.4s on your connection and 3.2s on theirs.

---

## Chrome DevTools throttling: useful, not sufficient

DevTools lets you simulate slower network conditions and CPU performance from the Network and Performance panels. This is genuinely useful for catching obvious regressions. It's not a substitute for real-device testing.

For network throttling, the built-in presets are a reasonable starting point:

- "Fast 3G": ~1.5Mbps download, 40ms latency — approximates a decent mobile connection
- "Slow 3G": ~400Kbps download, 200ms latency — approximates a poor signal or congested network

To add CPU throttling alongside network throttling, open the Performance panel and set the CPU throttle multiplier before recording. 4x slowdown is a reasonable approximation of a mid-range phone; 6x for lower-end devices.

The limitation: CPU throttling in DevTools is a multiplier applied to your existing hardware. A 6x slowdown on a fast Mac still doesn't fully reproduce the memory pressure, thermal constraints, or GPU pipeline behavior of a real low-end device. It's a direction, not a destination.

---

## WebPageTest gives you closer to the real thing

WebPageTest runs tests on actual devices and actual network connections, not simulations on your hardware. The free tier at webpagetest.org lets you test from real locations against real mobile device profiles.

A few settings that matter:

Set the test location to somewhere geographically relevant to your users. Latency scales with distance. Testing from a US East Coast location when half your users are in Southeast Asia will give you unrealistically fast numbers.

Use a mobile device profile. The "Motorola G (gen 4)" or similar mid-range Android preset is a reasonable proxy for the median visitor to most consumer apps.

Enable "First View" only initially — it's the cold cache scenario, which is what new users experience and what you most need to optimize for.

The waterfall view is where the value is. Look at what loads, in what order, what blocks what. Third-party scripts that seem fast locally often appear as long blocking requests here. Fonts that you never notice on your warm-cache machine show up as early blocking resources. It's the closest thing to watching a real user load your page.

---

## Lighthouse: right tool, often wrong setup

Lighthouse is easy to run, well-documented, and measures the right things. It's also commonly run in a way that undermines its usefulness.

Running Lighthouse on localhost — against your dev server, with hot module reloading active — gives you numbers that have nothing to do with production. Run it against your production or staging URL.

Running it on a fast connection without throttling gives you numbers your slowest users will never see. The default Lighthouse settings in Chrome DevTools apply simulated throttling automatically; if you're running it via the CLI, check your throttling configuration.

Running it once and treating the number as stable is also a mistake. Lighthouse results vary by 10–15% between runs on the same page due to background processes and timing variations. Run it three times and take the median.

---

## The ceiling on simulation

Every simulation tool has the same ceiling: it's running on your infrastructure, with your hardware, and making assumptions about user conditions that may not match reality.

The only way to know what your actual users experience is to measure it from their browsers. `PerformanceObserver` gives you real field data:

```typescript
new PerformanceObserver((list) => {
  const lcp = list.getEntries().at(-1)?.startTime;
  if (lcp) sendMetric({ metric: 'LCP', value: lcp, page: location.pathname });
}).observe({ type: 'largest-contentful-paint', buffered: true });
```

The distribution of real-user LCP is almost always wider than what your local tests suggest. The p75 is what matters for Core Web Vitals — the 75th percentile user's experience, not the median. That user might be on a slow connection in a weak signal area, and your simulation never represented them.

Once you have real-user data, you also get deploy-time regression detection. If a change you shipped moves the p75 LCP from 1.9s to 3.1s, you want to know within minutes. I built [RPAlert](https://rpalert.dev) specifically for this — it monitors LCP and Long Tasks from real browsers and sends a Slack or Discord alert when thresholds are crossed. The simulation tools tell you what might happen; real-user monitoring tells you what did.

---

Use DevTools throttling to catch things before they ship. Use WebPageTest to get a more honest picture of production conditions. Use real-user measurement to know what's actually happening. They're not substitutes for each other — they answer different questions at different points in the development cycle.
