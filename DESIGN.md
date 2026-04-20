# Think Like Me — Design

**Date:** 2026-04-20
**Owner:** kakha13 (kakhagiorgashvili@gmail.com)
**Status:** Approved, pending implementation

---

## Purpose

A global Claude Code skill that stores the user's personal, curated engineering rules and reasoning patterns, then actively consults them before drafting code or proposing designs. Rules are domain-scoped (mobile / backend / ui / general) and capture non-obvious technical patterns that generic answers miss — e.g., "mobile audio timers need background mode" or "Expo prebuild overwrites `ios/` and `android/` folders."

The skill complements Claude Code's existing auto-memory system rather than replacing it: memories remain the passive recall layer; rules become the active reasoning layer.

## Context and gap being filled

The user already has ~20 `feedback_*.md` / `project_*.md` / `reference_*.md` entries in auto-memory at `~/.claude/projects/-Users-kakha13-Developer-cheaperesim-mobile/memory/`. These are valuable but have three gaps:

1. **Passive, not proactive.** Auto-memory surfaces entries when their descriptions match conversation context. It does not guarantee the model will factor them into *how* it approaches the task before drafting.
2. **Unstructured.** Entries are freeform prose, which makes it hard for the model to treat them as a checklist.
3. **Project-scoped.** Memories live under a specific project directory. Cross-project patterns (e.g., "no emojis in UI", "use legacy-peer-deps when installing") get duplicated or lost.

This skill fills those gaps with a structured, global, proactively-consulted ruleset.

## File layout

```
~/.claude/skills/think-like-me/
├── SKILL.md                     # Entry point: activation trigger, application flow, capture paths
├── DESIGN.md                    # This document
├── rules/
│   ├── mobile.md                # iOS / Android / React Native / Expo rules
│   ├── backend.md               # Laravel / API / server / DB rules
│   ├── ui.md                    # Styling / layout / accessibility / UX rules
│   ├── general.md               # Cross-cutting: testing, naming, git, scope, etc.
│   └── <new-category>.md        # Auto-created when /new-rule or /learn-this needs a new bucket
├── agents/
│   ├── learn-this.md            # Reactive capture flow (from correction in conversation)
│   └── new-rule.md              # Proactive capture flow (user-described rule)
└── references/
    └── rule-format.md           # Canonical rule block spec + worked examples
```

**Responsibilities:**

- `SKILL.md` — thin router. Triggers broadly on technical tasks, describes how to pick category files, how to apply matching rules, and how to invoke the three capture paths.
- `rules/*.md` — the authoritative rules. One category per file. Each rule is a structured block.
- `agents/*.md` — detailed procedures for rule drafting. Loaded on demand to keep `SKILL.md` lean.
- `references/rule-format.md` — canonical format so rules can be hand-edited without drift.

## Activation trigger (SKILL.md description)

> Personal engineering rules and reasoning patterns the user has curated over time. Consult BEFORE writing, modifying, debugging, or designing any technical feature — rules often cover domain-specific gotchas (e.g., mobile audio timers need background mode) that generic answers miss. Also triggers on `/learn-this`, `"learn this"`, `"remember this"`, or `/new-rule` to capture new rules, and on detected corrections to previously proposed code.

Design intent: broad enough to fire on any "add X", "build Y", "fix Z", or "design W" request; explicit about the capture triggers so they register.

## Rule format

Each rule is a Markdown block inside `rules/<category>.md`.

**Required fields:** title (H3), `When`, `Do`, `Why`.
**Optional fields:** `Don't`, `Example`.

**Example — minimal:**

```markdown
### Mobile audio timers
**When:** mobile app + audio playback + timer/schedule/sleep-timer
**Do:** Enable iOS Background Modes (audio) + configure AVAudioSession; use Android foreground service. Schedule the stop via native side, not JS setTimeout.
**Why:** JS timers pause when the app backgrounds. Native audio keeps playing, but no JS fires to stop it — so the timer silently fails.
```

**Example — full:**

```markdown
### Expo prebuild overwrites ios/android folders
**When:** Expo project + editing native iOS/Android files directly (ios/, android/, Info.plist, AndroidManifest.xml)
**Do:** Make changes via `app.json` + config plugins. Run `expo prebuild --clean` to regenerate.
**Don't:** Edit files inside `ios/` or `android/` directly, even when it seems faster.
**Why:** Prebuild regenerates these folders from `app.json`. Hand edits survive until the next `expo prebuild` or `eas build`, then vanish silently.
**Example:** `npx expo install expo-camera` → add camera permission to `app.json` under `plugins`, not `Info.plist`.
```

