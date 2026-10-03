---
name: defprod-create-definition
description: Populates a DefProd product definition (brief + areas) by analysing documentation and codebase. Use when a product exists but its definition is empty.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash(ls:*)
  - Bash(cat:*)
  - AskUserQuestion
  - mcp__defprod__listProducts
  - mcp__defprod__getProduct
  - mcp__defprod__getBriefForProduct
  - mcp__defprod__patchBrief
  - mcp__defprod__listAreas
  - mcp__defprod__createArea
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

# Create Definition

Populates an existing DefProd product's brief and areas by analysing the project's documentation and codebase. This is a focused skill that handles only brief + areas — use `/defprod-create-area-stories` afterwards to populate user stories.

## When to use

- When a DefProd product has been created but its brief and areas are empty.
- When you want to quickly populate the high-level definition without going through the full `/defprod-onboard-product` workflow.
- When invoked via `/defprod-create-definition <product-name>`.

## Inputs

- **Product name** — the DefProd product to populate.

## Config

This skill consults `.defprod/defprod.json` for optional hints. If the file doesn't exist, the skill discovers everything automatically.

| Key | Type | Purpose |
|-----|------|---------|
| `products[].name` | `string` | Matches product name to locate config entry |
| `products[].frontendApp` | `string` | Path to the frontend app — focuses research |
| `products[].backendApp` | `string` | Path to the backend app — focuses research |

---

## Workflow

### Phase 1 — Research

Build a comprehensive understanding of the product before writing anything.

#### 1a. Resolve the product

1. Call `listProducts` from the DefProd MCP server. Find the product matching the provided name.
2. Call `getBriefForProduct` — check what's already populated. Never overwrite existing content.
3. Call `listAreas` — check if areas already exist.

If the product doesn't exist, tell the user and suggest creating it first (via `/defprod-onboard-product` or manually).

#### 1b. Check config

Read `.defprod/defprod.json` if it exists. If the product has a config entry with `frontendApp` or `backendApp`, use those paths to focus research.

#### 1c. Read documentation

Read all available documentation files:
- README files (root and package-level)
- `docs/` directory contents
- CONTRIBUTING guides
- API documentation
- Architecture decision records (ADRs)
- Changelog or release notes

#### 1d. Analyse codebase structure

Explore the source code to understand:
- Project structure — key directories, entry points, module boundaries
- Route definitions — what pages/endpoints exist
- Data models — what entities the system manages
- Technology stack — frameworks, libraries, patterns
- User-facing capabilities — what can users actually do

#### 1e. Build mental model

Before writing anything, synthesise your research into a clear picture:
- What does this product do?
- Who uses it and why?
- What are the major functional areas?
- What makes this product distinct?

---

### Phase 2 — Populate Brief

Call `patchBrief` from the DefProd MCP server to populate the product brief. Split into **three calls** to keep payloads manageable:

#### Call 1 — Core identity

- `description` — 2-3 sentences: what the product does and who it serves
- `positioning` — 1-3 sentences: market category, differentiation, and competitive context
- `problem.summary` — the core problem this product solves
- `problem.context` — background, market context, or motivation
- `problem.impact` — consequences of the problem being unsolved

#### Call 2 — Users and requirements

- `users` — 2-5 user personas, each with:
  - `name` — persona name (e.g. "Engineering Lead", "New Team Member")
  - `description` — who they are
  - `goals` — what they want to achieve
  - `painPoints` — frustrations the product addresses
- `requirements` — 10-25 functional requirements that describe what the product does

#### Call 3 — Success criteria and aesthetics

- `successCriteria` — 5-10 measurable criteria for product success
- `outOfScope` — explicit boundaries (what this product does NOT do)
- `aesthetics.tone` — communication tone
- `aesthetics.style` — visual and interaction style
- `aesthetics.principles` — 3-5 guiding design principles
- `references` — links to documentation, design systems, or resources

#### Present for review

Show the user a summary of the brief. Ask for feedback before proceeding to areas.

---

### Phase 3 — Create Areas

#### 3a. Identify areas

Based on the codebase research, identify the product's logical areas. Look for:
- Route groups or navigation structure
- Feature modules or domain boundaries
- Distinct user workflows
- Admin vs user-facing sections

#### 3b. Propose areas

Present a table:

| Area | Key | Description |
|------|-----|-------------|
| ... | ... | ... |

Ask the user to confirm, add, remove, or rename before creating.

#### 3c. Create areas

For each confirmed area, call `createArea` with:
- `name` — human-readable area name
- `key` — short uppercase identifier (e.g. `AUTH`, `BILLING`)
- `productId` — the product ID
- `description` — 1-2 sentences defining the area's scope

#### Present summary

Show the completed definition:
- Brief summary
- Areas created (count and names)

Suggest next steps:

> **Next steps:** Run `/defprod-create-area-stories <area>` for each area to populate user stories with acceptance criteria.

---

## Rules

- **Never overwrite existing content** — if the brief already has fields populated, leave them as-is. Only fill in empty fields.
- **Every field must be populated** — don't leave brief sections empty. If you can't determine a value from the codebase, make a reasonable inference and flag it for the user to review.
- **No artificial caps on areas** — if the product has 15 distinct areas, create 15 areas. Don't compress into an arbitrary number.
- **Requirements describe capabilities** — write them as "The system does X" or "Users can Y", not as future aspirations.
- **Batch MCP calls** where possible to minimise round trips.
- **Always confirm with the user** before creating areas. The brief can be populated directly (the user reviews it after), but areas represent structural decisions that benefit from confirmation.
