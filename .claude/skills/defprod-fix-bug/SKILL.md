---
name: defprod-fix-bug
description: Traces a reported bug back to user stories, fixes it, and verifies the fix against acceptance criteria. Halts if the bug cannot be reproduced. (Shim - the workflow now runs as a tracked change via /defprod-change with type=bug.)
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - AskUserQuestion
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

# Fix Bug (shim)

This skill is now a thin entry point onto the change workflow. The full bug
process — reproduce-and-trace, fix, regression verification — lives in the
stage skills driven by **`/defprod-change`**, which additionally tracks the
work as a DefProd change record through the delivery pipeline.

## What to do

Invoke **`/defprod-change`** with:

- **type preset**: `bug`.
- **the bug description** the user gave (expected vs actual, where it occurs,
  reproduction steps if known), as the intake material. If the user referenced
  an external ticket, pass its ref/URL so the tracker adapter can fetch it.
- **any oversight-override args** the user supplied (e.g. `--auto`, `--auto-all`,
  `<stage>=<oversight>`) — forward them verbatim; `/defprod-change` interprets them
  (see its *Oversight overrides* section). `--unattended` forwards verbatim too,
  though it is a run mode rather than an override — do not drop it for not being
  on that list.

The orchestrator will create (or resume) the change record and walk the
pipeline. For bugs, the **define stage** is reproduce-and-trace: it finds the
acceptance criteria that define correct behaviour and *halts the change* if
the bug cannot be reproduced — never fix an unconfirmed bug. The **test
stage** confirms the original reproduction is fixed, regression-checks all
criteria on the in-scope stories, and leaves a regression test behind.

## Why this changed

Bug fixes are now first-class **change records** in DefProd: visible in the
pipeline like any other work, with reproduce/fix/verify stamped as a side
effect of doing the work. The old phase content of this skill was
redistributed into `/defprod-change-define`, `/defprod-change-code`, and
`/defprod-change-test`.
