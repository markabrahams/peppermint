---
name: defprod-analyze-discrepancies
description: Finds drift between a product area's definition and the codebase implementation. Produces a structured discrepancy report.
allowed-tools:
  - Read
  - Edit
  - Glob
  - Grep
  - Bash(ls:*)
  - Bash(cat:*)
  - AskUserQuestion
  - mcp__defprod__listProducts
  - mcp__defprod__getProduct
  - mcp__defprod__listAreas
  - mcp__defprod__getArea
  - mcp__defprod__listUserStories
  - mcp__defprod__getUserStory
  - mcp__defprod__getBriefForProduct
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

# Analyse Discrepancies

Compares a product area's user stories and acceptance criteria against the actual codebase implementation. Identifies drift in both directions: features defined but not implemented, and features implemented but not defined. Produces a structured report that drives `/defprod-fix-discrepancies`.

## When to use

- When you suspect the product definition and codebase have drifted apart.
- After a period of development without definition updates.
- As a periodic health check on definition accuracy.
- When invoked via `/defprod-analyze-discrepancies <product-name> <area-key>`.

## Inputs

- **Product name** — the DefProd product.
- **Area key** — the area to analyse (e.g. `AUTH`, `BILLING`).

## Config

This skill consults `.defprod/defprod.json` for optional hints. If the file doesn't exist, the skill discovers everything automatically.

| Key | Type | Purpose |
|-----|------|---------|
| `products[].name` | `string` | Matches product name to locate config entry |
| `products[].frontendApp` | `string` | Path to the frontend app — focuses exploration |
| `products[].backendApp` | `string` | Path to the backend app — focuses exploration |

---

## Workflow

### Step 1 — Fetch the product definition

1. Call `listProducts` from the DefProd MCP server. Find the product by name.
2. Call `listAreas` for the product. Find the area by key.
3. Call `listUserStories` for the area. Read each story's title, description, and acceptance criteria.

If the area has no user stories, tell the user there's nothing to analyse and suggest running `/defprod-create-area-stories` first.

---

### Step 2 — Explore the codebase

Build a thorough inventory of what the codebase actually implements for this area.

#### 2a. Identify relevant code

Use `.defprod/defprod.json` paths if available. Otherwise, locate the code by:
- Searching for route definitions, module names, or directory names matching the area
- Following imports and dependencies from entry points
- Checking both frontend and backend code

#### 2b. Explore in parallel

Explore three dimensions simultaneously:

**Backend module** (if applicable):
- API routes and endpoints
- Controllers and handlers
- Services and business logic
- Data models and schemas
- Validation rules
- Access control and permissions

**Frontend module** (if applicable):
- Pages and components
- Forms and user interactions
- State management
- Navigation and routing
- Error handling and feedback

**Shared types/contracts** (if applicable):
- DTOs and interfaces
- API contracts
- Shared validation rules

For each capability found, document:
- What it does
- Which files implement it
- Key implementation details (e.g. validation rules, permission checks)

---

### Step 3 — Produce analysis

Compare the product definition against the codebase inventory. Produce **four tables**:

#### Table 1 — Stories or acceptance criteria with no implementation

Features defined in DefProd but not found in the codebase.

| Story | AC | Defined Behaviour | Recommendation |
|-------|----|--------------------|----------------|
| AREA-KEY | AC text | What the story says | Remove from definition / Implement |

#### Table 2 — Acceptance criteria with incomplete implementation

Features partially implemented — the code exists but doesn't fully satisfy the acceptance criteria.

| Story | AC | Defined Behaviour | Actual Behaviour | Recommendation |
|-------|----|--------------------|-------------------|----------------|
| AREA-KEY | AC text | What the story says | What the code does | Update definition / Fix code |

#### Table 3 — Implemented features with no story

Capabilities found in the codebase that have no corresponding user story.

| Feature | Files | Description | Recommendation |
|---------|-------|-------------|----------------|
| Feature name | file paths | What it does | Create story / Remove code |

#### Table 4 — Stories that diverge from implementation

Features where both the story and code exist, but the details don't match.

| Story | AC | Defined Behaviour | Actual Behaviour | Recommendation |
|-------|----|--------------------|-------------------|----------------|
| AREA-KEY | AC text | What the story says | What the code does | Update definition / Fix code |

#### Summary

After the tables, write a brief summary highlighting:
- The overall health of the definition (how much drift exists)
- Key themes (e.g. "Most drift is in validation rules" or "3 new features were added without stories")
- Recommended priority order for fixes

