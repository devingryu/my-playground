# lecnote — steering

Per-repo conventions for `repos/lecnote/`, a personal Python CLI that captures
lecture audio locally and links it to PDF page turns. Grows as later items land.

## Layout

- `src/` layout: package is `src/lecnote/`, imported as `lecnote`.
- One module per concern: `cli.py` (argparse), `session.py` (paths + schemas),
  `record.py` / `tag.py` / `process.py` (subcommand handlers), plus later
  `audio.py`, `hotkey.py`, `stt.py`, `normalize.py`, `mapping.py`, `notes.py`.
- Console script `lecnote` (`pyproject.toml` `[project.scripts]`) → `cli:main`.

## Conventions

- **Stdlib first.** The CLI skeleton uses `argparse` — no CLI framework
  dependency. Reach for a third-party package only when stdlib genuinely
  doesn't cover it.
- **Dependencies are added per item, not up front.** STT / audio / hotkey
  packages belong to their own design items; keep `pyproject.toml`
  `dependencies` empty until an item needs one, and record why in that item.
- **`session.py` is the source of truth for the on-disk data contract.**
  Session paths, `meta.json` / `pages.jsonl` / `transcript.jsonl` fields, and
  the `notes/` layout live there — `record` / `tag` / `process` use those
  helpers rather than re-deriving paths or field names.
- **Raw page log is append-only.** `record` appends to `pages.jsonl` and never
  rewrites it; debounce is applied later in `process`, not at record time.
  `tag` is the one exception — explicit human edits may rewrite lines.

## Data on disk

- Session data lives under `repos/lecnote/sessions/<session-id>/`. It is
  gitignored (inside `repos/`) and is not workspace-tracked — it's local
  runtime output, not source.
