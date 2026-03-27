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

Else, check folder `specification/{SpecFolder}/**/*.Management/**` or `specification/{SpecFolder}/resource-manager/**` in specs repo, find the "tspconfig.yaml"

Add the relative path (from specs repo) to the "tspconfig" column.

If no "tspconfig.yaml" is found, add "NoSpec" to "Go" column, and skip all the following steps for this row.

3. Find generated SDK folder and update the "SdkFolder" column
If "SdkFolder" column is existing, it means we have already found the generated SDK folder for this service. We can skip to next step.

Else, read the `tspconfig.yaml` file found in last step, check the `options/"@azure-tools/typespec-go"/module` property. Resolve the value if it has variable reference.
Extract folder `sdk/resourcemangager/**/**` according to the module path and put it in "SdkFolder" column.

4. Check if the Go SDK is already generated from TypeSpec

If there is a `tsp-location.yaml` file under the `SdkFolder` path, it means the Go SDK has already been released from TypeSpec. Add "AlreadyTypeSpec" to "Go" column.
Then check if there is a tag existed with `sdk/resourcemanager/{service}/{armmodule}/v{<latest_version>}`. `{service}` and `{armmodule}` can be extracted from "SdkFolder" column. `{latest_version}` can be extracted from the latest version in `CHANGELOG.md` file.
If such a tag exists, add "Done" to "Go" column. Skip all the following steps for this row.

5. Find the first conversion API version for the service, and add it to "SpecApiVersion" column

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
6. Find SDK API version and update the "SdkApiVersion" column

If "SdkApiVersion" column is existing, it means we have already found the SDK API version for this service. We can skip to next step.

Else, check the folder of "SdkFolder" column in sdk repo. The API version could be found in the comment of any operation with formats like `// Generated from API version <api-version>` in `xxx_client.go` file. Add the API version to "SdkApiVersion" column.

7. Compare `SpecApiVersion` with `SdkApiVersion`

Compare the API version in "SpecApiVersion" column with "SdkApiVersion" column. If they are not same, add "VersionNotEqual" to "Go" column.

8. Generate SDK via pipeline

If "SdkPr" column has a PR link, it means we have already generated the SDK. We can skip to next step.

Else, run the pipeline https://dev.azure.com/azure-sdk/internal/_build?definitionId=7426 via REST API
- Set "Path to API specification file" as value in "tspconfig" column
- Set "API version" in "SpecApiVersion" column
- Set "SDK release type" as beta
- Set "Create SDK pull request" to "true"
Use the token from Azure CLI to call the REST API of the "dev.azure.com" endpoint (preferably using `az rest` and let Azure CLI handle the token, with `Content-Type=application/json` via `--header`)

Wait for the pipeline run to complete. Check recent PR on https://github.com/Azure/azure-sdk-for-go/pulls, find "[AutoPR sdk-{SdkFolder}]*", replace "AutoPR" with "Refresh" for the PR title, and add the link of PR to "SdkPr" column.

9. Generate SDK with Swagger for "VersionNotEqual" services

If "SdkChangelog" column has a link, it means we have already generated SDK with Swagger spec. We can skip to next step.

For the services with "VersionNotEqual" status, we need to generate SDK with Swagger spec.

Follow these steps to generate SDK with Swagger spec:
1) Go to the folder of "tspconfig" column in specs repo, find the commit this config is first created. Go to the `specification/{Spec Folder}` in specs repo, get the -1 commit ID of that commit and add it to "SpecCommit" column.
2) Based on this commit ID, check the first found `specification/{SpecFolder}/resource-manager/**/readme.md` file. Use "SpecApiVersion" to find pattern like `### Tag: package-{SpecApiVersion}` and extract the whole tag into the "SwaggerTag" column. If there is no such pattern, use the latest tag in the `readme.md` file.
3) Go to the folder of "SdkFolder" column in sdk repo, edit the `autorest.md` file: add or update the tag in the yaml to `tag: {SwaggerTag}`.
4) Go to the sdk repo root folder, run `generator release-v2 c:/w/azure-sdk-for-go  c:/w/azure-rest-api-specs {service} {armservice} --skip-generate-example --spec-commit-hash={SpecCommit}`. `{service}` and `{armservice}` could be extracted from "SdkFolder" column. Push the new created branch to remote. Put the link of the `CHANGELOG.md` file from this new branch to "SdkChangelog" column.
5) Leave a comment in the "SdkPr" with the link of "SdkChangelog" column.
6) After all, you need to go back to main for the sdk repo.

<!-- 9. Check changelog in the PR

Extract the latest version's changelog from the PR and check each item to see if it is acceptable according to `documentation/development/breaking-changes/sdk-breaking-changes-guide-migration.md` under sdk repo. If there is any undocumented items, add "ManualReview" to "Go" column for this row. If all items are acceptable, update "Go" column to "ReadyReview". -->


If any problem happens in above steps, add "Error" to "Go" column and summarize the problem in "Comment" column.