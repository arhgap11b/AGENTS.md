# UI / Web

## [RULE: UI_PERF] REACT / NEXT.JS UI PERFORMANCE (ZERO UNNECESSARY RE-RENDERS)

UI performance is part of correctness.

REQUIRED:
- Architecture first, memoize second: reduce re-renders via state colocation (push state down to where it is used) and composition (`children` to isolate heavy subtrees) before reaching for `React.memo` / `useMemo`.
- Minimize client boundaries (Next.js): keep Server Components by default; add `"use client"` only for the smallest interactive islands.
- Referential stability at memoized boundaries:
  - Function props passed into a `React.memo` component (or through React Context) must be stable (`useCallback` or a module-level function).
  - Object/array props passed into a `React.memo` component (or through React Context) must be stable (`useMemo` or a module-level constant).
  - React Context `value` must be memoized; otherwise every provider render forces all consumers to re-render.
- Memoize the right boundaries: use `React.memo` for list items/cards/rows and other pure components that re-render frequently.
- Naming: memoization is an implementation detail. Do not encode it in component names (`MemoButton`, `MemoInlineHint`). Keep the canonical name and memoize inline/export.
- Never cheat hook deps: do not omit dependencies or disable `react-hooks/exhaustive-deps` to "stabilize" identity; fix the design instead.
- Keep a single source of truth: do not store duplicated/derived state; derive it (memoize only if it is expensive or crosses a component boundary).
- Lists: use stable keys (never `index` as `key`) to avoid remounts and wasted renders.

FORBIDDEN (perf bugs by default):
```jsx
// Unstable props => memoization is useless
<Child
  onSelect={(id) => setSelected(id)}
  filters={{ q }}
  items={items.map(mapToViewModel)}
/>

// Provider value recreated each render => all consumers re-render
<MyContext.Provider value={{ state, actions }}>
  {children}
</MyContext.Provider>
```

REQUIRED:
```jsx
const onSelect = useCallback((id) => setSelected(id), [setSelected]);
const filters = useMemo(() => ({ q }), [q]);
const viewItems = useMemo(() => items.map(mapToViewModel), [items]);

const contextValue = useMemo(() => ({ state, actions }), [state, actions]);

<MyContext.Provider value={contextValue}>
  <Child onSelect={onSelect} filters={filters} items={viewItems} />
</MyContext.Provider>
```

## [RULE: NO_EFFECT_CASCADES] NO EFFECT CASCADES (REACT)

FORBIDDEN: chains of local state sync via effects (A changes -> `useEffect` sets B -> re-render -> `useEffect` sets C). Derived state must not be stored just to keep it "in sync".

REQUIRED:
- Compute derived values during render (use `useMemo` only when needed).
- State transitions happen in event handlers.
- `useEffect` is for syncing with external systems only (API, subscriptions, DOM) and must be idempotent.

## [RULE: ABORT_STALE_REQUESTS] ABORT STALE CLIENT READS (SEARCH/FILTER)

REQUIRED: any input-driven read (search/filter/typeahead) must cancel stale in-flight requests (`AbortController`) or correlate responses (sequence IDs). Never let out-of-order responses overwrite newer state.

FORBIDDEN: firing `fetch` on every keystroke without cancellation/correlation.

## [RULE: COMPOSITION_OVER_CONFIG] COMPOSITION OVER CONFIG (NO PROP-FLAG EXPLOSION)

FORBIDDEN: mega-components extended via boolean/config props (`isSortable`, `hasSearch`, `hideHeader`, `disableHover`, ...), creating combinatorial complexity.

REQUIRED: extend behavior via composition (children, slots, subcomponents, render props). Keep containers generic; keep business logic in the composed parts.

## [RULE: A11Y] ACCESSIBILITY & SEMANTIC UI (KEYBOARD-FIRST)

Accessibility is part of correctness: the UI must be operable with keyboard and built on semantic elements.

REQUIRED:
- Use native interactive elements: `<button>`, `<a>`, `<input>`, `<select>`, `<textarea>` (not clickable `<div>`/`<span>`).
- Every interactive control must have an accessible name (visible text first; `aria-label` only when there is no visible label).
- Forms: every input has a real `<label>` (or `aria-labelledby` when layout demands it). Placeholders are not labels.
- Overlays (dialogs/menus): focus moves inside on open, is trapped while open, returns to the trigger on close; `Escape` closes.
- Never remove focus outlines without providing an accessible replacement (focus must stay visible).

FORBIDDEN:
```jsx
// Clickable non-semantic element
<div onClick={onSave}>Save</div>

// Placeholder as label
<input placeholder="Email" />
```

