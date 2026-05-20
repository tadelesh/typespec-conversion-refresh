# Refresh process of Go SDK for services migrated to TypeSpec

## Status data

The status data is `status.csv`. Columns (in order):

| # | Column | Description |
|---|---|---|
| 1 | Service | Human-readable service name |
| 2 | ARM Namespace | e.g. `Microsoft.Storage` |
| 3 | Spec Folder | Top-level folder under `specification/` in the specs repo |
| 4 | Go | Status — see "Terminal state" |
| 5 | tspconfig | Relative path to `tspconfig.yaml` in specs repo |
| 6 | SpecApiVersion | First conversion API version (YYYY-MM-DD or `multiple-service`) |
| 7 | SdkFolder | e.g. `sdk/resourcemanager/storage/armstorage` |
| 8 | SdkApiVersion | API version baked into the existing on-disk SDK |
| 9 | SdkPr | URL of the Go SDK refresh PR (output of step 8) |
| 10 | SwaggerCommit | Spec-repo commit used in step 9 |
| 11 | SwaggerTag | Readme tag used in step 9 (e.g. `package-2023-12-01`) |
| 12 | SdkChangelog | URL of the swagger-generated `CHANGELOG.md` produced in step 9 |
| 13 | Fix PR | URL of the spec-repo client-customization PR opened in step 11 |
| 14 | Comment | Free-form notes |

## Required repositories

