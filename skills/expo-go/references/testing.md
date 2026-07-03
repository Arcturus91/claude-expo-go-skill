# Testing an Expo Go app

Test what matters without a device: pure logic in plain unit tests, and Firestore rules against the local emulator.

## Philosophy

Local-first. Two kinds of tests, both with **no deploy, no secrets, no network to real services**:

1. **Pure unit tests** — your domain logic. Keep decisions in a pure layer (`src/engine` or
   `src/core`) that takes plain inputs and returns plain outputs, no expo/React/Firebase imports.
   That layer is trivially testable and never needs a device. (See
   [architecture-principles.md](architecture-principles.md) §1 — separate pure decisions from I/O.)
2. **Rules integration tests** — your Firestore security rules, run against the **local emulator**
   with a `demo-*` project so no real credentials are involved.

If a function is hard to test, it's usually because effects and decisions are tangled — pull the
decision into the pure layer.

## Unit tests with ts-jest (transpile-only)

`jest ~29` + `ts-jest ~29`. Jest runs `.ts` directly through ts-jest using a dedicated
`tsconfig.jest.json`:

```js
// jest.config.js
/** @type {import('jest').Config} */
module.exports = {
  testEnvironment: "node",
  testMatch: ["**/tests/**/*.test.ts"],
  transform: { "^.+\\.ts$": ["ts-jest", { tsconfig: "tsconfig.jest.json" }] },
};
```

```jsonc
// tsconfig.jest.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "module": "node16",
    "moduleResolution": "node16",
    "types": ["jest", "node"],
    "isolatedModules": true,
    "rootDir": "."
  }
}
```

**Gotcha:** with `isolatedModules: true`, ts-jest is **transpile-only — it does NOT type-check**. A
green test suite says nothing about types. Run `tsc --noEmit` **separately** (wire it as a
`typecheck` script and in CI). Relatedly, Expo's base tsconfig doesn't pull in node/jest globals, so
your main `tsconfig.json` must add them or `tsc` will flag `test`/`expect`/`process`:

```jsonc
// tsconfig.json
{ "extends": "expo/tsconfig.base",
  "compilerOptions": { "strict": true, "types": ["node", "jest", "react"] } }
```

A pure test needs no mocks — call the function and assert:

```ts
// tests/schedule.test.ts
import { computeSchedule, REMIND_HOURS, WINDOW_DAYS } from "../src/notifications/schedule";
const at = (y: number, mo: number, d: number, h = 0) => new Date(y, mo - 1, d, h).getTime();

test("disabled → empty", () => {
  expect(computeSchedule(at(2026, 7, 2, 10), { enabled: false }, [])).toEqual([]);
});

test("no items, morning → a full window of nudges, all in the future", () => {
  const out = computeSchedule(at(2026, 7, 2, 10), { enabled: true }, []);
  const nudges = out.filter((n) => n.data.type === "nudge");
  expect(nudges).toHaveLength(REMIND_HOURS.length * WINDOW_DAYS);
  expect(nudges.every((n) => n.fireDate > at(2026, 7, 2, 10))).toBe(true);
  expect(out.length).toBeLessThan(64); // stay under the iOS pending cap
});
```

## Firestore rules integration tests

Use `@firebase/rules-unit-testing ^5` against the running emulator. `initializeTestEnvironment`
loads your `firestore.rules`; `authenticatedContext(uid)` / `unauthenticatedContext()` give you a
client SDK `Firestore` that the emulator evaluates **through the rules**. Assert allowed vs denied
with `assertSucceeds` / `assertFails`. Test the two things rules exist for: **ownership/immutability**
and **query safety** (see [firestore-data-rules.md](firestore-data-rules.md)).

```ts
// tests/rules.test.ts
import { assertFails, assertSucceeds, initializeTestEnvironment,
         RulesTestEnvironment } from "@firebase/rules-unit-testing";
import { readFileSync } from "fs";
import { collection, doc, getDoc, getDocs, query, setDoc, updateDoc, where } from "firebase/firestore";

let env: RulesTestEnvironment;
beforeAll(async () => {
  env = await initializeTestEnvironment({
    projectId: "demo-myapp",
    firestore: { rules: readFileSync("firestore.rules", "utf8"), host: "127.0.0.1", port: 8080 },
  });
});
afterAll(() => env.cleanup());
beforeEach(() => env.clearFirestore());

test("owner reads own item; a stranger cannot", async () => {
  const owner = env.authenticatedContext("alice").firestore();
  const stranger = env.authenticatedContext("mallory").firestore();

  await assertSucceeds(setDoc(doc(owner, "items/i1"), { ownerId: "alice", title: "Buy milk" }));
  await assertSucceeds(getDoc(doc(owner, "items/i1")));
  await assertFails(getDoc(doc(stranger, "items/i1")));
});

test("ownerId is immutable; strangers can't scan the collection", async () => {
  const owner = env.authenticatedContext("alice").firestore();
  const stranger = env.authenticatedContext("mallory").firestore();
  await setDoc(doc(owner, "items/i1"), { ownerId: "alice", title: "Buy milk" });

  await assertFails(updateDoc(doc(owner, "items/i1"), { ownerId: "mallory" }));       // re-parenting
  await assertFails(getDocs(collection(stranger, "items")));                          // unfiltered list
  await assertSucceeds(getDocs(query(collection(owner, "items"),
    where("ownerId", "==", "alice"))));                                              // owner-scoped query
});
```

**Gotcha:** rules run only for the client SDK. To seed data a rule would forbid, wrap the setup in
`env.withSecurityRulesDisabled(async (ctx) => { /* ctx.firestore() bypasses rules */ })` — don't
loosen the rules to make seeding work.

## Running the emulator

Use a `demo-<name>` project id: the emulator treats `demo-*` as offline and needs **no real
credentials**. Run the suite two ways — a long-lived emulator for local iteration (`emu`), and a
one-shot `emulators:exec` that boots the emulator, runs the tests, and tears it down (CI / `test:e2e`):

```jsonc
// package.json → scripts
{
  "typecheck": "tsc --noEmit",
  "test":     "jest tests/schedule.test.ts",
  "test:int": "jest tests/rules.test.ts",
  "emu":      "npx firebase emulators:start --project demo-myapp",
  "test:e2e": "PATH=\"/opt/homebrew/opt/openjdk/bin:$PATH\" npx firebase emulators:exec --project demo-myapp \"npm run test:int\""
}
```

**Gotcha:** the **Firestore** emulator requires a **Java runtime** — without a JDK it won't start.
Install one (`brew install openjdk`); Homebrew's OpenJDK is **keg-only**, so its binary lives at
`/opt/homebrew/opt/openjdk/bin` and isn't on `PATH` by default. Prepend it inline in the script (as
above) rather than relying on a global shell change, so CI and teammates get the same result. (The
**Auth** emulator is pure Node and needs no Java.) Point the client at the emulator host/port you
passed to `initializeTestEnvironment` (`127.0.0.1:8080` above).

See [architecture-principles.md](architecture-principles.md) for why keeping decisions in a pure layer is what makes them this easy to test.
