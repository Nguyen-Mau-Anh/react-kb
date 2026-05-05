# Beyond the Web — 01. React Native with Expo SDK 51: expo-router 3, Tamagui 1 vs Nativewind 4

> **What / Why / How** — pick **Expo SDK 51 + expo-router 3**. Skip vanilla React Native CLI in 2026. Style with Nativewind 4 if you came from Tailwind; Tamagui 1 if you want better web parity.

---

## 1. Why React Native (And Why Expo)

### What

**React Native** lets you write iOS and Android apps using React. Components map to native UI: `<View>` becomes `UIView` on iOS and `android.view.ViewGroup` on Android. Animations and gestures run on the native side.

**Expo** is a managed framework on top of React Native. It bundles:
- A standardized project structure (`expo@51`, `expo-router@3`).
- Pre-built native modules (`expo-camera`, `expo-image`, `expo-notifications`, ~200 first-party).
- Build infrastructure (`eas-cli@13`) — no Xcode/Android Studio required for production builds.
- Over-the-air updates (`expo-updates@0.25`).
- Development client and `expo go` for instant testing.

### Why Expo, not vanilla React Native CLI

In 2024, Meta's React Native team officially recommended Expo as **the** way to build React Native apps. The community-maintained `react-native init` template was deprecated in favor of `npx create-expo-app`.

Real reasons:

| | Vanilla `react-native@0.74` (CLI) | Expo SDK 51 |
|--|------------------------------------|-------------|
| Native module install | Manual `pod install` per dep, often patches needed | `npx expo install <pkg>` handles native code via Config Plugins |
| iOS/Android upgrades | Hand-edit Gradle, Podfile, Info.plist — every release | `expo prebuild` regenerates from config |
| Cloud builds | Configure CI from scratch | `eas build` produces `.ipa` and `.aab` in the cloud |
| OTA updates | DIY (CodePush, Microsoft, deprecated) | `expo-updates@0.25` built-in |
| Routing | DIY with `react-navigation@6` | `expo-router@3` with file-system routes |
| First-party access to camera, sensors, files, biometrics, notifications | Each is a separate community package | Single `expo-*` family, version-locked to SDK |

The vanilla CLI still exists for teams with deep custom-native requirements (e.g., custom NDK, kernel-level integrations). For 99% of apps in 2026: **Expo**.

### When NOT to use React Native at all

- A pure web app — use Next.js (covered in Group 7).
- A simple wrapper around a website — use a PWA (`next-pwa@5` works fine).
- An iOS-only or Android-only app where platform-specific UI is the differentiator (Apple Music, Google Pay) — go fully native.
- Heavy 3D / game / AR — Unity, Unreal, or `react-three/native` (still rough).

---

## 2. The Real Stack in 2026

| Layer | Choice | Notes |
|-------|--------|-------|
| Framework | `expo@51` | SDK 51 is current stable through 2026; React 18.2, RN 0.74 |
| Router | `expo-router@3` | File-system routing, same mental model as Next.js App Router |
| Language | TypeScript 5.4 | Same as web; one shared `tsconfig` if monorepo |
| State | TanStack Query 5.28 + Zustand 4.5 | Identical to web stack (covered in `04.state-and-data/02-state-management.md`) |
| Forms | `react-hook-form@7.51` + `zod@3.23` | Identical to web |
| Styling | Nativewind 4.1 (Tailwind for RN) **or** Tamagui 1.108 | Choice deep-dived below |
| Navigation primitive | `react-native-reanimated@3.10` + `react-native-gesture-handler@2.16` | Bundled with Expo SDK 51 |
| Native bottom sheet | `@gorhom/bottom-sheet@5` | The de-facto modal/drawer lib |
| Lists | `@shopify/flash-list@1.6` | Replaces FlatList for perf |
| Animation | `react-native-reanimated@3.10` | Native-driven; runs at 60 FPS on the UI thread |
| Build/distribute | EAS Build + EAS Submit (`eas-cli@13`) | Fully cloud-managed |
| Auth | `expo-auth-session@5` (OAuth), `expo-secure-store@13` (Keychain/Keystore) | First-party |
| Storage | `expo-sqlite@13` for local SQL, `@op-sqlite/op-sqlite@5` for high-perf | SQLite is the default RN persistence story |
| Database (cloud) | Same as web — Neon Postgres, Supabase, Convex | Shared backend with the web app |
| Push notifications | `expo-notifications@0.28` | Wraps APNs + FCM |

