---
name: frontend-runtime-debugging
description: Use mid-debugging when a React UI does not reflect data that is known to be correct at its source — a component renders nothing or stale content even though the API returns the right response, a react-query / data-fetching hook seems stuck, or the network panel shows a successful 200 but the UI never updates. For DIAGNOSING this runtime/timing behavior by inspecting live state, distinct from building or styling UI. Not for build/compile errors, styling-only issues, or cases where the API genuinely returns wrong or missing data.
metadata:
  version: "1.0.0"
---

# Frontend Runtime Debugging: inspect live state before re-reading source

The data is correct on the wire but the UI is wrong. This is a runtime/timing bug, and the instinct that will waste your time is re-reading source files. Resist it. Reorder the reflex.

## When to use

Invoke mid-debugging, when the symptom appears — not at task start. Use when ALL of these hold:

- A React component renders nothing, stale, or empty content.
- The data it needs is known to be correct at the source (the API/backend returns the right response).
- The gap is somewhere between "correct response on the wire" and "correct render on screen."

Typical phrasings: "the API returns the right data but the component is blank", "react-query is stuck", "the request is 200 but the UI never updates", "it works after a refresh but not on first load".

## What this is NOT for

- **Build / compile errors** — that is a toolchain bug, not a runtime one.
- **Styling / layout-only issues** — the data renders, it just looks wrong.
- **The API genuinely returns wrong or no data** — that is a normal backend/contract bug. Verify the wire response FIRST; if the response is actually wrong or empty, stop and fix the backend/contract. This skill is only for when the wire is correct but the UI is not.

## The reordering (the load-bearing lesson)

1. **Inspect live runtime state BEFORE re-reading source.** The reflex is to re-read the component, schema, layout, and provider wiring hunting for the bug. When the data is correct at the wire but the UI is wrong, that reflex is a trap: the bug is frequently NOT in any single source file but in a runtime/timing interaction between pieces that are each individually correct. Source review is structurally blind to that class. Look at live state first.

2. **A 200 in the network panel can actively mislead.** A request showing 200 means the response *arrived*. It does NOT mean the data reached the component or was committed to the cache. The response can arrive and then be discarded, removed, or never committed. Do not conclude "the fetch works" from a 200 — confirm the data actually reached the consuming component's state.

3. **For react-query, inspect the live query observer of the affected component, not just the network.** Read the observer's `status`, `fetchStatus`, `dataUpdateCount`, and `error`.
   - A query stuck at `status: "pending"` / `fetchStatus: "fetching"` with `dataUpdateCount: 0` means the queryFn promise's result was never committed to the observer. That points to the query being removed or invalidated mid-flight — NOT to a failed fetch.
   - Confirm the queryFn itself is healthy by forcing the observer to refetch. If a manual refetch resolves correctly and immediately, the fetch/client/schema are all fine, and the real bug is that the *initial* query was stranded (removed by a cache purge, invalidated by an effect, or torn down by a remount). This single observation distinguishes "the data path is broken" from "the data path is fine but the query was killed" — completely different bugs.

4. **Once localized to "query stranded mid-flight," look upstream in lifecycle/cache management.** The cause is something that ran in the same moment the query started: a `removeQueries` / `invalidateQueries` / cache reset, or a mount/unmount/remount race. Look at auth/session transition effects, principal/tenant-switch purges, and provider remounts — NOT at the fetch code.

## Decision path

```
UI wrong, data correct at source?
  → confirm the wire response is actually correct (not this skill's job if it isn't)
  → read the affected component's live query observer
       status pending + fetchStatus fetching + dataUpdateCount 0?
          → force refetch on that observer
               refetch resolves instantly & correctly?
                  → queryFn is FINE. Bug = initial query stranded.
                    Hunt upstream: removeQueries/invalidateQueries/reset,
                    remount/unmount race, auth/session/tenant purge effect.
               refetch also fails?
                  → data path genuinely broken. Inspect queryFn/client/schema.
       has data but UI still blank?
          → render/selector/conditional bug in the component, now re-read source.
```

## Live-inspection tooling (leads, not stable APIs)

These reach a component's react-query observer from a running page by walking the React fiber tree from a rendered DOM element, read the observer's current query state, and force a refetch to prove the queryFn works.

**These snippets are version-sensitive.** They were derived against `@tanstack/react-query` v5.90 and React 18 fiber internals, rely on undocumented internals, and WILL drift. Treat them as starting points to adapt — re-verify the field names and fiber shape against the project's installed versions before trusting any result. They are leads, not guaranteed-stable APIs.

Walk the fiber tree from a DOM node to find hooks/observers:

```js
// Pick a DOM element rendered by the affected component (e.g. via $0 in devtools).
const el = $0; // or document.querySelector('[data-testid="..."]')
const key = Object.keys(el).find(k => k.startsWith('__reactFiber$'));
let fiber = el[key];

// Walk up/down the fiber tree; memoizedState is a linked list of hooks.
// react-query stores its observer on a hook's memoizedState.
function* fibers(root) {
  const stack = [root];
  while (stack.length) {
    const f = stack.pop();
    if (!f) continue;
    yield f;
    if (f.child) stack.push(f.child);
    if (f.sibling) stack.push(f.sibling);
  }
}

// Scan hook state for something that looks like a QueryObserver.
function findObservers(fiber) {
  const found = [];
  for (const f of fibers(fiber)) {
    let hook = f.memoizedState;
    while (hook) {
      const s = hook.memoizedState;
      if (s && typeof s === 'object') {
        // QueryObserver instances expose getCurrentResult()/getCurrentQuery().
        if (typeof s.getCurrentResult === 'function') found.push(s);
        // Some versions nest it; also check common wrappers.
        if (s.observer && typeof s.observer.getCurrentResult === 'function') found.push(s.observer);
      }
      hook = hook.next;
    }
  }
  return found;
}

const observers = findObservers(fiber);
```

Read the live query state of an observer:

```js
const obs = observers[0];
const r = obs.getCurrentResult();
console.log({
  status: r.status,            // 'pending' | 'error' | 'success'
  fetchStatus: r.fetchStatus,  // 'fetching' | 'paused' | 'idle'
  dataUpdateCount: obs.getCurrentQuery().state.dataUpdateCount,
  error: r.error,
});
// pending + fetching + dataUpdateCount 0  ==>  stranded mid-flight, not a failed fetch.
```

Force a refetch to prove the queryFn is healthy:

```js
obs.refetch().then(r => console.log('refetch:', r.status, r.data)).catch(e => console.log('refetch failed:', e));
// Resolves instantly & correctly  ==>  queryFn/client/schema are fine;
//                                       the original query was killed upstream.
```

If the project exposes the QueryClient (e.g. on `window` for debugging, or reachable from a provider fiber), prefer `queryClient.getQueryCache().findAll()` and inspect each query's `state` and `observers` directly — it is sturdier than fiber-walking, when available.

## Completion

When you have localized the bug:
1. State which side of the fork it fell on: "data path broken" vs "query stranded."
2. If stranded, name the upstream lifecycle/cache action that killed it.
3. Only then re-read the specific source file responsible — now you know which one.
