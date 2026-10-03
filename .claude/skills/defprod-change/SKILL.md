---
name: defprod-change
description: Drive a change record end-to-end through the DefProd pipeline — pull an accepted ticket from your tracker, create the change, then walk it stage by stage under your product's pipeline config until a human gate, a CI/CD handoff, or shipment. Use when starting any tracked piece of change work (feature, enhancement, or bug).
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Write
  - Edit
  - AskUserQuestion
  - mcp__defprod__listProducts
  - mcp__defprod__getProduct
  - mcp__defprod__getRepo
  - mcp__defprod__listTeamChanges
  - mcp__defprod__getEffectiveChangePipeline
  - mcp__defprod__assessChangeRisk
  - mcp__defprod__recordChangeDefect
  - mcp__defprod__confirmChangePipeline
  - mcp__defprod__createChange
  - mcp__defprod__listChanges
  - mcp__defprod__getChange
  - mcp__defprod__patchChange
  - mcp__defprod__startChangeStage
  - mcp__defprod__finishChangeStage
  - mcp__defprod__cancelChangeStage
  - mcp__defprod__cancelChange
  - mcp__defprod__reopenChange
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

# Drive a Change

The generic, type-blind orchestrator for change work. It owns the end-to-end
process — intake from your tracker, change-record creation, and the stage loop —
but does **no stage work itself**: each stage is performed (and stamped) by its
stage skill, and the loop yields to humans and CI/CD exactly where your
product's pipeline config says it should.

## When to use

- When starting a tracked piece of change work: `/defprod-change PROJ-123`,
  `/defprod-change <ticket-url>`, or bare `/defprod-change` for ad-hoc work.
- `/defprod-implement-feature` and `/defprod-fix-bug` are thin shims onto this
  skill with the type preset.

## Concepts

- **Change record** — a product-scoped work record in DefProd with a key like
  `CHG-07`, tracking intent, origin, and pipeline position.
- **Pipeline** — the product's enabled stages, a subsequence of:
  accept → design → define → code → test → review → merge → push → build →
  package → staging → ship. For a change in flight, read the one it is actually
  running under from `getChange` → `effectivePipeline`; the product's standing
  configuration is `getProduct` → `changePipeline` (entries
  `{stage, enabled, oversight}`; missing entries default to enabled with default
  oversight: accept=human; design/define/code/test/review/merge/push=agent;
  build/package/staging/ship=cicd). The two differ exactly when the repository has
  given the change's confirmed pipeline authority.
- **Oversight** — the level of oversight a stage runs under: `human`, `agent`,
  or `cicd`. Under `human` the agent still performs the stage; it simply may not
  finish it without a person's approval. The oversight map IS the orchestration
  policy. For stages backed by an agent stage-skill, the orchestrator translates
  the oversight into an execution **mode** it passes to
  the skill: `agent` → `autonomous` (run to completion, no questions),
  `human` → `interactive` (clarify as needed, and **never finish the stage
  without explicit human approval**). `cicd` hands the stage to CI/CD. Under
  `--unattended` a `human` stage is not dispatched at all — there is nobody to
  approve it, so the run parks there (see *Unattended runs*).

  The field was called `driver` until the server renamed it. An older server
  still reports and accepts only `driver`, so wherever this workflow reads
  `oversight` from a pipeline entry, fall back to `driver` when `oversight` is
  absent, and wherever it reports `oversight` on a stamp, retry with the same
  value as `driver` if the server refuses the field.
- **Risk assessment** — a scored severity / occurrence / detection vector with
  per-axis evidence, recorded on the change by `assessChangeRisk` (Step 5). The
  **agent scores; the server resolves**: the *risk category* (`low` | `medium` |
  `high`) is derived from the vector and is never an input. Assessment is an
  activity *within* `accept`, not a pipeline stage, and is re-scored at the
  `design` and `code` boundaries. Whether the category **selects** the pipeline is
  the repository's decision: where it grants confirmed pipelines authority, the
  change runs under the one confirmed for it; where it does not, the category is
  recorded and reported and selects nothing. You never have to work that out —
  `getChange` reports the pipeline in force and which of the two it came from.
- **Pipeline confirmation** — freezing onto the change, in full, the pipeline its
  category selected, via `confirmChangePipeline` (Step 5). The category is not an
  input here either: the server reads it from the change's own assessment and
  resolves the pipeline from it. A repo may declare bands it confirms
  **automatically** (`low` by default), in which case `assessChangeRisk` performs
  the confirmation itself and no separate call is wanted. Confirmation is a
  *record of what was selected*. Whether it is also **authority** is the repo's
  call: where the repo grants it, that frozen pipeline is what drives the change,
  and a later config edit cannot reach the change in flight; where it does not,
  the confirmation changes nothing about how the change is driven.
- **Oversight overlay** — a per-run `stage → oversight` map the caller supplies
  at invocation that takes precedence over the product's `changePipeline` oversight,
  **for this run only**. It is never written back to the product (no
  `patchProduct`) — it is a caller override, not a config edit. Resolution
  precedence (high → low): explicit per-stage override → shorthand → product
  config entry → built-in default. The overlay changes only the **resolved
  oversight**; the oversight→mode translation above and the stage skills themselves
  are unchanged. Constraints: it may set only `human`/`agent`, and only for
  skill-backed stages (design/define/code/test/review/merge/push) — it cannot
  hand a skill-less cicd stage (build/package/staging/ship) an agent to run, so
  such an override is ignored with a note. See **Oversight overrides** below.

## Oversight overrides (per-run)

The caller can override stage oversight for a single run without editing the
product's pipeline config. Pass overrides as invocation args alongside the
ticket ref (they coexist):

- **`--auto`** (alias `autonomous`) — flip every **skill-backed** stage to
  `agent`. The intent is **still confirmed with you** before the change is
  created, and the `accept` gate is preserved. Use when you're confident the
  build can run hands-off but still want to approve *what* is being built.
- **`--auto-all`** (alias `yolo`) — everything `--auto` does, **plus**
  auto-confirm the distilled intent and the `accept` gate: zero prompts from
  invocation to the CI/CD handoff (or ship). Use only when you're confident the
  change can go all the way through unattended.