The "shared backend, two clients" model with Next.js for web and Expo for mobile, both calling the same tRPC API or REST, is the most-replicated architecture in production React shops in 2026 (T3-Turbo monorepo, Cal.com mobile, etc.).

---

## 3. Setup — `create-expo-app` Path

```bash
npx create-expo-app@latest my-app --template
```

Pick **`Default (TypeScript)`** template. The CLI scaffolds:

```
my-app/
├── app/                          ← expo-router file-system routes
│   ├── _layout.tsx
│   ├── (tabs)/
│   │   ├── _layout.tsx
│   │   ├── index.tsx
│   │   └── explore.tsx
│   ├── +not-found.tsx
│   └── modal.tsx
├── assets/                       ← images, fonts
├── components/                   ← reusable UI
├── constants/
├── hooks/
├── package.json
├── app.json                      ← Expo config (name, icon, plugins)
├── tsconfig.json
└── babel.config.js
```

Run it:

```bash
npm run ios       # opens iOS simulator (macOS only)
npm run android   # opens Android emulator
npm run web       # opens in a browser via React Native Web
```

### Development client vs Expo Go

- **Expo Go** (`expo start`): a sandboxed app you install on a real device. Open the QR code → it loads your project. Limited to JS-only; no custom native modules.
- **Development client** (`npx expo run:ios` or `:android`): builds a custom dev client app that includes any native modules your project needs. Required once you add a Config Plugin or non-Expo-Go-compatible package.

For real development past day one: dev client.

---

## 4. expo-router 3 — File-System Routing for Mobile

### What

Same model as Next.js App Router (covered in `07.nextjs/01-fundamentals.md`):

| File | Meaning |
|------|---------|
| `app/index.tsx` | The home route (`/`) |
| `app/about.tsx` | Route `/about` |
| `app/_layout.tsx` | Wraps children — equivalent to Next.js `layout.tsx` |
| `app/(tabs)/` | Tab group — multiple pages share a tab bar via the layout |
| `app/[id].tsx` | Dynamic route, accessed via `useLocalSearchParams()` |
| `app/+not-found.tsx` | 404 page |
| `app/modal.tsx` + `presentation: 'modal'` in layout | Modal-style screen |

### Real example — tab + stack layouts

```tsx
// app/_layout.tsx — root stack
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      <Stack.Screen name="modal"  options={{ presentation: 'modal' }} />
      <Stack.Screen name="+not-found" />
    </Stack>
  );
}
```

```tsx
// app/(tabs)/_layout.tsx — bottom tab bar
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function TabsLayout() {
  return (
    <Tabs screenOptions={{ tabBarActiveTintColor: '#2563eb' }}>
      <Tabs.Screen name="index"   options={{ title: 'Home',    tabBarIcon: ({ color }) => <Ionicons name="home"   color={color} size={20} /> }} />
      <Tabs.Screen name="explore" options={{ title: 'Explore', tabBarIcon: ({ color }) => <Ionicons name="search" color={color} size={20} /> }} />
    </Tabs>
  );
}
```

### Navigating

```tsx
import { Link, useRouter } from 'expo-router';

// Declarative
<Link href="/about" asChild><Pressable><Text>About</Text></Pressable></Link>

// Imperative
const router = useRouter();
router.push('/profile/42');
router.replace('/login');
router.back();
```

`router.push('/profile/42')` autocompletes route paths if you enable typed routes:

```js
// app.json — enable typed routes
{ "expo": { "experiments": { "typedRoutes": true } } }
```

Same idea as Next.js's `experimental.typedRoutes` (covered in `07.nextjs/01-fundamentals.md`).

### Route groups for marketing-vs-app split

```
app/
├── (auth)/
│   ├── _layout.tsx     ← stack with no tab bar
│   └── login.tsx
└── (tabs)/
    ├── _layout.tsx     ← bottom tab bar
    ├── index.tsx
    └── settings.tsx
```