**Field semantics:**

- `When` reads like a context filter — domain + feature keywords, colon-separated. Used for matching.
- `Do` is the guidance. Terse. One paragraph maximum.
- `Why` explains the underlying mechanism (what makes the wrong approach fail). Required because it enables judgment on edge cases the rule does not literally cover.
- `Don't` is included only when the anti-pattern is tempting — typically the wrong thing Claude just reached for before being corrected.
- `Example` is included only when `Do` would be ambiguous without a concrete snippet.

## Application flow (normal task)

When the skill triggers on a task, BEFORE drafting an answer:

1. **Identify domain(s).** From task keywords and context — `mobile`, `backend`, `ui`, `general`, or other. Multi-domain tasks apply to multiple categories.
2. **Read matching `rules/<domain>.md` in full.** Files are user-curated and expected to stay readable.
3. **Scan for matches.** A rule matches when its `When:` keywords overlap with task context.
4. **Apply matches.** `Do` shapes approach. `Don't` flags paths to avoid. `Why` informs edge-case decisions when the rule isn't a perfect fit.
5. **Acknowledge briefly when a rule applies.** One-line confirmation before code: "Applying rule *Mobile audio timers* — enabling Background Modes + AVAudioSession." Lets the user catch a misapplied rule early. If zero rules match, do not emit this line — silence is fine.
6. **If zero rules match, proceed normally.** No rule is better than a forced rule.

## Capture paths

Three paths to add a rule. All end in the same dual-write (rule file + memory entry).

### Path A — `/learn-this` (reactive, from correction)

