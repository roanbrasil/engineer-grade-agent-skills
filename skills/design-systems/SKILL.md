---
name: design-systems
description: Design system architecture — invoked when building or maintaining component libraries, design tokens, theming, documentation, or distribution infrastructure.
---

# Design Systems

## What a Design System Is

```
┌─────────────────────────────────────────────────────────────┐
│                      Design System                          │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │    Design    │  │  Component   │  │  Documentation   │  │
│  │   Tokens     │  │   Library    │  │  & Guidelines    │  │
│  │              │  │              │  │                  │  │
│  │  colors      │  │  Button      │  │  Usage rules     │  │
│  │  spacing     │  │  Input       │  │  Patterns        │  │
│  │  typography  │  │  Modal       │  │  Accessibility   │  │
│  │  shadows     │  │  Card        │  │  Do / Don't      │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         ↓ consumed by
┌─────────────────────────────────────────────────────────────┐
│                    Product Teams                            │
│   Team A (Checkout)   Team B (Dashboard)   Team C (Auth)   │
└─────────────────────────────────────────────────────────────┘
```

A design system is NOT just a component library. It includes:
- **Principles**: brand voice, visual language, interaction philosophy
- **Patterns**: how components combine to solve UX problems
- **Usage guidelines**: when to use each component and when not to
- **Accessibility standards**: WCAG AA baseline for every component
- **Contribution model**: how teams propose and add new components

---

## Design Tokens

Three-tier token architecture:

```
Primitive tokens    →    Semantic tokens    →    Component tokens
(raw values)             (meaning)               (component-specific)

color-blue-500           color-action-primary     button-bg-color
  #3B82F6                  {color-blue-500}         {color-action-primary}

spacing-4                spacing-content-gap      card-padding
  16px                     {spacing-4}              {spacing-content-gap}
```

### W3C Design Token Format (JSON)

```json
{
  "color": {
    "blue": {
      "500": { "$value": "#3B82F6", "$type": "color" },
      "600": { "$value": "#2563EB", "$type": "color" }
    },
    "red": {
      "500": { "$value": "#EF4444", "$type": "color" }
    }
  },
  "spacing": {
    "1": { "$value": "4px",  "$type": "dimension" },
    "2": { "$value": "8px",  "$type": "dimension" },
    "4": { "$value": "16px", "$type": "dimension" },
    "8": { "$value": "32px", "$type": "dimension" }
  },
  "semantic": {
    "color": {
      "action": {
        "primary":       { "$value": "{color.blue.500}", "$type": "color" },
        "primary-hover": { "$value": "{color.blue.600}", "$type": "color" },
        "danger":        { "$value": "{color.red.500}",  "$type": "color" }
      }
    }
  }
}
```

### Style Dictionary — Token Transformation

```js
// style-dictionary.config.js
import StyleDictionary from 'style-dictionary';

export default {
  source: ['tokens/**/*.json'],
  platforms: {
    css: {
      transformGroup: 'css',
      prefix: 'ds',
      buildPath: 'dist/tokens/',
      files: [{
        destination: 'variables.css',
        format: 'css/variables',
        options: { outputReferences: true }
      }]
    },
    js: {
      transformGroup: 'js',
      buildPath: 'dist/tokens/',
      files: [{
        destination: 'tokens.js',
        format: 'javascript/es6'
      }]
    }
  }
};
```

Output CSS:

```css
:root {
  --ds-color-blue-500: #3B82F6;
  --ds-color-action-primary: var(--ds-color-blue-500);
  --ds-spacing-4: 16px;
}
```

---

## Component API Design

### Composition Over Configuration

```tsx
// BAD: explosion of variant props
<PrimaryButton large disabled />
<SecondaryButton small loading />

// GOOD: composable variant system
<Button variant="primary" size="lg" loading>
  Submit
</Button>
```

### Polymorphic Components

```tsx
type TextProps<T extends ElementType = 'span'> = {
  as?: T;
  size?: 'sm' | 'md' | 'lg' | 'xl';
  weight?: 'normal' | 'medium' | 'bold';
  children: ReactNode;
} & ComponentPropsWithoutRef<T>;

function Text<T extends ElementType = 'span'>({
  as,
  size = 'md',
  weight = 'normal',
  className,
  ...props
}: TextProps<T>) {
  const Component = as ?? 'span';
  return (
    <Component
      className={cn(styles.text, styles[`size-${size}`], styles[`weight-${weight}`], className)}
      {...props}
    />
  );
}

// Usage:
<Text as="h1" size="xl" weight="bold">Page Title</Text>
<Text as="p" size="md">Body paragraph</Text>
<Text as="label" htmlFor="email">Email</Text>
```

