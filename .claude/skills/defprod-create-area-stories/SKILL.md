---
name: defprod-create-area-stories
description: Creates comprehensive user stories with acceptance criteria for a single product area by analysing the codebase. Use when an area exists but has no stories.
allowed-tools:
  - Read
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
  - mcp__defprod__createUserStory
  - mcp__defprod__patchUserStory
  - mcp__defprod__getBriefForProduct
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

# Create Area Stories

Creates comprehensive user stories with acceptance criteria for a single product area by deep-diving the relevant source code, routes, tests, and documentation. Each story describes an existing, implemented capability.

## When to use

- When a product area has been created but has no user stories.
- When you want to populate stories for a specific area without running the full `/defprod-onboard-product` workflow.
- When invoked via `/defprod-create-area-stories <product-name> <area-key>`.

## Inputs

- **Product name** — the DefProd product.
- **Area key** — the area to populate (e.g. `AUTH`, `BILLING`).
- **`--no-decompose`** *(optional flag)* — skip Phase 2d decomposition pass in Add-more mode. Existing stories are preserved untouched; new stories are added only for matrix rows not already covered.

## Config

This skill consults `.defprod/defprod.json` for optional hints. If the file doesn't exist, the skill discovers everything automatically.

| Key | Type | Purpose |
|-----|------|---------|
| `products[].name` | `string` | Matches product name to locate config entry |
| `products[].frontendApp` | `string` | Path to the frontend app — focuses research |
| `products[].backendApp` | `string` | Path to the backend app — focuses research |

---

## Workflow

### Phase 1 — Resolve

#### 1a. Find the product and area

1. Call `listProducts` from the DefProd MCP server. Find the product by name.
2. Call `listAreas` for the product. Find the area by key.
3. Call `getBriefForProduct` — read the brief for context on the product's purpose and users.

If the product or area doesn't exist, tell the user and suggest creating it first.

#### 1b. Check for existing stories

Call `listUserStories` for the area. If stories already exist:

> **This area already has {count} user stories.** Would you like to:
> - **Add more** — I'll research and create stories for capabilities not yet covered
> - **Replace all** — I'll delete existing stories and create a fresh set
> - **Cancel** — keep the existing stories as-is

If adding more, note the existing story keys and titles to avoid duplicates.

---

### Phase 2 — Research

Deep-dive the source code for this area to understand every capability.

#### 2a. Map area to source modules

Identify which source files, directories, and modules correspond to this area. Use:
- `.defprod/defprod.json` paths (if available)
- The area description and name to locate relevant code
- Route definitions, navigation structure, and module boundaries

#### 2b. Research each module

For each relevant module, read:

- **Routes and pages** — what views/endpoints exist
- **Controllers/handlers** — what operations are available
- **Services** — business logic and data operations
- **Data models/schemas** — what entities are managed
- **Test files** — these reveal expected behaviour and edge cases
- **UI components and forms** — what users interact with
- **Permissions** — access control rules and role checks
- **Validation rules** — input constraints and business rules

#### 2c. Build a capability matrix

Before writing any stories, enumerate every entity the area exposes and record which verbs the codebase actually supports for it, **including each meaningful variant of each verb**. This matrix is the **required coverage checklist** for Phase 3 — every populated row must be backed by ≥1 story before the area is considered done.

| Entity | Verb | Variant | Permission / feature gate | Validations |
|---|---|---|---|---|

**The Variant column is mandatory.** A verb behaves differently under different conditions — those conditions are separate matrix rows, not footnotes. Variants to enumerate explicitly:

- **State variants** — each state transition is its own row (activate / decommission / suspend / migrate / archive / restore)
- **Configuration variants** — same verb under different tenant settings, feature flags, or configuration values (e.g. "hide Supplier Reference column when tenant does not expose it" is a separate row from "show standard column set")
- **Role / permission variants** — same verb scoped to a different role / feature-permission key (e.g. "VISP can run a line test on subscriber inventory" vs "retail user can request a Move on a service component")
- **Dialog / wizard variants** — same outcome reached via a different dialog or wizard (e.g. "new Modify Service dialog" vs "legacy Modify Service dialog", each its own row)
- **Filter / sort / export / column-toggle variants** — each interaction that materially changes the rendered list is a row
- **Sub-action variants on detail pages** — each distinct button / menu item that invokes a discrete operation is a row

