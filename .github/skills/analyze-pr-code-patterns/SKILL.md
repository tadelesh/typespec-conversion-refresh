---
name: analyze-pr-code-patterns
description: >
  Summarize an arbitrary GitHub pull request's file diff by categories and surface
  the recurring before/after code change patterns. Accepts a PR URL such as
  https://github.com/Azure/azure-sdk-for-go/pull/26354 and outputs a deduplicated,
   GitHub-style diff report inline in chat and writes the same report to a workspace file.
---

# Analyze PR Code Change Patterns

Summarize the file diff of any GitHub pull request by file-role categories, then mine and deduplicate the recurring before/after code change patterns. Output an inline Markdown report with categorized counts and real diff hunks copied verbatim from the PR, and save the same report to a workspace Markdown file.

## Input

The user provides:

- A PR URL, for example `https://github.com/Azure/azure-sdk-for-go/pull/26354`.
- Analyse PR code changes， for exmple `deleted、added or replaced code patterns`.

No other input is required. The skill must work for arbitrary GitHub repositories, not just `azure-sdk-for-go`.

## Guardrails

- Read-only. Do not push, comment, or modify the PR.
- Never paraphrase diff content. Diff hunks shown to the user must be copied verbatim from the PR patches.
- Do not invent occurrence counts. Counts must come from actual pattern matching across the fetched patches.
- Exclude comment-only changes from pattern mining and from the final report. Comment cleanup, generated docstring movement, and other non-code comment-only edits must not be listed as recurring code change patterns.
- Exclude package-reference-only changes from pattern mining and from the final report. Package declarations, import list edits, and module or package path reference rewrites without nearby behavioral code changes must not be listed as recurring code change patterns.
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

### 4. Mine recurring change patterns

For each file's `patch` text:

1. Split into hunks at `@@`.
2. Within each hunk, walk the lines in order and group contiguous runs of changed lines into **change blocks**. A change block is a maximal run of consecutive `-` and/or `+` lines (ignoring `---` / `+++` headers). Each block falls into exactly one of three shapes:
   - **Replacement** — the block contains both `-` and `+` lines.
   - **Pure deletion** — the block contains only `-` lines.
   - **Pure addition** — the block contains only `+` lines.

   You MUST capture all three shapes. Do not drop pure-deletion or pure-addition blocks just because they have no counterpart. Some patterns in real PRs (e.g. an entire helper being removed, or a whole `select` / channel setup being deleted) only appear as pure deletions.

3. Before normalization, filter out **comment-only** change blocks. If every changed line in the block is a comment line, comment delimiter, or blank line after removing the leading `+` / `-`, skip the block entirely. This includes standalone `// ...`, `/* ... */`, `* ...`, `# ...`, HTML comments, doc comment reflow, and generated-comment-only additions or removals. If a block mixes code and comments, keep it and preserve the real diff block verbatim.
4. Also filter out **package-reference-only** change blocks. If every changed line in the block is only a package declaration, import entry, require/replace/module reference, or a quoted package path/module path reference, skip the block entirely. Examples include Go `package ...` lines, lines inside `import (...)`, standalone import string entries, and versioned package path rewrites such as `/v3` to no-suffix imports. If a block mixes package references with executable code or behavioral API usage changes, keep it.

