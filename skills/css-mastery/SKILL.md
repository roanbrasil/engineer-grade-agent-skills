---
name: css-mastery
description: Expert CSS guidance — invoked when writing or reviewing CSS/SCSS for layout, responsive design, animations, modern CSS features, or performance.
---

# CSS Mastery

## The Box Model

Every element is a rectangular box. Two sizing modes control how `width` and `height` are calculated:

```
content-box (default)
┌──────────────────────────────┐
│         margin               │
│  ┌───────────────────────┐   │
│  │       border          │   │
│  │  ┌─────────────────┐  │   │
│  │  │    padding      │  │   │
│  │  │  ┌───────────┐  │  │   │
│  │  │  │  content  │  │  │   │  ← width/height apply here only
│  │  │  └───────────┘  │  │   │
│  │  └─────────────────┘  │   │
│  └───────────────────────┘   │
└──────────────────────────────┘

border-box
┌──────────────────────────────┐
│         margin               │
│  ┌───────────────────────┐   │  ← width/height apply here
│  │       border          │   │
│  │  ┌─────────────────┐  │   │
│  │  │    padding      │  │   │
│  │  │  ┌───────────┐  │  │   │
│  │  │  │  content  │  │  │   │
│  │  │  └───────────┘  │  │   │
│  │  └─────────────────┘  │   │
│  └───────────────────────┘   │
└──────────────────────────────┘
```

**Always apply globally:**

```css
*, *::before, *::after {
  box-sizing: border-box;
}
```

### Margin Collapsing

Vertical margins collapse between adjacent block elements — the larger margin wins.

```
div { margin-bottom: 24px; }
p   { margin-top: 16px; }
/* Gap between them = 24px, not 40px */
```

Collapsing does NOT happen when:
- Elements are in a flex or grid container
- The parent has `overflow: hidden/auto` (creates a Block Formatting Context)
- There is padding or border between parent and child
- Elements are absolutely or floatingly positioned

---

## Layout Systems

### Flexbox — One-Dimensional

```
Main axis (row):  ←────────────────────→
                  [item] [item] [item]

Main axis (col):  ↑
                  [item]
                  [item]
                  ↓
```

```css
.container {
  display: flex;
  flex-direction: row;          /* row | column | row-reverse | column-reverse */
  justify-content: space-between; /* main axis alignment */
  align-items: center;          /* cross axis alignment */
  flex-wrap: wrap;              /* allow wrapping */
  gap: 1rem;                    /* space between items; prefer over margin */
}

.item {
  flex: 1 1 200px; /* grow shrink basis */
  /* flex-grow: 1  — take available space */
  /* flex-shrink: 1 — shrink if needed */
  /* flex-basis: 200px — preferred size before grow/shrink */
}
```

**Common Flexbox use cases:**
- Navigation bars: `justify-content: space-between`
- Centering: `justify-content: center; align-items: center`
- Card rows with equal height
- Form label + input pairs

### CSS Grid — Two-Dimensional

```
grid-template-columns: 1fr 2fr 1fr

┌──────┬────────────┬──────┐
│  1fr │    2fr     │  1fr │
│      │            │      │
├──────┼────────────┼──────┤
│      │            │      │
└──────┴────────────┴──────┘
```

```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: auto;
  gap: 1.5rem;
}

/* Named areas */
.layout {
  display: grid;
  grid-template-areas:
    "header header header"
    "sidebar main   main"
    "footer footer footer";
  grid-template-columns: 240px 1fr 1fr;
}

.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main    { grid-area: main; }

/* Spanning */
.wide-card {
  grid-column: 1 / span 2;
  grid-row: 2 / 4;
}

/* Responsive without media queries */
.gallery {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  /* auto-fill: creates as many columns as fit (keeps empty tracks) */
  /* auto-fit: collapses empty tracks — items stretch to fill */
}
```

**auto-fill vs auto-fit:**

```
auto-fill (3 items, 5 columns fit):
[item][item][item][    ][    ]

auto-fit (3 items, 5 columns fit):
[  item  ][  item  ][  item  ]
```

### Positioning

```
static   — normal flow (default)
relative — offset from normal position; creates stacking context with z-index
absolute — removed from flow; positioned relative to nearest positioned ancestor
fixed    — relative to viewport; stays on scroll
sticky   — hybrid: relative until threshold, then fixed within scroll container
```

