---
name: defprod-create-cli-tests
description: Generates subprocess integration tests for user stories whose surface is `cli` (explicit or inferred from the `CLI-…` story-key prefix). One spec file per in-scope story, one `test()` per acceptance criterion, exercised by spawning the built CLI binary one-shot per assertion. Use for CLI test generation against any area that contains CLI stories.
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

# Create CLI Tests

Generates CLI integration tests for the stories in a product area whose surface is `cli` — either set explicitly via `UserStory.surface = 'cli'`, or inferred from the `CLI-…` story-key prefix when `surface` is unset. Each in-scope story gets its own test file with one `test()` per acceptance criterion. The harness spawns the built CLI binary as a one-shot Node subprocess per assertion, captures `stdout` / `stderr` / `exitCode`, and asserts on the observable surface. Stories on other surfaces are skipped with a one-line log entry — invoke the matching sibling skill, or `/defprod-create-area-tests` (the surface-aware dispatcher) for an end-to-end area walk.

This is the CLI-surface sibling of `defprod-create-ui-tests` (Playwright), `defprod-create-api-tests` (REST), and `defprod-create-mcp-tests` (MCP). Same per-AC contract; subprocess harness instead.

Shell-script "straggler" stories carrying `surface = 'script'` are out of scope here — they will be skipped. (A future `defprod-create-script-tests` sibling may pick them up; until then, set their surface to `cli` only if they invoke the built CLI binary, otherwise generate manually.)

## When to use

- The product has an area whose stories include the built CLI binary as the surface (typically a `CLI` area, but the filter also picks up CLI stories from a mixed-surface area).
- Invoked via `/defprod-create-cli-tests <product-name> <area-key>`.
- Do not use for UI / REST / MCP stories — invoke the matching sibling skill or the `/defprod-create-area-tests` dispatcher.

## Inputs

- **Product name** — the DefProd product.
- **Area key** — typically `CLI`.

## Config

This skill consults `.defprod/defprod.json` for optional hints.

| Key | Type | Purpose |
|-----|------|---------|
| `products[].name` | `string` | Matches product name to locate config entry |
| `products[].cliApp` | `string` | Path to the CLI app — used to find the test root and Jest config |
| `products[].cliTestDir` | `string` | Path to the CLI integration test directory (default `<cliApp>/tests/areas`) |
| `products[].cliBinaryPath` | `string` | Built CLI binary path (default `dist/<cliApp>/main.js`) |

---

## Workflow

### Phase 1 — Resolve

#### 1a. Find the product, area, and stories

1. Call `listProducts` and find the product by name.
2. Call `listAreas` and find the area by key (typically `CLI`).
3. Call `listUserStories` for the area.

#### 1b. Filter stories by surface (target: `cli`)

For every story returned by `listUserStories`, determine its **effective surface** before deciding whether it is in scope:

1. If `story.surface` is set explicitly, use that value.
2. Otherwise, fall back to the story-key prefix mapping from `libs/defprod-common/src/lib/modules/user-story/infer-surface-from-story-key.util.ts`:
   - `API-…` infers `api`, `MCP-…` infers `mcp`, `CLI-…` infers `cli`. Anything else infers nothing → unspecified.
3. **In-scope set**: stories whose effective surface is `cli`.
4. **Out-of-scope set**: every other story. Log each with one line, then move on:
   - `Skipped <STORY-KEY> (<title>): surface=<value> (explicit|inferred|unspecified); this skill targets cli.`

Record the `(explicit|inferred)` tag per in-scope story too — Phase 5c flags inference-routed stories in the coverage summary.

If the in-scope set is empty, halt with:
> No CLI stories found in this area (no story has surface=cli, explicit or inferred). Skipping.

#### 1c. Check for existing tests

Look under the CLI test directory. If specs exist for some stories, ask: Fill gaps / Replace all / Cancel.

### Phase 2 — Verify harness scaffolding

#### 2a. Test runner

