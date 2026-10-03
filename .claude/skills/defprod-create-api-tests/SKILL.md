---
name: defprod-create-api-tests
description: Generates HTTP integration tests for user stories whose surface is `api` (explicit or inferred from the `API-…` story-key prefix). One spec file per in-scope story, one `test()` per acceptance criterion, asserting against the project's RPC envelope. Use for REST / HTTP test generation against any area that contains API stories.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Write
  - Edit
  - AskUserQuestion
  - mcp__defprod__listProducts
  - mcp__defprod__listAreas
  - mcp__defprod__listUserStories
  - mcp__defprod__getUserStory
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

# Create API Tests

Generates HTTP integration tests for the stories in a product area whose surface is `api` — either set explicitly via `UserStory.surface = 'api'`, or inferred from the `API-…` story-key prefix when `surface` is unset. Each in-scope story gets its own test file with one `test()` call per acceptance criterion. Designed for backend / REST surfaces where the natural test runner is Vitest (preferred) or Jest, running against a live server's RPC endpoint. Stories on other surfaces are skipped with a one-line log entry — invoke the matching sibling skill, or `/defprod-create-area-tests` (the surface-aware dispatcher) for an end-to-end area walk.

This is the API-surface sibling of `defprod-create-ui-tests` (Playwright), `defprod-create-mcp-tests` (MCP SDK), and `defprod-create-cli-tests` (subprocess). Same per-AC contract; surface filter selects which stories the skill operates on.

## When to use

- The product has an area whose stories include REST / RPC surfaces (typically an `API` area, but the filter also picks up API stories from a mixed-surface area like `STATUS`).
- Invoked via `/defprod-create-api-tests <product-name> <area-key>`.
- Do not use for UI / MCP / CLI stories — invoke the matching sibling skill or the `/defprod-create-area-tests` dispatcher.

## Inputs

- **Product name** — the DefProd product.
- **Area key** — typically `API`. A "straggler" backend story from another area (e.g. a sync RPC under a STATUS area) can also be generated here.

## Config

This skill consults `.defprod/defprod.json` for optional hints. If absent, the skill discovers everything automatically.

| Key | Type | Purpose |
|-----|------|---------|
| `products[].name` | `string` | Matches product name to locate config entry |
| `products[].backendApp` | `string` | Path to the backend app — used to find the test root and Vitest/Jest config |
| `products[].apiTestDir` | `string` | Path to the API integration test directory (default `<backendApp>/tests/areas`) |
| `products[].apiBaseUrl` | `string` | Base URL the harness POSTs to (default `https://localhost:<dev-port>/api/v1/rpc`) |

---

## Workflow

### Phase 1 — Resolve

#### 1a. Find the product, area, and stories

1. Call `listProducts` from the DefProd MCP server. Find the product by name.
2. Call `listAreas` for the product. Find the area by key.
3. Call `listUserStories` for the area. Stories and acceptance criteria are the source of truth.

If the area has no stories, halt and suggest `/defprod-create-area-stories`.

#### 1b. Filter stories by surface (target: `api`)

For every story returned by `listUserStories`, determine its **effective surface** before deciding whether it is in scope:

1. If `story.surface` is set explicitly, use that value.
2. Otherwise, fall back to the story-key prefix mapping from `libs/defprod-common/src/lib/modules/user-story/infer-surface-from-story-key.util.ts`:
   - `API-…` infers `api`, `MCP-…` infers `mcp`, `CLI-…` infers `cli`. Anything else infers nothing → unspecified.
3. **In-scope set**: stories whose effective surface is `api`.
4. **Out-of-scope set**: every other story. Log each with one line, then move on:
   - `Skipped <STORY-KEY> (<title>): surface=<value> (explicit|inferred|unspecified); this skill targets api.`

Record the `(explicit|inferred)` tag per in-scope story too — Phase 5c flags inference-routed stories in the coverage summary.

If the in-scope set is empty, halt with:
> No API stories found in this area (no story has surface=api, explicit or inferred). Skipping.

