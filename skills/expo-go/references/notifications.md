# Local & scheduled notifications in Expo Go

Schedule on-device reminders about a user's own items/tasks with `expo-notifications` (~0.32, Expo SDK 54), gate them on the OS permission, and reconcile them safely. Verified against Expo SDK 54 docs.

## Local vs remote (start here)

- **Local / scheduled notifications WORK in Expo Go** (SDK 54): `scheduleNotificationAsync`, channels, and the foreground handler all run fine. This is everything below.
- **Remote PUSH notifications DO NOT work in Expo Go** — support was removed from Expo Go in SDK 53+. `getExpoPushTokenAsync` / device push tokens require a **development build**. For that path see [push-notifications-fcm.md](push-notifications-fcm.md).
- So: reminders you compute and schedule on-device = fine in Expo Go. Anything a server sends = dev build. Don't call `getExpoPushTokenAsync` in an Expo-Go-only build; it will fail.

Install: `npx expo install expo-notifications`.

## Permissions

Never assume the OS granted anything. Check first, request only if needed, and branch on the result.

```ts
import * as Notifications from 'expo-notifications';
import { Platform, Linking } from 'react-native';

// Returns true only when the OS will actually deliver notifications.
export async function ensureNotificationPermission(): Promise<boolean> {
  const current = await Notifications.getPermissionsAsync();
  // `granted` is a convenience boolean; on iOS also accept provisional (quiet) auth.
  if (current.granted ||
      current.ios?.status === Notifications.IosAuthorizationStatus.PROVISIONAL) {
    return true;
  }
  // undetermined → we may prompt. denied with canAskAgain === false → we may NOT.
  if (current.status === Notifications.PermissionStatus.UNDETERMINED || current.canAskAgain) {
    const asked = await Notifications.requestPermissionsAsync();
    return asked.granted;
  }
  // Denied and can't ask again: only Settings can flip it. Send the user there.
  Linking.openSettings();
  return false;
}
```

- `getPermissionsAsync()` / `requestPermissionsAsync()` resolve to a `PermissionResponse`: `{ status, granted, canAskAgain, expires, ios?, android? }`. `status` is one of `"granted" | "denied" | "undetermined"`.
- **Gotcha — iOS is one-shot.** The system prompt appears **once**. If the user denies, `requestPermissionsAsync()` will never show it again (`canAskAgain` becomes `false`); you must deep-link to `Linking.openSettings()`. Don't loop calling `requestPermissionsAsync`.
- **Gotcha — Android 13+ (API 33) needs the `POST_NOTIFICATIONS` runtime permission.** `expo-notifications` adds the `POST_NOTIFICATIONS` manifest entry for you, but you still must call `requestPermissionsAsync()` to trigger the runtime prompt. On older Android it's granted at install with no prompt. On Android, creating a channel (below) before requesting helps the prompt behave correctly.
- **iOS provisional / quiet permission:** you can request `ios: { allowProvisional: true }` to deliver quietly to Notification Center without a prompt; treat `IosAuthorizationStatus.PROVISIONAL` as "allowed" as shown above.
- **Gotcha — gate on the OS permission, not just an in-app toggle.** A user can enable your app's "Remind me" switch and still have notifications denied at the OS level. Always re-check `getPermissionsAsync()` at schedule time; a toggle in your DB is not proof of delivery.

## Foreground handler

By default nothing shows while your app is open. Register a handler once at module load (top level, before render):

```ts
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldPlaySound: false,
    shouldSetBadge: false,
    shouldShowBanner: true, // SDK 53+ replaced shouldShowAlert with Banner + List
    shouldShowList: true,
  }),
});
```

## Android channels

Android groups notifications into channels; the channel (not the notification) controls importance, sound, and vibration. Create it once at startup, then reference its `channelId` in triggers.

```ts
if (Platform.OS === 'android') {
  await Notifications.setNotificationChannelAsync('reminders', {
    name: 'Reminders',
    importance: Notifications.AndroidImportance.HIGH, // HIGH/MAX = heads-up banner
    vibrationPattern: [0, 250, 250, 250],
    lightColor: '#FF231F7C',
  });
}
```

**Gotcha:** channel settings are locked in at **first creation**. If you later change importance/sound in code, existing installs keep the old values until the app is reinstalled — the user can only change them in system settings. Pick sensible defaults up front.

## Split pure logic from the applier

