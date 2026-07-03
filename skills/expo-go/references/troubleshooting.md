# Troubleshooting: setup traps

The non-obvious failures that hit an Expo Go (SDK 54) + Firebase app during setup, each as symptom → cause → fix.

## 1. `npm install` fails on a peer-dep conflict

**Symptom:** install aborts with `ERESOLVE`, complaining about `react-dom` (via `vaul` or `@radix-ui/*`).

**Cause:** expo-router pulls **web-only** transitive deps that peer-require `react-dom`. A React Native app has no `react-dom`, so npm's strict peer resolver refuses to continue.

**Fix:** add a project **`.npmrc`** so npm tolerates the irrelevant web peers:

```ini
# expo-router pulls web-only deps (vaul/@radix-ui) that peer-require react-dom.
# We never build web, so those peers are irrelevant.
legacy-peer-deps=true
```

Prefer `npx expo install <pkg>` over raw `npm install` for native-adjacent packages — it also picks the SDK-matching version. See [project-setup.md](project-setup.md).

## 2. `tsc` errors on `getReactNativePersistence` from `firebase/auth`

**Symptom:** `tsc --noEmit` reports `getReactNativePersistence` has no exported member, even though the app runs fine on device.

**Cause:** that export only exists in Firebase's **react-native export condition**. Metro resolves it at runtime, but tsc's `bundler`/`node16` module resolution reads a different export map and never sees it. The code is correct; the type checker is looking at the wrong entry point.

**Fix:** import it with a scoped suppression and **leave the suppression in place** — removing it re-breaks `tsc`:

```ts
// @ts-expect-error — RN-only export; Metro resolves it at runtime, tsc does not.
import { getReactNativePersistence } from "firebase/auth";
```

Do not "fix" it by switching module resolution globally. Details in [firebase-setup.md](firebase-setup.md).

## 3. Firestore emulator won't start

**Symptom:** `firebase emulators:start` hangs or dies with a Java error; the **Firestore** emulator specifically fails to boot.

**Cause:** the Firestore emulator is a **Java** process and needs a JDK on `PATH`. (The **Auth** emulator is pure Node and does **not** need Java — Auth-only setups never hit this.)

**Fix:** install a JDK — e.g. Homebrew OpenJDK, which is **keg-only**, so its binary lives at `/opt/homebrew/opt/openjdk/bin` and is **not** on `PATH` by default. Prepend it in the emulator script rather than mutating your global shell:

```jsonc
// package.json
"emu": "firebase emulators:start --project demo-app",
"test:e2e": "PATH=\"/opt/homebrew/opt/openjdk/bin:$PATH\" firebase emulators:exec --project demo-app \"npm run test:int\""
```

Verify with `java -version`. More in [testing.md](testing.md).

## 4. User-facing dates are off by one near midnight

**Symptom:** an entry created at 9pm shows tomorrow's date, or a reminder fires on the wrong day. Only happens in the evening.

**Cause:** `new Date().toISOString().slice(0, 10)` returns the **UTC** date. In any negative-UTC-offset timezone (the Americas), an evening local timestamp has already rolled to the next UTC day, so the "date" jumps ahead.

**Fix:** never derive a user-visible or scheduled date from UTC. Build it from **local** getters:

```ts
export function localDateKey(d = new Date()): string {
  const pad = (n: number) => String(n).padStart(2, "0");
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`;
}
```

Use `localDateKey()` for anything the user sees or that you schedule against. This is a whole bug class, not a one-off — keep the UTC `toISOString().slice(0,10)` pattern out of date logic entirely. A pure helper like this belongs in `src/engine/` where it is trivially unit-testable.

## 5. Metro resolves a module that tsc (or the IDE) says is missing

**Symptom:** the app bundles and runs, but `tsc` / your editor flags a type-only or export mismatch on an RN-specific import (see trap #2 for the canonical case).

**Cause:** two resolvers disagree. **tsc** uses `bundler`/`node16` resolution and reads package `exports` maps one way; **Metro** has its own resolver and applies the `react-native` condition. When a package ships an RN-only export, they legitimately diverge.

**Fix:** prefer a **scoped `// @ts-expect-error`** on the single offending line over changing `moduleResolution` globally — a global change to satisfy one import tends to break resolution for dozens of others. Trust Metro for runtime behavior; suppress narrowly for the type checker.

---

See [project-setup.md](project-setup.md) for the clean-slate configuration these traps assume.
