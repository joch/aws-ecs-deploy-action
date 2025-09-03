# AWS ECS Deploy with Monitoring

The missing GitHub Action for ECS deployments - **actually shows you what's happening** during deployment.

## Why This Action?

- 🔄 **Shows real-time progress** - Watch containers spin up/down
- ❌ **Detects failures** - Knows when ECS rolls back your deployment  
- ⏱️ **Respects your time** - Configurable timeouts, no hanging builds
- 📦 **All-in-one** - Build, push to ECR, deploy to ECS, monitor status

## Quick Start

```yaml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: joch/aws-ecs-deploy-action@v1
        with:
          # AWS Configuration
          aws-assume-role-id: ${{ vars.AWS_ASSUMEROLE_ID }}
          aws-region: eu-north-1
          
          # ECR Configuration
          ecr-registry: 123456789.dkr.ecr.eu-north-1.amazonaws.com
          ecr-repository: my-app
          
          # ECS Configuration
          ecs-cluster: production
          ecs-service: my-app-service
          ecs-task-definition: my-app-task
          container-name: my-app
```

## What You'll See

You will get real-time updates:

```
📦 Deploying 123456789.dkr.ecr.eu-north-1.amazonaws.com/my-app:abc123 to production/my-app-service
⏳ Monitoring deployment (timeout: 600s)...

⏱️  [ 15s] OLD: 2/2 running | NEW: 0/2 running
⏱️  [ 30s] OLD: 2/2 running | NEW: 1/2 running
⏱️  [ 45s] OLD: 1/2 running | NEW: 2/2 running
⏱️  [ 60s] NEW: 2/2 running

✅ Deployment completed successfully!
```

Or if something goes wrong:

```
⏱️  [ 45s] OLD: 2/2 running | NEW: 0/2 running
⏱️  [ 60s] OLD: 2/2 running | NEW: 0/2 running

❌ Deployment failed - service rolled back

Recent events:
  - (service my-app-service) failed to launch task: container health check failed
  - (service my-app-service) rolling back to previous task definition
```

## Inputs

### Required

| Input | Description |
|-------|-------------|
| `aws-assume-role-id` | AWS IAM role to assume (with OIDC) |
| `aws-region` | AWS region |
| `ecr-registry` | ECR registry URL |
| `ecr-repository` | ECR repository name |
| `ecs-cluster` | ECS cluster name |
| `ecs-service` | ECS service name |
| `ecs-task-definition` | Task definition family name |
| `container-name` | Container name to update in task definition |

### Optional

| Input | Description | Default |
|-------|-------------|---------|
| `dockerfile` | Path to Dockerfile | `Dockerfile` |
| `build-context` | Docker build context | `.` |
| `build-args` | Docker build arguments (multiline) | - |
| `timeout` | Deployment timeout in seconds | `600` |

## Outputs

| Output | Description |
|--------|-------------|
| `image` | Docker image URI that was deployed |
| `task-definition-arn` | ARN of the new task definition |
| `deployment-status` | `success`, `failed`, or `timeout` |

## Examples

### With Build Arguments

```yaml
- uses: your-org/aws-ecs-deploy-action@v1
  with:
    aws-assume-role-id: ${{ vars.AWS_ASSUMEROLE_ID }}
    aws-region: us-east-1
    ecr-registry: ${{ vars.ECR_REGISTRY }}
    ecr-repository: my-app
    ecs-cluster: production
    ecs-service: my-app-service
    ecs-task-definition: my-app
    container-name: app
    build-args: |
      API_KEY=${{ secrets.API_KEY }}
      ENVIRONMENT=production
```

### Custom Dockerfile

```yaml
- uses: your-org/aws-ecs-deploy-action@v1
  with:
    aws-assume-role-id: ${{ vars.AWS_ASSUMEROLE_ID }}
    aws-region: us-east-1
    ecr-registry: ${{ vars.ECR_REGISTRY }}
    ecr-repository: backend
    ecs-cluster: production
    ecs-service: backend-service
    ecs-task-definition: backend
    container-name: api
    dockerfile: docker/Dockerfile.prod
    build-context: ./backend
```

## Prerequisites

### AWS IAM Role

Create an IAM role with OIDC trust relationship:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

The role needs permissions for:
- ECR: Push images
- ECS: Update services and task definitions
- IAM: Pass role (for task execution)

### ECS Setup

You need:
- An existing ECS cluster
- An ECS service
- A task definition with a container matching `container-name`

## Migration Guide

Replace your complex deployment workflow:

```yaml
# BEFORE: 100+ lines of bash scripts
jobs:
  build:
    # ... docker setup, build, push ...
  deploy:
    # ... download task def, update, deploy ...
  monitor:
    # ... custom monitoring logic ...

# AFTER: Just use this action
jobs:
  deploy:
    steps:
      - uses: actions/checkout@v4
      - uses: your-org/aws-ecs-deploy-action@v1
        with:
          # ... 8 required inputs ...
```

## Development

### Testing

The action includes automated tests that run on every push and PR:

```bash
# Validate action.yml syntax
yq eval '.' action.yml

# Check example workflows
for example in examples/*.yml; do
  yq eval '.' "$example"
done
```

### Releasing

This action uses semantic versioning and automated releases. To release a new version:

1. **Create a new tag** following semantic versioning:
   ```bash
   git tag v1.2.3
   git push origin v1.2.3
   ```

2. **Automatic tag management**: The release workflow automatically:
   - Creates a GitHub release with generated release notes
   - Updates the major version tag (e.g., `v1` → points to `v1.2.3`)
   - Updates the minor version tag (e.g., `v1.2` → points to `v1.2.3`)
   - Updates README examples to use the latest major version

3. **Version tags available for users**:
   - `@v1.2.3` - Exact version (most stable)
   - `@v1.2` - Latest patch of minor version
   - `@v1` - Latest minor/patch of major version (recommended)
   - `@main` - Latest development version (unstable)

### Commit Convention

For automatic release notes generation, use conventional commits:
- `feat:` or `feature:` - New features
- `fix:` - Bug fixes
- `docs:` - Documentation changes
- `chore:` or `ci:` - Maintenance tasks

Example:
```bash
git commit -m "feat: add support for custom docker registries"
git commit -m "fix: handle ECS service names with hyphens"
git commit -m "docs: improve IAM role setup instructions"
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests if applicable
5. Submit a pull request

## License

MIT