---

### Step 4 — Save report

Write the report to the project. Choose a location in the following order of preference (most preferred first — use the first option that exists):

1. **The app's e2e test structure**, ideally in the area-specific directory — e.g. `apps/defprod-front/e2e/areas/<area-key>/discrepancy-<YYYY-MM-DD>.md` (e.g. `apps/defprod-front/e2e/areas/ONBOARD/discrepancy-2026-05-16.md`). If the e2e structure exists but the area-specific directory does not, fall back to the next option rather than creating a new e2e directory.
2. **The area-specific documentation directory** — e.g. `docs/areas/<area-name>/discrepancy-<YYYY-MM-DD>.md` (e.g. `docs/areas/onboarding/discrepancy-2026-05-16.md`).
3. **The general documentation directory** — e.g. `docs/defprod-discrepancy-<area-key>-<YYYY-MM-DD>.md`.

Tell the user where the report was saved.

> **Discrepancy analysis complete for {Area Name}.**
>
> - {N} stories/ACs with no implementation
> - {N} partially implemented ACs
> - {N} implemented features with no story
> - {N} stories diverging from implementation
>
> Report saved to `<path>`
>
> **Next step:** Walk through the report with the user (Step 5) before any fixes are applied.

---

### Step 5 — Walk the report with the user

The report from Step 4 is a **draft**. It is not authoritative until the user has confirmed (or amended) every row. This step is mandatory and cannot be skipped silently.

Walk the report **table by table, row by row**.

For each row:

1. Show the user the relevant fields:
   - Story / Feature identifier
   - Acceptance criterion text (if applicable)
   - Defined behaviour
   - Actual behaviour (where the table includes it)
   - The skill's recommended action

2. Ask the user to confirm or change the recommendation using `AskUserQuestion`. Put the skill's recommendation first and label it `(Recommended)`. Typical options:
   - **Keep recommendation** *(Recommended)* — apply the action as written
   - **Switch direction** — flip the recommendation (e.g. "Update definition" → "Fix code", or vice versa)
   - **Defer / No action** — leave the discrepancy untouched for now and record the reason
   - **Edit wording** — the user provides custom text for the `Recommendation` cell

3. Immediately after the user answers, update the saved report on disk so the file always reflects the latest confirmed state. Use `Edit` to rewrite only the affected row(s). Never batch updates — write each one as it is confirmed, so an interrupted walkthrough leaves a coherent partial report.

4. Track for each row whether the user **kept**, **changed**, **deferred**, or **edited** the recommendation. Capture any free-text justification the user provides.

After every row in a table is processed, give the user a one-line summary of that table's outcomes (e.g. *"Table 2: 8 kept, 2 switched to 'Fix code', 1 deferred"*) before moving to the next table.

When all four tables are complete, append a **Walkthrough log** section to the report:

```markdown
## Walkthrough log

Walked through on <YYYY-MM-DD> with <user>.

| Table | Row | Original recommendation | Final action | Notes |
|-------|-----|-------------------------|--------------|-------|
| 1 | AREA-KEY-AC1 | Implement | Kept | — |
| 1 | AREA-KEY-AC2 | Remove from definition | Switched → Implement | User: this is on the roadmap for Q3 |
| ... | ... | ... | ... | ... |
```

Finally, update the user-facing summary with the post-walkthrough counts and the next step:

> **Walkthrough complete for {Area Name}.**
>
> - {N} rows kept as recommended
> - {N} rows changed by user
> - {N} rows deferred
>
> **Next step:** Run `/defprod-fix-discrepancies <report-path>` to apply the confirmed actions.

---

## Rules

- **Read the code, don't guess** — every classification must be based on actually reading the relevant source files. Don't infer implementation from file names alone.
- **Check both directions** — drift happens in both directions. Code without stories is as important as stories without code.
- **Be specific in recommendations** — "Update the story" is not enough. Specify what should change and why.
- **Don't fix anything** — this skill analyses and reports. Fixes happen in `/defprod-fix-discrepancies`.
- **Acceptance criteria are granular** — analyse each criterion individually, not just the story as a whole. One story might have 5 satisfied criteria and 1 that's drifted.
- **Always save the report** — even if no discrepancies are found, save a report confirming alignment.
- **Walkthrough is mandatory** — Step 5 must run for every report. The report on disk is a draft until every row has been explicitly kept, changed, deferred, or edited by the user. Update the file row-by-row, not in a single batch at the end.
