---
cssclasses: []
---
Tags: 

# Fetch Requests

A practical guide for structuring, optimizing, and hardening data-fetching in web apps. Rules of thumb, step-by-step flows, helpful additions, and common pitfalls.

---

## What to check before any request

1. Make sure we are not already fetching from a previous request.
2. Check if data needs to be fetched in the first place; maybe it’s cached somewhere.
3. Do we know the **staleness policy** (TTL) for this data?  
4. Will the request be **cancelled** if it becomes irrelevant (route change, new input)?  
5. What should the user see for **loading**, **empty**, and **error** states?  
6. Are we handling **authentication/authorization** (tokens, cookies) safely?  
7. Do we have a **timeout** and **retry** strategy?  
8. Is the request **idempotent**, and if not, how do we avoid duplicate writes?

---

## Rules of Thumb (quick reference)

- **Fetch only when needed**, reuse cached data when it’s fresh enough.

- **Deduplicate** identical in-flight requests to the same resource.

- Always use **AbortController** to cancel stale requests.

- Set a **timeout**; don’t let fetches hang forever.

- **Validate** responses (assume nothing about shape or type).

- **Never retry 4xx** (user/client errors). **Retry 5xx/network** with exponential backoff.

- Prefer **GET for reads**; for writes, use proper verbs (POST/PUT/PATCH/DELETE) and **idempotency keys** when repeating may occur.

- Default to **stale-while-revalidate** for great UX: show cached data instantly, refresh in background.

- **Keep secrets server-side**. Don’t ship private API keys to the browser.

- **Log & measure**: record request durations, status codes, and error rates.


> [!tip] If two parts of the UI need the same resource, fetch once and share via a cache/store.

---

## Step-by-Step: Read (GET) flow

1. **Define need** → When do we need the data (on mount, on scroll, on user action)?

2. **Check cache** → Memory/state/IndexedDB/localStorage. If present and fresh, use it. If stale, show it and revalidate in background.

3. **Build request** → URL + query params; headers (`Accept`, `Authorization` if needed); `credentials` if using cookies.

4. **Prepare control** → Create `AbortController`; set a hard **timeout** with `setTimeout(() => controller.abort(), X)`.

5. **Optionally debounce/throttle** (e.g., for live search) and **dedupe** in-flight requests.

6. **Send fetch** with `signal` from the controller.

7. **Handle response**
    
    - 2xx → verify `Content-Type`, parse accordingly, **validate** (schema/TypeScript), normalize, update state and cache (with timestamp/ETag).
    
    - 304 (Not Modified) → trust cache, bump freshness.
    
    - 4xx → show actionable message (401/403 → auth UX), do not retry.
    
	- 5xx/network → retry with backoff up to N attempts; otherwise surface error.

8. **Render states** → loading → success/empty/error; announce changes for accessibility if needed.

9. **Cleanup** → Clear timers; abort on unmount or when superseded by a newer request.


---

## Step-by-Step: Write (POST/PUT/PATCH/DELETE) flow

1. **Confirm idempotency** → Avoid duplicate writes; use **idempotency keys** or dedupe.

2. **Prepare optimistic UI** (optional) → Update UI immediately, rollback on failure.

3. **Send request** with timeout and abort capability.

4. **On success** → Update local cache; **invalidate or patch** any queries that depend on this data.

5. **On failure** → Rollback optimistic changes; distinguish validation (422) vs. permission (403) vs. server (5xx); selectively retry only safe operations.


---

## Helpful Additions (nice-to-haves)

- **Conditional requests**: Use ETag/`If-None-Match` or `If-Modified-Since` to reduce payloads.

- **Background refresh**: SWR pattern (serve cache instantly; refresh behind the scenes).

- **Prefetch on hover/intent** for likely next views.

- **Pagination/infinite scroll**: Prefer cursor-based APIs; keep `hasNextPage` in state.

- **Request batching** or **parallelization** where supported.

- **Circuit breaker**: Temporarily stop calls to a failing endpoint.

- **Offline support**: Service Worker caching + background sync for queued writes.

- **Schema validation**: Zod/valibot/TS runtime checks to guard against backend drift.

- **Observability**: Log request IDs, durations, and key error contexts; add metrics.


---

## Common Pitfalls (avoid these)

- **Duplicate fetches** from multiple components for the same data → Use a shared cache or dedupe map.

- **No timeout/cancellation** → Stuck spinners and state updates after unmount.

- **Blind JSON parsing** → 204/empty bodies cause crashes; check `Content-Type`.

- **Retrying non-idempotent writes** → Accidental duplicates.

