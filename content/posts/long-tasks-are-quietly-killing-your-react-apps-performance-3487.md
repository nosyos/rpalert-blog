---
dev_to_url: https://dev.to/nosyos/long-tasks-are-quietly-killing-your-react-apps-performance-3487
title: Long Tasks Are Quietly Killing Your React App's Performance
date: 2026-04-02
tags:
  - webdev
  - react
  - performance
  - javascript
slug: long-tasks-are-quietly-killing-your-react-apps-performance-3487
description: "Any task blocking the main thread for over 50ms makes your app feel broken. How to find and break up Long Tasks in React."
draft: false
---

Here's something that doesn't get talked about enough: your React app can have great LCP and FCP scores, pass all your Lighthouse checks, and still feel sluggish to use.

The culprit is usually Long Tasks.

---

## What's a Long Task?

The browser's main thread handles everything — parsing HTML, running JavaScript, responding to user input, painting pixels. It can only do one thing at a time.

A Long Task is any task that occupies the main thread for more than **50 milliseconds** without a break. While a Long Task is running, the browser can't respond to anything else. Click a button during a Long Task? Nothing happens — until the task finishes.

50ms might sound short, but human perception starts noticing unresponsiveness around 100ms. Any Long Task over that threshold will feel broken to a user.

---

## Why React Makes This Easy to Get Wrong

React renders synchronously by default (outside of concurrent features). When you trigger a state update, React processes the entire component tree update in one go. If that update is expensive — lots of components, heavy computations, large lists — it becomes a Long Task.

The tricky part: this doesn't show up in unit tests. It doesn't throw an error. It doesn't affect your Lighthouse score in a way that's obvious. It just makes your app feel slow.

Some common patterns that create Long Tasks in React apps:

**Rendering large lists without virtualization**
```jsx
// If `items` has 500+ entries, this creates a Long Task on every render
function ItemList({ items }) {
  return (
    <ul>
      {items.map(item => <ExpensiveItem key={item.id} item={item} />)}
    </ul>
  );
}
```

**Expensive computations in render**
```jsx
function Dashboard({ rawData }) {
  // This runs on every render, blocking the main thread
  const processed = rawData.reduce((acc, row) => {
    return heavyTransformation(acc, row);
  }, {});

  return <Chart data={processed} />;
}
```

**State updates that cascade through large component trees**
A single `setState` at the top of a deeply nested tree can trigger hundreds of re-renders in one synchronous block.

---

## How to Detect Long Tasks

The browser exposes this through `PerformanceObserver`:

```typescript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log('Long Task detected:', {
      duration: entry.duration,       // how long it ran (ms)
      startTime: entry.startTime,     // when it started
      attribution: entry.attribution, // which script caused it (limited support)
    });
  }
});

observer.observe({ type: 'longtask', buffered: true });
```

Run this in your production app for a day and look at the output. If you're seeing regular Long Tasks over 100ms — especially clustering around user interactions or page loads — you have a real problem.

One thing worth knowing: `entry.attribution` gives you some information about what caused the task, but browser support varies and the data is often vague. It'll tell you it was a script, but not always which function.

For more precise attribution, the Chrome DevTools Performance panel is your best friend. Record a session, look for the red triangles at the top of the flame chart — those are Long Tasks. Click into them and you'll see exactly which functions ran.

---

## Fixing Long Tasks

There's no single fix. The approach depends on what's causing the task.

**For expensive renders: useMemo**

```jsx
function Dashboard({ rawData }) {
  // Only recalculates when rawData changes
  const processed = useMemo(() => {
    return rawData.reduce((acc, row) => heavyTransformation(acc, row), {});
  }, [rawData]);

  return <Chart data={processed} />;
}
```

`useMemo` doesn't prevent Long Tasks on the first render, but it prevents them from happening again unnecessarily.

**For large lists: virtualization**

Libraries like `react-window` or `@tanstack/virtual` only render the rows visible in the viewport. If you have more than a couple hundred items in a list, this is almost always worth doing.

```jsx
import { FixedSizeList } from 'react-window';

function ItemList({ items }) {
  return (
    <FixedSizeList height={600} itemCount={items.length} itemSize={50} width="100%">
      {({ index, style }) => (
        <div style={style}>
          <ExpensiveItem item={items[index]} />
        </div>
      )}
    </FixedSizeList>
  );
}
```

**For non-urgent updates: useTransition (React 18+)**

```jsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleSearch(value) {
    setQuery(value); // urgent — update input immediately
    
    startTransition(() => {
      setResults(searchItems(value)); // non-urgent — can be interrupted
    });
  }

  return (
    <>
      <input value={query} onChange={e => handleSearch(e.target.value)} />
      {isPending ? <Spinner /> : <ResultsList results={results} />}
    </>
  );
}
```

`useTransition` tells React that the update inside `startTransition` is low priority. React can interrupt it if something more urgent comes in (like another keystroke). This is particularly effective for search-as-you-type patterns.

**For truly heavy work: move it off the main thread**

If you're doing something computationally expensive that can't be memoized — parsing a large dataset, running a sorting algorithm on thousands of items — consider a Web Worker:

```typescript
// worker.ts
self.onmessage = (e) => {
  const result = heavyComputation(e.data);
  self.postMessage(result);
};

// component
const worker = new Worker(new URL('./worker.ts', import.meta.url));

worker.onmessage = (e) => {
  setResult(e.data); // runs on main thread, but the computation didn't
};

worker.postMessage(largeDataset);
```

Web Workers don't have access to the DOM, so this only works for pure computation. But when it applies, it's the cleanest solution — zero Long Task, because the work literally doesn't happen on the main thread.

---

## The Connection to INP

If you've looked at your Core Web Vitals recently, you might have noticed INP (Interaction to Next Paint) — the metric that replaced FID in 2024. It measures how long it takes the page to respond to user interactions.

Long Tasks are the primary cause of bad INP. When a user clicks and there's a Long Task in progress, the browser queues the input event and processes it after the task finishes. If that task runs for 200ms, your INP for that interaction is 200ms+ — in the "needs improvement" range.

Fixing Long Tasks improves INP directly.

---

## Monitoring This in Production

DevTools is great for debugging a specific session, but it won't tell you how often Long Tasks are happening for real users across different devices.

The PerformanceObserver code above works in production. A few things worth tracking:

- **Count of Long Tasks per page load** — is this happening on every visit or just edge cases?
- **Duration** — are they 60ms or 400ms? The severity matters
- **When they happen** — during initial load, or triggered by user interactions?

If Long Tasks spike after a deploy, that's a signal something in the new code is blocking the main thread. Having an alert set up for unusual Long Task counts is worth it — it's the kind of regression that's easy to introduce and hard to notice until users start complaining.

This is actually what pushed me to build [RPAlert](https://rpalert.dev) — I kept finding out about Long Task spikes and LCP regressions from users instead of catching them myself. It handles the PerformanceObserver setup and sends a Discord alert when thresholds are crossed, so you don't have to build the plumbing yourself.

---

That's the gist of it. Long Tasks aren't glamorous, but they're one of the most direct causes of "this app feels slow" complaints — and they're largely invisible without instrumentation. Worth adding to your monitoring stack.