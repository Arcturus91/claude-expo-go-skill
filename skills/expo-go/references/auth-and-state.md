# Auth and session state

Wrap Firebase Auth in thin helpers and one React context that exposes the current user, their Firestore profile, and a loading flag — then route on `{ user, profile, loading }`.

Assumes the `auth` / `db` singletons from [firebase-setup.md](firebase-setup.md). The profile doc is `{ uid, role, displayName }` with `role: "owner" | "member"`.

## Thin auth helpers

Keep auth calls in one small module (`src/firebase/auth.ts`). `signUp` creates the account **and** writes the profile doc; `signIn` / `signOut` are one-liners:

```ts
import {
  createUserWithEmailAndPassword,
  signInWithEmailAndPassword,
  signOut,
} from "firebase/auth";
import { auth, db } from "./config";
import { createUserProfile } from "./db"; // setDoc(doc(db,"users",uid), {...})

export async function signUp(email: string, password: string, role: Role, displayName: string) {
  const cred = await createUserWithEmailAndPassword(auth, email.trim(), password);
  await createUserProfile(db, cred.user.uid, { role, displayName: displayName.trim() });
}

export const signIn = (email: string, password: string) =>
  signInWithEmailAndPassword(auth, email.trim(), password);

export const signOutUser = () => signOut(auth);
```

**Gotcha:** `createUserWithEmailAndPassword` signs the user in and persists the session **immediately**, before the `createUserProfile` write runs (or if that write throws). So "authed but no profile" is a **real, reachable state**, not a transient one — the auth listener can fire in that window, and a failed/interrupted write can leave it permanent. Design your routing for it (see below).

## Auth context provider

One context holds session state for the whole tree. Subscribe to `onAuthStateChanged`, load the profile after auth resolves, and expose `refresh()` to re-read the profile after signup or account linking:

```tsx
interface AuthState {
  user: User | null;
  profile: UserProfile | null;
  loading: boolean;
  refresh: () => Promise<void>;
}
const AuthContext = createContext<AuthState>({
  user: null, profile: null, loading: true, refresh: async () => {},
});
export const useAuth = () => useContext(AuthContext);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [profile, setProfile] = useState<UserProfile | null>(null);
  const [loading, setLoading] = useState(true);
  const seq = useRef(0);

  // Latest-wins: only the most recent read may call setProfile (see Gotcha).
  const loadProfile = useCallback(async (u: User | null) => {
    const token = ++seq.current;
    const p = u ? await getUserProfile(db, u.uid) : null;
    if (token === seq.current) setProfile(p);
  }, []);

  useEffect(() => {
    return onAuthStateChanged(auth, async (u) => {
      setUser(u);
      await loadProfile(u);
      setLoading(false);
    });
  }, [loadProfile]);

  const refresh = useCallback(() => loadProfile(auth.currentUser), [loadProfile]);
  const value = useMemo(
    () => ({ user, profile, loading, refresh }),
    [user, profile, loading, refresh],
  );
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```

`onAuthStateChanged` returns an **unsubscribe** function — return it straight from `useEffect` so the listener is torn down on unmount.

## Latest-wins guard

**Gotcha:** profile reads are async and can **race**. Right after signup the auth listener may fire and read the profile *before* the profile doc exists (returning `null`), while a later `refresh()` reads the real doc. If the responses land out of order, the stale `null` can clobber the good profile. The `seq` ref fixes this: each read captures a monotonically increasing token, and only the read whose token still equals `seq.current` is allowed to call `setProfile`. Any read that has since been superseded silently drops its result — the newest read always wins.

## Routing on auth + role

An index screen picks the destination from `{ user, profile, loading }`. The naive version below has a bug:

```tsx
export default function Index() {
  const { user, profile, loading } = useAuth();
  if (loading) return <Splash />;
  if (!user) return <Redirect href="/sign-in" />;
  if (!profile) return <Splash />;          // BUG: can spin forever
  return <Redirect href={profile.role === "owner" ? "/owner" : "/home"} />;
}
```

**Gotcha (important):** once `loading` is `false`, `user` set, and `profile` still `null`, the profile write failed or the doc was deleted — and `<Splash />` here traps the user on an **infinite spinner**. The account is bricked until they reinstall, because they can't even reach a sign-out button. Give them a visible escape instead:

```tsx
  if (!profile) {
    return (
      <View style={{ flex: 1, alignItems: "center", justifyContent: "center", gap: 12 }}>
        <Text>We couldn't load your profile.</Text>
        <Button title="Retry" onPress={refresh} />
        <Button title="Sign out" onPress={signOutUser} />
      </View>
    );
  }
```

(Pull `refresh` from `useAuth()`.) Distinguish "still loading" from "settled but empty" by the `loading` flag — never by `profile === null` alone. For nested route groups and guards, see [navigation-expo-router.md](navigation-expo-router.md).

## Performance note

Wrap the context `value` in `useMemo` and `refresh`/`loadProfile` in `useCallback` (as above). The provider re-renders on every auth or profile change; without memoization it hands consumers a brand-new object each render, re-rendering **every** `useAuth()` caller even when their slice of state didn't change.
