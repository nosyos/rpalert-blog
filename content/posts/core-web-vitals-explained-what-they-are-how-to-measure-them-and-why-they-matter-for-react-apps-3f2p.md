---
dev_to_url: https://dev.to/nosyos/core-web-vitals-explained-what-they-are-how-to-measure-them-and-why-they-matter-for-react-apps-3f2p
title: Core Web Vitals Explained　What They Are, How to Measure Them, and Why They Matter for React Apps
date: 2026-03-28
tags:
  - webdev
  - react
  - performance
  - javascript
description: "LCP, INP, and CLS — what each Core Web Vital measures, how to check them in a React app, and what thresholds actually matter."
draft: false
slug: core-web-vitals-explained-what-they-are-how-to-measure-them-and-why-they-matter-for-react-apps-3f2p
---
---

If you've seen the term "Core Web Vitals" and kept scrolling, this article is for you.

It's not just SEO jargon. These three metrics are the clearest signal we have for whether a web app *feels* fast to a real user — and they're measurable directly from your React code.

This article covers what the three metrics actually mean, how to measure them without any external tools, and what to do when they're bad.

---

## What Are Core Web Vitals?

Core Web Vitals are three metrics defined by Google to measure user experience from a loading and interactivity perspective. They're based on real user data, not synthetic benchmarks.

The three metrics:

| Metric | Measures | Good threshold |
|--------|----------|---------------|
| **LCP** — Largest Contentful Paint | Loading speed | ≤ 2.5s |
| **FCP** — First Contentful Paint | Time to first visible content (supporting metric, not a Core Web Vital)| ≤ 1.8s |
| **CLS** — Cumulative Layout Shift | Visual stability | ≤ 0.1 |

There's a fourth metric worth knowing: **INP (Interaction to Next Paint)**, which replaced FID (First Input Delay) in 2024. INP measures how responsive the page feels when you click or type. We'll cover it briefly at the end.

---

## LCP — Largest Contentful Paint

**What it measures**: How long until the largest visible element on screen finishes loading.

This is usually a hero image, a large heading, or the main content block. Whatever takes up the most screen real estate "above the fold."

**Why it matters**: LCP is the closest single metric to "when does this page feel loaded." Users don't think in milliseconds — they think "did it load or not." LCP is when the answer flips from "no" to "yes."

**What causes bad LCP**:
- Large, unoptimized images (the most common cause)
- Render-blocking JavaScript or CSS that delays the page from painting
- Slow server response times (TTFB)
- Third-party scripts loading before your content

**How to measure it in code**:

```typescript
const lcpObserver = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  // Use the last entry — LCP can be updated as more content loads
  const lastEntry = entries[entries.length - 1];
  
  console.log('LCP:', lastEntry.startTime, 'ms');
  console.log('Element:', lastEntry.element); // Which element triggered it
});

lcpObserver.observe({ type: 'largest-contentful-paint', buffered: true });
```

**Good**: ≤ 2.5s  
**Needs improvement**: 2.5s – 4.0s  
**Poor**: > 4.0s

---

## FCP — First Contentful Paint

**What it measures**: How long until the browser renders the first piece of DOM content — any text, image, or non-white canvas element.

**Why it matters**: FCP is a leading indicator. A slow FCP almost always means a slow LCP. If FCP is bad, users are staring at a blank screen, which is the worst user experience possible — worse than a slow load, because users don't even know if anything is happening.

**What causes bad FCP**:
- Render-blocking resources (CSS and JS that pause HTML parsing)
- Server-side rendering issues
- Heavy JavaScript bundles that need to parse before anything renders

**How to measure it**:

```typescript
const fcpObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.name === 'first-contentful-paint') {
      console.log('FCP:', entry.startTime, 'ms');
    }
  }
});

fcpObserver.observe({ type: 'paint', buffered: true });
```

**Good**: ≤ 1.8s  
**Needs improvement**: 1.8s – 3.0s  
**Poor**: > 3.0s

---

## CLS — Cumulative Layout Shift

**What it measures**: How much the page layout shifts unexpectedly after it starts loading.

You've experienced this. You're reading an article, an ad loads above the paragraph you're on, and everything shifts down. You accidentally click the ad. That's a layout shift — and CLS measures how much of this happens across the full page lifecycle.

**Why it matters**: Layout shifts erode user trust instantly. They also cause accidental clicks, which is particularly bad on e-commerce and form pages.

**What causes bad CLS**:
- Images and videos without `width` and `height` attributes set
- Ads, embeds, or iframes without reserved space
- Dynamically injected content above existing content
- Web fonts loading and causing text to reflow (FOIT/FOUT)

**How to measure it**:

```typescript
let clsValue = 0;

const clsObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    // Only count shifts that happen without user interaction
    if (!entry.hadRecentInput) {
      clsValue += entry.value;
      console.log('Current CLS:', clsValue);
    }
  }
});

clsObserver.observe({ type: 'layout-shift', buffered: true });
```

