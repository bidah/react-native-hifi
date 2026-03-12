---
name: building-native-ui
description: Use when building user interfaces in React Native or Expo, implementing navigation, layouts, animations, native controls, modals, gesture handling, or styling components
---

# Building Native UI with React Native and Expo

## Overview

React Native UI differs fundamentally from web UI. There is no CSS, no DOM, no browser layout engine. Everything renders to native platform views through a bridge. Understanding this architecture prevents the most common mistakes.

**Core principle:** Think in native components and Flexbox, not in HTML/CSS. Every visual element maps to a platform-native view.

## When to Use

- Building screens, components, or layouts in React Native
- Implementing navigation between screens
- Adding animations or gesture interactions
- Using native platform controls (switches, sliders, segmented controls)
- Styling components (StyleSheet, not CSS)
- Creating modals, form sheets, or bottom sheets

## Navigation with Expo Router

Expo Router uses **file-based routing**, similar to Next.js. Files in the `app/` directory automatically become routes.

### Directory Structure

```
app/
  _layout.tsx        # Root layout (wraps all routes)
  index.tsx          # Home screen (/)
  settings.tsx       # /settings
  (tabs)/
    _layout.tsx      # Tab navigator layout
    home.tsx         # Tab: home
    profile.tsx      # Tab: profile
  [id].tsx           # Dynamic route: /123, /abc
  (auth)/
    login.tsx        # Group: doesn't affect URL
```

### Layout Files

```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="index" options={{ title: 'Home' }} />
      <Stack.Screen name="settings" options={{ headerShown: false }} />
    </Stack>
  );
}
```

### Navigation

```tsx
import { Link, useRouter } from 'expo-router';

// Declarative
<Link href="/settings">Go to Settings</Link>
<Link href={{ pathname: '/user/[id]', params: { id: '42' } }}>User 42</Link>

// Imperative
const router = useRouter();
router.push('/settings');
router.replace('/login');  // No back button
router.back();
```

### Tab Navigation

```tsx
// app/(tabs)/_layout.tsx
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function TabLayout() {
  return (
    <Tabs>
      <Tabs.Screen
        name="home"
        options={{
          title: 'Home',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="home" size={size} color={color} />
          ),
        }}
      />
    </Tabs>
  );
}
```

## Flexbox Layouts

React Native uses Flexbox by default. Key differences from web Flexbox:

| Property | React Native Default | Web Default |
|----------|---------------------|-------------|
| `flexDirection` | `column` | `row` |
| `alignContent` | `flex-start` | `stretch` |
| Units | Unitless (dp) | px, em, rem, etc. |

### Common Layout Patterns

```tsx
// Centering content
const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
});

// Row with spacing
const styles = StyleSheet.create({
  row: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingHorizontal: 16,
  },
});

// Scrollable list with fixed header/footer
const styles = StyleSheet.create({
  container: { flex: 1 },
  header: { height: 60 },
  content: { flex: 1 },  // Takes remaining space
  footer: { height: 80 },
});
```

### Safe Areas

Always account for notches, status bars, and home indicators:

```tsx
import { SafeAreaView } from 'react-native-safe-area-context';

export default function Screen() {
  return (
    <SafeAreaView style={{ flex: 1 }} edges={['top', 'bottom']}>
      {/* Content */}
    </SafeAreaView>
  );
}
```

## React Native Reanimated

For performant, 60fps animations that run on the native thread.

### Basic Animation

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
} from 'react-native-reanimated';

function AnimatedBox() {
  const offset = useSharedValue(0);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: offset.value }],
  }));

  return (
    <Animated.View style={[styles.box, animatedStyle]}>
      <Pressable onPress={() => {
        offset.value = withSpring(offset.value === 0 ? 200 : 0);
      }}>
        <Text>Tap me</Text>
      </Pressable>
    </Animated.View>
  );
}
```

### Entering/Exiting Animations

```tsx
import Animated, { FadeIn, SlideOutRight } from 'react-native-reanimated';

<Animated.View entering={FadeIn.duration(500)} exiting={SlideOutRight}>
  <Text>I animate in and out</Text>
</Animated.View>
```

### Gesture Handler + Reanimated

```tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withDecay,
} from 'react-native-reanimated';

function DraggableCard() {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);

  const pan = Gesture.Pan()
    .onUpdate((event) => {
      translateX.value = event.translationX;
      translateY.value = event.translationY;
    })
    .onEnd((event) => {
      translateX.value = withDecay({ velocity: event.velocityX });
      translateY.value = withDecay({ velocity: event.velocityY });
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
    ],
  }));

  return (
    <GestureDetector gesture={pan}>
      <Animated.View style={[styles.card, animatedStyle]} />
    </GestureDetector>
  );
}
```

## Native iOS Controls via @expo/ui

Use platform-native controls that match iOS design language:

```tsx
import { Switch, Slider, SegmentedControl } from '@expo/ui';

