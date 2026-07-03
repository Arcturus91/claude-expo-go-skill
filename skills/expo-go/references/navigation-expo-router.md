# Navigation with expo-router

File-based navigation for Expo Go: your folder tree under `app/` *is* your route map, with per-segment layouts and redirect guards. Targets **expo-router ~6.0 / Expo SDK 54**.

## File-based routing basics

Every file in `app/` becomes a route; there is no central route config.

- `app/home.tsx` → `/home`
- `app/settings.tsx` → `/settings`
- `app/index.tsx` → the **index** (default) route of its folder (`app/index.tsx` = `/`)
- `_layout.tsx` → wraps every route in that folder (persistent UI: a stack, tabs, providers). It is not itself a route.
- `[id].tsx` → a **dynamic** route (see below).

**Gotcha:** filenames drive URLs, so keep them lowercase and stable. Renaming a file changes its `href`.

## Layout groups `(group)`

A folder wrapped in parentheses groups routes **without adding a URL segment**. `app/(app)/home.tsx` is still `/home`, not `/(app)/home`. Use groups to give each audience its own `_layout.tsx` (and its own guard):

```
app/
  _layout.tsx          // root: providers + <Stack>
  index.tsx            // entry guard → redirects by auth/role
  (auth)/
    _layout.tsx        // <Stack> for signed-out screens
    sign-in.tsx        // → /sign-in
  (app)/
    _layout.tsx        // <Tabs> + "member" guard
    home.tsx           // → /home
    items.tsx          // → /items
    settings.tsx       // → /settings
  (admin)/
    _layout.tsx        // <Stack> + "admin" guard
    dashboard.tsx      // → /dashboard
  item/
    [id].tsx           // → /item/:id  (NOT inside a group — see guard gotcha)
```

## The root layout

`app/_layout.tsx` is the outermost wrapper. Put app-wide providers here (safe-area, your auth context) and a **headerless** root `<Stack>` so each group controls its own header:

```tsx
import { Stack } from "expo-router";
import { SafeAreaProvider } from "react-native-safe-area-context";
import { AuthProvider } from "../src/state/AuthProvider";

export default function RootLayout() {
  return (
    <SafeAreaProvider>
      <AuthProvider>
        <Stack screenOptions={{ headerShown: false }} />
      </AuthProvider>
    </SafeAreaProvider>
  );
}
```

The `AuthProvider` here is what every guard below reads from — see [auth-and-state.md](auth-and-state.md).

## Redirect-based guards

`app/index.tsx` is the entry gate. It reads auth/role/loading from context and returns a `<Redirect>` — showing a splash while auth is still resolving so you never flash the wrong screen:

```tsx
import { Redirect } from "expo-router";
import { ActivityIndicator, View } from "react-native";
import { useAuth } from "../src/state/AuthProvider";

function Splash() {
  return (
    <View style={{ flex: 1, alignItems: "center", justifyContent: "center" }}>
      <ActivityIndicator />
    </View>
  );
}

export default function Index() {
  const { user, profile, loading } = useAuth();

  if (loading) return <Splash />;              // auth still resolving
  if (!user) return <Redirect href="/sign-in" />;
  if (!profile) return <Splash />;             // signed in, profile still loading
  if (profile.role === "admin") return <Redirect href="/dashboard" />;
  return <Redirect href="/home" />;
}
```

**Gotcha — guard every segment, not just `index`.** `index.tsx` only runs at `/`. A deep link straight to `/dashboard` skips it, so put the same check in each group's `_layout.tsx`. The group layout returns a `<Redirect>` before rendering its navigator:

```tsx
// app/(app)/_layout.tsx  — the "member" area
import { Redirect, Tabs } from "expo-router";
import { useAuth } from "../../src/state/AuthProvider";

export default function AppLayout() {
  const { user, profile, loading } = useAuth();

  if (loading) return null;
  if (!user) return <Redirect href="/sign-in" />;
  if (profile?.role !== "member") return <Redirect href="/" />;

  return (
    <Tabs screenOptions={{ headerShown: true }}>
      <Tabs.Screen name="home" options={{ title: "Home" }} />
      <Tabs.Screen name="items" options={{ title: "Items" }} />
      <Tabs.Screen name="settings" options={{ title: "Settings" }} />
    </Tabs>
  );
}
```

**Gotcha — a route outside every group has no parent guard.** `app/item/[id].tsx` is *not* inside `(app)` or `(admin)`, so a deep link to `/item/42` reaches it directly. Either move it into a guarded group, or guard it inside the screen itself. Never assume a parent guard is protecting a top-level route.

**Built-in alternative:** expo-router v6 also ships `<Stack.Protected guard={...}>` to declaratively hide screens in a layout. The manual redirect pattern above stays explicit about *where* unauthorized users land, which is why these examples use it.

## Stacks vs Tabs

Both are navigators you return from a `_layout.tsx`; you register children with `<Stack.Screen>` / `<Tabs.Screen>` to set titles and options (unlisted routes still work with defaults).

- **`<Tabs>`** — a persistent bottom tab bar; use for the member area's top-level sections (`(app)` above).
- **`<Stack>`** — push/pop with a header and a back button; use for drill-down flows like the admin area:

```tsx
// app/(admin)/_layout.tsx
import { Redirect, Stack } from "expo-router";
import { useAuth } from "../../src/state/AuthProvider";

export default function AdminLayout() {
  const { user, profile, loading } = useAuth();
  if (loading) return null;
  if (!user) return <Redirect href="/sign-in" />;
  if (profile?.role !== "admin") return <Redirect href="/" />;

  return (
    <Stack>
      <Stack.Screen name="dashboard" options={{ title: "Dashboard" }} />
      <Stack.Screen name="item/[id]" options={{ title: "Item" }} />
    </Stack>
  );
}
```

## Navigating

- **`<Redirect href="/home" />`** — declarative; return it from render (guards, entry gates). Replaces the current route, no history entry.
- **`useRouter()` → `router.push("/item/42")`** — imperative; adds to history, Back returns.
- **`router.replace("/home")`** — imperative; swaps the current route. Use **after auth actions** (sign-in, redeeming a code) so Back doesn't return to the login screen. `router.back()` pops one level.
- **`<Link href="/settings">`** — declarative navigation from JSX (a tappable link), like an `<a>` tag.

```tsx
import { useRouter } from "expo-router";

const router = useRouter();
// after a successful sign-in / onboarding step:
router.replace("/home");        // Back won't return to the form
// drilling into detail:
router.push(`/item/${id}`);
```

## Dynamic routes & params

`app/item/[id].tsx` matches `/item/anything`. Read the segment with `useLocalSearchParams`, typed via its generic, and guard against a missing/blank value before fetching:

```tsx
import { useLocalSearchParams, useRouter } from "expo-router";
import { useEffect, useState } from "react";

export default function ItemScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  const router = useRouter();
  const [item, setItem] = useState(null);

  useEffect(() => {
    if (!id) return;                 // blank/missing id → skip the fetch
    (async () => setItem(await getItem(id)))();
  }, [id]);

  // ... render item; router.back() to return
}
```

**Gotcha:** `useLocalSearchParams` values are always `string | string[] | undefined` at runtime — the generic is a compile-time hint, not validation. Always handle the missing case.

---

**Route guards are UX, not security.** A redirect only hides a screen; it does nothing to stop a request. Anyone can hit your backend directly, so enforce access on the server too — see [firestore-data-rules.md](firestore-data-rules.md).
