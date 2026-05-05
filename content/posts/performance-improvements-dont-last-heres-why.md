---
title: Performance Improvements Don't Last. Here's Why
date: 2026-04-30
tags:
  - react
  - performance
  - webdev
  - javascript
slug: performance-improvements-dont-last-heres-why
description: "Performance gains disappear through ordinary work — new dependencies, unoptimized images, tag manager scripts. The process that caused the problem never changed."
draft: false
---

A team spends a sprint optimizing LCP. Numbers improve. Six months later the app is slower than before the work started. Nobody made a single decision to make it slower. It just accumulated.

This is the normal trajectory without structural changes. Individual optimizations decay. Culture doesn't.

---

## Why the gains disappear

Performance degrades through ordinary work. A developer adds a new dependency. A designer hands off a 1.4MB hero image and nobody checks the size. Marketing adds a tag via the tag manager. A component gets a useEffect with a missing dependency that triggers extra renders. Each change is small, reviewed individually, and ships fine.

The problem is that performance review doesn't happen at the same granularity as code review. Code gets scrutinized line by line. Performance gets checked periodically, if at all, by whoever remembers to run Lighthouse.

The improvements you made six months ago are gone not because someone undid them, but because the process that created the degradation in the first place was never changed.

---

## Performance budgets only work with enforcement

A performance budget is a threshold: bundle size under 200KB, LCP under 2.5s, no new Long Tasks over 150ms. Most teams that try budgets define them and then don't enforce them, which means they don't exist.

**A budget without automated enforcement is a suggestion.** The only budget that changes behavior is one that fails a CI check and blocks a merge.

Bundlesize and Lighthouse CI both integrate into GitHub Actions and can fail a PR when thresholds are crossed:

```yaml
# .github/workflows/performance.yml
- name: Lighthouse CI
  uses: treosh/lighthouse-ci-action@v10
  with:
    urls: |
      https://staging.yourapp.com
    budgetPath: ./budget.json
    uploadArtifacts: true
```

```json
// budget.json
[
  {
    "path": "/*",
    "timings": [
      { "metric": "largest-contentful-paint", "budget": 2500 }
    ],
    "resourceSizes": [
      { "resourceType": "script", "budget": 300 }
    ]
  }
]
```

When a PR exceeds the budget, the check fails. The developer sees it before merge, not after deployment. This is the only version of a performance budget that actually works.

Start with thresholds loose enough that you're not blocking everything immediately. Tighten them incrementally as the baseline improves. The goal at the start is to prevent regression, not to hit an ideal number.

---

## Code review needs a performance lens

Most teams review code for correctness, readability, and security. Performance is rarely on the checklist, which means expensive patterns ship unnoticed.

A few things worth adding to your review process:

New dependencies should trigger a bundle size check. `bundlephobia.com` takes 30 seconds and shows exactly what a package adds to your bundle before you commit to it. A 40KB dependency that only uses two functions is worth questioning.

Components that render lists should be reviewed for scale. A list component that works with 50 items in staging might create Long Tasks at 500. If it's not using virtualization and the data could grow, flag it.

Images added to the codebase should have explicit dimensions and a format check. If someone commits a PNG larger than 100KB for a UI element, that's worth a comment.

None of this requires a formal checklist. It requires one or two engineers who know to look for it and normalize asking the question in review.

---

## Make performance data visible to everyone

Performance stays one engineer's problem when only one engineer can see the data. The moment your LCP numbers appear somewhere the whole team looks — a Slack channel, a dashboard on a shared screen, a weekly metric in the team standup — it becomes a shared concern.

The practical version: route your performance alerts to a channel where the whole team is present. When a deploy causes an LCP regression and a Slack message appears in `#engineering`, everyone sees it. The developer who shipped the change sees it. The product manager sees it. It becomes a team metric rather than an infrastructure metric.

I built [RPAlert](https://rpalert.dev) partly for this reason — the Slack and Discord integration means performance regressions surface in the same place where the team already communicates. It's a small thing that changes who feels responsible for the numbers.

The same logic applies to your analytics dashboard. If LCP trends are buried in a monitoring tool that only two people have logins for, performance will remain two people's concern.

---

## Designers and PMs are part of this

Performance problems that originate outside the engineering team can't be fixed purely by engineers. A design system that specifies large, uncompressed images as the standard will produce large, uncompressed images at every launch. A product process that doesn't include performance review before shipping a feature will produce features that haven't been evaluated for performance impact.

The lowest-effort version of this: add performance to your definition of done. Before a feature ships, someone confirms the LCP element on affected pages hasn't gotten worse. Not a full audit — a single check. If it fails, it's a bug, same as a broken form submission.

With designers specifically: the conversation is usually easier than expected once the data exists. Showing a designer that their 2MB hero image is causing a 1.2s LCP increase for mobile users is more persuasive than a general request to "optimize images." Specific numbers change specific behavior.

---

## The learning investment pays off asymmetrically

A single lunch-and-learn session where you walk the team through opening Chrome DevTools, running a Lighthouse audit, and interpreting the results changes how people work for months. Developers who've never looked at a performance waterfall start noticing things during their own testing. Designers start asking about image format before handing off assets.

The return on two hours of internal education is disproportionate to the investment. It doesn't require bringing in an outside expert or building a curriculum. It requires one person who knows the tools well enough to show them to the rest of the team.

---

Culture isn't a process you implement. It's the accumulated effect of small structural changes: enforcement that makes the budget real, review habits that catch expensive patterns early, visibility that makes performance everyone's number. The teams with consistently fast apps didn't get there through heroic optimization sprints. They just made it harder for the app to get slower without someone noticing.
