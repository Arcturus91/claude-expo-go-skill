# Firestore data-access patterns + query-safe security rules

Write db access as pure functions over an injected `Firestore`, denormalize an access key so cross-user reads pass the rules, and enforce every trust boundary in the rules — never the client.

Neutral model used throughout: `users/{uid}` with `role: "owner" | "member"`; `items/{itemId}` created by a member and denormalized with an `ownerId` so the linked owner can read them; `inviteCodes/{code}` an owner hands out to link a member. `me()` / the signed-in uid come from Auth — see [auth-and-state.md](auth-and-state.md).

## Data access as pure functions over an injected `Firestore`

Every function takes `db: Firestore` as its first argument. The app passes the singleton from your Firebase config; tests pass an emulator-backed instance (see [testing.md](testing.md)). No module-level `db` means no hidden global and trivially swappable test wiring.

```ts
import {
  addDoc, collection, doc, Firestore, getDoc, getDocs,
  query, QuerySnapshot, setDoc, where,
} from "firebase/firestore";

// Fold the doc id into the data in ONE place instead of repeating `{ id: d.id, ... }`.
const mapDocs = <T>(snap: QuerySnapshot): (T & { id: string })[] =>
  snap.docs.map((d) => ({ id: d.id, ...(d.data() as T) }));

// get one
export async function getItem(db: Firestore, itemId: string): Promise<Item | null> {
  const snap = await getDoc(doc(db, "items", itemId));
  return snap.exists() ? { id: snap.id, ...(snap.data() as Omit<Item, "id">) } : null;
}

// list a member's own items (single equality filter)
export async function itemsForMember(db: Firestore, memberId: string): Promise<Item[]> {
  const snap = await getDocs(query(collection(db, "items"), where("memberId", "==", memberId)));
  return mapDocs<Omit<Item, "id">>(snap) as Item[];
}

// create — addDoc mints the id; setDoc(doc(...)) when YOU own the id (e.g. users/{uid})
export async function createItem(db: Firestore, item: Omit<Item, "id">): Promise<string> {
  return (await addDoc(collection(db, "items"), item)).id;
}
```

## THE query-safety gotcha (headline)

**Gotcha:** a Firestore query is *rejected outright* unless its `where` filters, on their own, guarantee that every matched document satisfies the read rule. Rules do **not** filter your result set — you cannot "query broadly and let the rules drop what I'm not allowed to see." If a single returned doc could violate the rule, the whole query fails.

So to let an owner read all items belonging to their linked members, you must **(a) denormalize** the access key (`ownerId`) onto every item in the member's write-path, and **(b) query by a filter that matches a read-rule branch**, then narrow in memory.

```ts
// Read rule (see below): allow read if resource.data.memberId == me() || resource.data.ownerId == me()

// WRONG — rejected. No filter proves every matched doc has ownerId == me():
getDocs(collection(db, "items"));                                  // ❌ PERMISSION_DENIED
getDocs(query(collection(db, "items"), where("memberId","==",x))); // ❌ can't prove owner branch

// RIGHT — the filter matches the `ownerId == me()` rule branch, so every hit is provably allowed:
export async function itemsForOwnerView(db: Firestore, ownerId: string, memberId: string) {
  const snap = await getDocs(query(collection(db, "items"), where("ownerId", "==", ownerId)));
  return mapDocs<Omit<Item, "id">>(snap)
    .filter((it) => it.memberId === memberId)   // narrow client-side
    .sort((a, b) => a.createdAt - b.createdAt);
}
```

## Avoid composite-index churn

A **single equality filter + client-side sort/filter** (as above) needs no composite index. The moment you combine a `where` with an `orderBy` on a different field, or two range/inequality filters, Firestore demands a composite index you must define and deploy. For small per-user datasets, pulling one owner's rows and sorting in memory is simpler and index-free. **Tradeoff:** this reads the whole matched set — fine for small N; as data grows, add the real composite indexes and switch to `orderBy` + `limit` cursor pagination so you don't fetch everything.

## Writing the rules

