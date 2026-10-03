---
name: defprod-change-land
description: Land stage of the change workflow — commit with the Change trailer, then merge and/or push per the repo's flow, stamping the merge/push stages. The Change trailer is what lets CI hooks correlate later pipeline stages. Usually invoked by /defprod-change; works standalone too.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Write
  - AskUserQuestion
  - mcp__defprod__getChange
  - mcp__defprod__getUserStory
  - mcp__defprod__patchUserStory
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

# Change Stage: Land (merge / push)

Get the reviewed change onto the deployable branch, carrying the correlation
trailer that CI/CD hooks use to stamp the remaining pipeline stages.

## Change context (stamping preamble)

Resolve the current change context, in precedence order:
1. `.defprod/change` in the worktree root — JSON `{ productId, changeId, changeKey, productSlug, multiProduct }`, plus `unattended: true` when the run has nobody in the session.
2. A branch named `chg/<slug>/CHG-NN-*` (or legacy `chg/CHG-NN-*`) → resolve via `getChange { productId, key }`.
3. A `Change: <product-slug>/CHG-NN` trailer on the HEAD commit → resolve the
   slug to a product, then `getChange { productId, key }`. Tolerate a legacy
   bare `Change: CHG-NN` (no slug) on pre-existing history — resolve it with the
   pinned/known productId.

**A resolved carrier is a hint, not proof — validate it.** `getChange` the key
and confirm the change is live: if it is **shipped (frozen)** or **cancelled**,
the carrier is stale (a prior change left un-cleared). Disregard it — deleting a
stale `.defprod/change` file — and proceed as **no-context**. Only an *active*
change is a live context to stamp.

This stage handles **two pipeline stages** — `merge` and `push` — stamping
whichever of them are enabled in the product's pipeline. **If no context
resolves, proceed silently** (commit/push without stamping or trailer).

### Branch/pin consistency guard (before any commit)

The worktree can drift underneath you — when multiple change sessions run, a
concurrent session may check the tree over to another change's branch and
repoint `.defprod/change` between the moment this stage began and the moment you
commit. Committing then silently lands your work on **the wrong branch** (and the
intended push becomes a no-op). Before committing (Workflow step 1),
**re-validate** that the tree is still the one you resolved context for, and
**abort loudly** — do not commit — if any of these hold:

- **Pin ⇄ branch disagree.** HEAD is on a `chg/<slug>/CHG-NN-*` branch whose `CHG-NN`
  differs from the `changeKey` in `.defprod/change`. The tree was checked over
  to a different change's branch than the pin claims — corruption in progress.
- **Pin moved off your change.** A context was resolved at preamble, but a fresh
  read of `.defprod/change` now names a **different** `changeId` than the one you
  resolved. Another session claimed this tree; stop before your commit lands
  under its identity.
- **Branch ⇄ resolved-change disagree.** You resolved an active change (with a
  `chg/<slug>/CHG-NN-*` branch flow) but HEAD is on neither that change's branch nor the
  default branch — an unexpected checkout happened underneath the stage.

On any mismatch, **abort with a clear message** naming the expected vs. actual
branch and pinned change, and do nothing destructive (no commit, no branch
switch, no pin rewrite) — recovery is the operator's call. A clean tree (pin,
branch, and resolved change all agree, or genuine no-context) proceeds normally.

## Execution mode (autonomous / interactive)

The orchestrator passes a **mode** derived from this stage's `oversight`:
`agent` → `autonomous`, `human` → `interactive`. Invoked standalone with no
mode given, default to **interactive**.

- **autonomous** — commit and land without pausing for routine choices,
  `startChangeStage` before each merge/push you perform and `finishChangeStage`
  after it succeeds. Surface genuine blockers.
- **interactive** — keep the human in the loop: ask when the landing flow is
  ambiguous, and present before finishing.

**Unattended, landing needs the repository's permission on top of the oversight.**
`--unattended` parks before `merge`/`push` unless `Repo.allowUnattendedLand` is
set, so the orchestrator dispatches you there only when it re-read that field and
found it set — and when it does, it says so, passing `allowUnattendedLand=true`
with the mode. `agent` oversight is consent from the *pipeline*; this is a
separate consent from the *repository*.

Treat it as a floor you enforce too. Invoked with `unattended` and **no** such
statement — a standalone invocation, or an orchestrator too old to pass it — you
have no way to check it yourself (`getRepo` is not in your tool list) and an
unstated permission is not one. So:

1. **Commit** the work to the change branch, per step 1 below — do not merge, do
   not push, do not open a PR.
2. **`cancelChangeStage`** the landing stage you started. Committing is not
   landing, so leaving `merge` or `push` open would record a stage still underway
   that nothing will ever finish.
3. **Raise a review item and return `blocked`**, exactly as for any other
   blocker below — `kind: action`, because landing this change by hand (or
   granting the repository the permission) is a task only a person can do.
   Nothing is *undecided* here, which makes it tempting to skip the record; skip
   it and an unattended run ends having committed work that nobody is told
   about, which is the silence the whole contract exists to prevent. `blocked`
   always carries an item.

**Merge/push consent follows the oversight** (D14/D26). A stage's `oversight`
*is* the durable consent signal: a merge/push under `agent` oversight (→
**autonomous**) is standing consent — merge/push **without prompting**. One under
`human` oversight (→ **interactive**) makes the human the consent point — confirm
before you merge or push. Re-asking on an `agent` stage contradicts the config;
honour it. Only when run with **no oversight context** (standalone, consent
genuinely unknown) do
you default to committing and stopping.

### Blocked in an unattended run

The orchestrator passes **`unattended`** alongside the mode when there is nobody
in the session (it is also recorded as `"unattended": true` in the
`.defprod/change` pin). It means no question can be asked and no failure will be
noticed by a person watching.

**Autonomous does not mean press on.** Where this stage meets something it must
not settle alone, or cannot get past:

- integrating the trunk needs a non-trivial conflict resolution, where picking a
  side is a decision about behaviour rather than a mechanical merge;
- a commit hook, gate or protected-branch rule refuses, and clearing it means
  changing something outside this change.

…do not guess, and do not finish the stage. Instead:

1. **`cancelChangeStage { changeId, stage: '<the stage you started>' }`** —
   this skill drives `merge` and `push`, so cancel whichever one you had
   underway. The stage was started and is being abandoned, so record that rather
   than leaving it open forever or stamping a finish over work that stopped.
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

