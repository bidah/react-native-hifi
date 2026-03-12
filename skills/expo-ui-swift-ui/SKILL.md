---
name: expo-ui-swift-ui
description: Use when building iOS-specific UI components using SwiftUI views in Expo, embedding React Native views in SwiftUI with RNHostView, or implementing native iOS interfaces beyond cross-platform components
---

# iOS UI with SwiftUI via @expo/ui

## Overview

`@expo/ui/swift-ui` provides iOS-native SwiftUI views directly in React Native Expo apps. The API mirrors SwiftUI naming conventions, so iOS developers find it immediately familiar. These components render as true SwiftUI views, not approximations.

**Core principle:** Use `@expo/ui/swift-ui` for iOS-specific screens that must feel fully native. Combine with `RNHostView` to embed React Native content inside SwiftUI hierarchies when needed.

## When to Use

- Building iOS-specific settings screens, forms, or detail views
- Needing native iOS list styles (inset grouped, sidebar)
- Implementing SwiftUI navigation patterns (NavigationStack, NavigationSplitView)
- Embedding React Native views inside SwiftUI layouts via `RNHostView`
- Using iOS-specific controls not available in cross-platform React Native

## Import Pattern

```tsx
// iOS-specific SwiftUI components
import { List, Section, Label, Toggle, Picker, Gauge } from '@expo/ui/swift-ui';

// Cross-platform @expo/ui components (Switch, Slider, etc.) work on both platforms
import { Switch, Slider } from '@expo/ui';
```

## List and Section

SwiftUI's `List` provides native iOS list rendering with section headers, footers, and grouped styles:

```tsx
import { List, Section, Label, Text } from '@expo/ui/swift-ui';

function SettingsScreen() {
  return (
    <List style={{ flex: 1 }}>
      <Section header="General" footer="Customize your app experience">
        <Label
          title="Language"
          systemImage="globe"
          onPress={() => router.push('/language')}
        />
        <Label
          title="Notifications"
          systemImage="bell"
          onPress={() => router.push('/notifications')}
        />
      </Section>

      <Section header="Account">
        <Label
          title="Profile"
          systemImage="person.circle"
          onPress={() => router.push('/profile')}
        />
        <Label
          title="Privacy"
          systemImage="lock.shield"
          onPress={() => router.push('/privacy')}
        />
      </Section>
    </List>
  );
}
```

### List Styles

```tsx
// Inset grouped (default iOS Settings style)
<List listStyle="insetGrouped" style={{ flex: 1 }}>
  {/* ... */}
</List>

// Plain list
<List listStyle="plain" style={{ flex: 1 }}>
  {/* ... */}
</List>

// Sidebar (iPad)
<List listStyle="sidebar" style={{ flex: 1 }}>
  {/* ... */}
</List>
```

## Toggle

Native iOS toggle with SwiftUI styling:

```tsx
import { Toggle } from '@expo/ui/swift-ui';
import { useState } from 'react';

function NotificationSettings() {
  const [pushEnabled, setPushEnabled] = useState(true);
  const [soundEnabled, setSoundEnabled] = useState(false);

  return (
    <List style={{ flex: 1 }}>
      <Section header="Notifications">
        <Toggle
          value={pushEnabled}
          onValueChange={setPushEnabled}
          label="Push Notifications"
          systemImage="bell.fill"
        />
        <Toggle
          value={soundEnabled}
          onValueChange={setSoundEnabled}
          label="Sound"
          systemImage="speaker.wave.2"
        />
      </Section>
    </List>
  );
}
```

## Picker

Native iOS picker with multiple styles:

```tsx
import { Picker } from '@expo/ui/swift-ui';
import { useState } from 'react';

function ThemePicker() {
  const [theme, setTheme] = useState('system');

  return (
    <Picker
      selectedValue={theme}
      onValueChange={setTheme}
      label="Theme"
      style="menu"  // 'menu', 'wheel', 'segmented', 'inline'
    >
      <Picker.Item label="System" value="system" />
      <Picker.Item label="Light" value="light" />
      <Picker.Item label="Dark" value="dark" />
    </Picker>
  );
}
```

## Gauge

Native iOS gauge for displaying progress or levels:

```tsx
import { Gauge } from '@expo/ui/swift-ui';

function StorageIndicator({ used, total }) {
  return (
    <Gauge
      value={used / total}
      label="Storage"
      currentValueLabel={`${used} GB`}
      minimumValueLabel="0"
      maximumValueLabel={`${total} GB`}
      gaugeStyle="accessoryCircular"  // 'linearCapacity', 'accessoryCircular', 'accessoryLinear'
    />
  );
}
```

## Label

SwiftUI-style label with system image:

```tsx
import { Label } from '@expo/ui/swift-ui';

<Label
  title="Downloads"
  systemImage="arrow.down.circle"
  tintColor="#007AFF"
/>

<Label
  title="Favorites"
  subtitle="12 items"
  systemImage="heart.fill"
  tintColor="#FF3B30"
/>
```

## RNHostView: Embedding React Native in SwiftUI

`RNHostView` lets you embed React Native components inside a SwiftUI view hierarchy. This is the inverse of using SwiftUI in React Native.

### Use Cases

- A SwiftUI screen that needs a complex React Native component (chart, custom view)
- Widgets or App Clips that host React Native content
- Gradual migration from React Native to SwiftUI

