# AGENTS.md

This is a [Garnet](https://github.com/devingryu/garnet) workspace — a local-first,
file-based project tracker. `projects/` and `issues/` are Garnet's own native data,
managed through the app (or by an agent editing the files directly, per
[track-work](.agents/skills/track-work/SKILL.md) — there is no API and no server).
`decisions/`, `specs/`, `guides/`, `notes/` are plain markdown docs tracked here
directly. Code for anything this workspace is about lives in `repos/` (untracked,
cloned on demand) — so a doc that lives only inside a repo isn't tracked at all;
see [Per-repo steering](#per-repo-steering).

## Principles

- Keep this file minimal. Don't write repeatable procedures here — extract them as
  a skill in [.agents/skills](.agents/skills) and link it instead.
- When a new repeatable task appears, consider extracting it into a skill first.
- Notice a correction or convention that would apply beyond the task at hand?
  Say so, explain what rule you'd write and where, and ask before applying it.
  A one-off fix that lands only in today's output is a lesson the next session
  has to learn again.

## Working here

How to file, write, and record in this workspace. The mechanics — exact file
layout and field names — are in [track-work](.agents/skills/track-work/SKILL.md).

**One test underneath all of it: does this need to be found again?** If yes it
earns a file and a name. If no, it costs nothing — do it now and move on.

### Issue, TODO, or neither

- **Issue** — work that outlives one sitting, or that someone has to find later.
  If it needs a status, it's an issue.
- **`TODO` in code** — a fix so small that finding it again costs more than
  doing it, in a file you already have open. A `TODO` that survives your commit
  names its issue: `TODO(KEY-12)`. One without an ID is a promise nobody kept —
  fix it now or file it.
- **Neither** — you're doing it in this commit.

Finished work doesn't need an issue opened just to close it. That's the git log.

### Which document

`issue.md` says why *this* work exists and what done looks like; it dies with the
issue. A document outlives it — write one only when someone will need it *without*
caring about the ticket.

Findings don't belong in either. A ticket body and `issue.md` carry background,
why now, and the requirement — someone reading them wants the ask, not a tour of
the code. Put the as-is survey in `survey.md` beside them, written as soon as you
have it rather than at the stage that happens to need it: call paths, `file:line`
pointers, structural constraints, grouped by whatever seam the code has. Then the
ticket links to it, and whoever implements later stops re-deriving it.

| Folder | Holds | Test |
|---|---|---|
| [`decisions/`](decisions/README.md) | A choice, and what was rejected | Nothing was rejected? Not an ADR. |
| [`specs/`](specs/README.md) | What to build, before building it | Already built? It's a guide, or nothing. |
| [`guides/`](guides/README.md) | How to do a repeatable thing | Done once? Leave it in the issue. |
| [`notes/`](notes/README.md) | What happened and why | Investigations, incidents, spikes. |
| [`.agents/steering/`](.agents/steering/README.md) | How code is written in one repo | About one change, not every change? Not steering. |

Each folder's `README.md` is the index — read it instead of listing the
folder. **Append-only**: a new entry goes at the bottom, never sorted or
reformatted into place. An append lands in its own diff region and never
conflicts with someone else's; a re-sort touches every row and conflicts
with everyone's — the same reasoning as the issue timeline, applied to doc
indexes.

One subject, one document. About to write the second document on a subject? Edit
the first.

### Timeline

Garnet records status changes on its own. Manual notes exist for one thing: **why**.

- Write one when the next reader would ask *why did this stall*, *why this
  approach*, *why was that dropped*.
- Don't narrate what you did — that's the diff.
- Append-only. A wrong note gets a correction appended, never an edit.

### Writing

- Lead with the answer. First sentence is the point, not the run-up.
- Bullets by default, not an occasional garnish. Two or more separate facts,
  options, causes, or items is a list — never three sentences chained with
  "and"/"also" pretending to be one thought. A paragraph earns its place only
  by carrying a single line of reasoning too connected to break apart.
- Group before you list. A flat run of more than about five sibling bullets is
  a list nobody read back — find the two or three axes the items actually fall
  on, make those the parent bullets, and nest the rest under them. A parent
  names its group and carries a fact of its own; it is not a bare label. Nest
  one level, two at most — a third means the section wanted to be two sections.
- Keep the finding apart from the decision. A document that interleaves "here
  is what the code does today" with "here is what we decided" makes the reader
  sort them out. Lead with the decision and why; put the survey behind a link,
  for the reader who wants to check the reasoning.
- Before posting a doc or issue body: if it's mostly paragraphs, rewrite it as
  bullets before it goes out, not after someone asks.
- Plain words. Prefer the shorter one. Explain a term the first time or drop it.
- No progress updates in prose. How work is going lives in the issue; a document
  says what is true now.
- A wrong document is worse than none. Change something a document describes, fix
  it in the same commit — or delete the document.
- Delete rather than archive. Git remembers.
- Before writing anything: **who reads this, and when?** No answer, no document.

### Per-repo steering

Conventions that apply to *every* change in a repo under `repos/` — how code is
written there — live in [.agents/steering](.agents/steering/README.md), one file
per repo.

They live here rather than in the repo because `repos/` is gitignored: steering
written there is untracked by this workspace and absent from a fresh clone. Each
repo keeps a short `AGENTS.md` stub pointing back at the canonical file, so an
agent working inside one repo alone still finds it.

| Repo | Steering |
|---|---|
| `<repo-name>` | [`<repo-name>.md`](.agents/steering/<repo-name>.md) |

Editing something a steering file describes? Fix it in the same commit, same as
any other doc here.

## Skills

- [.agents/skills](.agents/skills) — skills shared across agent tools (Claude Code, etc.).
- Claude Code picks up the same skills via `.claude/skills` (symlink → `.agents/skills`).
- [track-work](.agents/skills/track-work/SKILL.md) — file an issue, record why in its
  timeline, and close it, by editing the files directly.
- [feature-pipeline](.agents/skills/feature-pipeline/SKILL.md) — run a feature
  through gated requirements/design/implement/test stages, each delegated to
  a tool-scoped subagent.
