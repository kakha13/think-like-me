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

> Caught — want me to save this as a rule? (`yes` to draft, `skip` to drop)

On `yes` → switch to Path A (load `agents/learn-this.md`). On silence, decline, or an unrelated follow-up → drop it. Do not pressure twice in the same exchange.

## Saving a rule (dual-write, used by all three paths after user approves the draft)

**Resolving `<current-project>`:** the project folder under `~/.claude/projects/` is derived from the absolute working-directory path with every `/` replaced by `-` (the leading `/` becomes a leading `-`). Example: `/Users/kakha13/Developer/cheaperesim-mobile` → `-Users-kakha13-Developer-cheaperesim-mobile`. Run `pwd` to get the current path, then apply the transform. If the transformed folder does not exist under `~/.claude/projects/`, auto-memory is not set up for this project — write only the rule file and tell the user the memory entry was skipped (offer to create the memory directory on request).

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
