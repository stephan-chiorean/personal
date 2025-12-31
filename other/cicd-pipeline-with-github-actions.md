---
id: cicd-pipeline-with-github-actions
alias: CI/CD Pipeline with GitHub Actions
type: kit
is_base: false
version: 1
tags:
  - foundation
  - infrastructure
  - cicd
description: Complete CI/CD pipeline with GitHub Actions including automated testing, building, security scanning, and deployment to multiple environments
---

# CI/CD Pipeline with GitHub Actions Kit

A comprehensive kit for implementing continuous integration and continuous deployment using GitHub Actions, including automated testing, building, security scanning, and multi-environment deployments.

## End State

After applying this kit, the application will have:

**CI/CD Pipeline:**
- Automated testing on every pull request
- Automated builds and Docker image creation
- Security scanning (dependencies, code, containers)
- Automated deployment to staging on merge to main
- Manual approval gates for production deployments
- Rollback capabilities
- Deployment notifications (Slack, email, etc.)

**GitHub Actions Workflows:**
- Pull request workflow (test, lint, security scan)
- Main branch workflow (test, build, deploy to staging)
- Production deployment workflow (with approval)
- Release workflow (tag creation, changelog, deployment)
- Scheduled workflows (dependency updates, cleanup)

**Quality Gates:**
- Unit tests must pass
- Integration tests must pass
- Code coverage thresholds
- Linting and formatting checks
- Security vulnerability scanning
- Performance benchmarks

**Deployment Targets:**
- Staging environment (auto-deploy)
- Production environment (manual approval)
- Preview environments (per pull request)
- Container registry (Docker Hub, ECR, GCR)

## Implementation Principles

- **Fail fast**: Run fast tests first, expensive tests later
- **Parallel execution**: Run independent jobs in parallel
- **Caching**: Cache dependencies and build artifacts
- **Security first**: Scan for vulnerabilities in every pipeline
- **Environment parity**: Keep staging and production similar
- **Immutable artifacts**: Build once, deploy everywhere
- **Rollback ready**: Always have rollback procedures
- **Notifications**: Notify on failures and deployments
- **Secrets management**: Use GitHub Secrets, never commit secrets
- **Idempotent deployments**: Deployments should be safe to run multiple times

## Pattern 1: Pull Request Workflow

Test and validate on every pull request:

```yaml
# .github/workflows/pr.yml
name: Pull Request

on:
  pull_request:
    branches:
      - main
      - develop

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x, 20.x]
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run type check
        run: npm run type-check
      
      - name: Run unit tests
        run: npm run test:unit
        env:
          CI: true
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          flags: unittests
          name: codecov-umbrella

  integration-test:
    name: Integration Tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20.x
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run npm audit
        run: npm audit --audit-level=moderate
      
      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

## Pattern 2: Main Branch Workflow

Build and deploy to staging:

```yaml
# .github/workflows/main.yml
name: Build and Deploy

on:
  push:
    branches:
      - main

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix={{SHORT_SHA}}-
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64

  test:
    name: Run Tests
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20.x
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm run test
      
      - name: Check coverage threshold
        run: npm run test:coverage
        env:
          COVERAGE_THRESHOLD: 80

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [build, test]
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Heroku Staging
        uses: akhileshns/heroku-deploy@v3.12.12
        with:
          heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
          heroku_app_name: ${{ secrets.HEROKU_STAGING_APP }}
          heroku_email: ${{ secrets.HEROKU_EMAIL }}
          healthcheck: https://${{ secrets.HEROKU_STAGING_APP }}.herokuapp.com/health
          check_health: true
          rollback_on_healthcheck_failed: true
      
      - name: Run database migrations
        run: |
          heroku run npm run migrate --app ${{ secrets.HEROKU_STAGING_APP }}
      
      - name: Notify deployment
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Deployed to staging: ${{ github.sha }}'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

## Pattern 3: Production Deployment Workflow

Deploy to production with approval:

```yaml
# .github/workflows/production.yml
name: Production Deployment

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy'
        required: true
        type: string
      skip_tests:
        description: 'Skip tests (not recommended)'
        required: false
        type: boolean
        default: false

jobs:
  approval:
    name: Wait for Approval
    runs-on: ubuntu-latest
    environment:
      name: production
    steps:
      - name: Wait for manual approval
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ github.TOKEN }}
          approvers: team-leads
          minimum-approvals: 2
          issue-title: 'Production Deployment Approval'
          issue-body: 'Approve deployment of version ${{ github.event.inputs.version }}'

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: approval
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.inputs.version }}
      
      - name: Build production image
        run: |
          docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.event.inputs.version }} .
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.event.inputs.version }}
      
      - name: Deploy to Kubernetes
        uses: azure/k8s-deploy@v4
        with:
          manifests: |
            k8s/deployment.yaml
            k8s/service.yaml
            k8s/ingress.yaml
          images: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.event.inputs.version }}
          kubectl-version: 'latest'
      
      - name: Run database migrations
        run: |
          kubectl set image deployment/{{APP_NAME}} migration=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.event.inputs.version }} --namespace production
          kubectl rollout status deployment/{{APP_NAME}} --namespace production
      
      - name: Run smoke tests
        run: |
          npm run test:smoke -- --base-url=https://example.com
      
      - name: Notify deployment
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Deployed to production: ${{ github.event.inputs.version }}'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

## Pattern 4: Release Workflow

Create releases with changelog:

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    name: Create Release
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Generate changelog
        id: changelog
        uses: metcalfc/changelog-generator@v4
        with:
          myToken: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Create Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          body: ${{ steps.changelog.outputs.changelog }}
          draft: false
          prerelease: false
      
      - name: Build and push release image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
      
      - name: Deploy to production
        uses: azure/k8s-deploy@v4
        with:
          manifests: k8s/
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}
```