- **`--interactive`** (alias `review-all`) — flip every skill-backed stage to
  `human` (each keeps you in the loop and won't finish without approval).
- **`<stage>=<oversight>`** — fine-grained per-stage override, space-separated
  and combinable with a shorthand; explicit pairs win. `<oversight>` ∈ {`human`,
  `agent`}. E.g. `--auto review=human` runs everything autonomously except
  review, which stays a human gate.

Examples:

```
/defprod-change PAY-12 --auto                 # loop autonomous; you still confirm the intent
/defprod-change PAY-12 --auto review=human    # autonomous except a human review gate
/defprod-change PAY-12 code=agent review=human
/defprod-change --auto-all                    # bare ad-hoc work, fully unattended
```

The same overrides flow through the `/defprod-implement-feature` and
`/defprod-fix-bug` shims. Invocation args, when present, **replace** any overlay
persisted from an earlier run; absent args, the persisted overlay stands.

**`--unattended` is not one of these.** It is a *run mode*, it sets no overlay,
and it refuses to be combined with one — see *Unattended runs* below.

## Unattended runs (`--unattended`)

A run with **nobody in the session**. It claims eligible work from your tracker,
drives it, and stops where your pipeline says a person is needed — leaving the
change **parked** for someone to pick up, or a **review item** where it could not
proceed at all.

```
/defprod-change --unattended             # claim eligible work and drive it
/defprod-change PROJ-123 --unattended    # drive one named ticket with nobody watching
```

### It is not an oversight override — and must not be combined with one

**`--unattended` sets no overlay at all.** It is documented next to the overrides
because callers reach for them together, and that is exactly the mistake:
`--auto` / `--auto-all` **override the risk-selected pipeline** — *"a `--auto` run
is autonomous whatever the risk category selected"* — so an unattended run wearing
that overlay drives a `high`-risk change straight past every human stage it has.
That is the inverse of what this mode is for: here the category is precisely what
decides how far the run gets.

- **Refuse to combine.** `--unattended` together with `--auto`, `--auto-all`,
  `--interactive` or any `<stage>=<oversight>` pair is a contradiction, not a
  preference. Stop and report it; never silently drop one of the two.
- **Ignore any persisted overlay.** An `oversightOverrides` object (or the
  legacy `driverOverrides`) left in `.defprod/change` by an earlier interactive
  run is **cleared**, not honoured — otherwise resuming a change unattended
  smuggles yesterday's `--auto` into a session with nobody in it.

What `--unattended` *does* take is the other half of `--auto-all`: **intake
consent**. The distilled intent is accepted as-is, the `accept` gate is not
prompted, and Step 5's pipeline confirmation is made without asking. That is not a
hole in the stop rule below — `accept` is `human` in every stock preset, so a rule
that stopped at the first `human` stage would halt every change before it began.
Intake consent is the orchestrator's; it was never a stage's oversight.

### Repository policy — three fields, read live

Take `repoId` from `.defprod/defprod.json` and call `getRepo`. Three nullish
fields govern this mode, and **each is conservative when unset**:

| Field | Unset | What it permits |
|---|---|---|
| `allowUnattendedLand` | **false** | Whether an unattended run may run the landing stages at all. |
| `claimUnownedWork` | **false** | Whether a ticket carrying no owner may be claimed. |
| `maxParkedUnattendedChanges` | **3** | How many of this runner's changes may sit parked before it claims no more. |

**Read them at the moment each decision is made, not once at run start.**
Withdrawal is an emergency brake and must bite work already in flight, exactly as
the withdrawal of `applyConfirmedPipeline` does.

**The direction is fixed and is not itself configurable.** This skill's own
default is to park; a repository field can only ever **loosen** from that. A host
that has not opted in is never moved — the only safe default for a skill that
installs everywhere, because an absent field means *nobody said*, not *it's fine*.

`maxParkedUnattendedChanges` is nullish and **`0` is a value**. Resolve it as
*unset → 3, otherwise as written*. `0` is the field's pause switch, and a `?? 3`
that swallowed it would make the one setting that stops the runner impossible to
express.

**An unreadable policy is never a permissive one.** Where the config carries no
`repoId`, where `getRepo` is refused, or where the server is too old to carry
these fields, take the two **permissions** as unset: park before landing, claim
nothing unowned. Failing open here would let a repository be moved by an agent
purely because a lookup failed.

The **cap is not a permission and does not default the same way.** Defaulting it
to 3 is meaningless when the same failure took away the `repoId` and `teamId` the
count is made with — a bound you cannot measure against is not a bound. That case
stops the run outright, at step 2 of *Claiming work* below.

### Claiming work

**This runs before Workflow Step 1, not inside it.** Claiming needs only the
repository — `repoId` straight from `.defprod/defprod.json`, and `teamId` from
the `getRepo` call above — so it does not wait on product resolution. It is the
claimed ticket that then feeds Step 1's ladder, exactly as a ticket named on the
command line would.

Skip the section entirely when the invocation named a ticket: a person choosing
the work **is** the claim.

1. **Count what is already parked.**
   `listTeamChanges { teamId, repoId, isCancelled: false }` — the cap is a
   *repository* field, so it spans every product in the repository, not any one
   of them. Keep the rows still active (not shipped, not cancelled) that **this
   runner created**: no field marks a change unattended, so *its* changes are the
   ones whose `createdBy` is the identity this run authenticates as. `getChange`
   each one, resolve the stage it is waiting on exactly as the loop does, and count
   it **parked** when that stage's oversight is `human` and it is either not
   started **or in progress**. The in-progress case is the review this mode
   prepares: its start is stamped and it is left open for a person, and a person
   already working it still holds one of this runner's places. An in-progress stage
   under `agent` oversight is live work, not a park, so it is not counted. A change
   a person has since carried past its human stage is no longer waiting on anyone
   here.

   Where the run genuinely cannot tell its own changes apart, count **every**
   parked active change in the repository instead. Over-counting only ever claims
   less, which is the safe direction; under-counting is how the cap is exceeded.

2. **If `parked >= maxParkedUnattendedChanges`, claim nothing and exit cleanly.**
   A full stop, not an error. With a cap of `0` the comparison holds from the
   start and nothing is ever claimed — that is the field doing its job, which is
   why `0` must never be read as unset.

   **If the count cannot be taken at all, claim nothing either** — a config with
   no `repoId`, a refused `getRepo`, no `teamId` to list by. The unreadable-policy
   rule above makes the *permissions* default to off; this is its counterpart for
   the *bound*, and it has to be stated separately because an uncountable cap is
   not a generous one. Exit `aborted`, naming the missing piece. Claiming on the
   strength of a bound you could not read is the one fail-open this mode cannot
   afford: nothing downstream would stop it.

3. **Ask the adapter for candidates** — `/defprod-change-tracker`, the
   `listClaimable` operation.

   **If the adapter has no `listClaimable` section, stop with `claimed-none`.**
   That skill is user-owned and is never overwritten once a team has edited it,
   so an older installation simply will not have the operation. Do **not**
   improvise a notion of "claimable" or "unowned" from the tracker directly:
   those are precisely the tracker-specific judgements the seam exists to hold,
   and guessing them is how an unattended run picks up somebody's work. Report
   that the adapter needs its fourth operation filled in.

   It reports each candidate's **ownership**; it does not decide eligibility.
   You do:
   - owned by the agent → claimable;
   - **unowned** → claimable **only if** `claimUnownedWork` is set;
   - owned by a person → never claimable.

   What counts as *unowned* is the tracker's own convention and therefore the
   adapter's business. Whether unowned counts is the repository's, and therefore
   yours.

4. **Drop candidates already spoken for**: a ticket the adapter reports as
   already promoted, and one whose `ref` appears in the `origin` field of an
   **open review item** in the queue below — which is why that template carries
   the ticket ref, so this check is a grep rather than a crawl through every
   change record. An open item is a question sitting in a person's court, and
   re-claiming its ticket re-grinds the work that raised it. (Step 3's
   `listChanges` dedupe still runs afterwards, on the one candidate taken, once
   Step 1 has resolved its product.)

5. **Take exactly one candidate**, the first eligible one in the order the
   adapter returned them, and hand it to Workflow Step 1 as the run's ticket.

   **One claim per invocation.** The skill does not loop over changes: it claims
   one, drives it, and exits with the outcome below. Claiming the next is the
   caller's job — it re-invokes, and the cap it re-counts at step 1 is what
   eventually makes that invocation a `claimed-none`. Keeping the loop outside
   means a run that wedges costs one change, not the whole queue, and it is why
   releasing the pin matters even though this skill never claims twice.

### The stop rule

Inside Step 6's loop, **before dispatching any stage**:

- **The next enabled stage is `review` and its resolved oversight is `human`** →
  **prepare the review, then park.** Call `startChangeStage { changeId, stage:
  'review', oversight: 'human' }`, then invoke `/defprod-change-review` with
  `mode=prepare` and `unattended`. It runs the review lenses over the diff and
  returns its findings **without finishing the stage and without changing code**.
  Put the findings in the interim commit (see *Parking*) and in the exit report,
  then park with `review` **in progress**. Never finish or cancel it: the person
  who picks the change up finishes `review` as a `human` stage, with the agent's
  findings to work from.

  This is the same record an attended review leaves. There, the stage skill stamps
  the start when the agent begins its pass, reports `human` (the oversight the
  stage actually receives), and then waits an unrecorded, open-ended time for the
  person's approval before it finishes. The prepare pass is the same work followed
  by the same wait, so it is stamped the same way. Leaving it unstarted would make
  a prepared review read as though none had happened.

  **Where `review` is already in progress under `human` oversight** when an
  unattended run reaches it (a named-ticket run on a change that is already
  parked), the review has been prepared. Park again without re-preparing it.
- **Any other next enabled stage whose resolved oversight is `human`** →
  **park**. Do not dispatch it, in any mode. The run exits cleanly, leaving the
  change active at that position.
- **The next enabled stage is `merge` or `push`** → park **unless**
  `allowUnattendedLand` is set on the repo, re-read at that moment. This holds
  whatever the band said: an all-`agent` band still parks before landing until a
  repository opts in. On a server that holds marked changes for a person (Step
  3), a change reaches this point with `review` already `human`, so the run parks
  at `review` first. This rule is the backstop for a change that carries no mark.
- **The next enabled stage's oversight is `cicd`** → hand off exactly as an
  interactive run does; where the pipeline has landing stages, they already went
  through the permission above. If the pipeline enables none and the run arrives
  here having pushed nothing, say so plainly rather than reporting a hand-off:
  there is nothing for CI to pick up.

**Parking needs no new state beyond the mark.** An active change whose next stage
carries `human` oversight, or whose `human` stage is in progress, **is** the parked
state, and `/defprod-change` resumes
from a change's recorded position. A park caused only by landing not being
permitted is the exception: its next stage is `agent`, so on its own it looks
exactly like a change in mid-flight, and the next session to pick it up would
land it without stopping. The `unattended` mark from Step 3 is what closes that
gap. With it, the server turns the change's `review` `human` for as long as
landing is not permitted, so every park before landing is also a park at a
`human` stage. Without it (an older server), say plainly in the exit report that
nothing will stop the next session at review.

A risk **rise** mid-run is this rule working, not fighting it: where the repo
grants the confirmed pipeline authority, a re-score into a stricter band can turn a
later stage `human`, and the run then parks there instead of driving on.

### Parking — and releasing the pin

Parking is three things, in order:

1. **Commit the outstanding work to the change's branch**, following the repo's
   commit conventions and the trailer rules in `/defprod-change-land` — except
   that this is an **interim** commit, so the trailer carries a stage ceiling,
   `Change: <product-slug>/CHG-NN:<last-finished-stage>` (probe for suffix support
   first, exactly as `defprod-change-design` documents). An unsuffixed trailer
   here would let a commit that delivered half a change walk it to `ship` forever.

   Where the run prepared a `review` (the stop rule above), put the findings in
   this commit's body under a heading such as `Agent review (unattended, not a
   sign-off):`, one line per finding with its severity and location, or `no
   findings`. The reviewer reads them from the branch. They are the agent's pass,
   never the human review itself.

   **Do not merge, do not push, do not open a pull request.** Those are the
   landing stages, and parking is what happens because they were not permitted.
   The branch therefore stays on the machine that made it; publishing it so a
   reviewer elsewhere can see it is that host's arrangement, not a permission this
   skill has.

2. **Release the pin.** The pin holds *pin present ⇔ a change is hands-on in
   this worktree*, and a parked change is hands-on for a **person, elsewhere** —
   so the invariant says release it. This is not bookkeeping: hold the pin and
   Step 4 refuses the next claim, and the runner deadlocks after exactly one
   change. **The pin is a lock; the bound on work-in-progress is
   `maxParkedUnattendedChanges`.** Conflating them makes the bound an accident of
   locking rather than a decision.

   **Release it the way the installation releases it**, per the same rule
   `/defprod-change-land` applies at the land hand-off: deleting
   `.defprod/change` is right only when the pin was written by hand. Where the
   installation's own tooling wrote it, call *that* tooling's release step, so
   everything else bound to the change — the worktree or environment, an isolated
   database, a running session — goes with it. Deleting the file directly there
   half-releases, and an unattended runner that half-releases leaks the binding on
   every park with nobody watching. `defprod-change/SKILL.local.md` is where an
   installation records that it owns the pin's lifecycle.

3. **Report the park**: the change key, the stage it is parked at, why (`human`
   oversight, or landing not permitted), and the branch carrying the work.

### Blocked mid-stage — raise a review item

Where a run cannot proceed *within* a stage — a judgement call it must not make
alone, or an obstacle it cannot clear — the stage raises a **review item** and the
change parks. Review items are this mode's **output**, never its input: a ticket
is a unit of work to consider, while a review item is one fork an agent is blocked
on, and implementing one unattended would bypass the very decision the record
exists to request.

A review item is a markdown file in the repository's review queue —
`reviewQueuePath` from `.defprod/defprod.json`, default `docs/reviews/` (create
the directory if it is absent):

```
---
id: REV####                 # next above the high-water mark of existing files
title: <one line, written for someone who was not here>
created: YYYY-MM-DD
kind: decision | action     # a question for a person, or a task only a person can do
raised_by: defprod-change/<stage>
context: <product-slug>/CHG-NN
origin: <tracker ref the change came from, or blank for ad-hoc work>
status: open
resolution:
---

