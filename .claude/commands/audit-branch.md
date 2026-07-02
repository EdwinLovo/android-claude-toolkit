---
description: Audit Kotlin source changes in the current branch against project standards; runs in an isolated sub-agent so findings aren't biased by the current conversation
argument-hint: (no args)
allowed-tools:
  - Skill
---

# /audit-branch

Invoke the `audit-branch` skill (`.claude/skills/audit-branch/SKILL.md`).

The skill is declared `context: fork`, so the audit runs in an isolated sub-agent with **no memory of this conversation**. That's deliberate — the audit should read the branch cold, as a second engineer would, so recent session context (decisions we made together, edits I just applied, my own reasoning about why a violation exists) cannot bias the findings.

Do not perform the audit inline in this session. Do not summarize files yourself. Do not filter the skill's output. Invoke the skill, then report its findings back verbatim so the user can review the raw, unbiased audit.
