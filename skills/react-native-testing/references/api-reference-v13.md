# RNTL v13 API Reference

Complete API reference for `@testing-library/react-native` v13.x (React 18+).

**Test renderer:** `react-test-renderer`
**Element type:** `ReactTestInstance`

## Core Pattern

```tsx
import { render, screen, userEvent } from '@testing-library/react-native';

jest.useFakeTimers(); // recommended when using userEvent

test('description', async () => {
  const user = userEvent.setup();
  render(<Component />); // sync in v13

  const button = screen.getByRole('button', { name: 'Submit' });
  await user.press(button);

  expect(screen.getByText('Done')).toBeOnTheScreen();
});
```

**Setup note:** In Expo projects, use `jest-expo` preset (not plain Jest). `jest-expo` bundles a compatible Jest version. Jest 30+ is NOT compatible with jest-expo. For RN 0.83+, you may need to add a `transform` rule for RN's TypeScript setup file that jest-expo's Babel config doesn't cover.

## render / renderAsync

`render` operates synchronously in v13, returning results immediately. `renderAsync` (v13.3+) handles React 19 and Suspense components asynchronously.

```ts
function render(
  component: React.Element<any>,
  options?: RenderOptions,
): RenderResult;

async function renderAsync(
  component: React.Element<any>,
  options?: RenderOptions,
): Promise<RenderResult>;
```

### Options

| Option | Type | Description |
|--------|------|-------------|
| `wrapper` | `React.ComponentType<any>` | Wraps tested component (useful for context providers) |
| `createNodeMock` | `(element) => unknown` | Custom mock refs |
| `concurrentRoot` | `boolean` | Enable concurrent rendering (default: false) |

## screen

```ts
let screen: {
  ...queries;
  rerender(element: React.Element<unknown>): void;
  rerenderAsync(element: React.Element<unknown>): Promise<void>;
  unmount(): void;
  unmountAsync(): Promise<void>;
  debug(options?: { message?: string; mapProps?: MapPropsFunction }): void;
  toJSON(): RendererJSON | null;
  root: ReactTestInstance;
  UNSAFE_root: ReactTestInstance;
};
```

## Queries

Each query = **variant** + **predicate** (e.g., `getByRole` = `getBy` + `ByRole`).

### Query Variants

| Variant | Assertion | Return Type | Async |
|---------|-----------|-------------|-------|
| `getBy*` | Exactly one match | `ReactTestInstance` (throws if 0 or >1) | No |
| `getAllBy*` | At least one match | `ReactTestInstance[]` (throws if 0) | No |
| `queryBy*` | Zero or one match | `ReactTestInstance \| null` (throws if >1) | No |
| `queryAllBy*` | No assertion | `ReactTestInstance[]` (empty if 0) | No |
| `findBy*` | Exactly one match | `Promise<ReactTestInstance>` | Yes |
| `findAllBy*` | At least one match | `Promise<ReactTestInstance[]>` | Yes |

`findBy*` / `findAllBy*` accept optional `waitForOptions: { timeout?, interval?, onTimeout? }`.

### Query Predicates

#### `*ByRole` (preferred)

```ts
getByRole(role: TextMatch, options?: {
  name?: TextMatch;
  disabled?: boolean;
  selected?: boolean;
  checked?: boolean | 'mixed';
  busy?: boolean;
  expanded?: boolean;
  value?: { min?: number; max?: number; now?: number; text?: TextMatch };
  includeHiddenElements?: boolean;
}): ReactTestInstance;
```

Matches elements by `role` or `accessibilityRole`. Element must be an accessibility element:
- `Text`, `TextInput`, `Switch` are by default
- `View` needs `accessible={true}` (or use `Pressable`/`TouchableOpacity`)

Common roles: `button`, `text`, `heading` (alias: `header`), `searchbox`, `switch`, `checkbox`, `radio`, `img`, `link`, `alert`, `menu`, `menuitem`, `tab`, `tablist`, `progressbar`, `slider`, `spinbutton`, `timer`, `toolbar`.

#### `*ByLabelText`

Matches by `aria-label`/`accessibilityLabel` or text content of element referenced by `aria-labelledby`/`accessibilityLabelledBy`.

#### `*ByPlaceholderText`

Matches `TextInput` by `placeholder` prop.

#### `*ByText`

Matches by text content. Joins `<Text>` siblings to find matches (like RN runtime).

#### `*ByDisplayValue`

Matches `TextInput` by current display value.

#### `*ByHintText`

Matches by `accessibilityHint` prop. Also available as `getByA11yHint` / `getByAccessibilityHint`.

#### `*ByTestId` (last resort)

Matches by `testID` prop. Use only when other queries don't work.

### TextMatch

```ts
type TextMatch = string | RegExp;
```

