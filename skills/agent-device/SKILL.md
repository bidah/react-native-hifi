---
name: agent-device
description: Use when automating mobile device interactions for testing or development, controlling iOS simulators or Android emulators, inspecting accessibility trees, capturing screenshots, or performing automated UI testing workflows
---

# Mobile Device Automation for Testing and Development

## Overview

Agent-driven device automation enables programmatic control of iOS simulators, Android emulators, and physical devices. This covers launching apps, interacting with UI elements, inspecting the accessibility tree, capturing screenshots, and streaming app output for automated testing and development workflows.

**Core principle:** Interact with the device through its accessibility tree, not pixel coordinates. Element references provide stable automation targets across different screen sizes and OS versions.

## When to Use

- Automating UI test flows on simulators/emulators
- Verifying visual appearance across device configurations
- Testing deep links, push notifications, and app lifecycle
- Capturing screenshots for documentation or review
- Debugging layout issues by inspecting the accessibility tree
- Running end-to-end test scenarios
- Performance profiling during automated interactions

## Device Control

### Opening and Managing Simulators

```bash
# List available simulators
xcrun simctl list devices available

# Boot a specific simulator
xcrun simctl boot "iPhone 16 Pro"

# Open Simulator app
open -a Simulator

# Android: list available emulators
emulator -list-avds

# Android: start emulator
emulator -avd Pixel_8_API_35
```

### Launching Apps

```bash
# iOS: install and launch on simulator
npx expo run:ios --device "iPhone 16 Pro"

# Android: install and launch on emulator
npx expo run:android

# Open app via scheme
xcrun simctl openurl booted "myapp://home"

# Android deep link
adb shell am start -a android.intent.action.VIEW -d "myapp://home"
```

## Core Commands

### Open

Launch apps or URLs on the device:

```bash
# Open a URL in the default browser
xcrun simctl openurl booted "https://example.com"

# Open a deep link in your app
xcrun simctl openurl booted "myapp://profile/42"

# Android equivalent
adb shell am start -a android.intent.action.VIEW -d "myapp://profile/42"
```

### Press

Simulate button presses:

```bash
# iOS: Home button
xcrun simctl ui booted button home

# iOS: Lock button
xcrun simctl ui booted button lock

# Android: Back button
adb shell input keyevent KEYCODE_BACK

# Android: Home button
adb shell input keyevent KEYCODE_HOME

# Android: Enter key
adb shell input keyevent KEYCODE_ENTER
```

### Fill (Text Input)

Enter text into focused fields:

```bash
# iOS: type text into focused field
xcrun simctl io booted input "Hello World"

# Android: input text
adb shell input text "Hello%sWorld"  # %s = space

# Android: paste from clipboard (handles special characters)
adb shell input keyevent KEYCODE_PASTE
```

### Scroll

```bash
# iOS: simulate scroll gesture
xcrun simctl io booted scroll --direction down --distance 300

# Android: swipe to scroll
adb shell input swipe 500 1500 500 500 300  # x1 y1 x2 y2 duration_ms
```

### Swipe

```bash
# iOS: swipe gesture
xcrun simctl io booted swipe --direction left

# Android: swipe gesture (coordinates)
adb shell input swipe 900 500 100 500 200  # Left swipe
adb shell input swipe 100 500 900 500 200  # Right swipe
adb shell input swipe 500 500 500 1500 200 # Down swipe
```

## Accessibility Tree Snapshots

The accessibility tree provides a structural representation of all UI elements, their labels, roles, and states. This is the primary mechanism for locating elements to interact with.

### iOS Accessibility Inspection

```bash
# Dump the accessibility hierarchy
xcrun simctl ui booted accessibility

# Using Accessibility Inspector (GUI)
open -a "Accessibility Inspector"
```

### Android Accessibility

```bash
# Dump UI hierarchy
adb shell uiautomator dump /sdcard/ui_dump.xml
adb pull /sdcard/ui_dump.xml

# View the dump
cat ui_dump.xml
```

### Element Properties

Each element in the accessibility tree exposes:

| Property | Description |
|----------|-------------|
| `label` | Human-readable name (accessibilityLabel in RN) |
| `role` | Element type (button, text, image, etc.) |
| `value` | Current value (text input content, slider position) |
| `traits` | Behavioral hints (selected, disabled, header) |
| `frame` | Position and size on screen |
| `identifier` | Test ID (testID prop in React Native) |

### Setting testID in React Native

