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
- `Issue Type` — **what** the thread is about (taxonomy of work); do not use this column for triage workflow.
- `Status` — **where** the issue is in handling (who waits on whom, or done); derive from `state`, **labels**, **timeline**, and **linked PRs**, not from issue type alone.

<!-- - `Comments` (single column, ordered by time, optional, and save the comments line by line with timestamp and author) -->

---

## Issue Type (accurate classification)

**Principle:** Prefer **explicit GitHub labels** on the issue when they map cleanly; otherwise use **title + body + first maintainer reply** (avoid reclassifying from stale side-thread keywords). `Issue Type` must be **one** value from the list below.

### Allowed values

- `Usage Question`
- `Code Gen Issue`
- `Service Issue`
- `Spec Issue`
- `Azure Core Issue`
- `Serialization Issue`
- `Feature Request`

### Step A — Label-first hints (case-insensitive label match)

When any of these labels is present on the issue, set **Issue Type** as follows (first match in this table wins):

| Label contains (examples)                                                                                                                        | Issue Type                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| `feature`, `feature-request`, `enhancement`                                                                                                      | `Feature Request`                                                  |
| `bug` **and** (title/body mentions generator, `autorest`, `codegen`, `fakes`, `mock`, wrong client surface)                                      | `Code Gen Issue`                                                   |
| `bug` **and** (runtime marshal/unmarshal, JSON casing, polymorphic discriminant)                                                                 | `Serialization Issue`                                              |
| `bug` **and** (REST returns wrong/absent data vs portal; nil field “in service” but spec says optional/required ambiguity → still often service) | Prefer `Service Issue` unless the thread proves spec text is wrong |
| `documentation`, `question` **and** no concrete defect                                                                                           | `Usage Question`                                                   |

If **no** label from the table applies, go to Step B.

### Step B — Keyword / content rules (use **title + body**; use comments only to disambiguate)

Apply rules in **strict precedence** (stop at first match):

1. **Serialization Issue** — `unmarshal`, `marshal`, `deserialize`, `serde`, `json`, `case sensitive`, `discriminator`, `AdditionalProperties`, `omitempty`, wrong type on wire vs model.
2. **Azure Core Issue** — `azcore`, `policy`, `retry`, `transport`, `pipeline`, `credential`, `BearerTokenPolicy`, `logging` in SDK core (not service REST shape).
3. **Code Gen Issue** — wrong method signature, wrong API version in client, missing operation that **exists in spec**, fake/record/replay tied to generated client, generator bug, wrong polymorphic type **generated** from spec.
4. **Spec Issue** — contract mismatch **with evidence** that **TypeSpec/OpenAPI/Swagger** text is wrong or inconsistent (e.g. maintainer says “spec fix needed”, links to **spec PR** or `specs` repo issue); not merely “API returns null” without spec proof.Or comments mentions like `Our SDK is auto-generated from service spec` or `It looks like the spec definition is wrong.`
5. **Service Issue** — runtime/service returns wrong data, throttling, RBAC on service, long-running operation state, **nil** fields where service behavior contradicts user expectation but **spec is not shown as wrong** in thread.Or comments indicates user was advised to open Azure support ticket or contact RP team. Or maintainer says “service team needs to fix” without spec link. Or user says “RP told me to ask here” without spec link. Or issue closed as “service issue” without spec link. Or no spec link but thread shows user confusion about service behavior that is not clearly documented in spec (e.g. “field is optional but service requires it” without spec proof, or “service returns 400 but I think it should be 404” without spec proof). Or title contains `Service Issue`, `Service Bug`, `Service Problem`, `Throttling`, `RBAC`, `LRO`, `Long-running operation`, `Nil field`, `Unexpected null`, `Unexpected empty`, `Unexpected 400`, `Unexpected 500`, `Wrong data from service`, `Service returns X not Y`, `Service behaves unexpectedly` etc. Or labels contain `service`, `service-issue`, `needs-service-attention` but no spec labels and no concrete defect described.
6. **Feature Request** — asks for new capability, new API version support, new resource type, new client method **not present in spec** / “please add X” without a bug on existing contract.Or title contains `Feature Request`, `Please add`, `Support for X`, `Missing API`, `New API for X`, `Feature` etc.Or labels contain `feature`, `feature-request`, `enhancement` but no bug labels and no concrete defect described.
7. **Usage Question** — how to call API, sample code, idempotent pattern, **no** confirmed defect after reading body.

### Disambiguation (common mistakes to avoid)

- **Service vs Spec:** If maintainers say “service bug” or “contact RP team” → `Service Issue`. If they say “spec needs update” with spec link → `Spec Issue`. If unclear → `Service Issue` (default) and set Status to reflect investigation.
- **Code Gen vs Serialization:** Generation/surface of API **vs** encoding of already-correct surface → Serialization wins when wire JSON shape is the topic.
- **Feature Request vs Usage Question:** “How do I …?” with existing API → Usage Question. “Please support …” new surface → Feature Request.

