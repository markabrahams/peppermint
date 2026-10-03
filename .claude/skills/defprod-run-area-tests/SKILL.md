---
name: defprod-run-area-tests
description: Runs e2e tests for a product area, classifies each failure as test fault or production fault, and saves a structured report.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Write
  - AskUserQuestion
  - mcp__defprod__listProducts
  - mcp__defprod__listAreas
  - mcp__defprod__getArea
  - mcp__defprod__listUserStories
  - mcp__defprod__getUserStory
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

# Run Area Tests

Runs end-to-end tests for a product area, analyses every failure and flaky test, classifies each as a test fault or production fault, and saves a structured report. The report drives `/defprod-fix-test-failures`.

## When to use

- When you want to check the health of a product area's test suite.
- After making changes that might affect an area's tests.
- When invoked via `/defprod-run-area-tests <product-name> <area-key>`.

## Inputs

- **Product name** — the DefProd product.
- **Area key** — the area to run tests for (e.g. `AUTH`, `BILLING`).

## Config

This skill consults `.defprod/defprod.json` for optional hints. If the file doesn't exist, the skill discovers everything automatically.

| Key | Type | Purpose |
|-----|------|---------|
| `products[].name` | `string` | Matches product name to locate config entry |
| `products[].e2eDir` | `string` | Path to the e2e test directory |

---

## Workflow

### Step 1 — Locate tests

Find the e2e tests for this area:

1. Check `e2eDir` from `.defprod/defprod.json` if available.
2. Search for test files in common patterns:
   - `e2e/areas/<AREA-KEY>/`
   - `e2e/<area-name>/`
   - Test files matching the area key in their path
3. Verify that `.spec.ts` (or `.test.ts`, `.cy.ts`, etc.) files exist.

If no tests are found, tell the user and suggest running `/defprod-create-area-tests` first.

---

### Step 2 — Run tests

Run the test suite for this area using the project's test runner.

**Playwright example:**
```bash
npx playwright test <test-path> --reporter=json,list
```

**Cypress example:**
```bash
npx cypress run --spec "<test-path>" --reporter json
```

Use whichever framework the project uses. If the project has a custom test command, use that.

Capture:
- Test results (pass/fail/skip/flaky per test)
- Error messages and stack traces for failures
- Timing information
- Any retry results (tests that failed then passed on retry are flaky)

---

### Step 3 — Collect failures and flaky tests

Extract every test that:
- **Failed** — did not pass on any attempt
- **Flaky** — failed on at least one attempt but passed on a retry

Record for each:
- Test name and file path
- Error message
- Stack trace
- Which attempt(s) failed

---

### Step 4 — Analyse each failure

For each failed or flaky test:

#### 4a. Read the test file

Understand what the test is trying to verify. Map it to the relevant user story and acceptance criteria using the test's comments or naming.

#### 4b. Read the error

Understand what actually went wrong:
- Timeout? Element not found? Assertion failed?
- What was the expected vs actual behaviour?

#### 4c. Cross-reference production code

Read the relevant production code that the test exercises. Check:
- Does the production code match what the test expects?
- Has the production code changed since the test was written?
- Is there a genuine bug in the production code?

#### 4d. Cross-reference user story

Call `getUserStory` from the DefProd MCP server if needed. Check:
- Do the acceptance criteria match what the test verifies?
- Has the story been updated since the test was written?

#### 4e. Classify the fault

- **Test fault** — the test code is wrong (bad selector, incorrect assertion, timing issue, outdated expectation). The production code is correct.
- **Production fault** — the production code has a bug or regression. The test is correctly checking the right thing and the behaviour doesn't match the acceptance criteria.

Provide a brief explanation of why you classified it this way.

---

### Step 5 — Save report

Write a structured report to the test directory, e.g. `<test-path>/<AREA-KEY>-test-check.md`.

#### Report format

```markdown
# Test Check — <Area Name>

**Date:** <YYYY-MM-DD>
**Total:** <N> | **Passed:** <N> | **Failed:** <N> | **Flaky:** <N>

## Summary

| # | Test | File | Status | Fault | Classification |
|---|------|------|--------|-------|----------------|
| 1 | test name | file.spec.ts:line | Failed | Test | Bad selector |
| 2 | test name | file.spec.ts:line | Flaky | Production | Race condition |

## Details

### 1. <Test name>

- **File:** `path/to/file.spec.ts:42`
- **Status:** Failed
- **Error:** `Element not found: [data-testid="submit-btn"]`
- **Classification:** Test fault
- **User story:** AREA-KEY — Story title
- **Acceptance criteria:** AC text
- **Analysis:** The submit button's data-testid was changed from "submit-btn" to "submit-button" in commit abc123. The test selector needs updating.
- **Recommended fix:** Update the selector in the test file to match the new data-testid.

### 2. <Test name>

...
```

---

### Step 6 — Output summary

Print a summary to the console:

> **Test results for {Area Name}:**
> - {N} passed, {N} failed, {N} flaky
> - {N} test faults, {N} production faults
>
> Report saved to `<path>/<AREA-KEY>-test-check.md`
>
> **Next step:** Run `/defprod-fix-test-failures <area>` to fix the failures.

---

## Rules

- **Analyse every failure** — don't skip or batch failures. Each one gets individual analysis and classification.
- **Read production code before classifying** — never classify a failure without reading the relevant production code. A failing test might be correct.
- **Be specific in recommendations** — "update the selector" is not enough. Specify which selector, in which file, and what it should be changed to.
- **Flaky tests are failures** — treat them with the same analysis rigour. A flaky test indicates either a test timing issue or a production race condition.
- **Don't fix anything** — this skill analyses and reports. Fixes happen in `/defprod-fix-test-failures`.
- **Always save the report** — even if all tests pass, save a report confirming the clean run.
