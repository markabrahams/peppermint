---
name: defprod-implement-product
description: Scaffolds a project and implements a product definition area-by-area from DefProd. Reads the brief, architecture, and user stories via MCP, then generates a working codebase. Designed for repeated invocation — each run picks up where it left off.
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
  - mcp__defprod__getBriefForProduct
  - mcp__defprod__getArchitectureForProduct
  - mcp__defprod__getArchitectureTree
  - mcp__defprod__listAreas
  - mcp__defprod__listUserStories
  - mcp__defprod__getUserStory
  - mcp__defprod__patchProduct
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

# Implement Product

Turns a DefProd product definition into a working codebase. Scaffolds the project (framework, build tooling, directory structure) then implements each area's user stories sequentially. Designed for **repeated invocation** — each run assesses current progress and advances to the next incomplete phase.

## When to use

- When the user wants to build a new codebase from a DefProd product definition.
- When invoked via `/defprod-implement-product <product-name>`.
- After completing the "Create a Product" guide in DefProd.

## Inputs

- **Product name** — the product name as defined in DefProd. Must match an existing product.

---

## Status Check (every invocation)

**Always start here.** Before doing any work, assess where this product stands.

### 1. Query DefProd for current state

Search for the product in DefProd using the DefProd MCP server:
- Call `listProducts` — look for a product matching this name.
- If not found, stop and tell the user.
- If found, extract `productId` and call:
  - `getProduct` — get product metadata including `implementationStatus` and `implementationProgress`
  - `getBriefForProduct` — read the full brief (problem, users, requirements, aesthetics)
  - `getArchitectureForProduct` + `getArchitectureTree` — read architecture guidance
  - `listAreas` — get all areas
  - `listUserStories` — get all stories grouped by area

### 2. Present status summary

Display a status table:

```
Product: <name>
DefProd ID: <id>
Implementation Status: idle | implementing | implemented

Phase 1 — Scaffold:     [check] Complete | [ ] Not started
Phase 2 — Areas:         [check] Complete | [progress] In progress (X of Y areas) | [ ] Not started
  <AREA-KEY>: Implemented | Implementing | Pending
  <AREA-KEY>: Implemented | Implementing | Pending
  ...
```

### 3. Determine next phase and proceed

Identify the first incomplete phase and proceed. If the product is already fully implemented, present the summary and ask if the user wants to re-implement any area.

---

## Phase 1 — Scaffold

**Gate**: Project does not yet have a framework setup (no `package.json` or equivalent).

**Skips if**: Project already exists (has `package.json`, `pyproject.toml`, `Cargo.toml`, or equivalent).

### 1a. Infer tech stack

Read the product brief and architecture from DefProd:
- Extract technology mentions from the brief's requirements and aesthetics
- Read the architecture tree for explicit technology choices (frameworks, databases, languages)
- If the architecture specifies a tech stack, follow it exactly

If no explicit tech stack is specified, infer from the brief:
- Web app with rich UI → React/Next.js or Vue/Nuxt (recommend React/Next.js)
- API/backend service → Node.js/Express or Python/FastAPI (recommend Node.js/Express)
- Full-stack app → Next.js (recommend)
- Mobile app → React Native (recommend)
- CLI tool → Node.js (recommend)

### 1b. Present recommended stack for approval

```
Based on your product brief and architecture, I recommend:

Framework: Next.js 14 (App Router)
Language: TypeScript
Styling: Tailwind CSS
Database: PostgreSQL with Prisma ORM
Testing: Vitest + Playwright

[Confirm] or describe your preferred stack.
```

Wait for user approval before proceeding.

### 1c. Scaffold the project

1. `git init` (if not already a git repo)
2. Initialise the chosen framework (e.g. `npx create-next-app@latest`)
3. Install core dependencies
4. Set up directory structure aligned with the architecture tree
5. Configure build tooling, linting, formatting
6. Create initial configuration files
7. Commit the foundation: `git add -A && git commit -m "Initial project scaffold"`

### 1d. Update progress

Call `patchProduct` via the DefProd MCP server:
```json
[
  { "op": "replace", "path": "/implementationStatus", "value": "implementing" },
  { "op": "add", "path": "/implementationProgress", "value": { "scaffolded": true, "areas": [] } }
]
```

---

## Phase 2 — Implement Areas

**Gate**: Project is scaffolded (Phase 1 complete or skipped).

### 2a. Present area selection

Show which areas have been implemented and which are pending:

```
Implementation progress: 2 of 8 areas.

Implemented:
  [check] AUTH (6 stories)
  [check] DASH (4 stories)

Pending:
  [ ] CUST (next, 12 stories)
  [ ] ORD (10 stories)
  [ ] BILL (8 stories)
  ...

How many areas would you like to implement this session?
  (a) Just CUST (recommended for limited context windows)
  (b) A specific number of areas (e.g. 3)
  (c) All remaining areas (6 — best with large context windows)
```