## What's needed
<the question or the task, stated so a cold reader knows exactly what to settle>

## Context
<everything a fresh session needs: files, commands run, exact output, what was
already tried and ruled out, and why this was not decided here>
```

Name it `REV####-<short-slug>.md` and commit it with the parked work — it travels
with the branch the question is about.

**Then put the item's whole body in the exit report, not just its path.** A
parked branch is not pushed (landing was not permitted, which is why the run
parked), so on a host with nobody on it the file reaches no one: the report is
the only thing that leaves. A path alone hands the reader a location they may
not be able to open. Reproduce `## What's needed` and `## Context` verbatim, then
name the id and path so the two can be reconciled once the branch is picked up.

**The stage that raised it calls `cancelChangeStage`, never `finishChangeStage`.**
A stage abandoned mid-flight is not a completed one, and a finish stamp on work
that stopped is the worst of the three records available.

### Exit report

Every unattended run ends by naming one outcome, then the detail behind it:

| Outcome | Means |
|---|---|
| `claimed-none` | Nothing eligible, or the parked cap is reached. Not an error. |
| `parked` | The change is active at a stage a person must take. Name change, stage, reason, branch, and — where the run prepared the review — the agent's findings. |
| `blocked` | A review item was raised. Name the change, and reproduce the item's body as above — its id and path alone are not the report. |
| `handed-to-cicd` | The pipeline reached its CI/CD boundary. |
| `shipped` | The pipeline ran to completion. |
| `aborted` | A precondition failed. Name what a person must do. |

