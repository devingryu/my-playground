---
name: feature-pipeline
description: Run a feature through 4 gated stages (requirements → design → implement → test), each stage using a fixed subagent and tool allowlist. Use when a filed issue is ready to move from idea to shipped code and you want requirements/design/implement/test to follow a formalized procedure and document template.
---

# feature-pipeline

## Prerequisite

- This pipeline starts only on top of a Garnet issue that is already filed. If
  there is no issue, create one first with the [track-work skill](../track-work/SKILL.md).
- Do not run this skill's procedure standalone without an issue.

## 1. Stages and owners

| Stage | Owner | Entry condition |
|---|---|---|
| requirements | Orchestrator direct conversation + delegation to a documentation subagent | Issue is filed |
| design | Delegated to subagent | requirements.md status is Confirmed |
| implement | Delegated to subagent | design.md status is Confirmed |
| test | Delegated to subagent | Code changes exist in a committable state |

- The orchestrator handles only the user interview in the requirements stage
  directly. The other 3 stages' output is always written by a subagent.
- Before entering each stage, the orchestrator checks the status field of the
  previous stage's document. If it is not Confirmed, it asks the user for
  approval first.
- Every stage writes at most two files: its own stage document, and
  `survey.md` — the as-is code survey (section 5). The survey is shared across
  stages and is the only place a "here is what the code does today" finding
  goes.

## 2. Tool allowlist per stage

| Stage | Executor | `tools:` | Notes |
|---|---|---|---|
| requirements - interview | Orchestrator (main session) | Unrestricted | Not subject to an allowlist |
| requirements - documentation | Subagent | Read, Grep, Glob, Bash, Write, Edit | Write/Edit targets are limited by prompt to exactly two files, `{issue-dir}/requirements.md` and `{issue-dir}/survey.md`. Editing/writing source code files is prohibited. Bash is allowed for read-only search only — `grep`/`rg`/`ast-grep`/`find`/`sed -n`/`cat`. It must not write a file, run a build, or touch git |
| design | Subagent | Read, Grep, Glob, Bash, Write, Edit | Write/Edit targets are limited by prompt to exactly two files, `{issue-dir}/design.md` and `{issue-dir}/survey.md`. Survey edits are appends only — an earlier stage's finding is never rewritten. Editing/writing source code files is prohibited. Bash is allowed for read-only search only — `grep`/`rg`/`ast-grep`/`find`/`sed -n`/`cat`. It must not write a file, run a build, or touch git |
| implement | Subagent | Read, Grep, Glob, Edit, Write, Bash | File scope is specified in the prompt via design.md's "files to implement" list — enforced not by the tools field but by the prompt plus a post-completion `git status` check |
| test | Subagent | Read, Grep, Glob, Bash, Write | Write is limited by prompt to exactly one file, `{issue-dir}/qa.md`. Editing/writing production code and test code is prohibited — only running tests and recording results |

- Claude Code subagent `tools:` fields cannot express file-level
  restrictions. For the requirements-documentation and design stages, Write/Edit
  are included in `tools`, but the prompt also states explicitly "no file
  other than the following two paths is a Write/Edit target: {absolute paths}"
  as a second enforcement layer.
- A doc-stage subagent may not have every tool the table lists — Grep and Glob
  have come back unavailable in practice. That is why the doc stages get
  read-only Bash: without it the subagent is left with `Read` alone, opens
  files by guessing paths, and cannot confirm a single line number. A design
  stage run without a search tool produced a design.md that asserted "the only
  positional-argument call site is dead code" when two live ones existed.
- The delegation prompt still tells a doc-stage subagent: never invent a
  `file:line` pointer, omit what you cannot verify, and name the unverified
  facts in your report. The orchestrator checks those before showing the
  document to the user.
- Line-oriented search is not enough on its own for a call-site survey. A
  `grep` for an argument pattern misses a call whose arguments span several
  lines, and a `grep` for a named argument misses the call sites that live on
  another branch. Enumerate the call sites by method name and read each one, or
  use a structural matcher (`ast-grep`) — do not pattern-match arguments and
  then state the result as exhaustive.
- After implement completes, the orchestrator checks `git status` to confirm
  whether files not listed in design.md were touched.

## 3. Requirements stage: interview → delegate → revise loop

