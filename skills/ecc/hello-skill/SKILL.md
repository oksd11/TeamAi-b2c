---
name: hello-skill
description: >-
  Simple project skill demo. Greets the user, explains how this skill is loaded,
  and lists nearby project skills. Use when the user mentions hello-skill,
  wants a skill template, or asks to try a sample Cursor skill.
disable-model-invocation: true
---

# Hello Skill

A minimal project skill. Use it as a template for new skills in `.cursor/skills/`.

## Instructions

When this skill is active:

1. Greet the user in one short sentence.
2. Say this skill lives at `.cursor/skills/hello-skill/SKILL.md` (project scope).
3. List other directories under `.cursor/skills/` (names only, no file dumps).
4. Offer one next step: copy this folder, rename it, and edit `SKILL.md`.

Do not run extra tools unless listing sibling skills. Keep the reply under 10 lines.

## Skill layout

```
.cursor/skills/hello-skill/
└── SKILL.md
```

Required frontmatter:

- `name`: lowercase letters, numbers, hyphens; max 64 chars
- `description`: what the skill does and when to use it

`disable-model-invocation: true` means the agent loads this skill only when the user names it.

## Example reply

> Hello. This is `hello-skill` from `.cursor/skills/hello-skill/SKILL.md`.
> Nearby skills: `tdd`, `research`, `api-design`, ...
> To make your own: copy this folder, change `name` / `description`, then write your steps.
