# Perf / Scale

## [RULE: BIG_O] ALGORITHMIC EFFICIENCY (O(1) LOOKUPS OVER O(N^2) ITERATIONS)

Transform data, do not iterate repeatedly.

REQUIRED:
- When merging/enriching/cross-referencing collections, precompute lookup maps/sets (`Map`, `Set`, `Record`) and do O(1) reads.
- For membership checks, use `Set.has()` instead of `Array.includes()` inside loops.

FORBIDDEN (O(N^2)):
`orders.map(o => ({ ...o, user: users.find(u => u.id === o.userId) }))`

REQUIRED (O(N)):
`const userById = new Map(users.map(u => [u.id, u])); orders.map(o => ({ ...o, user: userById.get(o.userId) }))`

FORBIDDEN (O(N^2) membership):
`items.filter(item => allowedIds.includes(item.id))`

REQUIRED (O(N) membership):
`const allowed = new Set(allowedIds); items.filter(item => allowed.has(item.id))`

FORBIDDEN (O(N^2) memory/GC):
Spread-based accumulation inside loops/reduces that allocates a brand new object/array every iteration.

```js
// Creates N new objects (and copies keys) for N items => GC + memory blowups at scale.
const byId = items.reduce((acc, item) => ({ ...acc, [item.id]: item }), {});
```

REQUIRED (O(1) extra space per iteration):
Mutate the local accumulator (still referentially-contained to the reducer call).

```js
const byId = items.reduce((acc, item) => {
  acc[item.id] = item;
  return acc;
}, {});
```

## [RULE: N_PLUS_ONE] [RULE: BOUNDED_CONCURRENCY] PREVENT N+1 QUERIES (BATCH I/O)

FORBIDDEN:
- `await fetch(...)` / `await db.query(...)` inside loops.
- Unbounded parallelism (e.g. `Promise.all(bigArray.map(fetch))`) over unknown/large collections (connection pool exhaustion, rate limits, OOM).

REQUIRED:
- Collect IDs/params, do ONE bulk/batch operation outside the loop (e.g. `WHERE id IN (...)` / batch endpoint), then map in memory.
- If batching is impossible, use bounded concurrency (chunking or a limiter like `p-limit`) and document the chosen limit.

## [RULE: BOUNDED_QUERIES] RESPECT MEMORY LIMITS (BOUNDED QUERIES)

REQUIRED:
- Never load unbounded collections into memory (DB/API). Default to pagination/cursors or strict `LIMIT`.
- When processing huge datasets (files/exports), prefer streaming over buffering into RAM.

## [RULE: STREAMING_IO] STREAM LARGE I/O (OOM PREVENTION)

REQUIRED:
- For potentially unbounded data (CSV exports, large JSON, big DB reads, streaming LLM responses), use streams / async iterators with backpressure.
- Keep space complexity O(1) relative to input size.

FORBIDDEN:
- Buffering entire datasets into memory (e.g. `fs.readFileSync` for huge files, building multi-megabyte strings/arrays before sending, `findAll()` over unknown-size tables).

## Perf/Scale Add-on Checklist (Apply Only If Perf/Scale Is Touched)

- Big-O is justified (avoid nested iteration joins, use Map/Set for lookups).
- Space complexity is considered (avoid spread accumulation and unbounded buffering).
- No N+1 I/O; batching is used when possible.
- Concurrency is bounded (no unbounded `Promise.all`).
- Collection reads are bounded/paginated by default.
- Heavy I/O uses streaming/backpressure.