1. **Commit** the change's work (if uncommitted), following the repo's commit
   conventions, and append the correlation trailer as the final line:

   ```
   Change: <product-slug>/CHG-NN
   ```

   The slug is the change's owning product slug — take it from `productSlug` in
   the `.defprod/change` pin (or `getProduct(productId).slug`). It makes the
   trailer product-scoped so CI can resolve the right product in a multi-product
   monorepo (a bare `CHG-NN` key is only unique *within* a product). Every commit
   belonging to the change carries it — it survives squashes and cherry-picks and
   is how CI resolves the change from a push range. **The trailer is always
   qualified this way, regardless of repo cardinality** — this is unchanged
   since v1.7.0 and distinct from the rule below.

   If the commit **subject or body** also mentions the change key (per the
   repo's own conventions — this stage doesn't mandate a mention), render that
   mention per the *Change-key qualification* rule in `defprod-change/SKILL.md`
   (the `multiProduct` flag in the `.defprod/change` pin, or the resolved
   context if there's no pin): qualified `<product-slug>/CHG-NN` in a
   multi-product repo, bare `CHG-NN` in a single-product one. This is
   independent of the trailer, which stays qualified either way — don't copy
   the trailer's form into the subject/body without checking `multiProduct`
   first.

   **This stage emits the trailer UNSUFFIXED.** The trailer accepts an optional
   stage ceiling, `Change: <product-slug>/CHG-NN:<stage>`, meaning *this commit is
   entitled to advance the change no further than `<stage>`*. Land does not use it:
   land IS the landing, so the change genuinely reaches whatever CI later reports.

   Every OTHER stage that commits an artefact must suffix with its own stage — see
   the trailer-ceiling rule in `defprod-change-design/SKILL.md`. The asymmetry is
   the point: a suffix on the landing would stop CI advancing the change at all,
   and its absence on an interim commit lets that commit claim the whole change
   forever.

2. **Land per the repo's flow** — ask the user if ambiguous. For every merge or
   push **you** perform, stamp the matching stage on **both sides** of the
   operation: `startChangeStage { stage }` immediately before you begin, then
   `finishChangeStage { stage }` once it succeeds. The start stamp records that
   the landing operation is underway (and under what oversight); the finish stamp
   records completion. If the operation fails, leave the stage started (or
   `cancelChangeStage` if you abandon it) — never finish a stage whose operation
   did not succeed.

   **Report the oversight it received.** Pass `oversight` on the start stamp:
   `agent` in autonomous mode, `human` in interactive mode — the inverse of the
   oversight-to-mode translation that set this stage's consent, so the stamp
   records the oversight the merge or push *actually received*. A merge or push
   performed by the platform or by CI is stamped by its own hook and reports its
   own oversight; never report one on its behalf. First report wins, and it is
   never inferred from configuration. An older server knows the field as
   `driver`: if the stamp is refused for naming `oversight`, retry with the same
   value as `driver`, then without it — neither refusal is a failed land.

   **If you pass `commitSha`, pass the full sha** of the commit you just made —
   `git rev-parse HEAD`, never `--short`. It is the stamp's idempotency key; an
   abbreviation is a different key for the same commit, and `--short` scales its
   length with the repo, so it is not even stable. The CI hooks already send the
   full sha, and a landing stamp should match them.
   - **Branch/PR flow**: push the `chg/<slug>/CHG-NN-*` branch and open/hand off the
     PR. If **you** compose the PR title/body, render any change-key mention
     per the same *Change-key qualification* rule as the commit subject/body
     (qualified in a multi-product repo, bare in a single-product one) — the
     branch name itself stays qualified either way, per Step 4 of
     `defprod-change/SKILL.md`. The `merge` stage finishes when the PR merges —
     if you perform the
     merge, `startChangeStage { stage: 'merge' }` before it and
     `finishChangeStage { stage: 'merge' }` after; if the platform/CI performs
     it, its hook stamps both instead. `push` is typically redundant here
     (often disabled in the pipeline).
   - **Trunk flow**: commit on the default branch; `merge` is typically
     disabled. `startChangeStage { stage: 'push' }`, push to origin, then
     `finishChangeStage { stage: 'push' }` on success.
   - **Local-merge trunk flow**: a change branch, merged **locally** into the
     trunk and pushed from there — no forge, so no PR to open. Both `merge` and
     `push` are typically enabled: stamp `merge` around the local merge, then
     `push` around the push of the trunk. Read *Integrating a diverged trunk*
     below before you merge — this flow is the one the duplication trap bites.

   Only stamp stages that are enabled — a disabled stage is rejected by the
   server; treat that as "not my pipeline's stage", not an error.

### Integrating a diverged trunk — never rewrite it

When the trunk has moved on the remote, integrate it by **fast-forward, falling
back to a merge commit**. Never rebase the trunk:

```sh
git fetch origin
git merge --ff-only origin/main   # the usual case
git merge origin/main             # only if --ff-only is refused
```

**Rebasing the trunk does not invalidate one branch — it invalidates every
outstanding change branch at once.** Each was cut from the pre-rebase trunk and
still carries the original commits, so merging any of them afterwards re-attaches
those originals alongside their rewritten copies and the trunk ends up holding
each commit **twice**, under two hashes. The damage is silent: it surfaces later
as double-counted commits in release notes and anything else derived from a
commit range, long after the land looks like it succeeded.

The reflex fix — "rebase the trunk, then rebase my branch onto it" — only works
when yours is the *only* branch outstanding. Where an installation runs changes
concurrently in separate worktrees, the other branches cannot be repaired at
all: git refuses to rebase a branch that is checked out elsewhere, and those
trees hold uncommitted work. Rewriting the trunk is therefore never the answer;
a merge bubble is the accepted cost.

**Check before you merge.** This is portable and needs no tooling:

```sh
git cherry main HEAD   # '-' = patch already in main, '+' = genuinely new
```

Any `-` line means merging this branch would land that commit a second time —
the trunk was rewritten after your branch was cut from it. **Do not merge.**
Rebase **your branch** onto the current trunk (`git rebase main`, which drops the
already-upstream commits) and re-check. Never rewrite the trunk to make the
check pass.

Two caveats worth knowing: `git cherry` compares patch content, so a copy whose
content was altered by conflict resolution during the rebase has a different
patch id and will **not** be flagged — a clean result means "no identical
duplicates", not "no duplicates". And if the installation provides its own land
preflight, prefer it; `SKILL.local.md` is where that is recorded.

3. **Merge/push consent = the stage oversight** (D14/D26). `agent` (autonomous) →
   merge/push **without prompting** (the oversight config is the standing consent);
   `human` (interactive) → confirm first. With **no oversight context** (standalone,
   consent unknown), stop after the commit and report. Never assume consent you
   were given by neither config nor a human.

4. **Clear the worktree pin on hand-off.** Once your landing actions have
   succeeded — the push to origin (trunk flow: the deployable branch; branch/PR
   flow: the change branch, with the PR handed off) or a merge you performed —
   and the change is handed to CI/CD or the platform, **clear
   `.defprod/change`** — the worktree's hands-on role is over. The remaining
   stages (`build`/`package`/`staging`/`ship`) are stamped by CI/CD via the
   commit **trailer** deploy range (D24), never via the pin, so nothing
   downstream needs it. This preserves the invariant *pin present ⇔ a change is
   hands-on in this worktree* and frees the worktree for the next change without
   waiting for `ship`. If you only committed and **stopped** for consent (no
   push/merge performed), that is **not** a hand-off — leave the pin.

   **Whose job is it to clear the pin?** Deleting the file is right only when
   the pin was written by hand. If the installation's tooling wrote it, that
   tooling owns the teardown — call *its* release step instead, so everything
   else bound to the change (the worktree or environment, an isolated database,
   a running session) is released along with the pin. Deleting the file directly
   in that setup half-releases: the pin goes, the rest of the claim stays, and
   the next change finds the environment still held. `defprod-change/SKILL.local.md`
   is where an installation records that it owns the pin's lifecycle — which is
   why the preamble above has you read it even on a standalone run.

5. **Advance the delivered stories to `completed`.** A story is delivered when
   the change carrying it lands — which is why this write belongs here and not
   in the test stage, where the acceptance criteria are verified but nothing has
   shipped yet. Run this **after** your landing actions have actually succeeded.
   Skip it entirely if you only committed and **stopped** for consent, or if the
   merge/push failed: nothing was delivered, so nothing may be marked delivered.
   With **no change context**, skip it too.

   For each story on the change's `userStoryIds`:

   - `getUserStory` it and enumerate **every** acceptance criterion — not just
     the ones this change was built to satisfy.
   - Decide whether the story is now **wholly** delivered: each AC is either
     satisfied by the work you just landed, or was already satisfied by
     previously delivered behaviour. An AC that no code implements, or one
     implemented but never verified by the test stage nor covered by existing
     tests, leaves the story incomplete.
   - **Wholly delivered** → patch it to `completed`:

     ```
     patchUserStory {
       userStoryId: '<id>',
       patch: [{ op: 'replace', path: '/status', value: 'completed' }],
       comment: 'delivered by <product-slug>/CHG-NN'
     }
     ```

     The call takes an **RFC 6902 patch array**, not a field bag — a bare
     `{ status }` is rejected.
   - **Partially delivered** → leave it at `inProgress` (patch it there the same
     way if it is still `backlog` or `ready`) and **name the outstanding
     acceptance criteria**
     in your stage summary. A story can be in scope for several changes, and
     completing it on the first change to touch it overstates delivery — which is
     worse than leaving it open, because nothing later corrects it.

   Write it **without prompting** in both autonomous and interactive mode, and
   report every transition you made. Never move a story backwards. If the write
   fails — an older server, a scope-limited key — report it and carry on: the
   land has already succeeded and a status write must never turn it into a
   failure.

## Rules

- The trailer is non-negotiable when a change context exists — a landed change
  without it is invisible to every downstream CI hook.
- Stamp `startChangeStage` before a merge or push you perform and
  `finishChangeStage` after it succeeds — both sides, even for quick atomic
  operations. The start stamp marks the landing as in progress; finish marks it
  done. (This supersedes any earlier "finish-only for quick operations"
  shortcut.)
- **Never rebase the trunk to integrate it.** Fast-forward, or merge. Rebasing
  it rewrites the commits every outstanding change branch was cut from, and
  merging those branches afterwards duplicates history under new hashes — the
  one failure in this stage that succeeds loudly and fails silently.
- **This stage owns the `→ completed` transition on a story's status**; the code
  stage owns `→ inProgress`. No other stage writes it. Only a landed change may
  complete a story, and only a story whose *every* acceptance criterion is met
  may be completed.
