# Backend / Data

## [RULE: SSOT] SINGLE SOURCE OF TRUTH (NO DUPLICATED STATE)

Duplicated state drifts. Drift creates invisible bugs and corrupted flows.

REQUIRED:
- Choose one canonical source for each piece of domain state. Everything else must be derived.
- If state exists in multiple representations (URL, LocalStorage, server payload), define the direction of sync and the exact moment it happens. Never allow two independent writers.

FORBIDDEN:
```jsx
// Derived/duplicated state => guaranteed drift
const [items, setItems] = useState(initialItems);
const [count, setCount] = useState(initialItems.length);
```

REQUIRED:
```jsx
const [items, setItems] = useState(initialItems);
const count = items.length;
```

## [RULE: IMPOSSIBLE_STATES] MAKE IMPOSSIBLE STATES UNREPRESENTABLE (NO BOOLEAN SOUP)

REQUIRED:
- Represent mutually exclusive lifecycle stages with a single `status` (discriminated union / state machine), not multiple independent booleans.
- Derive UI from `status` only.

FORBIDDEN:
- `isLoading/isError/isSuccess` that can contradict each other.

REQUIRED (complex lifecycles):
- Enforce allowed transitions (finite state machine). State changes must go through a single reducer/transition function that throws on invalid transitions.

FORBIDDEN:
- Assigning `status` directly without checking whether the transition is allowed.

```ts
type Status = "idle" | "loading" | "success" | "error";
type State = { status: Status; error: string | null };

const transitions: Record<Status, readonly Status[]> = {
  idle: ["loading"],
  loading: ["success", "error"],
  success: ["idle"],
  error: ["loading", "idle"],
};

export function transition(state: State, next: Status): State {
  if (!transitions[state.status].includes(next)) {
    throw new Error(`Invalid transition: ${state.status} -> ${next}`);
  }
  return next === "error" ? { ...state, status: next } : { ...state, status: next, error: null };
}
```

## [RULE: PURE_CORE] [RULE: DEPENDENCY_DIRECTION] ISOLATE DOMAIN LOGIC (PURE CORE, IMPURE SHELL)

REQUIRED:
- Business transforms/math/decision logic must be pure (no DB, `fetch`, `Date.now()`, `Math.random()`, DOM).
- The pure core must not import infrastructure modules (DB clients, HTTP clients, env/config readers).
- Inject impure dependencies (time/random/I/O) as arguments from the shell.
- Route handlers/hooks/controllers do I/O, call the pure core, then commit results.

FORBIDDEN:
- Importing `fetch`/DB clients/Next router into pure domain files.

## [RULE: CQS] COMMAND-QUERY SEPARATION (CQS)

REQUIRED:
- A function/endpoint is either a Query (read-only, no side effects, idempotent, cacheable) or a Command (mutates state/I/O, returns status or IDs).

FORBIDDEN:
- Mixing reads and writes in one call (e.g. `fetchUserAndMarkAsRead()`), which breaks retries, caching, and reasoning.

## [RULE: DATA_MINIMIZATION] DATA MINIMIZATION (STRICT PROJECTIONS, NO OVER-FETCHING)

Agents default to `SELECT *` (or "fetch the whole entity") and then pass huge objects through every layer, causing unnecessary RAM/network cost and increasing the risk of leaking PII.

REQUIRED:
- Fetch/compute/return only the fields the consumer needs (SQL projections, ORM `select`, explicit DTOs).
- API responses must be shaped DTOs, not raw DB/ORM entities. Exclude PII by default.
- In UI, pass minimal props (view models) instead of threading full server payloads deep into component trees.

FORBIDDEN:
- `SELECT *` in production code.
- Returning DB/ORM entities directly from API handlers.
- Passing giant objects through multiple layers when only `id/status/name` is used.

## [RULE: CACHE_SYMMETRY] CACHE SYMMETRY (INVALIDATION IS PART OF THE FEATURE)

Agents often add caches and forget invalidation, instantly violating [RULE: SSOT] and corrupting user-visible behavior.

