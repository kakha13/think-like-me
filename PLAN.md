# Think Like Me Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a personal, global Claude Code skill at `~/.claude/skills/think-like-me/` that actively consults curated engineering rules before drafting code, captures new rules via three flows (`/learn-this`, `/new-rule`, auto-detect), and integrates with the existing auto-memory system.

**Architecture:** Markdown-only skill. `SKILL.md` is a thin router that triggers broadly on technical tasks and directs to `rules/<category>.md` for rule application or `agents/<capture-path>.md` for rule drafting. Rules are structured Markdown blocks (`When` / `Do` / `Why` / optional `Don't` / `Example`). Bootstrap seeds rule files from the user's existing cross-project feedback memories.

**Tech Stack:** Markdown + YAML frontmatter. No code, no build step. All files under `~/.claude/skills/think-like-me/`.

**Source spec:** `~/.claude/skills/think-like-me/DESIGN.md`

**Note on commits:** `~/.claude/skills/` is NOT a git repository. Skip `git` steps — verification is a read-back check per task.

---

## File Structure

After implementation the skill directory will contain:

```
~/.claude/skills/think-like-me/
├── SKILL.md                       # Entry point (T2)
├── DESIGN.md                      # Exists — written during brainstorming
├── PLAN.md                        # This file
├── references/
│   └── rule-format.md             # Canonical rule format spec (T1)
├── agents/
│   ├── learn-this.md              # Reactive capture flow (T3)
│   └── new-rule.md                # Proactive capture flow (T4)
└── rules/
    ├── mobile.md                  # 7 seeded rules (T5)
    ├── backend.md                 # 2 seeded rules (T6)
    ├── ui.md                      # 1 seeded rule (T7)
    └── general.md                 # Empty skeleton (T8)
```

Responsibilities per file are specified in `DESIGN.md` § "File layout".

---

## Tasks

### Task 1: Scaffold directory + rule-format reference

**Files:**
- Create: `~/.claude/skills/think-like-me/references/rule-format.md`

- [ ] **Step 1: Write `references/rule-format.md`**

Write the file with this exact content:

````markdown
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
````

- [ ] **Step 2: Verify file exists and is readable**

Run: `ls -la ~/.claude/skills/think-like-me/references/rule-format.md`
Expected: file listed, non-zero size.

Run: `head -3 ~/.claude/skills/think-like-me/references/rule-format.md`
Expected: first line is `# Rule format`.

---

### Task 2: Write `SKILL.md`

**Files:**
- Create: `~/.claude/skills/think-like-me/SKILL.md`

- [ ] **Step 1: Write `SKILL.md`**

Write the file with this exact content:

````markdown
---
name: think-like-me
description: Personal engineering rules and reasoning patterns the user has curated over time. Consult BEFORE writing, modifying, debugging, or designing any technical feature — rules often cover domain-specific gotchas (e.g., mobile audio timers need background mode) that generic answers miss. Also triggers on `/learn-this`, "learn this", "remember this", or `/new-rule` to capture new rules, and on detected corrections to previously proposed code.
---

# Think Like Me

You are consulting the user's personal, curated engineering rules. These are domain-specific patterns and principles the user has accumulated — often capturing non-obvious gotchas (e.g. "mobile audio timers need background mode") that a generic answer would miss.

The rules are the **active reasoning layer** of the user's Claude Code setup. They sit alongside the existing passive auto-memory: memory surfaces entries when relevant, rules actively shape how you approach the task before you draft anything.

## When to consult rules

On ANY request that involves writing, modifying, debugging, or designing a technical feature, follow this flow BEFORE you draft:

1. **Identify the domain(s) of the task.** From task keywords and context, pick one or more of: `mobile`, `backend`, `ui`, `general`. Multi-domain tasks (e.g. "wire Paddle checkout into mobile app") apply to multiple categories.
2. **Read the matching `rules/<domain>.md` file(s) in full.** These files are user-curated and expected to stay readable.
3. **Scan each rule's `When:` line.** A rule applies when its keywords overlap with the task context.
4. **Apply matching rules.**
   - `Do:` shapes your approach — build around it, don't tack it on.
   - `Don't:` flags paths you'd otherwise reach for.
   - `Why:` informs edge-case decisions when a rule isn't a perfect fit.
