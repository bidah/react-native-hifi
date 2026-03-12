---
name: expo-ui-jetpack-compose
description: Use when building Android-specific UI components using Jetpack Compose views in Expo, implementing LazyColumn scrolling lists, or rendering XML vector drawable icons on Android
---

# Android UI with Jetpack Compose via @expo/ui

## Overview

`@expo/ui/jetpack-compose` provides Android-native Jetpack Compose views directly in React Native Expo apps. These components render using Android's modern declarative UI framework, giving you truly native Android look-and-feel without writing Kotlin.

**Core principle:** Use platform-specific `@expo/ui` imports to get native-quality components that follow each platform's design language. On Android, this means Jetpack Compose.

## When to Use

- Building Android-specific screens or components that must feel native
- Needing performant scrolling lists on Android (LazyColumn)
- Rendering Material Design 3 components natively
- Displaying Android XML vector drawable icons
- Building platform-specific UI alongside cross-platform shared code

## Import Pattern

```tsx
// Android-specific import
import { Button, LazyColumn, Icon } from '@expo/ui/jetpack-compose';

// For platform-specific rendering
import { Platform } from 'react-native';

function MyComponent() {
  if (Platform.OS === 'android') {
    return <AndroidView />;
  }
  return <IOSView />;
}
```

## LazyColumn for Scrolling Lists

`LazyColumn` is the Jetpack Compose equivalent of `RecyclerView`. It lazily composes and lays out only the visible items, making it efficient for large datasets.

### Basic Usage

```tsx
import { LazyColumn, Section, Item, Text } from '@expo/ui/jetpack-compose';

function ContactList() {
  const contacts = [
    { id: '1', name: 'Alice', phone: '555-0101' },
    { id: '2', name: 'Bob', phone: '555-0102' },
    { id: '3', name: 'Charlie', phone: '555-0103' },
  ];

  return (
    <LazyColumn style={{ flex: 1 }}>
      <Section header="Contacts">
        {contacts.map((contact) => (
          <Item key={contact.id}>
            <Text style={{ fontSize: 16, fontWeight: 'bold' }}>
              {contact.name}
            </Text>
            <Text style={{ fontSize: 14, color: '#666' }}>
              {contact.phone}
            </Text>
          </Item>
        ))}
      </Section>
    </LazyColumn>
  );
}
```

### Sectioned Lists

```tsx
import { LazyColumn, Section, Item, Text } from '@expo/ui/jetpack-compose';

function SettingsScreen() {
  return (
    <LazyColumn style={{ flex: 1 }}>
      <Section header="Account">
        <Item onPress={() => console.log('Profile')}>
          <Text>Profile</Text>
        </Item>
        <Item onPress={() => console.log('Security')}>
          <Text>Security</Text>
        </Item>
      </Section>

      <Section header="Preferences">
        <Item onPress={() => console.log('Notifications')}>
          <Text>Notifications</Text>
        </Item>
        <Item onPress={() => console.log('Theme')}>
          <Text>Theme</Text>
        </Item>
      </Section>

      <Section header="About">
        <Item>
          <Text>Version 1.0.0</Text>
        </Item>
      </Section>
    </LazyColumn>
  );
}
```

## Icon Component for XML Vector Drawables

Render Android XML vector drawables as icons:

```tsx
import { Icon } from '@expo/ui/jetpack-compose';

function NavigationBar() {
  return (
    <View style={{ flexDirection: 'row', justifyContent: 'space-around' }}>
      <Icon
        name="home"
        size={24}
        tintColor="#333"
      />
      <Icon
        name="search"
        size={24}
        tintColor="#333"
      />
      <Icon
        name="settings"
        size={24}
        tintColor="#333"
      />
    </View>
  );
}
```

### Using Material Icons

```tsx
import { Icon } from '@expo/ui/jetpack-compose';

// Material Design icons available by name
<Icon name="favorite" size={24} tintColor="red" />
<Icon name="share" size={24} tintColor="#1976D2" />
<Icon name="more_vert" size={24} tintColor="#757575" />
```

## Button Component

Native Material Design 3 buttons:

