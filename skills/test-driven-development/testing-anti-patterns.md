# Testing Anti-Patterns

**Load this reference when:** writing or changing tests, adding mocks, or tempted to add test-only methods to production code.

## Overview

Tests must verify real behavior, not mock behavior. In React Native, this means testing what the user sees and does, not component internals or native module implementation details.

**Core principle:** Test what the code does, not what the mocks do.

**Following strict TDD prevents these anti-patterns.**

## The Iron Laws

```
1. NEVER test mock behavior
2. NEVER add test-only methods to production classes
3. NEVER mock without understanding dependencies
4. NEVER test implementation details instead of user-visible behavior
5. NEVER skip waitFor for async state updates
```

## Anti-Pattern 1: Testing Mock Behavior

**The violation:**
```typescript
// BAD: Testing that the mock exists
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeOnTheScreen();
});
```

**Why this is wrong:**
- You're verifying the mock works, not that the component works
- Test passes when mock is present, fails when it's not
- Tells you nothing about real behavior

**your human partner's correction:** "Are we testing the behavior of a mock?"

**The fix:**
```typescript
// GOOD: Test real component or don't mock it
test('renders navigation sidebar', () => {
  render(<Page />);
  expect(screen.getByRole('navigation')).toBeOnTheScreen();
});

// OR if sidebar must be mocked for isolation:
// Don't assert on the mock - test Page's behavior with sidebar present
```

### Gate Function

```
BEFORE asserting on any mock element:
  Ask: "Am I testing real component behavior or just mock existence?"

  IF testing mock existence:
    STOP - Delete the assertion or unmock the component

  Test real behavior instead
```

## Anti-Pattern 2: Test-Only Methods in Production

**The violation:**
```typescript
// BAD: resetState() only used in tests
function useAuthStore() {
  const [user, setUser] = useState(null);

  const resetState = () => setUser(null); // Only called in tests!

  return { user, setUser, resetState };
}

// In tests
afterEach(() => result.current.resetState());
```

**Why this is wrong:**
- Production code polluted with test-only methods
- Dangerous if accidentally called in production
- Violates YAGNI and separation of concerns

**The fix:**
```typescript
// GOOD: Each test renders fresh - no cleanup of component state needed
// React Native Testing Library's render() creates a fresh tree each time

// For stores/singletons that persist, reset in jest.setup.js or beforeEach:
beforeEach(() => {
  // Re-initialize store to default state
  jest.resetModules();
});
```

### Gate Function

```
BEFORE adding any method to production code:
  Ask: "Is this only used by tests?"

  IF yes:
    STOP - Don't add it
    Put it in test utilities instead or restructure the test

  Ask: "Does this class own this resource's lifecycle?"

  IF no:
    STOP - Wrong class for this method
```

## Anti-Pattern 3: Mocking Without Understanding

**The violation:**
```typescript
// BAD: Mock breaks test logic
jest.mock('@react-native-async-storage/async-storage');

test('persists user preferences', async () => {
  const user = userEvent.setup();
  render(<SettingsScreen />);

  await user.press(screen.getByText('Dark Mode'));

  // AsyncStorage is fully mocked - this tests nothing about persistence!
  expect(AsyncStorage.setItem).toHaveBeenCalledWith('theme', 'dark');
});
```

**Why this is wrong:**
- You're just testing that a mock was called, not that persistence works
- If someone changes the storage key or mechanism, test still passes
- Test gives false confidence about real behavior

**The fix:**
```typescript
// GOOD: Test the observable outcome
test('applies dark mode after toggling setting', async () => {
  const user = userEvent.setup();
  render(<SettingsScreen />);

  await user.press(screen.getByText('Dark Mode'));

  // Test what the user sees, not the storage call
  await waitFor(() => {
    expect(screen.getByText('Dark Mode')).toBeOnTheScreen();
    // Verify dark theme is applied visually
  });
});

// If you must verify storage, use a real (in-memory) implementation
// rather than asserting on mock calls
```

### Gate Function

```
BEFORE mocking any method:
  STOP - Don't mock yet

  1. Ask: "What side effects does the real method have?"
  2. Ask: "Does this test depend on any of those side effects?"
  3. Ask: "Do I fully understand what this test needs?"

  IF depends on side effects:
    Mock at lower level (the actual slow/external operation)
    OR use test doubles that preserve necessary behavior
    NOT the high-level method the test depends on

  IF unsure what test depends on:
    Run test with real implementation FIRST
    Observe what actually needs to happen
    THEN add minimal mocking at the right level

  Red flags:
    - "I'll mock this to be safe"
    - "This might be slow, better mock it"
    - Mocking without understanding the dependency chain
```

