# Refresh process of Go SDK for services migrated to TypeSpec

## Status data

The status data is the "releases.csv" file.

## Required repositories

- [Azure/azure-sdk-for-go](https://github.com/Azure/azure-sdk-for-go), cloned at "../azure-sdk-for-go"
  Call this folder "sdk repo" for short.
- [Azure/azure-rest-api-specs](https://github.com/Azure/azure-rest-api-specs), cloned at "../azure-rest-api-specs"
  Call this folder "specs repo" for short.

## Terminal state

Following values in "Go" column is a terminal state. Skip process on these rows.

- **AlreadyTypeSpec**
  Go SDK is already generated and released from TypeSpec. We don't need to refresh again.
- **Done**
  We've refreshed the SDK from TypeSpec.
- **NoSpec**
  No TypeSpec source found. Nothing we can do.
- **VersionNotEqual**
  Version mismatch between SpecApiVersion and SdkApiVersion.

## Process

For each row in the status data, we will go through below steps.

1. Check status and skip all the following steps if the status is in terminal state.

2. Find TypeSpec config file and update the "tspconfig" column

If "tspconfig" column is existing, it means we have already found the config file for this service. We can skip to next step.

Else, check folder `specification/{SpecFolder}/**/*.Management/**` or `specification/{SpecFolder}/resource-manager/**` in specs repo, find the "tspconfig.yaml"

Add the relative path (from specs repo) to the "tspconfig" column.

If no "tspconfig.yaml" is found, add "NoSpec" to "Go" column, and skip all the following steps for this row.

2. Find generated SDK folder and update the "SdkFolder" column
If "SdkFolder" column is existing, it means we have already found the generated SDK folder for this service. We can skip to next step.

Else, read the `tspconfig.yaml` file found in last step, check the `options/"@azure-tools/typespec-go"/module` property. Resolve the value if it has variable reference.
Extract folder `sdk/resourcemangager/**/**` according to the module path and put it in "SdkFolder" column.

3. Check if the Go SDK is already generated from TypeSpec

If there is a `tsp-location.yaml` file under the `SdkFolder` path, it means the Go SDK has already been released from TypeSpec. Add "AlreadyTypeSpec" to "Go" column, and skip all the following steps for this row.

4. Find the first conversion API version for the service, and add it to "SpecApiVersion" column

If "SpecApiVersion" column is existing, it means we have already found the API version for this service. We can skip to next step.

Else, check the folder of "tspconfig" column in specs repo.

Find the first item in `Version` enum. Add it to the "SpecApiVersion" column.

The `Version` enum can be found as the enum specified in `@versioned` decorator. This usually in "main.tsp" file.

```ts
@versioned(Versions)
namespace Microsoft.NetApp;

enum Versions {
  ...
}
```
5. Find SDK API version and update the "SdkApiVersion" column

If "SdkApiVersion" column is existing, it means we have already found the SDK API version for this service. We can skip to next step.

Else, check the folder of "SdkFolder" column in sdk repo. The API version could be found in the comment of any operation with formats like `// Generated from API version <api-version>` in `xxx_client.go` file. Add the API version to "SdkApiVersion" column.

6. Compare `SpecApiVersion` with `SdkApiVersion`

Compare the API version in "SpecApiVersion" column with "SdkApiVersion" column. If they are not same, add "VersionNotEqual" to "Go" column, and skip all the following steps for this row.

7. Generate SDK {sdk} via pipeline

If "SdkPr" column has a PR link, it means we have already generated the SDK. We can skip to next step.

Else, run the pipeline https://dev.azure.com/azure-sdk/internal/_build?definitionId=7426 via REST API
- Set "Path to API specification file" as value in "tspconfig" column
- Set "API version" in "SpecApiVersion" column
- Set "SDK release type" as beta
- Set "Create SDK pull request" to "true"
Use the token from Azure CLI to call the REST API of the "dev.azure.com" endpoint (preferably using `az rest` and let Azure CLI handle the token, with `Content-Type=application/json` via `--header`)

Wait for the pipeline run to complete. Check recent PR on https://github.com/Azure/azure-sdk-for-go/pulls, find "[AutoPR sdk-SdkFolder]*", replace "AutoPR" with "Refresh" for the PR title, and add the link of PR to "SdkPr" column.

8. Check changelog in the PR

Extract the latest version's changelog from the PR and check each item to see if it is acceptable according to `documentation/development/breaking-changes/sdk-breaking-changes-guide-migration.md` under sdk repo. If there is any undocumented items, add "ManualReview" to "Go" column for this row. If all items are acceptable, update "Go" column to "ReadyReview".
