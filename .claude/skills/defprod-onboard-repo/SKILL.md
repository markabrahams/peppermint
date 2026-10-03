---
name: defprod-onboard-repo
description: Discovers apps and libraries in a repository (monorepo or single-app), proposes a product-to-package mapping, and writes an onboarding document. Use when onboarding any repository into DefProd.
allowed-tools:                              # Used by Claude Code for tool permissions — other agents ignore this field
  - Read
  - Glob
  - Grep
  - Bash(ls:*)
  - Bash(cat:*)
  - Write
  - Edit
  - AskUserQuestion
  - mcp__defprod__listProducts
  - mcp__defprod__listRepos
  - mcp__defprod__patchRepo
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

# Onboard Repo

Scans a repository to discover applications and libraries, proposes which applications should become DefProd products, and writes a confirmed onboarding document that drives per-product definition. Works for both multi-product monorepos and single-app repositories.

## When to use

- When onboarding an existing repository into DefProd.
- When invoked via `/defprod-onboard-repo`.

## Inputs

Run this skill from the **root of the repository**. No other inputs required — the skill discovers everything from the workspace configuration and codebase.

## Config

This skill consults `.defprod/defprod.json` for optional hints. If the file doesn't exist, the skill discovers everything automatically.

| Key | Type | Purpose |
|-----|------|---------|
| `products[].name` | `string` | Product name — used to pre-populate product candidates |
| `products[].frontendApp` | `string` | Path to the frontend app (e.g. `apps/web`) — skips classification heuristics |
| `products[].backendApp` | `string` | Path to the backend app (e.g. `apps/api`) — skips classification heuristics |

These keys help the skill classify packages faster and pre-seed the product proposal. Without them, it uses heuristics (see Phase 1).

---

## Workflow

### Phase 1 — Discover packages

Scan the repository to build a complete inventory of every package.

#### 1a. Check config first

Read `.defprod/defprod.json` if it exists. If `products` entries are defined with `frontendApp` or `backendApp` paths, use them as known app paths — skip classification heuristics for those packages. If `products[].name` values are set, use them to pre-seed product candidates in Phase 2.

#### 1b. Read workspace configuration

Look for and read (in priority order):

- `nx.json` + `project.json` files — Nx workspace
- `pnpm-workspace.yaml` — pnpm workspaces
- `turbo.json` — Turborepo
- Root `package.json` `workspaces` field — npm/yarn workspaces
- `lerna.json` — Lerna
- `rush.json` — Rush
- `angular.json` or `workspace.json` — Angular CLI workspace

If multiple configs exist, reconcile them — they typically describe the same packages.

If none of the above exist, check for a single root-level application:
- A root `package.json` (or equivalent manifest: `pyproject.toml`, `Cargo.toml`, `go.mod`) with `start`, `serve`, or `dev` scripts
- A `src/` directory or similar application entry point

If the repo is a **single-app repository** (one application, no workspace configuration), classify it as a single app and skip to Phase 1c with that single entry. Libraries within the repo (e.g. a `libs/` or `packages/` directory) should still be discovered and recorded.

If none of the above apply, fall back to scanning for directories that contain a `package.json` (or equivalent manifest).

#### 1c. Enumerate all packages

For each package discovered, record:

- **Name** — the package name from the manifest
- **Path** — relative path from repo root (e.g. `apps/customer-portal`, `libs/shared-utils`, or `.` for a single-app repo root)
- **Type** — classify as `app` or `library` using these heuristics:
  - **App indicators**: lives under `apps/`, `services/`, or similar; has `start`, `serve`, or `dev` scripts; has route definitions, controllers, or entry-point components; is not imported by other packages as a dependency
  - **Library indicators**: lives under `libs/`, `packages/`, or `shared/`; is listed as a dependency by other packages; exports types, utilities, or components consumed elsewhere; has no standalone `start` script
  - For a **single-app repo**, the root package is an `app`
  - If uncertain, classify as `uncertain` and flag for user decision
- **Description** — a 1-sentence summary derived from the manifest `description` field, README first paragraph, or the package's apparent purpose from its source structure

#### 1d. Map library usage