Jest is the most natural fit for subprocess tests (it sets `testEnvironment: 'node'` cleanly and handles per-test isolation). Vitest works too. Confirm the CLI app has a config file (e.g. `jest.config.areas.ts` or `vitest.config.areas.ts`) with:

- `testMatch` / `include` covering `tests/areas/**/*.test.ts`
- Per-test timeout ≥ 20000ms (subprocess spawn + boot adds 200ms-1s per assertion)
- `maxWorkers: 1` (or equivalent) — serialize to keep RPC roundtrips deterministic when tests share backend state

#### 2b. CLI spawn helper

Confirm a helper exists (or build it) that exposes:

- `runCli(args, opts)` — spawns the built CLI binary with the given args, captures stdout / stderr / exitCode, per-test timeout. Honors a config-file-path env var (so each test gets an isolated CLI config). Bypasses any local-only TLS check the project tolerates.
- `writeCliConfig(overrides)` — writes a minimal valid CLI config to a temp file (perms 0600) and returns the path.
- `withCliConfig(overrides, fn)` — write config, run `fn`, delete config.
- `getCliPath()` — resolves the binary path; defaults to the project's standard build output.

#### 2c. Fixture user helper

If the CLI's command surface needs an authenticated session (most real commands do), confirm a `withCliFixture(overrides, fn)` exists that provisions a fresh user + API key via the backend, writes a CLI config file pointing to that key, and yields `{...freshUser, configPath}` to `fn`. Cleanup runs in `finally`.

#### 2d. Built CLI binary present

