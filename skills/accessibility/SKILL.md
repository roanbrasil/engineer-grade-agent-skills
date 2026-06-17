---
name: accessibility
description: Web accessibility (a11y) — invoked when building UI components, reviewing code for WCAG compliance, fixing keyboard navigation, ARIA usage, color contrast, or screen reader support.
---

# Web Accessibility (a11y) — WCAG 2.1 AA

## WCAG Principles (POUR)

```
PERCEIVABLE    Information must be presentable to users in ways they can perceive.
               → Alt text, captions, sufficient contrast, adaptable layouts

OPERABLE       UI components and navigation must be operable.
               → Keyboard accessible, no seizure-inducing content, enough time

UNDERSTANDABLE Information and UI operation must be understandable.
               → Readable text, predictable behavior, input assistance

ROBUST         Content must be interpreted reliably by assistive technologies.
               → Valid HTML, correct ARIA, compatible with current/future AT
```

---

## Semantic HTML (The Foundation)

Semantic HTML gives meaning to structure. Screen readers, search engines, and browser accessibility trees all depend on it.

```html
<!-- BAD: meaningless divs -->
<div class="header">
  <div class="nav">
    <div onclick="go('/home')">Home</div>
  </div>
</div>
<div class="main">
  <div class="article">
    <div class="title">My Article</div>
  </div>
</div>

<!-- GOOD: semantic structure -->
<header>
  <nav aria-label="Main navigation">
    <ul>
      <li><a href="/home">Home</a></li>
    </ul>
  </nav>
</header>
<main>
  <article>
    <h1>My Article</h1>
    <p>Content...</p>
  </article>
</main>
<footer>
  <p>&copy; 2025 Company</p>
</footer>
```

### Heading Hierarchy

```html
<!-- BAD: skipped levels, multiple h1 -->
<h1>Page Title</h1>
<h3>Section</h3>  <!-- skipped h2 -->
<h1>Another Title</h1>  <!-- second h1 -->

<!-- GOOD: one h1, logical nesting -->
<h1>Annual Report 2025</h1>
  <h2>Financial Overview</h2>
    <h3>Revenue</h3>
    <h3>Expenses</h3>
  <h2>Team Updates</h2>
    <h3>Engineering</h3>
```

Screen reader users navigate by headings — the outline must make sense when read in isolation.

### Interactive Elements

```html
<!-- RULE: <button> for actions, <a> for navigation -->

<!-- BAD: div pretending to be a button -->
<div class="btn" onclick="submitForm()">Submit</div>
<!-- Missing: keyboard focus, Enter/Space handling, role, ARIA state -->

<!-- GOOD: native button -->
<button type="submit">Submit</button>
<!-- Gets: focus, keyboard, role="button", activated by Enter+Space -->

<!-- GOOD: link for navigation -->
<a href="/checkout">Go to checkout</a>
<!-- Gets: focus, keyboard, role="link", activated by Enter -->

<!-- Button with only an icon: needs accessible label -->
<button aria-label="Close dialog">
  <svg aria-hidden="true" focusable="false">...</svg>
</button>
```

### Forms

```html
<!-- Every input needs a label -->

<!-- Option 1: explicit label (preferred) -->
<label for="email">Email address</label>
<input type="email" id="email" name="email" required autocomplete="email">

<!-- Option 2: wrapping label -->
<label>
  Email address
  <input type="email" name="email">
</label>

<!-- Option 3: aria-label (when visible label isn't possible) -->
<input type="search" aria-label="Search products" placeholder="Search...">

<!-- Option 4: aria-labelledby (reference existing text) -->
<h2 id="billing-heading">Billing Address</h2>
<fieldset aria-labelledby="billing-heading">
  <legend class="sr-only">Billing Address</legend>
  ...
</fieldset>

<!-- Error messages linked to input -->
<label for="phone">Phone number</label>
<input
  type="tel"
  id="phone"
  aria-describedby="phone-hint phone-error"
  aria-invalid="true"
>
<p id="phone-hint" class="hint">Include country code (e.g. +1)</p>
<p id="phone-error" role="alert" class="error">
  Please enter a valid phone number
</p>
```

### Tables

```html
<table>
  <caption>Q3 Sales by Region</caption>
  <thead>
    <tr>
      <th scope="col">Region</th>
      <th scope="col">Revenue</th>
      <th scope="col">Growth</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">North America</th>
      <td>$4.2M</td>
      <td>+12%</td>
    </tr>
  </tbody>
  <tfoot>
    <tr>
      <th scope="row">Total</th>
      <td>$11.8M</td>
      <td>+9%</td>
    </tr>
  </tfoot>
</table>
```