Same `()` group convention. Logged-out users see `(auth)`; logged-in users see `(tabs)`.

---

## 5. Styling — Nativewind 4 vs Tamagui 1

The two real choices in 2026.

### Nativewind 4 (Tailwind for React Native)

`nativewind@4.1` brings Tailwind utility classes to React Native. Same `className=""` you write on web, transformed at build time into RN `style={}` objects.

```tsx
import { Text, View } from 'react-native';

<View className="flex-1 items-center justify-center bg-slate-50 p-6">
  <Text className="text-2xl font-semibold text-slate-900">Hello</Text>
  <Text className="mt-2 text-base text-slate-600">Welcome back.</Text>
</View>
```

Setup:

```bash
npm i nativewind@4 tailwindcss@3.4
npx tailwindcss init
```

```js
// metro.config.js
const { getDefaultConfig } = require('expo/metro-config');
const { withNativeWind } = require('nativewind/metro');
const config = getDefaultConfig(__dirname);
module.exports = withNativeWind(config, { input: './global.css' });
```

```css
/* global.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

```tsx
// app/_layout.tsx — import the CSS once at app root
import '../global.css';
```

### Why pick Nativewind

- **Same mental model as web Tailwind.** Engineers cross-train instantly.
- **Class string sharing**: a `Button` component can use the **exact same `className`** on web and mobile if you adopt Tailwind everywhere.
- **Smaller learning surface**: Tailwind utility names you already know.
- **Active maintenance**: backed by `Marklawlor`, used in the Expo official examples.

### Why Nativewind sometimes loses

- v3 → v4 was a near-rewrite; some libraries had to adapt.
- Slightly larger Metro overhead at build time than plain `StyleSheet.create`.

### Tamagui 1.108 — The Performance-First Alternative

`tamagui@1.108` is a UI kit + style system designed for React Native first, with web parity through React Native Web.

```tsx
import { YStack, Text, Button } from 'tamagui';

<YStack ai="center" jc="center" p="$4" space="$3" bg="$background">
  <Text fontSize="$8" fontWeight="700">Hello</Text>
  <Button theme="active">Get started</Button>
</YStack>
```

Tamagui ships:
- A token system (`$4`, `$background`, `$primary`).
- Optimizing compiler that **flattens runtime style logic to static JSX** — measurably faster than CSS-in-JS at scale.
- Prebuilt components (`YStack`, `XStack`, `Card`, `Sheet`, `Dialog`).
- Animations via `@tamagui/animations-moti` or `@tamagui/animations-reanimated`.

### Why Tamagui

- **Best-in-class web↔native parity.** The same `<Button>` works in Expo and on `next.js` via React Native Web with no per-platform code.
- **Compiler optimizations** make it the fastest CSS-in-JS-equivalent for RN.
- **Built-in components** save you from designing a system from scratch.

### Why Tamagui can be a hard pick

- Different mental model from Tailwind (token props instead of utility classes).
- Smaller community than Nativewind in mid-2026.
- Heavier setup than Nativewind.

### Decision rule

| Situation | Pick |
|-----------|------|
| Already on Tailwind for web; want minimum cognitive overhead | **Nativewind 4** |
| Targeting web AND mobile from one codebase, want shared components | **Tamagui 1** |
| Performance-critical mobile-first app | **Tamagui 1** |
| Want a polished pre-built component library | **Tamagui 1** |
| Just learning, want maximum tutorial coverage | **Nativewind 4** |

For the typical "Next.js web + Expo mobile in a Turborepo" stack: **Nativewind** is the path of least resistance. Choose Tamagui only if web-mobile-shared-components is a primary goal.

---

## 6. Data Fetching — Same Tools as Web

The whole TanStack Query + tRPC stack from `07.nextjs/06-api-routes.md` and `04.state-and-data/03-data-fetching.md` works identically in Expo. **Zero changes needed.**

```tsx
import { trpc } from '@/trpc/client';

