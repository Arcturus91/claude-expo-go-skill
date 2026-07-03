# Expo Go workflow: run on a device and share a preview

How to run your app on a real phone during development and hand a teammate a scannable preview through EAS Update — no native build, no TestFlight/Play Store. Targets **Expo SDK 54**, **expo-updates ~29**, and the **EAS CLI**. See also [project-setup.md](project-setup.md) and [troubleshooting.md](troubleshooting.md).

## Run it on your phone (dev loop)

1. Install **Expo Go** from the App Store (iOS) or Play Store (Android) on your phone.
2. From the project root, start the dev server:

   ```bash
   npx expo start
   ```

   A QR code prints in the terminal. On **iOS**, scan it with the built-in Camera app; on **Android**, open Expo Go and use **Scan QR code**. The JS bundle downloads and your app opens inside Expo Go.
3. **Same Wi-Fi = LAN.** Phone and computer must be on the same network; the QR encodes a `exp://<your-lan-ip>:8081` URL. Edits are pushed live — **Fast Refresh** re-renders on save.

   ```bash
   npx expo start --tunnel   # phone on cellular, or a firewall/VPN/public
                             # Wi-Fi blocks LAN — routes through an HTTPS tunnel
   ```

   **Gotcha:** `--tunnel` is slower and occasionally flaky, but it's the reliable fallback when the QR "opens" but the bundle never loads. It needs the `@expo/ngrok` helper, which the CLI installs on first use.
4. Emulators/simulators, if installed: press `a` (Android) or `i` (iOS) in the running terminal, or launch straight into one:

   ```bash
   npx expo start --android   # Android emulator
   npx expo start --ios       # iOS simulator (macOS only)
   ```

## Expo Go vs a dev client (why it matters)

Expo Go is a **prebuilt app with a fixed native runtime**. Your JS runs inside it, but it can only use the native modules Expo Go was compiled with (the Expo SDK set).

- **Pure-JS libraries are fine** — e.g. the `firebase` JS SDK, date/state/UI libraries. They're just JavaScript, so they ride along in the bundle.
- **Custom native modules are NOT** — anything shipping its own native code or a config plugin that edits the native project (e.g. `@react-native-firebase`, most Bluetooth/payment/MLKit SDKs) is not compiled into Expo Go and will crash or no-op.

The moment you add one native module you must switch to a **development build** (a "dev client" you build with `eas build`) — and you lose the "just send a URL, open in Expo Go" superpower, because now each tester needs *your* custom binary installed. So while you want frictionless sharing, **keep the dependency set Expo-Go-safe** (prefer JS-only packages and Expo SDK modules).

## Share a build with a teammate (EAS Update)

Goal: a teammate opens **your** app inside **their** Expo Go by scanning a QR — no native build, no store. EAS Update hosts your JS bundle in the cloud; Expo Go fetches it. One-time setup, then publish on every change.

**1. Install and log in to the EAS CLI** (once per machine):

```bash
npm install -g eas-cli
eas login
```

**2. Link the project to EAS** (creates/attaches a `projectId`; alias of `eas project:init`):

```bash
eas init
```

**3. Configure EAS Update.** This installs/links `expo-updates` and writes the `updates.url` + `runtimeVersion` into `app.json`:

```bash
eas update:configure
```

Confirm `app.json` ends up like this — the **runtimeVersion policy must tie to the SDK** so Expo Go can load it:

```json
{
  "expo": {
    "runtimeVersion": { "policy": "sdkVersion" },
    "updates": { "url": "https://u.expo.dev/<your-project-id>" },
    "extra": { "eas": { "projectId": "<your-project-id>" } }
  }
}
```

With `policy: "sdkVersion"`, the runtime *is* your Expo SDK version (e.g. SDK 54), which is exactly what a matching Expo Go knows how to run.

**4. Publish an update** to a named branch:

```bash
eas update --branch preview --message "Share build for review"
```

`--branch <name>` is required (pick any name, e.g. `preview`); `-m/--message` labels it. `-p android|ios|all` scopes platforms (default `all`). On **SDK 55+** an `--environment <name>` flag becomes required — on **SDK 54 it is not**.

**5. The teammate opens it.** Open the published update on your project's page at **expo.dev** (Projects → your app → the branch/update); it shows a **QR code**. They scan it — iOS Camera, or Expo Go → Scan QR code — and **your update opens in their Expo Go**. You can also generate the QR/URL directly:

```text
# Expo Go target (slug defaults to "exp"); returns a plain URL with format=url
https://qr.expo.dev/eas-update?projectId=<your-project-id>&branchId=<branch-id>&format=url
```

**Gotcha:** the update's **runtime/SDK must line up with the teammate's Expo Go**. Expo Go can only open projects built for an SDK version it ships, and it supports just a **narrow window of recent SDKs** — an Expo Go that's too old (or updated past your SDK) simply won't load your SDK-54 update. If it silently fails to open, first confirm both sides are on the same SDK, then that the runtimeVersion is the `sdkVersion` policy (not `appVersion`, which is for standalone builds). See [troubleshooting.md](troubleshooting.md).

## Limits & when to graduate

EAS Update + Expo Go is for **dev previews and quick review only**. Expo Go can't run custom native modules or config-plugin changes, and there's no home-screen icon or offline install — the teammate needs Expo Go and network access.

For real distribution you **graduate to `eas build`**: a **development build** to test native modules, and a **production build** (`eas build` → `eas submit`) for the App Store / Play Store. Those binaries still receive `eas update` pushes over-the-air, so the publish workflow above carries over. Verify current build/submit flags against the EAS docs before you get there.
