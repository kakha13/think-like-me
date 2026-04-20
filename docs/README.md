# think-like-me

A personal Claude Code skill that stores your curated engineering rules and actively consults them *before* Claude drafts code. Complements Claude Code's passive auto-memory: memory surfaces facts when relevant, rules shape the approach up front.

## Viewing these docs

These pages are rendered by **[Docsify](https://docsify.js.org)** — a zero-build, CDN-loaded markdown site. Because Docsify fetches markdown over HTTP, you must serve the skill folder (opening `index.html` via `file://` won't work — browsers block `fetch()` on local files).

```bash
# From ~/.claude/skills/think-like-me/
python3 -m http.server 8000
# then browse to → http://localhost:8000/docs/
```

Any static server works — `npx serve .`, `php -S localhost:8000`, `ruby -run -e httpd .` all do the job. The docs render the live skill files, so editing any `.md` in the skill and refreshing the page shows the update instantly.

## What the skill does

When you ask Claude to build, modify, debug, or design a technical feature, the skill:

1. Identifies the domain(s) of the task (`mobile`, `backend`, `ui`, `general`).
2. Reads the matching `rules/<domain>.md` file(s).
3. Applies any rules whose `When:` keywords overlap with the task.
4. Acknowledges the applied rule in one line before writing code so you can catch a misapplied rule early.

If no rule matches, Claude proceeds normally — no forced matches.

## Three ways to add a rule

| Invocation | When to use | Flow |
|---|---|---|
| `/learn-this` | Right after correcting Claude | Claude analyzes the correction, drafts a rule with `Don't` pre-filled from the wrong approach, asks for approval. |
| `/new-rule [description]` | Adding a rule from scratch (not tied to a correction) | Claude parses your prose into the structured block, asks if `Why:` is missing, shows draft. |
| Auto-detect | You correct Claude without typing a command | Claude offers once: *"Caught — want me to save this as a rule? (`yes` to draft, `skip` to drop)"* — never pressures twice. |

All three end the same: draft → approve → dual-write to `rules/<category>.md` **and** `~/.claude/projects/<project>/memory/feedback_<slug>.md` (with a pointer in `MEMORY.md`).

## Rule format

```
### Short searchable title
**When:** domain + feature keywords + specific context
**Do:**   the fix, generalized
**Don't:** what not to do (optional)
**Why:**  the mechanism that makes the wrong approach fail
**Example:** short snippet (optional)
```

`When` / `Do` / `Why` are required. `Why` in particular matters — it lets Claude judge edge cases the rule doesn't literally cover. Full spec: [rule-format.md](references/rule-format.md).

## File layout

```
~/.claude/skills/think-like-me/
├── SKILL.md                    ← AI-facing entry point (trigger, flow, dual-write)
├── DESIGN.md                   ← full design spec
├── PLAN.md                     ← implementation plan
├── docs/
│   ├── README.md               ← this file
│   ├── index.html              ← Docsify renderer
│   └── _sidebar.md             ← Docsify nav
├── references/rule-format.md   ← canonical rule block spec
├── agents/
│   ├── learn-this.md           ← /learn-this procedure
│   └── new-rule.md             ← /new-rule procedure
└── rules/
    ├── mobile.md               ← iOS / Android / React Native / Expo
    ├── backend.md              ← Laravel / API / server
    ├── ui.md                   ← styling / layout / icons
    └── general.md              ← cross-cutting (starts empty)
```

## Seeded rules (day one)

Migrated from cross-project entries in your auto-memory:

**Mobile (7):** NativeWind crash in tab FlatLists · Expo prebuild required for new plugins · Native module rebuild on dev client · EAS `--local` /var symlink fix · npm `--legacy-peer-deps` workaround · Google OAuth PKCE + IdToken clash · `.env.local` override trap

**Backend (2):** Laravel custom middleware priority · Sanctum v4.3 no `isExpired()`

**UI (1):** No emojis — use icon library

**General:** empty — accumulates via `/new-rule` and `/learn-this`

Project-specific memories (e.g. eSIM Access `orderNo` fix, processing-tab work, Paddle setup) were intentionally *not* promoted — they stay in project-scoped memory where they belong.

## Extending

- **Add a rule directly:** `/new-rule [your description including When, Do, Why]`
- **Teach from a correction:** correct Claude, then `/learn-this`
- **Add a new category:** just mention it in `/new-rule` (e.g. "this is a devops rule") — Claude will propose creating `rules/devops.md` and ask to confirm
- **Edit a rule:** open the relevant `rules/<domain>.md` in any editor; format stays the same
- **Remove a rule:** delete its block from the category file; if it had a mirrored memory entry, delete that too

## Design & reference

- [Design spec](DESIGN.md)
- [Implementation plan](PLAN.md)
- [Rule format](references/rule-format.md)
- [/learn-this flow](agents/learn-this.md)
- [/new-rule flow](agents/new-rule.md)