Fixed procedure the orchestrator follows.

1. Ask the user about purpose, scope, and constraints in question form. The
   orchestrator summarizes the user's statements rather than copying them
   verbatim.
2. Delegate the summarized content to the requirements-documentation subagent.
   The delegation prompt includes:
   - The issue path.
   - The orchestrator's summary of background/constraints/stage content.
   - An explicit statement that the only files to write are
     `{issue-dir}/requirements.md` and `{issue-dir}/survey.md`, and which
     content belongs in which (section 5).
   - An instruction to run items 1-5 of the AI-writing-style checklist in
     section 7 directly via grep.
3. The orchestrator shows the user the requirements.md the subagent wrote.
4. If Open Questions remain, or the user requests changes, return to step 1.
   There is no limit on the number of iterations. Once the user approves,
   change the document status to Confirmed and record a `kind: note` in the
   issue timeline (see section 6).

The design/implement/test stages do not have this interview loop. The
subagent reads only the previous stage's document and writes its output; the
orchestrator then shows the result to the user and asks only whether to
approve or rework it. On rework, the same subagent is called again with
feedback.

## 4. Implement/test work units and exception handling

- Before design.md's content, the implement/test delegation prompt tells the
  subagent to read and follow the target repo's own `AGENTS.md`/`CLAUDE.md`.
  design.md decides what to build, not that repo's architecture rules or
  build gate — repo-level rules always take precedence over design.md.
- The "Design" section of design.md lists work broken into pieces small
  enough to finish within a single subagent session. If the scope does not
  fit one session, design.md itself is split into multiple work items, and
  each item is handled by a separate implement subagent call.
- There is no quantitative rule for splitting sessions — the subagent writing
  design.md splits the items, and the orchestrator, on approval, only checks
  whether that split is reasonable.
- If an implement subagent judges that the instructions are too ambiguous to
  proceed, it does not guess and write code — it stops immediately and
  reports "which part of design.md is insufficient, and why." The
  orchestrator revises design.md first, then calls the same subagent again
  with the revised document.
- If the subagent finds an exception design.md did not anticipate but can
  still implement (e.g. a file not mentioned also needs to be touched, or the
  existing code structure differs from what was expected), it uses its own
  judgment, continues the implementation, and includes the exception and how
  it was handled in its completion report. The orchestrator decides with the
  user, based on that report, whether to update design.md.
- These two branches (cannot proceed → stop and revise design.md / can
  proceed but with an exception → continue and report afterward) are always
  stated explicitly in the implement delegation prompt.
- The test subagent writes `qa.md` and nothing else. It does not touch
  production or test code. If it finds a test gap, it does not add tests
  itself — it records the scenario in `qa.md` as 미검증 and says why.
- `qa.md` is the record that QA actually happened. A stage that only reports
  back in chat leaves nothing to check against months later, which is the
  whole reason this file exists.
- If the orchestrator judges from the test report that tests need to be
  added, it goes back to the implement stage and delegates writing the test
  code. There is no path in which the test stage itself fixes code.
- Before reporting done, an implement subagent runs that repo's own build
  gate if one is defined. If the repo has no such gate, it states that fact
  in the completion report.

## 5. Fixed document templates

**requirements.md** section order (fixed, no additions or omissions):

1. Meta header (date written / author / status / related ticket)
2. Background
3. Constraints
4. Progress stages (if any)
5. Milestones
6. Blockers
7. Test plan

**design.md** section order (fixed, no additions or omissions):

1. Meta header (date written / author / status / related ticket)
2. Background
3. Goals
4. Non-Goals — required
5. Design
6. Alternatives Considered — required
7. Open Questions

**survey.md** — the as-is code survey. No fixed section order: group it by
subsystem or call path, along whatever seam the code itself has. It carries:

- What the code does today, with `file:line` pointers.
- The mechanism that produces the problem — how the current structure causes
  it, not whether that structure was a good idea.
- Facts a later stage would otherwise rediscover: bean wiring, topic names,
  constants, guard conditions, schema.

It carries no decisions, goals, scope lines, or trade-offs.

It often exists before the pipeline starts — [track-work](../track-work/SKILL.md)
has the session that files the issue write it while the investigation is still in
hand. Stages append to that file rather than starting a new one.

