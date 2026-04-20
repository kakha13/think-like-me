# Mobile rules

iOS, Android, React Native, and Expo patterns. Applies whenever the task involves a mobile app, a simulator/emulator, a dev client, or any of these toolchains.

---

### NativeWind `className` can crash components inside tab navigators
**When:** React Native + Expo Router / tab navigator + component rendered inside a FlatList or similar list in a tab
**Do:** Use `StyleSheet.create` with inline `style={...}` for any component that renders inside a tab-scoped list. Reserve NativeWind `className` for screens outside navigators.
**Don't:** Ship NativeWind `className` on components that live inside tab-navigator FlatLists.
**Why:** NativeWind's className inside tab-navigator FlatLists has reproduced "Couldn't find a navigation context" render errors on tab switch. Mechanism is unclear, but converting to `StyleSheet.create` has consistently resolved it.

---

### Expo prebuild is required when adding a config plugin
**When:** Expo project + modifying `plugins` array in `app.config.ts` / `app.json` (including plugins that mutate `Info.plist`, `AndroidManifest.xml`, or `build.gradle`)
**Do:** Run `npx expo prebuild --platform <ios|android> --clean` after the change, then `npx expo run:<platform>`. Verify the expected mutation landed (e.g. `grep` the generated `Info.plist` for the injected value) before spending time on a native build.
**Don't:** Assume `expo run:<platform>` alone regenerates native files — it does not. Prebuild is skipped if `ios/` / `android/` already exists unless `--clean` is passed.
**Why:** `expo run:*` installs and builds the existing native project. Plugin mods only run during `prebuild`. Without a fresh prebuild, the new plugin's Info.plist / manifest edits never land, and the app will fail silently at runtime for the feature that depended on them.
**Example:** Adding `@react-native-google-signin/google-signin` with `iosUrlScheme` to `app.config.ts` — `CFBundleURLSchemes` in `Info.plist` will not pick it up until `expo prebuild --platform ios --clean` runs.

---

### Adding a native module requires rebuilding the dev client
**When:** adding a React Native package with a native side (iOS or Android module) + running on an existing dev client / simulator / emulator
**Do:** Rebuild the dev client on every platform that will use the new module — locally via `npx expo run:android` / `npx expo run:ios`, or in the cloud via `eas build --profile development --platform <ios|android>`. Install the fresh binary before expecting the module to work.
**Don't:** Assume a Metro reload will pick up new native modules — it will not.
**Why:** Native modules are compiled into the dev-client APK/IPA. A stale dev client lacks the ViewManager/native methods, and Fabric throws `IllegalViewOperationException` ("Can't find ViewManager 'ViewManagerAdapter_<name>'") the moment JS tries to render the view.

---

### EAS `--local` builds on macOS need a non-symlinked working directory
**When:** running `eas build --local` on macOS
**Do:** Prefix the command with `EAS_LOCAL_BUILD_WORKINGDIR=$HOME/.eas-local-build` (or any path that isn't under `/var/folders`). Example: `EAS_LOCAL_BUILD_WORKINGDIR=$HOME/.eas-local-build eas build --local --profile production --platform ios`.
**Don't:** Accept the default `$TMPDIR` when `--local` fails with a Metro "Unable to resolve module expo-router/entry" error containing a `/var/folders/.../../private/var/folders/...` path.
**Why:** macOS's `/var` → `/private/var` symlink collides with Metro's path resolver: `expo/scripts/resolveAppEntry.js` returns one prefix, Metro uses the other, and `path.relative` produces a garbage cross-symlink path. Fallbacks: drop `--local` and use cloud builds, or set `ENTRY_FILE=node_modules/expo-router/entry.js` in `eas.json` build-profile `env`.

---

### npm installs fail on react/react-dom peer conflict — use `--legacy-peer-deps`
**When:** running `npm install <pkg>` or `npx expo install <pkg>` in a project with a pre-existing react / react-dom version skew
**Do:** Append `-- --legacy-peer-deps` to forward the flag through `expo install`: `npx expo install <pkg> -- --legacy-peer-deps`. For plain npm, just add `--legacy-peer-deps`.
**Don't:** Unprompted, bump or downgrade `react` / `react-dom` to "fix" the peer — it can ripple into other peers. Leave version alignment for a deliberate cleanup pass.
**Why:** A single version skew (e.g. `react@19.2.0` vs `react-dom@19.2.5`) makes every install fail with `ERESOLVE could not resolve react peer dependency`, even when the package being installed is unrelated to React.

---

### Google OAuth with `ResponseType.IdToken` requires `usePKCE: false` in expo-auth-session
**When:** `expo-auth-session` + Google OAuth + `ResponseType.IdToken` (implicit id_token flow)
**Do:** Pass `usePKCE: false` in the `useAuthRequest` config. If Google then requires `nonce`, pass it in `extraParams: { nonce: <random> }` and verify server-side. Preferred long-term: switch to `ResponseType.Code` + PKCE (the default) and exchange the code for an id_token on the backend — this is Google's recommended mobile flow.
**Don't:** Leave the PKCE default on when using `ResponseType.IdToken` — Google's endpoint rejects with `400 invalid_request — parameter not allowed for this message type`.
**Why:** expo-auth-session (v55+) defaults `usePKCE` to `true` and adds `code_challenge` + `code_challenge_method=S256` to every auth URL, regardless of `responseType`. Implicit flow does not accept PKCE params, so Google fails the request.

---

### `.env.local` silently overrides `.env` at Metro bundle time
**When:** diagnosing dev-build "connection refused" / "couldn't load" errors or debugging wrong API URLs in an Expo project
**Do:** Check `.env.local` before `.env` — Expo prints `env: load .env.local .env` on Metro startup in that precedence order. To fall through to `.env`, rename `.env.local` to `.env.local.bak` or delete it. After changing any env var, re-bundle (Metro restart), not just reload JS — `EXPO_PUBLIC_*` values are inlined at bundle time.
**Don't:** Edit `.env` and expect it to take effect when a `.env.local` exists.
**Why:** `.env.local` is loaded first and wins on conflict. Common local trap: it points `EXPO_PUBLIC_API_URL` at `localhost:8000` for a local backend server; when the server isn't running, every API call fails and React Query surfaces a generic error screen.