If the area description mentions "manage", "CRUD", "provisioning", "configuration", or similar, the matrix must show the write verbs. If the codebase does NOT expose writes but the area description implies them, flag this to the user and consider updating the area description — the area is read-only in reality.

**Sanity check:** a typical area in a moderately complex frontend produces 15–35 matrix rows. If your matrix has fewer than 10 rows for a non-trivial area, you have likely collapsed variants — re-examine the codebase before proceeding.

---

#### 2d. Decomposition pass (Add-more mode only)

Skip this phase if you are populating a fresh area with no existing stories, or if the user invoked the skill with `--no-decompose`.

If the user chose **Add more** in Phase 1b, evaluate each existing story against the granularity contract (Phase 3.0) and the Phase 2c matrix:

**Granularity violations to detect:**
1. Title contains `and`, `or`, `if`, `depending on`, `across modes`, multiple distinct verbs, or multiple state transitions
2. The matrix contains ≥2 rows that this story alone is meant to cover
3. Acceptance criteria number > 12 and span multiple discrete capabilities (a single complex capability can legitimately produce 12+ ACs; multiple capabilities crammed together is the violation)

**For each violating story:**

1. Record the existing story: ID, title, ACs, persona, priority.
2. Plan the fine-grained replacement set — one new story per matrix row the parent should have covered.
3. **Check for external attachments** on the parent story via `getUserStory`:
   - Linked tests, code references, external system links, external comments
   - Any non-empty field indicating this story is referenced elsewhere
4. **Confirmation gate:**
   - Parent has external attachments → **explicit confirmation required.** Surface to the user: `Replace [STORY-KEY] '[title]' with N fine-grained stories: [list]. This story has external attachments — confirm before retiring?`
   - Parent has no external attachments → proceed without confirmation.
5. On approval (or no-attachment auto-approval):
   - Create the fine-grained replacements (in Phase 3, using the standard `createUserStory` flow)
   - Transfer relevant ACs from the parent to the appropriate child(ren)
   - Patch the parent to a tombstone status (use `patchUserStory` with `status: 'archived'` or equivalent — do not delete; tombstones preserve history and prevent dedup re-collisions in subsequent runs)

**Dedup behaviour:** when Phase 3 runs after Phase 2d, treat tombstoned stories as covered for dedup purposes — do not regenerate replacements for matrix rows whose tombstoned parent already has fine-grained children created in this run.

---

### Phase 3 — Create Stories

#### 3.0. Granularity contract (read before writing any story)

**One story = one acceptance-test cluster.**

- **Title format:** `As a <persona>, I want to <single verb-noun>`. If you need `and`, `or`, `if`, `depending on`, or `across modes` to describe the title, decompose into two or more stories.
- **Test-block test:** if a competent QA would write more than one automated E2E test block (Playwright `test('...')`, Cypress `it('...')`, etc.) for the happy path plus immediate variants, the story is too big — decompose further.
- **Force a separate story for each:**
  - State transition (activate, decommission, suspend, modify, migrate, archive, restore, …)
  - Configuration variant (tenant setting, feature flag, configuration value)
  - Role / feature-permission variant when the variant reaches the user-facing surface
  - Distinct dialog or wizard step that performs its own operation
  - Filter / sort / export / column-toggle that materially changes the rendered list
  - Sub-action button / menu item on a detail page that invokes a discrete operation

The matrix from Phase 2c is the enumeration scaffold — one row, one story (or one row, multiple stories if the verb has further nuance the variant column did not capture). The contract is what keeps you honest about *which* rows belong on the matrix in the first place.

