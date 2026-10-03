---
name: defprod-create-area-tests
description: Surface-aware dispatcher that walks every story in a product area, groups them by effective `surface` (explicit `UserStory.surface` or story-key-prefix inference), and routes each group to the matching `defprod-create-<surface>-tests` sibling skill. Use when you want to cover an entire area in one go without picking the right per-surface skill yourself.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash(ls:*)
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

# Create Area Tests (Dispatcher)

Routes test generation for every story in a product area to the right per-surface sibling skill. This skill does **not** generate tests itself — it inspects each story's `surface` (explicit or inferred), groups stories by surface, presents the routing plan to the user, and on confirmation invokes the matching `defprod-create-<surface>-tests` skill once per surface group.

This is the **dispatcher** counterpart to:
- `defprod-create-ui-tests` (Playwright e2e, target surface `ui`)
- `defprod-create-api-tests` (Vitest/Jest HTTP, target surface `api`)
- `defprod-create-mcp-tests` (Vitest + `@modelcontextprotocol/sdk`, target surface `mcp`)
- `defprod-create-cli-tests` (Jest subprocess, target surface `cli`)

Each sibling skill applies its own surface filter internally, so re-invoking a sibling on the same area is idempotent — only its own surface stories are picked up.

## When to use

- When a product area contains stories on **multiple surfaces** (e.g. `STATUS`, where STATUS-01 is API, STATUS-02 is UI, STATUS-03 is CLI) and you want one invocation to cover the whole area.
- When you want a per-area coverage view across all surfaces in one pass without picking which sibling skill to invoke.
- When invoked via `/defprod-create-area-tests <product-name> <area-key>`.

If you already know the area is single-surface (e.g. UI-only), it is more direct to invoke the matching sibling skill (`/defprod-create-ui-tests`, `/defprod-create-api-tests`, …) yourself — the dispatcher gives you no benefit when the area only routes to one sibling.

## Inputs

- **Product name** — the DefProd product.
- **Area key** — the area to walk (e.g. `STATUS`, `BILLING`, `CORE`).

## Config

This skill reads no config keys directly. Each sibling skill consults its own `.defprod/defprod.json` entries (`frontendApp`, `backendApp`, `mcpApp`, `cliApp`, …) when it runs.

---

## Workflow

### Phase 1 — Resolve

#### 1a. Find the product, area, and stories

1. Call `listProducts` from the DefProd MCP server. Find the product by name. Halt if not found.
2. Call `listAreas` for the product. Find the area by key. Halt if not found.
3. Call `listUserStories` for the area. If the area has no stories, halt and suggest `/defprod-create-area-stories`.

### Phase 2 — Group by effective surface

For every story returned by `listUserStories`, determine its **effective surface**:

1. If `story.surface` is set explicitly, use that value. Tag as `(explicit)`.
2. Otherwise, fall back to the story-key prefix mapping from `libs/defprod-common/src/lib/modules/user-story/infer-surface-from-story-key.util.ts`:
   - `API-…` infers `api`, `MCP-…` infers `mcp`, `CLI-…` infers `cli`.
   - Any other prefix infers nothing → effective surface is **`unspecified`**.
   - Tag inferred values as `(inferred)`.
3. Group stories by effective surface. Buckets:
   - `ui` → routes to `defprod-create-ui-tests`
   - `api` → routes to `defprod-create-api-tests`
   - `mcp` → routes to `defprod-create-mcp-tests`
   - `cli` → routes to `defprod-create-cli-tests`
   - `script`, `library`, `other`, `unspecified` → **no current routing target**. Surface to the user as "deferred" — they need a manual decision (e.g. set the surface explicitly, write tests by hand, or treat it as out of scope).

### Phase 3 — Present routing plan

Present a routing table to the user:

| Surface | Sibling skill | Stories | Inference-routed | Action |
|---|---|---:|---:|---|
| `ui` | `defprod-create-ui-tests` | N | n | Will invoke |
| `api` | `defprod-create-api-tests` | N | n | Will invoke |
| `mcp` | `defprod-create-mcp-tests` | N | n | Will invoke |
| `cli` | `defprod-create-cli-tests` | N | n | Will invoke |
| `script`/`library`/`other` | — | N | — | Deferred (no skill) |
| `unspecified` | — | N | — | Deferred (set `surface` or treat as out of scope) |

List the story keys in each row (or, if there are many, the first few and a count) so the user can see what is going where.

Ask:

> Proceed to invoke the sibling skills for the surfaces with routing targets?
> - **Proceed all** — invoke each sibling skill in turn (ui → api → mcp → cli, in that order, skipping empty buckets)
> - **Pick subset** — choose which surfaces to invoke now
> - **Cancel** — exit without invoking

### Phase 4 — Invoke siblings

For each in-scope surface (in the order `ui`, `api`, `mcp`, `cli`), invoke the matching sibling skill via the `Skill` tool (or, if the user is driving manually, instruct them to run the skill themselves):

- `Skill: defprod-create-ui-tests` with the same `<product-name> <area-key>` arguments
- `Skill: defprod-create-api-tests` with the same arguments
- `Skill: defprod-create-mcp-tests` with the same arguments
- `Skill: defprod-create-cli-tests` with the same arguments

Each sibling re-applies its own surface filter, so passing the whole area key is safe — the sibling will only operate on its own stories.

Between invocations, surface a one-line status update to the user so they can see progress:

> Routing `<surface>`: invoking `defprod-create-<surface>-tests` on `<N>` stories…

If any sibling skill halts (no harness scaffolding present, backend down, missing built binary, etc.), let it halt cleanly and continue with the next surface. Record the halt in the final summary.

### Phase 5 — Final summary

After all in-scope siblings have run, present:

| Surface | Sibling skill | Stories in scope | Specs created | Tests passing | Status |
|---|---|---:|---:|---:|---|
| `ui` | `defprod-create-ui-tests` | N | M | P | ok / partial / halted (reason) |
| `api` | `defprod-create-api-tests` | … | … | … | … |
| `mcp` | `defprod-create-mcp-tests` | … | … | … | … |
| `cli` | `defprod-create-cli-tests` | … | … | … | … |

**Deferred** (no routing target):
- `<STORY-KEY>` — `<surface>` (explicit|inferred|unspecified). Set `surface` explicitly or treat as out of scope.

**Inference-routed across the area** (candidates for `surface` backfill):
- `<STORY-KEY>` — routed to `<surface>` via story-key prefix; no explicit value set.

Suggest:

> **Next steps:**
> - Backfill `surface` on inference-routed stories you want to lock in.
> - Run `/defprod-run-area-tests <area>` to re-run the generated tests and get a classified failure report.
> - For deferred stories, decide whether to add a `script` / `library` test harness or treat as out of scope.

---

## Rules

- **The dispatcher never generates tests itself.** All test creation is delegated to the per-surface siblings. This keeps the dispatcher small and the siblings authoritative for their surfaces.
- **Group, don't filter.** Every story in the area must land in exactly one routing bucket (one of the four routed surfaces, the deferred bucket, or unspecified). Never silently drop a story.
- **Order matters for output stability.** Invoke siblings in the order `ui` → `api` → `mcp` → `cli` so the final summary is consistent across runs.
- **Sibling halts are not fatal.** If `defprod-create-ui-tests` halts because the Playwright config is missing, continue to `defprod-create-api-tests`. Surface the halt in the final summary, not as a dispatcher-level failure.
- **Inference is display-only.** Per CORE-38, never persist an inferred `surface` back to the story — the dispatcher only uses inference to decide routing, never to write.
- **Deferred buckets are visible.** Stories with `surface` in `{script, library, other, unspecified}` go in the deferred bucket and are listed in both Phase 3 and Phase 5 so the user sees the gap.
