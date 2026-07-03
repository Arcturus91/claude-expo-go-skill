# Reading device location with expo-location

Read a device's GPS coordinates (one-shot or as a live stream) and handle the location permission flow, on Expo SDK 54. All APIs below verified against current Expo docs (SDK 54).

## What works where

- **Foreground location works in Expo Go.** Asking for permission and reading the position while your app is open on screen — `getCurrentPositionAsync`, `watchPositionAsync` — runs fine in Expo Go on SDK 54. This covers almost every MVP.
- **Background location does NOT work in Expo Go.** Tracking the device while your app is backgrounded or closed (`startLocationUpdatesAsync` + a background task) requires a **development build**. The Expo docs state plainly: "You must use a development build to use background location since it is not supported in the Expo Go app." Android foreground/background *services* are also unavailable in Expo Go.
- If you need background tracking, see [expo-go-workflow.md](expo-go-workflow.md) for when to graduate from Expo Go to a dev build, and the [Background location](#background-location-post-mvp) note below.

## Install & config

```bash
npx expo install expo-location
```

Use `expo install` (not `npm install`) so you get the version that matches SDK 54.

Add the config plugin to `app.json`. The plugin writes the required native permission strings for you:

```json
{
  "expo": {
    "plugins": [
      [
        "expo-location",
        {
          "locationWhenInUsePermission": "Allow $(PRODUCT_NAME) to use your location."
        }
      ]
    ]
  }
}
```

- **iOS** — `locationWhenInUsePermission` sets `NSLocationWhenInUseUsageDescription` (the "while using the app" case). **Gotcha:** iOS *requires* a usage-description string. Without it the App Store rejects the build and the app can crash the moment you request permission. Always set it via the plugin. For background/"always" access, also set `locationAlwaysAndWhenInUsePermission` (writes `NSLocationAlwaysAndWhenInUseUsageDescription`) and `isIosBackgroundLocationEnabled: true`.
- **Android** — the plugin adds `ACCESS_FINE_LOCATION` and `ACCESS_COARSE_LOCATION` automatically. `ACCESS_BACKGROUND_LOCATION` is added only when you set `isAndroidBackgroundLocationEnabled: true` — leave it off for a foreground-only app.
- **Gotcha:** config-plugin changes are applied at build time. In Expo Go the base permission strings are already present, but if you edit these options you must reload; for a dev build you must rebuild.

## Permission flow

Request permission before reading. The response is `{ status, granted, canAskAgain, expires }`; `status` is `'granted'`, `'denied'`, or `'undetermined'`.

```typescript
import * as Location from 'expo-location';

const { status } = await Location.requestForegroundPermissionsAsync();
if (status !== 'granted') {
  // show an empty/error state — do NOT silently proceed
  return;
}
```

- Use `getForegroundPermissionsAsync()` to *check* status without prompting (e.g. on screen mount), and `requestForegroundPermissionsAsync()` to actually prompt.
- For background, call `requestBackgroundPermissionsAsync()` — but only after foreground is granted, and only in a dev build.
- Handle all three states honestly with loading / empty / error UI — never assume granted. See [architecture-principles.md](architecture-principles.md) §7.
- **Gotcha (iOS):** the user may pick "Allow While Using" vs "Allow Once" / "Always" — you get foreground rights but not necessarily background.
- **Gotcha:** once a user taps Deny, the OS will not show the prompt again. `canAskAgain` comes back `false`. Your only recourse is to explain and send them to the system Settings app (e.g. via `Linking.openSettings()`); you cannot re-prompt in-app.

## Reading location

**One-shot** — best for "locate me now" buttons:

```typescript
const location = await Location.getCurrentPositionAsync({
  accuracy: Location.Accuracy.Balanced,
});
const { latitude, longitude } = location.coords;
```

**Stream** — best for live-updating UIs. Returns a subscription you must clean up:

```typescript
useEffect(() => {
  let sub: Location.LocationSubscription;
  (async () => {
    sub = await Location.watchPositionAsync(
      { accuracy: Location.Accuracy.Balanced, timeInterval: 5000, distanceInterval: 10 },
      (loc) => setPosition(loc.coords)
    );
  })();
  return () => sub?.remove(); // Gotcha: ALWAYS remove on unmount or GPS keeps running
}, []);
```

- **Accuracy** (`Location.Accuracy`): `Lowest` (~3 km), `Low` (~1 km), `Balanced` (~100 m), `High` (~10 m), `Highest`, `BestForNavigation`. Higher accuracy = more battery and slower first fix. `Balanced` is a sensible default.
- **`coords`** contains `latitude`, `longitude`, `accuracy` (radius in meters), plus `altitude`, `altitudeAccuracy`, `heading`, and `speed` (any of which can be `null` depending on device/sensors). `location.timestamp` (top level) is when the fix was taken.
- **Gotcha:** the first fix can take a few seconds or return `null` fields — show a loading state and don't block your UI on it.

## Storing it

If you persist a user's coordinates to Firestore, treat them as **sensitive PII**:

- Lock down access in security rules so a user can only read/write their own location document — never make it world-readable. See [firestore-data-rules.md](firestore-data-rules.md).
- Do not `console.log` raw coordinates in production; logs leak.
- Store only what you need (e.g. rounded coordinates or a place name) and consider a retention limit.

## Background location (post-MVP)

Tracking location while the app is backgrounded needs a **development build** (not Expo Go), a background task registered with `expo-task-manager` and started via `Location.startLocationUpdatesAsync`, plus the extra config above (`isIosBackgroundLocationEnabled`, `isAndroidBackgroundLocationEnabled`, and the "always" usage strings). Expect heightened app-store scrutiny — both Apple and Google require a written justification for why your app tracks users in the background. Defer this until you actually need it.

---

See also: [maps.md](maps.md) for plotting a coordinate on a map, and [architecture-principles.md](architecture-principles.md) for permission-driven UI states.