- [Azure/azure-sdk-for-go](https://github.com/Azure/azure-sdk-for-go), cloned at "../azure-sdk-for-go"
  Call this folder "sdk repo" for short.
- [Azure/azure-rest-api-specs](https://github.com/Azure/azure-rest-api-specs), cloned at "../azure-rest-api-specs"
  Call this folder "specs repo" for short.

## Terminal state

Following values in "Go" column is a terminal state. Skip process on these rows.

- **Done**
  We've refreshed the SDK release from TypeSpec.
- **NoSpec**
  No TypeSpec source found. Nothing we can do.
- **ManualReview**
  There are some problems we cannot automatically handle. Manual review is needed.

## Process

For each row in the status data, we will go through below steps.

1. Check status and skip all the following steps if the status is in terminal state.

2. Find TypeSpec config file and update the "tspconfig" column

If "tspconfig" column is existing, it means we have already found the config file for this service. We can skip to next step.

Else, search for `tspconfig.yaml` under `specification/{SpecFolder}/`. In order of preference:

1. `specification/{SpecFolder}/**/resource-manager/**/tspconfig.yaml`
2. `specification/{SpecFolder}/**/*.Management/**/tspconfig.yaml`
3. Any `tspconfig.yaml` under `specification/{SpecFolder}/`.

If multiple candidates match, prefer the one whose ARM namespace (`name:` in the file or the parent folder name) matches the row's "ARM Namespace" column. If still ambiguous (e.g., shared root folder hosting multiple services such as `containerregistry`, `containerservice`, `azurestackhci`, or `Microsoft.Resources` sub-services like `bicep`/`deployments`/`deploymentStacks`/`resources`/`subscriptions`), pick the candidate whose folder name best matches the row's "Service" / sub-service name.

Add the relative path (from specs repo) to the "tspconfig" column.

If no "tspconfig.yaml" is found, add "NoSpec" to "Go" column, and skip all the following steps for this row.

3. Find generated SDK folder and update the "SdkFolder" column

If "SdkFolder" column is existing, it means we have already found the generated SDK folder for this service. We can skip to next step.

Else, read the `tspconfig.yaml` file found in last step, check the `options."@azure-tools/typespec-go".module` property. Resolve any variable references (typically `{service-dir}` and `{package-dir}` which are defined as siblings in the same `options."@azure-tools/typespec-go"` block, or `service-dir` at the top level). Strip the `github.com/Azure/azure-sdk-for-go/` prefix if present, and drop any trailing major-version suffix (e.g. `/v2`).

Extract folder `sdk/resourcemanager/**/**` according to the module path and put it in "SdkFolder" column.

Note: the SDK folder is not always derivable from the spec folder name. For example, `databoxedge` → `sdk/resourcemanager/databoxedge/armdataboxedge`, `management/Microsoft.Management/ServiceGroups` → `sdk/resourcemanager/management/armmanagement`, `resources/.../databoundaries` → `sdk/resourcemanager/databoundaries/armdataboundaries`. Always read the `module` property; do not guess from folder names.

4. Check if the Go SDK is already generated from TypeSpec

If there is a `tsp-location.yaml` file under the `SdkFolder` path, it means the Go SDK has already been released from TypeSpec. Add "AlreadyTypeSpec" to "Go" column.
Then check if there is a tag existed with `sdk/resourcemanager/{service}/{armmodule}/v{<latest_version>}`. `{service}` and `{armmodule}` can be extracted from "SdkFolder" column. `{latest_version}` can be extracted from the latest version in `CHANGELOG.md` file.
If such a tag exists, add "Done" to "Go" column. Skip all the following steps for this row.

5. Find the first conversion API version for the service, and add it to "SpecApiVersion" column

If "SpecApiVersion" column is existing, it means we have already found the API version for this service. We can skip to next step.

Else, check the folder of "tspconfig" (do not include nested folders) column in specs repo.

Find the first item in `Version` enum. Add it to the "SpecApiVersion" column.

The `Version` enum can be found as the enum specified in `@versioned` decorator. This usually in "main.tsp" file.

```ts
@versioned(Versions)
namespace Microsoft.NetApp;

enum Versions {
  ...
}
```

If `Version` enum does not exist, it means this is a multi-service package. Add "multiple-service" to "SpecApiVersion" column.

6. Find SDK API version and update the "SdkApiVersion" column

If "SdkApiVersion" column is existing, it means we have already found the SDK API version for this service. We can skip to next step.

Else, check the folder of "SdkFolder" column in sdk repo. The API version could be found in the comment of any operation with formats like `// Generated from API version <api-version>` in `xxx_client.go` file. Add the API version to "SdkApiVersion" column.

7. Compare `SpecApiVersion` with `SdkApiVersion`

Compare the API version in "SpecApiVersion" column with "SdkApiVersion" column. If they are not same, add "VersionNotEqual" to "Go" column.

8. Generate SDK via pipeline

If "SdkPr" column has a PR link, it means we have already generated the SDK. We can skip to next step.

Else, trigger pipeline https://dev.azure.com/azure-sdk/internal/_build?definitionId=7426 via the Azure DevOps Pipelines REST API. Use the token from Azure CLI to call the REST API of the "dev.azure.com" endpoint (preferably using `az rest` and let Azure CLI handle the token, with `Content-Type=application/json` via `--header`).

- Endpoint: `POST https://dev.azure.com/azure-sdk/internal/_apis/pipelines/7426/runs?api-version=7.1-preview.1`
- Resource id for `az rest --resource`: `499b84ac-1321-427f-aa17-267ca6975798`
- Body shape:

```json
{
  "resources": { "repositories": { "self": { "refName": "refs/heads/main" } } },
  "templateParameters": {
    "SdkReleaseType": "beta",
    "CreatePullRequest": true,
    "ApiVersion": "<SpecApiVersion>",
    "ConfigPath": "<tspconfig path>",
    "ConfigType": "TypeSpec"
  }
}
```

- `ApiVersion` should be the YYYY-MM-DD form of `SpecApiVersion` (or empty when `SpecApiVersion` is "multiple-service").
- `ConfigPath` is the value from the "tspconfig" column.
- Pass the JSON via `--body "@<file>"` rather than inline to avoid quoting issues.

Poll `GET https://dev.azure.com/azure-sdk/internal/_apis/pipelines/7426/runs/{runId}?api-version=7.1-preview.1` every 60–90 s until `state == "completed"`. Typical run takes 5–10 minutes.

Once complete, find the new PR on https://github.com/Azure/azure-sdk-for-go/pulls (search title `"[AutoPR sdk-{SdkFolder}]*-{runId}"`), rename the title prefix `[AutoPR …]` → `[Refresh …]` (`gh pr edit <num> --repo Azure/azure-sdk-for-go --title …`), and put the PR URL in "SdkPr".

9. Generate SDK with Swagger for "VersionNotEqual" services

If "SdkChangelog" column has a link, it means we have already generated SDK with Swagger spec. We can skip to next step.

If "SpecApiVersion" is "multiple-service", it means we cannot determine a single API version for the service. We can skip SDK generation with Swagger and directly go to step 10 to check the changelog.

For the services with "VersionNotEqual" status, we need to generate SDK with Swagger spec.

Follow these steps to generate SDK with Swagger spec:
1) In the specs repo, find the **earliest commit that introduced TypeSpec for this service**, considering possible later folder reorganizations:

```powershell
git log --reverse --diff-filter=A --format=%H -- `
  "specification/{SpecFolder}/**/*.tsp" `
  "specification/{SpecFolder}/**/tspconfig.yaml" | Select-Object -First 1
```

Do **not** rely on `git log --follow tspconfig.yaml` — that returns the most recent folder-refactor commit (e.g. "refactor(svc): migrate to unified folder structure") when the file was later moved, and its parent is **not** the correct swagger baseline.

The correct `SpecCommit` is the commit immediately before (parent of) the original TypeSpec migration commit on `specification/{SpecFolder}`:

```powershell
$tspMigration = git log --reverse --diff-filter=A --format=%H -- `
  "specification/{SpecFolder}/**/*.tsp" `
  "specification/{SpecFolder}/**/tspconfig.yaml" | Select-Object -First 1
git log "$tspMigration^" -n 1 --format=%H -- specification/{SpecFolder}
```

Sanity check: the commit date of `SpecCommit` should be **earlier** than the TypeSpec migration. If `SpecCommit` is a generic readme update unrelated to the service (e.g. a bulk emitter config change), that is fine — it just represents the swagger state at the time conversion started.

**Exception — SpecApiVersion postdates migration:** If the desired `SwaggerTag` / `SpecApiVersion` API version did not yet exist at the migration-time commit (e.g., the service migrated to TypeSpec in 2024 but `SpecApiVersion` is `2025-07-01-preview`), the parent-of-first-tsp commit will not contain the needed swagger files. In that case, pick the **most recent commit on `main` that still contains the required `specification/{SpecFolder}/resource-manager/**/<api-version>/*.json` files**. Verify with `git ls-tree -r --name-only <commit> -- specification/{SpecFolder} | Select-String "{api-version}"`. Document the reason in the row's Comment column.

Write the resolved commit hash into the "SpecCommit" column.

2) Based on this commit ID, check the first found `specification/{SpecFolder}/resource-manager/**/readme.md` file. Use "SpecApiVersion" to find pattern like `### Tag: package-{SpecApiVersion}` and extract the whole tag into the "SwaggerTag" column. If there is no such pattern, use the latest tag in the `readme.md` file.
3) Go to the folder of "SdkFolder" column in sdk repo, edit the `autorest.md` file: add or update the tag in the yaml to `tag: {SwaggerTag}`. Also update **both** `require:` URLs in `autorest.md` to point to the SpecCommit and the correct readme.md path found in step 2). The readme.md may be in a nested subfolder (e.g., `resource-manager/Microsoft.X/SubFolder/readme.md`) rather than at the `resource-manager/` root.

