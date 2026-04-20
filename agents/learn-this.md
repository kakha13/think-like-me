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