---

## ARIA

**First rule of ARIA: don't use ARIA if native HTML does it.**

```
Native HTML covers:
  <button>    role="button", keyboard, activation
  <a href>    role="link", keyboard
  <input type="checkbox">  role="checkbox", checked state
  <nav>       role="navigation"
  <main>      role="main"
  <h1>–<h6>  role="heading", aria-level

Use ARIA for:
  Custom widgets (combobox, tree, datepicker, slider)
  Dynamic content announcements (live regions)
  Relationships between elements (labelledby, describedby)
  States not expressible in HTML (aria-expanded, aria-selected)
```

### ARIA Reference Card

```html
<!-- Roles: override or add semantic meaning -->
<div role="dialog" aria-modal="true" aria-labelledby="dialog-title">
<div role="alert">           <!-- assertive live region; for urgent messages -->
<div role="status">          <!-- polite live region; for non-urgent updates -->
<ul role="listbox">          <!-- for custom select/combobox -->
<li role="option">           <!-- item in listbox -->

<!-- Labels -->
<button aria-label="Close">✕</button>                    <!-- direct text label -->
<input aria-labelledby="title-id hint-id">               <!-- reference elements -->
<input aria-describedby="description-id">                <!-- supplemental info -->

<!-- States -->
<button aria-expanded="false">Menu</button>              <!-- toggle state -->
<li role="option" aria-selected="true">Option A</li>     <!-- selection -->
<input type="checkbox" aria-checked="mixed">             <!-- tristate checkbox -->
<button aria-pressed="true">Bold</button>                <!-- toggle button -->
<div aria-busy="true">Loading...</div>                   <!-- loading state -->
<input aria-invalid="true" aria-describedby="err-id">   <!-- validation error -->
<button aria-disabled="true">Submit</button>             <!-- disabled + focusable -->

<!-- Hiding from AT -->
<svg aria-hidden="true">...</svg>                        <!-- decorative -->
<span aria-hidden="true">•</span>                       <!-- visual separator -->
```

### Live Regions

```html
<!-- Polite: announced when user is idle; for status messages -->
<div aria-live="polite" aria-atomic="true" class="sr-only">
  <!-- Dynamically inject: "3 items added to cart" -->
</div>

<!-- Assertive: announced immediately; only for urgent errors -->
<div role="alert">
  <!-- Inject: "Session expired. Please sign in again." -->
</div>
```

```tsx
// React: live region hook
function useLiveAnnouncement() {
  const [message, setMessage] = useState('');

  const announce = useCallback((text: string) => {
    setMessage('');        // Clear first to re-trigger announcement
    setTimeout(() => setMessage(text), 50);
  }, []);

  const LiveRegion = () => (
    <div
      role="status"
      aria-live="polite"
      aria-atomic="true"
      className="sr-only"
    >
      {message}
    </div>
  );

  return { announce, LiveRegion };
}

// Usage in cart:
const { announce, LiveRegion } = useLiveAnnouncement();

function handleAddToCart(item: Item) {
  addToCart(item);
  announce(`${item.name} added to cart. Cart has ${cartCount + 1} items.`);
}
```

---

## Keyboard Navigation

### Focus Management Rules

```
Tab key:    Move forward through interactive elements
Shift+Tab:  Move backward
Enter:      Activate links, submit forms, activate buttons
Space:      Activate buttons, toggle checkboxes
Escape:     Close dialogs, menus, tooltips
Arrow keys: Navigate within composite widgets (menu, listbox, tabs, slider)
Home/End:   Jump to first/last item in a widget
```

### Focus Indicators

```css
/* BAD: removing focus indicator */
:focus { outline: none; }
*:focus { outline: 0; }

/* GOOD: style it, don't remove it */
:focus-visible {
  outline: 2px solid #3b82f6;
  outline-offset: 3px;
  border-radius: 2px;
}

/* Hide outline for mouse users (only show for keyboard) */
:focus:not(:focus-visible) {
  outline: none;
}

/* High-contrast mode compatible */
@media (forced-colors: active) {
  :focus-visible {
    outline: 3px solid ButtonText;
  }
}
```

### tabindex