## Anti-Pattern 4: Testing Implementation Details

**The violation:**
```typescript
// BAD: Testing internal state instead of user-visible behavior
test('sets loading state', () => {
  const { result } = renderHook(() => useProfileData('123'));

  expect(result.current.state).toBe('loading');
  // Who cares about internal state names? Test what the user sees.
});

// BAD: Testing component instance methods
test('validates input', () => {
  const ref = createRef<LoginFormRef>();
  render(<LoginForm ref={ref} />);

  expect(ref.current?.validateEmail('bad')).toBe(false);
  // Users don't call validateEmail() - they type and press submit
});

// BAD: Testing style objects directly
test('button is red when disabled', () => {
  render(<SubmitButton disabled />);
  const button = screen.getByRole('button');
  expect(button.props.style.backgroundColor).toBe('red');
  // Fragile - breaks if styling approach changes
});
```

**Why this is wrong:**
- Internal state names, method signatures, and style objects are implementation details
- Tests break when you refactor without changing behavior
- Tests pass even when the user experience is broken

**The fix:**
```typescript
// GOOD: Test what the user sees
test('shows loading indicator while fetching profile', async () => {
  render(<ProfileScreen userId="123" />);

  expect(screen.getByText('Loading...')).toBeOnTheScreen();

  await waitFor(() => {
    expect(screen.queryByText('Loading...')).not.toBeOnTheScreen();
  });
});

// GOOD: Test through user interaction
test('shows error when invalid email submitted', async () => {
  const user = userEvent.setup();
  render(<LoginForm />);

  await user.type(screen.getByPlaceholderText('Email'), 'bad');
  await user.press(screen.getByRole('button', { name: 'Sign In' }));

  expect(screen.getByText('Invalid email address')).toBeOnTheScreen();
});
```

### Gate Function

```
BEFORE writing an assertion:
  Ask: "Would a user notice if this assertion failed?"

  IF no:
    STOP - You're testing implementation details
    Rewrite to assert on rendered output or user-observable behavior

  Ask: "Would this test break if I refactored without changing behavior?"

  IF yes:
    STOP - You're coupled to implementation
    Test the interface, not the internals
```

## Anti-Pattern 5: Directly Testing Native Module Internals

**The violation:**
```typescript
// BAD: Testing native module calls directly
test('uses camera', () => {
  render(<PhotoScreen />);
  expect(NativeModules.CameraModule.openCamera).toHaveBeenCalled();
});

// BAD: Asserting on native bridge calls
test('plays haptic feedback', () => {
  fireEvent.press(screen.getByText('Submit'));
  expect(ReactNativeHapticFeedback.trigger).toHaveBeenCalledWith('impactMedium');
});
```

**Why this is wrong:**
- Tests couple to native implementation that may change across RN versions
- You're testing the bridge, not your app's behavior
- Native modules should be treated as external dependencies

**The fix:**
```typescript
// GOOD: Test the user-visible result of the native interaction
test('shows captured photo after taking picture', async () => {
  // Mock the native module to return a known photo URI
  jest.mocked(ImagePicker.launchCameraAsync).mockResolvedValue({
    canceled: false,
    assets: [{ uri: 'file://photo.jpg' }],
  });

  const user = userEvent.setup();
  render(<PhotoScreen />);

  await user.press(screen.getByRole('button', { name: 'Take Photo' }));

  await waitFor(() => {
    expect(screen.getByRole('image')).toBeOnTheScreen();
  });
});
```

## Anti-Pattern 6: Not Using `waitFor` for Async State Updates

**The violation:**
```typescript
// BAD: No waiting for async updates
test('shows results after search', async () => {
  const user = userEvent.setup();
  render(<SearchScreen />);

  await user.type(screen.getByPlaceholderText('Search'), 'react native');
  await user.press(screen.getByRole('button', { name: 'Search' }));

  // This will fail - results haven't loaded yet!
  expect(screen.getByText('10 results found')).toBeOnTheScreen();
});

// BAD: Using arbitrary delays
test('shows results after search', async () => {
  // ...
  await new Promise(resolve => setTimeout(resolve, 1000)); // Never do this
  expect(screen.getByText('10 results found')).toBeOnTheScreen();
});
```