**An unattended run never asks a question and never forces a lock.** Where Step
1's resolution ladder ends at *ask the user*, and where Step 4 finds the worktree
pinned to a **different active change**, the run **aborts** — it does not guess a
product and it does not `--force`. A contradictory invocation (above) aborts the
same way.

## Workflow

### Step 1 — Resolve the product

Identify the DefProd product for this repo. One repo may host **several**
products (a monorepo), so resolve by this ladder — first match wins:

1. **Config fast path.** Read `.defprod/defprod.json` (the committed, non-secret
   config — also the home of `apiUrl`, `repoId`, and layout hints). If it pins a
   single `productId`, use it — the common single-product case, unchanged.
2. **Repo linkage.** Otherwise take `repoId` from the config and list the team's
   products (`listProducts`), keeping those whose `repoId` matches. This is the
   repo↔product linkage — the product carries its own `repoId`/`repoPackagePath`.
   - Exactly one candidate → use it.
3. **Narrow by package path.** More than one → narrow by `repoPackagePath`:
   keep candidates whose path is the sub-tree the change's files live under.
4. **Narrow by working area.** Still ambiguous (two products share a
   `repoPackagePath` — e.g. two apps served from one package) → narrow by the
   route/area the changed files belong to.
5. **Ask.** Still ambiguous, or no `repoId`/candidates at all → `listProducts`
   and ask the user, presenting candidates by **slug** and name.

Fetch the resolved product with `getProduct`, **note its `slug`** (carried into
the change context in Step 4 for the land trailer), and resolve the pipeline
config as above.

Then resolve the **oversight overlay** from the invocation args (see *Oversight
overrides*). Apply it on top of the pipeline config and **echo the effective
oversight map**, marking overridden stages (e.g. `review: agent * ← was human`),
then proceed — no confirmation prompt. Ignore (and note) any override targeting
a skill-less cicd stage.

Under **`--unattended`** there is no overlay to resolve: echo the pipeline as the
server resolved it, clear any `oversightOverrides` (or legacy `driverOverrides`)
persisted in the pin, and — where this ladder reached rung 5 — **abort** rather than ask, because nobody is there
to answer. The ticket in hand is the ladder's input at rungs 3–4: take the
package or area it names, and abort only if it genuinely leaves the product
ambiguous. The repository's unattended policy has already been read by then
(*Unattended runs* runs before this step on a claiming run, and a named-ticket
run reads it here) — and it is re-read at each decision it governs, never cached
for the run.

### Change-key qualification (single- vs multi-product repos)

This resolution ladder settles more than which product owns the change — it
also settles this repo's **cardinality**, which gates every *prose* rendering
of a change key from here on:

- **Single-product repo** — the config fast path (rung 1) pinned one
  `productId` outright, or the repo-linkage lookup (rung 2) returned exactly
  one candidate before any narrowing was needed.
- **Multi-product repo** — the repo-linkage lookup (rung 2) returned **more
  than one** candidate, and rungs 3–5 narrowed to the one in scope for this
  change.

