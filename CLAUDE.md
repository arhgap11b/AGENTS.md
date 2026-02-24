# CORE PRINCIPLES FOR RELIABLE SOFTWARE & WORKFLOW

## AGENT TOOL SAFETY & WORKFLOW (ALWAYS LOADED)

- Before touching code, read `docs/ARCHITECTURE.md` and `docs/INDEX.md` to align with current flow, contracts, and project context.
- Mandatory scope reset: treat each user request (or scope shift) as a new task. Re-classify scope and re-run doc routing for the current request; do not assume previously consulted docs are still sufficient.
- After reading `docs/INDEX.md`, you MUST open the specific docs that cover the touched area(s) for THIS task (see the routing table below). If no doc exists, say so explicitly and proceed with minimal assumptions.

- Track (internally) the exact docs you consulted for THIS task (Preflight); output only if the user asks.
- Documentation sync is mandatory: any change that affects behavior/contracts/project reality must be reconciled with the relevant CURRENT docs under `docs/` in the same change.
- This applies to all doc layers, not just code/architecture:
- Architecture & code: `docs/architecture/system-overview.md` (and its "Related docs"), `docs/ARCHITECTURE.md`, `docs/architecture/adr/DECISIONS.md`
- Navigation: `docs/INDEX.md`
- `docs/rd/**` and `docs/_archive/**` are research/history: DO NOT touch these folders.

Command safety (mandatory): never run any lint/build command (e.g. `npm run lint`, `npm run build`, `next build`, `eslint`, etc.) without explicit user confirmation in the current conversation.

Token safety (mandatory): DO NOT load optional modules defensively. Load modules only when routing triggers are met.

---

## INTERNAL PREFLIGHT (DO NOT PRINT BY DEFAULT)

Before the first code change for a task, run a short `Preflight:` checklist **internally** for traceability (docs routing, SSOT, boundaries).

Output policy:

- Do **not** print `Preflight:` automatically (it wastes tokens and often breaks flow).
- Print `Preflight:` only if the user explicitly asks for it, or if coordination requires it (e.g. risky/security-sensitive work, or a refactor touching >5 files).
- If blocked on explicit user confirmation (lint/build commands) or required user input, ask for that directly; no need to print the full log.

Preflight fields (internal checklist):

- Language: match the user's request language.
- Task (one sentence): what you are changing and why.
- Touched areas: (UI | Backend/Data | Perf/Scale | Workflow/Debugging | Prompts | Other)
- Docs consulted: (explicit doc paths; minimum: `docs/ARCHITECTURE.md` + `docs/INDEX.md`)
- Loaded modules: (explicit file paths from the routing table, or `none`)
- Boundaries & gateways: what external I/O is involved and where it is parsed/validated
- Canonical state (SSOT): what is the source of truth and what is derived vs persisted
- Dependency direction: what stays pure and how I/O is injected
- Complexity & concurrency: Big-O and how I/O concurrency is bounded (batching/chunking)
- Data structures & space: justify `Array` vs `Set`/`Map` and keep space complexity bounded under load
- Applied rules: cite the exact `[RULE: ...]` tags that govern the approach

---

## FOUNDATION RULES (ALWAYS LOADED)

### [RULE: STRICT_FAIL_FAST] STRICT FAIL-FAST (STOP THE LINE)

The system is built on invariants. Any invalid state must stop execution immediately. It is better to crash than to continue with corrupted or misleading state.

FORBIDDEN:

- Empty catches or silent promise handling (`catch {}`, `.catch(() => {})`)
- Masking invalid state with fallbacks (`||`, `??`, default objects/arrays)
- Optional chaining used to hide bugs (`obj?.a?.b`) instead of boundary validation
- "Stability" band-aids: retries/timeouts/defaults/flags that bypass a failing path

REQUIRED:

- Validate boundary inputs once and fail loudly (`throw`, HTTP `400/500`).
- Defaults are allowed only when explicitly specified by the contract, and only in gateways (see [RULE: SPEC_DEFAULTS_ONLY]).
- Keep internal contracts strict: fix producers/types/tests; do not add runtime typing/sanitizers/normalizers to "make it work".
- Assume external systems can fail or return garbage; handle it at the boundary, not with silent defaults.
- When changing behavior/contracts, document the decision and the why close to the code (PR notes/ADR/comments).

### [RULE: RULE_PRECEDENCE] RULE PRECEDENCE (CONFLICT ARBITRATION)

When rules pull in different directions, resolve conflicts using this precedence order:

1. Safety & scope (user request, tool safety, docs sync)
2. Integrity & invariants (no hidden errors, no band-aids)
3. Architecture boundaries (gateways, dependency direction, explicit deps)
4. Domain correctness (SSOT, impossible states, side effects correctness, external-contract safety)
5. Performance & scale (Big-O, N+1, bounded queries/concurrency, streaming, UI perf)
6. Cosmetics/hygiene

If a lower-priority concern would violate a higher-priority rule, choose the higher-priority rule and document the tradeoff.

### [RULE: ROOT_CAUSE] ROOT-CAUSE-OR-NOTHING (NO BAND-AIDS)

We fix causes, not symptoms. Any "band-aid" that lets the system limp forward is prohibited. If we cannot guarantee integrity, the system must stop loudly until the root cause is eliminated.

What counts as a band-aid (FORBIDDEN):

- Adding defaults to bypass missing/invalid data
- Increasing timeouts/retries to "stabilize" flaky behavior
- Skipping validation "temporarily"
- Feature flags that merely avoid the failing path instead of fixing it
- "Just catch and continue" to keep the pipeline moving
- Adding fallbacks to avoid errors
- "Containment" justified only by `// TODO` / TTL / "fix later" comments

REQUIRED:

- If integrity cannot be guaranteed, stop loudly until the root cause is eliminated.

These patterns contradict [RULE: STRICT_FAIL_FAST] and are forbidden by this policy.

### [RULE: BOUNDARY_GATEWAYS] [RULE: BOUNDARY_INTERNAL] BOUNDARIES, GATEWAYS, INTERNAL TRUST (PARSE, DON'T VALIDATE)

Boundary inputs are unpredictable I/O. Parse/validate them once at a gateway, convert into canonical internal shapes, then trust the contract everywhere else.

Boundary (gateway required):

- `request.json()`, query/search params, cookies/headers
- `process.env` (runtime/deployment config)
- `localStorage` / `sessionStorage`
- external HTTP APIs + any network response payloads
- 3rd-party callbacks/webhooks
- database reads (SQL/ORM rows, JSON columns)
- cache reads (Redis/in-memory caches)
- queue messages (pub/sub/event bus payloads)

Internal contract (no runtime typing):

- objects/arrays/constants we construct inside our codebase
- derived objects from our own helpers/providers
- post-gateway domain objects (already parsed/validated)
- domain objects returned by repositories/adapters (post-mapping)

REQUIRED:

- Read/parse boundary inputs only in gateways (config modules, storage modules, API clients, repositories/adapters, route handlers).
- Throw loudly on any contract breach. No silent defaults. No `try/catch` that continues.
- After a value passes through the gateway, downstream code must not re-validate/normalize it.
- For DB/cache/queue reads: map/validate in repository/adapter code. The core must not consume raw rows/payloads.

FORBIDDEN:

- Deep reads of `process.env.*` / `localStorage.getItem(...)` from random components or pure domain functions.
- Scattered row/payload shape checks across the codebase instead of one repository/adapter mapping.
- Scattered runtime typing for internal contracts (`typeof`, `Array.isArray`, `hasOwnProperty` ladders).
- Internal `sanitize*` / `normalize*` / runtime "migration" layers that coerce invalid state.

FORBIDDEN (internal contract guards / runtime "typing"):

```js
// Internal contract: defined in our codebase.
if (!Array.isArray(ITEMS)) throw new Error("ITEMS must be an array");
for (const item of ITEMS) {
  if (typeof item.id !== "string")
    throw new Error("item.id must be a string");
}
```

REQUIRED (internal):

```js
// Trust the internal contract; enforce via types/tests, not runtime guards.
for (const item of ITEMS) {
  use(item.id);
}
```

FORBIDDEN (boundary fallbacks / swallowing errors):

```js
// process.env is a boundary input: do not mask missing/invalid values with fallbacks.
const debug =
  (process.env.NEXT_PUBLIC_ANALYTICS_DBG || "").toLowerCase() === "true";

// localStorage is a boundary input: do not hide invalid persisted values as "no value".
const raw = localStorage.getItem("app.someKey");
const parsedViaFallback = raw ? JSON.parse(raw) : null;

let parsed;
try {
  parsed = JSON.parse(localStorage.getItem("app.someKey"));
} catch {
  parsed = {};
}

// FORBIDDEN: parsing helpers that accept multiple shapes and hide contract mismatches.
const parseJsonValue = (value) => {
  if (value == null) return null;
  if (typeof value === "string") return JSON.parse(value);
  return value;
};
```

