---
name: defprod-realize-product-from-template
description: Realize a new product from a DefProd template end-to-end — copy the template's definition, scaffold a working codebase from the template's linked starter Git repo, onboard and link the new repo, and verify it builds. Use when the user wants to create a product from a template that has an associated starter repo.
allowed-tools:                              # Used by Claude Code for tool permissions — other agents ignore this field
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
  - mcp__defprod__listRepos
  - mcp__defprod__patchProduct
  - mcp__defprod__patchRepo
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

# Realize Product from Template

Creates a new product from a DefProd **template** and scaffolds its codebase from the template's linked **starter Git repo**, then onboards and links the result.

A DefProd template is a product definition marked as a template. A template may *additionally* carry a starter Git repo — a battle-tested codebase (auth, payments, admin, app conventions) that new products can be scaffolded from instead of being built from scratch. This skill chains the definition copy and the code scaffold into one flow.

The realized product is **fully independent** of the template once this skill completes: the only thing DefProd retains is a non-functional `createdFromTemplateId` breadcrumb (stamped automatically when you create a product from a template). There is no live link and no upgrade path back to the template.

## When to use

- The user wants to "create a product from template X" or "spin up an app from the <name> starter".
- Invoked via `/defprod-realize-product-from-template`.

## Terminology

The word "repo" is overloaded. This skill concerns the **actual Git repository** that a template is linked to (via its `repoId`), not DefProd's internal data model. DefProd stores only the repo's metadata and a `bootstrapCommand`; it never clones or executes anything itself — your coding agent runs the command locally.

## Inputs

- The **template** to realize (a product with `isTemplate: true`). Accept a name or product ID; resolve with `listProducts({ isTemplate: true })` and confirm with the user if ambiguous.
- The **parent directory** on disk where the new repo should be created (default: the parent of the current working directory, i.e. `..`).
- The **app name** for the new product/repo (kebab-case). Default: the template name with a trailing " Template" removed, kebab-cased.

## Procedure

### 1. Resolve the template and its starter binding

1. Resolve the template product and confirm `isTemplate === true`.
2. Read the template's `repoId`. If set, `getRepo(repoId)` and read its `bootstrapCommand`.
3. Branch:
   - **Has `repoId` and `bootstrapCommand`** → full scaffold path (steps 2–5).
   - **No binding** → degrade gracefully: do the definition copy only (step 2), tell the user there is no starter repo to scaffold, and stop.

### 2. Copy the definition

Create a new product from the template. In the DefProd app, this is the **"Create product from template"** action on the template's detail view; it copies the full definition (brief, areas, stories, architecture) into a new product with `isTemplate: false` and stamps `createdFromTemplateId` automatically. When a starter repo is bound, the app also surfaces the exact bootstrap command to run.

Capture the **new product ID** and its **team ID** (`getProduct`).

### 3. Scaffold the codebase from the starter repo

1. Determine the app name (input or derived) and parent directory (input or `..`).
2. Resolve the `bootstrapCommand` template string: replace `{{appName}}` with the kebab-case app name and `{{parentDir}}` with the chosen parent directory. Example:
   ```
   npx tsx scripts/generate-app.ts .. acme-crm
   ```
3. **Show the resolved command to the user and get explicit confirmation before running it.** `bootstrapCommand` is an author-supplied shell string — never run an unexpected or untrusted command.
4. Run the command from the **root of the starter repo** (clone it first from `repo.url` if it is not already on disk). Confirm the new app directory was created.

### 4. Onboard and link the new repo

1. From the **root of the newly generated repo**, run the `defprod-onboard-repo` skill to discover packages.
2. Create/confirm a DefProd repo for the new repository **on the realized product's team** (the team from step 2) — not the template's team.
3. Link the new product to the new repo (the onboard flow links discovered packages to products; otherwise set the product's `repoId` + package path via `patchProduct`/`patchRepo`).

### 5. Verify

1. In the new repo, install dependencies if needed and run the project's build/typecheck/test command.
2. Report: new product ID, new repo location, link status, and build result.

## Notes

- **Cross-team:** always create/link the new repo on the realized product's team. The starter repo's URL and command are read as plain strings — the new product is never linked back to the starter's repo record, which keeps the realized product independent of the template.
- **No starter binding:** if the template has no linked repo or no `bootstrapCommand`, this skill behaves exactly like the plain "create product from template" action (definition copy only).
- **Trust:** `bootstrapCommand` is executed locally by your agent. Only run starter repos and commands you trust.