Wait for the user's choice before proceeding.

### 2b. Initialise area progress

Call `patchProduct` to populate the `implementationProgress.areas` array with all selected areas set to `pending`:

```json
[
  { "op": "add", "path": "/implementationProgress", "value": {
    "scaffolded": true,
    "areas": [
      { "areaId": "AREA-...", "key": "AUTH", "status": "implemented" },
      { "areaId": "AREA-...", "key": "CUST", "status": "pending" },
      { "areaId": "AREA-...", "key": "ORD", "status": "pending" }
    ]
  }}
]
```

Always send the **full areas array** — include previously implemented areas with their current status.

### 2c. Process each area

For each selected area, in order:

#### Read the area's definition

1. Fetch all user stories and acceptance criteria for this area from DefProd MCP.
2. Read the product brief for context (personas, requirements, aesthetics).
3. Read the architecture tree for technical guidance on this area's components.

#### Update area status to implementing

Call `patchProduct`:
```json
[
  { "op": "add", "path": "/implementationProgress", "value": {
    "scaffolded": true,
    "areas": [
      { "areaId": "AREA-...", "key": "AUTH", "status": "implemented" },
      { "areaId": "AREA-...", "key": "CUST", "status": "implementing" },
      { "areaId": "AREA-...", "key": "ORD", "status": "pending" }
    ]
  }}
]
```

#### Implement stories sequentially

Implement each user story in the area, building on the previous one:

1. Read the story's acceptance criteria carefully.
2. Write the code to satisfy all acceptance criteria.
3. Follow the architecture guidance for component structure, naming, and patterns.
4. Follow the brief's aesthetics for UI styling (tone, visual style, interaction principles).
5. Write clean, well-structured code with appropriate comments.

#### Verify the area

After implementing all stories in the area:

1. Start the dev server and verify the implemented features work.
2. Run any existing tests.
3. Create basic e2e tests for the area's key user flows.
4. **Skip verification for stories where dependencies aren't built yet** (e.g. area depends on another area not yet implemented). Note these for re-verification later.

#### Update area status to implemented

Call `patchProduct` with the area's status changed to `implemented`.

### 2d. Session complete

After processing all selected areas, present a summary:

```
Session complete. Implemented N areas this session.

Overall progress: X of Y areas now implemented.
Remaining: Z areas pending.
```

If areas remain, remind the user to re-invoke `/defprod-implement-product <name>` to continue.

### 2e. All areas complete

When all areas are implemented, call `patchProduct`:
```json
[
  { "op": "replace", "path": "/implementationStatus", "value": "implemented" }
]
```

---

## Progress Tracking

Throughout the implementation workflow, update the product's `implementationStatus` and `implementationProgress` via `patchProduct` to signal the guide UI about progress. The guide UI listens to ProductRoom socket events and updates in real time.

**Key states:**
- `implementationStatus: 'idle'` — no implementation started
- `implementationStatus: 'implementing'` — actively implementing
- `implementationStatus: 'implemented'` — all areas complete

**Progress structure:**
```json
{
  "scaffolded": true,
  "areas": [
    { "areaId": "AREA-...", "key": "AUTH", "status": "implemented" },
    { "areaId": "AREA-...", "key": "CUST", "status": "implementing" },
    { "areaId": "AREA-...", "key": "ORD", "status": "pending" }
  ]
}
```

Always send the **full progress object** on each update — do not attempt to patch individual array items. This ensures idempotent progress even if the skill is re-invoked.

---

## Properties

- **Idempotent**: Re-running checks current state and picks up where it left off.
- **Works for greenfield and existing codebases**: Scaffolding skips if project exists; implementation skips already-implemented areas.
- **Session scope**: User controls pace by choosing which areas to implement per invocation.
- **Progress via `patchProduct`**: No new MCP tools needed; backend Socket.IO events propagate automatically.

---

## Content Principles

### Follow the definition faithfully

The product brief, architecture, and user stories are the specification. Implement what they describe. If something seems wrong or incomplete, ask the user rather than improvising.

### Architecture-first

When the architecture tree specifies component structure, naming patterns, or technology choices, follow them. The architecture is authoritative for technical decisions.

### Area-by-area natural review points

Each area is a natural pause point. The user can review, test, and provide feedback before the next area begins. This keeps context windows manageable and gives the user control.

### NZ/British English

Use NZ/British English spelling throughout (e.g. "organisation", "colour", "behaviour", "analyse").

---

## Rules

- **Always start with the status check** — never skip it.
- **Never re-implement completed areas** without user confirmation.
- **Session scope is the user's choice** — always ask how many areas to process.
- **Scaffold phase is approval-gated** — always present the recommended tech stack and wait for confirmation.
- **Update progress after every state change** — the guide UI depends on these updates.
- **Skip verification for unbuilt dependencies** — note them for later re-verification.
- **Commit after each area** — create a meaningful commit after each area is implemented.
