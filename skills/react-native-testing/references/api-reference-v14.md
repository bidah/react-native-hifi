# RNTL v14 API Reference

Complete API reference for `@testing-library/react-native` v14.x (React 19+).

**Test renderer:** `test-renderer` (not `react-test-renderer`)
**Element type:** `HostElement` (not `ReactTestInstance`)

## Core Pattern

```tsx
import { render, screen, userEvent } from '@testing-library/react-native';

jest.useFakeTimers(); // recommended when using userEvent

test('description', async () => {
  const user = userEvent.setup();
  await render(<Component />); // async in v14 — always await

  const button = screen.getByRole('button', { name: 'Submit' });
  await user.press(button);

  expect(screen.getByText('Done')).toBeOnTheScreen();
});
```

**Setup note:** In Expo projects, use `jest-expo` preset (not plain Jest). `jest-expo` bundles a compatible Jest version. Jest 30+ is NOT compatible with jest-expo. For RN 0.83+, you may need to add a `transform` rule for RN's TypeScript setup file that jest-expo's Babel config doesn't cover.

## Key Rule: Always `await`

In v14, the following APIs are **async** and must always be awaited:

- `await render(<Component />)`
- `await fireEvent.press(element)` (and all fireEvent variants)
- `await screen.rerender(<Component />)`
- `await screen.unmount()`
- `await renderHook(() => useHook())`
- `await act(() => { ... })` (even with sync callbacks)

## render

```ts
async function render(
  component: React.Element<any>,
  options?: RenderOptions,
): Promise<RenderResult>;
```

There is no `renderAsync` in v14. The standard `render` is already async.

### Options

| Option | Type | Description |
|--------|------|-------------|
| `wrapper` | `React.ComponentType<any>` | Wraps tested component (useful for context providers) |
| `createNodeMock` | `(element) => unknown` | Custom mock refs |

Note: `concurrentRoot` and `unstable_validateStringsRenderedWithinText` are removed in v14.

## screen

```ts
let screen: {
  ...queries;
  rerender(element: React.Element<unknown>): Promise<void>;  // async
  unmount(): Promise<void>;                                    // async
  debug(options?: { message?: string; mapProps?: MapPropsFunction }): void;
  toJSON(): RendererJSON | null;
  container: HostElement;  // safe root host element
  root: HostElement;       // root host element
};
```

Note: `UNSAFE_root` is removed in v14. Use `container` or `root` instead.

## Queries

Each query = **variant** + **predicate** (e.g., `getByRole` = `getBy` + `ByRole`).

### Query Variants