## Pattern 5: Preview Environments

Deploy preview environments for pull requests:

```yaml
# .github/workflows/preview.yml
name: Preview Environment

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  deploy-preview:
    if: github.event.action != 'closed'
    name: Deploy Preview
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Vercel Preview
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
      
      - name: Comment PR with preview URL
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '🚀 Preview deployed: ${{ steps.deploy.outputs.preview-url }}'
            })

  destroy-preview:
    if: github.event.action == 'closed'
    name: Destroy Preview
    runs-on: ubuntu-latest
    steps:
      - name: Destroy preview environment
        run: |
          # Cleanup preview environment
          kubectl delete namespace preview-${{ github.event.pull_request.number }}
```

## Pattern 6: Scheduled Workflows

Automated maintenance tasks:

```yaml
# .github/workflows/scheduled.yml
name: Scheduled Tasks

on:
  schedule:
    - cron: '0 2 * * 1'  # Every Monday at 2 AM
  workflow_dispatch:

jobs:
  update-dependencies:
    name: Update Dependencies
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20.x
      
      - name: Update dependencies
        run: |
          npm update
          npm audit fix
      
      - name: Create PR
        uses: peter-evans/create-pull-request@v5
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          commit-message: 'chore: update dependencies'
          title: 'Automated dependency update'
          body: 'Automated weekly dependency update'

  cleanup-old-artifacts:
    name: Cleanup Old Artifacts
    runs-on: ubuntu-latest
    steps:
      - name: Delete old artifacts
        uses: geekyeggo/delete-artifact@v4
        with:
          name: |
            build-artifacts
            test-results
          retention-days: 30
```

## Pattern 7: Matrix Builds

Test across multiple platforms and versions:

```yaml
# .github/workflows/matrix.yml
name: Matrix Build

on: [push, pull_request]

jobs:
  test:
    name: Test on ${{ matrix.os }} with Node ${{ matrix.node }}
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [18.x, 20.x]
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
```

## Pattern 8: Caching Dependencies

Optimize build times with caching:

```yaml
# .github/workflows/cache.yml
name: Build with Cache

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Cache node modules
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20.x
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
      
      - name: Cache build artifacts
        uses: actions/cache@v3
        with:
          path: dist
          key: ${{ runner.os }}-build-${{ github.sha }}
```

## Pattern 9: Security Scanning

Comprehensive security scanning:

```yaml
# .github/workflows/security.yml
name: Security Scan

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Run OWASP ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.10.0
        with:
          target: 'https://example.com'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'
```

## Pattern 10: Performance Testing

Run performance benchmarks:

```yaml
# .github/workflows/performance.yml
name: Performance Tests

on:
  push:
    branches:
      - main
  schedule:
    - cron: '0 0 * * 0'  # Weekly

jobs:
  performance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v10
        with:
          urls: |
            https://example.com
            https://example.com/products
          uploadArtifacts: true
          temporaryPublicStorage: true
      
      - name: Run k6 load tests
        uses: grafana/k6-action@v0.3.0
        with:
          filename: tests/load.js
          cloud: true
          cloud-token: ${{ secrets.K6_CLOUD_TOKEN }}
```

## Best Practices

1. **Fail fast**: Run fast tests first, expensive operations later
2. **Parallel execution**: Run independent jobs in parallel
3. **Caching**: Cache dependencies, build artifacts, and Docker layers
4. **Security**: Scan for vulnerabilities in every pipeline
5. **Notifications**: Notify on failures and important events
6. **Secrets**: Use GitHub Secrets, never commit secrets
7. **Idempotent**: Deployments should be safe to run multiple times
8. **Rollback**: Always have rollback procedures ready
9. **Environment parity**: Keep environments as similar as possible
10. **Documentation**: Document deployment procedures and rollback steps

## Common Pitfalls

- **No caching**: Slow builds due to repeated dependency installation
- **Missing tests**: Deploying without running tests
- **Hardcoded secrets**: Committing secrets instead of using GitHub Secrets
- **No rollback plan**: Unable to quickly revert bad deployments
- **Sequential jobs**: Running jobs sequentially when they could be parallel
- **Missing notifications**: Not knowing when deployments fail
- **No security scanning**: Deploying vulnerable code
- **Environment drift**: Staging and production differ significantly

## Verification Criteria

After generation, verify:
- ✓ Tests run on every pull request
- ✓ Builds create Docker images and push to registry
- ✓ Security scans run in every pipeline
- ✓ Staging auto-deploys on merge to main
- ✓ Production requires manual approval
- ✓ Preview environments are created for PRs
- ✓ Rollback procedures are documented and tested
- ✓ Notifications are sent on deployments and failures
- ✓ Dependencies are cached to speed up builds
- ✓ Multiple environments can be deployed independently
- ✓ Release workflow creates tags and changelogs
- ✓ Scheduled tasks run maintenance operations

