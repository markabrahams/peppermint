---
name: defprod-create-ui-tests
description: Generates Playwright e2e tests from user stories whose surface is `ui` (explicit or inferred). One spec file per in-scope story, with test cases mapped to acceptance criteria. Use for browser-driven test generation on UI stories within a product area.
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

# Create UI Tests

Generates browser-driven end-to-end tests (Playwright preferred, Cypress also supported) for a product area's UI stories — those whose `surface` field is `ui`, either explicitly set or inferred from the story-key prefix when no other surface is known. Each in-scope story gets its own test file with test cases mapped to acceptance criteria. Stories on other surfaces (`api`, `mcp`, `cli`, `script`, `library`, `other`) are skipped — invoke the matching sibling skill, or `/defprod-create-area-tests` (the surface-aware dispatcher) for an end-to-end area walk.

This is the UI-surface sibling of `defprod-create-api-tests`, `defprod-create-mcp-tests`, and `defprod-create-cli-tests`. Same per-area input contract; surface filter selects which stories the skill operates on.

## When to use

- When a product area has UI user stories with acceptance criteria and you want automated browser tests.
- When invoked via `/defprod-create-ui-tests <product-name> <area-key>`.
- Do **not** use for backend / API / MCP / CLI stories — invoke the matching sibling skill instead.

## Inputs

- **Product name** — the DefProd product.
- **Area key** — the area to generate tests for (e.g. `AUTH`, `BILLING`).

## Config

This skill consults `.defprod/defprod.json` for optional hints. If the file doesn't exist, the skill discovers everything automatically.

| Key | Type | Purpose |
|-----|------|---------|
| `products[].name` | `string` | Matches product name to locate config entry |
| `products[].frontendApp` | `string` | Path to the frontend app — used to find existing tests |
| `products[].e2eDir` | `string` | Path to the e2e test directory — determines where tests are written |

---

## Workflow

### Phase 1 — Resolve

#### 1a. Find the product, area, and stories

1. Call `listProducts` from the DefProd MCP server. Find the product by name.
2. Call `listAreas` for the product. Find the area by key.
3. Call `listUserStories` for the area. These stories and their acceptance criteria are the source of truth for test generation.

If the area has no user stories, tell the user and suggest running `/defprod-create-area-stories` first.

#### 1b. Check for existing tests

Look for existing e2e tests:
- Check `e2eDir` from `.defprod/defprod.json` if available
- Search for common test directories: `e2e/`, `tests/e2e/`, `test/`, `cypress/`, `__tests__/`
- Look for test files matching the area name

If tests already exist for some stories, ask the user:

> **Found existing tests for {count} of {total} stories.** Would you like to:
> - **Fill gaps** — only create tests for stories without coverage
> - **Replace all** — regenerate tests for all stories
> - **Cancel** — keep existing tests as-is

#### 1c. Filter stories by surface (target: `ui`)

For every story returned by `listUserStories`, determine its **effective surface** before deciding whether it is in scope:

1. If `story.surface` is set explicitly, use that value.
2. Otherwise, fall back to the story-key prefix mapping from `libs/defprod-common/src/lib/modules/user-story/infer-surface-from-story-key.util.ts`:
   - `API-…` infers `api`, `MCP-…` infers `mcp`, `CLI-…` infers `cli`.
   - Any other prefix infers nothing → treat as **`ui`** for this skill (the historical default; UI stories rarely carry an explicit surface yet).
3. **In-scope set**: stories whose effective surface is `ui`.
4. **Out-of-scope set**: every other story. Log each with one line, then move on — do not generate tests for them:
   - `Skipped <STORY-KEY> (<title>): surface=<value> (explicit|inferred); this skill targets ui.`

Record the `(explicit|inferred)` tag per in-scope story too — Phase 5c flags inference-routed stories in the coverage summary so the user can decide whether to backfill `surface`.

If the in-scope set is empty, halt with:
> No UI stories found in this area (no story has surface=ui, explicit or inferred). Skipping.

---

### Phase 2 — Discover test setup

Before writing tests, understand the project's test infrastructure.

#### 2a. Identify test framework

Look for:
- **Playwright** — `playwright.config.ts`, `@playwright/test` in dependencies
- **Cypress** — `cypress.config.ts`, `cypress/` directory
- **Jest** — `jest.config.ts`, `@jest/globals` in dependencies
- **Vitest** — `vitest.config.ts`, `vitest` in dependencies
- Other frameworks — check `package.json` devDependencies

#### 2b. Read existing test patterns

If the project has existing e2e tests, read 2-3 of them to understand:
- Import patterns and test utilities
- Authentication/login patterns
- Navigation and page interaction patterns
- Assertion patterns
- Setup and teardown conventions
- Selector strategies (data-testid, roles, text, CSS)
- File naming conventions