| Variant | Assertion | Return Type | Async |
|---------|-----------|-------------|-------|
| `getBy*` | Exactly one match | `HostElement` (throws if 0 or >1) | No |
| `getAllBy*` | At least one match | `HostElement[]` (throws if 0) | No |
| `queryBy*` | Zero or one match | `HostElement \| null` (throws if >1) | No |
| `queryAllBy*` | No assertion | `HostElement[]` (empty if 0) | No |
| `findBy*` | Exactly one match | `Promise<HostElement>` | Yes |
| `findAllBy*` | At least one match | `Promise<HostElement[]>` | Yes |

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
}): HostElement;
```

Matches elements by `role` or `accessibilityRole`. Element must be an accessibility element:
- `Text`, `TextInput`, `Switch` are by default
- `View` needs `accessible={true}` (or use `Pressable`/`TouchableOpacity`)

Common roles: `button`, `text`, `heading` (alias: `header`), `searchbox`, `switch`, `checkbox`, `radio`, `img`, `link`, `alert`, `menu`, `menuitem`, `tab`, `tablist`, `progressbar`, `slider`, `spinbutton`, `timer`, `toolbar`.

#### `*ByLabelText`

Matches by `aria-label`/`accessibilityLabel` or text content of element referenced by `aria-labelledby`.

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

- **String**: exact full match by default. Use `{ exact: false }` for case-insensitive substring.
- **RegExp**: substring match by default. Use anchors (`^...$`) for full match.

```tsx
screen.getByText('Hello World'); // exact full match
screen.getByText('llo Worl', { exact: false }); // substring, case-insensitive
screen.getByText(/World/); // regex substring
screen.getByText(/^hello world$/i); // regex full match, case-insensitive
```

## User Event

Prefer `userEvent` over `fireEvent`. Always async.

### Setup

```ts
const user = userEvent.setup(options?: { delay?: number; advanceTimers?: (delay: number) => Promise<void> | void });
```

### Methods

- **`press(element)`** — Full press lifecycle: `pressIn` → `press` → `pressOut`. Minimum 130ms.
- **`longPress(element, { duration? })`** — Long press (default 500ms).
- **`type(textInput, text, { skipPress?, skipBlur?, submitEditing? })`** — Character-by-character typing.
- **`clear(textInput)`** — Clears `TextInput` content.
- **`paste(textInput, text)`** — Pastes text into `TextInput`.
- **`scrollTo(scrollView, { y?, x?, momentumY?, momentumX?, contentSize?, layoutMeasurement? })`** — Scrolls a host `ScrollView`.

## Fire Event

**Async** in v14 — always `await`:

```ts
async function fireEvent(
  element: HostElement,
  eventName: string,
  ...data: unknown[]
): Promise<void>;
```

There is no `fireEventAsync` in v14. The standard `fireEvent` is already async.

### Convenience Methods

```ts
await fireEvent.press(element, ...data);
await fireEvent.changeText(element, ...data);
await fireEvent.scroll(element, ...data);
```

## Jest Matchers

Available automatically with any `@testing-library/react-native` import.

### Element Existence
- `toBeOnTheScreen()` — Element is attached to the element tree

### Element Content
- `toHaveTextContent(text: string | RegExp, options?)` — Text content match
- `toContainElement(element: HostElement | null)` — Contains child element
- `toBeEmptyElement()` — No children or text content

### Element State
- `toHaveDisplayValue(value: string | RegExp, options?)` — TextInput display value
- `toHaveAccessibilityValue({ min?, max?, now?, text? })` — Accessibility value
- `toBeEnabled()` / `toBeDisabled()` — Disabled state via `aria-disabled` (checks ancestors)
- `toBeSelected()` — Selected state via `aria-selected`
- `toBeChecked()` / `toBePartiallyChecked()` — Checked state via `aria-checked`
- `toBeExpanded()` / `toBeCollapsed()` — Expanded state via `aria-expanded`
- `toBeBusy()` — Busy state via `aria-busy`

### Element Style
- `toBeVisible()` — Not hidden
- `toHaveStyle(style: StyleProp<Style>)` — Specific style match

### Other
- `toHaveAccessibleName(name?: string | RegExp, options?)` — Accessible name
- `toHaveProp(name: string, value?: unknown)` — Prop check (last resort)

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
- **`cleanup()`** — Automatic; don't call manually

## renderHook

**Async** in v14:

```ts
async function renderHook<Result, Props>(
  hookFn: (props?: Props) => Result,
  options?: { initialProps?: Props; wrapper?: React.ComponentType },
): Promise<{
  result: { current: Result };
  rerender: (props: Props) => Promise<void>;
  unmount: () => Promise<void>;
}>;
```

There is no `renderHookAsync` in v14.

## act

Always returns `Promise<T>` in v14, must be awaited even with sync callbacks:

```ts
async function act<T>(callback: () => T): Promise<T>;
```

Usually not needed — `render`, `fireEvent`, `userEvent`, and `waitFor` handle it internally.

## Configuration

```ts
function configure(options: Partial<{
  asyncUtilTimeout: number;
  defaultIncludeHiddenElements: boolean;
  defaultDebugOptions: Partial<DebugOptions>;
}>): void;
```

Note: `concurrentRoot` option is removed (always on).

## Removed APIs

The following are removed in v14:
- `renderAsync` — Use `render` (already async)
- `fireEventAsync` — Use `fireEvent` (already async)
- `renderHookAsync` — Use `renderHook` (already async)
- `rerenderAsync` / `unmountAsync` — Use `rerender` / `unmount` (already async)
- `update()` — Use `rerender()`
- `getQueriesForElement()` — Use `within()`
- `UNSAFE_root` — Use `container` or `root`
- `UNSAFE_getByType`, `UNSAFE_getAllByType`, `UNSAFE_queryByType`, `UNSAFE_queryAllByType`
- `UNSAFE_getByProps`, `UNSAFE_getAllByProps`, `UNSAFE_queryByProps`, `UNSAFE_queryAllByProps`
- `concurrentRoot` option — Always on
- `unstable_validateStringsRenderedWithinText` option — Always on

## Migration Codemods

- **`rntl-v14-update-deps`** — Updates `react-test-renderer` to `test-renderer` in `package.json`
- **`rntl-v14-async-functions`** — Adds `await` to `render`, `fireEvent`, `rerender`, `unmount`, `renderHook`, and `act` calls
