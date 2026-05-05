---
title: Slow Pages Cost Money. Here's How to Prove It
date: 2026-04-28
tags:
  - webdev
  - react
  - performance
  - javascript
slug: slow-pages-cost-money-heres-how-to-prove-it
description: "A 0.1s improvement in load time can mean 8% more conversions. How to connect Core Web Vitals to revenue and make the business case."
draft: false
---

Performance work stalls not because engineers don't care, but because the business case is vague. "The app feels faster" doesn't unlock budget. "We reduced LCP by 800ms and checkout conversion went up 12%" does.

The teams that get sustained investment in performance are the ones who learned to speak in numbers that matter to stakeholders. Here's how to build that argument.

---

## The numbers that already exist

You don't need to run your own study. The data is well-established at this point:

Google's research on Core Web Vitals found that sites meeting the "Good" threshold for LCP see 24% fewer abandoned page loads compared to sites in the "Poor" range. Deloitte found that a 0.1s improvement in mobile site speed correlates with an 8% increase in conversions for retail sites. Vodafone saw a 31% increase in sales after a 31% improvement in LCP.

These aren't cherry-picked outliers. The pattern holds across industries: slower pages lose users at predictable rates, and the loss scales with how slow the experience is.

The most direct way to frame it for a non-technical stakeholder: every 100ms of additional load time costs some percentage of your conversions. The exact number varies by industry, audience, and baseline speed, but the direction is never ambiguous.

---

## How to calculate your own cost

Generic industry statistics are useful for initial buy-in. Your own data is what closes the argument.

Start with what you have. Most teams have analytics that show page load time alongside conversion or retention metrics. Segment your users by LCP performance bucket — "Good" under 2.5s, "Needs Improvement" 2.5–4s, "Poor" over 4s — and compare conversion rates across those groups.

If your checkout conversion rate for users with LCP under 2.5s is 4.2% and for users with LCP over 4s it's 2.8%, the math becomes concrete. If 20% of your traffic is in the "Poor" bucket and you process 10,000 checkouts per month, closing that gap is worth roughly 280 additional conversions per month at whatever your average order value is.

This isn't a controlled experiment — there are confounding variables, slower devices correlate with other demographic factors, and so on. But it's directionally correct and it's your data, which is far more persuasive to a leadership team than a Deloitte study about retail sites.

---

## The cost of a regression is easier to calculate than the cost of being slow

Here's the argument that often lands faster: quantify what a performance regression costs, then show what it costs to not catch one quickly.

A deploy that increases LCP by 1.5s across your checkout flow and sits undetected for 4 hours: take your hourly transaction volume, apply the conversion rate delta you measured above, and multiply. For a moderately busy e-commerce site, a 4-hour regression can mean tens of thousands of dollars in lost revenue. That number is exact, not an estimate, because you have the actual transaction data from that window.

The ROI argument for monitoring then writes itself. If catching a regression in 10 minutes instead of 4 hours saves $30,000, the cost of whatever tooling enables that is trivially justified.

---

## Measuring before and after is non-negotiable

The most common failure mode in performance projects: teams do the work, don't have the data to prove it made a difference, and can't justify the next round of investment.

You need real-user measurements before you start any optimization, during the work, and continuously afterward. Not Lighthouse scores — those measure synthetic conditions on a controlled machine. Field data from actual users, segmented by page and device type.

`PerformanceObserver` gives you this without a third-party dependency:

```typescript
new PerformanceObserver((list) => {
  const lcp = list.getEntries().at(-1)?.startTime;
  if (lcp) {
    sendMetric({
      metric: 'LCP',
      value: lcp,
      page: location.pathname,
      deviceMemory: (navigator as any).deviceMemory,
    });
  }
}).observe({ type: 'largest-contentful-paint', buffered: true });
```

Sending `deviceMemory` alongside the metric lets you segment by device class — low-memory devices are a good proxy for slower hardware. The performance gap between your p50 and p75 users is often where the business impact lives.

Once you have this instrumented, connect it to your analytics. LCP by page, by device, over time. When you ship an optimization, you'll see the distribution shift in the data. That shift is your ROI evidence.

For the alerting side — catching regressions before they become hours-long revenue events — I built [RPAlert](https://rpalert.dev) to handle the threshold monitoring and Slack/Discord notification layer. The "cost of a 4-hour regression" calculation I described above is exactly the argument for having that kind of alerting in place: the monitoring cost is fixed and small, the regression cost is variable and potentially large.

---

## How to frame this for stakeholders

Engineers tend to present performance work as a technical improvement. Stakeholders hear "we made some things faster." The same work framed as "we identified that 18% of our users are experiencing load times that reduce checkout conversion by 1.4 percentage points, and we have a plan to move them into the Good tier" lands differently.

A few framings that work:

**Revenue at risk:** "X% of sessions have LCP over 4s. Based on our conversion data, this segment converts at Y% vs Z% for fast sessions. At our current volume, that's approximately $N/month in lost revenue."

**Regression cost:** "Our last deploy regression ran for 4 hours before we caught it. Based on transaction volume during that window, the estimated revenue impact was $N. We're proposing monitoring that would have caught it in under 10 minutes."

**Competitive framing:** Run WebPageTest on your main competitors. If you're 1.2s slower on mobile than your closest competitor, that's a meaningful talking point in a room where people think about market share.

---

## KPIs worth tracking continuously

LCP p75 by page — the 75th percentile is what Google uses for Core Web Vitals thresholds, and it's the right target because it represents your slower users, not the median.

Regression frequency and MTTR (mean time to resolution) — how often you have regressions and how quickly you fix them. This makes the monitoring ROI argument over time.

Conversion rate by performance bucket — LCP Good vs. Needs Improvement vs. Poor, segmented from your analytics. This is the number that connects engineering work to business outcomes.

None of these require expensive tooling to start. They require making the measurement a consistent practice, which is the harder organizational problem.

---

The teams that make performance a sustained priority aren't the ones with the most engineering time or the biggest budgets. They're the ones who connected their performance metrics to the numbers the business already cares about. That connection starts with measuring the right things from the right place — real users, in production, continuously.
