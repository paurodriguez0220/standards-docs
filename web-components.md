## Purpose

Standards for building, documenting, and testing React components. Applies to any React project.

---

## Component Design Principles

- One component, one responsibility. If you can't describe what a component renders in one sentence, split it.
- Prefer small, composable components over large monolithic ones.
- Separate concerns: a component that fetches data should not also manage complex render logic. Split into a container (data) and a presentational (UI) component.
- Props in, events out. Components receive data through props and communicate back through callback props.

---

## TypeScript

- Define an explicit `interface` for every component's props — no inline object types, no `any`.
- Export the props interface alongside the component so consumers can reference it.
- Use `children: React.ReactNode` for slot content.
- Don't use `React.FC` — use a plain function with an explicit return type.

```tsx
// Good
export interface ButtonProps {
  label: string;
  onClick: () => void;
  isDisabled?: boolean;
}

export function Button({ label, onClick, isDisabled = false }: ButtonProps): JSX.Element {
  return (
    <button onClick={onClick} disabled={isDisabled}>
      {label}
    </button>
  );
}

// Bad
const Button: React.FC<{ label: any; onClick: any }> = ({ label, onClick }) => ...
```

---

## File Structure

One component per file, kebab-case filename. Co-locate stories and tests next to the component.

```
src/components/
└── expense-card/
    ├── expense-card.tsx          ← component + exported props interface
    ├── expense-card.stories.ts   ← Storybook stories
    └── expense-card.test.tsx     ← unit tests
```

For simple, single-file components:
```
src/components/
├── bottom-nav.tsx
├── bottom-nav.stories.ts
└── bottom-nav.test.tsx
```

---

## Storybook

### Purpose

Stories are the component's interactive contract. They document every meaningful visual state and serve as a living style guide.

### Setup

Storybook 10 with Vite builder (required for Vite 8+; v8/v9 require Vite ≤7):

```bash
npx storybook@10 init
```

Storybook 10 ships `@vitest/spy` built-in — do **not** install `@storybook/test` separately. Import spy helpers from `storybook/test`:

```ts
import { fn } from 'storybook/test';
```

### Story format — CSF3

```ts
// button.stories.ts
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './button';

const meta: Meta<typeof Button> = {
  component: Button,
  title: 'Components/Button',
  args: {
    label: 'Click me',
    isDisabled: false,
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Default: Story = {};

export const Disabled: Story = {
  args: { isDisabled: true },
};
```

### Story rules

- Every exported component must have at least a `Default` story.
- Add a story for every visually or behaviourally distinct state: disabled, loading, empty, error.
- Use `args` for all dynamic values — no hardcoded props inside render functions.
- Use `argTypes` to document non-obvious props.
- Don't test behaviour in stories — that belongs in unit tests.
- For mobile-first components, add a `parameters.viewport` entry using the Storybook viewport addon set to iPhone dimensions (390×844).

---

## Unit Testing

### Tools

- **Vitest** — Vite-native test runner. Add to every React project.
- **React Testing Library (RTL)** — test through the DOM as a user would.
- **@testing-library/user-event** — simulate real user interactions (type, click, tab).

### Setup

```bash
npm install -D vitest @testing-library/react @testing-library/user-event @testing-library/jest-dom jsdom
```

`vite.config.ts`:
```ts
test: {
  environment: 'jsdom',
  setupFiles: ['./src/test-setup.ts'],
},
```

`src/test-setup.ts`:
```ts
import '@testing-library/jest-dom';
```

### What to test

- Rendering: renders without crashing for all required prop combinations.
- User interactions: clicking, typing, submitting forms.
- Conditional rendering: correct content shown for different prop states.
- Callbacks: the right callback props are called with the right arguments.

### What not to test

- Internal state that doesn't affect the rendered output.
- Implementation details — don't query by class name or assert on `useState` calls.
- Styling — Storybook covers visual states; unit tests cover behaviour.

### Structure — AAA

Follow the same Arrange / Act / Assert pattern from `testing.md`:

```tsx
it('calls onClick when button is clicked', async () => {
  // Arrange
  const onClick = vi.fn();
  render(<Button label="Save" onClick={onClick} />);

  // Act
  await userEvent.click(screen.getByRole('button', { name: 'Save' }));

  // Assert
  expect(onClick).toHaveBeenCalledOnce();
});
```