```html
<!-- tabindex="0": add to tab order (use sparingly; use native elements instead) -->
<div role="button" tabindex="0" onclick="activate()" onkeydown="handleKey(event)">
  Custom button
</div>

<!-- tabindex="-1": focusable via JS, not in tab order -->
<div id="modal-content" tabindex="-1">
  <!-- Move focus here programmatically when modal opens -->
</div>

<!-- NEVER use tabindex > 0: breaks natural tab order -->
<!-- <div tabindex="5"> — avoid completely -->
```

### Modal Focus Management

```tsx
function Modal({ isOpen, onClose, triggerRef, title, children }: ModalProps) {
  const dialogRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<Element | null>(null);

  useEffect(() => {
    if (isOpen) {
      // Save where focus was before modal opened
      previousFocusRef.current = document.activeElement;
      // Move focus into dialog
      dialogRef.current?.focus();
    } else {
      // Restore focus to trigger element when modal closes
      (previousFocusRef.current as HTMLElement)?.focus();
    }
  }, [isOpen]);

  // Focus trap
  function handleKeyDown(e: KeyboardEvent<HTMLDivElement>) {
    if (e.key === 'Escape') {
      onClose();
      return;
    }

    if (e.key !== 'Tab') return;

    const focusable = dialogRef.current!.querySelectorAll<HTMLElement>(
      '[href], button:not([disabled]), input:not([disabled]), select, textarea, [tabindex="0"]'
    );

    const first = focusable[0];
    const last = focusable[focusable.length - 1];

    if (e.shiftKey && document.activeElement === first) {
      e.preventDefault();
      last.focus();
    } else if (!e.shiftKey && document.activeElement === last) {
      e.preventDefault();
      first.focus();
    }
  }

  if (!isOpen) return null;

  return (
    <div className="modal-backdrop" onClick={onClose}>
      <div
        ref={dialogRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        tabIndex={-1}
        className="modal"
        onClick={e => e.stopPropagation()}
        onKeyDown={handleKeyDown}
      >
        <h2 id="modal-title">{title}</h2>
        <button
          aria-label="Close dialog"
          onClick={onClose}
        >
          ✕
        </button>
        {children}
      </div>
    </div>
  );
}
```

### Keyboard Pattern: Menu / Dropdown

```tsx
function MenuButton({ label, items }: MenuButtonProps) {
  const [open, setOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(-1);
  const menuRef = useRef<HTMLUListElement>(null);
  const buttonRef = useRef<HTMLButtonElement>(null);

  function handleKeyDown(e: KeyboardEvent<HTMLElement>) {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setActiveIndex(i => Math.min(i + 1, items.length - 1));
        break;
      case 'ArrowUp':
        e.preventDefault();
        setActiveIndex(i => Math.max(i - 1, 0));
        break;
      case 'Home':
        e.preventDefault();
        setActiveIndex(0);
        break;
      case 'End':
        e.preventDefault();
        setActiveIndex(items.length - 1);
        break;
      case 'Escape':
        setOpen(false);
        buttonRef.current?.focus();
        break;
    }
  }

  return (
    <div>
      <button
        ref={buttonRef}
        aria-haspopup="true"
        aria-expanded={open}
        onClick={() => setOpen(o => !o)}
      >
        {label}
      </button>
      {open && (
        <ul role="menu" ref={menuRef} onKeyDown={handleKeyDown}>
          {items.map((item, i) => (
            <li key={item.id} role="none">
              <button
                role="menuitem"
                tabIndex={i === activeIndex ? 0 : -1}
                onClick={() => { item.action(); setOpen(false); }}
              >
                {item.label}
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## Color and Contrast

```
WCAG 2.1 AA Requirements:
  Normal text (< 18pt or < 14pt bold):  4.5:1 contrast ratio
  Large text (≥ 18pt or ≥ 14pt bold):   3:1 contrast ratio
  UI components and focus indicators:    3:1 contrast ratio
  Decorative elements:                   No requirement

WCAG 2.1 AAA (enhanced):
  Normal text:  7:1
  Large text:   4.5:1
```

**Contrast calculation:**
Relative luminance formula. Tools do this for you:
- Chrome DevTools: inspect element → Accessibility tab → contrast ratio
- Colour Contrast Analyser (desktop app)
- axe DevTools browser extension

### Don't Use Color Alone

```html
<!-- BAD: only color conveys meaning -->
<span style="color: red">Required field</span>
<td class="positive">+$1,200</td>
<td class="negative">-$400</td>

<!-- GOOD: color + text/icon/pattern -->
<span style="color: red">
  <svg aria-hidden="true"><!-- asterisk icon --></svg>
  Required field
</span>

