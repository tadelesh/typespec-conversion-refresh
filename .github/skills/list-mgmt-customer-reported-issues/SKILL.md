# Skill: List Mgmt + Customer-Reported Open Issues

## Description

List open issues in Azure/azure-sdk-for-go that have both `Mgmt` and `customer-reported` labels, including title, URL, assignees, Issue Type, Status, and optionally all comments in chronological order, in CSV or Excel format.

## Input

| Parameter        | Type    | Default                                                     | Description                               |
| ---------------- | ------- | ----------------------------------------------------------- | ----------------------------------------- |
| repository       | string  | Azure/azure-sdk-for-go                                      | Repository to scan                        |
| include_comments | boolean | false                                                       | Whether to include comments in the output |
| save_to_excel    | boolean | true                                                        | Whether to save output as Excel file      |
| output_path      | string  | azure-sdk-for-go-mgmt-customer-reported-open-issues-02.xlsx | Output file path                          |

## Output

| Column     | Description                                                            |
| ---------- | ---------------------------------------------------------------------- |
| Issue #    | Issue number                                                           |
| Title      | Issue title                                                            |
| URL        | GitHub issue URL                                                       |
| Assignees  | Assigned GitHub users                                                  |
| Issue Type | Classification of the issue (taxonomy of work)                         |
| Status     | Where the issue is in handling (based on state, labels, comments, PRs) |
|            |

## Issue Type

### Allowed values

- Usage Question
- Code Gen Issue
- Service Issue
- Spec Issue
- Azure Core Issue
- Serialization Issue
- Feature Request

### Decision order

If the condition is met, there's no need to continue the evaluation.

1. **Feature Request override**
   - Label contains `feature-request`
   - Title contains `feature` or `feature request`
   - Body asks for a new field
   - Result: Issue Type = Feature Request; Status = Need to release new version

2. **Missing field path**
   - Body mentions a missing field
   - Comments:
     - Field added in a version => Resolved
     - Latest comment asks author => Need author feedback
     - Latest comment says investigating => Investigating
     - Else => Need to release new version
   - Result: Issue Type = Spec Issue

3. **Bug + bot-only comments + Service Attention**
   - Title contains bug
   - All comments are bots
   - Labels include Service Attention
   - Result: Issue Type = Service Issue; Status = Need Service Attention

4. **Latest maintainer comment classification**
   - Spec-related => Spec Issue
   - Service-related => Service Issue
   - Codegen => Code Gen Issue
   - Serialization => Serialization Issue
   - Azure Core => Azure Core Issue
   - How-to => Usage Question
   - Feature => Feature Request

5. **Label-based classification**
   - Feature/enhancement => Feature Request
   - Documentation/question => Usage Question
   - Bug + serialization => Serialization Issue
   - Bug + codegen => Code Gen Issue
   - Bug + runtime/service => Service Issue

6. **Title + body + label keyword fallback**
   - Serialization => Serialization Issue
   - Azure Core => Azure Core Issue
   - Codegen/fakes/regex => Code Gen Issue
   - Spec mismatch => Spec Issue
   - Runtime/service => Service Issue
   - New API request => Feature Request
   - How-to questions => Usage Question
   - Default => Usage Question

## Status

### Allowed values

- Todo
- In Progress
- Resolved
- Need author feedback
- Need Service Attention
- Suggested user to open tickets to related service to fix specs
- Investigating
- Need to release new version

### Decision order

If the condition is met, there's no need to continue the evaluation.

1. **Feature Request override** => Need to release new version
2. **Missing field path** => Based on comments
3. **Service Attention bug** => Need Service Attention
4. **Closed or duplicate** => Resolved
5. **Labels**
   - Customer response => Need author feedback
   - Service Attention => Need Service Attention
6. **Comments (latest human comment)**
   - Asks for info => Need author feedback
   - Suggest support ticket => Suggested user to open tickets to related service to fix specs
   - Investigating => Investigating
   - PR/release in progress => In Progress
   - Released in next version => Need to release new version
7. **Title/body only** => Todo or Investigating
8. **Default** => Todo

## Steps

1. **Query matching open issues**

```
gh api --paginate \
  'search/issues?q=repo:Azure/azure-sdk-for-go+is:issue+is:open+label:Mgmt+label:customer-reported&per_page=100' \
  --jq '.items[] | [.number, .title, .html_url] | @tsv'
```

2. **Fetch issue details and comments**

```
gh issue view <N> --json number,title,state,labels,assignees,body,url,comments,closedAt
gh issue view <N> --comments --paginate
```

3. **Classify Issue Type and Status**
   - Apply decision rules in the order above: Feature Request => Missing Field => Maintainer => Labels => Keywords => Default.

## Output

- CSV or Excel with headers: Issue #, Title, URL, Assignees, Issue Type, Status
- Include comments if include_comments=true

- add summary statistics in Excel for triage, e.g. count of issues by type and status from the column H and line 2 to line n, and pivot table for issue type and status.

## Notes

- Labels matched case-insensitively; preserve canonical Issue Type/Status in output.
- Latest maintainer comment overrides labels.
- Feature Request and Missing Field paths have highest priority.
- Optional comments include timestamp, author, text in chronological order.
- Can add summary statistics in Excel for triage.