function Profile() {
  const { data, isPending } = trpc.user.me.useQuery();
  if (isPending) return <ActivityIndicator />;
  return <Text>{data?.name}</Text>;
}
```

In a monorepo (`turbo@2` + `pnpm@9`, covered in `08.ecosystem/03-monorepo.md`):

```
apps/
├── web/        ← Next.js 14, uses tRPC client
└── mobile/     ← Expo SDK 51, uses tRPC client
packages/
├── api/        ← tRPC server (mounted in apps/web/app/api/trpc)
├── shared/     ← Zod schemas, types
└── ui/         ← Shared components (with Tamagui or per-platform variants)
```

This is exactly the `t3-turbo` template's architecture.

---

## 7. Auth — Real Patterns

### OAuth with `expo-auth-session@5`

```tsx
import * as AuthSession from 'expo-auth-session';
import * as WebBrowser from 'expo-web-browser';

WebBrowser.maybeCompleteAuthSession();

const discovery = AuthSession.useAutoDiscovery('https://yourdomain.com/api/auth');

function LoginScreen() {
  const [request, response, promptAsync] = AuthSession.useAuthRequest({
    clientId: 'your-client-id',
    scopes: ['openid', 'profile', 'email'],
    redirectUri: AuthSession.makeRedirectUri({ scheme: 'myapp' }),
  }, discovery);

  return <Button disabled={!request} title="Sign in" onPress={() => promptAsync()} />;
}
```

For a Next.js + Expo monorepo, the cleanest pattern: **the mobile app calls the same `next-auth@5` (Auth.js) endpoints the web uses**. Auth.js v5 supports a token-based flow exposed at `/api/auth/session` that mobile clients can read.

### Storing tokens — `expo-secure-store@13`

```tsx
import * as SecureStore from 'expo-secure-store';

await SecureStore.setItemAsync('session', JSON.stringify({ token, expiresAt }));
const stored = await SecureStore.getItemAsync('session');
```

iOS Keychain + Android Keystore under the hood. Don't use `AsyncStorage` for tokens — that's plain unencrypted file storage.

### Biometric prompt — `expo-local-authentication@13`

```tsx
import * as LocalAuthentication from 'expo-local-authentication';

const result = await LocalAuthentication.authenticateAsync({
  promptMessage: 'Unlock the app',
  fallbackLabel: 'Use passcode',
});
if (result.success) showAppHome();
```

Wrap your app's resume flow in a Face ID / Touch ID gate.

---

## 8. Lists — Use FlashList, Not FlatList

```bash
npx expo install @shopify/flash-list
```

```tsx
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={tasks}
  renderItem={({ item }) => <TaskRow task={item} />}
  estimatedItemSize={64}     // required — FlashList uses this for cell recycling
  keyExtractor={(t) => t.id}
