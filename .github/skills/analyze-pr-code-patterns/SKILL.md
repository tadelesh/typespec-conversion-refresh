---
name: analyze-pr-code-patterns
description: >
  Summarize an arbitrary GitHub pull request's file diff by categories and surface
  the recurring before/after code change patterns. Accepts a PR URL such as
  https://github.com/Azure/azure-sdk-for-go/pull/26354 and outputs a deduplicated,
  GitHub-style diff report inline in chat.
---

# Analyze PR Code Change Patterns

Summarize the file diff of any GitHub pull request by file-role categories, then mine and deduplicate the recurring before/after code change patterns. Output an inline Markdown report with categorized counts and real diff hunks copied verbatim from the PR.

## Input

The user provides:

- A PR URL, for example `https://github.com/Azure/azure-sdk-for-go/pull/26354`.

No other input is required. The skill must work for arbitrary GitHub repositories, not just `azure-sdk-for-go`.

## Guardrails

- Read-only. Do not push, comment, or modify the PR.
- Never paraphrase diff content. Diff hunks shown to the user must be copied verbatim from the PR patches.
- Do not invent occurrence counts. Counts must come from actual pattern matching across the fetched patches.
- If the PR has more than ~100 files, do not page only the first 100 — fetch all pages.
- If `gh` is unavailable or unauthenticated, stop and report the blocker.

## Steps

### 1. Parse PR URL

1. Extract `owner`, `repo`, and `pr_number` from the URL.
2. Confirm `gh` CLI is available and authenticated for that host.

### 2. Fetch all changed files and patches

Use the GitHub REST API with pagination so all files are retrieved:

```sh
gh api repos/{owner}/{repo}/pulls/{pr_number}/files --paginate > files.json
```

Each entry has `filename`, `status`, `additions`, `deletions`, and `patch`. Note that very large or binary diffs may have no `patch`; skip those for pattern mining but still count them in the category totals.

### 3. Categorize files by role

Bucket every changed file into one of the following categories based on path patterns. Categories are language- and repo-agnostic by intent, but the path heuristics below cover the common Azure SDK layout and most other repos:

- `fake / mock` — paths containing `/fake/`, `/mocks/`, or files ending in `_mock.go` / `_fake.go`.
- `example test` — files ending in `_example_test.go` or under `/examples/`.
- `test` — other test files (`_test.go`, `__tests__/`, etc.) not already captured above.
- `testdata / fixtures` — paths containing `/testdata/`, `/fixtures/`, or recording assets.
- `module metadata` — `go.mod`, `go.sum`, `package.json`, `pyproject.toml`, `Cargo.toml`, `setup.py`, etc.
- `docs / changelog` — `*.md`, `CHANGELOG*`, `README*`, `docs/**`.
- `core source` — everything else under the repo's source tree.

Report total count and a short per-category count.

### 4. Mine recurring before/after patterns

For each file's `patch` text:

1. Split into hunks at `@@`.
2. For each hunk, collect ordered pairs of contiguous removed lines (lines starting with `-`, excluding the `---` header) and added lines (lines starting with `+`, excluding the `+++` header).
3. Normalize each `(removed, added)` pair to make pattern grouping work across services:
   - Strip leading/trailing whitespace.
   - Replace identifier tokens that are clearly service- or version-specific (e.g. type names, package names, API version literals like `2024-06-01-preview`, semantic version strings like `v5` / `v6`) with placeholders.
   - Preserve operator characters and structural keywords so distinct patterns stay distinct.
4. Count how many files (and how many hunks) each normalized pattern appears in.
5. Rank patterns by occurrence and keep the top 8–12.

For each kept pattern, also retain ONE real, un-normalized example hunk to show the user.

### 5. Build the report

Output an inline Markdown report with the following sections:

1. **Header**
   - PR title, URL, base/head branches, total changed files.
2. **Category breakdown**
   - One row per category from Step 3 with counts and 1 representative file path.
3. **Recurring code change patterns**
   - Numbered list of the top patterns. For each:
     - Short pattern name.
     - Category and approximate occurrence count.
     - One real diff hunk copied verbatim, fenced as ```diff with `-` and `+` lines and 1–2 lines of context.
4. **Risk-oriented summary**
   - 2–4 bullet points calling out the patterns most worth reviewing manually (e.g. behavior changes in core source, module path / major-version changes, dependency changes), and which patterns are safe noise (e.g. comment cleanup, doc/log typo fixes).

Keep the report concise. Do not dump every file or every hunk.

## Final response format

Return ONLY the inline Markdown report described in Step 5. Do not write any files to the workspace.

## Example input

- `https://github.com/Azure/azure-sdk-for-go/pull/26354`
