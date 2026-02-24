# Workflow / Debugging

## Root-Cause-First Operating Procedure (RCA SOP)

1. Stop the line. Freeze deploys for the faulty component. No features ship until the cause is known and removed.
2. Reproduce and bound. Reproduce the failure with a minimal, deterministic case; quantify scope and blast radius.
3. Trace to the first bad state. Follow the signal to the earliest invariant violation (logs, metrics, spans, DB history).
4. Eliminate the cause. Change the design, contracts, or data so the invalid state cannot occur again.
5. Harden and prevent. Add assertions that preclude recurrence.
6. Delete mitigations. Remove any temporary toggles/guards introduced during investigation.
7. Document the why. Record Trigger -> First Bad State -> Cause -> Fix -> Prevention in the PR.

## Definition of Done for a Fix (Acceptance Gate)

A fix is DONE only if ALL are true:

- [ ] The root cause (not just the symptom) is removed or made unrepresentable
- [ ] New guardrails (assertions) make the bad state impossible to re-introduce silently
- [ ] No band-aids remain (no extra retries, timeouts, skips, or silent defaults)
- [ ] RCA is documented: Trigger -> First Bad State -> Cause -> Fix -> Prevention

This gate reinforces [RULE: STRICT_FAIL_FAST] and [RULE: EXPLICIT_DEPS].

## Forbidden Band-Aid Patterns (MUST NOT USE)

```js
// FORBIDDEN: masks invalid state with a default
const cfg = readConfig() || { mode: "safe" };

// FORBIDDEN: hides flakiness with retries/timeouts
async function callService() {
  for (let i = 0; i < 5; i++) {
    try {
      return await rpc();
    } catch {}
  }
  return {}; // pretend success
}

// FORBIDDEN: bypasses validation on error
if (!isValid(order)) {
  log.warn("Invalid order, skipping..."); // continues corrupt flow
}
```

Required instead (root-cause fix):

```js
// REQUIRED: make invalid state impossible and fail fast
const cfgRaw = readConfig();
assert(cfgRaw, "Missing config"); // loud failure

async function callService() {
  const res = await rpc(); // no blind retries
  assert(isSane(res), "Invariant breach from RPC");
  return res;
}

const problems = validate(order);
if (problems.length) {
  throw new Error(`Invalid order: ${problems.join(", ")}`); // stop the line
}
```

This operationalizes [RULE: STRICT_FAIL_FAST] and [RULE: ROOT_CAUSE].

## Reviewer Checklist (Reject if any item fails)

- [ ] There is a test that would have caught the original failure
- [ ] The change removes retries/timeouts/defaults added "for stability"
- [ ] Invariants are asserted at boundaries; errors are not swallowed

## Preventive Hardening Patterns (DO USE)

- Assert early: fail at the first point of inconsistency (not three layers later)
- Idempotency and deduplication for side-effectful operations
- Backpressure (not blind buffering); circuit-breakers that trip loudly (not silent fallbacks)

## RCA PR Template (Paste into PR description)

```text
### Root Cause
<first bad state, violated invariant, why it happened>

### Fix
<what makes this state impossible/visible>

### Tests
<new failing test before, passing after; coverage notes>

### Prevention
<guards, constraints, monitors, alerts>

### Rollout and Reversion
<deployment plan, kill switch that FAILS LOUDLY rather than bypassing>
```

## Debugging-First Requirements (Operationalized)

- Errors MUST be visible immediately
- Better to crash than continue with bad state
- Every error is a signal that needs investigation

These requirements operationalize [RULE: STRICT_FAIL_FAST] without changing its meaning.

## Forbidden Patterns (MUST NOT USE)

Never add defensive code that hides actual failures; visible errors are mandatory for debugging.

Scope note: the runtime validation patterns below are for external/boundary inputs only. Do not apply them to internal contracts (objects you construct inside our codebase and post-gateway domain objects) - that creates redundant runtime-typing noise (see [RULE: BOUNDARY_INTERNAL]).

```js
// BOUNDARY INPUT (request/external API/3rd-party callback): validate once, no fallbacks.
// FORBIDDEN: masks invalid state with a default
const masked = data?.nested?.property || "default";

// REQUIRED: fail loudly at the boundary, then use direct access
if (!data || !data.nested || typeof data.nested.property !== "string") {
  throw new Error("Missing required data.nested.property");
}
const boundaryValue = data.nested.property;

// INTERNAL CONTRACT (constructed in our code): do not add runtime typing/guards
const internalValue = internalData.nested.property;
```

```js
// All of these must be FORBIDDEN:

// 1. Optional chaining hiding errors
user?.profile?.settings?.theme;

// 2. Empty catches
try {
  doSomething();
} catch (e) {
  // Silent failure
}

// 3. Logical OR fallbacks masking errors
const config = userConfig || defaultConfig;

// 4. Nullish coalescing hiding undefined
const value = possiblyUndefined ?? "default";

// 5. Silent promise handling
promise.catch(() => {});

// 6. Defensive existence checks that hide bugs
if (object && object.method) {
  object.method();
}
// Instead of letting it fail if method doesn't exist
```

Why these are harmful:

- Optional chaining without checks hides undefined/null bugs and erases stack traces.
- Empty catch blocks swallow exceptions and destroy diagnostic signals.
- `||` / `??` fallbacks mask invalid state instead of surfacing it.
- Silent `.catch(() => {})` turns failures into ghost states that corrupt flows.