**Anti-pattern to avoid:** "Browse services in the list or detailed view" — collapses list, detail, columns, filters, scoping, and exports into one. Decompose into the discrete capabilities each of those represents.

#### 3a. Establish the canonical persona list

Call `getBriefForProduct` (if not already fetched). Build the set of allowed persona titles from `brief.users[].title` (lowercased). Add one fallback: `"any signed-in user"` — usable for stories that cross personas (shell, profile, settings).

Every story's description MUST begin with `"As a <canonical persona>, I want to …"`. Do NOT invent ad-hoc persona variants ("customer user with Radware DDoS", "VISP user in the satellite wizard"). If the story only applies under a condition, put the condition in the acceptance criteria and keep the persona canonical.

#### 3b. Draft stories

Write stories covering every implemented capability in the capability matrix. Each capability category below typically produces **multiple stories** when its variants are properly decomposed per the granularity contract (Phase 3.0) — not one story per category. Use the matrix as the enumeration scaffold; use these categories as a *completeness checklist* to ensure no category is silently skipped:

1. **Listing/browsing** — viewing collections; one story per distinct list-shape (default view, alternate column sets, filter variants, export variants)
2. **Detail views** — viewing individual items; one story per discrete information panel or sub-tab that has its own loading/refresh/scope
3. **Creation** — creating new items; one story per distinct creation entry point or wizard variant
4. **Editing** — modifying existing items; one story per discrete edit operation (rename, change-component, change-price, change-number…) — *not* one umbrella "edit" story
5. **Deletion** — removing items; one story per delete pathway when there is more than one
6. **State transitions** — one story per transition (activate / suspend / decommission / migrate / archive / restore …)
7. **Relationships** — linking items to other items
8. **Import/export** — bulk operations, data movement
9. **Notifications** — alerts, emails, in-app messages
10. **Permissions** — access control, roles
11. **Integrations** — third-party connections
12. **Validation** — input rules, constraints
13. **Configuration** — settings, preferences

Do NOT create a separate user story for each item in a sidenav, user menu, or tab bar. Collapse the navigation surface into ONE story whose AC lists the items as a table:

> The user menu exposes the following items, each visible only when the named feature permission is enabled:
>
> | Label | Destination | Feature permission |
> |---|---|---|
> | … | … | … |

#### 3c. Create stories via MCP

For each story, call `createUserStory` with:

- `title` — what the user can do, phrased as an outcome not a locator. Every title must contain a user-facing verb (`see`, `open`, `create`, `edit`, `delete`, `submit`, `download`, `manage`, `raise`, `escalate`, `invite`, `assign`, `approve`, …). "Invite team members by email" ✅ / "TeamInviteComponent" ❌ / "Open /team/invite" ❌.
- `key` — area key prefix + 2-digit zero-padded number, numbered up from 01 (e.g. `TEAM-01`, `TEAM-02`, `AUTH-01`). When adding to an area that already has stories, continue from the highest existing key.
- `areaKey` — the area's `key` (e.g. `TEAM`, `AUTH`). **Always prefer `areaKey` over `areaId`.** Opaque UUIDs carried in memory across batches drift silently and file stories under the wrong area; the semantic `areaKey` cannot. The backend also cross-checks that the story `key` starts with `${areaKey}-` and rejects mismatches.
- `description` — `"As a <canonical persona>, I want to <action> so that <benefit>."` Persona MUST come from the list built in Phase 3a.
- `status` — `completed` (these describe existing, implemented capabilities)
- `priority` — `high`, `medium`, or `low` based on the feature's centrality to the area
- `acceptanceCriteria` — see rules below

##### Acceptance criteria rules

