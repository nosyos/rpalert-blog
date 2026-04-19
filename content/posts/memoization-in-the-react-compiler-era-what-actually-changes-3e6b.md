---
dev_to_url: https://dev.to/nosyos/memoization-in-the-react-compiler-era-what-actually-changes-3e6b
title: What the React Compiler Quietly Skips
date: 2026-04-09
tags:
  - webdev
  - react
  - performance
  - javascript
slug: memoization-in-the-react-compiler-era-what-actually-changes-3e6b
description: "The React Compiler auto-memoizes re-renders, but it doesn't touch network waterfalls, bundle size, or Long Tasks. Here's what it skips and what you still own."
draft: false
---

React Compiler 1.0 went stable in October 2025. Half the tutorials I saw declared `useMemo` dead. It's not — and on most existing codebases, the compiler will silently skip the components you most want it to optimize.

---

## The compiler handles one thing

Re-render performance. It's a build-time plugin that analyzes your components and inserts memoization automatically, without you writing it.

The genuinely useful part: it can memoize values in code paths after an early return, which manual `useMemo` can't do.

```jsx
function Component({ isAdmin, data }) {
  if (!isAdmin) return null;
  const processed = expensiveTransformation(data); // compiler memoizes this
  return <Chart data={processed} />;
}
```

What it doesn't touch: first render cost, Long Tasks from large list renders, expensive one-time computations on mount. None of that changes.

---

## Silent bailouts

When the compiler encounters code it can't safely analyze — mutating props, reading mutable refs during render, class instances with internal state — it skips the component entirely. No warning. No error. It just leaves that component unoptimized and moves on.

This is the part that catches people off guard. You enable the compiler expecting your most expensive component to benefit, and nothing changes. The compiler bailed on it without telling you.

The diagnostic is in React DevTools. Successfully compiled components show a "memo ✨" badge. Check your heaviest components first. If the badge isn't there, that's your answer.

---

## Most existing codebases have violations

The compiler works well on clean, pure function components with immutable data. Greenfield Next.js apps tend to fit. Existing apps often don't.

Patterns that cause silent skips:

```jsx
// Direct mutation during render
function BadComponent({ items }) {
  items.push(newItem); // skipped
  return <List items={items} />;
}

// Mutable ref read during render
function AlsoProblematic({ inputRef }) {
  const value = inputRef.current; // skipped
  return <div>{value}</div>;
}

// Class instance methods
function WithClassInstance({ model }) {
  const label = model.getFormattedLabel(); // compiler can't track internal state
  return <span>{label}</span>;
}
```

None of these are bugs. Your app won't break. But the compiler won't help them.

**Before enabling the compiler on an existing codebase, run the ESLint plugin first.** `eslint-plugin-react-hooks` with `recommended-latest` includes compiler rules. The violation count is a rough proxy for actual benefit. High violation count means the compiler will spend most of its time bailing out.

---

## useMemo isn't dead

There's still one category where manual memoization is the right call: `useEffect` dependencies that need guaranteed reference stability.

```jsx
function Component({ userId }) {
  const options = useMemo(() => ({
    headers: { 'X-User-Id': userId }
  }), [userId]);

  useEffect(() => {
    fetchData(options);
  }, [options]);
}
```

The compiler's own docs call `useMemo` and `useCallback` valid escape hatches for this. The mental shift is from reaching for them by default to reaching for them when you have a specific reason. That's a real improvement — just not elimination.

For existing code with lots of manual memoization, don't rush to remove it. The docs explicitly recommend leaving it in place for now. Removing it can change the compiler's output in ways that don't surface until something re-renders unexpectedly.

---

## Roll it out on a subset first

Next.js 15+ supports annotation mode, which only compiles files that opt in:

```js
// next.config.js
const nextConfig = {
  experimental: {
    reactCompiler: {
      compilationMode: 'annotation',
    },
  },
};
```

```jsx
'use memo'; // top of each file you want compiled

export function MyComponent() {
  // compiler applies here
}
```

One thing worth following: pin the exact compiler version with `--save-exact`. The React team has said memoization behavior may change in minor versions. Auto-upgrading and then debugging unexpected re-render changes is not a good use of a morning.

---

## What to write going forward

New components: write them without manual memoization. Pure functions, no mutations during render, and the compiler handles it.

Existing components: run ESLint first, check the DevTools badges after enabling, and don't touch working `useMemo`/`useCallback` calls until you have a concrete reason.

For components doing genuinely heavy work — large list renders, expensive data transformations — the compiler helps with unnecessary re-renders, but the underlying cost is still there. Those still need virtualization, `useTransition` for non-urgent updates, or Web Workers for off-thread computation.

The compiler is a real improvement, particularly for deeply nested trees where unnecessary re-renders compound. It raises the floor for everyone. It just doesn't replace thinking about where the expensive work actually is.