1. Load `agents/learn-this.md`.
2. Analyze the most recent correction in the conversation (the user's corrective message and the prior assistant response it corrects, plus any preceding task message that provides context). Extract: what was the task, what was proposed, what was corrected, what's the correct approach, what's the underlying principle.
3. Draft the full rule (title + `When` + `Do` + `Why` + `Don't` + optional `Example`). `Don't` is often set here because the anti-pattern is literally what was just corrected.
4. Pick a category file. If none fits, propose creating a new one.
5. Show the draft in one message with `Save? (yes / edit / skip)` gate.
6. On `yes`: dual-write. On `edit`: iterate. On `skip`: drop silently.

### Path B — `/new-rule` (proactive, user-described)

1. Load `agents/new-rule.md`.
2. If the rule text is included in the invocation, parse it. If not, ask: "Describe the rule in your own words."
3. Format user's prose into the structured block — infer `When` from context, `Do` from the instruction, `Why` from the stated reason (ask if missing — `Why` is required).
4. Pick category, show draft, same save/edit/skip gate as Path A.

### Path C — Auto-detect correction (reactive, background)

1. **Detection signal:** user pushes back on a prior response — "no, don't do X", "actually use Y", "that's wrong because Z", or rejects code with a different approach.
2. **Offer once, in one line:** "Caught — want me to save this as a rule? (`yes` to draft, ignore to drop)"
3. On `yes`: switch to Path A. On silence or decline: drop, never pressure twice in the same exchange.

## Memory integration

**Reading during task flow:**

- Auto-memory already surfaces relevant `~/.claude/projects/<proj>/memory/` entries based on description matching. No change.
- The skill additionally reads `rules/<domain>.md`.
- When a memory entry and a rule cover the same topic, the rule wins — it is the structured, curated version.

**Writing on approved captures (dual-write):**

1. **Rule file:** append block to `~/.claude/skills/think-like-me/rules/<category>.md`. Create file if new category.
2. **Memory entry:** write `~/.claude/projects/<proj>/memory/feedback_<slug>.md` with proper frontmatter (`name`, `description`, `type: feedback`). Body is a compact 2–3 line summary plus a pointer: *"Full rule: ~/.claude/skills/think-like-me/rules/<category>.md → <title>"*.
3. **MEMORY.md pointer:** append `- [<title>](feedback_<slug>.md) — <one-line hook>` to the index.

**Why dual-write:**

- The rule file actively shapes reasoning when the skill triggers.
- The memory entry ensures the rule surfaces even in contexts where the skill does not trigger (e.g., quick one-off questions) via auto-memory's description matching.
- The memory pointer links back to the authoritative rule, so edits happen in one place.

**Global rules vs project memory:** memory is project-scoped by default. Default behavior on every capture: write the memory entry to the *currently active project's* memory folder. For rules the user flags as cross-project in the draft-review gate (explicit "this applies to all projects"), still write to the current project's memory (one concrete landing place), and re-surface the same rule to other projects' memory lazily — on first invocation of the skill inside that project, offer to mirror the entry. The rule file itself is global and needs no per-project mirroring.

## Bootstrap (day-1 state)

Seeded bootstrap. On skill creation, Claude inspects existing `~/.claude/projects/<current-project>/memory/feedback_*.md` entries, classifies each as cross-project (rule material) or project-specific (leave in memory), and proposes a full seed list for user approval **before writing any rule files**.

**Proposed seed list (candidates from current memory):**

- `rules/mobile.md`:
  - NativeWind crash in tab navigator FlatLists (from `feedback_nativewind_crash`)
  - Expo prebuild required when adding config plugins (from `feedback_expo_prebuild_plugin_not_injected`)
  - Android dev client stale — rebuild when adding native modules (from `feedback_android_dev_client_stale`)
  - EAS local build /var symlink workaround (from `feedback_eas_local_var_symlink`)
  - npm legacy-peer-deps required for Expo installs (from `feedback_npm_legacy_peer_deps`)
  - Google OAuth PKCE + id_token clash in expo-auth-session (from `feedback_google_oauth_pkce_idtoken`)
  - `.env.local` overrides `.env` silently (from `feedback_env_local_override`)

- `rules/backend.md`:
  - Laravel middleware priority for pre-Authenticate custom middleware (from `feedback_laravel_middleware_priority`)
  - Sanctum v4.3 has no `isExpired()` — use `expires_at->isPast()` (from `feedback_sanctum_no_isexpired`)

- `rules/ui.md`:
  - No emojis — use expo-symbols SymbolView (from `feedback_no_emojis`)

- `rules/general.md`:
  - (Empty at bootstrap; accumulates over time via `/new-rule` and `/learn-this`.)

**Skipped (remain in memory only):**

- All `project_*.md` entries — project-specific, not rule material.
- All `reference_*.md` entries — pointers to external resources, not reasoning patterns.
- `feedback_esimaccess_orderno` — specific to the eSIM Access provider API, not a cross-project pattern.

The seed list is shown to the user in one draft message; on approval, all files are written in a single operation.

## Non-goals

- **Not a replacement for auto-memory.** Memory handles project facts, references, and per-project corrections. This skill handles cross-project technical reasoning patterns.
- **Not an automatic transcript scraper.** Rules are added only via the three explicit capture paths (`/learn-this`, `/new-rule`, auto-detect with user consent). The skill never silently saves.
- **Not a code style enforcer.** Style (prettier, eslint, formatting) belongs in tooling. Rules capture *decisions* and *patterns*, not mechanical style.
- **Not a replacement for CLAUDE.md.** Project-specific instructions that every session must see belong in `CLAUDE.md`. Rules are broader, reusable patterns.

## Guardrails and edge cases

- **Rule conflicts.** If two rules contradict (e.g., different advice for overlapping `When` clauses), flag the conflict in the acknowledgment line and ask the user which applies before proceeding.
- **Stale rules.** Rules can become wrong as ecosystems evolve (e.g., a fix is no longer needed in a newer SDK). When a user corrects against a rule I'd just applied, treat it as a strong signal to propose updating or removing that rule, not just adding a new one.
- **Category drift.** If `rules/<category>.md` grows past ~50 rules, propose splitting into sub-files (e.g., `rules/mobile-ios.md` + `rules/mobile-android.md`) during the next capture.
- **Missing `Why` on `/new-rule`.** Required field. If the user does not supply a reason, ask once — do not infer.
- **Approval gate is non-negotiable.** Never save a rule without explicit user approval on the draft, even in auto-detect mode.

## Implementation outline (not part of this spec — produced by writing-plans next)

The implementation plan will cover, in order:

1. Create skill directory structure and `references/rule-format.md`.
2. Write `SKILL.md` with activation trigger and application flow.
3. Write `agents/learn-this.md` and `agents/new-rule.md`.
4. Run the bootstrap seed flow: inspect current project memory, draft seed list, await user approval, write rule files + memory entries + `MEMORY.md` updates.
5. Smoke test: invoke the skill on a realistic prompt ("add a sleep timer to the audio player") and verify correct rules are applied.
6. Smoke test each capture path.