#### 1c. Check for existing tests

Look under the api test directory (`<backendApp>/tests/areas/<AREA>/` by default). If specs exist for some stories, ask the user:

> **Found existing tests for {count} of {total} stories.** Would you like to:
> - **Fill gaps** — only create tests for stories without coverage
> - **Replace all** — regenerate tests for all stories
> - **Cancel** — keep existing tests as-is

### Phase 2 — Verify harness scaffolding

#### 2a. Test runner

Confirm the backend app declares Vitest (preferred) or Jest in its `package.json` devDependencies and has a config file (e.g. `vitest.config.api.ts` or `jest.config.ts`). If neither is present, halt and tell the user to choose a runner before generating specs.

#### 2b. Helpers

Confirm an HTTP client helper exists or build the minimum surface the specs will need. The skill's generated specs assume:

- An `ApiClient` (or similar) with `rpc(name, input)`, `rpcWithSession(...)`, `rpcWithApiKey(...)`, and `rest(path, body, opts)` methods.
- A `withFreshUserAndKey(fn)` fixture that signs up a throwaway user, verifies their email, creates an API key, yields a context, and cleans up via direct DB delete in `finally`.
- A `closeDb()` that releases any cached MongoDB connection (so the test process exits cleanly).

If these don't exist, halt and surface what needs to be built first — don't half-create.

#### 2c. Backend running

Probe the project's RPC endpoint with a GET (or harmless OPTIONS) to confirm the dev backend is up. Halt with a clear instruction if not.

### Phase 3 — Analyse testability

Classify each AC:

- **Testable via RPC** — verifiable through one or more RPC calls against the endpoint.
- **Indirectly testable** — confirmed via observable side-effects on subsequent RPC calls (e.g. "key is stored securely" → listing keys returns the new one).
- **Untestable via the API harness** — documentation-completeness, human-language quality, OpenAPI artifact presence (those are file-on-disk checks, not RPC checks). Mark with reason; no `test()` call.

Surface untestable items to the user before generating, so coverage gaps are explicit.

### Phase 4 — Create tests

#### 4a. Test structure (MANDATORY)

Every spec file MUST follow the AC-per-test contract: one `test()` per testable acceptance criterion, grouped under a single `describe` block named after the story.

```ts
import { describe, test, expect, afterAll } from 'vitest'; // or '@jest/globals'
import { ApiClient, withFreshUserAndKey, closeDb } from '../../../helpers/api-shared';

/**
 * Story: <STORY-KEY> — <Story title>
 * Acceptance criteria verified:
 * - AC1: <criterion>
 * - AC2: <criterion>
 *
 * Untestable acceptance criteria (documented, no test() generated):
 * - <criterion> — <reason>
 */
describe('<STORY-KEY> — <Story title>', () => {

  // Release the shared DB connection so the runner exits cleanly.
  // Per-test cleanup happens inside withFreshUserAndKey.
  afterAll(async () => { await closeDb(); });

  test('AC1: <focused assertion>', async () => {
    await withFreshUserAndKey(async (ctx) => {
      const res = await ApiClient.rpcWithApiKey('<rpcName>', { /* ... */ }, ctx.apiKey);
      expect(res.status).toBe(200);
      // ... per-AC assertions only ...
    });
  });

  // ... one test() per testable AC ...
});
```

**Strict requirements:**

1. **File header comment** lists the story key/title, every testable AC, and every untestable AC with its reason.
2. **Exactly one `describe(...)` block** per spec, named `'<STORY-KEY> — <story title>'`.
3. **One `test('ACn: ...', ...)` per testable AC.** Name MUST begin with `ACn:` where `n` matches the AC's position in the story's acceptance criteria array.
4. **Each `test()` is self-contained.** No shared mutable state; each test provisions and tears down its own fixtures.
5. **Untestable ACs are documented in the header**, not skipped — no `test.skip()` for them.

**FORBIDDEN — bundled-AC structure (do not generate):**