REQUIRED (boundary gateways):

```js
// Env gateway: validate once, export canonical config.
const dbUrl = process.env.DATABASE_URL;
if (!dbUrl) throw new Error("Missing DATABASE_URL");

const dbgRaw = process.env.NEXT_PUBLIC_ANALYTICS_DBG;
if (dbgRaw === undefined) throw new Error("Missing NEXT_PUBLIC_ANALYTICS_DBG");
const debug = dbgRaw.toLowerCase() === "true";

// localStorage gateway: explicit absence is allowed only if modeled; invalid JSON must throw.
const raw = localStorage.getItem("app.someKey");
const parsed = raw === null ? null : JSON.parse(raw);
```

### [RULE: EXPLICIT_DEPS] EXPLICIT DEPENDENCIES (NO HIDDEN GLOBALS)

Hidden dependencies are time bombs: every dependency must be explicit, isolated, and documented.

REQUIRED:

- Prefer explicit dependencies: pass what you need as arguments/props instead of reaching for globals.
- Keep impure dependencies explicit and isolated (time, randomness, `fetch`, storage, env).
- Document and centralize configuration (read `process.env` in a config module, not sprinkled across the codebase).

FORBIDDEN:

- Hidden globals/singletons/magic imports that make behavior depend on invisible state.
- Utilities that look pure but read config/storage/time internally.

### [RULE: SPEC_DEFAULTS_ONLY] DEFAULTS ONLY WHEN SPECIFIED (GATEWAYS ONLY)

Defaults are allowed only when the contract/spec explicitly defines them, and only in boundary gateways.

REQUIRED:

- Define defaults in the same gateway that parses the boundary input.
- Use explicit `null/undefined` checks and validate allowed values; throw on invalid data.
- Model optionality explicitly (`null`, discriminated union, optional field), not via `||/??`.

FORBIDDEN:

- `||` / `??` defaults in domain/core code.
- "Default injection" to hide missing required data.

```js
function parseMode(raw) {
  if (raw === null) return "safe"; // default is part of the spec
  if (raw === "safe" || raw === "fast") return raw;
  throw new Error("Invalid mode");
}
```

### [RULE: CANONICAL_REALITY] [RULE: ANTI_CORRUPTION_LAYER] ONE CANONICAL REALITY (NO RUNTIME VERSIONING)

We do not create "v2/v3", "new/old", "legacy", "migration" layers. When something changes, we update it as if it has always been that way.

WHY: versioned/legacy branches multiply states, create conflicting sources of truth, and turn product logic into archaeology. Our system must have one reality at any time.

CLARIFICATION (persisted data):

- This rule applies to application code (UI state, endpoints, prompts, internal contracts): no runtime compat branches, no "support old + new simultaneously".
- It does not apply to database schemas / long-lived persisted data and public API contracts: evolve those deterministically and, when a breaking change is unavoidable, use expand/contract (see [RULE: EXPAND_AND_CONTRACT]). The domain/core must still see one canonical model; any temporary dual-read/dual-write must be isolated to a boundary adapter/gateway and removed in the contract step.

REQUIRED:

- Inside the domain/core: one canonical model. No `v2/v3/new/old/legacy` branches.
- If an external system is legacy/versioned, adapt at the boundary into the canonical internal model.
- No version labels in names: components, hooks, files, endpoints, prompts must not be called `V2`, `V3`, `New`, `Old`, `Legacy`, `Next`, `Mk2`.
- Rewrite the canonical source of truth: change the existing artifact (prompt/config/component/endpoint) directly and remove the obsolete requirement.
- Prefer positive specs over negations: instead of "don't show X anymore", define exactly what is shown/output (a whitelist/format), so X is absent by construction.
- No runtime migrations/compat layers: do not introduce "migrate/normalize/sanitize/compat" code to support old shapes or old semantics. Remove the producer/requirement that created the old shape and keep only the new contract.
- Anti-corruption layer at boundaries: if an external system expects/returns legacy formats, translate at the boundary (adapters) into the canonical internal model. The core must not branch on versions.

FORBIDDEN (typical hidden-legacy patterns):

```js
// Naming that encodes history
export function composePromptV2() {}
export function composePromptV3() {}
export function composePromptLegacy() {}

// Version branching / runtime migrations
if (payload.version === 1) return migrateV1(payload);
if (payload.version === 2) return payload;

// Compatibility flags (two realities)
const prompt = flags.useNewPrompt ? NEW_PROMPT : OLD_PROMPT;
```

REQUIRED (single reality):