5. **Acknowledge briefly when a rule applies.** One-line confirmation before code: "Applying rule *Mobile audio timers* — enabling Background Modes + AVAudioSession." Lets the user catch a misapplied rule early. If zero rules match, skip this line — silence is fine.
6. **If zero rules match, proceed normally.** No rule is better than a forced rule.

## Capturing a new rule

Three paths add a rule to the skill. All end in the same dual-write: rule block in `rules/<category>.md` **and** memory entry in `~/.claude/projects/<current-project>/memory/`.

### Path A — `/learn-this` (reactive, from correction)

User just corrected you and wants to bake the correction into a rule. Load `agents/learn-this.md` and follow it.

### Path B — `/new-rule` (proactive, user-described)

User wants to add a rule from scratch, not tied to a correction. Invocation may include the rule text (e.g. `/new-rule for all Expo projects, never edit ios/ folder directly because prebuild will overwrite it`) or be bare. Load `agents/new-rule.md` and follow it.

### Path C — Auto-detect correction (reactive, background)

Detect when the user pushes back on your prior response: "no, don't do X", "actually use Y", "that's wrong because Z", or rejecting code with a different approach. When you see this, offer ONCE in one line:

> Caught — want me to save this as a rule? (`yes` to draft, ignore to drop)

On `yes` → switch to Path A (load `agents/learn-this.md`). On silence, decline, or an unrelated follow-up → drop it. Do not pressure twice in the same exchange.

## Saving a rule (dual-write, used by all three paths after user approves the draft)

1. **Rule file.** Append the rule block to `~/.claude/skills/think-like-me/rules/<category>.md`. Create the file if the category doesn't exist yet.
2. **Memory entry.** Write `~/.claude/projects/<current-project>/memory/feedback_<slug>.md` with frontmatter:

   ```
   ---
   name: <rule title>
   description: <one-line hook matching the rule's gist>
   type: feedback
   ---
   <2–3 line compact summary>

   Full rule: `~/.claude/skills/think-like-me/rules/<category>.md` → *<rule title>*
   ```
3. **MEMORY.md pointer.** Append one line to `~/.claude/projects/<current-project>/memory/MEMORY.md`:

   ```
   - [<rule title>](feedback_<slug>.md) — <one-line hook>
   ```

**Cross-project rules.** If the user marks a rule as cross-project during the draft-review gate, still write the memory entry to the current project's memory (one concrete landing place). The rule file is global — no per-project mirroring needed. When the skill is first invoked in a different project, offer to mirror the memory entry there lazily.

## Rule format

See `references/rule-format.md`. Minimum fields: title (H3), `When`, `Do`, `Why`. Optional: `Don't`, `Example`.

## Relationship to auto-memory

- Memory stays the passive recall layer: project facts, references, per-project corrections live there.
- Rules are the active reasoning layer: cross-project technical patterns live in `rules/*.md`.
- When a memory entry and a rule cover the same topic, the rule is authoritative — it's the structured, curated version.

## Guardrails

- **Rule conflicts.** If two matching rules give contradictory advice, flag the conflict in your acknowledgment line and ask the user which applies before proceeding.
- **Stale rules.** If the user corrects against a rule you just applied, treat it as a strong signal to propose updating or removing that rule, not just adding a new one.
- **Approval gate is non-negotiable.** Never save a rule without explicit user approval on the draft — even in auto-detect mode.
````

- [ ] **Step 2: Verify frontmatter parses and body structure is complete**

Run: `head -5 ~/.claude/skills/think-like-me/SKILL.md`
Expected: first line `---`, `name: think-like-me` on line 2, `description:` on line 3 with the full description, closing `---` on line 4.

Run: `grep -c "^## " ~/.claude/skills/think-like-me/SKILL.md`
Expected: `6` (top-level H2 sections: When to consult rules, Capturing a new rule, Saving a rule, Rule format, Relationship to auto-memory, Guardrails).