- **Count: as many as are needed to thoroughly verify the story.** There is no minimum and no maximum. A simple read-only page may need 3; a multi-step wizard may need 12. Do not pad to hit a target and do not stop short to keep the story small. The right number is a function of the story's scope.
- **User-observable, not implementation.** Each criterion must describe something the user can see, do, or experience. Component names, service/DAO class names, method calls, route constants used as routing config (`Route X maps to YComponent`), `canActivate`/`canDeactivate`/`ChangeGuard`, RxJS primitives (`Subject`, `Observable`, `ngOnInit`), `checkFeatureEnabled(...)`, and other TypeScript identifiers do NOT belong in acceptance criteria. Route URLs are allowed only as navigation outcomes (e.g. "clicking Services lands on `/portal/services`"), not as routing-module config.

  | ❌ Implementation-leaky | ✅ User-observable |
  |---|---|
  | `Route /portal/services maps to PortalServiceListComponent with canDeactivate: [ChangeGuard] and breadcrumb 'My Services'.` | `Clicking "Services" in the sidenav shows the list of services with the breadcrumb "My Services". If the user has unsaved edits, they are prompted to confirm before navigating away.` |
  | `onFilterToggle('Show Closed', value) sets this.showClosed to the boolean value.` | `Toggling "Show Closed" includes decommissioned and cancelled services in the list; toggling it off hides them.` |
  | `The 'Manage API Keys' item is only rendered when checkFeatureEnabled(['API', 'Change API Keys']) is true.` | `The "Manage API Keys" menu item is visible only to users whose role includes the "Change API Keys" permission.` |

- **Testable.** Specific enough that an e2e or unit test could assert it. "Works correctly" is not an acceptance criterion.
- **Concrete values over adjectives.** "Up to 50 items per page" beats "a reasonable number".
- **One concern per criterion.** Split compound criteria into separate lines.

##### "Open the list at /X" archetype

Route-opening stories are valid, but their acceptance criteria must focus on what the user sees and can do on that page, not on the route configuration:
- Which personas / feature permissions make the entry point visible.
- What the user sees on first load (columns, default sort/filter, empty-state text).
- What actions are available and where they lead, from the user's point of view.
- What happens to unsaved edits on navigation away (when relevant).

Route wiring, component class names, and host-class selectors belong in architecture elements (a later skill), not in user stories.

Batch creation: create up to 10 stories per batch to keep interactions manageable.

**Before each batch**, call `listAreas` for the product to get the fresh `{_id, key, name}` mapping. Do not carry area identifiers in your head across batches — look them up every time. The `areaKey` you pass to `createUserStory` must come from this fresh lookup.

**After each batch**, call `listUserStories` for the target area and confirm every newly-created story key appears in the result. If any are missing, stop immediately and investigate — the missing stories landed under the wrong area.

#### 3d. Post-generation lint (mandatory)

Before presenting the summary, self-review every story created in this run. If a story fails any of the checks, patch it (`patchUserStory`) before proceeding.

| Check | Trigger | Fix |
|---|---|---|
| AC implementation leak | Any acceptance criterion mentions a TypeScript identifier ending in `Component`/`Service`/`Module`, a method call like `name(...)`, an RxJS primitive (`Subject`, `Observable`), Angular lifecycle/guard identifiers (`ngOnInit`, `canActivate`, `canDeactivate`, `ChangeGuard`), `checkFeatureEnabled(`, or a route string used as routing-module config rather than a navigation destination. | Rewrite to user-observable wording using the examples above. |
| Title lacks user verb | Title has no verb from the allowed set (`see`, `view`, `open`, `create`, `edit`, `delete`, `submit`, `download`, `filter`, `sort`, `navigate`, `manage`, `invite`, `assign`, `approve`, `raise`, `escalate`, …). | Reword the title to state the outcome. |
| **Coarse title (granularity contract violation)** | Title contains `and`, `or`, `if`, `depending on`, `across modes`, multiple distinct verbs, or multiple state transitions (e.g. "Browse services in the list or detailed view", "Modify or decommission a service"). | Decompose into one story per discrete capability. The matrix from Phase 2c tells you what the children should be. |
| **Matrix row uncovered** | Any populated row in the Phase 2c matrix has zero stories pointing to it after Phase 3c completes. | Add the missing stories before moving on. The matrix is the required coverage gate, not a guideline. |
| Persona not canonical | Story description's persona is not a `title` from `brief.users` or the fallback `"any signed-in user"`. | Swap in the closest canonical persona; move qualifiers into acceptance criteria. |
| Write verbs missing but implied | Capability matrix shows write verbs (create/edit/delete/state-transition) for this area, but the stories contain none. | Add the missing stories before moving on. |
| Story filed under wrong area | After each batch, `listUserStories` for the target area does not include every newly-created key. | Stop immediately. The story was filed elsewhere — find it, `patchUserStory` its `/areaId` to the correct area, and audit every other story created in the same run. |
| Thin area | Area ends with **fewer than 3 stories** after lint. | STOP and ask the user: (a) merge with neighbouring area, (b) delete the area, (c) continue researching for missing capabilities. Do not silently leave a stub. |