/>
```

FlashList recycles cells like the native iOS `UITableView` and Android `RecyclerView`. **2–10× faster** than `FlatList` for any list >50 items. The Shopify team rebuilt their app on top of it; it's now the React Native default.

---

## 9. Forms — `react-hook-form` Works as-Is

The same `react-hook-form@7.51` + `zod@3.23` (covered in `04.state-and-data/01-forms.md`) works in React Native. Replace HTML `<input>` with `<TextInput>`:

```tsx
import { Controller, useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { TextInput, Text, Button } from 'react-native';
import { z } from 'zod';

const Schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

function LoginForm() {
  const { control, handleSubmit, formState: { errors } } = useForm({ resolver: zodResolver(Schema) });
  return (
    <>
      <Controller
        control={control}
        name="email"
        render={({ field: { value, onChange } }) => (
          <TextInput value={value} onChangeText={onChange} placeholder="Email" />
        )}
      />
      {errors.email && <Text>{errors.email.message}</Text>}
      <Button title="Sign in" onPress={handleSubmit(submit)} />
    </>
  );
}
```

`<Controller>` is necessary because RN inputs don't expose a stable `ref` API like the web. The pattern is documented at [react-hook-form.com — React Native](https://react-hook-form.com/get-started#ReactNative).

---

## 10. Animation — `react-native-reanimated@3.10`

The native-driven animation library. Animations run on the UI thread, not the JS thread, so they hit 60 FPS even when JS is busy.

```tsx
import Animated, { useSharedValue, useAnimatedStyle, withTiming } from 'react-native-reanimated';
import { Pressable } from 'react-native';

function PressableCard() {
  const scale = useSharedValue(1);
  const style = useAnimatedStyle(() => ({ transform: [{ scale: scale.value }] }));

  return (
    <Pressable
      onPressIn={() => { scale.value = withTiming(0.96, { duration: 80 }); }}
      onPressOut={() => { scale.value = withTiming(1,    { duration: 120 }); }}
    >
      <Animated.View style={[styles.card, style]}>
        <Text>Tap me</Text>
      </Animated.View>
    </Pressable>
  );
}
```

Pair with `react-native-gesture-handler@2.16` for drag, swipe, pinch — all also UI-thread-driven.

For higher-level orchestration (variants, layout animations) the equivalent of Framer Motion on RN is `moti@0.29` (built on Reanimated):

```tsx
import { MotiView } from 'moti';

<MotiView
  from={{ opacity: 0, translateY: 12 }}
  animate={{ opacity: 1, translateY: 0 }}
  transition={{ type: 'timing', duration: 250 }}
/>
```

Same mental model as Framer Motion; covered for web in `08.ecosystem/01-animation.md`.

---

## 11. Building & Shipping — EAS Build + EAS Submit

```bash
npm i -g eas-cli@13
eas login
eas build:configure
eas build --platform ios --profile production
eas build --platform android --profile production
eas submit --platform ios     # uploads to App Store Connect
eas submit --platform android # uploads to Google Play
```

**No Xcode or Android Studio needed locally.** EAS Build runs the native compile in the cloud and emails you the `.ipa` / `.aab`.

`eas.json` profiles:

```jsonc
{
  "cli": { "version": ">= 13.0.0" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal",
      "ios":     { "simulator": true }
    },
    "production": {
      "autoIncrement": true,
      "env": { "API_URL": "https://api.example.com" }
    }
  },
  "submit": {
    "production": {}
  }
}
```

### Pricing reality check

EAS Build free tier: 30 builds/month for hobby projects. Paid: $19/mo (Production plan) covers 30 priority builds/mo + concurrencies. Real teams typically pay $99/mo (Enterprise) for parallel builds.

### Over-the-air updates — `expo-updates@0.25`

```bash
eas update --branch production --message "Fix login bug"
```

Pushes a new JS bundle to all installed apps within minutes — no App Store review. Native code changes still need a real build, but JS-only fixes deploy as fast as a Vercel push.

---

## 12. Sharing Code Between Web (Next.js) and Mobile (Expo)

### The realistic split

| Code | Shared? | Notes |
|------|---------|-------|
| Zod schemas, TS types | ✅ Yes | `packages/shared` |
| tRPC server | ✅ Yes | `packages/api`; mounted in Next.js, called from both clients |
| tRPC client + TanStack Query setup | ✅ Mostly | Same code; provider wrappers differ |
| Custom business hooks (`useTaskList`, `useFormatPrice`) | ✅ Yes | If they don't touch DOM |
| UI components | ⚠ Partial | Either Tamagui (cross-platform) or platform-specific (`Button.tsx` + `Button.native.tsx`) |
| Routing | ❌ No | Next.js App Router and expo-router are different APIs (similar enough to make migration easy) |
| Auth | ⚠ Server-side same; client flow different | Web uses `next-auth/react`; mobile uses `expo-auth-session` calling the same Auth.js endpoints |

### File extension trick

Metro and Webpack both respect platform-specific extensions:

```
components/Button.tsx          ← shared / web
components/Button.native.tsx   ← React Native override
components/Button.ios.tsx      ← iOS-only override
components/Button.android.tsx  ← Android-only override
```

Import `'./Button'` and the right file is picked at build time.

---

## 13. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `Unable to resolve module` after installing a native package | You're on Expo Go but the package needs native code. Run `npx expo prebuild` then `npx expo run:ios`/`run:android` for a dev client. |
| App crashes only in production | Test with `eas build --profile preview` (matches prod settings) before submitting |
| AsyncStorage data lost on iOS update | iOS sometimes purges. Don't store anything you can't refetch; use `expo-secure-store` for sensitive data |
| OTA update breaks for users on older app versions | Use `runtimeVersion` policy in `app.json`; updates only target compatible versions |
| `expo-router` typed routes show errors | Run `npx expo start` once after enabling — it generates the route types |
| Slow list scrolling | Switch to `FlashList`, set `estimatedItemSize`, memoize row components |
| Nativewind classes not applying | `metro.config.js` not wrapping with `withNativeWind`, or `global.css` not imported in `_layout.tsx` |
| Tamagui flicker on first paint | Add the optimizing Babel plugin: `'@tamagui/babel-plugin'` |
| Build succeeds locally but fails on EAS | Run `eas build --local` to reproduce; check log for Node version, Cocoapods, Gradle JDK mismatches |
| `expo-image-picker` crashes on iOS 17+ | Add `NSPhotoLibraryUsageDescription` in `app.json` `ios.infoPlist` |

---

## 14. Decision Tree

```
Building a mobile app from scratch?
│   YES → Expo SDK 51 + expo-router 3 (don't use vanilla React Native CLI)
│   NO  ↓
Already on a vanilla react-native@0.74 codebase?
│   YES → Migrate gradually with `npx install-expo-modules` — adopt Expo SDK piece by piece
│   NO  ↓
Need iOS-specific UI quality (Apple Music, Health, Pay)?
│   YES → Native Swift + UIKit/SwiftUI
│   NO  ↓
Want one codebase for web + mobile + maybe desktop?
│   YES → Expo + Tamagui (shared components) OR Expo + Nativewind + per-platform components

Picking a styling library?
├── Already on Tailwind for web? → Nativewind 4
└── Want shared components across web and mobile? → Tamagui 1
```

---

## 15. What This Topic Connects To

- **Next.js fundamentals** (`07.nextjs/01-fundamentals.md`) — expo-router uses the same file-system mental model.
- **Routing** (`07.nextjs/02-routing.md`) — layouts, route groups, dynamic segments work the same in expo-router.
- **State management** (`04.state-and-data/02-state-management.md`) — Zustand 4 and TanStack Query work identically.
- **Forms** (`04.state-and-data/01-forms.md`) — `react-hook-form` + Zod work identically.
- **Auth** (`07.nextjs/04-auth.md`) — same Auth.js endpoints; different client flow.
- **Monorepo** (`08.ecosystem/03-monorepo.md`) — Turborepo + pnpm is the standard layout for web + mobile.
- **Real project** (`08.ecosystem/04-real-project.md`) — extending the Task Manager to mobile is the natural next step.

---

## Summary

| Tool | Pick when |
|------|-----------|
| Expo SDK 51 + `expo-router@3` | Default for any new React Native app in 2026 |
| `nativewind@4.1` | Came from Tailwind on web, want minimum overhead |
| `tamagui@1.108` | Want web/mobile shared components, performance-critical |
| `@shopify/flash-list@1.6` | Lists with >50 items |
| `react-native-reanimated@3.10` + `react-native-gesture-handler@2.16` | Native-thread animations and gestures |
| `moti@0.29` | Framer-Motion-style declarative animations |
| `expo-secure-store@13` | Tokens, sensitive credentials |
| `eas-cli@13` | Cloud builds and submissions |

| Rule | Why |
|------|-----|
| Default to Expo over vanilla React Native CLI | Meta and the React Native team officially recommend it |
| Use `expo-router` over `react-navigation` directly | File-system routing matches Next.js mental model |
| Use FlashList, not FlatList, for non-trivial lists | 2–10× perf win |
| Store tokens in `expo-secure-store`, not AsyncStorage | Encrypted, OS-level keychain |
| Reuse the same tRPC + TanStack Query stack as web | Zero rewrites; types flow through |

---

## Further reading

- [Expo docs](https://docs.expo.dev/)
- [expo-router](https://docs.expo.dev/router/introduction/)
- [Nativewind 4 docs](https://www.nativewind.dev/)
- [Tamagui docs](https://tamagui.dev/docs/intro/installation)
- [FlashList docs](https://shopify.github.io/flash-list/)
- [react-native-reanimated docs](https://docs.swmansion.com/react-native-reanimated/)
- [EAS Build docs](https://docs.expo.dev/build/introduction/)
- [t3-turbo](https://github.com/t3-oss/create-t3-turbo) — reference monorepo for Next.js + Expo + tRPC
