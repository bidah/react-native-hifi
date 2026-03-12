---
name: expo-tailwind-setup
description: Use when setting up or using Tailwind CSS v4 with NativeWind v5 in React Native or Expo projects, configuring universal styling across iOS Android and web, or implementing CSS-first theming
---

# Tailwind CSS v4 + NativeWind v5 for Expo

## Overview

NativeWind v5 brings Tailwind CSS v4 to React Native, enabling a single styling system across iOS, Android, and web. It compiles Tailwind classes into React Native compatible styles at build time using `react-native-css`.

**Core principle:** Write Tailwind classes once, render natively on all platforms. NativeWind translates CSS concepts into React Native's styling system, not a webview.

## When to Use

- Starting a new Expo/React Native project and want utility-first styling
- Porting a web app with Tailwind to React Native
- Building a universal app (iOS + Android + Web) with shared styling
- Wanting dark mode, responsive design, or theme variables without manual StyleSheet management

## Installation

### New Project

```bash
npx create-expo-app@latest my-app
cd my-app
npx expo install nativewind tailwindcss react-native-css
```

### Existing Project

```bash
npx expo install nativewind tailwindcss react-native-css
```

### Configuration

Create `global.css` in your project root:

```css
/* global.css */
@import "tailwindcss";
```

Add the CSS import to your root layout:

```tsx
// app/_layout.tsx
import "../global.css";

import { Stack } from "expo-router";

export default function RootLayout() {
  return <Stack />;
}
```

Update `metro.config.js`:

```js
// metro.config.js
const { getDefaultConfig } = require("expo/metro-config");
const { withNativeWind } = require("nativewind/metro");

const config = getDefaultConfig(__dirname);

module.exports = withNativeWind(config, {
  input: "./global.css",
});
```

Add the babel preset (if not using the new React compiler):

```js
// babel.config.js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: [
      ["babel-preset-expo", { jsxImportSource: "nativewind" }],
    ],
  };
};
```

For TypeScript, create or update `nativewind-env.d.ts`:

```ts
/// <reference types="nativewind/types" />
```

## Basic Usage

```tsx
import { View, Text, Pressable } from "react-native";

export default function HomeScreen() {
  return (
    <View className="flex-1 items-center justify-center bg-white dark:bg-gray-900">
      <Text className="text-2xl font-bold text-gray-900 dark:text-white">
        Hello NativeWind
      </Text>
      <Pressable className="mt-4 rounded-lg bg-blue-500 px-6 py-3 active:bg-blue-600">
        <Text className="text-base font-semibold text-white">
          Press Me
        </Text>
      </Pressable>
    </View>
  );
}
```

## CSS-First Theming

Tailwind CSS v4 uses CSS custom properties for theming. Define your design tokens in `global.css`:

```css
/* global.css */
@import "tailwindcss";

@theme {
  /* Colors */
  --color-primary: #3b82f6;
  --color-primary-dark: #2563eb;
  --color-secondary: #8b5cf6;
  --color-surface: #ffffff;
  --color-surface-dark: #1f2937;

  /* Spacing */
  --spacing-card: 16px;
  --spacing-section: 24px;

  /* Border radius */
  --radius-card: 12px;
  --radius-button: 8px;

  /* Font sizes */
  --font-size-heading: 24px;
  --font-size-body: 16px;
}
```

Use your custom tokens directly in classes:

```tsx
<View className="rounded-card bg-surface p-card dark:bg-surface-dark">
  <Text className="text-heading font-bold text-primary">
    Themed Component
  </Text>
</View>
```

## Dark Mode

NativeWind supports dark mode through Tailwind's `dark:` variant:

```tsx
<View className="bg-white dark:bg-gray-950">
  <Text className="text-gray-900 dark:text-gray-100">
    Adapts to system theme
  </Text>
</View>
```

To control dark mode programmatically:

```tsx
import { useColorScheme } from "nativewind";

function ThemeToggle() {
  const { colorScheme, setColorScheme } = useColorScheme();

  return (
    <Pressable onPress={() => setColorScheme(colorScheme === "dark" ? "light" : "dark")}>
      <Text className="text-gray-900 dark:text-white">
        Current: {colorScheme}
      </Text>
    </Pressable>
  );
}
```

## Custom Theme Variables

Override or extend Tailwind defaults with CSS variables for runtime theming:

```css
/* global.css */
@import "tailwindcss";

:root {
  --color-brand: #3b82f6;
  --color-brand-text: #ffffff;
}

.theme-sunset {
  --color-brand: #f97316;
  --color-brand-text: #ffffff;
}

.theme-forest {
  --color-brand: #22c55e;
  --color-brand-text: #ffffff;
}
```

```tsx
<View className="theme-sunset">
  <View className="bg-[--color-brand] p-4 rounded-lg">
    <Text className="text-[--color-brand-text]">Sunset Theme</Text>
  </View>
</View>
```

## Platform-Specific Styling

NativeWind provides platform variants via media queries:

```tsx
{/* iOS only */}
<View className="ios:bg-blue-500 android:bg-green-500">
  <Text className="ios:font-bold android:font-medium">
    Platform specific
  </Text>
</View>

{/* Web only */}
<View className="web:hover:bg-gray-100 web:cursor-pointer">
  <Text>Hoverable on web</Text>
</View>
```

### Responsive Design

```tsx
{/* Mobile-first responsive */}
<View className="flex-col sm:flex-row">
  <View className="w-full sm:w-1/2 p-4">
    <Text>Left column</Text>
  </View>
  <View className="w-full sm:w-1/2 p-4">
    <Text>Right column</Text>
  </View>
</View>
```

## Supported Features

### What Works

| Feature | Example |
|---------|---------|
| Flexbox | `flex-1`, `flex-row`, `items-center`, `justify-between` |
| Spacing | `p-4`, `mx-2`, `gap-3` |
| Sizing | `w-full`, `h-16`, `min-h-screen` |
| Typography | `text-lg`, `font-bold`, `text-center`, `leading-6` |
| Colors | `bg-blue-500`, `text-gray-900`, `border-red-300` |
| Borders | `rounded-lg`, `border`, `border-b-2` |
| Shadows | `shadow-sm`, `shadow-lg` (iOS; use `elevation-*` for Android) |
| Opacity | `opacity-50`, `bg-black/50` (background opacity) |
| Dark mode | `dark:bg-gray-900`, `dark:text-white` |
| Transforms | `rotate-45`, `scale-110`, `translate-x-4` |
| Transitions | `transition-colors`, `duration-200` |
| Arbitrary values | `w-[120px]`, `bg-[#1DA1F2]`, `text-[14px]` |
| Active state | `active:bg-blue-600`, `active:scale-95` |
| Platform | `ios:*`, `android:*`, `web:*` |

### What Does NOT Work

| CSS Feature | Reason | Alternative |
|-------------|--------|-------------|
| `hover:` | No hover on mobile | Use `active:` for press states |
| `grid` | Not supported in RN | Use Flexbox (`flex-row flex-wrap`) |
| `backdrop-blur` | Limited RN support | Use `expo-blur` |
| `animation` keyframes | Use Reanimated | `react-native-reanimated` |
| Pseudo-elements `::before` | No DOM | Create separate `View` elements |
| `overflow: auto` | Different scroll model | Use `ScrollView` or `FlatList` |

## Styling Third-Party Components

For components that do not accept `className`:

```tsx
import { cssInterop } from "nativewind";
import { Image } from "expo-image";

// Register the component to accept className
cssInterop(Image, { className: "style" });

// Now you can use className
<Image className="h-48 w-full rounded-lg" source={{ uri: imageUrl }} />
```

## TypeScript Configuration

Ensure type checking works with NativeWind:

```ts
// nativewind-env.d.ts
/// <reference types="nativewind/types" />
```

This adds the `className` prop to all React Native core components.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Missing `global.css` import in layout | Add `import "../global.css"` to `app/_layout.tsx` |
| Forgetting metro config | Must use `withNativeWind` wrapper in `metro.config.js` |
| Using CSS units (`px`, `rem`) | NativeWind handles conversion; use Tailwind classes as-is |
| Using web-only utilities | Check the supported features table above |
| Not clearing metro cache after setup | Run `npx expo start --clear` |
| Using `style` and `className` together | Both work; `style` overrides `className` for same properties |

## Quick Reference

| Task | Approach |
|------|----------|
| Install | `npx expo install nativewind tailwindcss react-native-css` |
| CSS entry point | `global.css` with `@import "tailwindcss"` |
| Metro config | `withNativeWind(config, { input: "./global.css" })` |
| Dark mode | `dark:` prefix on any utility |
| Platform style | `ios:`, `android:`, `web:` prefixes |
| Custom theme | `@theme { --color-*: value }` in `global.css` |
| Third-party components | `cssInterop(Component, { className: "style" })` |
| Clear cache | `npx expo start --clear` |
