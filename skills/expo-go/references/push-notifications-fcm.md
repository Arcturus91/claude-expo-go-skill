# Push notifications with FCM (a post-MVP follow-up)

A roadmap for adding remote push to a working Expo Go MVP — the moment you graduate from Expo Go to a development build, plus the exact `expo-notifications` + FCM wiring, verified against Expo SDK 54 / expo-notifications ~0.32 / firebase JS SDK v12 / firebase-admin.

## Read this first — push needs a development build, not Expo Go

**Remote push notifications do not work in Expo Go.** Since **Expo SDK 53**, `expo-notifications` **removed Android push support from Expo Go**, and it stays removed in SDK 54. So the first thing this feature forces is leaving Expo Go for a **development build** (dev client). There is no flag or workaround — Expo Go will simply, silently, never receive a remote push.

What still works vs what doesn't, so you don't chase ghosts:

- **Local / scheduled notifications** (`scheduleNotificationAsync`, `setNotificationHandler`, in-app banners) — **still work in Expo Go.** If your MVP only needs a reminder at a time you compute on-device, you may not need any of this file.
- **Remote push** (a server sends to the device via FCM/APNs) — **does not work in Expo Go.** This is the part that requires a dev build.

The dev build is the same app running your JS, just with the native push modules compiled in. See [expo-go-workflow.md](expo-go-workflow.md) for the "graduating from Expo Go → dev client" mechanics; everything below assumes you have done that.

## Two ways to do it

1. **`expo-notifications` + Expo's push service (recommended).** You keep the Expo/EAS workflow. You get an **Expo push token** and send to Expo's push API; Expo relays to **FCM (Android)** and **APNs (iOS)** under the hood. Lowest setup, one send path for both platforms, and it stays close to the pure-JS mental model you already have.
2. **`@react-native-firebase/messaging` (raw FCM).** A native config plugin that talks to FCM directly. You'd get native FCM token, `getDevicePushTokenAsync`, and native FCM features — but you also take on native config and leave the tidy Expo/EAS path. Also requires a dev build. Only reach for this if you need FCM-native features Expo's service doesn't expose (see the last section).

**Recommendation:** most MVPs graduating to push should take **approach 1**. The tradeoff is that you delegate delivery to Expo's service (a dependency) in exchange for far less native plumbing and a single cross-platform send.

## Setup (approach 1, step by step)

### Install and configure the plugin

```bash
npx expo install expo-notifications
```

Add the config plugin in `app.json` (icon/color/sounds are optional but the plugin entry is what wires the native module into your dev build):

```json
{
  "expo": {
    "plugins": [
      ["expo-notifications", { "icon": "./assets/notification-icon.png", "color": "#ffffff" }]
    ],
    "android": {
      "googleServicesFile": "./google-services.json"
    }
  }
}
```

Download `google-services.json` from the Firebase console (Project settings → your Android app) and point `android.googleServicesFile` at it. **Gotcha:** `google-services.json` holds only public identifiers, so it's safe to commit — but it's read at **build time**, so a change to it means a **new dev build**, not just a reload.

iOS uses **APNs**, not FCM. On the Expo push path, EAS manages your APNs key for you; you need an Apple Developer account and a **real device** (the iOS simulator cannot receive push).

### Build the dev client (this replaces Expo Go)

```bash
eas build --profile development --platform android
eas build --profile development --platform ios
```

Install the resulting build on a real device and run your JS against it with `npx expo start --dev-client`. This binary — not Expo Go — is where push works.

### Provide FCM v1 credentials to EAS

Even on the Expo push path, **Android delivery still goes through FCM**, so EAS needs credentials to send. **Use FCM HTTP v1 (a service account) — the legacy FCM "server key" is deprecated.** In the Firebase console enable Cloud Messaging and generate a service-account **private key** (JSON). Then upload it to EAS: on **expo.dev → Project settings → Credentials**, pick your Android Application Identifier, and under **Service Credentials → FCM V1 service account key** choose **Add a service account key** and upload the JSON. (The `eas credentials` CLI exposes the same under the Android → push notifications menu — pick the FCM V1 service account option, not a legacy key.) Verify the current menu labels against current Expo docs, as the dashboard wording shifts.

### Runtime: permission, token, handler, listeners

```ts
import * as Notifications from "expo-notifications";
import Constants from "expo-constants";
import { Platform } from "react-native";

// Decide how a notification shows while the app is foregrounded (SDK 54 fields).
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldPlaySound: true,
    shouldSetBadge: false,
    shouldShowBanner: true,
    shouldShowList: true,
  }),
});

export async function registerForPushAsync(): Promise<string | undefined> {
  if (Platform.OS === "android") {
    await Notifications.setNotificationChannelAsync("default", {
      name: "default",
      importance: Notifications.AndroidImportance.MAX,
    });
  }

  const { status: existing } = await Notifications.getPermissionsAsync();
  let status = existing;
  if (existing !== "granted") {
    status = (await Notifications.requestPermissionsAsync()).status;
  }
  if (status !== "granted") return undefined;

  const projectId =
    Constants?.expoConfig?.extra?.eas?.projectId ?? Constants?.easConfig?.projectId;
  const token = (await Notifications.getExpoPushTokenAsync({ projectId })).data;
  return token; // "ExponentPushToken[…]"
}
```

