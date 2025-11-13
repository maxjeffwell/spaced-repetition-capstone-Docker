# CI/CD Pipeline Guide

This guide covers the Continuous Integration and Continuous Deployment (CI/CD) pipeline for the Spaced Repetition Capstone application using GitHub Actions.

## Table of Contents

- [Overview](#overview)
- [GitHub Actions Workflows](#github-actions-workflows)
- [Setup Instructions](#setup-instructions)
- [Docker Hub Integration](#docker-hub-integration)
- [Workflow Details](#workflow-details)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

## Overview

### CI/CD Architecture

```
┌─────────────────────────────────────────────────┐
│            Developer Push/PR                     │
└──────────────────┬──────────────────────────────┘
                   │
         ┌─────────▼─────────┐
         │   GitHub Actions   │
         └─────────┬──────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
   ┌────▼────┐          ┌────▼────┐
   │   CI    │          │  Build  │
   │Workflow │          │   &     │
   └────┬────┘          │  Push  │
        │               └────┬────┘
   ┌────▼────┐              │
   │ - Lint  │         ┌────▼────┐
   │ - Test  │         │ Docker  │
   │ - Build │         │  Hub    │
   └─────────┘         └────┬────┘
                            │
                       ┌────▼────┐
                       │  Trivy  │
                       │Security │
                       │ Scan    │
                       └────┬────┘
                            │
                       ┌────▼────┐
                       │ GitHub  │
                       │Security │
                       │  Tab    │
                       └─────────┘
```

### Repositories

- **Server**: `maxjeffwell/spaced-repetition-capstone-server`
- **Client**: `maxjeffwell/spaced-repetition-capstone-client`
- **Docker Hub**:
  - `maxjeffwell/spaced-repetition-capstone-server:latest`
  - `maxjeffwell/spaced-repetition-capstone-client:latest`

## GitHub Actions Workflows

Both server and client repositories have two workflows:

### 1. CI Workflow (`.github/workflows/ci.yml`)

**Triggers:**
- Push to `master`, `main`, or `develop` branches
- Pull requests to these branches

**Actions:**
- Checkout code
- Setup Node.js (matrix: 18.x, 20.x)
- Install dependencies
- Run linter
- Run tests
- Build Docker image (verification only, no push)

### 2. Docker Build & Push Workflow (`.github/workflows/docker-build-push.yml`)

**Triggers:**
- Push to `master` or `main` branches
- Version tags (v*)
- Manual dispatch

**Actions:**
- Checkout code
- Setup QEMU and Docker Buildx
- Login to Docker Hub
- Extract metadata (tags)
- Build multi-platform images (amd64, arm64)
- Push to Docker Hub
- Run Trivy security scan
- Upload security results to GitHub

## Setup Instructions

### 1. Fork/Clone Repositories

```bash
# Clone both repositories
git clone git@github.com:maxjeffwell/spaced-repetition-capstone-server.git
git clone git@github.com:maxjeffwell/spaced-repetition-capstone-client.git
```

### 2. Create Docker Hub Account

1. Sign up at https://hub.docker.com
2. Create repositories:
   - `your-username/spaced-repetition-capstone-server`
   - `your-username/spaced-repetition-capstone-client`

### 3. Generate Docker Hub Access Token

```bash
# Login to Docker Hub
# Navigate to: Account Settings → Security → Access Tokens
# Click "New Access Token"
# Name: github-actions-spaced-repetition
# Permissions: Read, Write, Delete
# Copy the token (you'll only see it once!)
```

### 4. Configure GitHub Secrets

#### Server Repository

```bash
# Navigate to: Settings → Secrets and variables → Actions → New repository secret

# Add two secrets:
DOCKERHUB_USERNAME: your-dockerhub-username
DOCKERHUB_TOKEN: <paste-token-from-step-3>
```

#### Client Repository

```bash
# Same as server - add the same two secrets:
DOCKERHUB_USERNAME: your-dockerhub-username
DOCKERHUB_TOKEN: <paste-token-from-step-3>
```

### 5. Update Workflow Files (if needed)

If using different Docker Hub username, update workflow files:

**Server: `.github/workflows/docker-build-push.yml`**
```yaml
env:
  DOCKER_IMAGE: your-username/spaced-repetition-capstone-server
```

**Client: `.github/workflows/docker-build-push.yml`**
```yaml
env:
  DOCKER_IMAGE: your-username/spaced-repetition-capstone-client
```

### 6. Test Workflows

```bash
# Commit and push to trigger workflows
git add .
git commit -m "Configure CI/CD pipeline"
git push origin master

# Check Actions tab on GitHub
# https://github.com/your-username/spaced-repetition-capstone-server/actions
# https://github.com/your-username/spaced-repetition-capstone-client/actions
```

## Docker Hub Integration

### Image Tagging Strategy

Workflows automatically tag images based on trigger:

| Trigger | Tags Generated |
|---------|----------------|
| Push to `master` | `latest` |
| Push to `develop` | `develop` |
| Version tag `v1.2.3` | `1.2.3`, `1.2`, `1`, `latest` |
| Commit on `master` | `master-abc1234` (SHA) |

### Multi-Platform Builds

Images are built for:
- `linux/amd64` (Intel/AMD)
- `linux/arm64` (ARM, M1/M2 Macs, Raspberry Pi)

### Cache Optimization

Workflows use Docker BuildKit cache:
```yaml
cache-from: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache
cache-to: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache,mode=max
```

This speeds up builds by reusing layers.

## Workflow Details

### CI Workflow Breakdown

**Server CI (`.github/workflows/ci.yml`):**

```yaml
strategy:
  matrix:
    node-version: [18.x, 20.x]  # Test on multiple Node versions

steps:
  - Checkout code
  - Setup Node.js
  - npm ci                       # Clean install
  - npm run lint                 # ESLint check (continues on error)
  - npm test                     # Run tests (continues on error)
  - Build Docker image           # Verify Dockerfile works
  - Verify image exists          # Ensure build succeeded
```

**Client CI (`.github/workflows/ci.yml`):**

```yaml
strategy:
  matrix:
    node-version: [18.x, 20.x]

steps:
  - Checkout code
  - Setup Node.js
  - npm ci
  - npm run lint                 # ESLint check
  - npm test                     # Jest tests with CRA
  - npm run build                # Production build
  - Build Docker image
  - Verify image exists
```

### Docker Build & Push Workflow Breakdown

**Server Build (`.github/workflows/docker-build-push.yml`):**

```yaml
permissions:
  contents: read               # Read repo
  packages: write              # Write to GHCR (optional)
  security-events: write       # Upload Trivy results

steps:
  - Checkout code
  - Setup QEMU                 # Multi-platform emulation
  - Setup Docker Buildx        # Advanced builder
  - Login to Docker Hub        # Authenticate
  - Extract metadata           # Generate tags
  - Build and push image       # Multi-platform build
  - Run Trivy scanner          # Security scan
  - Upload to GitHub Security  # SARIF results
```

**Client Build (`.github/workflows/docker-build-push.yml`):**

Similar to server, with additional:

```yaml
build-args: |
  REACT_APP_API_BASE_URL=/api  # Build-time env var
```

## Troubleshooting

### Common Issues

#### 1. Authentication Failed

**Error:**
```
Error: Cannot perform an interactive login from a non TTY device
```

**Solution:**
```bash
# Verify secrets are set correctly
# GitHub → Repository → Settings → Secrets and variables → Actions

# Secrets should be:
DOCKERHUB_USERNAME: maxjeffwell
DOCKERHUB_TOKEN: dckr_pat_xxxxxxxxxxxxx
```

#### 2. Image Push Failed

**Error:**
```
denied: requested access to the resource is denied
```

**Solution:**
```bash
# Verify Docker Hub repository exists
# Create at: hub.docker.com → Repositories → Create Repository

# Verify token has Write permissions
# Docker Hub → Account Settings → Security → Access Tokens
```

#### 3. Tests Failing

**Error:**
```
npm test exited with code 1
```

**Solution:**
```bash
# Tests are set to continue-on-error, so won't block
# Fix tests locally, then push

# Run tests locally
npm test

# Check specific test files
npm test -- --verbose
```

#### 4. Build Timeout

**Error:**
```
The job running on runner GitHub Actions X has exceeded the maximum execution time of 360 minutes
```

**Solution:**
```bash
# Multi-platform builds can be slow
# Consider building only amd64 for development

# In workflow file:
platforms: linux/amd64  # Remove arm64 temporarily
```

#### 5. Trivy Scan Failures

**Error:**
```
Error: failed to scan image
```

**Solution:**
```yaml
# Already set to continue-on-error
continue-on-error: true

# View security issues:
# GitHub → Repository → Security → Code scanning alerts
```

### Debugging Workflows

```bash
# View workflow runs
https://github.com/maxjeffwell/spaced-repetition-capstone-server/actions

# Check specific run
# Click on workflow → Click on job → Expand steps

# Re-run failed jobs
# Click "Re-run failed jobs" button

# Enable debug logging
# Repository Settings → Secrets → Add:
ACTIONS_RUNNER_DEBUG: true
ACTIONS_STEP_DEBUG: true
```

## Best Practices

### 1. Branch Protection

```bash
# Settings → Branches → Add rule for master/main

# Require:
- Status checks to pass before merging
- Branches to be up to date before merging
- CI workflow must pass
```

### 2. Version Tagging

```bash
# Create version tags for releases
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0

# This triggers workflow and creates:
# - maxjeffwell/spaced-repetition-capstone-server:1.0.0
# - maxjeffwell/spaced-repetition-capstone-server:1.0
# - maxjeffwell/spaced-repetition-capstone-server:1
# - maxjeffwell/spaced-repetition-capstone-server:latest
```

### 3. Security Scanning

```bash
# View security scan results
# GitHub → Repository → Security → Code scanning

# Trivy scans for:
- Known CVEs in dependencies
- Misconfigurations
- Secrets in images
- License issues
```

### 4. Workflow Optimization

```yaml
# Cache npm dependencies
- uses: actions/setup-node@v4
  with:
    cache: 'npm'

# Use BuildKit cache for Docker
cache-from: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache
cache-to: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache,mode=max

# Run jobs in parallel when possible
jobs:
  test:
    ...
  lint:
    ...
  # These run simultaneously
```

### 5. Monitoring

```bash
# Set up notifications
# Settings → Notifications → Actions

# Email notifications for:
- Workflow failures
- Workflow successes (optional)

# Slack/Discord integration (optional)
# Use GitHub webhooks
```

## Manual Deployment

### Build and Push Manually

```bash
# Server
cd spaced-repetition-capstone-server
docker buildx build --platform linux/amd64,linux/arm64 \
  -t maxjeffwell/spaced-repetition-capstone-server:v1.0.0 \
  --push .

# Client
cd spaced-repetition-capstone-client
docker buildx build --platform linux/amd64,linux/arm64 \
  --build-arg REACT_APP_API_BASE_URL=/api \
  -t maxjeffwell/spaced-repetition-capstone-client:v1.0.0 \
  --push .
```

### Trigger Workflow Manually

```bash
# Via GitHub UI
# Actions → Workflow name → Run workflow → Select branch

# Via gh CLI
gh workflow run docker-build-push.yml
```

## Integration with Kubernetes

### Automatic Updates

Set up Kubernetes to automatically pull latest images:

```yaml
# In deployment spec:
spec:
  template:
    spec:
      containers:
      - name: server
        image: maxjeffwell/spaced-repetition-capstone-server:latest
        imagePullPolicy: Always  # Always pull latest
```

### Using Specific Versions

```bash
# Update to specific version
kubectl set image deployment/spaced-repetition-server \
  server=maxjeffwell/spaced-repetition-capstone-server:v1.0.0 \
  -n spaced-repetition
```

## Status Badges

Add build status badges to README.md:

```markdown
[![CI](https://github.com/maxjeffwell/spaced-repetition-capstone-server/actions/workflows/ci.yml/badge.svg)](https://github.com/maxjeffwell/spaced-repetition-capstone-server/actions/workflows/ci.yml)

[![Docker Build & Push](https://github.com/maxjeffwell/spaced-repetition-capstone-server/actions/workflows/docker-build-push.yml/badge.svg)](https://github.com/maxjeffwell/spaced-repetition-capstone-server/actions/workflows/docker-build-push.yml)
```

## Next Steps

- [Docker Guide](DOCKER.md) - Local development
- [Kubernetes Guide](KUBERNETES.md) - Production deployment
- [README](README.md) - Project overview

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Docker BuildX Documentation](https://docs.docker.com/buildx/working-with-buildx/)
- [Trivy Security Scanner](https://github.com/aquasecurity/trivy)
- [Docker Hub](https://hub.docker.com)
