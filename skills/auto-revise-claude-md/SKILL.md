---
name: auto-revise-claude-md
description: Use when ending a session, after completing significant work, or when the user says to update CLAUDE.md with session learnings. Reviews the conversation, determines what Claude needs to remember to work effectively, and applies those updates directly.
---

# Auto-Revise CLAUDE.md

Follow `/claude-md-management:revise-claude-md` with these overrides:

- **Apply what you need** — determine which learnings will make you more effective, then apply them directly using Edit. No approval step needed.
- **Also check auto-memory directory** (if one exists in the system prompt) as a target file
- **Routing**: team conventions → `./CLAUDE.md`, personal prefs → `./.claude.local.md`, cross-project → `~/.claude/CLAUDE.md`, session facts → auto-memory `MEMORY.md`
- **If nothing new was learned, say so and stop**

After applying, output a brief summary of what was added/updated and to which files.
