---
dev_to_url: https://dev.to/nosyos/detecting-performance-regressions-right-after-you-deploy-403f
title: Catching React Performance Regressions Before Your Users Do
date: 2026-04-14
tags:
  - webdev
  - react
  - performance
  - javascript
slug: detecting-performance-regressions-right-after-you-deploy-403f
description: "The first 30 minutes after deploy are the cheapest window to catch performance regressions. Here's how to detect them before users notice."
draft: false
---

Three hours after a deploy, someone posts a screenshot in Slack. One-star review. App "takes forever to load." You check Lighthouse — fine. You check Sentry — no errors. The regression started the moment you deployed. Nobody knew until a user complained.

This is the normal state of affairs for most teams, and it's not hard to fix.

---

## The first 30 minutes are the cheapest

Performance regressions don't announce themselves. They show up in production under conditions you can't fully replicate: real devices slower than your dev machine, networks that drop in and out, CDN cache misses on fresh deploys.

The first 10–30 minutes after a deploy are when regressions are cheapest to fix. You can just roll back. By the time a support ticket arrives, you're already hours into the impact window and the fix is a proper investigation, not a revert.

---

## Why your existing tools miss this

Lighthouse CI runs against staging with synthetic conditions. It won't catch regressions that only appear under real network speeds or with production data volumes. A LCP that went from 1.8s to 3.2s doesn't throw an exception — Sentry has nothing to report. APM tools tell you about backend latency, not what's happening in the browser.

The shared blind spot: real users on real devices. None of these tools will fire when your LCP degrades after a deploy.

---

## LCP is what to watch

For deploy regressions specifically, LCP is the right metric. It's the best proxy for perceived load speed, and it's where most regressions surface first. Long Tasks are the clearest signal of render bloat. FCP is a useful early warning.

The browser has a native API for all of this:

```typescript
const lcpObserver = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lcp = entries[entries.length - 1].startTime;

  if (lcp > 2500) {
    // don't batch threshold crossings — send immediately
    sendMetric({ metric: 'LCP', value: lcp, page: location.pathname });
  }
});
lcpObserver.observe({ type: 'largest-contentful-paint', buffered: true });
```

This runs in every user's browser. The question is what you do with the data. A minimal pipeline:

```
React app → PerformanceObserver → batch POST every 30s (immediate on threshold cross)
→ your API → threshold check → Discord/Slack alert
```

The batching distinction matters. Routine measurements can queue up — there's no reason to POST on every LCP reading. But when something crosses a threshold you care about, you want it sent immediately, not held for the next batch window.

For the alert destination: email gets buried. If your team is in Discord or Slack, that's where it should go. Someone needs to see it within five minutes of the regression starting.

---

## What the alert loop actually looks like

You deploy at 2pm. At 2:03, a Discord message arrives: LCP exceeded 2.5s on `/checkout`, three minutes after the last deploy. You open the diff, find a new image component missing `loading="lazy"`, fix it, deploy the hotfix by 2:15.

Fifteen minutes of degraded performance.

Without the alert: the first signal is a support ticket at 4:30pm. You dig through Sentry — nothing, because no exceptions were thrown. You run Lighthouse locally — looks fine, warm cache. You eventually find the image issue around 6pm. Four hours of impact instead of fifteen minutes.

The alert doesn't prevent the regression. It collapses the time between "regression exists" and "someone is fixing it."

---

## Building vs. not building this

The pipeline above isn't complicated to build. It's also not that complicated to maintain — until the edge cases around batching logic, threshold tuning, and webhook routing start to accumulate.

If you'd rather skip building it, I built [RPAlert](https://rpalert.dev) for exactly this reason — it handles the PerformanceObserver setup, threshold logic, and Discord/Slack routing. Install the SDK and wrap your root layout:

```bash
npm install rpalert-sdk
```

```tsx
import { RPAlertProvider } from "rpalert-sdk/react";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <RPAlertProvider apiKey="YOUR_API_KEY">
          {children}
        </RPAlertProvider>
      </body>
    </html>
  );
}
```

LCP, FCP, CLS, Long Tasks — all measured from that point. Alert fires when thresholds are crossed. There's a free tier if you want to verify the pipeline end to end before committing.

One thing worth being clear about: RPAlert isn't a Sentry replacement. Sentry tells you why something broke. RPAlert tells you when to go look at Sentry. Different jobs, and they work well together.

---

The goal isn't zero regressions — that's not realistic in any active codebase. The goal is making sure you find out before your users do.
