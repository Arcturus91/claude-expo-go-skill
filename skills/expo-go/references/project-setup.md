# Project setup

Scaffold a TypeScript Expo (SDK 54) app that runs in **Expo Go**, wire up expo-router, and lay out a `src/` tree with pure, testable domain logic.

## Create the app

The default `create-expo-app` template is already **TypeScript + expo-router** — the fastest correct start:

```bash
npx create-expo-app@latest my-app
cd my-app
```

If you want a minimal starting point instead, pass `--template` (run `npx create-expo-app --template` to see the current list, e.g. `blank`) and add expo-router yourself as below. When in doubt, verify against current Expo docs.

Pin the toolchain the way the working app does: Expo `~54.0.x`, React Native `0.81.5`, React `19.1.0`, TypeScript `~5.9`.

## Add expo-router (existing / minimal project)

Skip this if you used the default template — it is already wired. To add routing to a bare project, let Expo pick SDK-compatible versions:

```bash
npx expo install expo-router react-native-safe-area-context react-native-screens expo-linking expo-constants expo-status-bar
```

Then set the entry point in **package.json** (this replaces `App.tsx` as the app root):

```json
{ "main": "expo-router/entry" }
```

And register the plugin + a deep-link scheme in **app.json**:

```json
{
  "expo": {
    "scheme": "myapp",
    "plugins": ["expo-router"]
  }
}
```

expo-router is file-based: the first screen is `app/_layout.tsx` (root layout) plus `app/index.tsx`.

## Folder layout

Keep **routes** and **logic** apart. `app/` is owned by the router; everything else lives in `src/`:

```
app/                 # routes only — one file = one screen (expo-router)
  _layout.tsx
  index.tsx
src/
  firebase/          # Firebase init + typed data access (see firebase-setup.md)
  state/             # app/session state (context, stores)
  ui/                # shared presentational components
  engine/            # PURE domain logic — no React, no Firestore imports
assets/
```

**Gotcha:** keep `src/engine/` free of any `react`, `react-native`, or `firebase` imports. Pure functions in/out (dates, scoring, role rules) unit-test in milliseconds under plain Jest with no native mocks or emulator. The moment engine code imports Firestore, it stops being testable in isolation.

To use a top-level `src/` with expo-router, see the router "src directory" reference — verify against current Expo docs.

## Dependencies (Expo Go compatible)

Every package below is JS-only or bundled into Expo Go, so **no custom native build** is needed. Versions from a working SDK 54 app:

| Package | Version | Purpose |
| --- | --- | --- |
| `expo-router` | `~6.0` | file-based navigation |
| `firebase` | `^12.15` | auth + Firestore (JS SDK) |
| `@react-native-async-storage/async-storage` | `2.2.0` | persists the Firebase auth session |
| `react-native-safe-area-context` | `~5.6` | notch/safe-area insets |
| `react-native-screens` | `~4.16` | native screen primitives |
| `expo-notifications` | `~0.32` | local notifications (see caveats in [expo-go-workflow.md](expo-go-workflow.md)) |
| `expo-constants` | `~18.0` | read `expo` config / manifest at runtime |
| `expo-status-bar` | `~3.0` | status-bar control |

**Gotcha:** installing expo-router drags in web-only transitive deps (`vaul`, `@radix-ui/*`) that peer-require `react-dom`. An RN app has no `react-dom`, so a strict `npm install` **aborts on the peer conflict**. Fix it once with a project **`.npmrc`**:

```ini
# expo-router pulls web-only deps (vaul/@radix-ui) that peer-require react-dom.
# We never build web, so those peers are irrelevant. Keep npm from blocking installs.
legacy-peer-deps=true
```

Prefer `npx expo install <pkg>` over `npm install <pkg>` for anything native-adjacent — it resolves the version that matches your SDK. See [troubleshooting.md](troubleshooting.md) for the full setup-trap list.

## Environment variables

Only variables prefixed **`EXPO_PUBLIC_`** are inlined into the client bundle and readable as `process.env.EXPO_PUBLIC_*`. Everything else is stripped at build time. Since a mobile app ships to the device, treat all of these as public — never put a true secret here. Commit a **`.env.example`** (not `.env`):

```bash
# .env.example — copy to .env and fill in. All values ship to the client.
EXPO_PUBLIC_FIREBASE_API_KEY=
EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN=
EXPO_PUBLIC_FIREBASE_PROJECT_ID=
EXPO_PUBLIC_FIREBASE_APP_ID=
```

Add `.env` to `.gitignore`. See [firebase-setup.md](firebase-setup.md) for consuming these.

## TypeScript config

Extend Expo's base and turn on `strict`. **tsconfig.json**:

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "types": ["node", "jest", "react"]
  }
}
```

## npm scripts

```json
{
  "scripts": {
    "start": "expo start",
    "android": "expo start --android",
    "ios": "expo start --ios",
    "web": "expo start --web",
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "format": "prettier --write \"**/*.{ts,tsx,js,jsx,json,md}\""
  }
}
```

Run `npm run typecheck` and `npm run lint` clean before every commit — `strict` + `tsc --noEmit` catches most RN/Firebase mistakes before they hit a device. Lint uses ESLint flat config (`eslint.config.mjs`) with `eslint-config-expo` + `eslint-config-prettier` so formatting never surfaces as a lint error.

Next: [expo-go-workflow.md](expo-go-workflow.md) to run it on a device.