#### 3e. Present summary

After creating stories and running the lint, present:

| Area | Stories Created | Categories Covered | Personas Used |
|------|----------------|--------------------|---------------|
| {area} | {count} | {list} | {list} |

Show the story titles as a checklist. Ask the user:

> **Review the stories above.** Would you like to:
> - **Adjust any stories** — I can update titles, descriptions, or acceptance criteria
> - **Continue** — the stories look good, move on

---

## Rules

- **All stories are `completed`** — this skill describes existing capabilities, not planned work. Every story should reflect something the codebase already implements.
- **Every field on every story must be populated** — no empty descriptions, no missing acceptance criteria.
- **Granularity contract is non-negotiable** — one story = one acceptance-test cluster. Titles cannot contain `and`/`or`/`if`/`depending on`/`across modes`. Force a separate story for each state transition, configuration variant, role/permission variant, distinct dialog, and discrete sub-action. See Phase 3.0.
- **Matrix coverage is the quality gate** — every populated row in the Phase 2c matrix must be backed by ≥1 story before the area is "done". The matrix is required, not optional.
- **Add-more mode runs the decomposition pass** — Phase 2d evaluates each existing story against the contract; coarse stories get fine-grained replacements; unlinked parents are retired automatically, linked parents only after explicit confirmation. The `--no-decompose` flag opts out.
- **Favour completeness over arbitrary targets** — if the matrix has 30 rows, create at least 30 stories. Don't stop at 5 or 10 just because the area was previously light.
- **No minimum or maximum acceptance-criteria count** — write as many as the story needs for thorough verification, and no more. The right number is a function of story scope.
- **Acceptance criteria are user-observable** — never cite component names, service names, method calls, route constants used as config, or other implementation identifiers. See Phase 3c.
- **Personas are drawn from `brief.users`** — do not mint ad-hoc persona phrasings per story. See Phase 3a.
- **Titles state outcomes, not locators** — route URLs can appear in acceptance criteria as navigation destinations when helpful.
- **One story per navigation surface, not per item** — sidenavs, user menus, tab bars collapse into a single story with a table of items in acceptance criteria. This is the *only* anti-fragmentation rule; the rest of the contract pushes toward finer stories.
- **Halt on thin areas** — if an area ends with fewer than 3 stories after lint, ask the user before moving on.
- **Keys must be unique within the product** — use the area key as prefix followed by a 2-digit zero-padded number (e.g. `AUTH-01`, `AUTH-02`). When adding to an area that already has stories, continue numbering from the highest existing key.
- **Always pass `areaKey`, not `areaId`** — semantic keys can't drift the way opaque UUIDs carried across batches can. Re-fetch via `listAreas` before each batch; verify via `listUserStories` after each batch.
- **Acceptance criteria must be testable** — specific enough that a developer could write an automated test from them. "Works correctly" is not a criterion.
- **Don't invent capabilities** — only create stories for features you can verify exist in the codebase. If unsure whether something is implemented, check the code.
- **Batch MCP calls** where possible to minimise round trips.