---

### Task 3: Write `agents/learn-this.md`

**Files:**
- Create: `~/.claude/skills/think-like-me/agents/learn-this.md`

- [ ] **Step 1: Write `agents/learn-this.md`**

Write the file with this exact content:

````markdown
# Learn this — from a correction

Loaded when the user invokes `/learn-this`, says "learn this" / "remember this", or accepts your auto-detect offer to capture a correction.

## Inputs

- **The correction.** The user's most recent message that corrects a prior response.
- **The corrected response.** The assistant message the correction replaced.
- **Task context.** Preceding messages that set up the task being worked on.

## Procedure

### 1. Analyze

Extract these fields from the conversation:

- **Task:** what was the user trying to accomplish?
- **Attempt:** what did you propose?
- **Correction:** what did the user say was wrong?
- **Fix:** what's the correct approach, per the user's correction?
- **Principle:** what's the generalizable rule underneath this specific correction? Do not save a rule about "this specific endpoint" — generalize to the pattern.
- **Mechanism (the why):** what makes the wrong approach fail? If the user didn't say, infer from the correction language and your understanding of the domain. If you can't infer confidently, ask.

### 2. Draft the rule

Fill in the canonical block. `Don't` is often set here — it's literally what you just did wrong.

```
### <Short, searchable title>
**When:** <domain> + <feature keywords> + <specific context>
**Do:** <the fix, generalized>
**Don't:** <what you reached for that was wrong>
**Why:** <the mechanism>
**Example:** <optional — include only if non-obvious>
```

### 3. Pick a category

Match to one of: `mobile`, `backend`, `ui`, `general`. If none fits, propose creating a new one (e.g. `devops`, `data`). Confirm the new category name with the user before writing.

### 4. Show the draft

Present in one message:

> **Draft rule → `rules/<category>.md`**
>
> [full block verbatim]
>
> Save? (`yes` / `edit` / `skip`)

### 5. Handle response

- `yes` → dual-write per `SKILL.md`'s "Saving a rule" section.
- `edit` → ask what to change, revise, re-show. Repeat until `yes` or `skip`.
- `skip` → drop silently. Do not re-prompt.
- Anything else / silence → drop silently.

## Quality bar

- **Generalize.** A rule about "the `/orders/:uuid/sync` endpoint returned 500" is too narrow. Generalize to the pattern that caused the 500. If you cannot generalize, the correction might not be rule material — just fix and move on.
- **`Why` is mandatory.** No rule without a mechanism. If you can't state one, ask the user.
- **Title is a lookup key.** 3–8 words. Start with the domain or feature, not "Don't" or "Always".
````

- [ ] **Step 2: Verify file**

Run: `head -3 ~/.claude/skills/think-like-me/agents/learn-this.md`
Expected: first line `# Learn this — from a correction`.

Run: `grep -c "^### " ~/.claude/skills/think-like-me/agents/learn-this.md`
Expected: `5` (the five numbered sub-steps under Procedure).

---

### Task 4: Write `agents/new-rule.md`

**Files:**
- Create: `~/.claude/skills/think-like-me/agents/new-rule.md`

- [ ] **Step 1: Write `agents/new-rule.md`**

Write the file with this exact content:

````markdown
# New rule — user-described

Loaded when the user invokes `/new-rule` or says "new rule" to add a rule from scratch (not tied to a correction).

## Inputs

- **User's rule description.** May be included in the invocation (e.g. `/new-rule for all Expo projects, never edit ios/ folder directly because prebuild will overwrite it`) or be bare (`/new-rule`).
- **Prior context.** Optional — the user may be generalizing from something earlier in the conversation.

## Procedure

### 1. Get the rule description

If the invocation includes the rule text, skip to step 2. Otherwise ask once:

> Describe the rule in your own words. Include: what the rule is (`Do`), when it applies (`When`), and why it matters (`Why`).

### 2. Parse into fields

Map the user's prose to the canonical block. Be charitable about interpretation:

- **When:** the context filter. Look for phrases like "for X", "when Y", "in Z projects". Extract domain + feature keywords.
- **Do:** the instruction. The imperative part of what the user said.
- **Why:** the reason. Look for "because X", "since Y". **Required** — if the user didn't state one, ask: "What's the reason? That part's required — it lets me apply the rule to edge cases."
- **Don't:** include if the user flagged a specific anti-pattern ("never X").
- **Example:** include only if the user gave one or if the instruction would be ambiguous without one.

### 3. Pick a category

Same logic as `learn-this.md` step 3: match to `mobile` / `backend` / `ui` / `general`, or propose a new category with user confirmation.

### 4. Show the draft

> **Draft rule → `rules/<category>.md`**
>
> [full block verbatim]
>
> Save? (`yes` / `edit` / `skip`)

### 5. Handle response

- `yes` → dual-write per `SKILL.md`'s "Saving a rule" section.
- `edit` → ask what to change, revise, re-show.
- `skip` → drop silently.

## Quality bar

- **Don't over-infer.** If the user's prose is ambiguous, ask one clarifying question rather than guessing.
- **`Why` is mandatory.** If missing, ask. Do not invent a reason.
- **Flag scope.** If the user describes something that looks project-specific (e.g. "for this Laravel app, use X"), ask whether they want it global or project-scoped. Project-scoped facts belong in memory, not a rule.
````

- [ ] **Step 2: Verify file**

Run: `head -3 ~/.claude/skills/think-like-me/agents/new-rule.md`
Expected: first line `# New rule — user-described`.

Run: `grep -c "^### " ~/.claude/skills/think-like-me/agents/new-rule.md`
Expected: `5`.

---

### Task 5: Seed `rules/mobile.md` (7 rules)

**Files:**
- Create: `~/.claude/skills/think-like-me/rules/mobile.md`

Seed source: cross-project mobile memories from `~/.claude/projects/-Users-kakha13-Developer-cheaperesim-mobile/memory/`. Each rule below was drafted from the corresponding `feedback_*.md` and generalized to remove project-specific detail.

- [ ] **Step 1: Write `rules/mobile.md`**

Write the file with this exact content:

````markdown
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
````

- [ ] **Step 2: Verify file**

Run: `head -3 ~/.claude/skills/think-like-me/rules/mobile.md`
Expected: first line `# Mobile rules`.

Run: `grep -c "^### " ~/.claude/skills/think-like-me/rules/mobile.md`
Expected: `7` (seven rule titles).

Run: `grep -c "^\*\*When:" ~/.claude/skills/think-like-me/rules/mobile.md`
Expected: `7` (every rule has a When line).

Run: `grep -c "^\*\*Why:" ~/.claude/skills/think-like-me/rules/mobile.md`
Expected: `7` (every rule has a Why line).

---

### Task 6: Seed `rules/backend.md` (2 rules)

**Files:**
- Create: `~/.claude/skills/think-like-me/rules/backend.md`

Seed source: cross-project backend memories. Project-specific memories (e.g. `feedback_esimaccess_orderno` which is specific to one vendor API) are NOT promoted — they stay in memory.

- [ ] **Step 1: Write `rules/backend.md`**

Write the file with this exact content:

````markdown
# Backend rules

Server-side patterns: Laravel, PHP frameworks, APIs, authentication, databases. Applies whenever the task touches backend / API code.

---

### Laravel custom middleware needs explicit priority to run before `Authenticate`
**When:** Laravel + adding custom middleware that must run before the framework's `Authenticate` middleware (e.g., auto-login via URL token, request token swap, feature-flagged auth)
**Do:** Add both the custom middleware and `\Illuminate\Auth\Middleware\Authenticate::class` to `$middleware->priority([...])` in `bootstrap/app.php`, with the custom one listed first:
```php
$middleware->priority([
    HandleCheckoutToken::class,
    \Illuminate\Auth\Middleware\Authenticate::class,
]);
```
**Don't:** Rely on route-level ordering (`->middleware(['custom', 'auth'])`) — Laravel's priority list overrides that, and `Authenticate` wins by default.
**Why:** Laravel sorts middleware by a built-in priority list. Middleware not in the list gets bucketed after prioritized entries — so `Authenticate` runs first regardless of declaration order on the route.

