---
name: defprod-change-test
description: Test stage of the change workflow — verify the implementation against every acceptance criterion of the in-scope stories and ensure e2e coverage; for bugs, confirm the original reproduction is fixed and a regression test exists. Usually invoked by /defprod-change; works standalone too.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Write
  - Edit
  - mcp__defprod__getChange
  - mcp__defprod__getUserStory
  - mcp__defprod__listUserStories
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

# Change Stage: Test

Verify the change does what the definition says, and leave durable test
coverage behind.

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

If a context resolves: call `startChangeStage { changeId, stage: 'test' }`
before beginning, and `finishChangeStage { changeId, stage: 'test' }` when all
criteria verify and required tests are green. If abandoned mid-stage, call
`cancelChangeStage`. **If no context resolves, proceed silently.**

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

- a test fails and it is genuinely unclear whether the product or the test is
  wrong — the canonical case, and the reason test-writing is not fully
  automatable;
- an acceptance criterion needs a surface this environment cannot exercise (a
  third-party sandbox, a device, a paid integration), so it can be neither
  verified nor honestly declared covered.

…do not guess, and do not finish the stage. Instead:

1. **`cancelChangeStage { changeId, stage: 'test' }`** — the stage was
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

### Features & enhancements

1. **Verify each acceptance criterion** of every in-scope story, individually:
   in the running app where there is a dev server (browser automation for UI;
   direct calls for APIs), via the test suite otherwise. Never mark a
   criterion "assumed passing".
2. **E2e coverage** per in-scope story (use `e2eDir` from
   `.defprod/defprod.json` if configured; follow existing test patterns):
   - Test exists and passes → done.
   - Test exists but misses the new criteria → update it; run until green.
   - No test → create one covering all acceptance criteria; run until green.

### Bugs

1. **Confirm the fix**: repeat the original reproduction from the define stage
   and confirm correct behaviour.
2. **Regression-check**: verify *all* acceptance criteria on the in-scope
   stories, not just the broken one — fixes can introduce regressions.
3. **Regression test**: the suite should have caught this bug. Update the
   existing test (or create one) that reproduces the original condition and
   asserts correct behaviour; run until green.

### Both types

Run the compile check once more after any test changes. If verification
exposes implementation defects, report them — the orchestrator (or user)
returns the change to the code stage rather than patching around them here.

## Rules

- Acceptance criteria are the contract; the stage finishes only when each one
  has been *observed* passing.
- A story with no testable surface is flagged to the user, never padded with
  hollow assertions.