```css
/* Overlay pattern */
.parent {
  position: relative;
}
.overlay {
  position: absolute;
  inset: 0; /* shorthand for top/right/bottom/left: 0 */
}

/* Sticky header */
.header {
  position: sticky;
  top: 0;
  z-index: 100;
}
```

**Stacking context** is created by: `position` + `z-index` (not auto), `opacity < 1`, `transform`, `filter`, `will-change`, `isolation: isolate`.

### Multi-Column Layout

```css
.article-body {
  column-count: 3;
  /* or */
  column-width: 20ch; /* browser picks count */
  column-gap: 2rem;
  column-rule: 1px solid #e2e8f0; /* like border between columns */
}

/* Prevent element from breaking across columns */
.figure {
  break-inside: avoid;
}
```

---

## Responsive Design

### Mobile-First

Start with the smallest viewport. Layer complexity upward:

```css
/* Base: mobile */
.card {
  display: block;
  padding: 1rem;
}

/* Tablet */
@media (min-width: 768px) {
  .card {
    display: flex;
    padding: 1.5rem;
  }
}

/* Desktop */
@media (min-width: 1200px) {
  .card {
    padding: 2rem;
  }
}
```

### Fluid Sizing with clamp()

```css
/* clamp(minimum, preferred, maximum) */
h1 {
  font-size: clamp(1.75rem, 4vw, 3rem);
}

.container {
  width: clamp(320px, 90%, 1200px);
  margin-inline: auto;
}

.hero-gap {
  padding-block: clamp(3rem, 8vh, 8rem);
}
```

### Container Queries

Component responds to its container size, not the viewport:

```css
.card-wrapper {
  container-type: inline-size;
  container-name: card; /* optional; required for named @container */
}

@container card (min-width: 400px) {
  .card {
    display: flex;
    flex-direction: row;
  }
}

@container (min-width: 600px) {
  .card__image {
    width: 200px;
    flex-shrink: 0;
  }
}
```

### CSS Custom Properties (Variables)

```css
:root {
  /* Design tokens */
  --color-brand: #0066cc;
  --color-brand-hover: #0052a3;
  --spacing-base: 0.25rem;
  --radius-md: 0.5rem;
  --font-sans: 'Inter', system-ui, sans-serif;
}

.button {
  background: var(--color-brand);
  padding: calc(var(--spacing-base) * 3) calc(var(--spacing-base) * 6);
  border-radius: var(--radius-md);
  /* Fallback value */
  color: var(--color-text-on-brand, white);
}

/* Override in component scope */
.button--danger {
  --color-brand: #dc2626;
  --color-brand-hover: #b91c1c;
}
```

---

## Modern CSS Features

### Mathematical Functions

```css
/* calc(): mixed units */
.sidebar {
  width: calc(100% - 2rem);
}

/* min(): smallest value wins */
.container {
  width: min(100%, 1200px);
}

/* max(): largest value wins */
.text {
  font-size: max(1rem, 2.5vw); /* never smaller than 1rem */
}

/* clamp(min, val, max) */
.heading {
  font-size: clamp(1.5rem, 3vw + 1rem, 4rem);
}
```

### Logical Properties (RTL/LTR Support)

```css
/* Instead of:          Use: */
margin-left:            margin-inline-start
margin-right:           margin-inline-end
margin-top:             margin-block-start
margin-bottom:          margin-block-end
padding-left/right:     padding-inline
padding-top/bottom:     padding-block
border-left:            border-inline-start
width:                  inline-size
height:                 block-size
```

### Modern Selectors

```css
/* :is() — matches any selector in the list; takes highest specificity of the list */
:is(h1, h2, h3) + p {
  margin-top: 0.5rem;
}

/* :where() — same but specificity is always 0; great for reset styles */
:where(h1, h2, h3, h4) {
  line-height: 1.2;
}

/* :has() — parent selector; apply styles based on descendants */
.card:has(img) {
  padding-top: 0;
}

.form-field:has(input:invalid) label {
  color: red;
}

/* :not() */
li:not(:last-child) {
  border-bottom: 1px solid #e2e8f0;
}
```

### Cascade Layers

```css
/* Declare layer order — earlier = lower priority */
@layer reset, base, components, utilities;

@layer reset {
  * { margin: 0; padding: 0; }
}

@layer base {
  a { color: var(--color-brand); }
}

@layer components {
  .button { /* ... */ }
}

@layer utilities {
  .sr-only { /* screen reader only */ }
}

/* Unlayered styles always win over layered ones */
```