### Query priority (RTL)

Query elements the way a user would find them — by role first:

| Priority | Query | Use for |
|---|---|---|
| 1 | `getByRole` | Buttons, inputs, headings, lists |
| 2 | `getByLabelText` | Form fields with associated labels |
| 3 | `getByText` | Visible static text |
| 4 | `getByTestId` | Last resort — only when no semantic query fits |

Never query by class name, tag name, or CSS selector.

---

## Async Event Handlers

Every async event handler needs a `catch` block. A `try/finally` without `catch` silently swallows the error — the spinner stops, nothing happens, the user has no idea what went wrong.

```tsx
// Bad — error propagates as unhandled rejection, user sees spinner disappear
async function handleSubmit(e: React.FormEvent): Promise<void> {
  e.preventDefault();
  setIsSaving(true);
  try {
    await save(data);
  } finally {
    setIsSaving(false);
  }
}

// Good — error surfaced to the user
const [submitError, setSubmitError] = useState<string | undefined>();

async function handleSubmit(e: React.FormEvent): Promise<void> {
  e.preventDefault();
  setIsSaving(true);
  setSubmitError(undefined);
  try {
    await save(data);
  } catch (err) {
    setSubmitError(err instanceof Error ? err.message : 'Something went wrong');
  } finally {
    setIsSaving(false);
  }
}
```

Rules:
- Every async handler needs `catch` or a mechanism that surfaces the error in the UI.
- Keep a local error state next to every form or action that has an async handler.
- Clear the error at the start of each new attempt (`setSubmitError(undefined)`).
- Partial failures (e.g. first DB write succeeds, second fails) must be treated as errors — show the message, do not silently retry.

---

## Dual-Mode Modal Pattern

When a modal handles both **add** and **edit** flows for the same entity, use a single component with an optional `initialValues` prop. The presence of `initialValues` switches the modal into edit mode.

```tsx
export interface InitialValues {
  id: number;
  // ...fields pre-populated from DB
}

export interface ModalProps {
  onSave: (fields: ...) => Promise<void>;
  onClose: () => void;
  initialValues?: InitialValues;       // absent = add mode
  onDelete?: (id: number) => Promise<void>; // absent = no delete button
}

export function Modal({ initialValues, onSave, onClose, onDelete }: ModalProps) {
  const isEditMode = !!initialValues;
  // ...
}
```

Rules:
- The `isEditMode` boolean drives the title ("Add X" vs "Edit X"), submit label, and whether the delete button renders.
- Initialise all form state from `initialValues` in `useState` defaults — not in a `useEffect` that runs after mount, which causes a visible flicker.
- Guard any `useEffect` that resets form fields with `if (isEditMode) return` to prevent it from overwriting pre-filled state.
- Delete actions must require a confirmation step inline within the modal (e.g. "Delete X" button → "Cancel / Yes, delete" pair). Never navigate to a separate confirmation screen.
- After a successful save or delete, call `onClose()` — do not reset state manually; the unmount handles it.

### Testing dual-mode modals in Storybook

Write four stories minimum:
- `Default` — add mode, empty form
- `EditMode` — edit mode with `initialValues` pre-filled
- `CustomSplit` / similar — non-default secondary form variant
- `ConfirmDelete` — edit mode with the delete confirmation visible (use a `play` function to click the delete button)

---

## Mobile / Touch Considerations

When building for mobile PWA (iPhone 15+, 390×844 logical px):

- Minimum touch target: **44×44px** (Apple HIG). In Tailwind: `min-h-11 min-w-11`.
- No hover-only states — always pair with `active:` for tap feedback.
- Fixed bottom elements (e.g. BottomNav) must clear the home indicator: `pb-[env(safe-area-inset-bottom)]`.
- Set `viewport-fit=cover` in the HTML `<meta name="viewport">` tag to allow content to extend under notches and Dynamic Island.
- Preview components at iPhone dimensions in Storybook using the viewport addon before shipping.

---

## Related

- [Code Style & Conventions](code-style.md)
- [Testing](testing.md)
- [Design Patterns](design-patterns.md)

---
*Maintained by paurodriguez0220 · Last updated: 2026-06-20*
*Standards: https://github.com/paurodriguez0220/standards-docs*