`rules_version = '2'`, tiny helpers, per-collection ownership, immutable keys on update, and `get` without `list` on lookup-by-id collections so they can't be enumerated.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function signedIn() { return request.auth != null; }
    function me()       { return request.auth.uid; }

    match /users/{userId} {
      allow get:    if signedIn() && (userId == me() || resource.data.ownerId == me());
      allow list:   if false;                                   // never enumerate the user graph
      allow create: if signedIn() && userId == me();
      allow update: if signedIn() && userId == me()
        && request.resource.data.role == resource.data.role;    // role is immutable
    }

    match /items/{itemId} {
      allow read:   if signedIn() && (resource.data.memberId == me() || resource.data.ownerId == me());
      allow create: if signedIn() && request.resource.data.memberId == me()
        && request.resource.data.ownerId ==                     // stamp caller's REAL owner (trap 3)
             get(/databases/$(database)/documents/users/$(me())).data.ownerId;
      allow update: if signedIn() && resource.data.memberId == me()
        && request.resource.data.memberId == resource.data.memberId   // can't re-parent
        && request.resource.data.ownerId  == resource.data.ownerId;   // can't re-point
    }

    match /inviteCodes/{code} {
      allow get:    if signedIn();                               // look up by exact id…
      allow list:   if false;                                   // …but never enumerate the graph
      allow create: if signedIn() && request.resource.data.ownerId == me();
      allow update: if signedIn()
        && resource.data.used == false && request.resource.data.used == true
        && request.resource.data.memberId == me()
        && request.resource.data.ownerId == resource.data.ownerId;
    }
  }
}
```

**Gotcha:** in `rules_version = '2'`, `allow read` covers *both* `get` and `list`. Splitting them (`allow get` + `allow list: if false`) is what stops someone reading one code by id yet blocks dumping the whole `inviteCodes` collection to reconstruct the owner↔member graph.

## Enforce trust boundaries in RULES, not the client (critical lesson)

**Gotcha:** a screen guard or hidden button is UX only — anyone can call the SDK directly with their token. The rule is the only real control. Three traps:

1. **A "set-once from null" link that accepts any first value.** `allow update: if ownerId was null || unchanged` lets a member `updateDoc(users/me, { ownerId: "someStranger" })` and self-link to *anyone* with no consent step. **Fix:** validate the link against a real invite via `get(...)` — the new `ownerId` must be the owner of a code that names this member:
   ```
   allow update: if userId == me() && request.resource.data.role == resource.data.role
     && (request.resource.data.get('ownerId', null) == resource.data.get('ownerId', null)   // unchanged, or…
         || (resource.data.get('ownerId', null) == null                                     // first link only
             && get(/databases/$(database)/documents/inviteCodes/$(request.resource.data.linkCode)).data.ownerId == request.resource.data.ownerId
             && get(/databases/$(database)/documents/inviteCodes/$(request.resource.data.linkCode)).data.memberId == me()));
   ```
2. **`role` is only trustworthy if a rule validates it on create AND some rule actually reads it.** If no rule ever checks `resource.data.role`, a client can write `role: "owner"` to its own profile and self-grant privilege. Never gate access on a client-set field unless a rule enforces it — prefer deriving privilege from ownership relationships you control.
3. **A denormalized `ownerId` on a child doc must be pinned to the writer's actual owner on create** — `request.resource.data.ownerId == get(/databases/$(database)/documents/users/$(me())).data.ownerId` (see the `items` create rule). Without it a member can stamp a stranger's id onto their items and pollute that stranger's owner-view.

Also **validate the shape and range of any field your logic trusts** — e.g. `request.resource.data.score is int && request.resource.data.score >= 0 && request.resource.data.score <= 10`. Use `resource.data.get('field', default)` to type-check optional fields without erroring when absent.

## Batched writes & atomicity

Use `writeBatch` when a multi-doc invariant must hold — e.g. redeeming a code marks it used *and* links the member in one shot:

```ts
const batch = writeBatch(db);
batch.update(doc(db, "inviteCodes", code), { used: true, memberId, linkCode: code });
batch.update(doc(db, "users", memberId), { ownerId, linkCode: code });
await batch.commit();   // all-or-nothing
```

**Gotcha:** rules are evaluated **per write against committed state**, and a batch commits atomically — if *any* single write is denied, the entire batch fails and nothing lands. That is exactly what makes the redeem race safe: two members racing on the same code both try `used:false → true`, but only the first commit sees `used == false`, so the second is denied and its whole batch (including the link) rolls back.

## Testing rules locally

Exercise these exact db functions as authenticated owner/member/stranger against the emulator with `@firebase/rules-unit-testing` (`assertFails` for denials) — see [testing.md](testing.md).