- **Secrets in client** → Leaked API keys; always proxy via a backend.

- **Overfetching** → Missing pagination/filters; large payloads slow the app.

- **Inconsistent cache invalidation** → Showing stale data after mutations.

- **Swallowed errors** → Unhandled promise rejections; no user feedback.


> [!warning] Don’t ignore `AbortError`. Guard handlers so cancelled requests don’t clobber newer state.

---

## Minimal utilities (vanilla JS)

**Dedupe + timeout + parse safely**

```js
const inFlight = new Map();

async function request(url, { method = 'GET', headers = {}, body, timeout = 10000, dedupeKey, credentials } = {}) {
  const controller = new AbortController();
  const t = setTimeout(() => controller.abort(new Error('timeout')), timeout);

  const key = dedupeKey ?? `${method}:${url}:${body ?? ''}`;
  if (inFlight.has(key)) return inFlight.get(key); // dedupe identical in-flight

  const p = fetch(url, {
    method,
    headers: { 'Accept': 'application/json', ...headers },
    body,
    signal: controller.signal,
    credentials, // 'include' if using cookies, otherwise omit
  })
    .then(async (res) => {
      clearTimeout(t);
      if (!res.ok) {
        const msg = await safeErrorMessage(res);
        const err = new Error(`${res.status} ${res.statusText}: ${msg}`);
        err.status = res.status;
        throw err;
      }
      if (res.status === 204) return null; // no content
      const ct = res.headers.get('content-type') || '';
      if (ct.includes('application/json')) return res.json();
      return res.text();
    })
    .finally(() => inFlight.delete(key));

  inFlight.set(key, p);
  return p;
}

async function safeErrorMessage(res) {
  try {
    const ct = res.headers.get('content-type') || '';
    if (ct.includes('application/json')) {
      const data = await res.json();
      return data.message || JSON.stringify(data);
    }
    return await res.text();
  } catch {
    return 'Unknown error';
  }
}
```

**Retry with exponential backoff (retry only network/5xx)**

```js
async function withRetry(fn, { retries = 2, base = 300 } = {}) {
  let attempt = 0;
  while (true) {
    try {
      return await fn();
    } catch (e) {
      const retriable = !e.status || e.status >= 500;
      if (!retriable || ++attempt > retries) throw e;
      const delay = base * 2 ** (attempt - 1) + Math.random() * 100; // jitter
      await new Promise((r) => setTimeout(r, delay));
    }
  }
}
```

**Simple TTL cache (memory)**

```js
const cache = new Map();

async function cached(key, fetcher, ttlMs = 30000) {
  const now = Date.now();
  const hit = cache.get(key);
  if (hit && now - hit.time < ttlMs) return hit.data; // fresh
  const data = await fetcher();
  cache.set(key, { data, time: now });
  return data;
}
```

---

## React patterns (if applicable)

**Cancel on unmount and superseded requests**

```jsx
import { useEffect, useState } from 'react';

function Users({ query }) {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    fetch(`/api/users?q=${encodeURIComponent(query)}`, { signal: controller.signal })
      .then(r => r.ok ? r.json() : Promise.reject(r))
      .then(setUsers)
      .catch(err => {
        if (err.name === 'AbortError') return; // ignore cancellations
        setError(err);
      })
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [query]);

  // render loading/empty/error states
}
```

> [!tip] For complex apps, consider a data layer (TanStack Query/SWR/RTK Query) for caching, dedupe, retries, and invalidation.

---

## Security & Compliance checklist

- **HTTPS only**; enforce `SameSite` and `Secure` on cookies.

- **CSRF** protection for cookie-based auth; use tokens or double-submit pattern.

- **CORS** configured intentionally; don’t use `*` with credentials.

- **Input/Output** escaping to avoid XSS; never trust server strings.

- **Least privilege**: scope tokens; short-lived access tokens with refresh flows.


---

## When to cache vs. refetch (quick guide)

- **Cache**: static lists, reference data, user profile, feature flags.

- **SWR**: timelines, dashboards, feeds (okay to show slightly stale data).

- **Always refetch**: critical real-time data (payments status), after writes that affect the view.


---

## Copy-paste checklists

**Per-request**

-  Not already fetching / deduped

-  Cache checked & TTL known

-  AbortController + timeout set

-  Loading/empty/error states wired

-  Auth headers/credentials correct

-  Parse & validate response

-  Retry policy (5xx only)

-  Cleanup and cache update


**After mutation**

-  Update or invalidate related caches

-  Rollback plan (if optimistic)

-  User feedback (toasts, banners)
