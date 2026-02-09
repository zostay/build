# build

Shared build tools and reusable GitHub Actions workflows for continuous deployment using [genifest](https://github.com/zostay/genifest).

## Workflows

### Continuous Deployment (`cd.yaml`)

A reusable workflow for continuous deployment that uses genifest to update Kubernetes manifests. This workflow:

1. Installs the genifest CLI tool
2. Validates your genifest configuration
3. Runs genifest to update manifests
4. Optionally commits and pushes the changes

#### Important: Secrets and Variables Configuration

**GitHub Actions Limitation:** Jobs that call reusable workflows (`uses:`) **cannot** use the `environment:` keyword.

**Required Setup:** Configure these as **repository-level secrets and variables** in Settings → Secrets and variables → Actions:

- **Secrets** (Repository level): `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
- **Variables** (Repository level): `AWS_DEFAULT_REGION`, `AWS_ENDPOINT_URL`, `BUILD_NUMBER_BUCKET`

These will be inherited by the reusable workflow via `secrets: inherit`.

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
    secrets: inherit
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
    secrets: inherit
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
    secrets: inherit
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
| `aws-region` | AWS region for S3-compatible object store | No | `''` |
| `aws-endpoint-url` | AWS endpoint URL for S3-compatible object store | No | `''` |
| `build-number-bucket` | S3 bucket name for build number storage | No | `qubling-cloud-production` |

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
    secrets: inherit

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
    secrets: inherit
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

To use build number tracking, configure the following as repository-level secrets and variables:

**Secrets:**
- `AWS_ACCESS_KEY_ID`: Access key for your object store
- `AWS_SECRET_ACCESS_KEY`: Secret key for your object store

**Variables:**
- `AWS_DEFAULT_REGION`: Region where your bucket is located
- `AWS_ENDPOINT_URL`: Endpoint URL for your S3-compatible service (e.g., `https://objectstore.nyc1.civo.com`)
- `BUILD_NUMBER_BUCKET`: (Optional) Name of the bucket to use (defaults to `qubling-cloud-production`)

**Note:** These must be configured as repository-level secrets and variables, not environment-level, because reusable workflows cannot access environment-scoped secrets.

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
      aws-region: ${{ vars.AWS_DEFAULT_REGION }}
      aws-endpoint-url: ${{ vars.AWS_ENDPOINT_URL }}
      build-number-bucket: ${{ vars.BUILD_NUMBER_BUCKET }}
    secrets: inherit
```

**Note:** The `aws-region`, `aws-endpoint-url`, and `build-number-bucket` inputs must be passed from your repository's variables. The secrets (`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`) are automatically passed via `secrets: inherit`.

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
      aws-region: ${{ vars.AWS_DEFAULT_REGION }}
      aws-endpoint-url: ${{ vars.AWS_ENDPOINT_URL }}
      build-number-bucket: ${{ vars.BUILD_NUMBER_BUCKET }}
    secrets: inherit

  deploy-worker:
    uses: zostay/build/.github/workflows/cd.yaml@main
    with:
      genifest-group: 'worker'
      update-build-number: true
      build-number-app-name: 'myapp'
      build-number-image-name: 'worker'
      build-number-env-name: 'prod'
      aws-region: ${{ vars.AWS_DEFAULT_REGION }}
      aws-endpoint-url: ${{ vars.AWS_ENDPOINT_URL }}
      build-number-bucket: ${{ vars.BUILD_NUMBER_BUCKET }}
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

### Update Build Number (`update-build-number.yaml`)

A lightweight workflow that only updates the build number in object storage without running genifest. Use this for projects that handle their own builds but need to record the build number for centralized deployment.

#### Usage

```yaml
name: Record Build Number
on:
  push:
    branches:
      - main

jobs:
  record-build:
    uses: zostay/build/.github/workflows/update-build-number.yaml@master
    with:
      app-name: 'myapp'
      aws-region: ${{ vars.AWS_DEFAULT_REGION }}
      aws-endpoint-url: ${{ vars.AWS_ENDPOINT_URL }}
      bucket: ${{ vars.BUILD_NUMBER_BUCKET }}
    secrets: inherit
```

#### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `app-name` | Application name for build number | Yes | - |
| `image-name` | Image name for build number | No | `main` |
| `env-name` | Environment name for build number | No | `prod` |
| `aws-region` | AWS region for S3-compatible object store | Yes | - |
| `aws-endpoint-url` | AWS endpoint URL for S3-compatible object store | Yes | - |
| `bucket` | S3 bucket name for build number storage | No | `qubling-cloud-production` |

#### Secrets

| Secret | Description | Required |
|--------|-------------|----------|
| `AWS_ACCESS_KEY_ID` | AWS access key ID | Yes |
| `AWS_SECRET_ACCESS_KEY` | AWS secret access key | Yes |

#### Outputs

| Output | Description |
|--------|-------------|
| `build-number` | The build number that was recorded (e.g., `14.1`) |

#### When to Use

Use this workflow instead of the full CD workflow when:
- Your project builds container images but doesn't manage its own Kubernetes manifests
- You want to delegate manifest generation to a centralized deployment repository
- You need to record build numbers for coordination with other systems
- You want minimal overhead and fast execution

---

### GHCR Lifecycle Management (`ghcr-lifecycle.yaml`)

Manages container image lifecycle in GitHub Container Registry (ghcr.io). Similar to AWS ECR lifecycle policies, this workflow prunes old images while keeping a minimum number of recent versions.

#### Features

- **Minimum Version Retention**: Always keeps at least N versions regardless of age
- **Age-Based Pruning**: Only deletes versions older than `delete-after-days`
- **Per-Environment Lifecycle**: Apply retention policies independently per tag prefix (e.g., `prod-*` and `stg-*` each keep their own N versions)
- **Protected Tags**: Glob patterns to protect specific tags from deletion (e.g., `latest`, `v*`, `prod-*`)
- **Untagged Cleanup**: Option to include or exclude untagged images
- **Dry-Run Mode**: Preview what would be deleted without actually deleting
- **User/Org Support**: Automatically detects whether the package owner is a user or organization

#### Usage

```yaml
name: Cleanup Container Images
on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly on Sunday at midnight
  workflow_dispatch:      # Allow manual runs

jobs:
  cleanup:
    uses: zostay/build/.github/workflows/ghcr-lifecycle.yaml@master
    with:
      package-name: 'myapp'
      min-versions-to-keep: 10
      delete-after-days: 30
      protected-tags: 'latest,v*,prod-*'
    secrets:
      GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

#### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `package-name` | Name of the container package to manage | Yes | - |
| `package-owner` | Owner of the package (user or org) | No | Repository owner |
| `min-versions-to-keep` | Minimum number of versions to keep regardless of age (per prefix group if `tag-prefixes` is set) | No | `10` |
| `delete-after-days` | Versions older than this many days are eligible for deletion | No | `30` |
| `tag-prefixes` | Comma-separated list of tag prefixes to manage independently (e.g., `prod-,stg-`) | No | `''` |
| `dry-run` | If true, only report what would be deleted | No | `false` |
| `include-untagged` | If true, include untagged versions in cleanup | No | `true` |
| `protected-tags` | Comma-separated list of tag patterns to never delete | No | `latest` |

#### Secrets

| Secret | Description | Required |
|--------|-------------|----------|
| `GHCR_TOKEN` | GitHub token with `read:packages` and `delete:packages` permissions | Yes |

**Note:** You can use `${{ secrets.GITHUB_TOKEN }}` if the workflow has `packages: write` permission, or create a Personal Access Token (PAT) with the required scopes.

#### Outputs

| Output | Description |
|--------|-------------|
| `deleted-count` | Number of versions deleted |
| `kept-count` | Number of versions kept |
| `dry-run-would-delete` | Number of versions that would be deleted (dry-run only) |

#### Protected Tag Patterns

The `protected-tags` input accepts glob-style patterns:

| Pattern | Matches |
|---------|---------|
| `latest` | Exactly `latest` |
| `v*` | Any tag starting with `v` (e.g., `v1.0.0`, `v2.3.4`) |
| `prod-*` | Any tag starting with `prod-` (e.g., `prod-main`, `prod-hotfix`) |
| `*-stable` | Any tag ending with `-stable` |

Multiple patterns can be combined with commas: `latest,v*,prod-*,*-stable`

#### Deletion Logic

Versions are evaluated in order from newest to oldest:

1. **Protected tags**: Never deleted, regardless of age or count
2. **Minimum count**: The first N versions are always kept (where N = `min-versions-to-keep`)
3. **Age check**: Remaining versions are only deleted if older than `delete-after-days`
4. **Untagged handling**: Untagged versions follow the same rules if `include-untagged: true`

#### Per-Environment Lifecycle with Tag Prefixes

A common pattern is to tag container images by environment, such as `prod-29.1` for production and `stg-28.1` for staging. When cleaning up images, you typically want to apply retention policies **independently** for each environment - keeping 10 production versions shouldn't count against your staging quota and vice versa.

The `tag-prefixes` input enables this by grouping versions by their tag prefix and applying `min-versions-to-keep` separately to each group.

**Example: Independent retention for prod and staging environments**

```yaml
name: Cleanup Container Images
on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly
  workflow_dispatch:

jobs:
  cleanup:
    uses: zostay/build/.github/workflows/ghcr-lifecycle.yaml@master
    with:
      package-name: 'myapp'
      tag-prefixes: 'prod-,stg-'
      min-versions-to-keep: 10
      delete-after-days: 7
    secrets:
      GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

With this configuration:
- The 10 most recent `prod-*` tagged versions are always kept
- The 10 most recent `stg-*` tagged versions are always kept (separately)
- Untagged versions get their own quota of 10
- Tags that don't match any prefix (e.g., `latest`, `v1.0.0`) go into an "other" group with its own quota
- After the minimum count is satisfied for each group, versions older than 7 days are eligible for deletion

**How versions are grouped:**

| Tag | Group |
|-----|-------|
| `prod-29.1` | `prod-` |
| `prod-28.1` | `prod-` |
| `stg-15.2` | `stg-` |
| `stg-14.1` | `stg-` |
| `latest` | `other` |
| `v1.0.0` | `other` |
| *(no tag)* | `untagged` |

**Example: Three environments with different prefixes**

```yaml
jobs:
  cleanup:
    uses: zostay/build/.github/workflows/ghcr-lifecycle.yaml@master
    with:
      package-name: 'myapp'
      tag-prefixes: 'prod-,stg-,dev-'
      min-versions-to-keep: 5
      delete-after-days: 14
      protected-tags: 'latest'
    secrets:
      GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

This keeps 5 versions each for `prod-*`, `stg-*`, and `dev-*` independently, plus protects `latest` from ever being deleted.

#### Example: Conservative Cleanup

Keep more versions and only delete very old images:

```yaml
jobs:
  cleanup:
    uses: zostay/build/.github/workflows/ghcr-lifecycle.yaml@master
    with:
      package-name: 'myapp'
      min-versions-to-keep: 25
      delete-after-days: 90
      protected-tags: 'latest,v*,release-*'
    secrets:
      GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

#### Example: Aggressive Cleanup with Dry-Run

Preview aggressive cleanup before actually deleting:

```yaml
jobs:
  cleanup:
    uses: zostay/build/.github/workflows/ghcr-lifecycle.yaml@master
    with:
      package-name: 'myapp'
      min-versions-to-keep: 5
      delete-after-days: 7
      dry-run: true
    secrets:
      GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Requirements

Your repository should have a `genifest.yaml` configuration file. See the [genifest documentation](https://github.com/zostay/genifest) for configuration details.

## License

MIT License - see the genifest repository for full licensing details.