### Native CSS Nesting

```css
/* No Sass needed */
.card {
  padding: 1rem;
  border-radius: 0.5rem;

  &:hover {
    box-shadow: 0 4px 12px rgb(0 0 0 / 0.1);
  }

  & .card__title {
    font-size: 1.25rem;
  }

  @media (min-width: 768px) {
    padding: 1.5rem;
  }
}
```

### @scope

```css
/* Limit styles to a subtree; avoids deep selectors */
@scope (.card) to (.card__footer) {
  p { color: var(--color-text-secondary); }
}
```

---

## Specificity and the Cascade

```
Specificity order (high → low):
  1. !important (avoid in your own code)
  2. Inline styles (style="")
  3. ID selectors (#id)
  4. Class, attribute, pseudo-class (.class, [attr], :hover)
  5. Element, pseudo-element (div, ::before)

Specificity values (A, B, C):
  #nav .item:hover   → (0, 1, 1, 1) → wins over
  div.item:hover     → (0, 0, 1, 1)
```

**Use `@layer` to manage specificity** without specificity wars. Within a layer, normal specificity applies; layers themselves create a priority order.

---

## Animations and Transitions

```css
/* Transitions: simple state changes */
.button {
  background: var(--color-brand);
  transition: background 200ms ease, transform 150ms ease;
}

.button:hover {
  background: var(--color-brand-hover);
  transform: translateY(-1px);
}

/* Keyframe animations */
@keyframes slide-in {
  from {
    opacity: 0;
    transform: translateY(16px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.toast {
  animation: slide-in 300ms ease forwards;
  /* animation-fill-mode: forwards — keeps final state */
}

/* Respect user motion preferences */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

**`will-change` — use sparingly:**

```css
/* Only when you've profiled and confirmed benefit */
.animated-panel {
  will-change: transform; /* hints browser to create GPU layer */
}

/* Remove after animation */
.animated-panel.done {
  will-change: auto;
}
```

---

## Performance

```css
/* GPU-accelerated properties (no layout/paint reflow) */
.use-these-for-animation {
  transform: translateX(100px) scale(1.1);
  opacity: 0.5;
}

/* Avoid animating these (triggers layout) */
.avoid-animating {
  top: 100px;        /* → use transform: translateY() */
  left: 100px;       /* → use transform: translateX() */
  width: 200px;      /* → use transform: scaleX() */
  margin-top: 10px;  /* → use transform: translateY() */
}

/* Contain rendering scope */
.widget {
  contain: layout paint; /* changes inside don't affect outside */
}

/* Or for strict isolation */
.card {
  contain: strict;
}
```

**Critical CSS pattern:**

```html
<!-- Inline critical above-fold styles -->
<style>
  /* Only what's needed for first paint */
  body { margin: 0; font-family: system-ui; }
  .hero { /* ... */ }
</style>

<!-- Defer non-critical stylesheet -->
<link rel="preload" href="styles.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="styles.css"></noscript>
```

---

## Anti-Patterns

- **Magic numbers without comment**: `top: 37px` — why 37? Document it.
- **Deeply nested selectors**: `.page .container .section .card .title span { }` — high specificity, fragile
- **Overriding third-party with `!important` everywhere** — use `@layer` to slot third-party below your styles
- **Fixed px for font sizes** — use `rem` so users' browser font size preferences are respected
- **Not using `box-sizing: border-box` globally** — inconsistent sizing math everywhere
- **Using `margin` instead of `gap` in flex/grid** — gap is direction-agnostic and doesn't collapse
- **No `:focus-visible` styles** — keyboard users can't see where they are

---

## Quick Reference Checklist

- [ ] `box-sizing: border-box` applied globally
- [ ] Custom properties defined for all tokens (color, spacing, type)
- [ ] Flexbox for 1D layout; Grid for 2D; never floats for layout
- [ ] Mobile-first media queries with `min-width`
- [ ] `clamp()` for fluid type and spacing
- [ ] Container queries for component-responsive layouts
- [ ] `@layer` to manage cascade across design system layers
- [ ] GPU-only properties (`transform`, `opacity`) for animations
- [ ] `prefers-reduced-motion` respected
- [ ] Logical properties for RTL compatibility
- [ ] No magic numbers; no `!important` in own code