---

### Laravel Sanctum v4.3 has no `isExpired()` — check `expires_at->isPast()` manually
**When:** Laravel Sanctum v4.3 + checking whether a `PersonalAccessToken` is expired
**Do:** Use the manual null-safe check:
```php
$isExpired = $accessToken->expires_at && $accessToken->expires_at->isPast();
```
**Don't:** Call `$accessToken->isExpired()` — throws `BadMethodCallException`, the method does not exist in v4.3.x.
**Why:** `isExpired()` was never added to `PersonalAccessToken` in Sanctum v4.3. `expires_at` is a nullable Carbon instance on the model, and `isPast()` is the standard Carbon idiom.
````

- [ ] **Step 2: Verify file**

Run: `head -3 ~/.claude/skills/think-like-me/rules/backend.md`
Expected: first line `# Backend rules`.

Run: `grep -c "^### " ~/.claude/skills/think-like-me/rules/backend.md`
Expected: `2`.

Run: `grep -c "^\*\*When:" ~/.claude/skills/think-like-me/rules/backend.md`
Expected: `2`.

---

### Task 7: Seed `rules/ui.md` (1 rule)

**Files:**
- Create: `~/.claude/skills/think-like-me/rules/ui.md`

- [ ] **Step 1: Write `rules/ui.md`**

Write the file with this exact content:

````markdown
# UI rules

Styling, layout, icons, accessibility, UX patterns. Applies whenever the task involves user-facing visuals.

---

### No emojis in UI — use a proper icon library
**When:** any UI component + adding a symbol / glyph / status indicator / button icon
**Do:** Use the project's icon system (e.g. `SymbolView` from `expo-symbols` for React Native → SF Symbols on iOS and Material on Android; Lucide or Phosphor on web). If no icon library is set up yet, propose installing one as part of the task.
**Don't:** Use emoji characters (🎵, ⚠️, ✅), HTML entities, or inline unicode glyphs in product UI.
**Why:** Emojis render inconsistently across platforms, operating-system versions, and font stacks; they look unprofessional in product UI. Icon libraries render native-feeling glyphs and respect color + weight.
````

- [ ] **Step 2: Verify file**

Run: `head -3 ~/.claude/skills/think-like-me/rules/ui.md`
Expected: first line `# UI rules`.

Run: `grep -c "^### " ~/.claude/skills/think-like-me/rules/ui.md`
Expected: `1`.

---

### Task 8: Create empty `rules/general.md` skeleton

**Files:**
- Create: `~/.claude/skills/think-like-me/rules/general.md`

- [ ] **Step 1: Write `rules/general.md`**

Write the file with this exact content:

````markdown
# General rules

Cross-cutting principles that apply regardless of tech stack: testing, naming, git, scope discipline, debugging, commits. Accumulates over time via `/new-rule` and `/learn-this`.

Start empty. New rules append below.
````

- [ ] **Step 2: Verify file**

Run: `head -1 ~/.claude/skills/think-like-me/rules/general.md`
Expected: `# General rules`.

---

### Task 9: Final structure verification + manual smoke test plan

**Files:**
- No new files. This task verifies the end state.

- [ ] **Step 1: Inspect final tree**

Run: `find ~/.claude/skills/think-like-me -type f -name '*.md' | sort`

Expected output (exact):
```
/Users/kakha13/.claude/skills/think-like-me/DESIGN.md
/Users/kakha13/.claude/skills/think-like-me/PLAN.md
/Users/kakha13/.claude/skills/think-like-me/SKILL.md
/Users/kakha13/.claude/skills/think-like-me/agents/learn-this.md
/Users/kakha13/.claude/skills/think-like-me/agents/new-rule.md
/Users/kakha13/.claude/skills/think-like-me/references/rule-format.md
/Users/kakha13/.claude/skills/think-like-me/rules/backend.md
/Users/kakha13/.claude/skills/think-like-me/rules/general.md
/Users/kakha13/.claude/skills/think-like-me/rules/mobile.md
/Users/kakha13/.claude/skills/think-like-me/rules/ui.md
```

