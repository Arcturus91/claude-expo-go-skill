# Maps in Expo (React Native)

Add an interactive map with markers from your data — a feature that forces you to graduate from Expo Go to a development build.

## Read this first — maps need a development build, not Expo Go

Both mainstream map libraries — `react-native-maps` and Expo's own `expo-maps` — ship **native code** and need **native config** (a config plugin, and for Google Maps an API key) baked into the app binary. The prebuilt Expo Go app does not contain that native code or your API key, so a map will render blank or crash there.

Maps are therefore a "graduate from Expo Go" feature: you must make a **development build** (a custom dev client that includes these native modules). See [expo-go-workflow.md](expo-go-workflow.md) for how to create and run one. Everything below assumes SDK 54 and a dev build.

**Gotcha:** `expo-maps` requires the **New Architecture** (default on SDK 54). Verify against current docs if you're on an older/custom setup.

## Choosing a library

Both are valid on SDK 54. Verify the current recommendation against docs, but as of SDK 54:

| | `react-native-maps` | `expo-maps` |
|---|---|---|
| Maturity | Very mature, huge community, tons of examples | Newer, Expo first-party, still maturing |
| Providers | Google **or** Apple on both platforms (`PROVIDER_GOOGLE`) | Apple Maps on iOS, Google Maps on Android (native per-platform) |
| Markers | JSX children (`<Marker>`), custom React views/callouts | `markers` array prop; native markers only (no custom React-component markers) |
| API surface | Imperative + declarative, well-trodden | Cleaner, but you branch on `Platform.OS` |

**Recommendation for a junior dev: start with `react-native-maps`.** It's the most documented (most StackOverflow answers apply), gives one consistent `<MapView>`/`<Marker>` API across iOS and Android, and supports custom marker views. Reach for `expo-maps` if you specifically want Expo's first-party library or native Apple Maps on iOS. The rest of this file uses `react-native-maps` (version resolved by `expo install` for SDK 54; the Expo config plugin needs `react-native-maps` 1.22+).

## Setup

Install (this pins the SDK-54-compatible version — don't `npm install` a version yourself):

```bash
npx expo install react-native-maps
```

Add the config plugin in `app.json`. With no options it uses the default provider (Apple Maps on iOS). To use Google Maps you pass API keys:

```json
{
  "expo": {
    "plugins": [
      [
        "react-native-maps",
        {
          "androidGoogleMapsApiKey": "YOUR_ANDROID_KEY",
          "iosGoogleMapsApiKey": "YOUR_IOS_KEY"
        }
      ]
    ]
  }
}
```

**Gotcha: Android always needs a Google Maps API key** (there is no Apple Maps on Android). iOS uses **Apple Maps by default with no key**; you only need `iosGoogleMapsApiKey` if you opt into Google Maps on iOS via `PROVIDER_GOOGLE`. Get keys in Google Cloud Console (enable "Maps SDK for Android" / "Maps SDK for iOS").

**Gotcha: map API keys are not "secret" the way a server key is.** They're restricted by app signature — Android by package name + SHA-1 signing cert, iOS by bundle ID — and are embedded in the shipped binary. So a leaked key can't be used by another app. Still, restrict every key in Cloud Console and don't paste keys into public repos, issues, or chats. Prefer `"androidGoogleMapsApiKey": "$GOOGLE_MAPS_KEY_ANDROID"`-style env references and set them as EAS secrets.

Now create the development build that contains this native config:

```bash
eas build --profile development
```

**Any time you change plugins or keys in `app.json`, you must rebuild the dev build** — config changes are not picked up over-the-air.

## Basic map

A `<MapView>` fills the screen with `StyleSheet.absoluteFillObject` (a map with zero height shows nothing). Markers are `<Marker>` children — map over your data:

```jsx
import React from 'react';
import { StyleSheet } from 'react-native';
import MapView, { Marker } from 'react-native-maps';

const ITEMS = [
  { id: '1', title: 'Item A', coordinate: { latitude: 37.7825, longitude: -122.4324 } },
  { id: '2', title: 'Item B', coordinate: { latitude: 37.7749, longitude: -122.4194 } },
];

export default function ItemsMap() {
  return (
    <MapView
      style={StyleSheet.absoluteFillObject}
      initialRegion={{
        latitude: 37.78,
        longitude: -122.43,
        latitudeDelta: 0.0922, // vertical zoom span in degrees
        longitudeDelta: 0.0421,
      }}
    >
      {ITEMS.map(item => (
        <Marker
          key={item.id}
          coordinate={item.coordinate}
          title={item.title}
          description="Tap for details"
          onPress={() => console.log('tapped', item.id)}
        />
      ))}
    </MapView>
  );
}
```

**Gotcha:** `initialRegion` sets the starting view **once** (uncontrolled). Use the `region` prop instead if you need to move the camera programmatically from state.

## Showing the user's location

Turn on the built-in blue user-location dot with the `showsUserLocation` prop. This needs **location permission** — the config plugin for `react-native-maps` does not request it, so set up `expo-location` and request permission first (see [location.md](location.md)):

```jsx
import * as Location from 'expo-location';

const [region, setRegion] = React.useState(null);

React.useEffect(() => {
  (async () => {
    const { status } = await Location.requestForegroundPermissionsAsync();
    if (status !== 'granted') return; // handle gracefully
    const { coords } = await Location.getCurrentPositionAsync({});
    setRegion({
      latitude: coords.latitude,
      longitude: coords.longitude,
      latitudeDelta: 0.02,
      longitudeDelta: 0.02,
    });
  })();
}, []);

// <MapView showsUserLocation region={region} ...>
```

`showsUserLocation` draws the dot; centering the camera on the user is a separate step — set the region from the coords you got via `expo-location`, as above.

## Gotchas & testing

- **You cannot test maps in Expo Go.** Run the map on your **development build** on a real device or a properly-configured emulator. In Expo Go you'll see a blank area or a native error.
- **Android emulator needs Google Play services / a Google APIs system image.** A plain AOSP emulator renders a blank Google map. Use an image labeled "Google Play" or test on a physical device.
- **Blank map on a real device usually means a bad/missing API key** or the wrong API not enabled in Cloud Console (Android: "Maps SDK for Android"). Check device logs.
- **Rebuild after config changes.** New key or plugin option in `app.json` → new `eas build`; it won't hot-reload.
- **Performance with many markers:** hundreds of `<Marker>`s cause jank. Limit what you render to the visible region, set `tracksViewChanges={false}` on static custom markers, and cluster at low zoom (e.g. `react-native-map-clustering`). Fetch/paginate marker data rather than loading everything — see [architecture-principles.md](architecture-principles.md).

For the `expo-maps` alternative, the shape differs: import `{ AppleMaps, GoogleMaps } from 'expo-maps'`, render `AppleMaps.View` on iOS / `GoogleMaps.View` on Android, pass markers as a `markers` array prop, add the `expo-maps` config plugin (with `requestLocationPermission`) and the Google key under `android.config.googleMaps.apiKey`. Verify the current `expo-maps` API against docs before using.

Related: [location.md](location.md) · [expo-go-workflow.md](expo-go-workflow.md) · [architecture-principles.md](architecture-principles.md)
