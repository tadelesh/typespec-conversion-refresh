---
name: release-azure-sdk-for-go
description: Use when the user wants to release a package from Azure/azure-sdk-for-go by triggering its Azure DevOps pipeline, monitoring the build, and approving the related release stages. User provides an Azure SDK for Go package such as "armfrontdoor" or "sdk/resourcemanager/frontdoor/armfrontdoor".
---

# Release Azure SDK for Go

Trigger the Azure DevOps release pipeline for an Azure/azure-sdk-for-go package, monitor the build, and approve the related release stages.

## Scope

This skill is only for packages published from Azure/azure-sdk-for-go.

- Use it only when the requested package belongs to Azure/azure-sdk-for-go.
- Do not use it for other Go repositories.
- Do not route through Python, Java, JavaScript, or .NET release pipelines.

## Prerequisites

- Azure CLI installed and authenticated with `az login`
- Azure DevOps CLI extension available to the current `az` installation
- Access to `https://dev.azure.com/azure-sdk`, project `internal`
- A local Azure/azure-sdk-for-go checkout, for example `/path/to/azure-sdk-for-go`

## Input

- **SDK package name** (required): e.g., `sdk/resourcemanager/frontdoor/armfrontdoor` or `armfrontdoor`
- **Local path to Azure/azure-sdk-for-go checkout** (optional, default: `/path/to/azure-sdk-for-go`)

## Workflow

### Step 0: Check Prerequisites

Verify Azure authentication first:

```
az account show --query "{name:name, user:user.name}" -o json
```

Verify the Azure DevOps extension is available:

```
az extension show --name azure-devops --output json
```

- If Azure CLI is missing, tell the user to install it from https://learn.microsoft.com/cli/azure/install-azure-cli and run `az login`.
- If Azure login is missing, tell the user to run `az login` and stop.
- If the Azure DevOps extension is missing, install it before proceeding:

```
az extension add --name azure-devops
```

Do not continue until these checks pass.

### Step 1: Resolve the Package in Azure/azure-sdk-for-go

Normalize the input to a repo-relative package directory.

- If the user already provided a repo-relative directory such as `sdk/resourcemanager/frontdoor/armfrontdoor`, use it directly.
- If the user provided a short name or legacy package name, search the local Azure/azure-sdk-for-go checkout for the matching module.

Useful search patterns:

```
cd <azure-sdk-for-go-root>
rg -n --glob 'sdk/**/go.mod' --glob 'services/**/go.mod' '<package-identifier>'
rg -n --glob 'sdk/**/README.md' --glob 'services/**/README.md' '<package-identifier>'
```

Extract these values:

- `package_dir`: repo-relative package directory
- `package_name`: final path segment or explicit package identifier
- `service_name`: the service folder used by the release pipeline

For `sdk/resourcemanager/<service>/<module>` and `sdk/<service>/<module>`, use `<service>` as the service name.

If multiple package directories match and you cannot determine the intended package unambiguously, stop and ask the user which package to release.

### Step 2: Find the Go Release Pipeline

Look up the pipeline definition by the Azure SDK for Go naming convention `go - <service_name>`:

```
az devops invoke --area build --resource definitions --route-parameters project=internal --org https://dev.azure.com/azure-sdk --api-version=7.0 --query-parameters "name=go - <service_name>" --query "value[0].{id:id,name:name}" -o json
```

Extract the pipeline `id` and `name`.

If no pipeline is found, report that there is no matching Azure SDK for Go release pipeline for the resolved service name and stop.

### Step 3: Trigger the Pipeline

Use an Azure DevOps access token from `az` and trigger the pipeline with the resolved Go package target.

Run an inline Python snippet:

```python
import json, subprocess
from urllib.request import Request, urlopen

token = subprocess.run(
    "az account get-access-token --resource 499b84ac-1321-427f-aa17-267ca6975798 --query accessToken -o tsv",
    shell=True,
    capture_output=True,
    text=True,
    check=True,
).stdout.strip()

url = "https://dev.azure.com/azure-sdk/internal/_apis/build/builds?api-version=7.0"
body = json.dumps({
    "definition": {"id": <pipeline_id>},
    "sourceBranch": "refs/heads/main",
    "parameters": json.dumps({
        "BuildTargetingString": "<package_dir>",
        "Skip.CreateApiReview": "true"
    })
})

req = Request(url, method="POST", data=body.encode())
req.add_header("Authorization", f"Bearer {token}")
req.add_header("Content-Type", "application/json")
with urlopen(req) as resp:
    result = json.loads(resp.read())

build_id = result["id"]
build_url = f"https://dev.azure.com/azure-sdk/internal/_build/results?buildId={build_id}&view=results"
print(json.dumps({"build_id": build_id, "build_url": build_url}))
```

Print the build link for the user:

```
Pipeline triggered: <pipeline_name>
Build ID: <build_id>
Build URL: <build_url>
```

### Step 4: Monitor the Build

Poll the build until it reaches a terminal state:

```
az pipelines build show --id <build_id> --org https://dev.azure.com/azure-sdk --project internal --query "{status:status,result:result,buildNumber:buildNumber}" -o json
```

- Continue polling while `status` is not completed.
- If the build completes with a failed, canceled, or partially succeeded result, report the failure and stop.

### Step 5: Approve the Matching Release Stages

Query pending approvals from the Azure DevOps release endpoint:

```
az devops invoke --area release --resource approvals --route-parameters project=internal --org https://vsrm.dev.azure.com/azure-sdk --api-version=7.1-preview.3 --query-parameters "statusFilter=pending" -o json
```

Approve only the approvals that clearly belong to the triggered Azure SDK for Go release.

Use the approval metadata to match on the resolved service name, package directory, release definition, or current build number. Never approve unrelated pending releases.

Approve a matching approval with:

```
az devops invoke --area release --resource approvals --route-parameters project=internal approvalId=<approval_id> --org https://vsrm.dev.azure.com/azure-sdk --api-version=7.1-preview.3 --http-method PATCH --in-file <json-file>
```

Where the JSON body is:

```json
{
  "status": "approved",
  "comments": "Approved by release-azure-sdk-for-go skill"
}
```

After approving the matching stages, continue monitoring until the release pipeline finishes or until there are no more pending approvals related to this build.

### Step 6: Report the Result

When the release succeeds, report:

```
Released <package_dir> successfully.
Pipeline: <pipeline_name>
Build: <build_url>
```

If the build output or release logs clearly show a released version, include it. Do not invent or guess a version.

## Rules

- Always use `az` CLI for Azure DevOps authentication. Never ask the user for a PAT.
- This skill is Azure/azure-sdk-for-go only.
- Use the Go pipeline naming convention `go - <service_name>`.
- Use the repo-relative package directory as `BuildTargetingString` whenever possible.
- The `parameters` field in the Azure DevOps trigger request must be a JSON-encoded string, not an object.
- Azure DevOps org is always `https://dev.azure.com/azure-sdk` and project is always `internal`.
- Release approvals are queried from `https://vsrm.dev.azure.com/azure-sdk`.
- Do not mention PyPI or use Python package verification. This is a Go package release workflow.
- Do not depend on helper scripts that are not present in this skill folder.
- When a release fails, report the failure root cause and build URL to the user, then stop. Do not trigger another pipeline or attempt automated remediation unless the user explicitly asks.
