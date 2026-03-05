# GitHub Actions Deployment Guide

This document describes the GitHub Actions workflows set up for this project.

## Workflows Overview

### 1. Build and Test (`build-and-test.yml`)
- **Trigger**: Push to `main` or `develop` branches, Pull requests
- **Jobs**:
  - `test-client`: Runs client tests and builds React app
  - `test-server`: Tests server with PostgreSQL and Redis services
  - `test-worker`: Tests worker with Redis service
  - `build-docker-images`: Builds Docker images (only on push)

### 2. Deploy (`deploy.yml`)
- **Trigger**: Push to `main` branch with changes to service code
- **Jobs**:
  - `build-and-push`: Builds and pushes Docker images to Docker Hub
  - `update-kubernetes`: Updates k8s manifests with new image references
  - `deploy-to-cluster`: Deploys to Kubernetes cluster (optional, requires setup)

### 3. Security Scan (`security-scan.yml`)
- **Trigger**: Push/PR to `main`/`develop`, daily schedule
- **Jobs**:
  - `trivy-scan`: Scans Docker images for vulnerabilities
  - `dependency-check`: Runs npm audit on dependencies

### 4. Lint (`lint.yml`)
- **Trigger**: Push/PR to `main`/`develop`
- **Jobs**:
  - `lint-yaml`: Validates Kubernetes YAML files
  - `lint-dockerfiles`: Lints all Dockerfiles
  - `lint-client`: Runs ESLint on React code

## Setup Instructions

### Prerequisites
1. GitHub repository with this code
2. Docker Hub account (for image publishing)
3. (Optional) Kubernetes cluster for deployment

### Required Secrets

Add these secrets to your GitHub repository settings (`Settings > Secrets and variables > Actions`):

#### For Docker Hub Publishing
- `DOCKER_USERNAME`: Your Docker Hub username
- `DOCKER_PASSWORD`: Your Docker Hub personal access token (not password)

#### For Kubernetes Deployment (Optional)
- `KUBE_CONFIG`: Base64-encoded kubeconfig file
  ```bash
  # Generate base64-encoded kubeconfig
  cat ~/.kube/config | base64 -w 0 | pbcopy
  # Paste into GitHub secrets
  ```

### Step-by-Step Setup

1. **Create GitHub Secrets**:
   - Go to your repository → Settings → Secrets and variables → Actions
   - Click "New repository secret"
   - Add `DOCKER_USERNAME` and `DOCKER_PASSWORD`

2. **Optional: Configure Kubernetes Deployment**:
   - If you have a Kubernetes cluster, add `KUBE_CONFIG` secret
   - Update the `deploy-to-cluster` job in `deploy.yml` if using a different namespace

3. **Run Tests**:
   - Create a pull request to trigger `build-and-test.yml`
   - Verify all tests pass

4. **Deploy to Production**:
   - Merge to `main` branch
   - `deploy.yml` workflow automatically builds and pushes images
   - Images are tagged as `latest` and git SHA

## Modifying Workflows

### Change Docker Registry
To use GitHub Container Registry instead of Docker Hub:

1. Update `deploy.yml`:
   ```yaml
   - name: Log in to GitHub Container Registry
     uses: docker/login-action@v2
     with:
       registry: ghcr.io
       username: ${{ github.actor }}
       password: ${{ secrets.GITHUB_TOKEN }}
   ```

2. Update image references:
   ```yaml
   images: ghcr.io/${{ github.repository_owner }}/multi-${{ matrix.service.name }}
   ```

### Add Manual Deployment Trigger
To allow manual deployments, add workflow_dispatch:

```yaml
on:
  workflow_dispatch:
  push:
    branches: [main]
```

### Deploy to Different Namespace
In `deploy.yml`, add namespace flag:

```yaml
- name: Deploy to Kubernetes
  run: |
    kubectl apply -n your-namespace -f k8s/
```

## Troubleshooting

### Docker Push Fails
- Verify `DOCKER_USERNAME` and `DOCKER_PASSWORD` secrets are correct
- Ensure Docker Hub token has push permissions
- Check Docker Hub account is not rate-limited

### Kubernetes Deployment Fails
- Verify kubeconfig is correctly base64-encoded
- Check cluster connectivity: `kubectl cluster-info`
- Ensure service accounts have required permissions
- Check pod logs: `kubectl logs -f deployment/client-deployment`

### Build Failures
- Check Docker build logs in GitHub Actions
- Ensure all dependencies are listed in package.json
- Verify Dockerfile syntax with hadolint

### Test Failures
- View test output in GitHub Actions logs
- Run tests locally: `cd client && npm test`
- Check service connectivity in test environment

## Viewing Workflow Results

1. Go to repository → Actions tab
2. Click on workflow name (Build and Test, Deploy, etc.)
3. Click on the latest run
4. Expand job logs to see detailed output
5. Download artifacts if available (audit reports, etc.)

## Performance Tips

1. **Cache Dependencies**:
   - Workflows use npm cache for faster builds
   - Docker layer caching is enabled

2. **Parallel Execution**:
   - Test jobs run in parallel
   - Docker image builds run in parallel matrix

3. **Conditional Steps**:
   - Docker image build only on push (not PR)
   - Deployment only on main branch

## Security Best Practices

1. **Secrets Management**:
   - Never commit secrets to repository
   - Use organization secrets for shared values
   - Rotate tokens regularly

2. **Image Scanning**:
   - Trivy scans images for vulnerabilities
   - Fix high/critical vulnerabilities before merging

3. **Dependency Updates**:
   - npm audit runs on every build
   - Consider Dependabot for automated updates

4. **Access Control**:
   - Limit deployment triggers to main branch
   - Use OpenID Connect for cloud credential federation

## Next Steps

1. Test workflows by creating a pull request
2. Verify Docker images push to Docker Hub
3. (Optional) Configure Kubernetes deployment
4. Monitor workflows in GitHub Actions tab
5. Set up branch protection rules requiring all checks to pass