REQUIRED:
```jsx
<button type="button" onClick={onSave}>
  Save
</button>

<label htmlFor="email">Email</label>
<input id="email" name="email" />
```

## [RULE: CLS] LAYOUT STABILITY (CLS) (NO UI JUMPS)

Layout shifts are UX bugs. The UI must not "jump" while loading.

REQUIRED:
- Media must reserve space: use `next/image` and set `width`+`height`, or use `fill` inside a container with an explicit size/aspect ratio.
- Responsive images must set `sizes` so the browser downloads the correct variant.
- Loading/skeleton placeholders must match the final block size (same height/aspect ratio).

FORBIDDEN:
```jsx
<img src="/hero.jpg" />
<Image src="/hero.jpg" fill alt="Hero" />
```

REQUIRED:
```jsx
<Image
  src="/hero.jpg"
  width={1280}
  height={720}
  sizes="(max-width: 768px) 100vw, 768px"
  alt="Hero"
/>
```

## [RULE: ANIMATIONS] ANIMATIONS (TRANSFORM/OPACITY ONLY)

Animations must not trigger layout thrash.

REQUIRED:
- Animate only `transform` and `opacity` for motion (avoid layout-changing properties).
- Respect `prefers-reduced-motion`: reduce/disable animations for users who request it.

FORBIDDEN:
```css
.panel {
  transition: height 200ms ease;
}
```

REQUIRED:
```css
.panel {
  transition: transform 200ms ease, opacity 200ms ease;
}

@media (prefers-reduced-motion: reduce) {
  .panel {
    transition: none;
  }
}
```

## [RULE: RESPONSIVE] RESPONSIVE UI (MOBILE-FIRST)

Any UI change must work on mobile from the start (not as a later "adaptation").

REQUIRED:
- Think mobile-first: design and verify on a small viewport first, then scale up.
- Avoid fixed layout widths/heights; prefer flex/grid with `max-width`, `min()`, `clamp()` where appropriate.
- Controls must be comfortable to tap; avoid cramped touch targets.

FORBIDDEN:
```css
.container {
  width: 1200px;
}
```

REQUIRED:
```css
.container {
  width: 100%;
  max-width: 1200px;
}
```

## [RULE: CSS_HYGIENE] CSS HYGIENE (REUSE, DON'T DUPLICATE)

Avoid CSS sprawl: reuse existing classes and media-query blocks instead of creating near-duplicates.

REQUIRED:
- Before adding a new class, search for an existing one that already expresses the same intent. Prefer reuse over one-off classes.
- If you need to extend a style, modify the existing class (or add a variant) instead of creating a new class for a single property.
- For media queries, do not duplicate identical breakpoints just to add one rule. Put the new rule inside the existing breakpoint block.
- If a class/breakpoint truly does not exist, create it next to the related styles (same file/section) so it stays discoverable.

FORBIDDEN:
```css
/* Same class re-created elsewhere (different file/section) for one tweak */
/* buttons.css */
.primaryButton {
  padding: 12px 16px;
  border-radius: 12px;
}
/* checkout.css */
.primaryButton {
  min-width: 220px; /* one-off override */
}

/* Duplicate class for one property */
.primaryButton {
  padding: 12px 16px;
  border-radius: 12px;
}
.primaryButtonTight {
  padding: 10px 16px; /* one-off */
  border-radius: 12px;
}

/* Duplicate breakpoint blocks */
@media (max-width: 768px) {
  .title {
    font-size: 20px;
  }
}
@media (max-width: 768px) {
  .subtitle {
    margin-top: 8px;
  }
}
```

REQUIRED:
```css
/* One canonical class + variants live together */
.primaryButton {
  padding: 12px 16px;
  border-radius: 12px;
}
.primaryButton.isCheckout {
  min-width: 220px;
}
.primaryButton.isTight {
  padding: 10px 16px;
}

@media (max-width: 768px) {
  .title {
    font-size: 20px;
  }
  .subtitle {
    margin-top: 8px;
  }
}
```

## UI Add-on Checklist (Apply Only If UI Is Touched)

- React lists use stable keys (never `index` as `key`).
- Memoization boundaries are justified and props/context values are referentially stable where needed.
- No local-state sync chains via `useEffect`.
- Input-driven reads cancel stale requests or correlate responses.
- UI is semantic and keyboard-accessible.
- No layout shifts (reserve space, correct `sizes`).
- Animations use transform/opacity and respect reduced motion.
- Responsive behavior verified on small viewports first.
- No CSS duplication (classes and breakpoints reused).