5. Normalize each block aggressively so semantically-equivalent variations collapse into one pattern. Apply these rules in order:
   - Strip leading/trailing whitespace on each line.
   - Replace API version literals (`\d{4}-\d{2}-\d{2}(?:-preview)?`) with `<APIVER>`.
   - Replace module-path major-version segments like `/v5`, `/v6` with `/v<N>`.
   - Replace quoted semver literals like `"1.2.3"` or `"2.0.0-beta.1"` with `"<SEMVER>"`.
   - Replace version-named identifiers like `version20240601Preview` with `version<TS>`.
   - Replace identifier prefixes that carry a per-parameter or per-field name in front of common generator-suffix tokens. Specifically, for any identifier ending in one of `Unescaped`, `Unencoded`, `Param`, `Var`, `Value`, `Server`, `Transport`, `Client`, `Factory`, `Pager`, `Poller`, `Response`, `Request`, `Options`, replace the leading identifier portion with `<ID>` so e.g. `startTimeUnescaped`, `endTimeUnescaped`, `timeGrainUnescaped` all collapse to `<ID>Unescaped`, and `dispatchGetFooPager`, `dispatchGetBarPager` collapse to `dispatch<ID>Pager`.
   - Replace quoted query-parameter literals inside `qp.Get("...")`, `req.URL.Query().Get("...")`, and similar accessors with `qp.Get("<KEY>")` so per-field variants of the same shape unify.
   - Replace bare service-specific Go type names (CamelCase identifiers used as type references) inside type assertions, struct literals, and function parameters with `<TYPE>` when they are clearly tied to the package name (e.g. `AgriServiceClient`, `AgriServiceServerTransport` → `<TYPE>`). Do NOT touch standard library or `azcore` / `arm` / `runtime` types.
   - Preserve operator characters, keywords, control flow, and standard library identifiers so distinct semantic patterns stay distinct.
   - Tag the normalized block with its shape (`replacement`, `deletion`, or `addition`) so the same lines under a different shape do not collapse together.
6. Count how many files (and how many blocks) each normalized pattern appears in.
7. List EVERY distinct normalized pattern, sorted by file-count descending then block-count descending. Do not apply a Top-N cap. Patterns that occur only once MUST still be listed at the tail of the report. The only exception is binary or unparseable patches, which are reported separately as a count.
8. Exhaustive display is mandatory: the final report MUST enumerate all remaining non-comment, non-package-reference pattern entries. You MUST NOT return only a high-level summary, only top patterns, or an abbreviated sample list.

For each pattern, retain ONE real, un-normalized example block to show the user, including its `@@` header and 1–2 lines of surrounding context.

### 5. Build the report

Output an inline Markdown report with the following sections:

1. **Header**
   - PR title, URL, base/head branches, total changed files.
2. **Category breakdown**
   - One row per category from Step 3 with counts and 1 representative file path.
3. **Recurring code change patterns**
   - Numbered list of ALL distinct non-comment, non-package-reference patterns, sorted by file-count descending. Include every pattern, even those with file-count = 1. For each:
     - Short pattern name.
     - Category, shape (`replacement` / `deletion` / `addition`), file-count, and block-count.
     - One real diff block copied verbatim, fenced as ```diff with `-`and/or`+`lines and 1–2 lines of context. Pure-deletion blocks must be shown as`-`-only, pure-addition blocks as `+`-only — do not fabricate a counterpart side.
   - To keep the report readable when there are many tail patterns, you MAY group the trailing patterns that have file-count = 1 under a single "## Singleton patterns" subsection, but every singleton pattern must still be listed individually with its own diff block.
   - You MUST output the full numbered list from first pattern to last pattern; do not truncate with wording like "..." or "remaining patterns omitted".
4. **Risk-oriented summary**
   - 2–4 bullet points calling out the patterns most worth reviewing manually (e.g. behavior changes in core source, module path / major-version changes, dependency changes), and which patterns are safe noise (e.g. comment cleanup, doc/log typo fixes).

Do not omit any remaining non-comment, non-package-reference pattern. The diff blocks themselves should remain minimal (no unrelated surrounding lines), but the pattern list itself must be exhaustive after comment-only and package-reference-only filtering.

### 6. Write report to file

After composing the Markdown report in Step 5, write it to the workspace root as:

`PR_<pr_number>_code_change_report.md`

Example for PR 26354:

`PR_26354_code_change_report.md`

Rules:

- The file content must match the inline report exactly (no summary-only truncation in the file).
- Overwrite the same file name if it already exists for that PR number.
- Keep Markdown valid and preserve all ` ```diff ` blocks verbatim.

## Final response format

Return the inline Markdown report described in Step 5, and include one final line confirming the output file path written in Step 6.
Do not replace the full pattern list with a summary response.

## Example input

- `https://github.com/Azure/azure-sdk-for-go/pull/26354`
