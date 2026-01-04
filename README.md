# build

Shared build tools and reusable GitHub Actions workflows for continuous deployment using [genifest](https://github.com/zostay/genifest).

## Workflows

### Continuous Deployment (`cd.yaml`)

A reusable workflow for continuous deployment that uses genifest to update Kubernetes manifests. This workflow:

1. Installs the genifest CLI tool
2. Validates your genifest configuration
3. Runs genifest to update manifests
4. Optionally commits and pushes the changes

#### Basic Usage

```yaml
name: Deploy
on:
  push:
    branches:
      - main

jobs:
  deploy:
    uses: zostay/build/.github/workflows/cd.yaml@main
```

#### Example with Pull Request Creation

```yaml
name: Deploy via Pull Request
on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  create-deployment-pr:
    uses: zostay/build/.github/workflows/cd.yaml@main
    with:
      genifest-version: 'latest'
      genifest-group: 'prod'
      genifest-tag: '!secret-*'
      working-directory: './k8s'
      output-mode: 'markdown'             # markdown output for rich PR descriptions
      commit-changes: true
      push-changes: true
      create-pr: true                     # enable PR creation
      pr-target-branch: 'deploy-prod'     # optional: custom target branch
      commit-message: 'chore: update production manifests'
```

This workflow will:
1. Create a timestamped deployment branch from the current branch
2. Run genifest and capture output in Markdown format
3. Commit the manifest changes to the deployment branch
4. Create a pull request targeting `deploy-prod` with the genifest output as the description

#### Full Example with All Options

```yaml
name: Deploy to Production
on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  update-manifests:
    uses: zostay/build/.github/workflows/cd.yaml@main
    with:
      genifest-version: 'latest'          # or specific version like '1.0.0-rc6'
      genifest-group: 'prod'              # genifest group to run
      genifest-tag: '!secret-*'           # additional tag expressions
      working-directory: './k8s'          # directory containing genifest.yaml
      output-mode: 'markdown'             # auto, color, plain, or markdown
      commit-changes: true                # whether to commit changes
      commit-message: 'chore: update production manifests'
      push-changes: true                  # whether to push committed changes
      create-pr: true                     # create pull request for approval
      pr-target-branch: 'deploy-main'     # target branch for PR
      git-user-name: 'github-actions[bot]'
      git-user-email: 'github-actions[bot]@users.noreply.github.com'
      environment-variables: '{"REGISTRY_URL":"ghcr.io","APP_VERSION":"v1.2.3","DEBUG":"true"}'
```

#### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `genifest-version` | Version of genifest to install | No | `latest` |
| `genifest-group` | Genifest group to run | No | `all` |
| `genifest-tag` | Additional tag expression | No | `''` |
| `working-directory` | Directory containing genifest.yaml | No | `.` |
| `output-mode` | Output mode (auto, color, plain, markdown) | No | `markdown` |
| `commit-changes` | Whether to commit manifest changes | No | `true` |
| `commit-message` | Commit message for changes | No | `chore: update kubernetes manifests via genifest` |
| `push-changes` | Whether to push committed changes | No | `true` |
| `create-pr` | Whether to create a pull request for changes | No | `false` |
| `pr-target-branch` | Target branch for PR (defaults to deploy-{source-branch}) | No | `''` |
| `git-user-name` | Git user name for commits | No | `github-actions[bot]` |
| `git-user-email` | Git user email for commits | No | `github-actions[bot]@users.noreply.github.com` |
| `environment-variables` | Environment variables to set for genifest execution (JSON format) | No | `'{}'` |
| `update-build-number` | Whether to update the build number in object store | No | `false` |
| `build-number-app-name` | Application name for build number (required if update-build-number is true) | No | `''` |
| `build-number-image-name` | Image name for build number | No | `main` |
| `build-number-env-name` | Environment name for build number | No | `prod` |

#### Outputs

| Output | Description |
|--------|-------------|
| `changes-made` | Whether any manifest changes were made (`true` or `false`) |
| `commit-sha` | SHA of the commit with manifest changes (if committed) |
| `pr-number` | Pull request number (if created) |
| `pr-url` | Pull request URL (if created) |

#### Using Outputs

```yaml
jobs:
  update-manifests:
    uses: zostay/build/.github/workflows/cd.yaml@main
    with:
      genifest-group: 'prod'
      create-pr: true

  notify:
    needs: update-manifests
    runs-on: ubuntu-latest
    if: needs.update-manifests.outputs.changes-made == 'true'
    steps:
      - name: Notify about deployment
        run: |
          echo "Manifests updated with commit: ${{ needs.update-manifests.outputs.commit-sha }}"
          if [ -n "${{ needs.update-manifests.outputs.pr-url }}" ]; then
            echo "Pull request created: ${{ needs.update-manifests.outputs.pr-url }}"
          fi
```

## Enhanced PR-Based Deployment Workflow

When `create-pr: true` is enabled, the workflow implements a state-of-the-art deployment process:

### Branch Management
- Creates timestamped deployment branches (`deploy-{BUILD_TAG}-{TIMESTAMP}`)
- Handles existing deploy branches by merging deployment-specific changes
- Maintains clean separation between source and deployment branches

### Markdown Logging
- Captures genifest output in a structured Markdown format
- Includes environment variables, genifest version, configuration, and detailed output
- Provides rich, readable deployment reports for review

### Pull Request Integration
- Automatically creates pull requests targeting deployment branches
- Uses the captured Markdown log as the PR description
- Provides complete visibility into what changes are being deployed
- Enables team review and approval before deployment

This approach ensures that all deployment changes go through proper review processes while maintaining full automation and detailed audit trails.

## Environment Variables

The workflow supports setting custom environment variables that will be available during genifest execution. This is useful for passing configuration values, API tokens, or other dynamic values that genifest might need.

### Usage

Environment variables are passed as a JSON string using the `environment-variables` input:

```yaml
jobs:
  deploy:
    uses: zostay/build/.github/workflows/cd.yaml@main
    with:
      genifest-group: 'prod'
      environment-variables: '{"API_TOKEN":"${{ secrets.API_TOKEN }}","REGISTRY_URL":"ghcr.io","APP_VERSION":"v1.2.3"}'
```

### Features

- **JSON Format**: Variables are specified as a JSON object with string keys and values
- **Validation**: Variable names are validated to ensure they're valid shell identifiers
- **Logging**: Custom variables are logged separately in the deployment report for visibility
- **Security**: Sensitive values should use GitHub secrets and will be masked in logs
- **Flexible**: Supports any number of environment variables

### Examples

#### Basic Environment Variables
```yaml
environment-variables: '{"DEBUG":"true","LOG_LEVEL":"info"}'
```

#### Using GitHub Secrets
```yaml
environment-variables: '{"API_TOKEN":"${{ secrets.API_TOKEN }}","DB_PASSWORD":"${{ secrets.DB_PASSWORD }}"}'
```

#### Complex Configuration
```yaml
environment-variables: |
  {
    "REGISTRY_URL": "ghcr.io",
    "APP_VERSION": "${{ github.ref_name }}",
    "BUILD_NUMBER": "${{ github.run_number }}",
    "ENVIRONMENT": "production",
    "DEBUG": "false"
  }
```

### Variable Name Requirements

- Must start with a letter (a-z, A-Z) or underscore (_)
- Can contain letters, numbers, and underscores
- Cannot contain spaces or special characters
- Examples: `API_TOKEN`, `registry_url`, `AppVersion`, `_internal_var`

## Build Number Management

The workflow can automatically update build numbers in an S3-compatible object store. This is useful for tracking which build number is deployed to specific environments, enabling coordinated deployments across multiple projects.

### Features

- **Automatic Build Numbers**: Build numbers are automatically set to `{run_number}.{run_attempt}` (e.g., "14.1")
- **Multi-Environment Support**: Track build numbers for different environments (prod, staging, etc.)
- **Multi-Image Support**: Track separate build numbers for different images within the same application
- **S3-Compatible Storage**: Works with AWS S3, Civo Object Store, MinIO, or any S3-compatible service

### Configuration

To use build number tracking, you need to configure the following secrets and variables in your repository or GitHub environment:

**Secrets:**
- `AWS_ACCESS_KEY_ID`: Access key for your object store
- `AWS_SECRET_ACCESS_KEY`: Secret key for your object store

**Variables:**
- `AWS_DEFAULT_REGION`: Region where your bucket is located
- `AWS_ENDPOINT_URL`: Endpoint URL for your S3-compatible service (e.g., `https://objectstore.nyc1.civo.com`)
- `BUILD_NUMBER_BUCKET`: (Optional) Name of the bucket to use (defaults to `qubling-cloud-production`)

### Usage Example

```yaml
jobs:
  deploy:
    uses: zostay/build/.github/workflows/cd.yaml@main
    with:
      genifest-group: 'prod'
      update-build-number: true
      build-number-app-name: 'myapp'
      build-number-image-name: 'main'
      build-number-env-name: 'prod'
    secrets: inherit
```

This will store the build number at the S3 path: `build-number/myapp/main/prod.txt`

### Multiple Images Example

If your application has multiple images (e.g., web server and worker):

```yaml
jobs:
  deploy-web:
    uses: zostay/build/.github/workflows/cd.yaml@main
    with:
      genifest-group: 'web'
      update-build-number: true
      build-number-app-name: 'myapp'
      build-number-image-name: 'web'
      build-number-env-name: 'prod'
    secrets: inherit

  deploy-worker:
    uses: zostay/build/.github/workflows/cd.yaml@main
    with:
      genifest-group: 'worker'
      update-build-number: true
      build-number-app-name: 'myapp'
      build-number-image-name: 'worker'
      build-number-env-name: 'prod'
    secrets: inherit
```

### How It Works

When enabled, the workflow:
1. Downloads the `set-build-number` script from this repository (if not already present)
2. Constructs a build number from `github.run_number` and `github.run_attempt` (e.g., "14.1")
3. Stores the build number as a text file in your S3-compatible object store at:
   ```
   build-number/{app-name}/{image-name}/{env-name}.txt
   ```
4. This build number can then be retrieved by other systems or workflows to coordinate deployments

## Requirements

Your repository should have a `genifest.yaml` configuration file. See the [genifest documentation](https://github.com/zostay/genifest) for configuration details.

## License

MIT License - see the genifest repository for full licensing details.
