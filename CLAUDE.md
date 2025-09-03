# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a GitHub Action for deploying Docker images to AWS ECS (Elastic Container Service) with real-time deployment monitoring. It's a composite action that automates the entire deployment pipeline: building Docker images, pushing to ECR, updating ECS task definitions, and monitoring the deployment status.

## Key Architecture Components

### Action Structure
- **Composite Action**: Uses `runs.using: composite` with multiple steps
- **Main Steps**:
  1. Docker Buildx setup
  2. AWS credentials configuration via OIDC
  3. ECR login
  4. Docker build and push
  5. ECS task definition update
  6. Deployment monitoring with real-time status updates

### Core Technologies
- **AWS Services**: ECS, ECR, IAM (with OIDC)
- **Tools**: Docker, AWS CLI, jq for JSON processing
- **CI/CD**: GitHub Actions composite action

## Common Development Commands

### Testing the Action
```bash
# Validate action.yml syntax
yq eval '.' action.yml

# Check example workflows
for example in examples/*.yml; do
  yq eval '.' "$example"
done

# Extract and check bash scripts from action.yml
yq eval '.runs.steps[] | select(.shell == "bash") | .run' action.yml > /tmp/scripts.sh
shellcheck -S warning /tmp/scripts.sh
```

### Releasing a New Version
```bash
# Create and push a semantic version tag
git tag v1.2.3
git push origin v1.2.3

# The release workflow automatically:
# - Creates GitHub release with generated notes
# - Updates major version tag (v1 → v1.2.3)
# - Updates minor version tag (v1.2 → v1.2.3)
# - Updates README examples
```

## Critical Implementation Details

### Deployment Monitoring Logic
The monitoring step (lines 149-292 in action.yml) implements comprehensive deployment tracking:
- Polls ECS service status every 5 seconds with detailed table output
- Displays for each deployment:
  - Task definition revision number
  - Deployment status (PRIMARY/ACTIVE)
  - Rollout state (IN_PROGRESS/COMPLETED)
  - Desired, Running, Pending, and Failed task counts
  - Health percentage (running/desired * 100)
- Shows inline error messages when tasks fail or become unhealthy
- Detects successful completion when `rolloutState` is `COMPLETED`
- Detects rollback when new task definition matches the previous one
- Implements configurable timeout (default 600s)

### Task Definition Update Pattern
- Downloads current task definition using AWS CLI
- Strips metadata fields (compatibilities, ARN, revision, etc.)
- Updates container image using jq
- Registers new task definition
- Forces new deployment on the service

### Error Detection
The action detects deployment failures by checking if:
- The service rolls back to the previous task definition
- The deployment times out
- The new deployment never becomes PRIMARY

## Commit Conventions

Use conventional commits for automatic release notes:
- `feat:` or `feature:` - New features
- `fix:` - Bug fixes
- `docs:` - Documentation changes
- `chore:` or `ci:` - Maintenance tasks

## Important Constraints

- The action requires GitHub OIDC authentication (no AWS access keys)
- All bash scripts run with `set -e` implied (exit on error)
- Output is limited to avoid GitHub Actions log size limits
- The action uses GitHub's hosted Docker actions for reliability