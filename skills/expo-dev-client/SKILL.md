---
name: expo-dev-client
description: Use when building custom development clients for Expo apps, integrating custom native modules, using Apple ecosystem features, third-party native SDKs, or when Expo Go limitations are encountered
---

# Expo Dev Client

## Overview

A custom development client is a debug build of your app that includes your specific native modules and configuration. It replaces Expo Go when you need native code that Expo Go does not bundle.

**Core principle:** Use Expo Go for rapid prototyping. Switch to a dev client when you need custom native code, and use EAS Build to create and distribute it.

## When to Use

Use a dev client when your app requires:

- **Custom native modules** not included in Expo Go (e.g., Bluetooth, NFC, custom camera pipelines)
- **Apple ecosystem integration** (HealthKit, HomeKit, CarPlay, App Clips)
- **Third-party native SDKs** (Stripe native, Mapbox, Firebase Analytics with custom config)
- **Custom native configuration** (specific entitlements, background modes, custom URL schemes)
- **Config plugins** that modify native project files
- **Expo Modules API** for writing your own native modules

### When NOT to Use

- App only uses libraries included in Expo Go (check the Expo SDK docs)
- Quick prototyping or demo purposes (stick with Expo Go)
- Web-only features

## Expo Go vs Dev Client

| Feature | Expo Go | Dev Client |
|---------|---------|------------|
| Native code | Pre-bundled only | Any native module |
| Setup time | Zero | Requires build step |
| OTA updates | Yes | Yes |
| Custom config plugins | No | Yes |
| App Store distribution | No | Yes (via TestFlight) |
| Custom app icon/splash | No | Yes |
| Background modes | Limited | Full control |
| Fast Refresh | Yes | Yes |

## Setup

### 1. Install the Dev Client Package

```bash
npx expo install expo-dev-client
```

This adds the development client UI (launcher, error overlay, dev menu) to your app.

### 2. Configure EAS Build

```bash
# Install EAS CLI if needed
npm install -g eas-cli

# Initialize EAS in your project
eas build:configure
```

This creates `eas.json`:

```json
{
  "cli": {
    "version": ">= 5.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": {
        "simulator": true
      }
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {}
  }
}
```

### 3. Build the Dev Client

```bash
# iOS Simulator build
eas build --platform ios --profile development

# iOS device build (requires Apple Developer account)
eas build --platform ios --profile development \
  --local  # Optional: build locally instead of EAS servers

# Android build
eas build --platform android --profile development
```

### 4. Install and Run

```bash
# Install on simulator/emulator (automatic after build)
eas build:run --platform ios

# Or start dev server pointing to the dev client
npx expo start --dev-client
```

## Distribution

### Internal Distribution (Testing)

For sharing dev builds with your team:

```json
// eas.json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    }
  }
}
```

Register test devices:

```bash
# Register iOS devices (creates provisioning profile)
eas device:create

# Build for registered devices
eas build --platform ios --profile development
```

Team members install via the link provided after build completes.

### TestFlight Distribution

For broader beta testing:

```json
// eas.json
{
  "build": {
    "preview": {
      "distribution": "internal",
      "ios": {
        "buildConfiguration": "Release"
      }
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "your@email.com",
        "ascAppId": "1234567890"
      }
    }
  }
}
```

```bash
# Build and submit to TestFlight
eas build --platform ios --profile preview
eas submit --platform ios
```

### Local Builds

Build on your machine instead of EAS cloud servers:

```bash
# Requires Xcode / Android Studio installed
eas build --platform ios --profile development --local

# Output: .app (simulator) or .ipa (device)
```

Advantages of local builds:
- No queue time
- Full control over build environment
- Access to local secrets/certificates
- No EAS build minutes consumed

## Adding Native Modules

### With Config Plugins

Most Expo-compatible libraries use config plugins that automatically configure native code:

```bash
npx expo install expo-camera
```

```json
// app.json
{
  "expo": {
    "plugins": [
      [
        "expo-camera",
        {
          "cameraPermission": "Allow $(PRODUCT_NAME) to access your camera"
        }
      ]
    ]
  }
}
```

Rebuild your dev client after adding plugins:

```bash
eas build --platform ios --profile development
```

### Custom Native Modules (Expo Modules API)

```bash
# Create a new native module
npx create-expo-module my-native-module --local
```

This scaffolds:

```
modules/
  my-native-module/
    android/
      src/main/java/.../MyNativeModule.kt
    ios/
      MyNativeModule.swift
    src/
      index.ts          # TypeScript API
    expo-module.config.json
```

## Development Workflow

```
1. npx expo start --dev-client
2. Open dev client app on device/simulator
3. Dev client connects to your local dev server
4. Fast Refresh works for JS/TS changes
5. Native code changes require rebuild
```

### Dev Menu

Shake device or press `Cmd+D` (iOS simulator) / `Cmd+M` (Android emulator):

- Reload JS bundle
- Toggle Fast Refresh
- Open debugger
- Performance monitor
- Element inspector

## Config Plugins

Config plugins modify native project configuration at build time without ejecting:

```ts
// plugins/withCustomEntitlement.ts
import { ConfigPlugin, withEntitlementsPlist } from 'expo/config-plugins';

const withCustomEntitlement: ConfigPlugin = (config) => {
  return withEntitlementsPlist(config, (mod) => {
    mod.modResults['com.apple.developer.healthkit'] = true;
    mod.modResults['com.apple.developer.healthkit.access'] = ['health-records'];
    return mod;
  });
};

export default withCustomEntitlement;
```

```json
// app.json
{
  "expo": {
    "plugins": ["./plugins/withCustomEntitlement"]
  }
}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Running `npx expo start` without `--dev-client` | Must use `--dev-client` flag to connect to custom builds |
| Forgetting to rebuild after adding native module | Any native dependency change requires `eas build` |
| Using Expo Go config with dev client | Remove `expo-go` specific workarounds |
| Not registering iOS test devices | Run `eas device:create` before building for physical devices |
| Mixing development and production profiles | Keep `eas.json` profiles separate and clear |
| Building for simulator when targeting device | Set `"simulator": false` in the iOS build profile for devices |

## Quick Reference

| Task | Command |
|------|---------|
| Install dev client | `npx expo install expo-dev-client` |
| Configure EAS | `eas build:configure` |
| Build for iOS simulator | `eas build --platform ios --profile development` |
| Build locally | `eas build --platform ios --profile development --local` |
| Start dev server | `npx expo start --dev-client` |
| Register device | `eas device:create` |
| Install build | `eas build:run --platform ios` |
| Submit to TestFlight | `eas submit --platform ios` |
| Create native module | `npx create-expo-module my-module --local` |
