---
name: expo-go
description: Use when starting a demo or MVP mobile app with Expo Go — a React Native app with Firebase email/password auth and Firestore as the database, runnable on a physical phone with no native build and shareable via EAS Update. Covers project scaffolding, expo-router navigation, Firebase setup, secure Firestore rules, session/role state, local testing, and the setup traps. Triggers: Expo, Expo Go, expo-router, EAS Update, "start a mobile app", MVP, React Native + Firebase, Firebase auth, Firestore, "run on my phone".
---

# Start a demo/MVP mobile app in Expo Go (React Native + Firebase)

The proven initial scaffolding for a mobile app you can build fast, run on your own phone, and hand to a teammate — **React Native via Expo Go, Firebase email/password auth, and Firestore as the database**. Stack-agnostic in domain; opinionated in setup, because these are the choices that reliably work.

This file is a **router**: read the reference that matches your task. New to all of this? Follow the reading path below in order.

## Verified stack

- **Expo SDK 54** (`expo ~54.0.x`), **React Native 0.81**, **React 19.1**
- **expo-router 6** — file-based routing
- **firebase JS SDK v12** — email/password auth + Firestore, pure JS (no native modules → runs in Expo Go)
- **expo-updates ~29** (EAS Update, for sharing)

Pin to these until you have a reason to move. An SDK bump moves every `expo-*` version together — when you upgrade, re-verify commands and APIs against current Expo/Firebase docs, don't trust memory.

## Why Expo Go for an MVP

Expo Go runs your JS on a real device with **no native build step**, as long as every library is in the Expo Go runtime. `firebase` (pure JS) qualifies; `@react-native-firebase` (native) does not. That's the whole point for a demo: you write JS, scan a QR, and it's on your phone — and with EAS Update a teammate opens it in *their* Expo Go from a link. The first native-only module you add ends that and pushes you to a dev client. See [expo-go-workflow.md](references/expo-go-workflow.md).

## Reading path — building your first app

1. **[architecture-principles.md](references/architecture-principles.md)** — the handful of habits that keep an MVP correct and cheap to change. Read this first.
2. **[project-setup.md](references/project-setup.md)** — scaffold the project, folder layout, dependencies.
3. **[firebase-setup.md](references/firebase-setup.md)** — initialize Firebase auth + Firestore in React Native.
4. **[auth-and-state.md](references/auth-and-state.md)** — sign-in/up, session persistence, role-based routing.
5. **[navigation-expo-router.md](references/navigation-expo-router.md)** — screens, tabs/stacks, route guards, params.
6. **[firestore-data-rules.md](references/firestore-data-rules.md)** — model your data and write secure, query-safe rules.
7. **[testing.md](references/testing.md)** — prove logic and rules locally (no cloud, no secrets).
8. **[expo-go-workflow.md](references/expo-go-workflow.md)** — run on your phone and share the build with your team.
9. **[troubleshooting.md](references/troubleshooting.md)** — the setup traps, when you hit one.

## Route by task

| I want to… | Read |
|---|---|
| Learn the principles that keep the app maintainable | [architecture-principles.md](references/architecture-principles.md) |
| Scaffold a project, folder layout, install deps | [project-setup.md](references/project-setup.md) |
| Initialize Firebase (auth + Firestore) in RN | [firebase-setup.md](references/firebase-setup.md) |
| Auth context, role-based routing, session state | [auth-and-state.md](references/auth-and-state.md) |
| Add screens, tabs, stacks, route guards, params | [navigation-expo-router.md](references/navigation-expo-router.md) |
| Model data + write secure, query-safe Firestore rules | [firestore-data-rules.md](references/firestore-data-rules.md) |
| Unit + Firestore-rules tests (local, no cloud) | [testing.md](references/testing.md) |
| Run on my phone via QR, share a build with a teammate | [expo-go-workflow.md](references/expo-go-workflow.md) |
| Something broke during setup | [troubleshooting.md](references/troubleshooting.md) |

## Principles baked into these references

- **Expo-Go-compatible by default** — pure-JS libraries only; confirm a lib runs in Expo Go before adding it.
- **Trust boundaries live in Firestore rules, not the client** — a screen guard is UX; the rule is the actual control. See [firestore-data-rules.md](references/firestore-data-rules.md).
- **Local-first testing** — pure logic in plain unit tests; rules against the local emulator. No cloud, no secrets. See [testing.md](references/testing.md).
- **Model data for how you read it** — no unbounded scans, kill N+1 by denormalizing, paginate lists that grow. See [architecture-principles.md](references/architecture-principles.md).
- **Don't fabricate library APIs** — when unsure of a flag, version, or signature, check current Expo/Firebase docs.