```js
export function composePrompt() {}
// One canonical internal shape; if external versions exist, adapt at the boundary.
```

Prompt example (most common):

```text
FORBIDDEN (patching history):
- "Previously we displayed X. Now do not display X in plain text."
- "Ignore earlier instructions that asked to show X."

REQUIRED (rewrite as if the old rule never existed):
- Remove the instruction that required X.
- Define the output format/sections that must appear (and only those).
```

### [RULE: NO_GOD_FILES] NO GOD FILES (1000 LINES MAX PER SOURCE FILE)

A file over 1000 lines is a design failure (mixed responsibilities, hidden coupling, review/debug pain). Line count is a proxy: a "short" file that mixes layers (DB I/O + domain transforms + UI) is also a god file.

Exception: `AGENTS.md` is allowed to exceed 1000 lines.

REQUIRED:

- If a change would push a file over 1000 lines, refactor first: extract unrelated concerns into dedicated modules (API client, domain transforms, step handlers, state machine, UI rendering).
- If a file is already >1000 lines, any meaningful change must include a refactor that brings it below 1000 (split by cohesive responsibilities / architectural layers; code that changes together must stay together).
- When splitting, decompose along strict axes: architectural layers (I/O adapters/gateways, pure domain/core, state orchestration, UI/presentation) or vertical feature slices. Code that changes together must stay together.
- Single responsibility per file: if logic is not directly about this file's primary concern, it belongs in a different file.

FORBIDDEN:

- "God" components/hooks/services that own everything end-to-end in one file.
- Growing an oversized file further instead of decomposing.
- Splitting into meaningless chunks (`*.part1.*`, `*.part2.*`) or dumping into blind bins (`utils.ts`, `helpers.ts`) just to bypass the limit.

### [RULE: NO_DEAD_CODE] NO DEAD CODE (DELETE FAILED ATTEMPTS)

If a solution was implemented and it did not work, then either fix it (if it is fundamentally correct and only needs small, concrete adjustments) or delete it (if the approach is wrong and the next solution no longer depends on it). Keeping abandoned code creates a graveyard of unclear logic and future bugs.

REQUIRED:

- Choose one path: finish the current approach or remove it entirely. No half-implemented "maybe later" code.
- If an approach is abandoned, remove it end-to-end: call sites, exports, files, flags, configs, prompts, docs that exist only for the failed attempt.
- Do not keep commented-out code "for history". History lives in Git/PR notes, not in production code.

Decision rule:

- If the failure indicates a wrong model/contract/design, delete the attempt and redesign.
- If the failure is a small fix away (clear, bounded changes), keep the code and complete it.

Examples:

```jsx
// FORBIDDEN: abandoned workaround left behind
useEffect(() => {
  setValue(expensiveDerivation(input));
}, [input]);

// REQUIRED: keep only the final approach (derive or compute where it belongs)
const value = useMemo(() => expensiveDerivation(input), [input]);
```

```text
PROMPTS

FORBIDDEN: keeping "patches" after the pivot
- Add "Do NOT output X" while still keeping the instruction that required X.
- Keep old prompt blocks under "legacy" comments.

REQUIRED: remove the old requirement; keep only the canonical instructions/output format.
```

### [RULE: EARLY_RETURN] GUARD CLAUSES (EARLY RETURN) OVER ARROW CODE

REQUIRED:

- Handle invalid/unauthorized/error cases first via `throw`/`return` and keep the happy path flat.

FORBIDDEN:

- Deep nesting that hides the happy path.

---

## ROUTING TABLE: OPTIONAL MODULES (LOAD ONLY IF TRIGGERED)

You MUST read the relevant module files in `agents/` BEFORE writing code if the task touches their triggers.
Constraint: DO NOT load modules defensively if triggers are not met.

| If the task touches / involves...                                                                                                | You MUST load and read:        |
| -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| UI/Web: React/Next.js components, hooks (`useEffect`), DOM, CSS, accessibility, animations, responsive design, client-side fetch | `agents/ui-web.md`             |
| Backend/Data: API routes, DB reads/writes, schemas, caching, side effects, workflows, payments/email/webhooks, domain state      | `agents/backend-data.md`       |
| Perf/Scale: large collections, search/filter, batching, pagination, OOM/memory, streaming exports, heavy I/O, background jobs    | `agents/perf-scale.md`         |
| Workflow/Debugging: incident response, RCA, investigations, error triage, flaky failures                                         | `agents/workflow-debugging.md` |

---

## UNIVERSAL GATE CHECKLIST (MUST PASS)

