# ADR 0001: Concurrent edits resolve mechanically or go to a human — never an agent's guess

- Status: Accepted
- Date: 2026-08-17

## Context

Multiple people (or agents acting for them) can edit this workspace at the
same time. Two different shapes of conflict can result, and they need
different answers:

- **Append/append.** Two people add different entries to the same
  append-only structure — an issue's `timeline`, or one of the doc-category
  `README.md` indexes. Order doesn't change meaning here; both entries are
  simply true and belong in the file.
- **Scalar/scalar.** Two people set the same single-value field — `status`,
  `title`, `priority`, `assignee` — to different values. Here order *does*
  matter, and there's no mechanical way to tell which one should win: that's
  a real decision about the state of the work, not a merge problem.

Treating both shapes the same way — either always auto-merging, or always
stopping for a human — is wrong for one of the two cases every time.

## Decision

**Append/append conflicts resolve mechanically: keep both, ordered by
timestamp.** No judgment involved — this is exactly why timeline entries
and doc-index rows are append-only in the first place. An agent may perform
this merge itself.

**Scalar/scalar conflicts always escalate to a human. An agent never
guesses.** Show both values and who/when set them; the human picks. This
applies regardless of how "obvious" the right answer looks from context —
the whole point is removing the agent's subjective judgment from state that
represents someone's real decision.

**Reduce how often this happens at all, cheaply, before either rule is
needed:**

- Keep structures append-only wherever the data allows it (timelines, doc
  indexes) — an append lands in its own diff region and essentially never
  conflicts.
- Treat `assignee` as a soft, unenforced signal for "who's actively working
  this issue right now" — don't edit someone else's assigned issue without
  asking first. No tooling required, just a convention.

## Alternatives considered

- **Let git's default merge strategy handle everything.** Rejected: a naive
  automatic merge (e.g. blindly preferring one side) would silently pick a
  `status` value for someone, which is exactly the outcome this ADR exists
  to prevent. Plain git conflict markers on a scalar field are the correct
  outcome — a human still has to resolve them by hand today, this ADR
  doesn't remove that step, only makes it unambiguous when it happens.
- **A `.gitattributes` merge driver that auto-merges append/append
  conflicts for real.** Not rejected, deferred — worth building once this
  actually happens often enough to justify a script. Until then, the rare
  append/append conflict git does produce is resolved by hand using the
  same timestamp-order rule; nothing here depends on the driver existing.
- **Automate resolution with an LLM (even a cheap/lightweight model), via a
  hook or merge driver.** Rejected for both conflict shapes, for different
  reasons:
  - For scalar conflicts, any automatic decision — model-based or not —
    defeats the point of this ADR. The problem was never "no algorithm is
    smart enough to pick," it's "this isn't an algorithm's decision to
    make."
  - For append/append conflicts, a model adds cost, latency, and
    non-determinism to something a five-line deterministic script already
    solves for free.
- **A locking/reservation mechanism** — mark an issue "checked out" before
  editing, enforced by the app. Rejected as disproportionate machinery for
  how rarely two people edit the exact same file at the exact same moment
  in a workspace this size; the unenforced `assignee` convention above
  covers the common case at no cost.

## Consequences

**Easier**
- Most real work never hits a conflict at all — timelines and doc indexes
  are safe to edit concurrently by construction.
- The rare conflict that does happen has an unambiguous answer: mechanical
  for appends, "stop and ask" for scalars. No case falls into a gray zone
  where an agent has to use its own judgment.

**Harder**
- Scalar-field conflicts still require a human in the loop by hand (or an
  agent pausing to ask) — this ADR makes that rare and unambiguous, not
  automatic.
- The append/append mechanical merge isn't automated yet; today it's a
  by-hand step using the timestamp-order rule when git does flag a
  conflict.

**Later.** A merge driver for `.garnet.yaml`'s `timeline` block and the doc
`README.md` tables could apply the keep-both/timestamp-order rule
automatically, removing even that manual step. Deferred until it's actually
needed often enough to be worth building.
