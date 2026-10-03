---
name: defprod-change-tracker
description: USER-OWNED adapter — how this team's external tracker (JIRA, Notion, Linear, a docs repo, …) is read and written by the change workflow. Fill in the four operations below for your tracker; /defprod-change invokes this skill for ticket fetch, promotion link write-back, terminal close write-back, and — for an unattended run — listing the tickets it may claim.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - WebFetch
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

# Change Tracker Adapter (fill me in)

`/defprod-change` is tracker-agnostic. This skill is the seam: it defines the
**four operations** the change workflow needs from your tracker, and **you
edit this file** to say how each is done for your team. The worked examples
below cover common setups — replace the "Your tracker" sections with real
instructions and delete the examples you don't use.

Only the fourth, `listClaimable`, is optional: it is what an unattended run uses
to find work, and a team that never runs one can leave it unfilled.

> **This file is user-owned.** The skills installer never overwrites a
> locally-modified copy of this skill on update.

## The four operations

> **Change-key form.** The `changeKey` passed into `link`/`close` arrives
> already rendered correctly for this repo — qualified `<product-slug>/CHG-NN`
> in a multi-product repo, bare `CHG-NN` in a single-product one (the
> *Change-key qualification* rule in `defprod-change/SKILL.md` is the single
> source of truth). Write it into your tracker **as given** — don't reformat
> or re-derive it. The worked examples below use `<changeKey>` as a
> placeholder for this already-correct value.

### 1. `fetch(ref)` — read an accepted ticket

Given a ticket reference or URL, obtain: **title**, **url**, the material
needed to distill an **intent** (what & why), and the **origin** —
`{ system, ref, url }`, the tracker-side address this work came from. Also
report whether the ticket already carries a DefProd change link (for dedupe).

**The origin is an output of `fetch`, not an afterthought at create time.**
`system` names the tracker: a stable lowercase string (`jira`, `linear`,
`brain`), the *same* one every time, because it is half of the dedupe key that
catches a double promotion. `ref` is the ticket's own identifier, and `url` is
the deep link, or null where the tracker has no addressable one.

Return it whenever a ticket was fetched at all — it is what makes a promotion a
two-way link rather than a one-way note in your tracker. It is **independent of
`source`**: work the team raised in its own backlog is internal *and* has an
origin.

**Your tracker:** _describe how to fetch a ticket here._

### 2. `link(ref, changeKey)` — mark the ticket promoted

Record the DefProd change key (e.g. `<changeKey>`) on the ticket so a second
promotion attempt is caught at fetch time.

**Your tracker:** _describe how to write the link here._

### 3. `close(ref, outcome)` — terminal write-back

When the change ships or is cancelled, write the outcome back (e.g. a comment
"Shipped as `<changeKey>`" or a status flip).

Only `ship` and cancellation are outcomes. A change that **parked** for a person
is still active and still promoted — `/defprod-change` does not call `close` for
it, and neither should you.

**Your tracker:** _describe how to close out the ticket here._

### 4. `listClaimable()` — candidate work for an unattended run (optional)

Used only by `/defprod-change --unattended`, which runs with nobody in the
session and therefore has to be *told* what it may pick up. Return the open,
actionable tickets that are not already promoted, each with:

- **`ref`** — the ticket's own identifier, as `fetch` would take it;
- **`title`** — one line, enough for a human reading the run's report;
- **`ownership`** — exactly one of `agent`, `unowned`, or `person` (see below);
- **`alreadyPromoted`** — the same already-promoted signal `fetch` reports.

Return them in whatever order your team considers most important first; the
runner takes the first eligible one.

**Report ownership; do not decide eligibility.** The split is deliberate, and it
is the whole reason this operation lives here rather than in the orchestrator:

- **What counts as *unowned* is your tracker's convention**, which only this file
  knows — `owner: unassigned` in a markdown intent repo, an empty assignee in
  Jira or Linear, a blank person property in Notion. Say which, below.
- **Whether unowned work may be claimed is the repository's decision**, made once
  in DefProd via `Repo.claimUnownedWork` and read live by the orchestrator. It is
  off unless set, so a tracker full of unassigned tickets grants nothing by
  itself.

So classify honestly and let the policy do its job. `agent` means the ticket
carries an owner and that owner **is** the agent identity the run authenticates
as — a one-word edit a person made deliberately, which is what makes it consent.
Never report `agent` or `unowned` for a ticket assigned to a person because the
work *looks* like agent work; that is a claim the person never made.

**Your tracker:** _describe how to list claimable tickets, and say exactly what
"unowned" looks like here — or delete this section if you never run unattended._

---

## Worked examples

### Example A — markdown intents repo (a "team brain")

A git repo of intent files like `intents/INT0007-bulk-export.md` with
frontmatter including `status:` and `link:` fields.

- **fetch**: `Read` the intent file by ref (e.g. `INT0007` →
  `~/team-brain/intents/INT0007-*.md`). Title = H1; intent material = body.
  Already-promoted check: frontmatter `link:` is non-empty.
- **link**: set frontmatter `link: <changeKey>` and `status: promoted`; commit.
- **close**: set `status: done` (or `dropped`), append an outcome line; commit.
- **listClaimable**: `grep` the files with `status: open` and an empty `link:`.
  Ownership comes from the `owner:` line — the agent's own identity → `agent`,
  the literal `unassigned` (or a missing/empty value) → `unowned`, anything else
  → `person`.

### Example B — JIRA via an MCP connector

Assumes a JIRA/Atlassian MCP server is connected.

- **fetch**: get the issue by key; title = summary, url = browse link, intent
  material = description + recent comments. Already-promoted check: look for a
  `DefProd: <changeKey>` label or comment.
- **link**: add a comment `Promoted to DefProd as <changeKey>` (or set a custom
  field / label if your project has one).
- **close**: add a comment `Shipped as <changeKey>` / `Cancelled (<changeKey>)`; or
  transition the issue per your team's workflow.
- **listClaimable**: search the project's ready-for-development status with no
  DefProd label. **Unowned = no assignee.** Assignee equal to the agent's own
  account → `agent`; any other assignee → `person`.

### Example C — Notion via an MCP connector

- **fetch**: read the page by URL/id; title = page title, intent material =
  page content. Already-promoted check: a `DefProd change` property.
- **link**: set the `DefProd change` property to `<changeKey>`.
- **close**: set the page status property and append an outcome block.
- **listClaimable**: query the database for the ready status with an empty
  `DefProd change` property. **Unowned = an empty `Owner` person property.**

---

## Rules

- All tracker writes are **best-effort**: report failures, never block the
  change workflow on them.
- Never store tracker credentials in this file — rely on connectors (MCP
  servers, CLIs) already configured in the environment.
- DefProd is the source of truth between `link` and `close` — do not add
  per-stage status mirroring here.
- **`listClaimable` reports ownership, never eligibility.** What "unowned" means
  is this file's to define; whether unowned work may be claimed is
  `Repo.claimUnownedWork`, read live by the orchestrator and off unless set.
  Widening the definition of "unowned" here to get more work picked up bypasses a
  decision the repository made deliberately.