```tsx
<Pressable
  testID="submit-button"
  accessibilityLabel="Submit form"
  accessibilityRole="button"
  onPress={handleSubmit}
>
  <Text>Submit</Text>
</Pressable>

<TextInput
  testID="email-input"
  accessibilityLabel="Email address"
  placeholder="Enter email"
/>
```

## Element References (@e5 Pattern)

Element references provide stable identifiers for automation targets. Instead of fragile XPath or coordinate-based targeting, use accessibility identifiers:

```tsx
// In your React Native code, set testID for automation
<View testID="@e5-dashboard">
  <Text testID="@e5-welcome-text">Welcome</Text>
  <Pressable testID="@e5-settings-btn">
    <Text>Settings</Text>
  </Pressable>
  <FlatList
    testID="@e5-item-list"
    data={items}
    renderItem={({ item }) => (
      <View testID={`@e5-item-${item.id}`}>
        <Text>{item.name}</Text>
      </View>
    )}
  />
</View>
```

### Benefits of Element References

- **Stable across screen sizes:** No pixel coordinates to break
- **Survives layout changes:** Element identity is semantic, not positional
- **Readable test output:** `@e5-settings-btn` is self-documenting
- **Structural diffing:** Compare trees between test runs to detect regressions

## Structural Diffing

Compare accessibility tree snapshots between builds to detect unintended UI changes:

```bash
# Capture baseline
xcrun simctl ui booted accessibility > baseline.txt

# After code changes, capture again
xcrun simctl ui booted accessibility > current.txt

# Diff the structures
diff baseline.txt current.txt
```

### What to Look For

- Missing or added elements (feature regressions)
- Changed accessibility labels (broken VoiceOver)
- Altered element hierarchy (layout changes)
- Changed roles or traits (interaction regressions)

## App Output Streaming

Stream app logs in real time for debugging during automation:

```bash
# iOS: stream app logs from simulator
xcrun simctl spawn booted log stream --predicate 'subsystem == "com.myapp"' --level debug

# Simpler: all simulator logs
xcrun simctl spawn booted log stream --level info | grep MyApp

# Android: logcat filtered to your app
adb logcat --pid=$(adb shell pidof com.myapp) -v time

# React Native specific logs
npx react-native log-ios
npx react-native log-android
```

## Network Inspection

Monitor network requests during automated testing:

```bash
# iOS: use nscurl for HTTPS diagnostics
nscurl --ats-diagnostics https://api.example.com

# Android: enable network monitoring
adb shell settings put global http_proxy "localhost:8888"

# React Native: use Flipper or React Native Debugger for network inspection
# Or add to your app:
```

```tsx
// Enable network inspection in development
if (__DEV__) {
  // XMLHttpRequest logging
  const originalFetch = global.fetch;
  global.fetch = async (...args) => {
    console.log('FETCH:', args[0]);
    const response = await originalFetch(...args);
    console.log('RESPONSE:', response.status);
    return response;
  };
}
```

## Performance Metrics

### iOS Performance

```bash
# CPU and memory usage
xcrun simctl spawn booted log stream --predicate 'eventMessage contains "memory"'

# Instruments from command line
xcrun xctrace record --device booted --template "Time Profiler" --launch -- com.myapp
```

### Android Performance

```bash
# CPU info
adb shell dumpsys cpuinfo | grep myapp

# Memory info
adb shell dumpsys meminfo com.myapp

# GPU rendering
adb shell dumpsys gfxinfo com.myapp

# Frame rate monitoring
adb shell dumpsys SurfaceFlinger --latency
```

### React Native Performance

```tsx
// Enable performance monitoring
import { PerformanceObserver } from 'react-native-performance';

// In development
if (__DEV__) {
  const observer = new PerformanceObserver((list) => {
    list.getEntries().forEach((entry) => {
      console.log(`${entry.name}: ${entry.duration}ms`);
    });
  });
  observer.observe({ entryTypes: ['measure'] });
}
```

## Deep Links

Test deep link handling:

```bash
# iOS simulator
xcrun simctl openurl booted "myapp://settings"
xcrun simctl openurl booted "myapp://user/42"
xcrun simctl openurl booted "https://myapp.com/share/abc"  # Universal links

# Android emulator
adb shell am start -a android.intent.action.VIEW -d "myapp://settings"
adb shell am start -a android.intent.action.VIEW -d "https://myapp.com/share/abc"
```

### Verifying Deep Link Handling

