---
name: defprod-untracked-change
description: The front door for recordless work (ADR0004) — real, quality-gated change that carries no trackable decision and does not move the product definition, so it gets NO change record. A thin sibling to /defprod-change that reuses the code/test/review/land stage skills (quality gates intact) but establishes no change context and stamps nothing. Use for small, decision-free edits (label/copy tweaks, lint, comment tidy, mechanical refactors); it escalates to /defprod-change the moment a design decision, a definition change, or a decision worth tracking appears.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - AskUserQuestion
  - mcp__defprod__listProducts
  - mcp__defprod__getProduct
  - mcp__defprod__listAreas
  - mcp__defprod__listUserStories
  - mcp__defprod__getUserStory
  - mcp__defprod__getChange
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

# Untracked Change

The workflow front door for **recordless work**: real, quality-gated change that
carries no trackable decision and does not move the product definition — so it
gets **no change record** (`CHG`), no stage stamps, no `Change:` trailer. It
rides as ordinary commits, exactly as a label rename or a lint fix should.

This is the sibling of `/defprod-change`. Same quality gates (code → test →
review → land), reusing the **same stage skills** — but with the change-record
machinery removed. The grain rationale is **ADR0004** (intent, not size): a
change deserves a record when it carries a decision worth tracking; everything
below that line comes here.

## When to use

- Decision-free mechanical edits: label / copy / wording tweaks, lint and
  formatting, comment tidy-ups, dead-code removal, small refactors with no
  behaviour decision.
- Work where there is **no decision worth tracking** and the **product
  definition does not change**.

## When NOT to use — escalate to `/defprod-change`

Stop and use `/defprod-change` (which creates a record and re-enters with
context) the moment any of these is true — these are the **escalation
tripwires**, checked at the entry gate and again any time one surfaces mid-flow:

- **A real design decision** is involved (more than one reasonable approach with
  consequences).
- **The product definition must change** — a user story, area, brief, or
  architecture element needs creating or editing.
- **The work carries a decision the business should know** (ADR / DEC / brain-note
  territory — see DEC0026).

When unsure, prefer `/defprod-change`: a record is cheap to create, but
recordless work that *should* have been tracked is invisible after the fact.

## Workflow

### Step 0 — Require a clean worktree (no change in flight)

This skill must run **context-free** so the reused stage skills stamp nothing.
Verify none of the three change-context carriers resolves:

1. `.defprod/change` in the worktree root.
2. The current branch is named `chg/<slug>/CHG-NN-*` (or legacy `chg/CHG-NN-*`).
3. The HEAD commit has a `Change: CHG-NN` trailer.

If a carrier is present, `getChange` on its key:
- **Active (not shipped, not cancelled)** → STOP: a tracked change is in flight
  here. Finish/land it, or do this work in a separate worktree.
- **Terminal/stale** (shipped/frozen or cancelled) → **self-heal**: delete the
  stale `.defprod/change` file and proceed as no-context (same rule the stage
  skills apply on read). A stale carrier never blocks untracked work.

### Step 1 — Entry gate

Confirm with the user (or, for obviously trivial edits, on inspection) that this
is genuinely untracked work: none of the escalation tripwires above applies. If
one does, STOP and recommend `/defprod-change`.

### Step 2 — Read-only alignment

Verify the work does not **contradict** the product definition: read the
relevant area(s) and stories (`listAreas`, `listUserStories`, `getUserStory`) and
confirm the change is consistent with their acceptance criteria. This stage is
**read-only** — it must never `createUserStory` / `patchUserStory` / patch any
definition entity. If the work turns out to *require* a definition change, that
is a tripwire → escalate to `/defprod-change`.

### Step 3 — Code → Test → Review (reused stage skills, context-free)

Invoke the standard stage skills in sequence. With no change context they
**self-suppress stamping** and run as plain quality gates:

1. `/defprod-change-code` — implement and compile-check.
2. `/defprod-change-test` — verify behaviour; add/extend tests as the change
   warrants. **Not skipped** — running test + review is what separates this from
   a bare `git commit`.
3. `/defprod-change-review` — review the diff for correctness and convention.

**Capability dispatch:** prefer a repo-local override of any stage skill if one
exists (identical no-context behaviour). Invoked this way the stage skills take
no orchestrator mode, so they follow their standalone default (interactive) —
the human stays in the loop, so a quick edit is overseen rather than
rubber-stamped, and landing still honours the repo's push/merge consent (below).

### Step 4 — Land (no record artifacts)

Invoke `/defprod-change-land`. With no context it commits and lands **silently**:

- **No `Change:` trailer** (there is no change to correlate) — use the repo's
  ordinary commit conventions, with an appropriate conventional-commit prefix
  (`chore:`, `fix:`, `style:`, …).
- **No `[skip ci]`** — untracked work is shippable; it deploys like anything else
  on the deployable branch.
- Honour the repo's standing push/merge consent exactly as `/defprod-change-land`
  does.

## Rules

- **Establishes no change context, ever** — never writes `.defprod/change`, never
  creates a `chg/<slug>/CHG-NN-*` branch, never appends a `Change:` trailer.
- **Stamps nothing** — there is no change record; the reused stage skills run in
  their silent, no-context mode.
- **The quality gates are not optional** — test and review run; that is the whole
  difference between this and an untracked raw commit.
- **Escalation is one-way and cheap** — at any tripwire, stop and hand off to
  `/defprod-change`; do not quietly grow recordless work past the line.