**Important:** Existing `require:` URLs in `autorest.md` often contain stale commits from the last AutoRest run (years before TypeSpec migration). Always overwrite both URLs — do not assume the existing values are correct, even if they look plausible.

4) Go to the sdk repo root folder, run `generator release-v2 c:/w/azure-sdk-for-go  c:/w/azure-rest-api-specs {service} {armservice} --skip-generate-example --spec-commit-hash={SpecCommit}`. `{service}` and `{armservice}` could be extracted from "SdkFolder" column. The `--spec-commit-hash` argument is **required** — it is the authoritative input for swagger generation. Verify after the run that the resulting `autorest.md` `require:` URLs contain `{SpecCommit}`; if they do not, the generator did not pick up the override and the run must be repeated. Push the new created branch to remote. Put the link of the `CHANGELOG.md` file from this new branch to "SdkChangelog" column.
5) Leave a comment in the "SdkPr" with the link of "SdkChangelog" column.
6) After all, you need to go back to main for the sdk repo.

10. Check changelog against breaking changes guide

From the PR in "SdkPr" column, extract the latest version's changelog from the `CHANGELOG.md` file in the SDK folder.

**If versions are equal (step 9 was skipped):** review all changelog items from the TypeSpec-generated changelog in the PR.

**If versions are not equal (step 9 was executed):** also extract the latest version's changelog from the swagger-generated `CHANGELOG.md` in step 9. Compare the two changelogs and compute the **diff** — only the items that appear in the TypeSpec changelog but NOT in the swagger changelog need to be reviewed. Items present in both changelogs are caused by the API version difference, not the TypeSpec conversion, and can be ignored.