### Slot Pattern (Compound Components)

```tsx
// Context for compound component
const CardContext = createContext<CardContextValue | null>(null);

function Card({ children, className }: CardProps) {
  return (
    <CardContext.Provider value={{}}>
      <div className={cn(styles.card, className)}>{children}</div>
    </CardContext.Provider>
  );
}

Card.Header = function CardHeader({ children }: { children: ReactNode }) {
  return <div className={styles.cardHeader}>{children}</div>;
};

Card.Body = function CardBody({ children }: { children: ReactNode }) {
  return <div className={styles.cardBody}>{children}</div>;
};

Card.Footer = function CardFooter({ children }: { children: ReactNode }) {
  return <div className={styles.cardFooter}>{children}</div>;
};

// Usage:
<Card>
  <Card.Header>
    <h2>Order Summary</h2>
  </Card.Header>
  <Card.Body>
    <OrderItems />
  </Card.Body>
  <Card.Footer>
    <Button>Checkout</Button>
  </Card.Footer>
</Card>
```

### Controlled + Uncontrolled

```tsx
interface InputProps extends ComponentPropsWithoutRef<'input'> {
  label: string;
  error?: string;
  // Controlled: pass value + onChange
  // Uncontrolled: pass defaultValue
}

function Input({ label, error, id, ...props }: InputProps) {
  const inputId = id ?? useId();

  return (
    <div className={styles.field}>
      <label htmlFor={inputId}>{label}</label>
      <input
        id={inputId}
        aria-invalid={error ? 'true' : undefined}
        aria-describedby={error ? `${inputId}-error` : undefined}
        className={cn(styles.input, error && styles.inputError)}
        {...props}
      />
      {error && (
        <span id={`${inputId}-error`} className={styles.errorText} role="alert">
          {error}
        </span>
      )}
    </div>
  );
}
```

### `className` Extension Point

```tsx
// ALWAYS allow consumers to extend styles
interface ButtonProps extends ComponentPropsWithoutRef<'button'> {
  variant?: 'primary' | 'secondary' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
}

function Button({ variant = 'primary', size = 'md', className, ...props }: ButtonProps) {
  return (
    <button
      className={cn(
        styles.button,
        styles[`button--${variant}`],
        styles[`button--${size}`],
        className  // consumer styles last so they can override
      )}
      {...props}
    />
  );
}
```

---

## Documentation with Storybook

```tsx
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  component: Button,
  title: 'Components/Button',
  tags: ['autodocs'],
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'ghost'],
    },
    size: {
      control: 'radio',
      options: ['sm', 'md', 'lg'],
    },
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: { variant: 'primary', children: 'Click me' },
};

export const Loading: Story = {
  args: { variant: 'primary', loading: true, children: 'Saving...' },
};

export const Disabled: Story = {
  args: { disabled: true, children: 'Unavailable' },
};

// All interactive states documented as stories
export const AllVariants: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: '1rem', flexWrap: 'wrap' }}>
      <Button variant="primary">Primary</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="ghost">Ghost</Button>
    </div>
  ),
};
```

---

## Versioning and Distribution

### Package Structure

```
@myorg/design-system/
├── src/
│   ├── components/
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.module.css
│   │   │   ├── Button.test.tsx
│   │   │   └── Button.stories.tsx
│   ├── tokens/
│   │   └── tokens.json
│   └── index.ts          ← named exports only
├── dist/
│   ├── index.js          ← ESM
│   ├── index.cjs         ← CJS
│   └── tokens/
│       └── variables.css
└── package.json
```

### package.json

```json
{
  "name": "@myorg/design-system",
  "version": "3.2.0",
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    },
    "./tokens/css": "./dist/tokens/variables.css"
  },
  "sideEffects": false,
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  }
}
```

**Tree-shaking**: `sideEffects: false` + named exports enable bundlers to drop unused components.

### Changesets for Versioning

```bash
# When making a change
npx changeset
# → prompts for: which packages changed, semver bump type, changelog entry

# In CI, release:
npx changeset version   # bumps versions, updates CHANGELOG.md
npx changeset publish   # publishes to npm
```

---

## Theming

### CSS Custom Property Theming