**qa.md** — the test stage's checklist and its results. Derive one row per item
from requirements.md's Test plan and design.md's work items; do not invent a
different set. Two groups, both required even when one is empty:

- **자동 검증** — items a unit or integration test covers. Each row carries the
  check, the test class or command that proves it, the result, and the evidence
  (test name, counts, command output).
- **수동 검증** — items no automated test reaches, such as confirming rows land
  in another service's database after a real run. Each row carries the check,
  the procedure concretely enough that a person can follow it without this
  conversation, the result, and the evidence.

Result is one of 통과 / 실패 / 미검증. `미검증` is a legitimate outcome and must
carry a reason; a blank result or an unverified item written as 통과 is the one
thing this document exists to prevent.

- The two stage templates fix the section order and names. The subagent does
  not drop a section or add a new one.
- The delegation prompt pastes the section list above verbatim.
- The "Open Questions" section holds only items the user must decide
  directly. A judgment call that can be made during the next stage's
  execution goes in the "Design" section as "orchestrator/subagent judgment
  call," not in Open Questions.
- If even one open question remains, the document status is not changed to
  Confirmed.

### What goes where

- requirements.md and design.md are read by someone asking **why this work
  exists and what was decided**. A sentence earns its place there by serving
  that question.
- survey.md is read on demand — by a person checking the reasoning, and by the
  design/implement subagents, which are pointed at it directly.
- A stage document states a finding only where a decision rests on it, and then
  in one line that links into survey.md. The investigation itself never goes
  inline.
- When a survey.md exists, both stage documents link to it from Background.

### Structure within a section

- Group before listing. More than about five sibling bullets under one heading
  means the axes were not found — name the two or three axes as parent bullets
  and nest the items under them.
- Nest one level, two at most. A third level means the section wanted to be two
  sections.
- A parent bullet names its group and carries a fact of its own. A bare label
  like "Related items:" is not a parent bullet.

## 6. Issue timeline recording

- At each stage transition, the orchestrator adds a `kind: note` entry to
  `timeline` in `.garnet.yaml` (either editing directly or using the
  track-work skill procedure).
- Three points trigger a recording:
  - A document's status changes to Confirmed and the next stage starts.
  - A previous stage's document is reopened.
  - A fact different from the approval criteria is discovered and the plan
    changes.
- The recorded content states why the transition happened, not what was
  done. Example phrasing:
  - "requirements.md switched to Confirmed — starting design stage."
  - "Found a missing requirement during design review, reopened
    requirements.md."
  - "Found a constraint during implementation that differs from the tech
    spec — updated design.md and re-approved."
- `kind: status` is used only when the issue's own status (todo/in-progress/
  done) changes. It is a separate event from pipeline stage transitions.
- The document status field (Draft/In Review/Confirmed) exists only in the
  document file itself and is not automatically linked to the issue status.
  The orchestrator updates both values manually, separately.

## 7. Banned AI-writing-style rules

Immediately after a subagent writes its output, the same subagent checks its
own document in the following order and fixes any violation immediately.

1. Check whether a banned opener appears at the start of a paragraph or
   section: "It's important to note that", "This document covers", "In
   conclusion".
2. Check for banned words.
   - English words: delve, underscore, showcase, pivotal, intricate,
     meticulously, realm, garnered, notably.
   - Vague-hedge phrases: "it could be said that", "various", "several",
     "numerous".
3. Check that vague quantity expressions have been replaced with a concrete
   count or example.
4. Check that a section heading is not repeated verbatim in the first
   sentence beneath it.
5. Check that the number of list items fits the content naturally (not a
   forced triad).
6. Check that every bullet is a complete sentence with a subject, verb, and
   fact.
7. Check that no heading has more than about five ungrouped sibling bullets
   under it.
8. Check that no as-is code investigation sits in requirements.md or design.md
   where it belongs in survey.md.

- Items 1-5 are grep-able patterns, so the subagent checks them mechanically
  with its own grep right after writing the document.
- Items 6-8 (bullet completeness, grouping, finding-vs-decision separation)
  are judgment, not patterns, and no separate subagent is spun up for them.
  The orchestrator reads the output directly before showing it to the user and
  judges it. If it finds a problem, the orchestrator either fixes it directly
  or calls the same subagent again with the specific violation locations
  pointed out for a rewrite.
