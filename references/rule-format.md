# Rule format

Rules live in `rules/<category>.md`. Each rule is a Markdown H3 heading followed by labeled fields.

## Required fields

- `**When:**` — context filter. Domain + feature keywords, separated by `+`. Matching during task triage uses keyword overlap with the incoming prompt.
- `**Do:**` — the guidance. One paragraph. Terse.
- `**Why:**` — the underlying mechanism that makes the wrong approach fail. Required because it lets the model judge edge cases the rule doesn't literally cover.

## Optional fields

- `**Don't:**` — explicit anti-pattern. Include when the wrong thing is tempting or was literally what was just corrected.
- `**Example:**` — short code snippet or concrete scenario. Include when `Do` would be ambiguous without one.

## Canonical minimal rule

```
### Mobile audio timers
**When:** mobile app + audio playback + timer/schedule/sleep-timer
**Do:** Enable iOS Background Modes (audio) + configure AVAudioSession; use Android foreground service. Schedule the stop via native side, not JS setTimeout.
**Why:** JS timers pause when the app backgrounds. Native audio keeps playing, but no JS fires to stop it — so the timer silently fails.
```

## Canonical full rule

```
### Expo prebuild overwrites ios/android folders
**When:** Expo project + editing native iOS/Android files directly (ios/, android/, Info.plist, AndroidManifest.xml)
**Do:** Make changes via `app.json` + config plugins. Run `expo prebuild --clean` to regenerate.
**Don't:** Edit files inside `ios/` or `android/` directly, even when it seems faster.
**Why:** Prebuild regenerates these folders from `app.json`. Hand edits survive until the next `expo prebuild` or `eas build`, then vanish silently.
**Example:** `npx expo install expo-camera` → add camera permission to `app.json` under `plugins`, not `Info.plist`.
```

## Editing notes

- Rules in the same category file are unordered — append new ones at the bottom of their category.
- Keep each rule under ~200 words. If it wants to grow longer, split into multiple tightly-scoped rules.
- Category files stay readable in full. When a category exceeds ~50 rules, propose splitting into sub-files (e.g. `mobile-ios.md` + `mobile-android.md`) during the next capture.