- [ ] **Step 2: Verify rule counts across all category files**

Run: `grep -c "^### " ~/.claude/skills/think-like-me/rules/*.md`

Expected (exact):
```
/Users/kakha13/.claude/skills/think-like-me/rules/backend.md:2
/Users/kakha13/.claude/skills/think-like-me/rules/general.md:0
/Users/kakha13/.claude/skills/think-like-me/rules/mobile.md:7
/Users/kakha13/.claude/skills/think-like-me/rules/ui.md:1
```

- [ ] **Step 3: Verify SKILL.md frontmatter parses**

Run: `awk '/^---$/{n++; next} n==1' ~/.claude/skills/think-like-me/SKILL.md | head -5`

Expected: shows frontmatter fields `name:` and `description:` — both populated, `description` starts with "Personal engineering rules".

- [ ] **Step 4: Instructions for user smoke test (manual — requires new Claude Code session)**

The automated steps above verify file integrity. The skill itself must be exercised in a live session to confirm triggering and flow. Present these four smoke tests to the user as the final verification:

**Smoke 1 — Rule application.** In a new Claude Code session, run:

> `Add a sleep timer feature to the audio player in my React Native app.`

Expected: the skill triggers, Claude reads `rules/mobile.md`, and acknowledges with a line like "Applying rule *Expo prebuild is required when adding a config plugin*" OR (if background-mode ever becomes a rule) "Applying rule *Mobile audio timers*". At minimum, Claude should surface at least one relevant mobile rule before proposing code. Note: the "Mobile audio timers" rule from the DESIGN example is NOT seeded in bootstrap — it would be added via `/new-rule` or `/learn-this` later.

**Smoke 2 — `/new-rule` flow.** Run:

> `/new-rule for all Expo projects, always lock userInterfaceStyle to "light" in app.json because iOS chrome flips to dark mode otherwise and breaks hardcoded light-theme UI`

Expected: Claude loads `agents/new-rule.md`, parses the prose into a draft rule block with `When` / `Do` / `Why` filled in, proposes the `mobile` category, shows the draft, and asks `Save? (yes / edit / skip)`. On `yes`, it dual-writes to `rules/mobile.md` and the current project's memory.

**Smoke 3 — `/learn-this` flow.** In a session where you've just corrected Claude's proposed code, run:

> `/learn-this`

Expected: Claude loads `agents/learn-this.md`, analyzes the preceding correction, drafts a rule with `Don't` populated from the wrong approach, picks a category, and asks for approval.

**Smoke 4 — Auto-detect.** In a session, correct Claude with a clear correction phrase ("no, don't use setTimeout, use a native timer") without invoking `/learn-this`. Expected: Claude offers once in a single line: "Caught — want me to save this as a rule? (yes to draft, ignore to drop)". If you ignore, it never re-prompts.

Report any deviations from expected behavior — they indicate the skill's prompt needs tightening.

---

## Self-Review Checklist (post-implementation)

After all 9 tasks complete, verify against the spec:

- **Spec coverage:** every DESIGN.md section has a task producing it:
  - File layout → T1 + T2 + T3 + T4 + T5 + T6 + T7 + T8 ✓
  - Activation trigger → T2 (SKILL.md description) ✓
  - Rule format → T1 (rule-format.md) + used in T5/T6/T7 ✓
  - Application flow → T2 (SKILL.md § When to consult rules) ✓
  - Three capture paths → T2 (SKILL.md) + T3 + T4 ✓
  - Memory integration → T2 (SKILL.md § Saving a rule) ✓
  - Bootstrap → T5 + T6 + T7 (seeded rules) + T8 (empty general) ✓
  - Guardrails / edge cases → T2 (SKILL.md § Guardrails) ✓
- **Placeholder scan:** no TBD/TODO anywhere in the written files.
- **Type consistency:** file paths in `SKILL.md` match the actual layout; rule format fields (`When`, `Do`, `Why`, `Don't`, `Example`) used consistently in `references/rule-format.md`, `agents/*.md`, and all seeded rule blocks.