Run `ls <cliBinaryPath>`. If missing, halt and tell the user to build first (`npx nx build <cli-app> --no-tui`, or whatever the project's build command is).

#### 2e. Backend running

If commands hit a backend, probe its base URL. Halt with a clear instruction if down.

### Phase 3 — Analyse testability

Classify each AC:

- **One-shot testable** — verifiable via a single `runCli([...args])` invocation + assertion on stdout / stderr / exitCode.
- **State-setup testable** — requires backend state set up via the project's RPC helper before the CLI invocation.
- **Untestable in one-shot mode** — REPL-only ACs (interactive prompts, tab completion, arrow-key history) require a PTY. Document and skip.
- **Untestable without external infra** — LLM-driven ACs (natural-language commands, chat sessions) require an AI provider key + deterministic responses. Document and skip (or write coverage that asserts only the invocation surface, not the LLM output content).
- **Documentation-only** — claims about how data is stored on disk (file shape, mode bits) — assert by inspecting the artifact post-invocation.

### Phase 4 — Create tests

#### 4a. Test structure (MANDATORY)

```ts
import { afterAll, beforeAll, describe, test, expect } from '@jest/globals';
import * as fs from 'fs';
import { runCli, getCliPath } from '../../../helpers/cli-spawn';
import { withCliFixture } from '../../../helpers/cli-fixture';
import { closeDb, rpcCall } from '../../../helpers/api-shared';

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

  beforeAll(() => {
    if ( ! fs.existsSync(getCliPath()) ) {
      throw new Error(`[CLI harness] CLI binary not found at ${getCliPath()}.`);
    }
  });

  afterAll(async () => { await closeDb(); });

  test('AC1: <focused assertion>', async () => {
    await withCliFixture({}, async (ctx) => {
      const res = await runCli(['/cmd', 'subcmd', 'arg'], { configPath: ctx.configPath });
      expect(res.exitCode).toBe(0);
      expect(res.stdout).toContain('expected substring');
    });
  });

  // ... one test() per testable AC ...
});
```

**Strict requirements:**

1. **File header comment** lists story key/title, testable ACs, and untestable ACs (with reason).
2. **Exactly one `describe(...)`** per spec, named `'<STORY-KEY> — <story title>'`.
3. **One `test('ACn: ...', ...)` per testable AC.** Name MUST begin with `ACn:`.
4. **Each `test()` is self-contained** — own user, own API key, own config file. Cleanup is automatic.
5. **`beforeAll` only for the binary-exists health check.**
6. **`afterAll` only for `closeDb()`** — release the cached MongoDB connection.

**FORBIDDEN — bundled-AC structure (do not generate):**

```ts
// ✗ One test() walking through multiple ACs:
test('user can list, view, and reorder products', async () => { /* masks per-AC failures */ });
```

#### 4b. Test writing guidelines

- **`withCliFixture` for anything that hits the backend.** It writes the config + provisions the API key. Use the lower-level `withCliConfig` only for pure config behavior with no backend roundtrip.
- **Set up backend state via `rpcCall(ctx.apiKey, ...)` before invoking the CLI.** The CLI reads; the test sets up. Don't try to do setup through the CLI itself unless the AC is about that.
- **Assert on exit code first, then content.** `expect(res.exitCode).toBe(0)` for happy paths, `expect(res.exitCode).not.toBe(0)` for error paths.
- **Match content semantically, not verbatim.** Substrings, IDs, JSON shapes — not full strings (CLI output reflows on cosmetic changes).
- **Use the CLI's `--json` flag for assertions on data-bearing output.** Plain output is for humans; JSON is for tests. Most list/view commands should support `--json`.
- **REPL / LLM ACs are usually untestable** — document them as such and skip; cover what you can via one-shot equivalents.
- **Avoid sleeps.** A CLI subprocess returns when done.
- **One AC per `test()`** — never combine.

#### 4c. Write files

Create files under `<cliTestDir>/<AREA>/<STORY-KEY>/<short-name>.test.ts`. Batch 5-10 per write pass.

#### 4d. Retrofit existing bundled-AC tests in the area

Split bundles into one test per AC. Preserve assertion bodies. Same run.

### Phase 5 — Run & Report

#### 5a. Run

```bash
npx jest --config=<jestConfig> <cliTestDir>/<AREA>
```

Each subprocess is ~200ms-1s; a typical 6-AC story runs in ~6-10s.

#### 5b. Fix

Classify each failure as **test fault** (wrong args, wrong assertion, fixture off — fix the spec) or **product fault** (the CLI doesn't honor the documented behavior — surface to the user; either fix the implementation paired with their authorization, or `.skip` the AC and tag the story with `production-gap`).

#### 5c. Present coverage summary

| Story | Key | Surface | Total ACs | Testable | `test()` calls | Passing | Status |
|---|---|---|---:|---:|---:|---:|---|
| ... | ... | `cli` *(inferred)* / `cli` | ... | ... | ... | ... | Pass/Fail |

Mark the **Surface** column with *(inferred)* for any story routed in via inference rather than an explicit `surface` value — those are candidates for a `patchUserStory /surface` backfill.

**Out-of-scope stories** (other surfaces, skipped):
- `<STORY-KEY>` — `<surface>` (explicit|inferred|unspecified). Use `/defprod-create-<surface>-tests`, or `/defprod-create-area-tests` for an area-wide walk.

Flag mismatches. List untestable items + reasons. List production-gap tags applied. List any production fixes applied in this run.

Suggest:

> **Next steps:**
> - Backfill `surface` on any inference-routed stories you want to lock to CLI.
> - Run `/defprod-run-area-tests <area>` to produce a classified failure report.

---

## Rules

- **Every story gets a spec file.**
- **Every testable AC gets its own `test()`.**
- **Test structure is non-negotiable** — one describe per story, one `test('ACn: ...')` per AC.
- **CLI subprocesses are expensive (~200ms-1s)** — keep specs lean.
- **Honor the build-prerequisite contract** — every spec includes the binary-exists health check in `beforeAll`.
- **REPL / LLM / tab-completion are PTY-only** — document as untestable, don't half-test with `expect()` against unstable output.
- **Match existing helper / fixture / import conventions.**
- **Don't modify production code** in this skill — surface product faults to the user.
- **No new dependencies.** Jest/Vitest, `child_process`, native `fs`, the project's existing helpers.
- **Test names trace to criteria** — every `test()` begins with `ACn:`.
- **Retrofit bundled-AC specs in the same run.**
- **Batch file creation** — 5-10 files per pass.
