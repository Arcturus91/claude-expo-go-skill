# Expo Go — React Native + Firebase MVP starter (Claude Code skill)

A Claude Code **skill** with the proven initial scaffolding for a **demo/MVP mobile app**: React Native
via **Expo Go**, **Firebase email/password auth**, and **Firestore** as the database — runnable on a real
phone with no native build, and shareable to a teammate via **EAS Update**.

Built for engineers starting their first mobile app. It's a `SKILL.md` router plus focused reference
files, each distilled from a real working app and genericized for reuse, with an
`architecture-principles.md` that carries the system-engineering habits worth keeping from day one.

## Verified stack

Expo SDK 54 · React Native 0.81 · React 19.1 · expo-router 6 · firebase JS SDK v12 · expo-updates ~29.
Firebase is used via its **pure-JS** SDK, so the app runs in Expo Go with no native build.

## Install

```text
/plugin marketplace add Arcturus91/claude-expo-go-skill
/plugin install expo-go@expo-go-skill
```

Claude then picks up the skill automatically when you work on an Expo / React Native app.

## Contents

Follow them in this order for a new app:

| # | Reference | Covers |
|---|---|---|
| 1 | `architecture-principles.md` | The habits that keep an MVP correct and cheap to change |
| 2 | `project-setup.md` | create-expo-app, TypeScript, folder layout, dependency install |
| 3 | `firebase-setup.md` | firebase v12 init in RN, auth persistence, emulator wiring |
| 4 | `auth-and-state.md` | auth context, session persistence, role-based routing |
| 5 | `navigation-expo-router.md` | file routes, layout groups, route guards, params |
| 6 | `firestore-data-rules.md` | data-access patterns + secure, query-safe security rules |
| 7 | `testing.md` | pure unit tests + Firestore-rules tests against the emulator |
| 8 | `expo-go-workflow.md` | run on a real phone via QR, share a build with a teammate (EAS Update) |
| 9 | `troubleshooting.md` | the setup traps that cost the most time |

## License

MIT — see [LICENSE](LICENSE).
