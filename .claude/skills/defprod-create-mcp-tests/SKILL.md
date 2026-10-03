---
name: defprod-create-mcp-tests
description: Generates integration tests for user stories whose surface is `mcp` (explicit or inferred from the `MCP-…` story-key prefix), using the official @modelcontextprotocol/sdk client. One spec file per in-scope story, one `test()` per acceptance criterion, exercised through the running MCP server. Use for MCP test generation against any area that contains MCP stories.
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

# Create MCP Tests

Generates MCP integration tests for the stories in a product area whose surface is `mcp` — either set explicitly via `UserStory.surface = 'mcp'`, or inferred from the `MCP-…` story-key prefix when `surface` is unset. Each in-scope story gets its own test file with one `test()` per acceptance criterion. The harness uses the official `@modelcontextprotocol/sdk` client connected over the streamable-HTTP `/mcp` endpoint (and the SSE `/mcp/sse` + `/mcp/messages` endpoints when the story explicitly requires SSE). Stories on other surfaces are skipped with a one-line log entry — invoke the matching sibling skill, or `/defprod-create-area-tests` (the surface-aware dispatcher) for an end-to-end area walk.

This is the MCP-surface sibling of `defprod-create-ui-tests` (Playwright), `defprod-create-api-tests` (REST), and `defprod-create-cli-tests` (subprocess). Same per-AC contract, different harness.

## When to use

- The product has an area whose stories include MCP surfaces (typically an `MCP` area, but the filter also picks up MCP stories from a mixed-surface area).
- Invoked via `/defprod-create-mcp-tests <product-name> <area-key>`.
- Do not use for UI / REST / CLI stories — invoke the matching sibling skill or the `/defprod-create-area-tests` dispatcher.

## Inputs

- **Product name** — the DefProd product.
- **Area key** — typically `MCP`.

## Config

This skill consults `.defprod/defprod.json` for optional hints. If absent, the skill discovers everything automatically.

| Key | Type | Purpose |
|-----|------|---------|
| `products[].name` | `string` | Matches product name to locate config entry |
| `products[].mcpApp` | `string` | Path to the MCP server app — used to find the test root and Vitest config |
| `products[].mcpTestDir` | `string` | Path to the MCP integration test directory (default `<mcpApp>/tests/areas`) |
| `products[].mcpBaseUrl` | `string` | MCP server base URL the harness connects to (default `http://localhost:<dev-port>`) |

---

## Workflow

### Phase 1 — Resolve

#### 1a. Find the product, area, and stories

1. Call `listProducts` and find the product by name.
2. Call `listAreas` and find the area by key (typically `MCP`).
3. Call `listUserStories` for the area.

If empty, halt and suggest `/defprod-create-area-stories`.

#### 1b. Filter stories by surface (target: `mcp`)

For every story returned by `listUserStories`, determine its **effective surface** before deciding whether it is in scope:

1. If `story.surface` is set explicitly, use that value.
2. Otherwise, fall back to the story-key prefix mapping from `libs/defprod-common/src/lib/modules/user-story/infer-surface-from-story-key.util.ts`:
   - `API-…` infers `api`, `MCP-…` infers `mcp`, `CLI-…` infers `cli`. Anything else infers nothing → unspecified.
3. **In-scope set**: stories whose effective surface is `mcp`.
4. **Out-of-scope set**: every other story. Log each with one line, then move on:
   - `Skipped <STORY-KEY> (<title>): surface=<value> (explicit|inferred|unspecified); this skill targets mcp.`

Record the `(explicit|inferred)` tag per in-scope story too — Phase 5c flags inference-routed stories in the coverage summary.

If the in-scope set is empty, halt with:
> No MCP stories found in this area (no story has surface=mcp, explicit or inferred). Skipping.

#### 1c. Check for existing tests

Look under the MCP test directory. If specs exist for some stories, ask:

> **Found existing tests for {count} of {total} stories.** Fill gaps / Replace all / Cancel.

### Phase 2 — Verify harness scaffolding

#### 2a. Test runner

Vitest is the preferred runner (Jest works too; the per-AC contract is identical). Confirm the MCP app has a Vitest config (e.g. `vitest.config.mcp.ts`) that sets:

- `include: ['tests/areas/**/*.test.ts']`
- `environment: 'node'`
- `testTimeout` ≥ 15000 (suites talk to two live servers — the MCP server and the backend that provisions fixture users)
- `fileParallelism: false` so signup-then-connect flows don't race
- Module alias for the project's shared common library if one exists

#### 2b. MCP client helper

Confirm a helper exists (or build it) that exposes:

- `connectStreamable(apiKey)` — returns `{ client, close }` connected via `StreamableHTTPClientTransport` to `/mcp`.
- `connectSse(apiKey)` — returns `{ client, close }` via `SSEClientTransport` to `/mcp/sse` + `/mcp/messages`.
- `withMcpClient(transport, apiKey, fn)` — connect, run, always close.
- `rawMcpPost(body, headers)` — low-level fetch for assertions that bypass the SDK (auth-format checks, raw JSON-RPC envelope).
- `getMcpBaseUrl()` — reads `MCP_BASE_URL` env, defaults to the project's dev port.

#### 2c. Fixture user helper

The MCP harness needs an API key to forward to tool calls. Confirm a `withFreshUserAndKey(fn)` fixture exists (same shape as the API harness expects) that provisions a throwaway user + API key via the backend's signup → verify → createApiKey flow, yields a context, and cleans up via direct DB delete.

#### 2d. Both servers running

Probe the project's backend (for fixture provisioning) and MCP server (system under test). Halt with a clear instruction if either is down.

### Phase 3 — Analyse testability

Classify each AC:

- **Testable via the MCP SDK** — `client.listTools()`, `client.callTool()`, `client.getServerCapabilities()`, or a connect/disconnect round-trip.
- **Testable via raw HTTP** — auth-format checks, missing-header rejection, raw JSON-RPC envelope inspection. Use `rawMcpPost()`.
- **Indirectly testable** — implementation detail confirmed through observable behavior (e.g. "API key is stored in request context" → verified by a tool call succeeding under that key).
- **Untestable via the MCP harness** — documentation completeness, file-on-disk content checks, behaviors that require backend code inspection. Mark with reason; no `test()` call.

### Phase 4 — Create tests

#### 4a. Test structure (MANDATORY)

```ts
import { afterAll, describe, expect, test } from 'vitest';
import { withMcpClient } from '../../../helpers/mcp-client';
import { closeDb, withFreshUserAndKey } from '../../../helpers/api-shared';

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

  afterAll(async () => { await closeDb(); });

  test('AC1: <focused assertion>', async () => {
    await withFreshUserAndKey(async (ctx) => {
      await withMcpClient('streamableHttp', ctx.apiKey, async (client) => {
        const res = await client.callTool({ name: '<toolName>', arguments: { /* ... */ } });
        expect(res.isError).toBeFalsy();
        // ... per-AC assertions only ...
      });
    });
  });

  // ... one test() per testable AC ...
});
```

**Strict requirements:**

1. **File header comment** lists story key/title, testable ACs, and untestable ACs (with reason).
2. **Exactly one `describe(...)`** per spec, named `'<STORY-KEY> — <story title>'`.
3. **One `test('ACn: ...', ...)` per testable AC.** Name MUST begin with `ACn:` matching the AC's position in the array.
4. **Each `test()` is self-contained** — own MCP session, own fixture user, cleanup via `withMcpClient`'s finally / `withFreshUserAndKey`'s finally.
5. **`afterAll` only for `closeDb()`** — release the cached MongoDB connection.

**FORBIDDEN — bundled-AC structure (do not generate):**

```ts
// ✗ One test() walking through multiple ACs:
test('agent can create + read + update + delete a product', async () => { /* masks per-AC failures */ });
```

#### 4b. Test writing guidelines

- **Prefer streamable HTTP for tool ACs.** Use SSE only when the AC explicitly references SSE (e.g. an "MCP server supports SSE transport" story).
- **MCP errors come back in the result, not as throws.** Tool-execution errors return `{ isError: true, content: [{ type: 'text', text: '...' }] }`. Assert on `result.isError` and the text shape, never on a thrown SDK error (unless the error is at the protocol layer — e.g. unknown tool name).
- **Match content semantically, not verbatim.** Look for sub-strings that prove the data made the round trip (e.g. `/PRODUCT-/` for a product create). Don't deep-match full text.
- **Provision real entities via the MCP tools being tested**, not via direct DB writes. If a test needs a product, call `createProduct` first via the SDK; clean up via `deleteProduct` in `finally`.
- **`withFreshUserAndKey` always.** Never assume a pre-existing user/key.
- **Avoid sleeps.** Connect / listTools / callTool are synchronous round-trips.
- **One AC per `test()`** — never combine.

#### 4c. Write files

Create files under `<mcpTestDir>/<AREA>/<STORY-KEY>/<short-name>.test.ts`. Batch 5-10 per write pass.

#### 4d. Retrofit existing bundled-AC tests in the area

If any existing specs bundle multiple ACs, split into one test per AC. Same run.

### Phase 5 — Run & Report

#### 5a. Run

```bash
npx vitest run -c <vitestConfig> <mcpTestDir>/<AREA>
```

#### 5b. Fix

Classify each failure as **test fault** (assertion / fixture / SDK usage off — fix the spec) or **product fault** (MCP server returns the wrong shape, error path doesn't include `isError: true`, transport bug — surface to the user; either fix the implementation paired with their authorization, or `.skip` the affected AC and tag the story with `production-gap`).

#### 5c. Present coverage summary

| Story | Key | Surface | Total ACs | Testable | `test()` calls | Passing | Status |
|---|---|---|---:|---:|---:|---:|---|
| ... | ... | `mcp` *(inferred)* / `mcp` | ... | ... | ... | ... | Pass/Fail |

Mark the **Surface** column with *(inferred)* for any story routed in via inference rather than an explicit `surface` value — those are candidates for a `patchUserStory /surface` backfill.

**Out-of-scope stories** (other surfaces, skipped):
- `<STORY-KEY>` — `<surface>` (explicit|inferred|unspecified). Use `/defprod-create-<surface>-tests`, or `/defprod-create-area-tests` for an area-wide walk.

Flag mismatches between testable AC count and `test()` count. List untestable items + reasons. List any production-gap tags applied.

Suggest:

> **Next steps:**
> - Backfill `surface` on any inference-routed stories you want to lock to MCP.
> - Run `/defprod-run-area-tests <area>` to produce a classified failure report.

---

## Rules

- **Every story gets a spec file.**
- **Every testable AC gets its own `test()`** — never bundle.
- **Test structure is non-negotiable** — one describe per story, one `test('ACn: ...')` per AC.
- **MCP errors live in the result envelope, not in thrown exceptions** — assert on `isError` and content shape.
- **Streamable HTTP by default; SSE only when the AC mandates it.**
- **Match existing helper / fixture / import conventions** in the project.
- **Don't modify production code** — surface product faults to the user instead.
- **No new dependencies.** Vitest/Jest, `@modelcontextprotocol/sdk` (already a transitive dep of the MCP server), native `fetch`, the project's existing helpers.
- **Untestable items are documented in the header** with reason.
- **Test names trace to criteria** — every `test()` begins with `ACn:`.
- **Retrofit bundled-AC specs in the same run.**
- **Batch file creation** — 5-10 files per pass.