```css
/* Light theme (default) */
:root {
  --ds-color-background: #ffffff;
  --ds-color-surface: #f8fafc;
  --ds-color-text-primary: #0f172a;
  --ds-color-text-secondary: #64748b;
  --ds-color-border: #e2e8f0;
  --ds-color-action-primary: #3b82f6;
}

/* Dark theme */
[data-theme="dark"] {
  --ds-color-background: #0f172a;
  --ds-color-surface: #1e293b;
  --ds-color-text-primary: #f1f5f9;
  --ds-color-text-secondary: #94a3b8;
  --ds-color-border: #334155;
  --ds-color-action-primary: #60a5fa;
}

/* System preference */
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    /* same dark variables */
  }
}
```

### React ThemeProvider

```tsx
type Theme = 'light' | 'dark' | 'system';

const ThemeContext = createContext<{
  theme: Theme;
  setTheme: (t: Theme) => void;
} | null>(null);

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>(() =>
    (localStorage.getItem('theme') as Theme) ?? 'system'
  );

  useEffect(() => {
    const root = document.documentElement;
    const resolved = theme === 'system'
      ? (window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light')
      : theme;

    root.setAttribute('data-theme', resolved);
    localStorage.setItem('theme', theme);
  }, [theme]);

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

export const useTheme = () => {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used within ThemeProvider');
  return ctx;
};
```

---

## Accessibility Built-in

Every component ships with accessibility baked in. Not optional.

```tsx
// Modal with full keyboard + ARIA support
function Modal({ isOpen, onClose, title, children }: ModalProps) {
  const titleId = useId();
  const dialogRef = useRef<HTMLDivElement>(null);

  // Focus trap
  useEffect(() => {
    if (!isOpen) return;
    const el = dialogRef.current;
    if (!el) return;

    const focusable = el.querySelectorAll<HTMLElement>(
      'a[href], button:not([disabled]), input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    focusable[0]?.focus();

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
      if (e.key === 'Tab') {
        // Trap focus within modal
        const first = focusable[0];
        const last = focusable[focusable.length - 1];
        if (e.shiftKey ? document.activeElement === first : document.activeElement === last) {
          e.preventDefault();
          (e.shiftKey ? last : first).focus();
        }
      }
    };

    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return (
    <div
      role="dialog"
      aria-modal="true"
      aria-labelledby={titleId}
      ref={dialogRef}
    >
      <h2 id={titleId}>{title}</h2>
      <button aria-label="Close dialog" onClick={onClose}>✕</button>
      {children}
    </div>
  );
}
```

### Accessibility Checklist Per Component

- [ ] Correct semantic element or ARIA role
- [ ] Visible label (or `aria-label` / `aria-labelledby`)
- [ ] Keyboard operable (Tab to focus, Enter/Space to activate)
- [ ] Focus indicator visible (never `outline: none` without replacement)
- [ ] Color contrast meets 4.5:1 (text) and 3:1 (UI components)
- [ ] State communicated via ARIA (`aria-expanded`, `aria-selected`, `aria-checked`)
- [ ] Motion can be disabled via `prefers-reduced-motion`
- [ ] Tested with axe-core (zero violations rule)

---

## Anti-Patterns

- **Naming components by visual appearance**: `RedButton`, `BigText` — use semantic names: `Button variant="danger"`, `Text size="xl"`
- **Token proliferation without governance**: 400 one-off color values — enforce primitives → semantic → component tiers
- **Breaking changes in patch versions**: consumer builds break silently — use Changesets; strict semver
- **No accessibility testing in CI**: a11y regressions go unnoticed — `jest-axe` on every component test
- **Blocking className customization**: `overflow: hidden` on styles — always spread `className`
- **Barrel exports importing everything**: prevents tree-shaking — keep `index.ts` minimal or use deep imports
- **Skipping Storybook stories for edge cases**: bugs in loading/error/empty states — story per state

---

## Quick Reference Checklist

- [ ] Three-tier token architecture: primitive → semantic → component
- [ ] Style Dictionary (or equivalent) generating CSS vars, JS, and platform outputs
- [ ] Compound component pattern for complex UI (Card, Tabs, Select)
- [ ] Polymorphic `as` prop for Text and layout components
- [ ] Both controlled and uncontrolled interfaces on form inputs
- [ ] `className` extension point on every component
- [ ] Storybook stories covering all variants, states, and edge cases
- [ ] Chromatic CI for visual regression
- [ ] Changesets for automated changelog and semver
- [ ] CSS custom property theming with `[data-theme]` attribute
- [ ] `ThemeProvider` with system preference detection
- [ ] Zero axe-core violations in component tests
