---
name: defprod-change-define
description: Define stage of the change workflow — align the change with the product definition. For features/enhancements - user story alignment (find, update, or create stories); for bugs - reproduce and trace to the acceptance criteria that define correct behaviour. Links in-scope stories to the change record. Usually invoked by /defprod-change; works standalone too.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Write
  - AskUserQuestion
  - mcp__defprod__listAreas
  - mcp__defprod__getArea
  - mcp__defprod__listUserStories
  - mcp__defprod__getUserStory
  - mcp__defprod__createUserStory
  - mcp__defprod__patchUserStory
  - mcp__defprod__getChange
  - mcp__defprod__patchChange
  - mcp__defprod__startChangeStage
  - mcp__defprod__finishChangeStage
  - mcp__defprod__cancelChangeStage
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.
>
> **Then read `defprod-change/SKILL.local.md` too, if it exists**, and apply the
> parts that govern this stage. Installation policy for the *whole* pipeline is
> recorded with the orchestrator, because that is the skill that owns the
> pipeline — but this stage also runs standalone, and a stage that only read its
> own directory would silently skip policy the installation considers mandatory.
> That is a real failure mode, not a hypothetical: anything the setup binds to a
> change for its lifetime — a worktree or environment claimed for it, a database,
> a terminal session, an approval gate — is typically claimed and released by
> orchestrator-level policy, so a standalone stage that ignores it leaves the
> claim behind.

# Change Stage: Define

Connect the change to the product definition **before any code is written**.
The output is a confirmed scope: the user story IDs (and acceptance criteria)
that this change implements, linked onto the change record.

## Change context (stamping preamble)

Resolve the current change context, in precedence order:
1. `.defprod/change` in the worktree root — JSON `{ productId, changeId, changeKey, productSlug, multiProduct }`, plus `unattended: true` when the run has nobody in the session.
2. A branch named `chg/<slug>/CHG-NN-*` (or legacy `chg/CHG-NN-*`) → resolve via `getChange { productId, key }`.
3. A `Change: <product-slug>/CHG-NN` trailer on the HEAD commit → resolve the slug
   to a product, then `getChange { productId, key }` (tolerate a legacy bare
   `Change: CHG-NN` on older history, resolved with the pinned productId).

**A resolved carrier is a hint, not proof — validate it.** `getChange` the key
and confirm the change is live: if it is **shipped (frozen)** or **cancelled**,
the carrier is stale (a prior change left un-cleared). Disregard it — deleting a
stale `.defprod/change` file — and proceed as **no-context**. Only an *active*
change is a live context to stamp.

If a context resolves: call `startChangeStage { changeId, stage: 'define' }`
before beginning, and `finishChangeStage { changeId, stage: 'define' }` when
scope is confirmed. If abandoned mid-stage, call `cancelChangeStage`.
**If no context resolves, proceed silently.**

**Report the oversight the stage received.** Pass `oversight` when you stamp:
`agent` in autonomous mode, `human` in interactive mode. That is the exact inverse
of the oversight-to-mode translation the orchestrator applied to get here, so the stamp
records the oversight the stage *actually received* rather than what configuration
expected of it. Invoked standalone with no mode, report `human` — you are being
driven by the person who invoked you.

Report it on `startChangeStage`; where the start was never reported, pass it on
`finishChangeStage` instead. **First report wins**, and it is never inferred from
configuration — a stage stamped without it reads honestly unknown, which is the
point of recording it at all. An older server knows the field as `driver`: if a
stamp is refused for naming `oversight`, retry with the same value as `driver`,
and if that is refused too, retry without it. Neither refusal is a stage failure.

**If you pass `commitSha`, pass the full sha.** Take it from `git rev-parse HEAD`
(or `%H`), never `git rev-parse --short HEAD`. It is the stamp's idempotency key,
and an abbreviation is a different key for the same commit — worse, `--short`
scales its length with the repo, so the same commit abbreviates differently over
time. The server accepts short values only so older records keep validating.

## Execution mode (autonomous / interactive)

The orchestrator passes a **mode** derived from this stage's `oversight`:
`agent` → `autonomous`, `human` → `interactive`. Invoked standalone with no
mode given, default to **interactive**.