For each library, identify which apps consume it. Check:
- `package.json` dependencies/devDependencies across all apps
- TypeScript path aliases (`tsconfig.json` paths)
- Import statements in app source files (sample — don't exhaustively scan)

Record this as a "used by" list for each library.

For a **single-app repo with no libraries**, skip this step.

#### 1e. Sync discovered packages to DefProd

After discovering all packages, update the repository entity in DefProd so the guide UI reflects progress in real time.

1. Call `listRepos` from the `defprod` MCP server to find the repo matching this repository (match by URL or name).
2. If a matching repo is found:
   - **First**, call `patchRepo` to set `onboardingStatus` to `analysing` (signals to the guide UI that the skill is actively running):
     ```json
     [{ "op": "replace", "path": "/onboardingStatus", "value": "analysing" }]
     ```
   - **Then**, call `patchRepo` to set `/discoveredPackages` to the full list of discovered packages. Each entry must include `name`, `path`, and `type` (`app` or `library`):
     ```json
     [
       {
         "op": "replace",
         "path": "/discoveredPackages",
         "value": [
           { "name": "customer-portal", "path": "apps/customer-portal", "type": "app" },
           { "name": "shared-utils", "path": "libs/shared-utils", "type": "library" }
         ]
       }
     ]
     ```
   - **Record the repo ID** — store the matched repo's `_id` for use in Phase 4 (writing the onboarding document) and later by `/defprod-onboard-product`.

If no matching repo is found, skip this step — the user may not have created the repo entity yet (e.g. running the skill outside the guide flow). The skill continues to work without a repo entity.

---

### Phase 2 — Propose products

#### 2a. Generate product candidates

For each **app**, create a product candidate:

- **Product name** — a human-readable name derived from the package name, README title, or app purpose. Clean up technical names (e.g. `feenix-portal` → "Feenix Portal", `admin-api` → "Admin API").
- **Mapping type** — initially `single` (one package = one product).
- **Packages** — the app's path.
- **Description** — 1-2 sentences explaining what this application does and who uses it.

Libraries are **not** product candidates. They will appear in architecture elements later.

#### 2b. Present to user

**For a single-app repository** (one app, zero or few libraries), present a simplified view:

**1. Repository — Application**

| Package | Path | Description |
|---|---|---|
| ... | ... | ... |

**2. Repository — Libraries** *(only if libraries were discovered)*

| Package | Path | Description |
|---|---|---|
| ... | ... | ... |

**3. Proposed Product**

| Product | Mapping | Package(s) |
|---|---|---|
| ... | single | ... |

Then ask the user:

> **Please review the proposed product.** You can:
> - **Confirm** — proceed as-is
> - **Split** — split the app into multiple products (e.g. admin and customer sides)
> - **Rename** — change the product name
> - **Add notes** — add context that will help during definition

**For a multi-app repository (monorepo)**, present the full view:

**1. Repository Inventory — Applications**

| Package | Path | Description |
|---|---|---|
| ... | ... | ... |

**2. Repository Inventory — Libraries**

| Package | Path | Used By |
|---|---|---|
| ... | ... | ... |

**3. Proposed Products**

| Product | Mapping | Package(s) |
|---|---|---|
| ... | ... | ... |

Then ask the user to review and adjust:

> **Please review the proposed product list.** You can:
> - **Confirm** — proceed as-is
> - **Split** — a single package into multiple products (e.g. admin and customer sides of one app)
> - **Merge** — multiple packages into one product (e.g. API gateway + worker = one product)
> - **Remove** — exclude a package that shouldn't be a DefProd product
> - **Rename** — change a product name
> - **Add notes** — add context for any product that will help during definition

---

#### 2c. Sync confirmed products to DefProd

After the user confirms the product list, if a matching repo was found in Phase 1e:

1. **Match against existing products**: Call `listProducts` to retrieve all products for the team. For each confirmed product candidate, check if a DefProd product with the same name already exists. If it does, record its `_id` as the `productId` for that entry.

2. **Patch the repo**: Call `patchRepo` to set the `onboardingStatus` to `confirmed` and populate the `products` array in a single patch call. Include `productId` for any products that were matched in step 1:

```json
[
  { "op": "replace", "path": "/onboardingStatus", "value": "confirmed" },
  {
    "op": "replace",
    "path": "/products",
    "value": [
      {
        "name": "Customer Portal",
        "mappingType": "single",
        "packagePaths": ["apps/customer-portal"],
        "productId": "6612f1a..."
      },
      {
        "name": "Admin API",
        "mappingType": "single",
        "packagePaths": ["apps/admin-api"]
      }
    ]
  }
]
```

Each entry in `products` must include `name` (the human-readable product name), `mappingType` (`single`, `partial`, or `multiple`), and `packagePaths` (array of package paths). Include `productId` when a matching product already exists in DefProd. Omit `productId` for products that don't exist yet — the backend populates it automatically when `/defprod-onboard-product` creates each product.

If no matching repo was found in Phase 1e, skip this step.

---

### Phase 3 — Capture detailed mappings

After the user confirms the product list, gather any additional detail needed for products with non-trivial mappings.

#### For `partial` mappings (one package, multiple products):

Ask the user to specify or confirm:
- **Include paths** — glob patterns for the directories/files that belong to this product (e.g. `apps/angular-app/src/app/admin/**`)
- **Exclude paths** — glob patterns for directories/files that do NOT belong (e.g. `apps/angular-app/src/app/customer/**`)

If the user doesn't specify exact paths, propose them based on directory structure inspection.

#### For `multiple` mappings (multiple packages, one product):

Confirm which packages are included and their roles (e.g. "API gateway handles routing, worker handles background jobs").

#### For `single` mappings:

No additional detail needed — the whole package is the product.

---

### Phase 4 — Write onboarding document

Write the confirmed onboarding document to `docs/defprod-onboarding.md` in the repo root.

#### Document structure

```markdown
# DefProd Onboarding — <Repo Name>

> Generated by /defprod-onboard-repo on <date>.
> This document drives the /defprod-onboard-product skill for per-product definition.

**Repo ID**: `<repo-id or "none">`

## Repository Inventory

### Applications

| Package | Path | Description |
|---|---|---|
| ... | ... | ... |

### Libraries

| Package | Path | Used By |
|---|---|---|
| ... | ... | ... |

## Products Overview

| Product | Mapping | Package(s) |
|---|---|---|
| ... | single / partial / multiple | ... |

## Product Details

### <Product Name>

- **Mapping**: single | partial | multiple
- **Packages**: `<path>` [, `<path>`, ...]
- **Description**: <1-2 sentence summary>
- **Include paths**: <glob patterns, or "whole package">
- **Exclude paths**: <glob patterns, or "none">
- **Notes**: <any context from the user — special considerations, shared modules, deployment topology, etc.>

### <Product Name>

...
```

#### Document rules

- **One section per product** in the Product Details section, in the same order as the overview table.
- **Include/exclude paths** must be explicit for partial mappings. Use glob syntax.
- **Libraries table** must list all libraries with their "used by" information — `/defprod-onboard-product` uses this to know which shared libraries to follow into for context. For single-app repos with no libraries, the Libraries table may be omitted or left empty.
- **Notes** — preserve any context the user provided during the review. If the user didn't provide notes, leave this field as "None".
- **Repo ID** — if a matching repo entity was found in Phase 1e, include its `_id` in the `Repo ID` field. If no repo entity was found, set this to `none`. `/defprod-onboard-product` uses this to link created products back to the repo.

---

### Phase 5 — Present summary and next steps

After writing the document, present:

1. Confirmation that `docs/defprod-onboarding.md` was written.
2. A summary: total apps, total libraries, total products.
3. Next steps:

> **Next steps**: For each product, run `/defprod-onboard-product <product-name>` to create the full DefProd product definition. Start with the smallest or simplest product to validate the process.
>
> Products to define:
> 1. <Product Name>
> 2. <Product Name>
> 3. ...

For a **single-product repo**, simplify:

> **Next step**: Run `/defprod-onboard-product <product-name>` to create the full DefProd product definition.

---

## Rules

- **Never create DefProd products** in this skill — product creation happens in `/defprod-onboard-product`. This skill only discovers and documents the mapping.
- **Always get user confirmation** before writing the onboarding document. Never write based on unconfirmed proposals.
- **Be conservative with classification** — if uncertain whether something is an app or library, classify as `uncertain` and let the user decide.
- **Respect existing onboarding documents** — if `docs/defprod-onboarding.md` already exists, read it first and ask the user whether to replace or update it.
- **Product names should be human-readable** — not technical package names. "Customer Portal" not "customer-portal". "Admin API" not "@acme/admin-api".
- **Description quality matters** — the descriptions in this document are used by `/defprod-onboard-product` to understand the product's purpose before diving into code. Make them informative.
- **Single-app repos are first-class** — do not treat them as a degenerate case. The workflow should feel natural: discover the app, confirm it as a product, write the onboarding document, proceed to `/defprod-onboard-product`.
