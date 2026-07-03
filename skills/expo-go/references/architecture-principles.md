Engineering principles for your first Expo + Firebase app — the small set of habits that keep an MVP correct and cheap to change as it grows. Distilled from production system-engineering practice and adapted to a mobile + Firestore stack.

Read this once before you build, and again when the app starts feeling messy. Each principle has a **why** and a **do this**. None of them cost time up front; all of them save it later.

## 1. Separate pure logic from I/O

Your app has two kinds of code: **decisions** (what the next value should be, whether something is valid, how to format) and **effects** (read Firestore, write Firestore, show a notification, navigate). Keep them apart.

- Put decisions in a pure module — plain functions, no `firebase`, no React, no `Date.now()` passed in from outside. A file like `src/core/` that takes inputs and returns outputs.
- Put effects in thin wrappers that call the pure functions.

**Why:** pure functions are trivially testable (no emulator, no mocks — see [testing.md](testing.md)) and reusable (the same logic can later run in a Cloud Function). Most real bugs hide in the decision logic; making it pure makes them visible.

**Do this:** if a function both computes *and* talks to the network, split it. `computeNextState(input)` (pure) + `saveNextState(db, id, computeNextState(input))` (effect).

## 2. Small, focused files — one responsibility each

A file should do one thing. When a file grows past ~150–200 lines or you struggle to name it, it's doing too much — split it.

**Why:** small files are easier for *you* to hold in your head and easier for an AI assistant to edit reliably. A screen that fetches, transforms, validates, and renders is four responsibilities in a trench coat.

**Do this:** data access in `src/firebase/db.ts`, auth state in `src/state/`, UI atoms in `src/ui/`, decisions in `src/core/`, routes in `app/`. See [project-setup.md](project-setup.md) for the layout.

## 3. DRY and YAGNI — in that order of restraint

- **DRY:** if the same fact lives in two places, they *will* drift. One formatter, one parser, one doc-mapping helper.
- **YAGNI:** don't build the thing you *might* need. Build the MVP. A demo does not need offline sync, a plugin system, or five auth providers.

**Why:** every abstraction you add is code someone must understand. Duplicated facts and speculative abstractions are the two most common ways an MVP rots.

**Do this — zero-ripple change:** when you must change a shared function that has many callers, add a *new* variant and let the old one delegate to it, instead of editing every call site. Small blast radius beats a big "clean" rewrite.

## 4. Additive and non-breaking by default

New capability should be **opt-in**. Don't change the meaning of an existing field, function signature, or document shape — add a new one and migrate readers on their own schedule.

**Why:** in a real app the screens, the database, and (later) teammates all depend on the current contract. Breaking it turns one change into ten.

**Do this:** adding pagination? Make "no limit passed" behave exactly as before. Adding a field? Give it a default and don't require existing docs to have it.

## 5. The trust boundary is the database rules, not the client

A route guard or a disabled button is **UX**, not security. Anyone can call Firestore directly with your public config (which ships in the app bundle). The Firestore **security rules** are the only real control.

**Why:** "the app never sends that request" is not a guarantee — the app is not the only client. If a rule lets a user set their own `role: "owner"`, they will.

**Do this:** enforce ownership, immutability, role, and linking/consent **in the rules** and test them against the emulator. Never trust a client-supplied field for a privilege decision. Full patterns in [firestore-data-rules.md](firestore-data-rules.md).

## 6. Model data for the reads — no scans, no N+1, bounded lists

How you store data is dictated by how you'll read it. Three rules that keep reads fast and cheap:

- **No unbounded scans in runtime.** Never fetch a whole collection and filter on the client at runtime. Always scope a query with a `where(...)` that matches how you read. (Firestore *bills per document read* — an unfiltered read of 10k docs costs 10k reads.)
- **Kill N+1 by denormalizing.** If rendering a list means "fetch the list, then fetch one more doc per row," you have an N+1. Copy the few fields you need onto the row **in the write path that owns them**, so one query paints the screen. One read beats one-plus-N.
- **Paginate lists that grow.** A list that's fine with 10 items throttles and costs O(n) at 10,000. Add a `limit` + cursor for any collection that grows without bound, and keep the un-paginated call working (principle 4).

**Why:** these are the three failure modes that turn a snappy demo into a slow, expensive app the moment real data arrives. Design for "what happens at thousands of rows?" from day one — it's a one-line question that catches most scale bugs.

**Do this:** for every read path, write one sentence: *"at 1,000s of rows, this…"*. If the answer is "reads the whole collection" or "does a fetch per item," fix it now — it's cheap while the code is small.

## 7. Design for eventual consistency and honest UI states

Firestore reads can lag a write. "Create a doc, then immediately query for it" may not return it yet. And every screen has more than a happy path.

**Why:** the two states juniors forget are **empty** and **error**, and the trap is treating "still loading" as "nothing there." A read failure that renders an empty list looks exactly like success with no data — and quietly lies to the user.

**Do this:** every data screen handles **loading / empty / error** explicitly. Never leave a user stuck on a spinner with no escape — if `user` exists but their profile failed to load, give them a sign-out/retry, don't trap them (see [auth-and-state.md](auth-and-state.md)). After a write, read back from the source of truth rather than assuming a query already reflects it.

## 8. Verify against the docs — don't fabricate APIs

When you're unsure of a command flag, a version, or a function signature, **look it up** in the current Expo/Firebase docs. Library APIs change between SDK versions; your memory (and an LLM's) lags reality.

**Why:** a fabricated flag or a wrong-version snippet costs more time to debug than it ever saved to guess. "I'm pretty sure" is not a source.

**Do this:** an SDK bump moves every `expo-*` package together — when you upgrade, re-verify. Pin the versions in [project-setup.md](project-setup.md) until you have a reason to move.

## 9. Test locally first — red-green, no cloud, no secrets

Prove logic works *before* you deploy anything. Pure functions get plain unit tests; security rules get tested against the **local emulator**. No network, no real project, no secrets.

**Why:** a local harness makes the whole build loop fast and safe. You catch the bug on your machine in seconds instead of in a shared environment in minutes.

**Do this:** write the test first for anything with real logic (a decision function, a rule). See [testing.md](testing.md). Keep a "does this invariant still hold?" test around the things that must never break.

## 10. Don't leak secrets; validate at the boundary

- `EXPO_PUBLIC_*` values are embedded in the app bundle — they are **public**, not secret. Never put a private key or admin credential there. Access control lives in the rules ([firestore-data-rules.md](firestore-data-rules.md)).
- Validate input where it enters your system — not just the *shape* (is it a number?) but the *domain* (is it in range?). A field your logic trusts must be one your rules and code actually constrain.
- Don't log personal data.

**Why:** the browser/app is an untrusted client and every input is adversarial-until-checked. These are the cheapest security habits and the ones most often skipped in a demo.

---

### The one-line version

Keep decisions pure and small, make the database rules your real security, model data for how you read it (no scans, no N+1, paginate growth), handle loading/empty/error honestly, verify APIs against docs, and test locally first. Do these on the MVP and the app stays cheap to change when it's no longer an MVP.