- Before showing a subagent's output to the user, the orchestrator confirms
  from the subagent's response that the items 1-5 checklist passed; if there
  is no mention of passing, it asks the subagent to recheck.
- If this workspace's documents are written in a language other than
  English, translate the banned openers/words/fillers above into that
  language's equivalents before using this checklist, and keep both lists
  side by side in the delegation prompt.

## 8. Document storage location

- If an epic exists: place requirements.md/design.md under the epic issue's
  directory.
- If no epic exists: place them in the directory of the issue the work
  targets.
- Do not create documents as standalone files outside an issue/epic
  directory. Workspace-shared document folders such as `specs/` and `notes/`
  are not used as the storage location for this pipeline's output.
- The delegation prompt always specifies absolute paths as the "files to
  write" — the stage document, plus `survey.md` for the requirements and
  design stages, and `qa.md` for the test stage. `survey.md` lives beside the stage documents, under the same
  issue or epic directory.
- If an epic is created later and the document needs to move, the move
  procedure itself is not fixed. The only enforced condition is that the
  document must remain findable from the original issue after the move —
  either by leaving a `kind: note` in the original issue's timeline pointing
  to the new location, or by referencing it with a plain markdown link.
  Either one is sufficient.

## 9. (Optional) Persona-based use-case review

Optional review the orchestrator may run before confirming design.md. This
uses persona framing for a different purpose than the ban in sections 3-4 —
it is not meant to improve a coding subagent's task performance, it is meant
to simulate a prospective user's reaction for design-review reference.

- **Whether to run it**: not required. The orchestrator decides based on the
  feature's weight (scope of impact, size of the user-facing surface).
- **Persona count**: no fixed number — 2-3 for a small feature, 4-6 for a
  feature with wide impact, adjusted by the orchestrator. At least one
  persona must sit on an edge-case/adversarial axis (a user who misuses the
  feature, or who lacks it and is inconvenienced) — demographic-only axes
  tend to miss this angle.
- **Persona axes**: demographics (age, location) are not the primary axis.
  Combine skill level (novice/expert), goal (what they're trying to do with
  this feature), context of use (when/why they use it), and edge-case/
  adversarial use instead.