Add listeners once (e.g. in a top-level effect) and remove them on cleanup:

```ts
const received = Notifications.addNotificationReceivedListener((n) => {
  /* arrived while app is open */
});
const response = Notifications.addNotificationResponseReceivedListener((r) => {
  /* user tapped it — route based on r.notification.request.content.data */
});
return () => {
  received.remove();
  response.remove();
};
```

**Which token, which sender:**
- **`getExpoPushTokenAsync({ projectId })`** → an **Expo push token** (`ExponentPushToken[…]`). Send it via **Expo's push API**. This is the recommended path.
- **`getDevicePushTokenAsync()`** → the **native FCM (Android) / APNs (iOS)** token. Only send this via a native sender (e.g. `firebase-admin`) — the Expo push API expects the Expo token, not this one.

### Persist the token on the user's doc

Write the token onto the signed-in user's Firestore doc so a backend can target them:

```ts
import { doc, setDoc, deleteField } from "firebase/firestore";
import { db } from "../firebase/config";

await setDoc(
  doc(db, "users", uid),
  { expoPushToken: token, pushPlatform: Platform.OS },
  { merge: true }
);
```

**Gotcha:** push tokens **rotate** — refresh on each app start / sign-in and rewrite if it changed, and **clear it on sign-out** (`{ expoPushToken: deleteField() }`) so you don't push to a device the user left. (expo-notifications can also emit token changes — verify `addPushTokenListener` against current docs.) Lock the field down in your rules so a user can only write their **own** token — see [firestore-data-rules.md](firestore-data-rules.md). Keeping the token on the row the owner already writes fits the denormalized, owner-writes-its-own-data model in [architecture-principles.md](architecture-principles.md).

## Sending a notification

**Never loop over other users' tokens from the client.** Send from a trusted backend (a Firebase Cloud Function), which reads the target user's doc and sends. Two shapes, matching the two token types:

**A. Expo push token → Expo push API** (`POST https://exp.host/--/api/v2/push/send`). No secret key; the token itself is the address. A single object or an array of up to 100:

```ts
await fetch("https://exp.host/--/api/v2/push/send", {
  method: "POST",
  headers: { Accept: "application/json", "Content-Type": "application/json" },
  body: JSON.stringify({
    to: expoPushToken, // "ExponentPushToken[…]"
    title: "Hello",
    body: "World",
    data: { screen: "home" },
  }),
});
```

The response is `{ data: [{ status: "ok", id } | { status: "error", message, details }] }`. **Gotcha:** handle `details.error === "DeviceNotRegistered"` by deleting that stored token — the device uninstalled or the token rotated.

**B. Native FCM token → `firebase-admin`** (only if you stored `getDevicePushTokenAsync` tokens). Runs server-side in a Cloud Function, keeping credentials off the client:

```ts
import { initializeApp } from "firebase-admin/app";
import { getMessaging } from "firebase-admin/messaging";

initializeApp();

await getMessaging().send({
  token: deviceFcmToken,
  notification: { title: "Hello", body: "World" },
  data: { screen: "home" },
});
```

`getMessaging().send(message)` is the current API (the namespaced `admin.messaging().send(...)` is equivalent). **Gotcha:** the legacy `sendToDevice` / `sendMulticast` helpers were **removed in firebase-admin v13** — use `send()`, or `sendEach()` for a batch. firebase-admin speaks FCM HTTP v1 for you, so you don't hand-roll the OAuth token.

## Gotchas and testing

- **Expo Go silently drops remote push** — if nothing arrives, first confirm you're on the **dev build**, not Expo Go.
- **iOS:** real device only (no simulator push) and APNs must be configured; **Android:** the dev build must include `google-services.json` and EAS must hold the FCM v1 credentials.
- **Foreground vs background/quit differ.** Foreground display is governed by your `setNotificationHandler`; background/quit delivery is the OS's job and the payload lands in the response listener when tapped. Test all three states.
- **Always test on a real device.** Use Expo's push tool or a `curl` to the send endpoint with a real token before wiring a backend.

## When `@react-native-firebase` instead

Reach for approach 2 only when you need FCM features Expo's service doesn't surface — e.g. FCM topic subscriptions, data-only background handlers with native control, or an existing FCM backend you must speak to directly. The cost: native config, a heavier dev build, and leaving the Expo push convenience (one token, one cross-platform send). For most apps graduating from an Expo Go MVP, approach 1 is the better trade. See [firebase-setup.md](firebase-setup.md) for why the pure-JS `firebase` SDK stays the client default.