- [ ] Scope is respected (no unrelated cleanup) and docs are synced when behavior/contracts change.
- [ ] `Log:` is present and lists `Touched areas` and `Loaded modules`.
- [ ] No hidden errors: no empty catches, no silent promise handling, no fallback masking.
- [ ] Boundary I/O is handled in gateways; no deep reads or scattered parsing/normalization.
- [ ] Root cause is fixed; no TODO/TTL containment or "stability" band-aids.
- [ ] Files are cohesive and under 1000 lines (except `AGENTS.md`); no `.part1/.part2` or blind helper dumps.
- [ ] Failed attempts are removed or completed; no dead code.

---

## MASTER RULE INDEX (TRACEABILITY LOOKUP)

If a user or document cites a `[RULE: ...]` tag not defined in this file, load its definition from the locations below.

| Tag                                                                                                                                                                                                                                                                                                                                | Definition location      |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| `[RULE: STRICT_FAIL_FAST]`, `[RULE: RULE_PRECEDENCE]`, `[RULE: ROOT_CAUSE]`, `[RULE: BOUNDARY_GATEWAYS]`, `[RULE: BOUNDARY_INTERNAL]`, `[RULE: EXPLICIT_DEPS]`, `[RULE: SPEC_DEFAULTS_ONLY]`, `[RULE: CANONICAL_REALITY]`, `[RULE: ANTI_CORRUPTION_LAYER]`, `[RULE: NO_GOD_FILES]`, `[RULE: NO_DEAD_CODE]`, `[RULE: EARLY_RETURN]` | `AGENTS.md`              |
| `[RULE: UI_PERF]`, `[RULE: NO_EFFECT_CASCADES]`, `[RULE: COMPOSITION_OVER_CONFIG]`, `[RULE: ABORT_STALE_REQUESTS]`, `[RULE: A11Y]`, `[RULE: CLS]`, `[RULE: ANIMATIONS]`, `[RULE: RESPONSIVE]`, `[RULE: CSS_HYGIENE]`                                                                                                               | `agents/ui-web.md`       |
| `[RULE: SSOT]`, `[RULE: IMPOSSIBLE_STATES]`, `[RULE: PURE_CORE]`, `[RULE: DEPENDENCY_DIRECTION]`, `[RULE: CQS]`, `[RULE: DATA_MINIMIZATION]`, `[RULE: CACHE_SYMMETRY]`, `[RULE: SIDE_EFFECTS]`, `[RULE: DISTRIBUTED_TX]`, `[RULE: EXPAND_AND_CONTRACT]`                                                                            | `agents/backend-data.md` |
| `[RULE: BIG_O]`, `[RULE: N_PLUS_ONE]`, `[RULE: BOUNDED_CONCURRENCY]`, `[RULE: BOUNDED_QUERIES]`, `[RULE: STREAMING_IO]`                                                                                                                                                                                                            | `agents/perf-scale.md`   |

---

## GIT SAFETY

- Never use rollback commands like `git checkout`, `git restore`, `git reset --hard`.
- Any rollback must be manual and surgical (edit specific lines).
- If you need an older fragment, open the source and copy the needed block manually; do not use automated git rollback commands.

---

## REMINDER

- Never roll back completed work. If the result needs follow-up, bring changes to a working state or leave an explicit draft, but do not erase progress.
- Work sequentially: if a task touches multiple layers (backend, frontend), plan steps up front and record intermediate results.
- Respect colleagues' time: the user expects an outcome, not a "changed mind". Either finish or explicitly agree on a pause, but do not stop without an outcome.
- Do not touch unrelated/other people's changes: if `git status` shows changes outside the current task, do not "fix" or roll them back (out of scope).
- When multiple agents work in parallel, do not edit files already changed by another agent without explicit coordination.
- If you notice changes you did not make, treat this as normal parallel work by another agent. Do not touch those files and continue your task unless the user explicitly asks you to coordinate on them.
- No drive-by "cleanup": do not refactor styles/code "while you're here" outside scope, even if it seems "better".
- Before editing, document the contract/changes: define requirements and checks before touching code to avoid "rolling back" due to uncertainty.
- Structural changes to oversized files: if a file is slightly >1000 lines and can be split manually and obviously (minimal moves, no auto-slicers/scripts), it can be done without extra confirmation. If a file is heavily oversized (roughly >2000 lines) and/or splitting requires a slicer/script, mass moves, or re-ordering sections, confirm in chat first.

## LINE ENDINGS

- Both CRLF and LF are acceptable.
- Never rewrite files just to change line endings.