- **Making a persona concrete**: a shallow role tag ("You are a senior
  developer") is not enough — give each persona concrete goals and
  constraints that tie into the actual content of design.md and conflict
  with each other (example: "opens this workspace once a week and needs to
  understand the past week's progress within 5 minutes"). Personas built
  from demographic tags alone tend to converge on similar answers, losing
  the diversity the exercise is for.
- **Subagent delegation**: one subagent call per persona. The delegation
  prompt includes design.md's path (read-only), that persona's concrete
  goals/constraints, and a request for free-form answers to "how would this
  person use the feature, what would feel missing, what related feature
  would they want." Tools: Read only — this subagent writes nothing.
- **Handling the output**: persona responses are not merged into design.md.
  Keep them as reference material in a separate section or a separate file
  (e.g. `{issue-dir}/persona-review.md`), used as supporting material in
  upward design review. A human decides what to act on — there is no
  automatic path from a persona response to a design change.
- **Stated limitation**: this output does not replace real user research.
  LLM personas tend to converge toward each other more than a real user
  population would, and tend to underrepresent minority/non-mainstream
  viewpoints — the mandatory edge-case persona rule above exists because of
  this.

## 10. Delegation prompt examples

Below are delegation prompt examples that can be pasted directly into an
Agent tool call, adapted with this workspace's actual paths. They do not
include persona phrasing ("You are a senior ~").

**Design stage delegation example:**

```
Read <workspace-root>/issues/<ISSUE-ID>/requirements.md and
<workspace-root>/issues/<ISSUE-ID>/survey.md, and nothing else in that
directory. Write a design.md for this issue at exactly this path:
<workspace-root>/issues/<ISSUE-ID>/design.md

Constraints:
- Use only these tools: Read, Grep, Glob, Bash, Write, Edit. Bash is for
  read-only search only (`grep`, `rg`, `ast-grep`, `find`, `sed -n`, `cat`) —
  never to write a file, run a build, or run git.
- The only files you may Write or Edit are
  <workspace-root>/issues/<ISSUE-ID>/design.md and
  <workspace-root>/issues/<ISSUE-ID>/survey.md. Do not touch any source code
  file, do not run Bash.
- survey.md is append-only for you. Add new as-is findings you turn up; never
  rewrite or delete what the requirements stage recorded there.
- Follow this exact section order, no additions or omissions:
  1. Meta header (date written / author / status / related ticket)
  2. Background
  3. Goals
  4. Non-Goals
  5. Design
  6. Alternatives Considered
  7. Open Questions
  Non-Goals and Alternatives Considered are mandatory — do not skip them.
- design.md answers "what are we building and why." Every as-is code finding
  goes in survey.md instead. State a finding in design.md only where a
  decision rests on it, in one line, linking to survey.md for the detail.
- Write in this workspace's established document language, in bullet-point
  style. Prose paragraphs only for the one or two sentences explaining why a
  section exists.
- Group before you list. More than about five sibling bullets under one
  heading means you have not found the axes — name two or three axes as parent
  bullets and nest the items under them. Nest one level, two at most.
- Open Questions may contain only items that require a user decision. If none
  remain, write "None" with a one-line reason.
- Never invent a file:line pointer. If a tool you need is unavailable and you
  cannot verify a pointer, omit it and list that fact as unverified in your
  report.
- After writing the file, grep your own output for the banned openers and
  words listed in section 7, and fix any hits before reporting done.
- Do not write in a persona or role-play voice. Write as a plain technical
  document.
- Report back: file paths written, section list confirmation, the result of
  the banned-word grep check, and every fact you could not verify.
```

**Test stage delegation example:**

```
Read <workspace-root>/issues/<ISSUE-ID>/requirements.md and
<workspace-root>/issues/<ISSUE-ID>/design.md, then audit what the branch's
tests actually verify. Record the result at exactly this path:
<workspace-root>/issues/<ISSUE-ID>/qa.md

Constraints:
- Tools allowed: Read, Grep, Glob, Bash, Write.
- The only file you may Write is
  <workspace-root>/issues/<ISSUE-ID>/qa.md. Do not edit production code, and
  do not edit or add test code — not even a test you think is missing.
- Derive the checklist from requirements.md's Test plan and design.md's work
  items. Do not substitute a checklist of your own.
- Group it under 자동 검증 and 수동 검증, per section 5. Every row carries the
  check, how it is proven, the result (통과 / 실패 / 미검증), and the evidence.
- Run the repo's own test command and quote the real counts. Never write 통과
  for something you did not observe passing.
- A scenario with no test is recorded as 미검증 with the reason. That is the
  point of the audit — do not paper over it, and do not fix it yourself.
- For 수동 검증 rows, write the procedure concretely enough that someone can
  follow it later without this conversation.
- Report back: the file you wrote, the counts per result, and the 미검증 rows
  with what would be needed to close each.
```

**Implement stage delegation example:**

```
First read the target repo's own AGENTS.md/CLAUDE.md and follow its
conventions section for everything you write. Then read
<workspace-root>/issues/<ISSUE-ID>/design.md and implement item 2 of its
Design section only (name the item here).
<workspace-root>/issues/<ISSUE-ID>/survey.md holds the as-is code survey for
this issue — read it for the file/line pointers and existing structure rather
than rediscovering them, but treat design.md as the decision of record.

Constraints:
- Tools allowed: Read, Grep, Glob, Edit, Write, Bash.
- The target repo's own AGENTS.md/CLAUDE.md conventions take precedence over
  design.md for any code-style or architecture question design.md doesn't
  decide.
- Only touch files listed in design.md's "files to implement" list for this
  item. If you need to touch a file not on that list, proceed but report it
  as an exception in your completion summary — do not stop for that alone.
- If any instruction in design.md is too ambiguous to implement without
  guessing, stop immediately. Do not write speculative code. Report exactly
  which part of design.md is insufficient and why, then wait for a revised
  design.md.
- If you can implement but discover a fact design.md didn't anticipate
  (e.g. an existing structure differs from what's described), proceed with
  your best judgment and describe the exception plus how you handled it in
  your final report.
- Before reporting done, run the target repo's own build gate if one exists,
  and fix anything it flags.
- Do not write in a persona or role-play voice.
- When done, report: files changed, build-gate result, any exceptions found
  and how handled, and whether the item's scope is complete.
```
