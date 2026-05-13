---
name: list-mgmt-customer-reported-issues
description: >
  List open issues in Azure/azure-sdk-for-go that have both mgmt and customer-reported labels,
  including title, URL, assignees, issue type, status, and optionally all comments in time order
  in a CSV file.
---

# List Mgmt + Customer-Reported Open Issues

List all open issues in `Azure/azure-sdk-for-go` that contain both labels:

- `Mgmt`
- `customer-reported`

## Input

Optional user inputs:

- Repository override (default: `Azure/azure-sdk-for-go`)
- Whether to include comments in output
- Whether to save to Excel
- Output file path (default: `azure-sdk-for-go-mgmt-customer-reported-open-issues-02.xlsx`)

## Output schema

Return or save rows with these fields:

- `Issue #`
- `Title`
- `URL`
- `Assignees`
- `Issue Type`
- `Status` (Analyse comments and labels, set Status value, e.g., `need service attention`, `Need author feedback`, `Suggested user to open tickets to related service to fix specs`, `Investigating`, `Need to release new version`, etc.)
- `Comments` (single column, ordered by time, optional, and save the comments line by line with timestamp and author)

## Issue Type mapping

Use this normalized set:

- `Issue Type`
- `Usage Question`
- `Code Gen Issue`
- `Service Issue`
- `Spec Issue`
- `Azure Core Issue`
- `Serialization Issue`
- `Feature Request`

Suggested classification rules:

- If title/body contains `feature request`, `request`, `missing field/client`, classify as `Feature Request`.
- If title/body contains `unmarshal`, `marshal`, `serde`, `case-sensitive JSON`, classify as `Serialization Issue`.
- If issue points to generator/fake transport/autorest output problem, classify as `Code Gen Issue`.
- If issue points to service behavior/runtime backend mismatch, classify as `Service Issue`.
- If issue points to OpenAPI/TypeSpec/spec contract mismatch, classify as `Spec Issue`.
- If issue concerns `azcore` pipeline/core abstractions, classify as `Azure Core Issue`.
- If issue is primarily how-to usage, classify as `Usage Question`.
- If none clearly applies, use `Issue Type`.

## Status mapping

Use this normalized set:

- `Todo`
- `In Progress`
- `Resolved`
- `Need author feedback`
- `Suggested user to open tickets to related service to fix specs`
- `Investigating`

Suggested classification rules:

- If there is no label or comments added, classify as `Todo`.
- If comments indicate active investigation or a PR is linked, classify as `In Progress`.
- If issue is closed or comments indicate resolution, classify as `Resolved`.
- if comments indicate waiting on user response, classify as `Need author feedback`.
- if comments indicate issue is due to service behavior or spec contract and user is advised to open tickets to related service team, classify as `Suggested user to open tickets to related service to fix specs`.
- if comments indicate issue is under investigation but no progress yet, classify as `Investigating`.
- if comments indicate issue is being tracked but no active work, classify as `Todo`.

## Steps

### 1. Query matching open issues

Use GitHub search to find issues with both labels:

```sh
gh api --paginate \
  'search/issues?q=repo:Azure/azure-sdk-for-go+is:issue+is:open+label:Mgmt+label:customer-reported&per_page=100' \
  --jq '.items[] | [.number, .title, .html_url] | @tsv'
```

### 2. Fetch details and comments for each issue

For each issue number, fetch:

- assignees
- labels/state for status
- comment timeline (created order)

Example:

```sh
gh api repos/Azure/azure-sdk-for-go/issues/<number>
gh api repos/Azure/azure-sdk-for-go/issues/<number>/comments --paginate
```

### 3. Classify issue type

Apply the mapping rules above against title/body/comments context.

### 4. Format comments

If comments requested, produce one string per issue in chronological order:

`[YYYY-MM-DD author] comment text | [YYYY-MM-DD author] comment text ...`

### 5. Output

Provide either:

- Markdown table in chat, or
- Excel file with header:

```excel
Issue #,Title,URL,Assignees,Issue Type,Status,CommentsContent, CommentsCount
```

## Notes

- Match labels case-insensitively as GitHub normalizes labels, but preserve canonical output form.
- Keep CSV cells quoted when they contain commas/newlines.
- If no issues match, return an empty result with header only.