Match these patterns exactly in generated tests.

#### 2c. Determine directory structure

Decide where tests go based on:
1. `e2eDir` from `.defprod/defprod.json` — if set, use it
2. Existing test directory structure — follow the same pattern
3. Default: `e2e/areas/<AREA-KEY>/<STORY-KEY>/` (one directory per story)

---

### Phase 3 — Analyse testability

Not every acceptance criterion can be tested via e2e tests. Before generating, classify each criterion:

- **Testable** — can be verified through UI interaction or API calls
- **Untestable** — requires external systems, manual verification, or is non-functional

For untestable criteria, note the reason:
- Backend-only logic with no observable UI effect
- Requires external integration (payment provider, email service)
- Non-functional requirement (performance, security)
- Hardware-dependent (mobile-specific, printer)

Present any untestable items to the user so they're aware of coverage gaps.

---

### Phase 4 — Create tests

#### 4a. Generate test files

For each user story, create a test file containing:

1. **File header comment** — maps the test to the DefProd story:
   ```
   /**
    * Story: AREA-KEY — Story title
    * Acceptance criteria verified by this test:
    * - AC1: [criterion text]
    * - AC2: [criterion text]
    */
   ```

2. **Describe block** — named after the story title

3. **Test cases** — one or more tests covering the testable acceptance criteria. Each test should:
   - Set up the required state (login, navigate, create test data)
   - Perform the user action described in the criterion
   - Assert the expected outcome
   - Clean up any created test data

#### 4b. Test writing guidelines

- **Match existing test style** exactly — imports, selectors, assertions, naming
- **Use resilient selectors** — prefer data-testid, ARIA roles, and accessible names over brittle CSS selectors
- **Handle timing** — use framework-provided waits (e.g. Playwright's auto-waiting) rather than arbitrary sleeps
- **Isolate tests** — each test should be independent. Don't rely on ordering or shared state between tests
- **Use descriptive names** — test names should describe what's being verified, not how
- **Keep tests focused** — one logical assertion per test. Don't combine unrelated checks

#### 4c. Write files

Create the test files in the determined directory structure. Batch up to 5-10 files per creation pass.

---

### Phase 5 — Run & Report

#### 5a. Run the tests

Run the generated tests using the project's test runner:
- Playwright: `npx playwright test <path>`
- Cypress: `npx cypress run --spec <path>`
- Or the project's configured test command

Expect some failures on first run — this is normal.

#### 5b. Fix failing tests

For tests that fail:
1. Read the error output
2. Determine if the failure is a test bug (wrong selector, bad timing, incorrect assertion) or a real product issue
3. Fix test bugs immediately
4. Flag real product issues to the user — don't modify production code

Iterate until all tests pass or all remaining failures are confirmed product issues.

#### 5c. Present coverage summary

| Story | Key | Surface | Tests | Testable ACs | Covered ACs | Status |
|-------|-----|---------|-------|--------------|-------------|--------|
| ... | ... | `ui` *(inferred)* / `ui` | ... | ... | ... | Pass/Fail |

Mark the **Surface** column with *(inferred)* for any story routed in via inference rather than an explicit `surface` value — that flags candidates for `patchUserStory /surface` backfill.

**Out-of-scope stories** (other surfaces, skipped):
- `<STORY-KEY>` — `<surface>` (explicit|inferred). Use `/defprod-create-<surface>-tests` or `/defprod-create-area-tests`.

**Untestable items** (in-scope stories, ACs that can't be automated):
- [criterion] — [reason]

Suggest next steps:

> **Next steps:**
> - Backfill `surface` on any inference-routed stories you want to lock to UI.
> - Run `/defprod-run-area-tests <area>` to re-run these tests and get a failure analysis report.
> - For stories on other surfaces, invoke the matching sibling skill, or `/defprod-create-area-tests <product> <area>` to walk the whole area surface-by-surface.

---

## Rules

- **Every story gets a test file** — even if some acceptance criteria are untestable, create a test file for the testable ones.
- **Every testable acceptance criterion must be covered** — don't skip criteria unless they're genuinely untestable.
- **Match existing test style** — if the project has established test patterns, follow them exactly. Don't introduce new patterns.
- **Don't modify production code** — this skill only creates tests. If a test reveals a product bug, report it to the user.
- **Untestable items must be documented** — explain specifically why each untestable criterion can't be automated, not just "it's complex".
- **Test names must trace to criteria** — someone reading the test file should be able to map each test back to a specific acceptance criterion.
- **Batch file creation** — create up to 5-10 test files per pass to keep interactions manageable.