```ts
// ✗ One test() walking through multiple ACs sequentially:
test('user can create and list and delete a key', async () => { /* hides per-AC failures */ });
```

#### 4b. Test writing guidelines

- **Verify case names before generating.** A story's plain-English title may not match the actual RPC name (e.g. "generate a new API key" vs `createApiKey`). Grep the backend module to confirm.
- **Type tests strongly** if the project ships a `CaseName` enum and a typed `ApiClient.rpc<TCase>(...)` form — case-name typos surface at compile time, not at run time.
- **Assert on the envelope shape**, not on free-text fields. `meta.dataCategory`, `meta.status`, response code. Avoid asserting on `error.detail` (human-readable, changes freely).
- **Cleanup via `try/finally`** whenever a test creates anything beyond the fresh-user fixture.
- **Avoid sleeps.** If you must wait for an async side-effect, poll the relevant RPC with a tight loop + 5s ceiling.
- **One AC per `test()`** — never combine.

#### 4c. Write files

Create files under `<apiTestDir>/<AREA>/<STORY-KEY>/<short-name>.test.ts`. Batch up to 5-10 files per write pass.

#### 4d. Retrofit existing bundled-AC tests in the area

If any existing specs in the area bundle multiple ACs into a single test, split them into one test per AC. Preserve assertion bodies verbatim — only the structure changes. Retrofit happens in the same run.

### Phase 5 — Run & Report

#### 5a. Run the tests

Use the project's runner. For Vitest:

```bash
npx vitest run -c <vitestConfig> <apiTestDir>/<AREA>
```

For Jest:

```bash
npx jest --config=<jestConfig> <apiTestDir>/<AREA>
```

Expect some failures on first run.

#### 5b. Fix failing tests

Classify each failure:

- **Test fault** — spec is wrong (selector, fixture, args). Fix in the spec.
- **Product fault** — the implementation doesn't honor the documented behavior. Surface to the user; either fix the implementation (paired with their authorization) or tag the story with `production-gap` and `.skip` the affected AC, citing the tag.

Iterate until green or every remaining failure is a tracked product issue.

#### 5c. Present coverage summary

| Story | Key | Surface | Total ACs | Testable | `test()` calls | Passing | Status |
|---|---|---|---:|---:|---:|---:|---|
| ... | ... | `api` *(inferred)* / `api` | ... | ... | ... | ... | Pass/Fail |

Mark the **Surface** column with *(inferred)* for any story routed in via inference rather than an explicit `surface` value — those are candidates for a `patchUserStory /surface` backfill so future runs no longer rely on the inference helper.

**Out-of-scope stories** (other surfaces, skipped):
- `<STORY-KEY>` — `<surface>` (explicit|inferred|unspecified). Use `/defprod-create-<surface>-tests`, or `/defprod-create-area-tests` for an area-wide walk.

Flag any row where `test()` calls != testable ACs. List untestable items with their reasons. List any production-gap tags applied.

Suggest:

> **Next steps:**
> - Backfill `surface` on any inference-routed stories you want to lock to API.
> - Run `/defprod-run-area-tests <area>` to produce a classified failure report.
> - Once all green, commit and consider whether `/defprod-sync-story-test-status` should publish coverage to the dashboard.

---

## Rules

- **Every story gets a spec file** — even if some ACs are untestable, the testable ones still ship.
- **Every testable AC must be covered by its own `test()`** — never bundle.
- **Test structure is non-negotiable** — one describe per story, one `test('ACn: ...')` per AC.
- **Untestable items are documented in the header**, not skipped silently.
- **Don't modify production code** in this skill — surface product faults to the user instead.
- **Match the project's existing helper / fixture / import conventions.** Structure is always one test per AC.
- **No new dependencies.** Vitest/Jest, native `fetch`, the project's existing helpers.
- **Test names trace to criteria** — every `test()` begins with `ACn:`.
- **Retrofit bundled-AC specs in the same run** — area ends fully consistent.
- **Batch file creation** — 5-10 files per pass.
