# Firebase setup

Initialize the Firebase JS SDK once — app, auth (with React Native session persistence), and Firestore — as shared singletons that also point at the local emulator when you flip a flag.

## Why the JS SDK (not @react-native-firebase)

Use the **`firebase` JS SDK** (v12). It is pure JavaScript, so it runs unchanged inside **Expo Go** — no custom native build, no config plugin, no `expo prebuild`. `@react-native-firebase` wraps the native iOS/Android SDKs and therefore requires a development build; it will **not** load in Expo Go. Stay on the JS SDK unless you have a concrete reason to eject.

Put everything below in one module (e.g. `src/firebase/config.ts`) and import it once at startup for its side effects, plus the exported `auth` / `db`.

## Config from env

Build `firebaseConfig` from `process.env.EXPO_PUBLIC_*` with safe demo fallbacks, so a fresh clone boots against the emulator with no `.env` file:

```ts
const firebaseConfig = {
  apiKey: process.env.EXPO_PUBLIC_FIREBASE_API_KEY ?? "demo-key",
  authDomain: process.env.EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN ?? "demo-app.firebaseapp.com",
  projectId: process.env.EXPO_PUBLIC_FIREBASE_PROJECT_ID ?? "demo-app",
  storageBucket: process.env.EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.EXPO_PUBLIC_FIREBASE_APP_ID ?? "demo-app",
};
```

**Gotcha:** `EXPO_PUBLIC_*` vars are inlined into the client bundle at build time, so they ship to every device — they are **not secrets**. A Firebase `apiKey` is only a project identifier (not a credential), so exposing it is expected; but **never** put a private key or admin secret in an `EXPO_PUBLIC_` var. Access control lives entirely in your Firestore security rules, not in the config — see [firestore-data-rules.md](firestore-data-rules.md).

## Initialize once

Firebase throws if you call `initializeApp` twice, and Fast Refresh re-runs modules. Guard it:

```ts
import { getApp, getApps, initializeApp } from "firebase/app";

const app = getApps().length ? getApp() : initializeApp(firebaseConfig);
```

## Auth persistence in RN

```ts
import AsyncStorage from "@react-native-async-storage/async-storage";
// @ts-expect-error -- RN-only export, present at runtime, absent from the web type surface.
import { Auth, getAuth, getReactNativePersistence, initializeAuth } from "firebase/auth";

let authInstance: Auth;
try {
  authInstance = initializeAuth(app, { persistence: getReactNativePersistence(AsyncStorage) });
} catch {
  authInstance = getAuth(app); // already initialized (Fast Refresh) → reuse it
}
export const auth = authInstance;
```

**Gotcha:** you must use `initializeAuth(app, { persistence: getReactNativePersistence(AsyncStorage) })`. Plain `getAuth(app)` uses in-memory persistence in React Native, so the session is **lost on every reload** and the user is bounced back to sign-in.

**Gotcha:** `getReactNativePersistence` is exported only from firebase's `react-native` build. Metro resolves it at runtime via the `react-native` export condition, but tsc / the bundler's type resolution sees the web type surface and reports it as missing. Import it with a `// @ts-expect-error` on the line above and keep the suppression — removing it breaks `tsc`. (With `@react-native-async-storage/async-storage` 2.x you pass the default export directly, as shown.)

**Gotcha:** `initializeAuth` may run only **once** per app. On Fast Refresh the module re-executes and the call throws; the `try/catch` falls back to `getAuth(app)` to grab the already-initialized instance.

## Firestore singleton

```ts
import { Firestore, getFirestore } from "firebase/firestore";

export const db: Firestore = getFirestore(app);
```

## Local emulator wiring

Gate emulator wiring behind an env flag so production never points at localhost. Call the connect functions **synchronously, right after** the instances exist and before any read/write:

```ts
import { connectAuthEmulator } from "firebase/auth";
import { connectFirestoreEmulator } from "firebase/firestore";

if (process.env.EXPO_PUBLIC_USE_EMULATOR === "1") {
  connectAuthEmulator(auth, "http://127.0.0.1:9099", { disableWarnings: true });
  connectFirestoreEmulator(db, "127.0.0.1", 8080);
}
```

**Gotcha:** use a `demo-` prefixed `projectId` (e.g. `demo-app`) for local work. The Firebase Emulator Suite treats any `demo-*` project as offline-only, so it runs **without real credentials** and never touches a live project. Use `127.0.0.1`, not `localhost` — on a device/emulator the two don't always resolve the same. See [testing.md](testing.md) for starting the emulator and pointing a device at your host.

Next: consume `auth` / `db` from the auth context in [auth-and-state.md](auth-and-state.md).
