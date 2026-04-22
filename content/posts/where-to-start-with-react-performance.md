---
title: Where to Start with React Performance
date: 2026-04-22
tags:
  - react
  - performance
  - webdev
  - javascript
slug: where-to-start-with-react-performance
description: Eight articles on React performance, ordered so each one gives you something to apply before moving to the next. Start here if you don't know where to start.
draft: false
---

You've probably already tried something. Added `useMemo` in a few places. Run Lighthouse. Checked the bundle size. Maybe split a route or two.

And the app still feels slow.

The issue is usually not the optimization — it's that the mental model came later, or never. Performance work done without a clear picture of what you're measuring is mostly guesswork. Some of it sticks. A lot doesn't.

The articles below are ordered so that each one gives you something concrete before you move to the next. They're not meant to be read in a weekend. Work through one, apply it to your actual app, then come back.

---

## Understand what the browser measures before you touch anything

Before profiling, before optimizing, read [Core Web Vitals Explained](https://dev.to/nosyos/core-web-vitals-explained-what-they-are-how-to-measure-them-and-why-they-matter-for-react-apps-3f2p).

LCP, INP, CLS — these aren't SEO checkboxes. They're the closest thing we have to a standardized measurement of how fast your app feels to a real user. The article walks through what each metric actually captures, how to read them in a React app, and which thresholds matter in practice. If you've skimmed the MDN page and moved on, this will fill in the gaps that MDN skips.

---

## Find your LCP element before you do anything else

Lighthouse will show green image audits while your LCP sits at 4.2 seconds. I've seen it. The culprit was a CSS `background-image` used for the hero. `next/image` doesn't touch those. Nobody had checked which element was actually being measured.

[Most LCP Fixes Come Down to One Image](https://dev.to/nosyos/most-lcp-fixes-come-down-to-one-image-2i09) is about exactly this: the diagnosis step most developers skip. You cannot fix LCP reliably until you know what element the browser is treating as "largest." That's the only point of this article, and it's worth the ten minutes.

---

## Two sources of drag that don't show up in your component tree

You fix the hero. LCP improves. The page still feels sluggish during interaction. This is usually long tasks — JavaScript work that blocks the main thread long enough for users to notice.

[Long Tasks Are Quietly Killing Your React App's Performance](https://dev.to/nosyos/long-tasks-are-quietly-killing-your-react-apps-performance-3487) explains what they are, where to find them in DevTools, and why React apps are particularly prone to generating them. Read this before you reach for any scheduler-level fixes.

Then read [The Scripts You Didn't Write Are Slowing Down Your App](https://dev.to/nosyos/the-scripts-you-didnt-write-are-slowing-down-your-app). Analytics tags, chat widgets, tag managers firing pixels for campaigns that ended months ago — these all compete for main thread time. The engineering team usually has no idea how many are running or what they cost. This article gives you the tools to find out.

---

## Your development environment is not your production environment

This one is short. Read [Why Your App Feels Fast in Staging and Slow in Production](https://dev.to/nosyos/why-your-app-feels-fast-in-staging-and-slow-in-production-27e6), then look at how you've been profiling. CPU throttling, real network conditions, cold cache behavior — the article is a checklist you can run against your current setup immediately.

---

## Don't assume the React Compiler handles everything

If you're on React 19 or thinking about the compiler, [What the React Compiler Quietly Skips](https://dev.to/nosyos/memoization-in-the-react-compiler-era-what-actually-changes-3e6b) covers what it does and doesn't automate. Side effects, context, components with non-deterministic output — these are still your problem. The article is specific enough to be useful without being alarmist about it.

---

## Stop letting fixed problems come back

Performance regressions are quiet. A dependency updates, a feature ships, and the LCP you worked to fix climbs back above three seconds. Nobody notices until a user says something.

[Catching React Performance Regressions Before Your Users Do](https://dev.to/nosyos/detecting-performance-regressions-right-after-you-deploy-403f) is about wiring performance checks into CI so regressions surface before merge. It's the step most teams skip because it feels like overhead — until they've been burned once.

CI catches regressions in test conditions. But production is different. [Monitoring Past Performance vs. Alerting Real-Time Issues](https://dev.to/nosyos/monitoring-past-performance-vs-alerting-real-time-issues-what-react-teams-are-missing-hdc) draws the line between historical analytics and real-time alerting. Most teams have one and assume it covers both. It doesn't.

If you'd rather not build the alerting layer yourself, [RPAlert](https://rpalert.dev) detects LCP degradation in production and sends a notification to Slack or Discord within 60 seconds. There's a free tier, and setup is one `npm install` and a component wrapper. It's not a replacement for understanding your metrics — but once you understand them, you'll want to know the moment they break.

---

## Read in this order

1. [Core Web Vitals Explained](https://dev.to/nosyos/core-web-vitals-explained-what-they-are-how-to-measure-them-and-why-they-matter-for-react-apps-3f2p)
2. [Most LCP Fixes Come Down to One Image](https://dev.to/nosyos/most-lcp-fixes-come-down-to-one-image-2i09)
3. [Long Tasks Are Quietly Killing Your React App's Performance](https://dev.to/nosyos/long-tasks-are-quietly-killing-your-react-apps-performance-3487)
4. [The Scripts You Didn't Write Are Slowing Down Your App](https://dev.to/nosyos/the-scripts-you-didnt-write-are-slowing-down-your-app)
5. [Why Your App Feels Fast in Staging and Slow in Production](https://dev.to/nosyos/why-your-app-feels-fast-in-staging-and-slow-in-production-27e6)
6. [What the React Compiler Quietly Skips](https://dev.to/nosyos/memoization-in-the-react-compiler-era-what-actually-changes-3e6b)
7. [Catching React Performance Regressions Before Your Users Do](https://dev.to/nosyos/detecting-performance-regressions-right-after-you-deploy-403f)
8. [Monitoring Past Performance vs. Alerting Real-Time Issues](https://dev.to/nosyos/monitoring-past-performance-vs-alerting-real-time-issues-what-react-teams-are-missing-hdc)

Eight articles. Try each concept against your own app before moving to the next. Doing it that way, this sequence will teach you more about production React performance than most tutorials manage in twice the length.
