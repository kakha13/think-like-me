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
- `edit` → ask what to change, revise, re-show. Repeat until `yes` or `skip`.
- `skip` → drop silently.
- Anything else / silence → drop silently.

## Quality bar

- **Don't over-infer.** If the user's prose is ambiguous, ask one clarifying question rather than guessing.
- **`Why` is mandatory.** If missing, ask. Do not invent a reason.
- **Flag scope.** If the user describes something that looks project-specific (e.g. "for this Laravel app, use X"), ask whether they want it global or project-scoped. Project-scoped facts belong in memory, not a rule.