```tsx
import { Button } from '@expo/ui/jetpack-compose';

function ActionButtons() {
  return (
    <View style={{ gap: 12 }}>
      <Button
        title="Filled Button"
        onPress={() => console.log('Pressed')}
        variant="filled"
      />
      <Button
        title="Outlined Button"
        onPress={() => console.log('Pressed')}
        variant="outlined"
      />
      <Button
        title="Text Button"
        onPress={() => console.log('Pressed')}
        variant="text"
      />
    </View>
  );
}
```

## Host Component Wrapping

When you need to wrap Jetpack Compose views for use within React Native's view hierarchy:

```tsx
import { requireNativeView } from 'expo';

// Wrap a native Android Compose component
const NativeComposeView = requireNativeView('MyComposeView');

function ComposeWrapper({ data, onAction }) {
  return (
    <NativeComposeView
      style={{ flex: 1 }}
      data={data}
      onAction={onAction}
    />
  );
}
```

### Writing a Custom Compose Module

If you need a Compose component not available in `@expo/ui`:

```kotlin
// android/src/main/java/com/myapp/MyComposeView.kt
package com.myapp

import expo.modules.kotlin.modules.Module
import expo.modules.kotlin.modules.ModuleDefinition

class MyComposeModule : Module() {
  override fun definition() = ModuleDefinition {
    Name("MyComposeView")

    View(MyComposeView::class) {
      Prop("title") { view: MyComposeView, title: String ->
        view.setTitle(title)
      }

      Events("onAction")
    }
  }
}
```

## Platform-Conditional Rendering

Pattern for using Jetpack Compose components alongside cross-platform code:

```tsx
import { Platform, View, Text } from 'react-native';

// Lazy imports to avoid loading on wrong platform
const AndroidList = Platform.OS === 'android'
  ? require('@expo/ui/jetpack-compose').LazyColumn
  : null;

function AdaptiveList({ items }) {
  if (Platform.OS === 'android' && AndroidList) {
    return (
      <AndroidList style={{ flex: 1 }}>
        {/* Android-native rendering */}
      </AndroidList>
    );
  }

  // Fallback for iOS/web
  return (
    <FlatList
      data={items}
      renderItem={({ item }) => <Text>{item.name}</Text>}
    />
  );
}
```

### File-Based Platform Splits

For larger platform differences, use Expo Router's platform file extensions:

```
components/
  SettingsList.tsx           # Shared/default
  SettingsList.android.tsx   # Android with Jetpack Compose
  SettingsList.ios.tsx       # iOS with SwiftUI
```

```tsx
// components/SettingsList.android.tsx
import { LazyColumn, Section, Item, Text } from '@expo/ui/jetpack-compose';

export function SettingsList({ sections }) {
  return (
    <LazyColumn style={{ flex: 1 }}>
      {sections.map((section) => (
        <Section key={section.title} header={section.title}>
          {section.items.map((item) => (
            <Item key={item.id} onPress={item.onPress}>
              <Text>{item.label}</Text>
            </Item>
          ))}
        </Section>
      ))}
    </LazyColumn>
  );
}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Importing `@expo/ui/jetpack-compose` on iOS | Guard imports with `Platform.OS` check or use platform file extensions |
| Using `FlatList` patterns with `LazyColumn` | `LazyColumn` uses `Section`/`Item` children, not `renderItem` |
| Forgetting to rebuild after adding native dependency | Run `eas build` or `npx expo run:android` after adding `@expo/ui` |
| Mixing Compose and View styling | Compose components have their own styling; use props, not `StyleSheet` |
| Expecting identical rendering across platforms | Jetpack Compose follows Material Design; iOS follows Human Interface Guidelines |

## Quick Reference

| Task | Pattern |
|------|---------|
| Import Compose components | `import { ... } from '@expo/ui/jetpack-compose'` |
| Scrolling list | `<LazyColumn>` with `<Section>` and `<Item>` |
| Icon | `<Icon name="icon_name" size={24} />` |
| Button | `<Button title="Label" variant="filled" onPress={fn} />` |
| Platform guard | `Platform.OS === 'android'` or `.android.tsx` file |
| Custom native view | `requireNativeView('ViewName')` |
| Host wrapping | Expo Modules API with Kotlin + Compose |