```tsx
// In your app, verify routing works
import { useURL } from 'expo-linking';

function DeepLinkHandler() {
  const url = useURL();

  useEffect(() => {
    if (url) {
      console.log('Deep link received:', url);
    }
  }, [url]);
}
```

## Push Notifications

### iOS Simulator

```bash
# Send push notification to simulator
xcrun simctl push booted com.myapp notification.apns
```

Create `notification.apns`:

```json
{
  "aps": {
    "alert": {
      "title": "Test Notification",
      "body": "This is a test push notification"
    },
    "badge": 1,
    "sound": "default"
  },
  "customData": {
    "screen": "orders",
    "orderId": "12345"
  }
}
```

### Android Emulator

```bash
# Use Firebase CLI or adb
adb shell am broadcast \
  -a com.google.android.c2dm.intent.RECEIVE \
  -c com.myapp \
  --es "title" "Test Notification" \
  --es "body" "This is a test"
```

## Screenshots and Recording

```bash
# iOS: screenshot
xcrun simctl io booted screenshot screenshot.png

# iOS: record video
xcrun simctl io booted recordVideo output.mp4
# Press Ctrl+C to stop

# Android: screenshot
adb exec-out screencap -p > screenshot.png

# Android: record video
adb shell screenrecord /sdcard/recording.mp4
# Press Ctrl+C to stop, then pull:
adb pull /sdcard/recording.mp4
```

## Integration with Development Workflow

### Automated Test Script

```bash
#!/bin/bash
# run-e2e.sh - Example automated test flow

APP_SCHEME="myapp"
SIMULATOR="iPhone 16 Pro"

# 1. Boot simulator
xcrun simctl boot "$SIMULATOR" 2>/dev/null || true

# 2. Install and launch app
npx expo run:ios --device "$SIMULATOR" --no-bundler &
sleep 10

# 3. Capture initial state
xcrun simctl io booted screenshot "01-launch.png"

# 4. Test deep link
xcrun simctl openurl booted "$APP_SCHEME://settings"
sleep 2
xcrun simctl io booted screenshot "02-settings.png"

# 5. Test push notification
xcrun simctl push booted com.myapp test-notification.apns
sleep 1
xcrun simctl io booted screenshot "03-notification.png"

# 6. Dump accessibility tree for verification
xcrun simctl ui booted accessibility > accessibility-dump.txt

echo "Test artifacts captured."
```

### CI Integration

```yaml
# .github/workflows/e2e.yml
jobs:
  e2e-ios:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup
        run: |
          npm ci
          npx expo prebuild --platform ios
      - name: Boot Simulator
        run: |
          xcrun simctl boot "iPhone 16 Pro"
      - name: Build and Test
        run: |
          xcodebuild -workspace ios/*.xcworkspace \
            -scheme MyApp \
            -sdk iphonesimulator \
            -destination "name=iPhone 16 Pro" \
            test
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using pixel coordinates for taps | Use `testID` / accessibility identifiers instead |
| Hardcoding simulator names | Use `xcrun simctl list` to find available devices |
| Not waiting for app launch | Add appropriate delays or poll for accessibility elements |
| Missing `testID` on interactive elements | Add `testID` to all tappable/testable components |
| Forgetting to kill old simulator instances | Check `xcrun simctl list devices booted` first |
| Not handling Android input special characters | Use `%s` for space in `adb shell input text` |
| Recording without stopping | Always handle Ctrl+C or timeout for recordings |

## Quick Reference

| Task | iOS Command | Android Command |
|------|-------------|-----------------|
| Boot device | `xcrun simctl boot "Name"` | `emulator -avd Name` |
| Launch app | `xcrun simctl openurl booted scheme://` | `adb shell am start -d scheme://` |
| Screenshot | `xcrun simctl io booted screenshot f.png` | `adb exec-out screencap -p > f.png` |
| Record video | `xcrun simctl io booted recordVideo f.mp4` | `adb shell screenrecord /sdcard/f.mp4` |
| Type text | `xcrun simctl io booted input "text"` | `adb shell input text "text"` |
| Press back | N/A | `adb shell input keyevent KEYCODE_BACK` |
| Send push | `xcrun simctl push booted id file.apns` | `adb shell am broadcast ...` |
| Stream logs | `xcrun simctl spawn booted log stream` | `adb logcat` |
| Accessibility dump | `xcrun simctl ui booted accessibility` | `adb shell uiautomator dump` |
| Deep link | `xcrun simctl openurl booted url` | `adb shell am start -d url` |
