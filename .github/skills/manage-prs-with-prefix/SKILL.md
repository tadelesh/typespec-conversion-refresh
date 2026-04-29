---
name: manage-prs-with-prefix
description: >
  List or close Azure/azure-sdk-for-go pull requests whose titles start with a
  specific prefix such as [Migrate-Check-TypeSpec]  or [Migrate-Check-Main-Branch]. Uses GitHub CLI with exact
  prefix matching against currently open pull requests.
---

# Manage PRs with Prefix

List or close open pull requests in Azure/azure-sdk-for-go by title prefix.

## Input

The user provides one of the following:

- A title prefix, for example `[Migrate-Check-TypeSpec]` or `[Migrate-Check-Main-Branch]`
- An action, either `list` or `close`
- Optionally, a repository override if the target is not `Azure/azure-sdk-for-go`

If the user asks to close PRs, treat that as a destructive action and only do it when the request is explicit. Just deal with the input in a case-insensitive manner.

## Default repository

- [Azure/azure-sdk-for-go](https://github.com/Azure/azure-sdk-for-go)

## Steps

### 1. Resolve the repository and prefix

1. Use `Azure/azure-sdk-for-go` unless the user explicitly names another repository.
2. Use the exact prefix the user supplied.
3. Match only titles that start with that prefix.

Use GitHub CLI API queries for reliable prefix matching:

```sh
gh api --paginate 'repos/{owner}/{repo}/pulls?state=open&per_page=100' \
  --jq '.[] | select(.title | startswith("{prefix}"))'
```

This is stricter than GitHub search and avoids false positives from partial title matches.

### 2. List matching PRs

For a listing request, return the currently open PRs whose titles start with the prefix.

Use a command like:

```sh
gh api --paginate 'repos/{owner}/{repo}/pulls?state=open&per_page=100' \
  --jq '.[] | select(.title | startswith("{prefix}")) | [.number, .title, .html_url, .created_at] | @tsv'
```

Summarize the results with:

- PR number
- Full title
- URL
- Whether the prefix also matched longer variants such as `-Mitigate`

If there are no matches, say so explicitly.

### 3. Close matching PRs

For a close request:

1. Re-query the live set of open PRs using the same `startswith("{prefix}")` filter.
2. Collect the PR numbers.
3. Close each PR with GitHub CLI.

Example:

```sh
matches=(${(@f)$(gh api --paginate 'repos/{owner}/{repo}/pulls?state=open&per_page=100' \
  --jq '.[] | select(.title | startswith("{prefix}")) | .number')})

for pr in ${matches[@]}; do
  gh pr close "$pr" --repo {owner}/{repo}
done
```

After closing, report:

- How many PRs were closed
- Which PR numbers were closed
- Whether the prefix included variants like `[Migrate-Check-TypeSpec-Mitigate]` or `[Migrate-Check-Main-Branch]`

If there are no matches, do not attempt any close operations.

### 4. Prefix interpretation rules

- `[Migrate-Check-TypeSpec]` matches only titles starting with that exact bracketed token.
- `[Migrate-Check-TypeSpec` matches both `[Migrate-Check-TypeSpec]...` and `[Migrate-Check-TypeSpec-Mitigate]...` because both start with that shorter prefix.
- `[Migrate-Check-Main-Branch]` matches only titles starting with that exact bracketed token.
- `[Migrate-Check-Typespec]` is different from `[Migrate-Check-TypeSpec]` because matching is case-sensitive inside the title string.

### 5. Final response

When listing PRs, provide the matching set clearly and mention any excluded near-matches if that matters.

When closing PRs, confirm the operation completed and list the PR numbers that were closed.