**Why this is wrong:**
- State updates from async operations are not synchronous
- Arbitrary timeouts are flaky and slow
- Tests become timing-dependent and break unpredictably

**The fix:**
```typescript
// GOOD: Use waitFor or findBy* for async results
test('shows results after search', async () => {
  const user = userEvent.setup();
  render(<SearchScreen />);

  await user.type(screen.getByPlaceholderText('Search'), 'react native');
  await user.press(screen.getByRole('button', { name: 'Search' }));

  // waitFor retries until assertion passes
  await waitFor(() => {
    expect(screen.getByText('10 results found')).toBeOnTheScreen();
  });

  // OR use findBy* which is shorthand for waitFor + getBy*
  expect(await screen.findByText('10 results found')).toBeOnTheScreen();
});
```

## Anti-Pattern 7: Snapshot Testing Overuse

**The violation:**
```typescript
// BAD: Snapshot as the only test
test('renders correctly', () => {
  const tree = render(<UserProfile user={mockUser} />).toJSON();
  expect(tree).toMatchSnapshot();
});
```

**Why this is wrong:**
- Snapshots test everything and nothing at the same time
- Any change triggers snapshot failure, even irrelevant ones
- Developers blindly update snapshots without reviewing diffs
- Snapshots don't describe intended behavior
- Large snapshots are unreadable in code review

**The fix:**
```typescript
// GOOD: Test specific behaviors
test('displays user name and email', () => {
  render(<UserProfile user={{ name: 'Alice', email: 'alice@example.com' }} />);

  expect(screen.getByText('Alice')).toBeOnTheScreen();
  expect(screen.getByText('alice@example.com')).toBeOnTheScreen();
});

test('shows edit button for own profile', () => {
  render(<UserProfile user={currentUser} isOwnProfile />);

  expect(screen.getByRole('button', { name: 'Edit Profile' })).toBeOnTheScreen();
});
```

### Gate Function

```
BEFORE writing a snapshot test:
  Ask: "What specific behavior am I trying to verify?"

  IF you can name specific behaviors:
    Write behavioral tests instead of snapshots

  IF you can't name specific behaviors:
    You don't understand the requirements well enough to test
    Clarify requirements first

  Snapshots are acceptable ONLY for:
    - Catching unintended changes to serializable config/data structures
    - Never as a substitute for behavioral tests
```

## Anti-Pattern 8: Testing Platform-Specific Code Without Proper Mocking

**The violation:**
```typescript
// BAD: Test only runs correctly on one platform
test('shows iOS-specific settings', () => {
  render(<SettingsScreen />);
  // This test breaks on Android CI!
  expect(screen.getByText('Face ID')).toBeOnTheScreen();
});
```

**Why this is wrong:**
- Tests behave differently based on which platform runs them
- CI environment platform may differ from development
- Platform-specific behavior must be explicitly tested

**The fix:**
```typescript
// GOOD: Mock Platform for platform-specific tests
import { Platform } from 'react-native';

test('shows Face ID option on iOS', () => {
  Platform.OS = 'ios';
  render(<SettingsScreen />);
  expect(screen.getByText('Face ID')).toBeOnTheScreen();
});

test('shows Fingerprint option on Android', () => {
  Platform.OS = 'android';
  render(<SettingsScreen />);
  expect(screen.getByText('Fingerprint')).toBeOnTheScreen();
});
```

## Anti-Pattern 9: Not Cleaning Up After Async Operations

**The violation:**
```typescript
// BAD: Test completes but async operations keep running
test('starts fetching data', () => {
  render(<DataScreen />);
  expect(screen.getByText('Loading...')).toBeOnTheScreen();
  // Test ends, but fetch is still in-flight
  // Causes "act() warning" or state update on unmounted component
});
```

**Why this is wrong:**
- Async operations completing after test ends cause warnings and flaky tests
- State updates on unmounted components indicate real cleanup issues
- Can cause test pollution - one test's async work affects the next

