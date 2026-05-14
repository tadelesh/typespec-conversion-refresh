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
- `Status` — **where** the issue is in handling (who waits on whom, or done); derive from **`state`**, **labels**, **comments (who said what last)**, and **linked PRs** — not from Issue Type alone.

<!-- - `Comments` (single column, ordered by time, optional, and save the comments line by line with timestamp and author) -->

---

## Classification inputs (use all of these)

For **every** issue, base both **Issue Type** and **Status** on the same evidence bundle:

| Source | What to read | Why it matters |
|--------|----------------|----------------|
| **Labels** | Full current label set on the issue (case-insensitive match) | Triage intent from repo owners; often authoritative for **Status** |
| **Title** | Issue title only | Short intent; good for **Issue Type** hint; can mislead if body clarifies |
| **Body** | First post / description | Fuller repro and keywords; pair with title as **“header text”** |
| **Comments** | All comments in **chronological order** | Thread evolution; **last substantive maintainer/collaborator comment** often overrides early guesses |

**Comment hygiene**

- Treat **repository collaborators / Microsoft / Azure SDK maintainers** as “maintainer voice” when they state root cause or next step.
- **Deprioritize** pure automation: `github-actions`, `dependabot`, generic `bot` accounts **unless** the comment only adds a label reference you already verified on the issue.
- If **title** and **body** disagree, trust **body** for technical detail; if **early comments** and **latest maintainer comment** disagree, trust the **latest maintainer comment** for Issue Type **unless** labels explicitly encode a different triage (then prefer label + maintainer: see conflict rules below).

---

## Issue Type (title + labels + comments + body)

**Principle:** Pick **exactly one** allowed value. Combine **labels** + **title/body keywords** for a first guess, then **refine using comments** (especially explicit maintainer statements). `Issue Type` is **not** workflow.

### Allowed values

- `Usage Question`
- `Code Gen Issue`
- `Service Issue`
- `Spec Issue`
- `Azure Core Issue`
- `Serialization Issue`
- `Feature Request`

### Decision order (follow in sequence)

**Step 1 — Explicit maintainer classification in comments (highest weight for type)**

Scan comments **newest → oldest** for the **latest** clear statement from a maintainer/collaborator that names the problem class. If found, map to Issue Type:

| Maintainer phrasing (examples) | Issue Type |
|--------------------------------|------------|
| “spec issue”, “TypeSpec”, “OpenAPI needs”, “swagger fix”, “contract in spec is wrong” | `Spec Issue` |
| “service bug”, “RP issue”, “resource provider”, “service team”, “backend returns” | `Service Issue` |
| “codegen”, “generator”, “autorest”, “incorrectly generated”, “client surface wrong vs spec” | `Code Gen Issue` |
| “serialization”, “unmarshal”, “marshal”, “JSON shape”, “discriminator”, “polymorphic” | `Serialization Issue` |
| “azcore”, “pipeline”, “retry policy”, “credential”, “transport” (SDK core) | `Azure Core Issue` |
| “how to”, “by design”, “use this API”, “sample”, no defect | `Usage Question` |
| “feature”, “enhancement”, “new API version”, “not supported yet”, “request to add” | `Feature Request` |

If Step 1 matches, **use it**. If a **label** seems to disagree, apply **Step 4** (maintainer comment outranks label unless the maintainer statement is clearly superseded by a **newer** label change explained in comments).

**Step 2 — Labels (case-insensitive)**

Apply when Step 1 did **not** yield a maintainer verdict, **or** only to **break ties**:

| Condition on labels | Issue Type |
|---------------------|------------|
| `feature`, `feature-request`, `enhancement` (and not contradicted by maintainer “this is a bug in spec/service”) | `Feature Request` |
| `documentation` / `question` **and** header text has no concrete defect | `Usage Question` |
| `bug` + header/comments show marshal/JSON/discriminator | `Serialization Issue` |
| `bug` + header/comments show generator/autorest/codegen/fakes | `Code Gen Issue` |
| `bug` + service/runtime symptoms, no spec-wrong evidence | `Service Issue` |

**Step 3 — Header text (title + body) keyword precedence**

Use only if Steps 1–2 insufficient. Apply **first match wins** in this order:

1. **Serialization Issue** — `unmarshal`, `marshal`, `deserialize`, `serde`, `json`, `case sensitive`, `discriminator`, `AdditionalProperties`, `omitempty`, wire vs model mismatch.
2. **Azure Core Issue** — `azcore`, `policy`, `retry`, `transport`, `pipeline`, `credential`, `BearerTokenPolicy`, `logging` (core SDK, not REST contract).
3. **Code Gen Issue** — wrong generated surface, wrong API version in client, missing op that **exists in spec**, fakes/mocks tied to codegen.
4. **Spec Issue** — OpenAPI/TypeSpec/Swagger mismatch **with link or maintainer confirmation** in thread.
5. **Service Issue** — runtime wrong data, throttling, RBAC, LRO state; nil fields without proven spec error.
6. **Feature Request** — new capability / new API surface requested.
7. **Usage Question** — how-to only, no confirmed defect.

**Step 4 — Conflicts (label vs maintainer comment vs title)**

Resolve in this **strict priority**:

1. Latest **maintainer comment** that explicitly classifies root cause (Step 1 table).
2. **Labels** that encode triage (`Spec Issue` / service attention / question) **if** no newer maintainer comment contradicts them.
3. **Header text** (body over title) keyword precedence (Step 3).

