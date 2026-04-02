---
name: go-sdk-validation
description: >
  Validate Go SDK generation for a swagger-to-typespec conversion.
  Accepts either a conversion PR link from Azure/azure-rest-api-specs or a tspconfig.yaml path/URL.
  Generates the SDK locally, reviews the changelog against the breaking changes guide,
  classifies breaking changes as resolvable or acceptable, applies TypeSpec customization
  fixes, and re-generates until the changelog is clean.
---

# Go SDK Validation for Swagger-to-TypeSpec Conversion

Validate Go SDK generation for a converted TypeSpec spec, review the changelog, and apply fixes.

## Input

The user provides one of the following:

- **A conversion PR link** (e.g. `https://github.com/Azure/azure-rest-api-specs/pull/12345` or `https://github.com/Azure/azure-rest-api-specs-pr/pull/27607`)
- **A tspconfig.yaml path** relative to the specs repo (e.g. `specification/network/Network.Management/tspconfig.yaml`)
- **A tspconfig.yaml URL** on GitHub (e.g. `https://github.com/Azure/azure-rest-api-specs/blob/main/specification/network/Network.Management/tspconfig.yaml`)

## Required repositories

- [Azure/azure-sdk-for-go](https://github.com/Azure/azure-sdk-for-go), cloned at `../azure-sdk-for-go` (the "sdk repo").
- **Specs repo** — one of the following, determined by the user's input:
  - [Azure/azure-rest-api-specs](https://github.com/Azure/azure-rest-api-specs), cloned at `../azure-rest-api-specs`
  - [Azure/azure-rest-api-specs-pr](https://github.com/Azure/azure-rest-api-specs-pr), cloned at `../azure-rest-api-specs-pr`

Decide which specs repo to use based on the PR link or URL the user provides. If the URL contains `azure-rest-api-specs-pr`, use the private repo. Otherwise use the public repo. If the user provides only a tspconfig path without a URL, ask which repo to use.

## Steps

### 1. Resolve input and prepare specs repo

**If the user provided a conversion PR link:**
1) Use the GitHub CLI or API to get the PR details and changed files.
2) Find the `tspconfig.yaml` file path from the PR's changed files (look for `specification/**/**/tspconfig.yaml`).
3) Run `gh pr checkout {PR_NUMBER}` in the specs repo directory to fetch and checkout the PR branch.

**If the user provided a tspconfig.yaml path or URL:**
1) If a URL, extract the relative path from it (the part after `specification/...`).
2) Ensure the specs repo is on the correct branch that contains the tspconfig (if the URL points to a specific branch, check it out).

Extract the spec folder name from the path pattern `specification/{SpecFolder}/...`.

### 2. Find the SDK folder

Read the `tspconfig.yaml` file found in step 1, check the `options/"@azure-tools/typespec-go"/module` property. Resolve the value if it has variable reference.
Extract folder `sdk/resourcemanager/**/**` according to the module path.

### 3. Find the TypeSpec API version

Check the TypeSpec folder (where `tspconfig.yaml` is located) in specs repo.

Find the last item in the `Version` enum. The enum is specified in the `@versioned` decorator, usually in `main.tsp`:

```ts
@versioned(Versions)
namespace Microsoft.NetApp;

enum Versions {
  ...
}
```

### 4. Find SDK API version

Check the SDK folder in the sdk repo. The API version can be found in the comment of any operation with the format `// Generated from API version <api-version>` in `xxx_client.go` files.

### 5. Compare API versions

Compare the TypeSpec API version (step 3) with the SDK API version (step 4).

If they are the same, skip to step 7.

If they are different, proceed to step 6 to generate a swagger baseline.

### 6. Generate SDK with Swagger for version-not-equal services

This step generates the SDK from the pre-conversion swagger spec to establish a proper changelog baseline.

1) In the specs repo, find the commit that first created the `tspconfig.yaml` file (e.g. using `git log --diff-filter=A --format="%H" -- {tspconfig_path}`). The "SwaggerCommit" is the direct parent of that commit (`{commit}~1`). This represents the repo state right before the TypeSpec conversion was introduced.
2) Based on SwaggerCommit, check the first found `specification/{SpecFolder}/resource-manager/**/readme.md` file. Use the TypeSpec API version to find pattern like `### Tag: package-{api-version}` and extract the whole tag as "SwaggerTag". If there is no such pattern, use the latest tag in the `readme.md` file.
3) Go to the SDK folder in the sdk repo, edit the `autorest.md` file: add or update the tag in the yaml according to the tag found `tag: {SwaggerTag}`. Also update the `require` URLs in `autorest.md` to point to the SwaggerCommit and the correct readme.md path found in step 2). The readme.md may be in a nested subfolder (e.g., `resource-manager/Microsoft.X/SubFolder/readme.md`) rather than at the `resource-manager/` root.
4) Go to the sdk repo root folder, run `generator release-v2 {sdk_repo_path} {specs_repo_path} {service} {armservice} --skip-generate-example --spec-commit-hash={SwaggerCommit}`. `{service}` and `{armservice}` can be extracted from the SDK folder path `sdk/resourcemanager/{service}/{armservice}`.
5) Note the generated `CHANGELOG.md` content from this branch for comparison.
6) Go back to main branch for the sdk repo.

### 7. Generate SDK locally using `generator generate`

Go to the sdk repo root folder, run:

```
generator generate {sdk_repo_path} {specs_repo_path} --tsp-config {tspconfig}
```

Where `{tspconfig}` is the relative path to `tspconfig.yaml` from the specs repo root.

If generation fails, report the error and stop.

### 8. Check changelog against breaking changes guide

Extract the latest version's changelog from the `CHANGELOG.md` file in the SDK folder generated by TypeSpec in step 7.

**If versions are equal (step 6 was skipped):** review all changelog items from the TypeSpec-generated changelog.

**If versions are not equal (step 6 was executed):** also extract the latest version's changelog from the swagger-generated `CHANGELOG.md` in step 6. Compare the two changelogs and compute the **diff** — only the items that appear in the TypeSpec changelog but NOT in the swagger changelog need to be reviewed. Items present in both changelogs are caused by the API version difference, not the TypeSpec conversion, and can be ignored.

Read the [breaking changes guide](https://github.com/Azure/azure-sdk-for-go/blob/main/documentation/development/breaking-changes/sdk-breaking-changes-guide-migration.md) (`documentation/development/breaking-changes/sdk-breaking-changes-guide-migration.md` in the sdk repo) and use it to classify each changelog item (or diff item for version-not-equal) into one of:

- **Resolvable** — can be fixed with TypeSpec customizations (e.g. `@@clientName`, `@@clientLocation`, `@@alternateType` in `client.tsp`, or emitter options in `tspconfig.yaml`). Apply the fix method described in the guide.
- **Acceptable** — expected changes that ship in a new major version. No fix needed.

### 9. Apply fixes

For each **resolvable** breaking change, apply the fix described in the breaking changes guide. Fixes are applied in the specs repo (on the conversion PR branch if working from a PR, or on the current branch). Common fix locations are `client.tsp` (for decorators like `@@clientName`, `@@clientLocation`, `@@alternateType`) and `tspconfig.yaml` (for emitter options) in the service's TypeSpec folder.

After making fixes:
1) Commit the changes in the specs repo.
2) Re-run step 7 (`generator generate`) to regenerate the SDK and verify the fixes.
3) Repeat steps 8–9 until the changelog only contains acceptable changes or is clean.

### 10. Finalize

Once all resolvable breaking changes have been addressed, summarize the result to the user, including:
- The Go SDK generation result
- The remaining acceptable breaking changes (if any)
- The fixes applied