**The rule:** in a multi-product repo, every *prose* rendering of a change key
that this pipeline authors — commit subject/body, PR title/body,
`defprod-change-tracker` link/close write-backs, and any change-key mention
inside the change record's own narrative content (the `intent` composed below,
or the `design` field a stage skill records) — renders qualified,
`<product-slug>/CHG-NN`. In a single-product repo, all of those render **bare**,
`CHG-NN`, full stop. Echo which case applies once, alongside the product
resolution (e.g. "3 products share this repo — change-key mentions qualify as
`defprod/CHG-NN`" or "single product repo — change-key mentions stay bare"),
then proceed without asking.

**This does not govern the commit trailer or the branch name** — both stay
unconditionally qualified whenever a product resolves (Step 4), unchanged
since v1.7.0/v1.13.0: CI's stamping (`defprod-stamp.sh`) depends on that fixed
form regardless of cardinality.

**This is the single statement of the rule.** Persist the determination as
`multiProduct` in the `.defprod/change` pin (Step 4) so stage skills read it
rather than re-deriving cardinality themselves (most don't carry `listProducts`
in their tool allowlist). Every site that renders a change key as prose
cross-references this section rather than restating the policy — one edit
here, not N.

### Step 2 — Fetch the ticket (intake)

- **With a ticket ref/URL**: fetch it via the **`/defprod-change-tracker`**
  adapter skill (the user-owned skill that knows how to talk to your tracker).
  If the adapter is unfilled or absent, fall back to asking the user to paste
  the ticket content or describe the work — except under **`--unattended`**,
  where there is no one to ask: exit `aborted`, naming the ticket the adapter
  could not fetch. Note this is `aborted`, not the `claimed-none` an absent
  `listClaimable` produces: there being no work to claim is a normal, quiet
  outcome, while a ticket that exists and cannot be read is something broken.
- **Bare invocation**: this is ad-hoc internal work — compose the intent
  interactively with the user.

Classify the **type** (`feature` | `enhancement` | `bug`) from the ticket or
ask. Distill the **intent** (markdown: what we are changing and why — the
accepted decision, not a paste of the ticket) and **confirm it with the user
before creating anything**. Under **`--auto-all`** or **`--unattended`**, skip
this confirmation and the `accept` gate — the distilled intent is accepted as-is
and recorded as the change's intent. `--auto` does **not** skip it: you still
confirm *what* is being built. An `--unattended` run never reaches the ad-hoc path:
it always arrives holding a ticket ref — the one it was invoked with, or the one
*Claiming work* already selected before Step 1 — because there is no one to
compose an intent with. If the intent text mentions another change (a duplicate, a
superseded predecessor, a related sibling), render that mention per
*Change-key qualification* above.

### Step 3 — Dedupe, then create the change

For external tickets, check for an existing promotion first:
1. Ask the adapter whether the ticket already carries a DefProd change link.
2. Call `listChanges { productId, originSystem, originRef }`.

If an **active** change exists → resume it (re-establish context, continue at
Step 6) instead of creating a duplicate. If a **cancelled** one exists →
prefer `reopenChange` over creating anew. The server independently rejects
duplicate creation against active changes.

Otherwise call `createChange` with `{ productId, title, type, intent,
source: 'external' | 'internal', origin: { system, ref, url } }`.

**Under `--unattended`, mark the change as unattended work.** Pass
`unattended: true` to `createChange`. Where dedupe resumed an existing active
change instead, mark it with `patchChange` — `[{ op: 'add', path: '/unattended',
value: true }]` — before dispatching any stage. The mark is one-way (the server
never lets it be cleared), and it is what lets the server hold the change for a
person: while the repository does not set `allowUnattendedLand`, the pipeline in
force for a marked change requires `human` oversight at `review` (or, with
`review` off, at the first enabled landing stage), whatever its risk band says,
and `getChange` reports `unattendedReviewGate: true`. That requirement is
recorded on the change, so it binds whoever picks the change up next, not just
this run.

**An older server does not know the field.** If `createChange` or `patchChange`
is refused for naming `unattended`, retry without it and carry on, but say so in
the exit report: the change carries no mark, so a later session will read its
pipeline as the band left it and may land it without stopping for a person.

**`origin` and `source` are independent — never condition one on the other.**
They answer different questions:

- **`origin` — where is this work written down?** Pass the origin the adapter
  returned from `fetch`, **whenever intake fetched a ticket at all**, whatever
  its source. Omit it only for genuinely ad-hoc work — a bare invocation with
  no ticket behind it — because there is then nothing to point at.
- **`source` — did this work come from outside the team?** `external` for a
  customer, a partner, an outside report; `internal` for the team's own
  backlog. A team's own tracker is still a tracker, so `internal` **with** an
  origin is the normal case, not a contradiction.

A change promoted from a ticket but carrying no `origin` is a defect, not a
tidy omission: the ticket records the change key, the change records nothing,
and the change's own detail view has no origin to show. Treating
`source: external` as the trigger for setting `origin` is the specific mistake
that produces it.

### Step 4 — Establish the change context

Make the change discoverable by stage skills and CI hooks:

1. Write **`.defprod/change`** (git-ignored; add to `.gitignore` if needed) in
   the worktree root:
   `{ "productId": "...", "changeId": "...", "changeKey": "CHG-NN", "productSlug": "...", "multiProduct": true|false }`
   (`productSlug` is the resolved product's slug from Step 1 — the land stage
   uses it for the `Change: <slug>/CHG-NN` trailer without a round-trip.
   `multiProduct` is the cardinality determination from *Change-key
   qualification* above — stage skills read it directly instead of
   re-resolving cardinality.)
   **The pin is a lock, not a claim.** Before writing it, read any existing
   `.defprod/change` and check it: if it names a **different** change, `getChange`
   it — if that change is still **active** (not shipped/cancelled), this worktree
   is already hands-on for another change. **Refuse** to overwrite the pin and
   stop with a clear message (the other change's key + its branch), unless the
   operator explicitly forces it (`--force`) — **an unattended run never forces
   it**, it aborts, because there is no operator there to own the collision. A leftover pin from a
   **shipped/cancelled** change is stale — replace it freely. This makes two
   sessions unable to silently share one tree: the second change must be forced,
   or belongs in a separate worktree/branch. If an oversight overlay was resolved
   in Step 1, persist it here too as an `oversightOverrides` object (e.g.
   `"oversightOverrides": { "review": "human", "code": "agent" }`) so the override
   survives a CI/CD-handoff → resume cycle. A pin written before the rename
   carries it as `driverOverrides`: read that as the same thing, and write it
   back under the new name. It is cleared with the pin on
   ship/cancel. Under `--unattended` persist `"unattended": true` instead (that
   mode has no overlay to persist), so a stage skill can tell that nobody is in
   the session. It goes with the pin when the change parks — whoever resumes it
   is, by then, in the session.
2. In branch-based flows, create the branch **`chg/<product-slug>/CHG-NN-<short-slug>`** — the `<slug>/CHG-NN` tail matches the commit trailer (D23). (Legacy bare `chg/CHG-NN-<short-slug>` branches are still recognised by the stage skills and CI.)
3. (Commits made later by `/defprod-change-land` carry the
   `Change: <product-slug>/CHG-NN` trailer.)

Then perform the **link write-back**: ask the adapter to mark the ticket
promoted with the change key (`link` operation).

**Then confirm the link points both ways.** A promotion is one fact recorded in
two places, and only one of them is a deliberate step — so the forward half is
the half that goes missing. Re-read the change (`getChange`) and check its
`origin` matches the ticket you just linked. If it is absent, the create call
dropped it: repair it now with `patchChange` — `/origin` is patchable, unlike
the lifecycle and provenance fields around it — rather than leaving a one-way
record. If your client cannot pass nested arguments through an MCP tool call
(some mangle objects in transit), reach for whatever direct RPC path the repo
has instead of giving up and proceeding with no origin.

**Do it here, at the boundary — not later.** This check is worth the round-trip
because the repair window closes: a **shipped change is frozen**, and
`patchChange` refuses it outright with no override. Nothing between here and
ship catches a missing origin, so the change sails through, and by the time a
human notices the empty origin on the change detail and asks where the work
came from, the record can no longer be corrected at all.

### Step 5 — Assess the risk (an activity within `accept`)

The change now exists and has an id, so score it. Assessment is an **activity
within `accept`, never a pipeline stage** — as a stage, every later re-score
would be a backward jump, and the backward-jump rule would wipe the stamps of
design, define, code and everything after them, so re-assessing would destroy the
record's own history. This step therefore calls no stage-action tool, moves no
position, and disturbs no stamps.

**If assessment is unavailable, skip it and continue.** `assessChangeRisk` is
gated by a feature flag and exists only on servers new enough to carry it. A
rejection — flag off, tool absent, insufficient scope — is **not** a pipeline
failure: note it in one line and go to Step 6. Risk assessment never blocks
change work.

1. **Do the lookups before scoring.** `assessChangeRisk`'s own tool description
   states what each axis's evidence must contain — read it and satisfy it rather
   than guessing. An axis without evidence is rejected, because an unsupported
   score is not an assessment. Two errors are worth naming because they invert
   the outcome:
   - **Detection is INVERTED** — `1` = certain to be caught before a customer
     sees it, `10` = silent.
   - **An unfamiliar subsystem scores occurrence HIGH, not low.** Unfamiliarity
     is an argument for a high occurrence, never a reason to shrug one off.
2. **Score what is knowable at `accept`, and be explicit about what is not.**
   Here the change is still an intent string: no diff exists, so the modules
   touched and the coverage over the changed path are unknown. This is the
   **least-informed score the change will ever have** — expected, not a defect.
   Use the prior art in this repo, `docs/rules` (or this repo's equivalent), and
   the incident record for occurrence; for detection, say plainly that no diff
   exists yet and name whatever coverage exists over the surface the intent
   implicates — or record that none does. **Do not invent diff-level evidence you
   cannot have at this point.**
3. **Use the `core` overlay.** Whether a migration is involved is usually not
   known at `accept`, so the initial assessment is scored against the core
   anchors and omits `writeSet` / `sideEffects`. The `dbm` overlay — which
   requires both declarations, empty arrays permitted but stated explicitly —
   belongs to a later re-score, once a migration is known to be in play.
4. **Never supply a category.** It is not an input: the server derives it from
   the vector by a severity-weighted lookup and rejects a supplied one. The only
   way to move the category is to move an axis **and** write the evidence for
   that axis. Compute one locally to *report* if you wish — you simply cannot
   assert one as stored truth.
5. **Report the resolved category** from the response
   (`riskAssessment.category`), with the vector and a one-line reason per axis.
6. **Report the pipeline the category selects, and whether it governs.** Call
   `getEffectiveChangePipeline { productId, riskCategory }` for the resolved
   category, and read `effectivePipeline` / `effectivePipelineSource` from the
   change itself for what is actually in force, then show the difference between them
   — or state that there is none.
7. **Confirm the pipeline** — freeze onto the change the pipeline its category
   selected, so the record still answers *"what oversight did this change
   receive?"* after the configuration has moved on.

   **First check whether it is already done.** Where the repo auto-confirms the
   resolved band — `low` unless it says otherwise — `assessChangeRisk` performed
   the confirmation itself, and the response comes back with
   `confirmedPipelineCategory` already set. Report that it was confirmed
   automatically and **do not call `confirmChangePipeline`**; a second call would
   add a duplicate entry to the change's history for a confirmation that did not
   change. Auto-confirmation is the whole reason a routine change costs no human
   touchpoint here — do not reintroduce one.

   Otherwise the band needs confirming explicitly. Call
   `confirmChangePipeline { changeId }`. Supply **no** category and **no**
   pipeline: both are resolved server-side from the change's own assessment,
   which is what stops a caller confirming lighter oversight than the evidence
   earned. Pass per-stage `overrides` only if the repo has opted into them —
   where it has not, supplying them is **rejected**, not ignored, so a refusal
   naming `allowConfirmedPipelineOverride` means the repo is in its strict
   default, not that you called it wrongly.

   **Consent follows the oversight, as everywhere else.** With a human in the loop
   (default, and under `--auto`, which preserves the `accept` gate), present the
   pipeline and confirm it with them before calling — this is the one moment
   the design intends a human to see the oversight level before it is recorded.
   Under `--auto-all` or `--unattended`, confirm without prompting — in both,
   intake consent was already given at invocation.

   **If confirmation is unavailable, note it and continue**, exactly as for
   assessment: the tool is gated by the same flag and exists only on servers new
   enough to carry it. It never blocks change work.

**Whether any of this reshapes the run is the repository's decision, not yours to
infer.** `getChange` reports `effectivePipelineSource` — `confirmed` (the frozen
confirmation governs), `assessed` (no confirmation stands, so the stricter of the
configuration and the current category governs), or `configuration` (the standing
config governs, because the repo has not granted authority or the change carries no
assessment). Report which one applies, in those terms, so the reader is never left
guessing whether a category they can see actually gated the change.

Where the source is `configuration`, say so plainly and run Step 6 exactly as if
no assessment had been made — that is the calibration posture, in which the value
is in the assessment being recorded and read rather than applied.

A confirmed pipeline does **not** change this. It records *what the category
selected*; the run still proceeds under the pipeline in force. Confirming and
applying are separate, and only the first exists today.

Assessment itself needs no gate in any oversight mode — nothing is applied, so there
is nothing to approve. Report and continue under `--auto`, `--auto-all` and
`--interactive` alike. Confirmation is the exception noted above, because a human
choosing an oversight level is the point of it.

### Step 6 — The stage loop

Repeat until the pipeline ends or control leaves the agent:

1. Fetch the change (`getChange`) and **take the oversight map from its
   `effectivePipeline`** — the pipeline this change is actually running under,
   already resolved by the server, with `effectivePipelineSource` naming which of
   the three it came from. Re-read it **every iteration**: never plan the run
   upfront, because the change's position, its assessment and the repo's policy
   can all move underneath you — including the withdrawal of authority, which is
   an emergency brake and must bite in flight.

   Where `effectivePipeline` is **absent** — an older server that predates it —
   fall back to the product's pipeline config (`getEffectiveChangePipeline`, or
   `getProduct` → `changePipeline`) exactly as before, and treat the source as
   `configuration`. Never synthesise the field from a repo setting yourself: the
   ladder that produces it is the server's, and guessing it is how a run ends up
   driven under a pipeline nobody selected.

   Then **re-apply the run overlay** (the resolved overrides, persisted in
   `.defprod/change` as `oversightOverrides`) so the override is honoured every
   iteration and survives a resume. The overlay is the caller's, so it still sits
   on top — a `--auto` run is autonomous whatever the risk category selected.
   **Under `--unattended` there is no overlay**, by design: the whole point of
   that mode is that the category governs, so the resolved pipeline stands as the
   server gave it.
2. Determine the stage to take: the current stage itself when it is **in
   progress** (a parked `review` left open for a person, or work a previous
   session did not finish), otherwise the next enabled stage after the current
   position. Re-dispatching an in-progress stage re-stamps its start harmlessly,
   because the first report wins.
   - No next stage → the change is shipped or at pipeline end; go to Step 7.
3. **Under `--unattended`, apply the stop rule first** (see *Unattended runs*):
   a next stage whose resolved oversight is `human` **parks the change** and ends
   the run without dispatching it, and `merge`/`push` park too unless
   `allowUnattendedLand` is set on the repo, re-read now. Only where the stop
   rule does not fire does the run continue into the dispatch below.
4. Consult that stage's **oversight** and act:
   - **`cicd`** → END the run. Report that the change is handed to the
     CI/CD pipeline (its hooks stamp `finishChangeStage` from here — see
     `defprod-stamp.sh` in defprod-scripts).
   - **`agent`** or **`human`** on a **skill-backed** stage → **call
     `startChangeStage { changeId, stage, oversight }` yourself, immediately
     before dispatching**, passing the overlay-resolved `oversight` (retried as
     `driver` if an older server refuses the field) — then invoke the stage's
     skill, passing the change type **and the mode**: `agent` → `mode=autonomous`,
     `human` → `mode=interactive`. Under `--unattended`, also pass `unattended`
     alongside the mode — the resolved oversight is always `agent` there (the stop
     rule caught the rest), and the flag is what tells the stage that a blocker
     must become a review item rather than a question or a guess.

     If you attach a `commitSha` to any stamp, it is the **full** sha
     (`git rev-parse HEAD`), never `--short` — the stage skills say the same, and
     the CI hooks already send the full form.

     **Dispatching `merge` or `push` unattended, say so explicitly**: pass
     `allowUnattendedLand=true` with the mode. Reaching that dispatch at all
     means the stop rule re-read the field and found it set, and `/defprod-change-land`
     will not land without being told — it has no `getRepo` of its own, and an
     unstated permission is not one. This is the clause that makes D64's *flip a
     field* actually flip something: without it, a repository that opted in still
     parks.

     **Why the orchestrator stamps the start, when the skill stamps it too.** Only
     the stage skills used to call it, so a skill that was substituted, run
     standalone, or bypassed left no start time and no trace that one was owed. A
     stage with no start does not merely read as unknown: it contributes to
     **neither** the work nor the wait half of the stage-time breakdown, so it
     vanishes from the analytics entirely — and because nothing refuses the finish,
     the gap accrues silently for as long as the harness keeps misbehaving. Measured
     across a real deployment it reached a double-digit percentage of all finished
     stages. This is **added, not moved**: the stage skills keep their own start
     stamp because they are documented as working standalone. Double-stamping is
     free — the tool contract is first-write-wins — and the orchestrator is also the
     only party that knows the per-run oversight overlay, so its stamp is the one
     that can report the oversight correctly. In interactive mode the skill keeps the human
     in the loop and will **not** `finishChangeStage` without explicit approval,
     so a human gate is honoured *inside* the stage — not by stopping the loop
     before it. (This replaces the earlier "human → stop the loop" rule **for a
     run with a person in it**: the approval-before-finish gate is the better
     control point precisely because they are asked with the proposed result in
     hand. That reasoning has no referent under `--unattended`, where the stop
     rule in step 3 applies instead and a `human` stage is never dispatched at
     all.) After the stage finishes, continue the loop — re-read the config and
     consult the next stage's oversight.
   - **`human`** or **`agent`** on a **skill-less** stage (`build`, `package`,
     `staging`, `ship` — CI/CD territory) → nothing for the agent to run: hand
     off as for `cicd`, or STOP and report if a human must act.

     | Stage | Skill |
     |-------|-------|
     | design | `/defprod-change-design` |
     | define | `/defprod-change-define` |
     | code | `/defprod-change-code` |
     | test | `/defprod-change-test` |
     | review | `/defprod-change-review` |
     | merge, push | `/defprod-change-land` (handles both) |
     | build, package, staging, ship | no skill — cicd territory |

     **Capability dispatch**: if the repo has a local override skill for the
     stage (e.g. a project-specific `change-test`), prefer it. An override skill
     **must stamp the stage it runs**, exactly as the standard stage skills do —
     `startChangeStage` on entry and `finishChangeStage` when the stage's
     done-condition is met, reporting `oversight`. This is an obligation on the
     override, not a property you may assume of it: the orchestrator's own start
     stamp above covers the start, but nothing else covers the finish, so an
     override that does not stamp leaves the change parked at a stage it has
     actually completed.
5. The stage skill stamps its own start and finish as well. The orchestrator
   stamps only the **start**, per step 4 — it never calls `finishChangeStage` for
   stage work it delegated, because only the skill knows whether the stage's
   done-condition was actually met (and, in interactive mode, whether the human
   approved it).
6. **If a stage reported itself blocked** rather than finished — an unattended
   run's stage skills raise a review item instead of guessing — do not advance
   and do not retry. Park the change (see *Blocked mid-stage*) and end the run
   reporting `blocked`.
7. **If the stage that just finished was `design` or `code`, re-assess the risk**
   before continuing the loop — see *Re-assessment at the design and code
   boundaries* below.
8. **If the stage that just finished was `code` and the change is a `bug`,
   classify the defect it repaired** — see *Defect classification after `code`*
   below. Then continue at 1.

#### Defect classification after `code`

A `bug` change repaired a defect. Record what kind it was, with
`recordChangeDefect`, once the diff exists.

**An activity, never a pipeline stage** — for the same reason risk assessment is
one. It calls no stage-action tool, moves no position and disturbs no stamps.
Anchored to the `code` boundary rather than to a stage because pipelines are
per-band configurable: hanging it off `review` would skip it for exactly the
low-risk changes whose defect data is most worth having.

**After `code`, not at `accept`.** Half the fields need the diff — what had to be
fixed, and the nature of the mistake, are fiction before the fix exists.

**It never blocks.** A tool that is absent, refused, or on a server too old is one
line of note and then carry on. Nothing here may stop a fix shipping.

1. **One classification, the most severe defect.** Where the fix repaired
   several, rank by who they affected — customer beats staff beats developer —
   and break ties by impact severity. Record that one and let the rest go. The
   number this produces is *customer-facing defects*, not *defects*.
2. **Score impact severity on the risk rubric's Severity anchors**, the same ones
   used to predict a change's severity, and **within the audience you named**.
   The anchors assume a customer, so a staff-only defect that breaks a core admin
   workflow reads as an 8 and overstates its business impact badly.
3. **`foundBy` is what actually exposed it**, not what should have. It is the
   cheapest field here — you already know how you found the bug — and the one
   that says which gate is worth investing in.
4. **Give the commit, not the change.** `git bisect` / `git log -S` / `blame`
   returns a SHA; pass it as `introducedInCommit`, pass that commit's `Change:`
   trailer as `introducedInCommitTrailer`, and let the server derive the change
   link and the dormancy. Do not look a change up yourself. Where the defect is in
   code that never worked there is no introducing commit — record `age: base` and
   omit both rather than guessing.
5. **It can be amended after ship**, unlike every other content field. Realised
   impact is often only learned once a customer explains what actually happened,
   so a later correction is the system working; the history is kept in the change
   event trail.

#### Re-assessment at the design and code boundaries

Re-scoring is **mandatory at the end of `design`** — scope and approach are now
known — and **at the end of `code`**, where the diff exists and the rubric can
finally run its lookups for real. If either stage is disabled or skipped in this
product's pipeline, its boundary simply doesn't exist; re-assess after each of the
two that actually ran.

It is the same `assessChangeRisk` call as Step 5, with the same evidence
discipline and the same tolerance: if assessment is unavailable, note it in one
line and carry on. Re-scoring is always safe — the current assessment is replaced,
the full history is kept in the change's event trail, and no stage position or
stamp is touched. **The orchestrator owns this**, not the stage skills: keeping it
here means it still happens when a repo substitutes its own local override skill
for a stage, and it cannot double-score. A stage skill invoked standalone
therefore does not re-assess.

Three things differ from the `accept`-time score:

1. **The lookups are real now — do them properly.** At `code`-end especially, run
   coverage over the changed paths and name the specific test that would fail on
   the failure mode, or state plainly that none exists. This is the assessment the
   earlier one was standing in for, so **do not carry forward `accept`-time
   evidence strings the diff has since superseded** — rewrite them.
2. **The overlay may change.** The initial score is always `core`, because whether
   a migration is involved usually isn't known at `accept`. Once one is in play,
   switch to the `dbm` overlay — at which point `writeSet` **and** `sideEffects`
   are both required, empty arrays permitted but stated explicitly rather than
   omitted, and reversibility is derived from those declarations and overrides your
   authored value unless you record a justification.
3. **Report the movement, not just the value.** Compare against the previous
   category and say whether it rose, fell, or held.

**Application is asymmetric — a ratchet on the band, not on individual axes.**
Raising oversight is always safe to automate; lowering it is precisely what a
human was consulted about.

- **Category rose** → the earlier pipeline choice was made against a risk picture
  now known to be wrong, so re-selection is **forced**: the stricter preset
  applies and is **announced**, not asked about, in every oversight mode.
- **Category fell** → the pipeline **does not relax**. You may *propose*
  relaxation to a human; you may never apply it, because an agent does not reduce
  oversight below an explicit human choice. Under `--auto`, `--auto-all` or
  `--unattended` a fall is **recorded and ignored** — no prompt, no relaxation;
  in the last of those there is nobody to propose it to. A fall is often
  legitimate and earned — adding a test that fails on the identified failure mode
  is the cheapest way there is to lower detection — which is why falls travel the
  proposal path rather than being discarded.

**Whether a rise reshapes the run depends on the repository, and you do not have
to work it out.** Re-read `effectivePipeline` after the re-score: where the repo
grants authority, a rise genuinely tightens the run — the confirmation it cleared
drops the change onto the stricter of the configuration and its new category, so
the remaining stages are driven under that from the next iteration onward, and you
announce it rather than asking. Where the repo has not granted authority, the
source stays `configuration`: state that the category rose, name the stricter
pipeline it *would* have selected, and continue under the pipeline in force. The
fall branch is a proposal in both cases.

**Re-confirm when the category moves, because the server invalidates.** A
re-assessment that lands the change in a band the repo does *not* auto-confirm
**clears** any existing confirmation — otherwise a change assessed `low`,
auto-confirmed, then re-scored `high` at the `design` or `code` boundary would
carry a record reading "low oversight confirmed" against high-risk work. So after
a re-score that moved the category:

- **Into an auto-confirmed band** → the server has already replaced the
  confirmation. Report it; call nothing.
- **Into any other band** → the confirmation is now cleared, and the change has
  none. Confirm again per Step 5's step 7, with the same consent rule.

This is what makes the rise branch's *"re-selection is forced"* concrete: the
record is re-made against the risk picture now known to be right — and where the
repo grants authority, re-confirming is also what puts the change back under a
pipeline it has deliberately selected rather than the interim floor.

One signal worth surfacing: if the category **rose to `medium` or `high` at
`design`-end**, the design was conducted below the depth floor that category now
implies, which is worth saying out loud because it suggests the approach may have
been under-discussed. It does not retroactively re-run the design stage.

### Step 7 — Terminal write-back

When the change reaches `ship` finished (or is cancelled), ask the adapter to
write the outcome back to the ticket (`close` operation). Best-effort: a
failed tracker write is reported, never blocking.

On cancellation (`cancelChange`), also **delete `.defprod/change`** — a
cancelled change is no longer hands-on in the worktree. (On `ship`, the pin was
already cleared at the land hand-off, Step 6 / `change-land`.)

**A parked change gets no close write-back.** Parking is not an outcome — the
change is still active, and the ticket is still promoted, not done. The `close`
operation belongs to `ship` and to cancellation alone.

## Rules

- **Re-consult the oversight map every iteration.** A human gate must never be
  steamrolled because an earlier plan said "continue". The gate now lives in
  the stage skill's **interactive** approval-before-finish, so "continue" means
  re-reading the next stage's oversight and invoking its skill with the right
  mode — never advancing a `human`-oversight stage to `finished` unprompted.
- **`--unattended` sets no oversight overlay, and never combines with one.** The
  overlay outranks the risk-selected pipeline, so an unattended run wearing
  `--auto` drives a `high`-risk change past every human stage it has — the exact
  inverse of the mode's purpose. It takes intake consent only, which is the
  orchestrator's and was never a stage's oversight; that is also why `accept` being
  `human` in every preset does not halt the run on the spot.
- **Unattended: a `human` stage parks the change; the repo's fields only ever
  loosen.** Park before `merge`/`push` unless `allowUnattendedLand` is set, claim
  an unowned ticket only where `claimUnownedWork` is set, and claim nothing once
  `maxParkedUnattendedChanges` is reached — `0` is a value, not an absence. Read
  all three **live**, so withdrawal bites work already in flight. Absence of a
  field means *nobody said*, which is never permission.
- **Unattended: mark the change, and prepare a human review rather than skip it.**
  Pass `unattended: true` at creation (or patch it on when resuming), so the
  server holds the change for a person while landing is not permitted. At a
  `human` `review`, stamp its start with `oversight: human`, run it in `prepare`
  mode, record the findings in the parked commit, and leave the stage in progress
  for the person who finishes it, exactly as an attended review waits for approval.
- **Unattended: release the pin when the change parks.** A parked change is
  hands-on for a person elsewhere, so the worktree is free — hold the pin and the
  runner deadlocks after exactly one change. The pin is a **lock**; the bound on
  work-in-progress is `maxParkedUnattendedChanges`, counted from the listing.
  They are two mechanisms, not one.
- **Unattended: blocked means a review item, never a guess and never silence.**
  A stage that cannot proceed alone `cancelChangeStage`s, writes a self-describing
  `REV####` into the repo's review queue, and parks. Review items are this mode's
  output; a run never claims one as input.
- **Oversight overrides are per-run, never config.** Resolve the overlay on top of
  the freshly-read config each iteration; never `patchProduct`. Precedence:
  explicit pair → shorthand → config → default. The overlay sets only
  `human`/`agent` on skill-backed stages — a skill-less cicd stage can't be
  overridden to agent. Echo the effective map once at run start; don't prompt.
- **The intent field is the accepted decision** — confirmed by the user, not a
  ticket paste. (`--auto-all` and `--unattended` accept the distilled intent
  as-is; `--auto` does not.)
- **Never write lifecycle state via patch** — position moves only through the
  stage-action tools.
- **Risk is scored, never asserted.** Supply severity / occurrence / detection
  with per-axis evidence and let the server derive the category. Never send a
  category, and never try to patch `riskAssessment`, `confirmedPipeline` or
  `confirmedPipelineCategory` — `patchChange` refuses all three, deliberately: a
  patchable category would restore exactly the assertable category the derivation
  exists to remove. The confirmation fields have their own write path,
  `confirmChangePipeline`, which likewise derives rather than accepts the
  category; patch is still never it.
- **Never confirm a pipeline twice for the same selection.** Check whether
  `assessChangeRisk` already auto-confirmed the band before calling
  `confirmChangePipeline`. The change's history is an audit trail, and it should
  carry one entry per confirmation that actually changed.
- **Risk assessment never blocks the pipeline.** It is an activity within
  `accept`, not a stage — it moves no position and touches no stamps, and when
  the tool is unavailable (flag off, older server) the run simply continues
  without it. The same holds for every re-assessment.
- **Re-assess at the `design` and `code` boundaries, and ratchet one way.** A
  risen category forces the stricter pipeline and is announced; a fallen one is
  only ever *proposed* to a human, never applied, and is recorded-and-ignored
  under `--auto`, `--auto-all` or `--unattended`. Raising oversight is safe to
  automate; lowering it is the thing the human was consulted about.
- **The server decides whether risk selects the pipeline; you read the answer.**
  Take the oversight map from `getChange`'s `effectivePipeline` and report its
  `effectivePipelineSource`. Never infer authority from a repo setting, and never
  reshape a run around a category the server did not say was governing.
- **One change at a time per worktree** — `.defprod/change` **locks** it: Step 4
  refuses to claim a tree already pinned to a *different active* change (override
  only with `--force`), so parallel changes cannot silently share a tree — they
  belong in separate worktrees or branches. The pin is cleared at the land
  hand-off, on cancel, and — under `--unattended` — when the change parks, since
  a parked change is hands-on for a person elsewhere. Stage skills **self-heal**
  a stale pin on read
  (validate the pinned change is active; delete it if shipped/cancelled), so a
  leftover pin never traps the next change. As a backstop against a tree that
  drifts *mid-stage*, `/defprod-change-land` re-validates branch/pin consistency
  before committing and aborts on mismatch. Invariant: *pin present ⇔ a change is
  hands-on in this worktree*.
- Mid-flight tracker sync is out of scope: DefProd is the source of truth
  between the link and close bookends.
