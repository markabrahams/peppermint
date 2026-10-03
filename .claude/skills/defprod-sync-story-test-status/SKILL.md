---
name: defprod-sync-story-test-status
description: Runs scripts/sync-story-test-status.sh to detect per-story test coverage across the project's test harnesses (Playwright e2e, Vitest, Jest, or any mix), execute the suites, and POST results to DefProd so the test-status dashboard reflects current state. Reads credentials and the multi-suite test config from an env file (default ~/.config/defprod/prod.env).
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

# Sync Story Test Status

Runs `scripts/sync-story-test-status.sh` (shipped separately in the `@defprod/scripts` package, or copied into your repo's `scripts/` directory) so the live DefProd product reflects the current state of your test suites — covered vs. uncovered stories, and pass/fail/flaky results for each covered story.

The script understands multiple harnesses in a single invocation: Playwright (UI e2e), Vitest, and Jest. Configure a list of suites and the script walks each one for coverage, runs it with its JSON reporter, and posts a merged sync payload.

## When to use

- After landing material changes to any test suite (new tests, retrofits, deletions) and you want production to reflect the new coverage map.
- When the user asks to "sync test status to production", "push test results to DefProd", or invokes `/defprod-sync-story-test-status`.
- When story coverage badges in the DefProd UI look stale.

## Prerequisites

1. **Sync script** — `scripts/sync-story-test-status.sh` present in the repo (typically installed from `@defprod/scripts`).
2. **Env file** — a config file (default `~/.config/defprod/prod.env`, permissions `0600`) with:
   - `DEFPROD_PRODUCT_ID`
   - `DEFPROD_API_URL` (production RPC endpoint, e.g. `https://app.defprod.com/api/v1/rpc`)
   - `DEFPROD_API_KEY` (read-write product-scoped key)
   - `DEFPROD_TEST_SUITES` — semicolon-separated list of `harness:dir:config` entries. `harness` is one of `playwright`, `vitest`, `jest`. Example:

     ```
     DEFPROD_TEST_SUITES=playwright:e2e:playwright.config.ts;vitest:apps/backend/tests/areas:apps/backend/vitest.config.api.ts;jest:apps/cli/tests/areas:apps/cli/jest.config.ts
     ```

   - (Optional / legacy) `DEFPROD_TEST_DIR` + `DEFPROD_PLAYWRIGHT_CONFIG` — only consulted when `DEFPROD_TEST_SUITES` is unset; the script then falls back to a single Playwright suite.
   - (Optional) `DEFPROD_PLAYWRIGHT_PROJECTS` — comma-separated list of Playwright projects to run (default `chromium`). Set empty to run every project the config declares.
3. **Test runners installed** — each suite's runner (`@playwright/test`, `vitest`, `jest`) must be on the project's PATH or in `node_modules/`. The script invokes them via `npx`.
4. **Build prerequisites met** — e.g. CLI-spawning Jest suites need the binary built before the suite runs. The skill won't run build steps for you.
5. **App / backend running** if the suites require it (Playwright e2e, integration tests against a local server, etc.). The skill assumes they are already up.

## Arguments

| Arg | Purpose |
|-----|---------|
| `--area-key <KEY>` | Optional. Narrow scope to a single area (e.g. `CORE`, `API`, `CLI`). The orphan-delete in `syncStoryTestStatus` is also scoped to this single area, so other areas are unaffected. Omit for a full-product sync. |
| `--test-suites <list>` | Optional. Override `DEFPROD_TEST_SUITES` for this run only — useful for syncing a single harness without touching others. Same semicolon-separated `harness:dir:config` format. |
| `--project <list>` | Optional. Comma-separated Playwright project names; overrides `DEFPROD_PLAYWRIGHT_PROJECTS`. |
| `--env-file <path>` | Optional. Override the default env-file location (default `~/.config/defprod/prod.env` or `DEFPROD_ENV_FILE`). |
| `--dry-run` | Optional. Run coverage + tests but print the payload instead of POSTing. |
| `--skip-run` | Optional. Check coverage only; do not execute the harnesses. Fast way to refresh the covered/uncovered flags. |

## Multi-area sync safety

`syncStoryTestStatus` upserts every record in the payload AND deletes any record whose story key is **not** in the payload but whose `areaKey` IS in the payload's `areaKeys` list. The script populates `areaKeys` from the API's `listAreas` response — so a full-product sync without `--area-key` will:

1. Upsert every story present in any covered test dir under any suite.
2. Mark every story not covered by any listed suite as `uncovered`.
3. Delete prior records for areas that no listed suite covers at all.

For a production sweep this is fine **only if** `DEFPROD_TEST_SUITES` covers every area that already has a dashboard entry. For partial syncs (e.g. syncing a single harness while others are paused), use `--area-key` to scope the run and prevent collateral overwrites.

If an area spans multiple harnesses (a common pattern is a "status" or "operations" area with both UI and CLI surfaces), make sure all the relevant suites are listed before syncing that area — otherwise the harnesses that aren't in the config will see their stories downgraded.

## Workflow

### Step 1 — Confirm scope

Read the destination from the env file so the user can see what they are about to write to. Use `grep` rather than `cat` to avoid echoing the API key:

```bash
grep -E '^(DEFPROD_API_URL|DEFPROD_PRODUCT_ID|DEFPROD_TEST_SUITES)=' ~/.config/defprod/prod.env
```

Show the URL, product ID, and the list of suites to the user. Confirm with `AskUserQuestion` if any of these are true:

- The user did not pass `--dry-run`.
- `DEFPROD_API_URL` points at a production host.
- An `--area-key` was not specified (i.e. a full-product write).

The default option should be **"Yes, sync to production"**; the alternative is **"Run as --dry-run first"**.

### Step 2 — Run the script

Invoke from the repository root so relative paths in the env file resolve correctly:

```bash
./scripts/sync-story-test-status.sh \
  --env-file ~/.config/defprod/prod.env \
  [--area-key <KEY>] \
  [--test-suites <list>] \
  [--project <list>] \
  [--dry-run] \
  [--skip-run]
```

For each suite listed in `DEFPROD_TEST_SUITES`, the script will:

1. Fetch the product's areas and user stories via the RPC API.
2. Walk the suite's `<dir>/[areas/]<AREA>/<STORY-KEY>/` tree to detect coverage. The file glob depends on the harness — `*.spec.ts` for Playwright, `*.test.ts` for Vitest/Jest.
3. Run the harness with its JSON reporter (Playwright via package-local binary; Vitest via `npx vitest run -c <config> --reporter=json`; Jest via `npx jest --config <config> --json`).
4. Parse the reporter output into per-story totals (passed / failed / flaky / skipped, plus failed test titles).

After all suites finish, the script merges per-story results (first-wins on overlap), enriches each story from the API with its coverage flag, and POSTs a `syncStoryTestStatus` use-case request (unless `--dry-run`).

A full one-shot sync takes wall-clock proportional to your Playwright suite. To run just one area faster, set `--area-key` plus a `--test-suites` value that contains only the harness covering that area.

### Step 3 — Report results

After the script exits, surface to the user:

- The per-suite summary (`Suite: <harness> <dir>` and `Covered stories: N`).
- The per-story summary (one line per story with pass/fail counts and any failed test titles — appears under `Per-story summary:`).
- The final `Upserted: N, Deleted: M` line from the script.
- If the script exited non-zero, the error from the script's `stderr` (common causes: missing browser binaries, malformed `DEFPROD_TEST_SUITES`).

If `--dry-run` was used, point out that nothing was written and offer to re-run without `--dry-run`.

## Rules

- **Read the env file once and show it to the user.** Production writes are irreversible at the orphan-delete step; confirm scope first.
- **Never edit the env file.** If a key is missing or wrong, tell the user; do not modify it.
- **Never echo the API key.** When showing env file contents, restrict output to `DEFPROD_API_URL`, `DEFPROD_PRODUCT_ID`, and `DEFPROD_TEST_SUITES`.
- **Confirmation is mandatory** for production writes — bypass only when the user explicitly passed `--dry-run`.
- **Honor `--area-key` for partial syncs.** Without it, the script's `areaKeys` payload list spans the full product, which triggers orphan-delete in every area. Use `--area-key` when only some of the suites are being run.
- **Do not start or stop dev servers** from this skill. Assume the project's harness prerequisites are already met.
- **Do not retry on partial failure** beyond one re-run with `--skip-run` to refresh coverage flags only. Test failures should be investigated, not papered over.
