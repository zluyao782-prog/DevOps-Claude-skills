---
name: cicd-pipeline
description: >
  Use this skill whenever the user wants to create, generate, or improve a CI/CD pipeline or workflow.
  Triggers include: any mention of "pipeline", "CI/CD", "GitHub Actions", "GitLab CI", "Jenkins",
  "流水线", "持续集成", "持续部署", "自动化部署", or requests to automate testing/building/deploying.
  Also trigger when the user shows a Dockerfile, k8s manifest, or app code and asks how to deploy it automatically.
  Do NOT use for pure infrastructure provisioning without a pipeline context — use terraform-module instead.
---

# CI/CD Pipeline Generator

## Overview

This skill generates production-grade CI/CD pipeline configurations. It doesn't just produce
syntactically valid YAML — it encodes caching strategies, security scanning, secret management,
multi-environment promotion, and rollback logic that take engineers months to get right.

## Quick Reference

| Platform | File location | Trigger phrase |
|----------|--------------|----------------|
| GitHub Actions | `.github/workflows/*.yml` | "GitHub Actions", "GHA" |
| GitLab CI | `.gitlab-ci.yml` | "GitLab CI", "GitLab" |
| Jenkins | `Jenkinsfile` | "Jenkins", "Jenkinsfile" |
| CircleCI | `.circleci/config.yml` | "CircleCI" |

---

## Workflow: Generating a Pipeline

### Step 1 — Gather Context

Before generating anything, extract or ask for:

**Required:**
- Language / framework (Go, Node.js, Python, Java/Maven, Java/Gradle, Rust...)
- Target deployment platform (EKS, GKE, AKS, bare-metal k8s, EC2, Lambda, Cloud Run, Heroku...)
- CI platform (GitHub Actions, GitLab CI, Jenkins, CircleCI, Tekton...)

**Infer from context if possible, ask only if ambiguous:**
- Deployment strategy (rolling / blue-green / canary) — default: rolling
- Number of environments (dev / staging / prod) — default: 2
- Container registry (ECR, GCR, Docker Hub, GHCR) — infer from cloud provider
- Test framework — infer from language

**Optional enhancements (offer proactively):**
- Security scanning (Trivy, Snyk, SAST)
- Slack/PagerDuty notifications
- Manual approval gate before prod
- Cache warming strategy

### Step 2 — Select the Right Template

Use the matrix below to pick the base template:

| Language | Build tool | Base template |
|----------|-----------|---------------|
| Go | `go build` | [Go template](#go-github-actions) |
| Node.js | npm / yarn / pnpm | [Node template](#nodejs-github-actions) |
| Python | pip / poetry | [Python template](#python-github-actions) |
| Java | Maven | [Java/Maven template](#java-github-actions) |
| Java | Gradle | Same as Maven, swap commands |
| Rust | cargo | [Rust template](#rust-github-actions) |
| Any | Docker-only | [Docker template](#docker-only) |

### Step 3 — Generate with Best Practices Injected

Every generated pipeline MUST include:

**Performance:**
- Dependency caching keyed on lockfile hash (`package-lock.json`, `go.sum`, `pom.xml`, etc.)
- Docker layer caching (use `cache-from`/`cache-to` with registry or GHA cache)
- Parallel jobs where possible (lint + test can run concurrently)

**Security:**
- NEVER hardcode secrets — use platform secret stores
- Use OIDC for cloud auth (no long-lived credentials in secrets)
- Pin action versions to commit SHAs, not floating tags (`actions/checkout@v4` → `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`)
- Container image scan before push (Trivy recommended)

**Reliability:**
- Rollback step triggered on deploy failure
- Health check poll after deploy (retry with backoff)
- Timeout on every job (prevent hung pipelines)
- `fail-fast: false` on matrix builds unless explicitly desired

**Observability:**
- Job summary output with test results and image digest
- Deployment annotation to monitoring system (optional, offer it)

### Step 4 — Deliver

Output:
1. The pipeline YAML file(s) with inline comments explaining non-obvious decisions
2. A short "what this does" summary (5-10 lines)
3. Required secrets/env vars list with setup instructions
4. Any follow-up recommendations (e.g., "add branch protection rules", "enable Dependabot")

---

## Templates

### Go — GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  GO_VERSION: "1.22"

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - uses: golangci/golangci-lint-action@v4
        with:
          version: latest

  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - name: Run tests
        run: go test -v -race -coverprofile=coverage.out ./...
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: coverage.out

  build-push:
    name: Build & Push Image
    runs-on: ubuntu-latest
    needs: [lint, test]
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
      id-token: write   # OIDC for cloud auth
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: build-push
    steps:
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [build-push, security-scan]
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_STAGING }}
          aws-region: us-east-1
      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name staging-cluster --region us-east-1
      - name: Deploy
        run: |
          kubectl set image deployment/APP_NAME \
            app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --namespace=staging
          kubectl rollout status deployment/APP_NAME --namespace=staging --timeout=5m
      - name: Rollback on failure
        if: failure()
        run: kubectl rollout undo deployment/APP_NAME --namespace=staging

  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production   # Requires manual approval in GitHub Environment settings
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_PROD }}
          aws-region: us-east-1
      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name prod-cluster --region us-east-1
      - name: Deploy
        run: |
          kubectl set image deployment/APP_NAME \
            app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --namespace=production
          kubectl rollout status deployment/APP_NAME --namespace=production --timeout=10m
      - name: Rollback on failure
        if: failure()
        run: kubectl rollout undo deployment/APP_NAME --namespace=production
```

### Node.js — GitHub Actions

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:

env:
  NODE_VERSION: "20"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint-test:
    name: Lint & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm          # or 'yarn' / 'pnpm'
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage

  build-push:
    name: Build & Push Image
    runs-on: ubuntu-latest
    needs: lint-test
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        id: build
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            NODE_VERSION=${{ env.NODE_VERSION }}
```

### GitLab CI — Multi-stage

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - build
  - scan
  - deploy-staging
  - deploy-prod

variables:
  DOCKER_DRIVER: overlay2
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

default:
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY

lint:
  stage: lint
  image: golangci/golangci-lint:latest   # Adjust for your language
  script:
    - golangci-lint run ./...
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

test:
  stage: test
  image: golang:1.22
  cache:
    key: go-modules-$CI_COMMIT_REF_SLUG
    paths:
      - .go/pkg/mod/
  script:
    - go test -race -coverprofile=coverage.out ./...
    - go tool cover -func=coverage.out
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

build:
  stage: build
  script:
    - docker build --cache-from $CI_REGISTRY_IMAGE:latest -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

security-scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE_TAG
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy-staging:
  stage: deploy-staging
  image: bitnami/kubectl:latest
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - kubectl config use-context staging
    - kubectl set image deployment/app app=$IMAGE_TAG -n staging
    - kubectl rollout status deployment/app -n staging --timeout=5m
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy-prod:
  stage: deploy-prod
  image: bitnami/kubectl:latest
  environment:
    name: production
    url: https://example.com
  script:
    - kubectl config use-context production
    - kubectl set image deployment/app app=$IMAGE_TAG -n production
    - kubectl rollout status deployment/app -n production --timeout=10m
  when: manual           # Require explicit click in GitLab UI
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

## Common Customizations

### Blue-Green Deployment (GitHub Actions + K8s)

```yaml
- name: Blue-green deploy
  run: |
    CURRENT=$(kubectl get svc app -o jsonpath='{.spec.selector.slot}' -n production)
    NEW=$([ "$CURRENT" = "blue" ] && echo "green" || echo "blue")
    
    # Deploy to inactive slot
    kubectl set image deployment/app-$NEW app=$IMAGE -n production
    kubectl rollout status deployment/app-$NEW -n production --timeout=5m
    
    # Switch traffic
    kubectl patch svc app -p "{\"spec\":{\"selector\":{\"slot\":\"$NEW\"}}}" -n production
    echo "Switched traffic from $CURRENT to $NEW"
```

### Canary Deployment (percentage-based)

```yaml
- name: Canary deploy (10%)
  run: |
    # Scale canary to 10% of total desired replicas
    TOTAL=$(kubectl get deploy app -o jsonpath='{.spec.replicas}' -n production)
    CANARY=$((TOTAL / 10))
    
    kubectl set image deployment/app-canary app=$IMAGE -n production
    kubectl scale deployment/app-canary --replicas=$CANARY -n production
    kubectl rollout status deployment/app-canary -n production --timeout=5m
```

### Slack Notification on Failure

```yaml
- name: Notify on failure
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "❌ Deploy failed: ${{ github.repository }} @ ${{ github.sha }}",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "❌ *Deploy failed*\nRepo: ${{ github.repository }}\nCommit: `${{ github.sha }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View run>"
            }
          }
        ]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Required Secrets Reference

After generating a pipeline, always output a table like this:

| Secret name | Where to get it | Required for |
|-------------|----------------|-------------|
| `AWS_ROLE_ARN_STAGING` | AWS IAM → Roles → your OIDC role ARN | EKS deploy (staging) |
| `AWS_ROLE_ARN_PROD` | AWS IAM → Roles → your OIDC role ARN | EKS deploy (prod) |
| `SLACK_WEBHOOK_URL` | Slack API → Incoming Webhooks | Failure notifications |

---

## Troubleshooting

**"Error: Resource not accessible by integration"**
→ Check that `permissions` block includes the right scopes. For GHCR push, add `packages: write`.

**Docker layer cache not working**
→ Ensure `cache-from` references the correct registry path. With `type=gha`, check the cache size limit (default 10GB, configurable).

**Rollout status times out**
→ Increase `--timeout` OR check pod events: `kubectl describe pod -l app=NAME -n NAMESPACE`

**OIDC auth fails**
→ Verify the trust relationship in the IAM role allows `token.actions.githubusercontent.com` as the OIDC provider, with the correct subject condition (`repo:ORG/REPO:ref:refs/heads/main`).