REQUIRED:
- If you introduce caching for a read path (Query), you MUST design and implement deterministic invalidation in the same change, inside the corresponding write paths (Commands).
- Prefer explicit strategies: key deletion, tag-based invalidation, write-through, or event-driven invalidation.
- Document keys/tags and add at least one test that proves invalidation.

FORBIDDEN:
- Fire-and-forget caching without a concrete invalidation strategy.
- Using TTL as the only consistency mechanism for mutable business data.

## [RULE: SIDE_EFFECTS] SIDE EFFECTS (EXPLICIT, IDEMPOTENT, ATOMIC)

Assume duplicates and partial failure: double-clicks, re-submits, retries, webhook re-deliveries.

REQUIRED:
- Side-effectful operations must be idempotent (dedupe/idempotency key) or must reject duplicates loudly.
- Multi-step writes must be atomic (all-or-nothing) or have explicit compensation/rollback.
- Handle race conditions explicitly: cancel stale in-flight requests (`AbortController`) or correlate responses (sequence/correlation IDs). Never overwrite state with out-of-order network responses.
- For shared backend resources, use optimistic concurrency control (ETag/row-version) or DB-atomic mutations (e.g. `UPDATE ... SET val = val + 1`) to prevent lost updates; reject conflicts loudly.
- Prefer immutable updates so state/render changes stay predictable.

FORBIDDEN:
- Fire-and-forget writes that can run twice and silently create duplicates.
- Partial updates that can leave the system inconsistent.
- Blind read-modify-write cycles in app memory (read row -> mutate in Node.js -> write back) without OCC or explicit transactional locking.

## [RULE: DISTRIBUTED_TX] PARTIAL FAILURES (DB + EXTERNAL I/O)

Fail-fast alone does not prevent inconsistency when a workflow touches both our DB and an external system (payments, email, 3rd-party APIs). Timeouts and partial outages create "half-done" reality.

REQUIRED:
- For DB + external side effects, design for partial failures: Transactional Outbox, Saga/compensation, or another explicit delivery/rollback mechanism.
- Make external calls idempotent (idempotency keys) so retries are safe and do not duplicate writes/charges.
- Persist the intent in the DB first (outbox) or ensure you can compensate if a later step fails.

FORBIDDEN:
- Sequential "write then call external" flows without outbox/compensation (`await db.update(); await stripe.charge();`) that can leave the DB committed when the external call fails or times out.
- Treating a timeout as "probably succeeded" without an explicit reconciliation mechanism.

## [RULE: EXPAND_AND_CONTRACT] ZERO-DOWNTIME SCHEMA & PUBLIC API CHANGES (EXPAND/CONTRACT)

For persisted schemas and public API contracts depended on by existing clients, one-shot breaking changes can cause downtime.

REQUIRED (parallel change):
- Expand: add the new field/column/behavior in a backwards-compatible way (additive change). If needed, dual-write at the boundary.
- Migrate: backfill data and update all producers/clients to the new contract.
- Contract: remove the old field/column/behavior only after all dependencies are gone.

REQUIRED (keep the core canonical):
- The domain/core still operates on one canonical model. Any temporary dual-read/dual-write is allowed only inside a narrow boundary adapter/gateway and must be removed in the contract step.
- Avoid destructive drop/rename in a single deploy.

FORBIDDEN:
- One-shot destructive renames/drops of DB columns or public endpoints that existing clients depend on.
- Letting expand leak into the core as permanent version branches (`if (v1) ... else ...`).

## Backend/Data Add-on Checklist (Apply Only If Backend/Data Is Touched)

- SSOT is explicit; no dual writers.
- State machines: impossible states are unrepresentable and transitions are controlled.
- Commands vs queries are separated.
- Side effects are idempotent/atomic; lost updates prevented.
- DB + external workflows handle partial failures (outbox/saga/compensation).
- If caching is added, invalidation ships in the same change (no TTL-only consistency).
- Over-fetching is prevented (strict projections + DTOs; no raw ORM entities).
- Expand/contract is used for zero-downtime schema/public API changes.
