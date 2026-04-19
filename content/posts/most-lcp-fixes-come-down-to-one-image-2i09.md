---
title: Most LCP Fixes Come Down to One Image
date: 2026-04-16
tags:
  - react
  - performance
  - webdev
  - nextjs
slug: most-lcp-fixes-come-down-to-one-image-2i09
description: "Most LCP issues trace back to one image — a CSS background-image, a missing fetchpriority, or a lazy-loaded hero. Find the LCP element first."
draft: false
---

A Next.js app with `next/image` on every image component. Lighthouse image audit: no issues. LCP: 4.2 seconds. The hero was a CSS `background-image`. `next/image` doesn't touch those. Nobody had checked what the LCP element actually was.

---

## Find your LCP element before you do anything else

This is the step most people skip. They add `next/image`, run Lighthouse, see green checkmarks on the image audit, and wonder why LCP is still slow.

Open Chrome DevTools, run Lighthouse, and look at what it marks as the LCP element. If it's a `background-image` set via CSS, the browser can't preload it the same way it handles a real `<img>` tag, and it won't get early fetch priority. Move it to an `<img>` element. This one change has fixed more LCP problems than anything else I've seen.

---

## `fetchpriority="high"` is doing more work than most developers realize

The browser assigns fetch priority based on what it finds during the initial HTML parse. Images discovered late — inside components that render after hydration, or below the fold at first scan — get normal or low priority. By the time the browser decides to fetch them, the LCP window is already closing.

For your LCP image, you want the fetch to start as early as possible.

```html
<img
  src="/hero.webp"
  fetchpriority="high"
  width={1200}
  height={600}
  alt="..."
/>
```

In Next.js, the `priority` prop on `next/image` sets this automatically and also injects a preload link into `<head>`:

```tsx
<Image
  src="/hero.webp"
  priority
  width={1200}
  height={600}
  alt="..."
/>
```

Don't use `priority` on more than one or two images per page. Telling the browser everything is urgent means nothing is.

---

## next/image defaults will silently hurt your LCP

`next/image` lazy-loads by default. That means if your LCP image is rendered via `next/image` without `priority`, the browser is intentionally delaying its fetch until the image is about to enter the viewport.

I've seen this cause regressions on otherwise well-optimized pages. The format is correct, the dimensions are explicit, Lighthouse scores are green — but LCP is 300ms slower than it should be because someone forgot `priority`. It doesn't throw a warning. It just quietly loads late.

For any image that could be the LCP element on any viewport — hero images, above-the-fold product shots, article cover images — set `priority`. Default to it rather than remembering to add it.

---

## Explicit dimensions are non-negotiable

A browser that doesn't know an image's dimensions reserves no space for it. When the image loads, content shifts. That's a CLS problem, not just a performance one — it makes the page feel broken to users even if the load time is acceptable.

`next/image` will warn you when dimensions are missing. Every other `<img>` in your codebase that doesn't go through `next/image` should have explicit `width` and `height` set too. It takes ten seconds and prevents a class of layout bugs entirely.

---

## Format: stop overthinking it

WebP is 25–35% smaller than JPEG at equivalent quality. AVIF is another 20–30% on top of that. `next/image` serves AVIF to browsers that support it and falls back to WebP automatically — you don't need to configure anything.

The format switch matters, but once you're serving WebP, the gains from AVIF are marginal compared to getting `fetchpriority` right on the LCP element. Fix the priority first.

---

## Optimizing once isn't enough

Lighthouse confirms the fix on your machine. It doesn't tell you whether it holds under real conditions — actual devices, varied networks, CDN behavior on cold loads.

Measuring from real users is the only way to know:

```typescript
new PerformanceObserver((list) => {
  const lcp = list.getEntries().at(-1)?.startTime;
  if (lcp) sendMetric({ metric: 'LCP', value: lcp, page: location.pathname });
}).observe({ type: 'largest-contentful-paint', buffered: true });
```

The harder problem is that optimizations regress. A new developer adds a hero image without `priority`. Someone replaces an `<img>` with a CSS background. The LCP you fixed at 1.6s quietly climbs back to 3.2s after the next deploy, and nobody notices until a user mentions it. If you want to catch that within minutes rather than days, I built [RPAlert](https://rpalert.dev) for exactly this — it handles the LCP monitoring and alerting layer for React apps, collecting field data from real browsers and posting to Slack or Discord when thresholds are crossed. Worth setting up after you've done the optimization work, so the gains actually stick.

---

The fix is almost always the same: find the LCP element, make it a real `<img>` tag, set `fetchpriority="high"`, give it explicit dimensions. Everything else is secondary.