**Good**: ≤ 0.1  
**Needs improvement**: 0.1 – 0.25  
**Poor**: > 0.25

---

## How These Metrics Relate to Each Other

Understanding the sequence helps:

```plaintext
Navigation starts
    ↓
FCP fires — first pixel of content rendered
    ↓
LCP fires — largest content element rendered
    ↓
Page becomes interactive
    ↓
CLS accumulates throughout — tracks all layout shifts
```

In practice: if FCP is bad, LCP will be bad too. If FCP is fine but LCP is bad, the issue is usually the main content (an image, a large element) taking too long. CLS is independent — a page can have great LCP and terrible CLS.

---

## Measuring in Your React App: A Complete Setup

Here's a minimal but complete implementation that collects all three metrics and logs them:

```typescript
// utils/web-vitals.ts

type MetricName = 'LCP' | 'FCP' | 'CLS';
type MetricReport = {
  name: MetricName;
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
};

function getRating(name: MetricName, value: number): 'good' | 'needs-improvement' | 'poor' {
  const thresholds = {
    LCP: [2500, 4000],
    FCP: [1800, 3000],
    CLS: [0.1, 0.25],
  };
  
  const [good, poor] = thresholds[name];
  if (value <= good) return 'good';
  if (value <= poor) return 'needs-improvement';
  return 'poor';
}

export function initWebVitals(onMetric: (metric: MetricReport) => void) {
  // LCP
  new PerformanceObserver((list) => {
    const entries = list.getEntries();
    const last = entries[entries.length - 1];
    const value = last.startTime;
    onMetric({ name: 'LCP', value, rating: getRating('LCP', value) });
  }).observe({ type: 'largest-contentful-paint', buffered: true });

  // FCP
  new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      if (entry.name === 'first-contentful-paint') {
        const value = entry.startTime;
        onMetric({ name: 'FCP', value, rating: getRating('FCP', value) });
      }
    }
  }).observe({ type: 'paint', buffered: true });

  // CLS
  let clsValue = 0;
  new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      if (!(entry as any).hadRecentInput) {
        clsValue += (entry as any).value;
        onMetric({ name: 'CLS', value: clsValue, rating: getRating('CLS', clsValue) });
      }
    }
  }).observe({ type: 'layout-shift', buffered: true });
}
```

Usage in your React app:

```typescript
// App.tsx or main.tsx
import { initWebVitals } from './utils/web-vitals';

initWebVitals((metric) => {
  console.log(`${metric.name}: ${metric.value} (${metric.rating})`);
  // Send to your analytics endpoint, logging service, etc.
});
```

---

## The Measurement Gap: Local vs. Production

Here's the part that most tutorials skip.

Lighthouse and DevTools give you **synthetic measurements** — they simulate a specific device and network condition in a controlled environment. This is useful for relative comparisons ("did my change make it better or worse?"), but it doesn't tell you what real users experience.

Real users have:
- Older devices with slower CPUs
- Variable network conditions (3G, congested WiFi)
- Many browser tabs open
- Cold cache (no previous visit to your site)

The only way to know your real-world Core Web Vitals is to **measure in production**, from real browsers. The code above does exactly that — it runs in your users' browsers and captures their actual experience.

What you do with those measurements is a separate question. At minimum, log them somewhere. Ideally, set up alerting so you know when they degrade — particularly after deploys.

---

## Quick Wins for Each Metric

If you're seeing bad numbers, here's where to start:

**Bad LCP?**
1. Check if the LCP element is an image — if so, add `fetchpriority="high"` to it
2. Convert images to WebP format
3. If using Next.js, switch to `next/image`
4. Check TTFB — if your server responds slowly, everything else suffers

**Bad FCP?**
1. Identify and remove render-blocking CSS/JS
2. Inline critical CSS
3. If using SSR, check that your server isn't doing too much work before sending HTML

**Bad CLS?**
1. Add explicit `width` and `height` to all images and videos
2. Reserve space for ads and dynamic embeds with CSS `min-height`
3. Avoid inserting content above existing content after page load

---

## What About INP?

INP (Interaction to Next Paint) replaced FID in March 2024. It measures how quickly the page responds to user interactions — clicks, taps, keyboard input.

**Good threshold**: ≤ 200ms

The most common cause of bad INP in React apps is expensive state updates that block the main thread. If you're seeing high INP, Long Tasks are usually the culprit — something is blocking the browser from responding to user input.

We'll cover Long Tasks in depth in the next article.

---

## Summary

Core Web Vitals aren't just for SEO. They're the most concrete way to measure whether your app feels fast to a real user.

The three metrics tell a story:
- **FCP**: Does anything appear quickly?
- **INP**: Does the page respond to interactions?
- **CLS**: Does the layout stay stable while loading?

FCP is worth tracking too — it's a leading indicator for LCP — but it's a supporting metric, not a Core Web Vital.


