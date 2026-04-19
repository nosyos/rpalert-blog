---
dev_to_url: https://dev.to/nosyos/monitoring-past-performance-vs-alerting-real-time-issues-what-react-teams-are-missing-hdc
title: Monitoring and Alerting Are Different Jobs. Most React Teams Only Have One.
date: 2026-03-27
tags:
  - react
  - performance
  - webdev
  - javascript
slug: monitoring-vs-alerting-react-teams
description: Performance monitoring and real-time alerting are two different jobs. Here's what most React teams are missing.
draft: false
---

# Monitoring and Alerting Are Different Jobs. Most React Teams Only Have One.

_Tags: #react #performance #webdev #javascript_

---

Tuesday you deploy. Thursday your LCP climbs from 1.2s to 2.8s. Friday the reviews start coming in. Your monitoring dashboard shows the spike clearly — it just waited until you opened it to tell you.

That's the gap. Not a tooling failure exactly. More of a category error.

---

## What monitoring tools are actually built for

Datadog, New Relic, your APM of choice — these are built for historical analysis. What happened over the past week, which pages are consistently slow, where the backend bottlenecks are. That's genuinely useful work, and they do it well.

What they're not designed for: telling you that something is wrong right now, minutes after it started. The data aggregation that makes historical analysis accurate also introduces latency. By the time a threshold alert fires in most APM tools, your users have already been in the slow experience for 10–20 minutes.

Error trackers have the same gap from the other direction. Sentry is excellent at catching exceptions. A LCP that degraded from 1.8s to 3.5s after a deploy doesn't throw an exception. Nothing fires.

---

## The question your tools can't answer

"Did the deploy I just pushed make the app slower for real users right now?"

That's a different question from "what does our performance look like historically" or "are there errors in production." It needs a different data source — real browsers, in real time — and a different output: an alert somewhere your team will see it within minutes, not a dashboard someone has to remember to check.

The measurement layer is straightforward with the browser's native API:

```typescript
new PerformanceObserver((list) => {
  const lcp = list.getEntries().at(-1)?.startTime;
  if (lcp && lcp > 2500) {
    sendMetric({ metric: 'LCP', value: lcp, page: location.pathname });
  }
}).observe({ type: 'largest-contentful-paint', buffered: true });
```

This runs in your users' browsers and gives you real field data. The rest — threshold logic, batching, webhook routing to Slack or Discord — is buildable but accumulates edge cases fast.

I kept rebuilding this same pipeline across different projects, which is why I built [RPAlert](https://rpalert.dev). SDK install and a provider wrapper:

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

LCP, FCP, CLS, Long Tasks — all measured. Alert fires to Discord or Slack when thresholds are crossed.

![Discord Notification](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/airdhcd1wldlpny1rnq5.png)

---

## This doesn't replace what you already have

RPAlert isn't a Sentry or Datadog replacement. Sentry tells you why something broke. Datadog tells you what happened over time. RPAlert tells you when to go look at both of those.

The alert is the smoke detector. The investigation still happens with the tools you already have.

![Dashboard](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8qf5ls1o23cqemvvnjb4.png)

---

Historical monitoring and real-time alerting answer different questions. If your current stack only answers one of them, you're going to keep finding out about performance regressions from your users.
