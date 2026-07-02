---
description: Update CLAUDE.md, path-scoped rules, and skills to reflect new patterns introduced in the current branch
argument-hint: (no args)
allowed-tools:
  - Read
  - Edit
  - Write
  - Bash
  - Grep
  - Glob
---

# /update-docs

Update project documentation to reflect new patterns, components, or conventions introduced in the current branch. Covers `CLAUDE.md`, `.claude/rules/*.md`, `.claude/skills/*/SKILL.md`, and per-user memory.

---

## Scope rules

- Only document patterns **already in the codebase** — never aspirational or planned work
- Do not document things derivable from reading the code (file structure, class names)
- Prefer updating existing sections over adding new ones
- Minimum accurate additions — no rewrites, no padding

---

## Steps

### 1 — Review the branch diff

```
git diff main...HEAD
```

Identify what is genuinely new at the **pattern or convention** level (not a one-off feature implementation).

---

### 2 — Update the right doc

| What changed | Where to update |
|---|---|
| New reusable UI component in `presentation/ui/components/<category>/` | `CLAUDE.md` — Component files section |
| New dialog pattern or wrapper rule | `CLAUDE.md` — Dialogs section, `.claude/rules/dialogs.md` |
| New `<THEME>` token type or new usage rule | `CLAUDE.md` — Theme usage table, `.claude/rules/theming-and-tokens.md` |
| New screen-level architectural pattern (delegates, contracts, ...) | `.claude/skills/screen-anatomy/SKILL.md`, `.claude/rules/screens.md`/`contracts.md`/`delegates.md` |
| New data layer pattern (data source split, base method, mapper rule) | `.claude/skills/data-feature/SKILL.md`, `.claude/rules/repositories.md`/`data-sources.md`/`mappers.md` |
| New ViewModel pattern (constant placement, reduce helper, ...) | `.claude/rules/viewmodels.md`, `.claude/rules/constants-in-viewmodels.md` |
| New error handling utility | `.claude/rules/error-handling.md`, `CLAUDE.md` — Errors & UiText |
| New slash command | `CLAUDE.md` — Skills & Commands table + create the file under `.claude/commands/` |
| New project-wide convention | `CLAUDE.md` — Conventions section |

For any new rule you add, remember the frontmatter shape:
```yaml
---
name: <slug>
description: <one line>
paths:
  - "<glob>"
---
```

---

### 3 — Check memory

Review whether any non-obvious patterns, user preferences, or architectural decisions from this session should be saved to the persistent memory store at:
```
~/.claude/projects/<project-slug>/memory/
```

Save only things that:
- Would not be obvious from reading the code or docs
- Should influence future behavior in new conversations
- Are not already in `CLAUDE.md` or a rule/skill file

Update `MEMORY.md` index if a new memory file is written.

---

### 4 — Confirm

Print a summary:

```
Updated:
- <file> — <what was added/changed and why>
- ...

No changes needed:
- <file> — already accurate / not affected by this branch
```
