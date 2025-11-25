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
      output-mode: 'plain'                # auto, color, plain, or markdown
      commit-changes: true                # whether to commit changes
      commit-message: 'chore: update production manifests'
      push-changes: true                  # whether to push committed changes
      git-user-name: 'github-actions[bot]'
      git-user-email: 'github-actions[bot]@users.noreply.github.com'
```

#### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `genifest-version` | Version of genifest to install | No | `latest` |
| `genifest-group` | Genifest group to run | No | `all` |
| `genifest-tag` | Additional tag expression | No | `''` |
| `working-directory` | Directory containing genifest.yaml | No | `.` |
| `output-mode` | Output mode (auto, color, plain, markdown) | No | `plain` |
| `commit-changes` | Whether to commit manifest changes | No | `true` |
| `commit-message` | Commit message for changes | No | `chore: update kubernetes manifests via genifest` |
| `push-changes` | Whether to push committed changes | No | `true` |
| `git-user-name` | Git user name for commits | No | `github-actions[bot]` |
| `git-user-email` | Git user email for commits | No | `github-actions[bot]@users.noreply.github.com` |

#### Outputs

| Output | Description |
|--------|-------------|
| `changes-made` | Whether any manifest changes were made (`true` or `false`) |
| `commit-sha` | SHA of the commit with manifest changes (if committed) |

#### Using Outputs

```yaml
jobs:
  update-manifests:
    uses: zostay/build/.github/workflows/cd.yaml@main
    with:
      genifest-group: 'prod'
  
  notify:
    needs: update-manifests
    runs-on: ubuntu-latest
    if: needs.update-manifests.outputs.changes-made == 'true'
    steps:
      - name: Notify about deployment
        run: |
          echo "Manifests updated with commit: ${{ needs.update-manifests.outputs.commit-sha }}"
```

## Requirements

Your repository should have a `genifest.yaml` configuration file. See the [genifest documentation](https://github.com/zostay/genifest) for configuration details.

## License

MIT License - see the genifest repository for full licensing details.
