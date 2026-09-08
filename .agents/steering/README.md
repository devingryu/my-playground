# Steering

Per-repo conventions — the rules that apply to *every* change in a given repo
under `repos/`, not to one piece of work.

Steering answers "how is code written here", which none of the four doc folders
covers: it isn't a decision (nothing was rejected), isn't a spec (it describes
what already holds), isn't a procedure, and isn't an account of an incident.

| Repo | Steering |
|---|---|
| `lecnote` | [`lecnote.md`](lecnote.md) |

Add one row per repo under `repos/` that has conventions worth writing down.
Not every repo needs one — a repo with no real conventions of its own, or one
that's just vendored/reference material, doesn't earn a file just to have one.

## Why it lives here and not in the repo

`repos/` is gitignored (see [.gitignore](../../.gitignore)) — a steering file
written there is invisible to this workspace and vanishes from a fresh clone.
The canonical copy is here; each repo keeps a short `AGENTS.md` stub pointing
back at it, so an agent working inside one repo alone still finds it.

A stub looks like this:

```markdown
# AGENTS.md

Conventions for this repo are tracked in the workspace, not here:

- **[.agents/steering/<repo-name>.md](../../.agents/steering/<repo-name>.md)** —
  what it actually covers.

Don't add conventions to this file — it exists only to point at the canonical
one. This repo is cloned into a gitignored `repos/` directory, so anything
written here is invisible to the workspace that tracks it.
```

## Writing one

- Only rules that hold for every change. One-off context belongs in the issue.
- **State what the code actually does, not what it should do.** A convention
  nothing enforces is written as a convention, not as an enforced rule — an
  agent that trusts a false "the linter catches this" spends its checking
  budget in the wrong place.
- Same rules as every other doc here: English, bullets, shortest word that
  works. See [AGENTS.md](../../AGENTS.md#writing).

New rows go at the **bottom** of the table, never re-sorted.