<td class="positive" aria-label="Positive $1,200">
  <svg aria-hidden="true"><!-- up arrow --></svg>
  +$1,200
</td>
```

---

## Screen Reader Utility Class

```css
/* sr-only: visually hidden but available to screen readers */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
}

/* Make focusable sr-only elements visible on focus (for skip links) */
.sr-only:focus,
.sr-only:focus-within {
  position: static;
  width: auto;
  height: auto;
  padding: inherit;
  margin: inherit;
  overflow: visible;
  clip: auto;
  white-space: normal;
}
```

```html
<!-- Skip link: first interactive element on page -->
<a href="#main-content" class="sr-only">Skip to main content</a>

<nav>...</nav>
<main id="main-content">
  <!-- keyboard users jump here, skipping nav -->
</main>
```

---

## Testing

### Automated Testing

```tsx
// jest-axe: catch violations in component tests
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

test('Button has no accessibility violations', async () => {
  const { container } = render(<Button onClick={jest.fn()}>Click me</Button>);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});

test('Modal with open state has no violations', async () => {
  const { container } = render(
    <Modal isOpen={true} onClose={jest.fn()} title="Confirm">
      Are you sure?
    </Modal>
  );
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

### Manual Testing Protocol

**Keyboard-only test (no mouse):**
1. Tab through entire page
2. Verify logical tab order matches visual order
3. Verify all interactive elements reachable
4. Verify focus indicator always visible
5. Activate buttons (Enter/Space), links (Enter), dismiss dialogs (Escape)
6. Test form submission with keyboard only

**Screen reader test:**
- Windows: NVDA + Firefox (free)
- macOS: VoiceOver + Safari (built-in, Cmd+F5 to toggle)
- iOS: VoiceOver + Safari (Settings → Accessibility → VoiceOver)
- Android: TalkBack + Chrome

**Zoom test:**
- Browser zoom to 200%
- No horizontal scrolling
- No content overlap or truncation
- Text still readable (uses `rem` not `px`)

**Color perception:**
- Chrome DevTools → Rendering → Emulate vision deficiencies
- Test: Deuteranopia, Protanopia, Achromatopsia
- Verify all UI state changes are conveyed without color alone

---

## Anti-Patterns

- **`aria-label` on every element** — creates noise for screen reader users; label only interactive and landmark elements
- **Placeholder as the only label** — placeholder disappears on type, isn't announced reliably; always use `<label>`
- **Custom checkbox with no keyboard support** — must handle Space key, `aria-checked`, visible focus
- **`display: none` to "hide" validation errors** — screen readers skip these; use `aria-invalid` + live region
- **`autofocus` on page load** — disorienting for screen reader users who haven't heard the page structure yet
- **Opened modals that don't trap focus** — keyboard users escape the modal and operate background content
- **Icon buttons with no label** — `<button><img src="trash.png"></button>` announces nothing useful
- **Non-descriptive link text** — "Click here", "Read more" — screen reader link lists become useless

---

## Quick Reference Checklist

**Structure:**
- [ ] One `<h1>` per page; heading levels not skipped
- [ ] `<main>`, `<nav>`, `<header>`, `<footer>`, `<aside>` landmarks used correctly
- [ ] All lists use `<ul>`/`<ol>`/`<dl>`; no list-like divs

**Interactive elements:**
- [ ] `<button>` for actions, `<a href>` for navigation; no `<div onClick>`
- [ ] All icon-only buttons have `aria-label`
- [ ] Every form input has a visible `<label>` or `aria-label`
- [ ] Error messages linked via `aria-describedby`; field marked `aria-invalid="true"`

**Keyboard:**
- [ ] All interactive elements reachable by Tab
- [ ] Focus indicator visible (`outline` not removed)
- [ ] Modal traps focus; Escape closes; focus returns to trigger on close
- [ ] Custom widgets follow ARIA keyboard patterns (arrows, Home/End, Escape)

**Color and contrast:**
- [ ] Normal text ≥ 4.5:1 contrast ratio
- [ ] UI component borders ≥ 3:1 contrast ratio
- [ ] No information conveyed by color alone

**Dynamic content:**
- [ ] Loading states announced via `aria-live` or `aria-busy`
- [ ] Success/error messages announced via live region
- [ ] Page title updated on SPA route changes

**Testing:**
- [ ] Zero axe-core violations in automated tests
- [ ] Keyboard-only navigation verified
- [ ] Screen reader tested (VoiceOver/NVDA)
- [ ] 200% zoom tested
