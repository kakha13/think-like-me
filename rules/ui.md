# UI rules

Styling, layout, icons, accessibility, UX patterns. Applies whenever the task involves user-facing visuals.

---

### No emojis in UI — use a proper icon library
**When:** any UI component + adding a symbol / glyph / status indicator / button icon
**Do:** Use the project's icon system (e.g. `SymbolView` from `expo-symbols` for React Native → SF Symbols on iOS and Material on Android; Lucide or Phosphor on web). If no icon library is set up yet, propose installing one as part of the task.
**Don't:** Use emoji characters (🎵, ⚠️, ✅), HTML entities, or inline unicode glyphs in product UI.
**Why:** Emojis render inconsistently across platforms, operating-system versions, and font stacks; they look unprofessional in product UI. Icon libraries render native-feeling glyphs and respect color + weight.
