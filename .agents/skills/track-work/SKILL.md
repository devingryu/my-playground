---
name: track-work
description: File a Garnet issue, record why in its timeline, and close it — by editing the files directly. Use when starting a piece of work in this workspace, when work stalls or changes direction, or when you need to write .garnet.yaml by hand.
---

# Track work in a Garnet workspace

Whether something deserves an issue at all is a convention, not a procedure —
that's in [AGENTS.md](../../../AGENTS.md#working-here). This is the mechanics.

Agents edit these files directly; there is no API and no server — files are
the source of truth, by design. The app reads the same files, so a
hand-written issue and one made in the UI are indistinguishable, as long as
the schema below is exact.

## Before you start

Check whether the issue already exists:

```bash
ls issues/
grep -rl "<keyword>" issues/*/
```

Reopening a closed issue beats filing a near-duplicate. Two issues for one piece
of work is the failure mode this workspace is trying to avoid.

## File an issue

An issue is a **directory**, named by its ID. The ID is not stored
inside the file — the directory name *is* the ID.

**Allocate the ID**: highest existing number for that project, plus one. Scan
`issues/` for `<KEY>-<n>` directories; ignore malformed names rather than
failing. Gaps are fine, reuse is not.

```
issues/KEY-4/
├── .garnet.yaml   # metadata + timeline
├── issue.md       # the description — free-form markdown
└── ...            # any number of attached documents
```

`.garnet.yaml` — **field order is load-bearing**, `timeline` last so appends land
at the end of the file and diffs stay small:

```yaml
title: Assignee dropdown shows the raw email
type: bug
status: todo
parent: ""
reporter: you@example.com
assignee: ""
links: []
timeline: []
```

- `title` — required, and structured metadata; it is not parsed out of `issue.md`.
- `type` — must be one of the project's declared `issue-types` (`projects/<KEY>/project.md`).
- `status` — must be a status id from `projects/<KEY>/workflow.md`, e.g. `todo`.
- `parent` — an issue ID, or empty. A link only; children are never nested as
  subdirectories, so re-parenting never moves a directory.
- `reporter` / `assignee` — emails, and the email is the identity key (git-style,
  no user registry). Take yours from `.garnet.local.yaml` (untracked, per-machine):
  ```yaml
  user:
      name: Your Name
      email: you@example.com
  ```
  `assignee` must be a member declared in `projects/<KEY>/project.md`.
- `links` — `{type, target}` pairs, e.g. `{type: blocks, target: KEY-5}`.

Write `issue.md` in the same commit. An issue with no body is a title pretending
to be a ticket.

**Write it as bullets, not paragraphs** (AGENTS.md's Writing section) — this is
the rule most worth re-checking before saving, since issue bodies drift into
prose easily. Concretely:

- What's broken/needed, as a list of facts if there's more than one.
- Where in the code, as a list of file/line pointers if there's more than one.
- What's explicitly out of scope, as its own short list.

A paragraph is fine for the one or two sentences that frame *why this issue
exists at all* — everything after that should default to a list.

**Code findings go in `survey.md`, not in the body.** Both the tracker ticket and
`issue.md` stay on background, why now, and the requirement; the reader wants the
ask. The as-is survey — call paths, `file:line` pointers, structural constraints,
grouped along whatever seam the code has — goes in `survey.md` in the same issue
directory, and the body links to it.

Write it at filing time, while the investigation that produced the issue is still
in hand. It is the same file the [feature-pipeline](../feature-pipeline/SKILL.md)
stages read and append to, so filing it early costs nothing and saves the
implementing session from re-deriving it. Note which commit or branch the line
numbers came from — they go stale.

The tracker ticket takes the same shape: real headings (`## 배경`), not bracketed
labels, and bullets rather than paragraphs.

## Record why

Append to `timeline`. Never edit or reorder an existing entry — it's the
record of what happened, so editing it would defeat its purpose.

Two kinds:

```yaml
timeline:
  - {at: 2026-08-14T10:00:00Z, by: you@example.com, kind: status, from: todo, to: in-progress}
  - {at: 2026-08-14T14:00:00Z, by: you@example.com, kind: note, body: "Parked — waiting on a decision upstream."}
```

- `at` — RFC 3339, UTC.
- `kind: status` — carries `from`/`to`. Write one every time you change `status`;
  the app does this automatically, so doing it by hand and forgetting the entry
  leaves an issue whose history lies.
- `kind: note` — carries `body`. This is the one Jira loses: *why*, not *what*.

## Close it

Set `status` to a closed-category status, append the matching `kind: status`
entry, and check `workflow.md` allows that transition — the app enforces it, and
a hand-written file that skips a step will look valid but won't be reachable the
same way in the UI.

Leave the issue directory in place. Closed is a status, not a deletion.

## Links between things

Plain relative markdown links, no custom syntax:

```markdown
See [KEY-3](../KEY-3/) and [the storage decision](../../decisions/0002-some-decision.md).
```

Never write a backlink by hand. Garnet derives them by scanning, so a
hand-maintained reverse link is one half of a pair that will go stale.
