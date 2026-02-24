# AGENTS.md

Battle-tested principles for AI coding agents. Drop into any project — the agent reads these rules before touching your code.

## How it works

```
CLAUDE.md          ← main rules file (loaded by Claude Code / Cursor / etc.)
AGENTS.md          ← symlink → CLAUDE.md (for tools that look for AGENTS.md)
agents/            ← optional domain modules (loaded on demand)
```

The agent loads `CLAUDE.md` (aka `AGENTS.md`) on every session. The `agents/` modules are loaded **only when the task touches their domain** — this saves tokens and keeps context focused.

## What's inside

### CLAUDE.md — Foundation (always loaded)

Core rules that apply to **every** code change:

| Rule | What it enforces |
|------|-----------------|
| **STRICT_FAIL_FAST** | Crash early, crash loudly. No empty catches, no silent fallbacks, no `\|\|`/`??` defaults that mask invalid state. |
| **ROOT_CAUSE** | Fix causes, not symptoms. Band-aids (retries, timeouts, feature flags that skip the bug) are forbidden. |
| **BOUNDARY_GATEWAYS** | Parse & validate external input (env, localStorage, API responses, DB rows) once at the boundary. Trust internal contracts — no scattered `typeof` checks. |
| **EXPLICIT_DEPS** | No hidden globals. Pass dependencies explicitly. Read `process.env` in a config module, not from random files. |
| **SPEC_DEFAULTS_ONLY** | Defaults are allowed only when the spec defines them, and only in boundary gateways. No `??` "just in case". |
| **CANONICAL_REALITY** | One version of truth. No `v2/v3/legacy/new` branches in code. Change the thing as if it was always that way. |
| **NO_GOD_FILES** | Max 1000 lines per source file. Split by responsibility, not by `part1/part2`. |
| **NO_DEAD_CODE** | Delete failed attempts. No commented-out code, no "maybe later" blocks. Git is the history. |
| **EARLY_RETURN** | Guard clauses first, happy path flat. No arrow code. |
| **RULE_PRECEDENCE** | When rules conflict: safety > integrity > architecture > domain > performance > cosmetics. |

Plus: workflow rules (scope reset per task, docs sync, preflight checklist, git safety, no drive-by cleanups).

### agents/ui-web.md — UI & Frontend

Loaded when the task touches React/Next.js components, hooks, CSS, animations, accessibility.

| Rule | What it enforces |
|------|-----------------|
| **UI_PERF** | Zero unnecessary re-renders. State colocation, stable refs at memo boundaries, memoized context values. |
| **NO_EFFECT_CASCADES** | No chains of `useEffect` syncing local state. Derive during render; effects are for external systems only. |
| **COMPOSITION_OVER_CONFIG** | No mega-components with boolean prop explosions. Extend via children/slots/subcomponents. |
| **ABORT_STALE_REQUESTS** | Search/filter must cancel stale in-flight requests. No out-of-order response overwrites. |
| **A11Y** | Semantic HTML, keyboard-first. `<button>` not `<div onClick>`. Real labels, focus traps in modals. |
| **CLS** | No layout shifts. Reserve space for images/skeletons. Set `width`+`height` or `sizes`. |
| **ANIMATIONS** | Transform/opacity only. Respect `prefers-reduced-motion`. |
| **RESPONSIVE** | Mobile-first. No fixed widths. Flex/grid with `max-width`/`clamp()`. |
| **CSS_HYGIENE** | Reuse classes and breakpoint blocks. No near-duplicate one-off styles. |

### agents/backend-data.md — Backend & Data

Loaded when the task touches API routes, DB, schemas, caching, side effects, payments, webhooks.

| Rule | What it enforces |
|------|-----------------|
| **SSOT** | Single source of truth. No duplicated state. One canonical source, everything else derived. |
| **IMPOSSIBLE_STATES** | Discriminated unions / state machines over boolean soup. `isLoading/isError/isSuccess` that can contradict = forbidden. |
| **PURE_CORE** | Business logic is pure (no DB, fetch, Date.now). Inject I/O from the shell. |
| **CQS** | Command-query separation. A function either reads or writes, never both. |
| **DATA_MINIMIZATION** | Strict projections. No `SELECT *`. No raw ORM entities in API responses. |
| **CACHE_SYMMETRY** | If you add a cache, you ship invalidation in the same change. No TTL-only consistency for mutable data. |
| **SIDE_EFFECTS** | Idempotent, atomic, explicit. Handle double-clicks, retries, race conditions. No fire-and-forget writes. |
| **DISTRIBUTED_TX** | DB + external I/O = partial failure risk. Use outbox/saga/compensation. No blind `await db; await stripe;`. |
| **EXPAND_AND_CONTRACT** | Zero-downtime schema changes. Add new → migrate → remove old. No one-shot destructive renames. |

### agents/perf-scale.md — Performance & Scale

Loaded when the task involves large collections, batching, pagination, streaming, memory concerns.

| Rule | What it enforces |
|------|-----------------|
| **BIG_O** | O(1) lookups via Map/Set, not nested `.find()` inside `.map()`. No spread accumulation in loops. |
| **N_PLUS_ONE** | No `await fetch()` inside loops. Batch I/O, then map in memory. |
| **BOUNDED_CONCURRENCY** | No unbounded `Promise.all(bigArray.map(fetch))`. Chunk or use a limiter. |
| **BOUNDED_QUERIES** | Never load unbounded collections. Default to pagination/cursors/LIMIT. |
| **STREAMING_IO** | Large I/O (CSV, big JSON, LLM responses) must stream. O(1) space relative to input size. |

### agents/workflow-debugging.md — Incidents & Debugging

Loaded during incident response, RCA, error triage, flaky failure investigation.

Provides: RCA operating procedure (7 steps), definition of done for fixes, forbidden band-aid patterns with code examples, reviewer checklist, PR template for root cause documentation.

## Recommended project structure

For best results, also create these docs in your project:

```
docs/
  ARCHITECTURE.md    ← system overview, boundaries, data flow
  INDEX.md           ← navigation map to all docs
  architecture/
    adr/             ← architecture decision records
  rd/                ← research & history (read-only)
  _archive/          ← archived docs (read-only)
```

The agent will read `docs/ARCHITECTURE.md` and `docs/INDEX.md` before touching code — this gives it project context and prevents guesswork.

## Usage

### Claude Code / Claude

Copy `CLAUDE.md`, `AGENTS.md` (symlink), and `agents/` into your project root. Done.

### Cursor / other AI editors

Most tools look for `AGENTS.md` or `CLAUDE.md` in the project root. The symlink ensures both names resolve to the same file.

### Customize

- Edit rules in `CLAUDE.md` to match your stack/preferences
- Add/remove modules in `agents/` for your domain
- The routing table in `CLAUDE.md` controls when each module loads

## License

Use however you want. No attribution required.