function SettingsScreen() {
  const [enabled, setEnabled] = useState(false);
  const [volume, setVolume] = useState(0.5);
  const [tab, setTab] = useState(0);

  return (
    <View>
      <Switch value={enabled} onValueChange={setEnabled} />

      <Slider
        value={volume}
        onValueChange={setVolume}
        minimumValue={0}
        maximumValue={1}
      />

      <SegmentedControl
        values={['Daily', 'Weekly', 'Monthly']}
        selectedIndex={tab}
        onValueChange={setTab}
      />
    </View>
  );
}
```

## SF Symbols Integration

Use Apple's SF Symbols icon set on iOS:

```tsx
import { SymbolView } from 'expo-symbols';

<SymbolView
  name="heart.fill"
  style={{ width: 24, height: 24 }}
  tintColor="red"
  weight="semibold"
/>
```

Common symbol names: `chevron.right`, `gear`, `person.fill`, `star.fill`, `bell.fill`, `magnifyingglass`, `plus.circle.fill`.

## Modals and Form Sheets

### Stack Modal

```tsx
// app/_layout.tsx
<Stack>
  <Stack.Screen name="index" />
  <Stack.Screen
    name="modal"
    options={{
      presentation: 'modal',
      headerTitle: 'Settings',
    }}
  />
</Stack>

// app/modal.tsx - presented as native modal
```

### Form Sheet (iOS 16+)

```tsx
<Stack.Screen
  name="form-sheet"
  options={{
    presentation: 'formSheet',
    sheetAllowedDetents: [0.5, 1.0],  // Half and full screen
    sheetGrabberVisible: true,
    sheetCornerRadius: 24,
  }}
/>
```

## Host Component Wrapping

When you need to wrap native views for use with platform-specific APIs:

```tsx
import { requireNativeView } from 'expo';

const NativeMapView = requireNativeView('MapView');

function Map({ region, onRegionChange }) {
  return (
    <NativeMapView
      style={{ flex: 1 }}
      region={region}
      onRegionChange={onRegionChange}
    />
  );
}
```

## Styling in React Native

### StyleSheet API

```tsx
import { StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
    padding: 16,
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 8,
  },
  // Shadow (iOS)
  card: {
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    // Shadow (Android)
    elevation: 3,
    borderRadius: 8,
    backgroundColor: '#fff',
    padding: 16,
  },
});
```

### Key Differences from CSS

- No cascading: styles don't inherit (except `Text` within `Text`)
- No units: all numbers are density-independent pixels
- No shorthand: `margin: 16` works, but `margin: '16px 8px'` does not
- camelCase: `backgroundColor` not `background-color`
- No pseudo-classes: use `Pressable` with style functions for hover/press states
- Platform-specific shadows: `shadowX` for iOS, `elevation` for Android

### Pressable Styling

```tsx
<Pressable
  style={({ pressed }) => [
    styles.button,
    pressed && styles.buttonPressed,
  ]}
>
  {({ pressed }) => (
    <Text style={pressed ? styles.textPressed : styles.text}>
      Press me
    </Text>
  )}
</Pressable>
```

### Platform-Specific Styles

```tsx
import { Platform, StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  container: {
    ...Platform.select({
      ios: { shadowColor: '#000', shadowOpacity: 0.1 },
      android: { elevation: 3 },
    }),
  },
});
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `100vh` or CSS units | Use `flex: 1` or `Dimensions.get('window').height` |
| Forgetting SafeAreaView | Always wrap top-level screens in SafeAreaView |
| Animating with `setState` | Use Reanimated shared values for 60fps animations |
| Nesting `ScrollView` in `ScrollView` | Use `FlatList` with `ListHeaderComponent` |
| Using `overflow: 'hidden'` expecting CSS behavior | Android clips children differently; test both platforms |
| Hardcoding dimensions | Use `flex`, `%`, or `useWindowDimensions()` |
| Missing `key` prop in lists | Use `FlatList` with `keyExtractor` instead of `.map()` |

## Quick Reference

| Task | Solution |
|------|----------|
| Navigate to screen | `router.push('/path')` or `<Link href="/path">` |
| Tab navigation | `(tabs)/_layout.tsx` with `<Tabs>` |
| Center content | `justifyContent: 'center', alignItems: 'center'` |
| Spring animation | `offset.value = withSpring(target)` |
| Native switch | `<Switch>` from `@expo/ui` |
| Modal screen | `presentation: 'modal'` in Stack options |
| Platform check | `Platform.OS === 'ios'` |
| Safe area | `<SafeAreaView edges={['top']}>` |