**The fix:**
```typescript
// GOOD: Wait for async operations to complete
test('fetches and displays data', async () => {
  render(<DataScreen />);

  // Wait for loading to finish
  await waitFor(() => {
    expect(screen.queryByText('Loading...')).not.toBeOnTheScreen();
  });

  expect(screen.getByText('Data loaded')).toBeOnTheScreen();
});

// GOOD: Mock timers if component uses intervals/timeouts
test('auto-refreshes every 30 seconds', () => {
  jest.useFakeTimers();
  render(<DataScreen />);

  jest.advanceTimersByTime(30000);
  // Assert on refresh behavior...

  jest.useRealTimers();
});
```

## Anti-Pattern 10: Incomplete Mocks

**The violation:**
```typescript
// BAD: Partial mock - only fields you think you need
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // Missing: metadata that downstream code uses
};

// Later: breaks when code accesses response.metadata.requestId
```

**Why this is wrong:**
- **Partial mocks hide structural assumptions** - You only mocked fields you know about
- **Downstream code may depend on fields you didn't include** - Silent failures
- **Tests pass but integration fails** - Mock incomplete, real API complete
- **False confidence** - Test proves nothing about real behavior

**The Iron Rule:** Mock the COMPLETE data structure as it exists in reality, not just fields your immediate test uses.

**The fix:**
```typescript
// GOOD: Mirror real API completeness
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // All fields real API returns
};
```

### Gate Function

```
BEFORE creating mock responses:
  Check: "What fields does the real API response contain?"

  Actions:
    1. Examine actual API response from docs/examples
    2. Include ALL fields system might consume downstream
    3. Verify mock matches real response schema completely

  Critical:
    If you're creating a mock, you must understand the ENTIRE structure
    Partial mocks fail silently when code depends on omitted fields

  If uncertain: Include all documented fields
```

## Anti-Pattern 11: Integration Tests as Afterthought

**The violation:**
```
Implementation complete
No tests written
"Ready for testing"
```

**Why this is wrong:**
- Testing is part of implementation, not optional follow-up
- TDD would have caught this
- Can't claim complete without tests

**The fix:**
```
TDD cycle:
1. Write failing test
2. Implement to pass
3. Refactor
4. THEN claim complete
```

## When Mocks Become Too Complex

**Warning signs:**
- Mock setup longer than test logic
- Mocking everything to make test pass
- Mocks missing methods real components have
- Test breaks when mock changes

**your human partner's question:** "Do we need to be using a mock here?"

**Consider:** Integration tests with real components often simpler than complex mocks. In React Native, rendering a real component tree with `render()` is usually fast enough without mocking children.

## TDD Prevents These Anti-Patterns

**Why TDD helps:**
1. **Write test first** - Forces you to think about what you're actually testing
2. **Watch it fail** - Confirms test tests real behavior, not mocks
3. **Minimal implementation** - No test-only methods creep in
4. **Real dependencies** - You see what the test actually needs before mocking

**If you're testing mock behavior, you violated TDD** - you added mocks without watching test fail against real code first.

## Quick Reference

| Anti-Pattern | Fix |
|--------------|-----|
| Assert on mock elements | Test real component or unmock it |
| Test-only methods in production | Move to test utilities |
| Mock without understanding | Understand dependencies first, mock minimally |
| Test implementation details | Test user-visible behavior instead |
| Test native module internals | Mock native modules, test observable outcomes |
| Skip `waitFor` for async | Use `waitFor` or `findBy*` queries |
| Snapshot testing overuse | Write behavioral tests with specific assertions |
| Platform-specific without mocking | Mock `Platform.OS` explicitly per test |
| No async cleanup | Wait for operations to complete, use fake timers |
| Incomplete mocks | Mirror real API completely |
| Tests as afterthought | TDD - tests first |
| Over-complex mocks | Consider integration tests with real components |

## Red Flags

- Assertion checks for `*-mock` test IDs
- Methods only called in test files
- Mock setup is >50% of test
- Test fails when you remove mock
- Can't explain why mock is needed
- Mocking "just to be safe"
- Using `getByTestID` when semantic queries would work
- Asserting on component state or props instead of rendered output
- `setTimeout` or `sleep` in tests instead of `waitFor`
- Snapshot tests with no behavioral tests alongside them
- `act()` warnings in test output (unresolved async operations)

## The Bottom Line

**Mocks are tools to isolate, not things to test.**

**Tests should reflect the user's experience, not the developer's implementation.**

If TDD reveals you're testing mock behavior or implementation details, you've gone wrong.

Fix: Test real, user-visible behavior or question why you're mocking at all.
