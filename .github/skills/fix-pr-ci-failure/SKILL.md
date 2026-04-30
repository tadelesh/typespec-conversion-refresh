---
name: fix-pr-ci-failure
description: >
  Fix a Go SDK PR CI failure when it is a syntax error, then run all armservice live tests
  in record and playback mode, push test-proxy assets, and commit changes. Accepts a PR URL
  such as https://github.com/Azure/azure-sdk-for-go/pull/26321.
---

# Fix PR CI Failure + Live Test Validation

Fix syntax-only CI failures for an Azure/azure-sdk-for-go PR, then validate armservice live tests in record and playback mode.

## Input

The user provides:

- A PR URL, for example `https://github.com/Azure/azure-sdk-for-go/pull/26321`

Optional user constraints:

- Maximum replay attempts (default in this skill: 10)

## Required repository

- [Azure/azure-sdk-for-go](https://github.com/Azure/azure-sdk-for-go), cloned locally (commonly at `../azure-sdk-for-go`)

## Guardrails

- Only auto-fix CI failures that are syntax-related (for example: parser errors, compile-time syntax issues, malformed literals, missing commas/braces/parentheses, bad imports causing parse failure).
- If the failure is not syntax-related, stop and report why it is out of scope.
- Do not use destructive git commands.
- Keep edits minimal and targeted to the failing syntax issue and any live-test fixes discovered during required test runs.

## Steps

### 1. Resolve PR and prepare branch

1. Parse owner/repo/PR number from input URL.
2. In local sdk repo, fetch and checkout the PR branch.

Example:

```sh
gh pr checkout 26321 --repo Azure/azure-sdk-for-go
```

3. Confirm working tree status before editing.

### 2. Identify CI failure and classify error type

1. Inspect PR checks and failing job logs.
2. Extract the first actionable root-cause error.
3. Classify the failure:

- Syntax error or live test failure or dependency error: continue.
- Not syntax error or live test failure or dependency error: stop and report as out of scope for this skill.

Useful commands:

```sh
gh pr checks 26321 --repo Azure/azure-sdk-for-go
gh run list --repo Azure/azure-sdk-for-go --branch <pr-branch> --limit 20
gh run view <run-id> --repo Azure/azure-sdk-for-go --log-failed
```

### 3. Fix syntax error and verify locally

1. Apply a minimal code fix for the syntax issue.
2. Run the narrow local command that validates the fix (typically `go test` or `go test ./...` scoped to failing package).
3. If syntax errors remain, repeat until syntax check is clean.

### 4. Locate armservice live tests

1. Determine the armservice folder under `sdk/resourcemanager/<service>/<armservice>` related to the PR changes.
2. Detect whether this armservice has live tests (for example by finding `*_live_test.go`, `Test*TestSuite`, or existing test-proxy assets files).
3. If no live tests exist, report and skip to Step 8 (commit only syntax fix).

### 5. Prepare environment for live tests

If live tests exist for this armservice, run:

```sh
source ~/.bash_profile
```

Then ensure required env vars for record mode are set (for example subscription/tenant and any service-specific env vars).

### 6. Run all live tests in record mode

1. In armservice package path, run all live test suites with `go test -run TestxxxxxxTestSuite` pattern.
2. Include all suite names for that package in a single regex alternation.

Example:

```sh
export AZURE_RECORD_MODE="record"
go test -v -timeout 60m -run 'TestAaaTestSuite|TestBbbTestSuite|TestCccTestSuite' 2>&1
```

3. If record mode fails, fix the discovered errors, then rerun record mode until it passes.

### 7. Run playback loop (max 10 attempts)

After record mode passes:

1. Export playback mode:

```sh
export AZURE_RECORD_MODE="playback"
```

2. Rerun the same full live-test suite command.
3. If playback fails, fix the error and rerun.
4. Repeat until playback passes, with a hard limit of 10 total playback attempts.
5. If still failing after 10 attempts, stop and report unresolved playback failure.

### 8. Push test-proxy assets

When playback passes, run from the armservice path:

```sh
test-proxy push -a assets.json
```

If this fails, fix the issue and rerun until success.

### 9. Commit changes

1. Review changed files to ensure only relevant edits are included.
2. Stage and commit with a clear message summarizing:

- syntax fix
- live test fixes
- test-proxy asset update

Example:

```sh
git add <relevant-files>
git commit -m "Fix CI syntax error and pass armservice live tests in playback"
```

3. Report commit SHA and changed files summary.

## Final response format

Return:

- PR processed
- CI root cause and syntax classification result
- Files changed for syntax fix
- Record mode result
- Playback attempts used and final result
- `test-proxy push -a assets.json` result
- Commit SHA
- Any remaining risks/blockers

## Example input

- `https://github.com/Azure/azure-sdk-for-go/pull/26321`
