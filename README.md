# Garnet workspace template

A starting point for a new [Garnet](https://github.com/devingryu/garnet)
workspace — nothing but the agent-facing conventions for using Garnet well.
No sample project, no sample issue: create those from inside the app.

## Use it

1. Use this as a template (or `git clone` it and re-point `origin`).
2. Open the resulting directory in Garnet.
3. Create a project from the app — "New project" works on an empty
   workspace.

## What's here

- `AGENTS.md` / `CLAUDE.md` — how an agent should file issues, write docs,
  and behave when edits collide.
- `.agents/skills/track-work/` — the mechanics of filing/closing an issue by
  hand, for when an agent needs to edit `.garnet.yaml` directly.
- `decisions/`, `specs/`, `guides/`, `notes/` — empty doc-category indexes,
  each with a `.template.md` for the shape of a new entry.
- `decisions/0001-...md` — a worked example of a real ADR, and the actual
  policy this template uses for merge conflicts (append/append merges
  mechanically, scalar conflicts always go to a human).