- **autonomous** — run the stage end to end without pausing: at each fork take
  the reasonable default and `finishChangeStage` once the done-condition is met.
  Surface genuine blockers, never routine choices.
- **interactive** — keep the human in the loop: ask clarifying questions at real
  decision points, and **always present the result for explicit approval before
  `finishChangeStage`**.

Where the workflow below says "confirm with the user" / "present … for
confirmation" / "ask the user", that is the **interactive** path — in
**autonomous** mode make the documented default choice and proceed.

### Blocked in an unattended run

The orchestrator passes **`unattended`** alongside the mode when there is nobody
in the session (it is also recorded as `"unattended": true` in the
`.defprod/change` pin). It means no question can be asked and no failure will be
noticed by a person watching.

**Autonomous does not mean press on.** Where this stage meets something it must
not settle alone, or cannot get past:

- the intent contradicts an existing story's acceptance criteria, so honouring it
  means overwriting a contract somebody agreed;
- the intent is too vague to write acceptance criteria from, so any story you
  wrote would be invention rather than alignment.

…do not guess, and do not finish the stage. Instead:

1. **`cancelChangeStage { changeId, stage: 'define' }`** — the stage was
   started and is being abandoned, so record that rather than leaving it open
   forever or stamping a finish over work that stopped.
2. **Raise a review item** per the *Blocked mid-stage* contract in
   `defprod-change/SKILL.md`: a `REV####` markdown file in the repo's review
   queue, `context: <product-slug>/CHG-NN`, and a `## Context` block a cold reader
   can act on — what you were doing, what you tried and ruled out, the exact
   command or assertion output, and the specific question or task a person must
   settle. **Fill in `origin` too** — the tracker ref the change came from, blank
   only for ad-hoc work. A claiming run greps that field to avoid re-claiming a
   ticket whose question is still open, so an item that omits it lets the same
   blocker be ground through again on the next run.
3. **Return `blocked`** to the orchestrator, naming the item. It parks the change.

Silence is the failure this replaces. A stage that presses on past a judgement
call produces work nobody asked for; one that fails without a record produces
nothing at all — and unattended, nobody sees either until much later.

## Workflow — by change type

### Features & enhancements: user story alignment

1. **Retrieve context**: `listAreas`, then `listUserStories` for the area(s)
   the change relates to. Read the stories and their acceptance criteria.
2. **Find relevant stories** and present them to the user for confirmation.
3. **Assess coverage** — one of three outcomes:
   - **Fully covered** — stories already describe the change. Note the IDs.
   - **Partially covered** — `patchUserStory` to update titles, descriptions,
     or acceptance criteria. Confirm with the user.
   - **Not covered** — `createUserStory` with clear acceptance criteria.
     Confirm with the user.
4. **Record scope**: state the in-scope story IDs.

### Bugs: reproduce and trace

1. **Understand the bug**: expected vs actual, where it occurs, reproduction
   steps. Ask for clarification if too vague to attempt.
2. **Trace to the definition**: find the story whose acceptance criteria
   define the *correct* behaviour — that contract is what "fixed" means. If no
   story covers the behaviour, create or update one (with user confirmation).
3. **Reproduce it** (UI via browser, API via calls, data via queries, or code
   inspection for evident logic errors). Document what you observed.
4. **Decision gate**: if the bug cannot be reproduced — HALT. Report what you
   tried, what you observed, and possible explanations. Cancel the stage work
   (`cancelChangeStage`) and return control to the user. Never fix an
   unconfirmed bug.

### Both types — link the scope

`patchChange` the change's `userStoryIds` to the in-scope story IDs.

## Rules

- **No code before definition.** Every change traces to acceptance criteria —
  that contract drives the test stage later.
- Story changes are always **confirmed with the user** before they are written.
- Acceptance criteria state capability, not implementation — keep routes,
  config keys, and file paths out of them.
- **Enumerated values are the exception, and belong in the criterion.** Where a
  capability turns on a fixed set of states, name the values or describe them in
  prose — either is fine. "Records whether the time was measured, corrected,
  recovered or inferred" is testable; "records the time's provenance" hides the
  set and is not. The set IS the capability, so leaving it out removes the
  contract rather than the implementation detail.