### Usage

```tsx
import { RNHostView } from '@expo/ui/swift-ui';

// In your SwiftUI-driven layout
function HybridScreen() {
  return (
    <List style={{ flex: 1 }}>
      <Section header="Native Section">
        <Label title="Native Row" systemImage="star" />
      </Section>

      <Section header="React Native Content">
        <RNHostView style={{ height: 200 }}>
          {/* This React Native subtree renders inside SwiftUI */}
          <View style={{ flex: 1, backgroundColor: '#f0f0f0', borderRadius: 8 }}>
            <Text style={{ padding: 16, fontSize: 16 }}>
              React Native content embedded in SwiftUI
            </Text>
            <CustomChart data={chartData} />
          </View>
        </RNHostView>
      </Section>
    </List>
  );
}
```

### Constraints

- `RNHostView` children receive a fixed frame from SwiftUI
- Layout is driven by SwiftUI, not React Native's Flexbox
- Props passed to children must be serializable
- Performance is good for static/semi-static content; avoid heavy animation mixing

## Host Component Wrapping

Wrap custom SwiftUI views for use in React Native:

```tsx
import { requireNativeView } from 'expo';

const NativeSwiftUIView = requireNativeView('MySwiftUIView');

function CustomNativeView({ title, onAction }) {
  return (
    <NativeSwiftUIView
      style={{ height: 200 }}
      title={title}
      onAction={onAction}
    />
  );
}
```

### Writing a Custom SwiftUI Module

```swift
// ios/MySwiftUIView.swift
import ExpoModulesCore
import SwiftUI

class MySwiftUIModule: Module {
  public func definition() -> ModuleDefinition {
    Name("MySwiftUIView")

    View(MySwiftUIView.self) {
      Prop("title") { (view, title: String) in
        view.title = title
      }

      Events("onAction")
    }
  }
}

struct MySwiftUIView: ExpoView {
  @State var title: String = ""
  let onAction = EventDispatcher()

  var body: some View {
    VStack {
      Text(title)
        .font(.headline)
      Button("Tap Me") {
        onAction(["action": "tapped"])
      }
    }
  }
}
```

## Platform-Conditional Rendering

### File-Based Platform Splits

```
components/
  SettingsForm.tsx        # Shared logic/default
  SettingsForm.ios.tsx    # SwiftUI-based iOS version
  SettingsForm.android.tsx # Jetpack Compose Android version
```

```tsx
// components/SettingsForm.ios.tsx
import { List, Section, Toggle, Picker } from '@expo/ui/swift-ui';

export function SettingsForm({ settings, onUpdate }) {
  return (
    <List listStyle="insetGrouped" style={{ flex: 1 }}>
      <Section header="Appearance">
        <Toggle
          value={settings.darkMode}
          onValueChange={(v) => onUpdate('darkMode', v)}
          label="Dark Mode"
          systemImage="moon.fill"
        />
        <Picker
          selectedValue={settings.accentColor}
          onValueChange={(v) => onUpdate('accentColor', v)}
          label="Accent Color"
          style="menu"
        >
          <Picker.Item label="Blue" value="blue" />
          <Picker.Item label="Purple" value="purple" />
          <Picker.Item label="Green" value="green" />
        </Picker>
      </Section>
    </List>
  );
}
```

### Inline Platform Check

```tsx
import { Platform } from 'react-native';

function AdaptiveSettings() {
  if (Platform.OS === 'ios') {
    const { List, Section, Toggle } = require('@expo/ui/swift-ui');
    return (
      <List listStyle="insetGrouped" style={{ flex: 1 }}>
        {/* SwiftUI rendering */}
      </List>
    );
  }

  // Cross-platform fallback
  return <ScrollView>{/* ... */}</ScrollView>;
}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Importing `@expo/ui/swift-ui` on Android | Guard with `Platform.OS` or use `.ios.tsx` file extension |
| Applying React Native `StyleSheet` to SwiftUI components | SwiftUI components use their own props for styling; use `listStyle`, `gaugeStyle`, etc. |
| Oversizing `RNHostView` content | Content must fit within the SwiftUI-provided frame; set explicit `height` |
| Not rebuilding after adding `@expo/ui` | Requires native rebuild: `npx expo run:ios` or `eas build` |
| Mixing SwiftUI navigation with Expo Router | Use Expo Router for screen navigation; SwiftUI for in-screen UI only |
| Expecting SwiftUI hot reload | SwiftUI views require a native rebuild when the Swift code changes |

## Quick Reference

| Task | Pattern |
|------|---------|
| Import SwiftUI components | `import { ... } from '@expo/ui/swift-ui'` |
| Grouped list | `<List listStyle="insetGrouped">` |
| Section with header | `<Section header="Title">` |
| Toggle | `<Toggle value={v} onValueChange={fn} label="Text" />` |
| Picker | `<Picker selectedValue={v} onValueChange={fn} style="menu">` |
| System icon label | `<Label title="Text" systemImage="icon.name" />` |
| Embed RN in SwiftUI | `<RNHostView><ReactNativeContent /></RNHostView>` |
| Custom SwiftUI module | Expo Modules API with Swift + SwiftUI |
| Platform guard | `Platform.OS === 'ios'` or `.ios.tsx` file |