Keep *what to schedule* (pure, unit-testable, zero expo imports) separate from *talking to the OS*. See [architecture-principles.md](architecture-principles.md) §1 and [testing.md](testing.md).

```ts
// schedule.ts — PURE. No expo import. Fully unit-testable.
export type PlannedNotification = {
  key: string;                 // stable dedupe id, e.g. `item-${id}`
  title: string;
  body: string;
  fireAt: Date;                // absolute local fire time
};

// now + prefs + your domain data in → a plain list out. Deterministic.
export function computeSchedule(
  now: Date,
  prefs: { enabled: boolean; hour: number; minute: number },
  items: { id: string; title: string; dueOn: string }[] // dueOn = local 'YYYY-MM-DD'
): PlannedNotification[] {
  if (!prefs.enabled) return [];
  return items
    .map(it => ({
      key: `item-${it.id}`,
      title: 'Reminder',
      body: it.title,
      fireAt: atLocalTime(it.dueOn, prefs.hour, prefs.minute),
    }))
    .filter(p => p.fireAt.getTime() > now.getTime()); // future only (see Dates)
}
```

```ts
// applier.ts — thin. The ONLY file that imports expo-notifications.
async function scheduleOne(p: PlannedNotification) {
  await Notifications.scheduleNotificationAsync({
    content: { title: p.title, body: p.body, data: { app: 'myapp', key: p.key } },
    trigger: { type: Notifications.SchedulableTriggerInputTypes.DATE, date: p.fireAt },
  });
}
```

Trigger shapes (all verified): `{ type: DATE, date }`, `{ type: TIME_INTERVAL, seconds, repeats? }`, `{ type: DAILY, hour, minute }`, `{ type: WEEKLY, weekday /* 1=Sun */, hour, minute }`. A bare `Date` or Unix number is treated as `DATE`. Add `channelId` to any trigger on Android.

## The reconcile pattern

You almost never want "add another notification". You want "make the OS match my plan". **Tag** yours, cancel only yours, then schedule the fresh set.

```ts
export async function reconcile(planned: PlannedNotification[]) {
  // 1. Cancel only OUR notifications (leave other libs / OS ones alone).
  const scheduled = await Notifications.getAllScheduledNotificationsAsync();
  await Promise.all(
    scheduled
      .filter(n => n.content.data?.app === 'myapp')
      .map(n => Notifications.cancelScheduledNotificationAsync(n.identifier))
  );
  // 2. Schedule the fresh set.
  for (const p of planned) await scheduleOne(p);
}
```

`cancelAllScheduledNotificationsAsync()` exists but nukes **everyone's** — prefer the tagged filter above.

## Reconcile pitfalls (real)

- **Gotcha — cancel-then-schedule is NOT atomic.** Two reconciles running concurrently (e.g. a screen focus + a data sync) both read the old set, both cancel, both schedule → duplicates. Serialize it: a module-level in-flight promise/mutex so a second call awaits the first.
- **Gotcha — don't swallow scheduling errors.** If cancel succeeds but `scheduleNotificationAsync` throws and you catch-and-ignore it as "best effort", you now have **zero** reminders while your DB says "enabled". Surface the failure (log + user-visible state), or schedule before cancelling the old ones.
- **Gotcha — iOS caps pending local notifications at 64** and silently drops everything past it. Bound your plan (e.g. only the next N days / next 64 items) so the ones that matter survive.

## Dates

- **Gotcha — schedule against LOCAL time.** `new Date().toISOString().slice(0,10)` is **UTC** and is off by a day in negative-UTC-offset zones (Americas) after mid-afternoon. Use a local helper:

```ts
export function localYMD(d: Date): string { // 'YYYY-MM-DD' in the device's zone
  const p = (n: number) => String(n).padStart(2, '0');
  return `${d.getFullYear()}-${p(d.getMonth() + 1)}-${p(d.getDate())}`;
}
export function atLocalTime(ymd: string, hour: number, minute: number): Date {
  const [y, m, day] = ymd.split('-').map(Number);
  return new Date(y, m - 1, day, hour, minute, 0, 0); // local-time constructor
}
```

- **Only schedule fire times strictly in the future.** A `DATE` trigger whose `date` is in the past fires **immediately** — filter with `fireAt.getTime() > Date.now()` (done in `computeSchedule`).

See [testing.md](testing.md) for unit-testing `computeSchedule` with a fixed `now`, and [push-notifications-fcm.md](push-notifications-fcm.md) for server-sent remote push (dev build required).
