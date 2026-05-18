---
name: react-native-expo
description: Build or audit a React Native + Expo bare workflow app — project structure, native modules, platform differences, entry point. Use when the user says "set up React Native", "add a native module", "fix an Expo issue", or "why does this work on iOS but not Android".
---

You are building or auditing a React Native + Expo bare workflow application.

## Bare workflow vs managed workflow

- **Managed** — Expo controls everything, no native code access, limited to Expo SDK
- **Bare** — you have full access to native iOS/Android projects, can add any native module
- Witly uses bare workflow — needed for `expo-av` (audio), `expo-secure-store`, and custom native modules

## Entry point

```js
// index.js — required for bare workflow, must be the "main" in package.json
import { registerRootComponent } from 'expo'
import App from './App'
registerRootComponent(App)
```

```json
// package.json
{ "main": "index.js" }
```

Without this, the app won't boot on physical devices.

## Project structure

```
frontend/
  src/
    screens/          ← one file per screen
    navigation/       ← navigators, types
    services/         ← all API calls (never in screens)
    context/          ← AuthContext, ThemeContext
    hooks/            ← useProfile, useAIResponse, etc.
    types/            ← TypeScript interfaces
    constants/        ← HUMOR_STYLES, PERSONA_TYPES, etc.
    components/
      common/         ← reusable UI components
  assets/
    images/           ← SVGs, PNGs
  App.tsx             ← root: providers + navigator
  index.js            ← entry point
```

## App.tsx pattern

```tsx
export default function App() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <AuthProvider>
        <ThemeProvider>
          <AppNavigator />
        </ThemeProvider>
      </AuthProvider>
    </GestureHandlerRootView>
  )
}
```

`GestureHandlerRootView` must wrap everything — required by react-native-gesture-handler.

## Platform differences

```tsx
import { Platform, StyleSheet } from 'react-native'

// Conditional styling
const styles = StyleSheet.create({
  shadow: Platform.select({
    ios: { shadowColor: '#000', shadowOffset: { width: 0, height: 2 }, shadowOpacity: 0.1, shadowRadius: 4 },
    android: { elevation: 4 },
  }),
})

// Conditional logic
if (Platform.OS === 'ios') {
  // iOS-only
}
```

Key differences:
- **Shadows**: iOS uses `shadow*` props, Android uses `elevation`
- **Fonts**: custom fonts need different loading on each platform
- **Text**: Android adds extra padding — use `includeFontPadding: false` to remove
- **Status bar**: use `expo-status-bar` to handle both platforms consistently
- **Keyboard**: `KeyboardAvoidingView` behavior differs — `padding` on iOS, `height` on Android

## Audio recording (expo-av)

```tsx
import { Audio } from 'expo-av'

// Request permission before first use
const { granted } = await Audio.requestPermissionsAsync()

// Configure audio session
await Audio.setAudioModeAsync({
  allowsRecordingIOS: true,
  playsInSilentModeIOS: true,
})

// Start recording
const { recording } = await Audio.Recording.createAsync(
  Audio.RecordingOptionsPresets.HIGH_QUALITY
)

// Stop and get URI
await recording.stopAndUnloadAsync()
const uri = recording.getURI()

// Upload as FormData
const formData = new FormData()
formData.append('audio', { uri, type: 'audio/m4a', name: 'recording.m4a' } as any)
```

## Common Expo bare issues

- **"No bundle URL present"** — Metro bundler not running. Start with `npx expo start`.
- **Native module not found** — added a package but didn't run `npx expo prebuild` or `pod install` (iOS). Native modules need a rebuild.
- **Works on simulator, not device** — check `ALLOWED_HOSTS` in Django includes the device's local IP. Also check the API URL points to local IP, not `localhost`.
- **Hermes JS engine** — Expo uses Hermes on React Native 0.70+. `ReadableStream` is not supported. Use `axios` with `responseType: 'json'` instead of fetch streaming.
- **`registerRootComponent` missing** — app boots in Expo Go but not as standalone. Always include `index.js`.

## Running the app

```bash
npx expo start              # start Metro bundler
npx expo run:android        # build and run on Android
npx expo run:ios            # build and run on iOS (Mac only)
npx expo prebuild           # generate native projects after adding native modules
cd ios && pod install        # install iOS native deps after prebuild
```