Read the [breaking changes guide](https://github.com/Azure/azure-sdk-for-go/blob/main/documentation/development/breaking-changes/sdk-breaking-changes-guide-migration.md) (`documentation/development/breaking-changes/sdk-breaking-changes-guide-migration.md` in the sdk repo) and use it to classify each changelog item (or diff item for version-not-equal) into one of:

- **Resolvable** — can be fixed with TypeSpec customizations (e.g. `@@clientName`, `@@clientLocation`, `@@alternateType` in `client.tsp`, or emitter options in `tspconfig.yaml`). Add the resolvable items and their suggested fixes to the "Comment" column.
- **Acceptable** — expected changes that ship in a new major version. No fix needed.

If there are any resolvable items or any items that cannot be classified, add "ManualReview" to "Go" column for this row.
If all items are acceptable, update "Go" column to "Done".

Common acceptable patterns observed from previous refresh runs (no customization needed):

- `ARMBaseModel` (or other generated base resource) struct removed and its fields (`ID`, `Name`, `Type`, `SystemData`) inlined into the resource struct. This is the standard TypeSpec emitter pattern.
- New `SystemData` field appearing on resource structs after `ARMBaseModel` removal — compensation for the inlining above.
- Structs/operations only present in older swagger API versions that are no longer in the spec for the new API version (an API-version delta, not a TypeSpec conversion artifact).
- Pure additive features (new fields, new properties on existing structs).

11. Apply client customizations and open Fix PR in spec repo

If step 10 produced one or more **Resolvable** items, apply the suggested TypeSpec customizations in the specs repo:

1) In the specs repo, create a new branch off `main` named `refresh/{SpecFolder}-customization` (or similar — keep it scoped to a single tspconfig folder so the fix PR is small).
2) Edit `client.tsp` in the same folder as the tspconfig (create it if it does not exist). Apply customizations per the [breaking changes guide](https://github.com/Azure/azure-sdk-for-go/blob/main/documentation/development/breaking-changes/sdk-breaking-changes-guide-migration.md). Typical fixes:
   - `@@clientName(Original, "New", "go")` — rename a model, operation, or parameter for the Go client only.
   - `@@clientLocation(Operation, NewInterface, "go")` — move an operation to a different client/interface.
   - `@@alternateType(Property, NewType, "go")` — change a property's type in the Go client.
   - Adjust `options."@azure-tools/typespec-go"` in `tspconfig.yaml` (e.g. `module`, `head-as-boolean`, `single-client`) for emitter-level fixes.
3) Commit and push the branch, then open a PR against `Azure/azure-rest-api-specs:main`. Title format: `Refresh client customization for {Service}`.
4) Put the PR URL in the **"Fix PR"** column.
5) After the Fix PR is merged, re-run step 8 (regenerate via pipeline) and step 10 (re-classify). When all items are acceptable, set "Go" to "Done".

If step 10 produced no resolvable items, leave the "Fix PR" column empty.

If any problem happens in above steps, add "Error" to "Go" column and summarize the problem in "Comment" column.

## Status verification

When auditing existing rows (re-deriving "Go" from on-disk state), apply the strict rule from step 4 against both the spec repo and the SDK repo at HEAD on `main`:

- `Done` ⇔ `tsp-location.yaml` exists under `SdkFolder` **AND** tag `sdk/resourcemanager/{service}/{armmodule}/v{latest_CHANGELOG_version}` exists in the SDK repo.
- `AlreadyTypeSpec` ⇔ `tsp-location.yaml` exists, no matching tag.
- `NoSpec` ⇔ no `tspconfig.yaml` anywhere under `specification/{SpecFolder}/`.
- `ManualReview` is preserved only when "SdkPr" is set (a refresh attempt was made and required manual analysis).
- Otherwise the row is "needs processing" (empty "Go").

Common false-positive sources to watch for:

- An open/unmerged refresh PR makes the row appear "not Done" on `main` even though step 10 was completed. Strict re-audit will demote it to empty; this is correct — the row will return to Done after the PR merges and a release tag is published.
- Renamed spec folders (e.g. `web/certificationregistration` → `certificateregistration`, `web/domainregistration` → `domainregistration`). When matching against external sources keep both names as aliases.
- A single spec folder shared by multiple services (e.g. `containerregistry`, `azurestackhci`, `containerservice` each have multiple sub-services). When filtering rows by spec folder, do not assume one row per folder.