- **String**: exact full match by default. Use `{ exact: false }` for case-insensitive substring match.
- **RegExp**: substring match by default. Use anchors (`^...$`) for full match.

### Common Query Options

- **`includeHiddenElements`** (alias: `hidden`): Include elements hidden from accessibility. Default: `false`.
- **`exact`**: Default `true`. When `false`, matches substrings case-insensitively. No effect on RegExp.
- **`normalizer`**: Custom text normalization function. Default trims whitespace and collapses multiple spaces.

## User Event

Prefer `userEvent` over `fireEvent`. User Event simulates realistic interaction sequences on **host elements only**.

### Setup

```ts
const user = userEvent.setup(options?: { delay?: number; advanceTimers?: (delay: number) => Promise<void> | void });
```

### Methods

- **`press(element)`** — Full press lifecycle: `pressIn` → `press` → `pressOut`. Minimum 130ms.
- **`longPress(element, { duration? })`** — Long press (default 500ms). Emits `pressIn`, `longPress`, `pressOut`.
- **`type(textInput, text, { skipPress?, skipBlur?, submitEditing? })`** — Character-by-character typing. Appends to existing text.
- **`clear(textInput)`** — Clears `TextInput` content.
- **`paste(textInput, text)`** — Pastes text into `TextInput`.
- **`scrollTo(scrollView, { y?, x?, momentumY?, momentumX?, contentSize?, layoutMeasurement? })`** — Scrolls a host `ScrollView`.

## Fire Event

Use when `userEvent` doesn't support the event. Sync in v13.

```ts
function fireEvent(
  element: ReactTestInstance,
  eventName: string,
  ...data: unknown[]
): void;

// Async variant for React 19 / Suspense
async function fireEventAsync(...): Promise<void>;
```

### Convenience Methods

```ts
fireEvent.press(element, ...data);
fireEvent.changeText(element, ...data);
fireEvent.scroll(element, ...data);
```

## Jest Matchers

Available automatically with any `@testing-library/react-native` import.

### Element Existence
- `toBeOnTheScreen()` — Element is attached to the element tree

### Element Content
- `toHaveTextContent(text: string | RegExp, options?)` — Text content match
- `toContainElement(element: ReactTestInstance | null)` — Contains child element
- `toBeEmptyElement()` — No children or text content

### Element State
- `toHaveDisplayValue(value: string | RegExp, options?)` — TextInput display value
- `toHaveAccessibilityValue({ min?, max?, now?, text? })` — Accessibility value (partial match)
- `toBeEnabled()` / `toBeDisabled()` — Disabled state via `aria-disabled` (checks ancestors)
- `toBeSelected()` — Selected state via `aria-selected`
- `toBeChecked()` / `toBePartiallyChecked()` — Checked state via `aria-checked`
- `toBeExpanded()` / `toBeCollapsed()` — Expanded state via `aria-expanded`
- `toBeBusy()` — Busy state via `aria-busy`

### Element Style
- `toBeVisible()` — Not hidden (`display: none`, `opacity: 0`, or hidden from a11y)
- `toHaveStyle(style: StyleProp<Style>)` — Specific style match

### Other
- `toHaveAccessibleName(name?: string | RegExp, options?)` — Accessible name
- `toHaveProp(name: string, value?: unknown)` — Prop existence/value check (last resort)

## Async Utilities

### waitFor

```ts
function waitFor<T>(
  expectation: () => T,
  options?: { timeout?: number; interval?: number },
): Promise<T>;
```

Rules:
- No side effects inside callback
- One assertion per `waitFor`
- Never pass empty callback
- Prefer `findBy*` over `waitFor` + `getBy*`

### waitForElementToBeRemoved

```ts
function waitForElementToBeRemoved<T>(
  expectation: () => T,
  options?: { timeout?: number; interval?: number },
): Promise<T>;
```

## Other Helpers

- **`within(element)`** — Scoped queries on a subtree
- **`cleanup()`** — Automatic after each test; don't call manually
- **`act(callback)`** — Usually not needed; render, fireEvent, userEvent handle it

## renderHook / renderHookAsync

```ts
function renderHook<Result, Props>(
  hookFn: (props?: Props) => Result,
  options?: { initialProps?: Props; wrapper?: React.ComponentType },
): {
  result: { current: Result };
  rerender: (props: Props) => void;
  unmount: () => void;
};
```

## Configuration

```ts
function configure(options: Partial<{
  asyncUtilTimeout: number;
  defaultIncludeHiddenElements: boolean;
  concurrentRoot: boolean;
  defaultDebugOptions: Partial<DebugOptions>;
}>): void;
```

## React 19 Compatibility

Use async variants for React 19 / Suspense:
- `renderAsync` instead of `render`
- `rerenderAsync` instead of `rerender`
- `unmountAsync` instead of `unmount`
- `fireEventAsync` instead of `fireEvent`