---

## Status (accurate classification)

**Principle:** `Status` is **workflow**, not issue category. **Never** set status from “feature request” wording alone — that belongs in **Issue Type**. Use **`state`**, **labels**, **last meaningful maintainer comment**, and **linked PRs**.

### Allowed values

- `Todo`
- `In Progress`
- `Resolved`
- `Need author feedback`
- `Need Service Attention`
- `Suggested user to open tickets to related service to fix specs`
- `Investigating`
- `Need to release new version`

### Step 1 — Hard signals (highest priority)

| Signal                                                                                                                              | Status                                                           |
| ----------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `state` is **closed** (or locked as resolved duplicate)                                                                             | `Resolved`                                                       |
| Label `need customer response`, `needs author feedback`, `waiting-for-customer`, `more-information-needed` (repo-specific variants) | `Need author feedback`                                           |
| Label `Service Attention`, `service attention`, `needs-service-attention`                                                           | `Need Service Attention`                                         |
| Maintainer comment explicitly asks user to open **Azure support** / **service team** ticket / ICM and **no** SDK PR linked          | `Suggested user to open tickets to related service to fix specs` |
| **Open** linked PR in `Azure/azure-sdk-for-go` (or comment “fix in PR #…”) fixing this issue                                        | `In Progress`                                                    |
| Maintainer states **next SDK release** will contain fix / cherry-pick merged to release branch                                      | `Need to release new version`                                    |

### Step 2 — Timeline (when Step 1 did not decide)

- Last comment is from **maintainer/bot** asking the **author** for logs, repro, API version, or packet capture → `Need author feedback`.
- Last comment is from **author** with requested info; no maintainer follow-up in **≥ 14 days** (configurable) → `Todo` or `Need Service Attention` if thread says waiting on service — prefer **`Todo`** unless label says service.
- Thread shows active investigation (repro confirmed, bisect, cross-link to service issue) but no PR → `Investigating`.
- Issue assigned and recent maintainer activity without waiting on user → `In Progress`.

### Step 3 — Default

- If open, not waiting on user, no service label, no PR → `Investigating` (not `Todo`) when there was maintainer engagement in last 30 days; otherwise `Todo`.

### Mistakes to avoid

- Do **not** map “feature request” / “missing API” text to **`Need to release new version`** unless a maintainer **committed** to a release or a release PR exists. Otherwise keep **`Investigating`** or **`Todo`** and put “missing capability” under **Issue Type** = `Feature Request`.
- **`Resolved`** only when closed **or** duplicate with clear resolution pointer — not when discussion merely stalled.

---

## Recommended `gh` data (reduce guesswork)

Always fetch structured fields so labels and state are authoritative:

```sh
gh issue view <N> --repo Azure/azure-sdk-for-go \
  --json number,title,state,labels,assignees,body,url,comments,closedAt
```

For linked PRs (when status might be In Progress):

```sh
gh pr list --repo Azure/azure-sdk-for-go --search "<N>" --state open --json number,title,url
```

(Adjust search to `is:linked issue:<N>` patterns supported by your `gh` version, or parse timeline events if needed.)

---

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
- labels and `state` (for status)
- body + comment timeline (created order)

Example:

```sh
gh api repos/Azure/azure-sdk-for-go/issues/<number>
gh api repos/Azure/azure-sdk-for-go/issues/<number>/comments --paginate
```

Prefer `gh issue view <N> --json ...` for a single JSON payload.

### 3. Classify Issue Type

1. Run **Issue Type → Step A (labels)**.
2. If unset, run **Step B** in **strict precedence** order.
3. If still ambiguous, set the **higher-precedence** type from Step B and add a short internal note in chat (not in CSV) explaining the tie-break.

### 4. Classify Status

1. Run **Status → Step 1** in table order.
2. Else **Step 2** (timeline + assignment).
3. Else **Step 3** default.

Re-read **last 2–3 non-bot comments** before finalizing `Need author feedback` vs `Investigating`.

### 5. Output

Provide either:

- Markdown table in chat, or
- Excel file with header:

```excel
Issue #,Title,URL,Assignees,Issue Type,Status
```

Add summary statistics (e.g. count by issue type/status) if output to Excel.

## Notes

- Match labels **case-insensitively** as GitHub normalizes labels, but preserve **canonical output form** in the spreadsheet (Title Case as in allowed lists).
- Keep CSV cells quoted when they contain commas/newlines.
- If no issues match, return an empty result with header only.
- **Issue Type** and **Status** are independent: never copy the same keyword rule to both columns; when in doubt, prefer **labels + state** for Status and **labels + title/body precedence** for Issue Type.
