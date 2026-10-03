---
name: defprod-implement-feature
description: End-to-end workflow for implementing a feature — user story alignment, implementation, and verification. Use when starting any coding task. (Shim - the workflow now runs as a tracked change via /defprod-change with type=feature.)
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

# Implement Feature (shim)

This skill is now a thin entry point onto the change workflow. The full
feature process — definition alignment, implementation, verification — lives
in the stage skills driven by **`/defprod-change`**, which additionally tracks
the work as a DefProd change record through the delivery pipeline.

## What to do

Invoke **`/defprod-change`** with:

- **type preset**: `feature` (or `enhancement` if the request improves an
  existing capability rather than adding one).
- **the task description** the user gave, as the intake material. If the user
  referenced an external ticket, pass its ref/URL so the tracker adapter can
  fetch it; otherwise it proceeds as ad-hoc internal work.
- **any oversight-override args** the user supplied (e.g. `--auto`, `--auto-all`,
  `<stage>=<oversight>`) — forward them verbatim; `/defprod-change` interprets them
  (see its *Oversight overrides* section). `--unattended` forwards verbatim too,
  though it is a run mode rather than an override — do not drop it for not being
  on that list.

The orchestrator will create (or resume) the change record and walk the
pipeline: define (user story alignment) → code → test → review → land, with
human gates and CI/CD handoff per the product's pipeline configuration.

## Why this changed

Feature work is now a first-class **change record** in DefProd: PMs see where
it stands (waiting for review / reviewing / reviewed) without asking, and
every stage is stamped as a side effect of doing the work. The old Phase 1/2/3
content of this skill was redistributed into `/defprod-change-define`,
`/defprod-change-code`, and `/defprod-change-test`.
