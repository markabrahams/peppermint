---
name: defprod-change-review
description: Review stage of the change workflow — review the change's diff for correctness, security, test coverage, scope fidelity, and convention adherence before it lands. Usually invoked by /defprod-change; works standalone too.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Write
  - AskUserQuestion
  - mcp__defprod__getChange
  - mcp__defprod__getUserStory
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

# Change Stage: Review

Review the change before it lands. This skill is **self-sufficient and
agent-portable** — it relies only on `Read`/`Grep`/`Bash` and the DefProd MCP,
so it gives the same review on any host (Claude Code, Cursor, or otherwise),
not just hosts that ship a built-in review command.

If your host has a richer native review command (e.g. `/code-review`) and you
are reviewing on a surface it fits — a GitHub pull request, say — you may prefer
it there. But for the DefProd change-record flow (diff-based, no pull request
required) this skill is the primary path, not a fallback. If you do lean on such
a reviewer, treat it as an engine for the correctness lens only — the
scope-fidelity lens, the confidence pass, severity ranking, and DefProd stamping
stay this skill's responsibility regardless of host.

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

If a context resolves: call `startChangeStage { changeId, stage: 'review' }`
before beginning, and `finishChangeStage { changeId, stage: 'review' }` when
the review passes (findings resolved or accepted). If abandoned mid-stage,
call `cancelChangeStage`. **If no context resolves, proceed silently.**

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

Where the workflow below says "raise … with the user" / "ask the user", that is
the **interactive** path — in **autonomous** mode fix clear-cut defects and
proceed, recording accepted judgement calls in the finish note.

### Prepare mode (unattended, ahead of a human review)

The orchestrator passes **`mode=prepare`** when an unattended run reaches a
`review` stage that a person must take — nobody is in the session to finish it,
but the reviewer should not start from a blank diff. In this mode:

- **Stamp the start, never the end.** Call `startChangeStage { changeId, stage:
  'review', oversight: 'human' }` before running the lenses (the orchestrator will
  usually have stamped it already; a second start is a no-op). Never call
  `finishChangeStage` or `cancelChangeStage`: the stage stays in progress for the
  person who finishes it. This is the record interactive mode leaves while it
  waits for approval, and `human` is the oversight the stage actually receives.
- **Change nothing.** Run Workflow steps 1–5 (scope, context, lenses, confidence
  pass, ranking) but do not fix findings, not even clear-cut ones. The change has
  passed its test stage, and code altered now would reach the reviewer untested.
  The person decides what to fix.
- **Return the ranked findings** to the orchestrator: severity, `file:line`, what
  is wrong and why, and the suggested fix, or `no findings`. The orchestrator
  records them in the parked commit and the exit report.

Prepare mode never produces a verdict. The review is finished by the person who
picks the change up, in interactive mode.

### Blocked in an unattended run

The orchestrator passes **`unattended`** alongside the mode when there is nobody
in the session (it is also recorded as `"unattended": true` in the
`.defprod/change` pin). It means no question can be asked and no failure will be
noticed by a person watching.

**Autonomous does not mean press on.** Where this stage meets something it must
not settle alone, or cannot get past:

- a finding whose fix needs a design decision, rather than a clear-cut defect you
  can simply correct;
- a diff that has drifted outside the change's scope, where trimming it is a call
  about what the change is for.

…do not guess, and do not finish the stage. Instead:

1. **`cancelChangeStage { changeId, stage: 'review' }`** — the stage was
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

## Workflow

1. **Scope the diff**: everything the change touched (uncommitted work and/or
   the branch diff against the default branch).
2. **Gather context** before judging the lines in isolation — this is what
   keeps findings grounded and cuts false positives:
   - **Repo rules & known traps** — read the rules/contributing docs the repo
     defines for the touched surfaces, so convention findings cite an actual
     written rule. Many repos also document their own *recurring* pitfalls (a
     gotchas / known-traps doc, or a rule marked blocking); consult those and
     check the diff against them.
   - **Callers & tests** — identify the callers of any changed function or
     interface, and the existing tests over the changed region. A change is
     only safe in the context of who depends on it and what covers it.
   - **Git history** — `git log`/`git blame` the modified regions. A line that
     looks wrong in isolation is often a deliberate prior fix; conversely the
     history can reveal a re-introduced regression.
3. **Review against these lenses** — judge only what the diff changed (the
   confidence pass below drops the rest):
   - **Correctness & regressions** — logic errors, unhandled edge cases, races,
     broken error paths; and silent behaviour changes for existing callers:
     changed return types / shapes / nullability, altered persistence or
     propagation, breaking changes to a published contract or response shape.
   - **Error handling & resilience** — null / undefined and otherwise
     unexpected inputs; failures from external or dependency calls handled
     rather than swallowed; async rejections caught; batch operations that
     shouldn't abort wholesale when a single item fails.
   - **Security** — input validation and sanitisation; authorization and
     access-scoping; multi-tenant / data isolation; secrets kept out of logs
     and commits; injection-prone sinks.
   - **Test coverage** — what covers the changed paths; new branches and edge
     cases tested; existing tests still *valid* (not merely still passing)
     after a refactor. Report a missing test as a finding — say what to test
     and where it would live.
   - **Scope fidelity** — does the diff implement the linked stories'
     acceptance criteria, the whole criteria, and nothing beyond them?
   - **Conventions & standards** — style, architecture patterns, naming, and
     rules the repo defines (cite the written rule); plus type-safety escape
     hatches a typechecker won't flag (e.g. a cast that suppresses an error).
4. **Score each finding for confidence (single pass)** — for every candidate
   finding, assign a 0–100 confidence that it is a *real, change-introduced*
   issue, then **discard anything below 80**. Score down (and drop) findings
   that are:
   - pre-existing — on lines the diff did not change;
   - things a compiler/typechecker/linter would catch (CI runs separately);
   - pedantic nitpicks a senior engineer wouldn't raise, or a convention not
     actually written in the repo's rules;
   - likely-intentional changes related to the broader change;
   - silenced deliberately in-code (e.g. a lint-ignore with a reason).
   For a convention finding, confirm the repo rule actually names the issue
   before keeping it. The goal is a short list of high-confidence findings, not
   exhaustive nitpicking. Confidence (is it real?) is orthogonal to severity
   (how bad?) — score both.
5. **Rank and resolve the survivors** — tag each by severity: **blocking**
   (data loss, broken behaviour, security holes, crashes), **important**
   (behaviour regressions, missing error handling, test gaps for changed
   logic), or **minor** (style, docs, small cleanups); never file a nitpick as
   blocking. State each as location (`file:line`), what's wrong, why it matters
   (the concrete consequence), and the fix. Fix clear-cut defects; raise
   judgement calls with the user. Re-run the compile check after any fix.
6. **Verdict**: finish the stage only when no **blocking** or **important**
   finding is unresolved; minor findings may be accepted and recorded in the
   finish note.

## Rules

- A review that changes code re-runs the tests it might have invalidated.
- Findings the user explicitly accepts are recorded in the finish note rather
  than silently dropped.
