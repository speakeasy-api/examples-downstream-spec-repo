# Spec Repo with Downstream SDK Generation

This example demonstrates a "spec repo" workflow where a central repository holds OpenAPI specs, and changes trigger SDK generation in separate downstream SDK repositories.

## Use case

This pattern is useful when:

- A central team manages the API specification
- Multiple SDK repositories need to be generated from the same spec
- SDK generation should be triggered automatically when the spec changes
- SDK PRs should be created in downstream repos for review before merging

## How it works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Spec Repository                               │
│  ┌─────────────┐    ┌─────────────────────────────────────────────┐    │
│  │ OpenAPI     │    │ trigger-downstream-sdk-generation.yaml      │    │
│  │ Spec        │───▶│ 1. Runs `speakeasy run` to tag spec         │    │
│  │ (PR change) │    │ 2. Triggers downstream SDK workflows        │    │
│  └─────────────┘    │ 3. Polls for completion                     │    │
│                     │ 4. Updates PR comment with status           │    │
│                     └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ gh workflow run
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Downstream SDK Repos                             │
│  ┌──────────────────────────┐    ┌──────────────────────────┐          │
│  │ TypeScript SDK           │    │ Python SDK               │          │
│  │ - Pulls tagged spec      │    │ - Pulls tagged spec      │          │
│  │ - Generates SDK          │    │ - Generates SDK          │          │
│  │ - Creates PR             │    │ - Creates PR             │          │
│  └──────────────────────────┘    └──────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Repository structure

```
spec-repo-downstream-sdks/
├── .speakeasy/
│   └── workflow.yaml               # Source-only workflow
├── .github/workflows/
│   └── trigger-downstream-sdk-generation.yaml
├── specs/
│   └── openapi.yaml                # OpenAPI spec
└── README.md
```

## Required secrets

### On the spec repository

| Secret | Description | How to obtain |
|--------|-------------|---------------|
| `SPEAKEASY_API_KEY` | Authenticates the Speakeasy CLI | [Speakeasy Platform](https://app.speakeasy.com) → **API Keys** page |
| `DOWNSTREAM_SDK_TOKEN` | Triggers workflows in downstream SDK repos | See [Creating the downstream SDK token](#creating-the-downstream-sdk-token) |

### On each downstream SDK repository

| Secret | Description | How to obtain |
|--------|-------------|---------------|
| `SPEAKEASY_API_KEY` | Authenticates the Speakeasy CLI | Same key as spec repo, or a separate key |

## Creating the downstream SDK token

The `DOWNSTREAM_SDK_TOKEN` is a fine-grained Personal Access Token (PAT) that allows the spec repo to trigger workflows in the downstream SDK repositories.

### Steps to create

1. Go to [GitHub Settings → Developer Settings → Fine-grained personal access tokens](https://github.com/settings/tokens?type=beta)
2. Click **Generate new token**
3. Set the following:
   - **Token name**: `spec-repo-downstream-trigger` (or similar)
   - **Expiration**: Choose an appropriate expiration
   - **Resource owner**: Select the organization that owns the SDK repos (e.g., `speakeasy-api`)
   - **Repository access**: Select **Only select repositories** and choose all downstream SDK repos
4. Under **Permissions → Repository permissions**, set:
   - **Actions**: Read and write (to trigger `workflow_dispatch`)
   - **Contents**: Read and write (to push branches)
   - **Pull requests**: Read and write (to create PRs)
5. Click **Generate token** and copy the value
6. Add the token as a secret named `DOWNSTREAM_SDK_TOKEN` in the spec repository

## Setting up downstream SDK repositories

Each downstream SDK repository needs to be set up with Speakeasy:

```bash
# Clone the repo
gh repo clone speakeasy-api/examples-downstream-spec-sdk-typescript
cd examples-downstream-spec-sdk-typescript

# Run Speakeasy quickstart to configure the SDK
speakeasy quickstart

# Configure GitHub Actions
speakeasy configure github
```

### Modify the generated workflow

After running `speakeasy configure github`, modify the generated workflow file (e.g., `.github/workflows/generate.yaml`) to accept `workflow_dispatch` inputs:

```yaml
on:
  workflow_dispatch:
    inputs:
      force:
        description: "Force SDK regeneration"
        required: false
        default: "false"
      feature_branch:
        description: "Branch for SDK changes"
        required: false
      environment:
        description: "Environment variables (e.g., tag=branch-name)"
        required: false
  # ... keep other triggers
```

And pass these inputs to the workflow executor:

```yaml
jobs:
  generate:
    uses: speakeasy-api/sdk-generation-action/.github/workflows/workflow-executor.yaml@v15
    with:
      mode: pr
      force: ${{ inputs.force }}
      feature_branch: ${{ inputs.feature_branch }}
      environment: ${{ inputs.environment }}
    secrets:
      github_access_token: ${{ secrets.GITHUB_TOKEN }}
      speakeasy_api_key: ${{ secrets.SPEAKEASY_API_KEY }}
```

## Workflow

1. **Create a PR** in the spec repository with changes to `specs/openapi.yaml`
2. **Workflow runs**:
   - `speakeasy run` executes and tags the spec in the registry with the branch name
   - Downstream SDK workflows are triggered via `gh workflow run`
   - The spec PR is updated with a comment showing generation status
3. **Review SDK PRs** in the downstream repositories
4. **Merge** the spec PR and SDK PRs as needed

## Downstream SDK repositories

This example uses the following downstream SDK repositories:

- [speakeasy-api/examples-downstream-spec-sdk-typescript](https://github.com/speakeasy-api/examples-downstream-spec-sdk-typescript)
- [speakeasy-api/examples-downstream-spec-sdk-python](https://github.com/speakeasy-api/examples-downstream-spec-sdk-python)

## Related documentation

- [Publish specs to API registry](/docs/guides/source-only-workflow/)
- [SDK generation action](https://github.com/speakeasy-api/sdk-generation-action)
- [Speakeasy CLI CI/CD usage](https://www.speakeasy.com/docs/speakeasy-reference/cli/getting-started#using-the-speakeasy-cli-in-cicd)