*Example:* Title says “Bug in SDK” but maintainer writes “spec defines the wrong type” → **`Spec Issue`**, not Code Gen.

### Disambiguation (common mistakes)

- **Service vs Spec:** Maintainer says contact RP / service → `Service Issue`. Maintainer links spec PR or says spec wrong → `Spec Issue`. Only title says “wrong response” → default `Service Issue` until a maintainer specifies spec.
- **Code Gen vs Serialization:** Wire encoding vs wrong generated method list → Serialization when JSON/model mapping is the topic.
- **Feature vs Usage:** Title “How to …?” + body shows misunderstanding → `Usage Question`. “Please add client for …” → `Feature Request`.

---

## Status (labels + comments + title/body for context only)

**Principle:** `Status` answers **who is blocked** and **whether work shipped**. **Do not** infer Status from Issue Type (e.g. Feature Request ≠ Need to release). Use **`state`**, **labels**, then **comment thread** (especially **last human direction**).

### Allowed values

- `Todo`
- `In Progress`
- `Resolved`
- `Need author feedback`
- `Need Service Attention`
- `Suggested user to open tickets to related service to fix specs`
- `Investigating`
- `Need to release new version`

### Decision order (follow in sequence)

**Step 1 — `state` and closure**

| Signal | Status |
|--------|--------|
| `state` is **closed**, or duplicate with resolution noted | `Resolved` |

**Step 2 — Labels (case-insensitive)**

If issue is **open**, apply **first match** in this table:

| Label pattern (examples) | Status |
|--------------------------|--------|
| `need customer response`, `needs author feedback`, `waiting-for-customer`, `more-information-needed`, `needs-information` (if used in repo) | `Need author feedback` |
| `Service Attention`, `service attention`, `needs-service-attention` | `Need Service Attention` |

Issues matched by this skill already carry `customer-reported`; **do not** treat that label alone as “waiting on customer” — pair it with a **feedback** / **information** label or an **open maintainer ask** in comments.

**Step 3 — Comments (newest substantive human comment)**

Read **from newest to oldest**; skip pure bots unless they only mirror labels you already applied in Step 2. Prefer **maintainer/collaborator** voice.

| Comment pattern (examples) | Status |
|------------------------------|--------|
| “please provide”, “could you share”, “need repro”, “API version”, “full request/response”, “HAR”, “correlation id”, “?” directed at opener with no reply yet expected | `Need author feedback` |
| “please open a support ticket”, “contact the service team”, “file with RP”, “ICM”, “Azure support” (and **no** SDK fix PR linked) | `Suggested user to open tickets to related service to fix specs` |
| “we are investigating”, “looking into”, “reproduced”, “confirmed”, internal thread link, no PR yet | `Investigating` |
| “fix in PR #…”, “opened PR”, linked `azure-sdk-for-go` PR, cherry-pick | `In Progress` |
| “will be in next release”, “released in v…”, “package version … fixes”, merge to release branch stated | `Need to release new version` |
| Maintainer asked for info; **opener replied** with data; **no** maintainer follow-up for **≥ 14 days** | `Todo` (queue) **or** `Need Service Attention` if thread says blocked on service — use label from Step 2 if present |
| Assigned + recent maintainer progress, not waiting on customer | `In Progress` |

**Step 4 — Title/body only when comments are empty or generic**

If there are **no** useful comments yet:

- Body contains only questions and labels include `question` → `Need author feedback` **or** `Investigating` depending on whether a maintainer has engaged (if **no** maintainer comment at all → `Todo`).
- Do **not** set `Resolved`, `Need to release new version`, or `Suggested user…` from title/body alone — wait for maintainer comment or label.

**Step 5 — Default for open issues**

- Maintainer engaged recently, no blocker identified → `Investigating`.
- No maintainer comment yet → `Todo`.

### Mistakes to avoid

- **Need to release new version:** only when a maintainer **states** a release / version **or** a shipping PR exists — **not** because the title mentions “new version” or Issue Type is Feature Request.
- **Need author feedback:** requires **outstanding ask** to the author (by comment or label). If the **author** spoke last and supplied requested info → **not** this status; use `Todo` / `Investigating` / `In Progress` per Step 3.
- **Resolved:** closed state only (or duplicate with clear resolution), not “no activity”.

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

1. Read **labels**, **title**, **body**, and **all comments** (chronological).
2. Run **Issue Type → Decision order**: Step 1 (maintainer phrasing in comments, newest first) → Step 2 (labels) → Step 3 (title+body keywords) → Step 4 (conflicts).
3. If still ambiguous after Step 4, pick the best fit from Step 3 and add a **one-line** rationale in chat only (not in the CSV).

### 4. Classify Status

1. Read **`state`** and **labels** first.
2. Scan **comments newest → oldest** for the **last substantive human** maintainer direction (Step 3 table).
3. Apply **Decision order** Steps 1–5; use **title/body** only when comments are empty or non-informative (Step 4).
4. Before finalizing, confirm **`Need author feedback`** matches an **open** ask to the author (comment or label), not merely because the issue is customer-reported.

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
- **Issue Type** and **Status** are independent: do not set Status from Issue Type keywords (e.g. “feature” in the title).
- **Primary signals:** for **Issue Type**, prefer **latest maintainer classification in comments**, then **labels**, then **title+body**; for **Status**, prefer **`state` + labels**, then **latest maintainer ask or PR/release language in comments**, then **title/body** only if the thread is still thin.
