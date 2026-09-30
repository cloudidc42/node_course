# Part 41 | ขั้นตอนที่ 721-740 จาก 1000

## CI/CD กับ GitHub Actions - Automated Testing และ Deployment

---

## สารบัญ

1. [CI/CD คืออะไร](#cicd-คืออะไร)
2. [GitHub Actions พื้นฐาน](#github-actions-พื้นฐาน)
3. [Workflow สำหรับ Node.js](#workflow-สำหรับ-nodejs)
4. [Testing Pipeline](#testing-pipeline)
5. [Docker Build และ Push](#docker-build-และ-push)
6. [Deployment Workflows](#deployment-workflows)
7. [Environment Secrets](#environment-secrets)
8. [Advanced Workflows](#advanced-workflows)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## CI/CD คืออะไร

### ขั้นตอนที่ 721: ทำความเข้าใจ CI/CD

```
CI = Continuous Integration
  → ทุกครั้งที่ push code จะ:
    1. Run tests อัตโนมัติ
    2. Build application
    3. Static code analysis
    4. Security scanning

CD = Continuous Delivery / Deployment
  → หลังจาก CI สำเร็จ:
    Delivery: deploy ไปยัง staging (manual approval ก่อน production)
    Deployment: deploy ไปยัง production อัตโนมัติ

Pipeline:
  Code Push → Build → Test → Security Scan → Deploy to Staging → Deploy to Production
```

---

## GitHub Actions พื้นฐาน

### ขั้นตอนที่ 722: โครงสร้าง Workflow

```yaml
# .github/workflows/basic.yml

# ชื่อ workflow
name: CI/CD Pipeline

# Events ที่ trigger workflow
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  # Manual trigger
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

# Jobs
jobs:
  # Job 1: Build and Test
  test:
    name: Build and Test
    runs-on: ubuntu-latest   # Runner type

    # Matrix strategy - test บน Node versions ต่างๆ
    strategy:
      matrix:
        node-version: [18.x, 20.x]

    steps:
      # Checkout code
      - name: Checkout code
        uses: actions/checkout@v4

      # Setup Node.js
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      # Install dependencies
      - name: Install dependencies
        run: npm ci

      # Run tests
      - name: Run tests
        run: npm test
        env:
          NODE_ENV: test
          MONGODB_URI: ${{ secrets.MONGODB_TEST_URI }}

      # Upload coverage
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        if: matrix.node-version == '18.x'
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
```

### ขั้นตอนที่ 723: Contexts และ Expressions

```yaml
# การใช้ contexts
steps:
  - name: Show context info
    run: |
      echo "Event: ${{ github.event_name }}"
      echo "Branch: ${{ github.ref_name }}"
      echo "SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      echo "Repository: ${{ github.repository }}"

  # เงื่อนไข
  - name: Deploy to production
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    run: echo "Deploying to production"

  - name: Deploy to staging
    if: github.ref == 'refs/heads/develop'
    run: echo "Deploying to staging"

  # ใช้ env context
  - name: Use environment
    env:
      MY_VAR: "hello"
    run: echo $MY_VAR

  # Expression functions
  - name: Check if needs deployment
    if: |
      contains(github.event.head_commit.message, '[deploy]') ||
      startsWith(github.ref, 'refs/tags/v')
    run: echo "Deployment triggered"
```

---

## Workflow สำหรับ Node.js

### ขั้นตอนที่ 724: Comprehensive Node.js Pipeline

```yaml
# .github/workflows/nodejs.yml
name: Node.js CI

on:
  push:
    branches: [main, develop, 'feature/**']
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '18.x'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ==========================================
  # Lint
  # ==========================================
  lint:
    name: Lint Code
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Run Prettier check
        run: npm run format:check

  # ==========================================
  # Security Audit
  # ==========================================
  security:
    name: Security Audit
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: npm audit
        run: npm audit --audit-level=high

      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'myapp'
          path: '.'
          format: 'HTML'

      - name: Snyk Security Scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  # ==========================================
  # Test
  # ==========================================
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    needs: [lint]

    services:
      # Start MongoDB สำหรับ integration tests
      mongodb:
        image: mongo:6.0
        ports:
          - 27017:27017
        options: >-
          --health-cmd "mongosh --eval 'db.adminCommand(\"ping\")'"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      # Start Redis
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm run test:unit
        env:
          NODE_ENV: test

      - name: Run integration tests
        run: npm run test:integration
        env:
          NODE_ENV: test
          MONGODB_URI: mongodb://localhost:27017/test
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test-secret

      - name: Generate coverage report
        run: npm run test:coverage

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          fail_ci_if_error: true

      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results
          path: |
            coverage/
            test-results.xml

  # ==========================================
  # Build Docker Image
  # ==========================================
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [test, security]
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix={{branch}}-

      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64

      - name: Sign Docker image
        if: github.event_name != 'pull_request'
        run: |
          cosign sign --yes \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}
        env:
          COSIGN_EXPERIMENTAL: 'true'
```

---

## Testing Pipeline

### ขั้นตอนที่ 725: End-to-End Testing

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # ทุกวันตี 2

jobs:
  e2e:
    name: Playwright E2E Tests
    runs-on: ubuntu-latest
    timeout-minutes: 60

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18.x'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps chromium firefox

      - name: Start application
        run: |
          docker-compose up -d
          # รอให้ app พร้อม
          npx wait-on http://localhost:3000/health --timeout 60000

      - name: Run Playwright tests
        run: npx playwright test
        env:
          BASE_URL: http://localhost:3000
          CI: true

      - name: Upload Playwright report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30

      - name: Stop application
        if: always()
        run: docker-compose down
```

### ขั้นตอนที่ 726: Performance Testing

```yaml
# Performance testing ด้วย k6
  performance:
    name: Performance Tests
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Start application
        run: docker-compose up -d
      
      - name: Wait for app
        run: npx wait-on http://localhost:3000/health

      - name: Run k6 tests
        uses: grafana/k6-action@v0.3.0
        with:
          filename: tests/performance/load-test.js
          flags: --out json=results.json
        env:
          K6_CLOUD_TOKEN: ${{ secrets.K6_CLOUD_TOKEN }}

      - name: Upload k6 results
        uses: actions/upload-artifact@v3
        with:
          name: k6-results
          path: results.json
```

```javascript
// tests/performance/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const responseTime = new Trend('response_time');

export const options = {
  stages: [
    { duration: '30s', target: 10 },   // Ramp up
    { duration: '1m', target: 50 },    // Stay at 50
    { duration: '30s', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% requests < 500ms
    errors: ['rate<0.01'],             // Error rate < 1%
  },
};

export default function() {
  const baseUrl = __ENV.BASE_URL || 'http://localhost:3000';

  // Test API endpoint
  const response = http.get(`${baseUrl}/api/posts`);

  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  errorRate.add(response.status !== 200);
  responseTime.add(response.timings.duration);

  sleep(1);
}
```

---

## Docker Build และ Push

### ขั้นตอนที่ 727: Multi-Platform Docker Build

```yaml
# .github/workflows/docker.yml
name: Build and Push Docker

on:
  release:
    types: [published]
  push:
    tags: ['v*.*.*']

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64,linux/arm/v7
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/myapp:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ github.ref_name }}
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.ref_name }}
          build-args: |
            VERSION=${{ github.ref_name }}
            BUILD_DATE=${{ fromJSON(steps.date.outputs.result) }}
            VCS_REF=${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Update Docker Hub Description
        uses: peter-evans/dockerhub-description@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_PASSWORD }}
          repository: ${{ secrets.DOCKERHUB_USERNAME }}/myapp
```

---

## Deployment Workflows

### ขั้นตอนที่ 728: Deploy to Staging

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy to Staging

on:
  push:
    branches: [develop]

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: myapp
          IMAGE_TAG: staging-${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster staging \
            --service myapp \
            --force-new-deployment
```

### ขั้นตอนที่ 729: Deploy to Production (Manual Approval)

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy (e.g. v1.2.3)'
        required: true
      confirm:
        description: 'Type "deploy" to confirm'
        required: true

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Validate confirmation
        if: github.event.inputs.confirm != 'deploy'
        run: |
          echo "❌ Confirmation not provided correctly"
          exit 1

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: validate
    environment:
      name: production
      url: https://example.com

    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.inputs.version }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.PROD_AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.PROD_AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1

      - name: Get image from ECR
        id: get-image
        env:
          ECR_REGISTRY: ${{ secrets.ECR_REGISTRY }}
          IMAGE_TAG: ${{ github.event.inputs.version }}
        run: |
          echo "image=$ECR_REGISTRY/myapp:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Create ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: myapp
          image: ${{ steps.get-image.outputs.image }}

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: myapp-production
          cluster: production
          wait-for-service-stability: true

      - name: Verify deployment
        run: |
          # ตรวจสอบว่า deployment สำเร็จ
          response=$(curl -s -o /dev/null -w "%{http_code}" https://example.com/health)
          if [ $response -ne 200 ]; then
            echo "❌ Health check failed"
            exit 1
          fi
          echo "✅ Deployment verified"

      - name: Notify Slack
        uses: rtCamp/action-slack-notify@v2
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
          SLACK_MESSAGE: "✅ Deployed ${{ github.event.inputs.version }} to production"
          SLACK_COLOR: "good"
```

### ขั้นตอนที่ 730: Deploy กับ SSH

```yaml
# Deploy ไปยัง VPS ด้วย SSH
  deploy-vps:
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /app/myapp

            # Pull latest image
            docker pull ${{ secrets.REGISTRY }}/myapp:latest

            # Update và restart
            docker-compose pull
            docker-compose up -d --no-deps app

            # Health check
            sleep 10
            curl -f http://localhost:3000/health || exit 1

            echo "✅ Deployment complete"
```

---

## Environment Secrets

### ขั้นตอนที่ 731: การจัดการ Secrets

```yaml
# ใช้ GitHub Secrets ใน workflows
jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      # อ่าน secrets จาก repository secrets
      - name: Use secrets
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          JWT_SECRET: ${{ secrets.JWT_SECRET }}
          REDIS_PASSWORD: ${{ secrets.REDIS_PASSWORD }}
        run: echo "Secrets loaded"

      # สร้าง .env file จาก secrets
      - name: Create .env file
        run: |
          cat << EOF > .env
          NODE_ENV=production
          DATABASE_URL=${{ secrets.DATABASE_URL }}
          JWT_SECRET=${{ secrets.JWT_SECRET }}
          REDIS_URL=${{ secrets.REDIS_URL }}
          EOF

      # ใช้ environment-specific secrets
      - name: Deploy to environment
        environment: ${{ github.ref_name == 'main' && 'production' || 'staging' }}
        run: echo "Deployed to ${{ github.ref_name }}"
```

---

## Advanced Workflows

### ขั้นตอนที่ 732: Reusable Workflows

```yaml
# .github/workflows/reusable-test.yml
name: Reusable Test Workflow

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
      run-e2e:
        required: false
        type: boolean
        default: false
    secrets:
      mongodb-uri:
        required: true
    outputs:
      coverage:
        description: "Coverage percentage"
        value: ${{ jobs.test.outputs.coverage }}

jobs:
  test:
    runs-on: ubuntu-latest
    outputs:
      coverage: ${{ steps.coverage.outputs.percentage }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}

      - run: npm ci

      - name: Run tests
        run: npm test
        env:
          MONGODB_URI: ${{ secrets.mongodb-uri }}

      - id: coverage
        run: echo "percentage=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')" >> $GITHUB_OUTPUT
```

```yaml
# ใช้ reusable workflow
# .github/workflows/main.yml
jobs:
  test:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '18.x'
      run-e2e: true
    secrets:
      mongodb-uri: ${{ secrets.MONGODB_URI }}
```

### ขั้นตอนที่ 733: Workflow Concurrency

```yaml
# ป้องกัน concurrent deployments
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# หรือสำหรับ deployments ที่ต้องรันให้เสร็จ
concurrency:
  group: deploy-${{ github.ref_name }}
  cancel-in-progress: false
```

### ขั้นตอนที่ 734: Release Automation

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags: ['v*.*.*']

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Fetch all history สำหรับ changelog

      - name: Generate Changelog
        id: changelog
        uses: mikepenz/release-changelog-builder-action@v4
        with:
          configuration: ".github/changelog-config.json"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          body: ${{ steps.changelog.outputs.changelog }}
          files: |
            dist/**
            *.tar.gz
          draft: false
          prerelease: ${{ contains(github.ref_name, '-') }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Update CHANGELOG.md
        run: |
          echo "## ${{ github.ref_name }}" | cat - CHANGELOG.md > temp && mv temp CHANGELOG.md
          echo "${{ steps.changelog.outputs.changelog }}" >> CHANGELOG.md
          git config user.email "actions@github.com"
          git config user.name "GitHub Actions"
          git add CHANGELOG.md
          git commit -m "Update CHANGELOG for ${{ github.ref_name }}"
          git push origin HEAD:main
```

---

## Code Quality Gates

### ขั้นตอนที่ 735: SonarCloud Analysis

```yaml
# .github/workflows/sonar.yml
name: SonarCloud Analysis

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  sonarcloud:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18.x'

      - name: Install and test
        run: |
          npm ci
          npm run test:coverage

      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

```properties
# sonar-project.properties
sonar.projectKey=myapp
sonar.organization=myorg
sonar.sources=src
sonar.tests=__tests__
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.exclusions=**/*.test.js,**/node_modules/**
sonar.coverage.exclusions=**/*.test.js
```

---

## Notification Systems

### ขั้นตอนที่ 736: Slack Notifications

```yaml
# Notify Slack
  notify:
    runs-on: ubuntu-latest
    needs: [deploy]
    if: always()

    steps:
      - name: Notify success
        if: needs.deploy.result == 'success'
        uses: rtCamp/action-slack-notify@v2
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
          SLACK_COLOR: '#36a64f'
          SLACK_ICON: https://github.com/rtCamp.png?size=48
          SLACK_MESSAGE: |
            ✅ *Deployment Successful*
            *Environment:* Production
            *Version:* ${{ github.ref_name }}
            *Deployed by:* ${{ github.actor }}
            *Commit:* ${{ github.sha }}
          SLACK_TITLE: 'Deployment Success'
          SLACK_USERNAME: GitHub Actions

      - name: Notify failure
        if: needs.deploy.result == 'failure'
        uses: rtCamp/action-slack-notify@v2
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
          SLACK_COLOR: '#ff0000'
          SLACK_MESSAGE: |
            ❌ *Deployment Failed*
            *Environment:* Production
            *Version:* ${{ github.ref_name }}
            *Actor:* ${{ github.actor }}
            *View logs:* ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
          SLACK_TITLE: 'Deployment Failed'
```

---

## Dependency Updates

### ขั้นตอนที่ 737: Dependabot Configuration

```yaml
# .github/dependabot.yml
version: 2

updates:
  # npm dependencies
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Bangkok"
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
      - "automated"
    reviewers:
      - "team-lead"
    groups:
      # รวม minor/patch updates ของ dependencies เข้าด้วยกัน
      dev-dependencies:
        dependency-type: "development"
        update-types: ["minor", "patch"]
      production-minor-patch:
        dependency-type: "production"
        update-types: ["minor", "patch"]

  # Docker
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

---

## Branch Protection Rules

### ขั้นตอนที่ 738: Branch Protection Setup

```yaml
# .github/workflows/branch-protection.yml
# ตั้งค่า branch protection ผ่าน GitHub API

name: Setup Branch Protection

on:
  workflow_dispatch:

jobs:
  protect:
    runs-on: ubuntu-latest
    steps:
      - name: Protect main branch
        uses: octokit/request-action@v2.x
        with:
          route: PUT /repos/{owner}/{repo}/branches/main/protection
          owner: ${{ github.repository_owner }}
          repo: ${{ github.event.repository.name }}
          required_status_checks: |
            strict: true
            contexts:
              - "lint"
              - "test"
              - "security"
              - "build"
          enforce_admins: false
          required_pull_request_reviews: |
            dismiss_stale_reviews: true
            require_code_owner_reviews: true
            required_approving_review_count: 2
          restrictions: null
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Caching Strategies

### ขั้นตอนที่ 739: Cache Dependencies

```yaml
# Comprehensive caching
steps:
  - uses: actions/checkout@v4

  # Cache node_modules
  - name: Cache dependencies
    uses: actions/cache@v3
    with:
      path: ~/.npm
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node-

  # Cache build output
  - name: Cache build
    uses: actions/cache@v3
    with:
      path: dist/
      key: build-${{ github.sha }}
      restore-keys: |
        build-${{ github.ref_name }}-

  # Cache Docker layers
  - name: Cache Docker layers
    uses: actions/cache@v3
    with:
      path: /tmp/.buildx-cache
      key: ${{ runner.os }}-buildx-${{ github.sha }}
      restore-keys: |
        ${{ runner.os }}-buildx-

  - name: Build with cache
    uses: docker/build-push-action@v5
    with:
      cache-from: type=local,src=/tmp/.buildx-cache
      cache-to: type=local,dest=/tmp/.buildx-cache-new
```

---

## Complete Pipeline Example

### ขั้นตอนที่ 740: Full CI/CD Pipeline

```yaml
# .github/workflows/complete-pipeline.yml
name: Complete CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  release:
    types: [published]

jobs:
  # ====== Checks ======
  lint:
    uses: ./.github/workflows/lint.yml

  security:
    uses: ./.github/workflows/security.yml
    secrets: inherit

  # ====== Test ======
  test:
    needs: lint
    uses: ./.github/workflows/test.yml
    secrets: inherit

  # ====== Build ======
  build:
    needs: [test, security]
    uses: ./.github/workflows/build.yml
    secrets: inherit

  # ====== Deploy Staging ======
  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
        run: echo "Deploying to staging"

  # ====== Deploy Production ======
  deploy-production:
    needs: build
    if: github.event_name == 'release'
    environment: production
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: echo "Deploying to production"

  # ====== Notify ======
  notify:
    needs: [deploy-staging, deploy-production]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Send notification
        run: echo "Sending notification"
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Complete Pipeline

สร้าง GitHub Actions workflow ที่:
- Lint code
- Run unit + integration tests
- Build Docker image
- Push ไปยัง registry
- Deploy ไปยัง staging เมื่อ push develop
- Deploy ไปยัง production เมื่อ create release

### แบบฝึกหัดที่ 2: สร้าง Rollback Workflow

```yaml
# TODO: สร้าง workflow สำหรับ rollback
# เมื่อ deployment ล้มเหลว ให้ rollback ไปยัง version ก่อนหน้า
name: Rollback
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to rollback to'
        required: true
```

### แบบฝึกหัดที่ 3: Database Migration

```yaml
# TODO: สร้าง workflow สำหรับ run database migrations
# ก่อน deployment และ rollback ถ้า migration ล้มเหลว
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **CI/CD Concepts** - Continuous Integration และ Deployment
2. **GitHub Actions** - workflows, jobs, steps, actions
3. **Testing Pipeline** - unit, integration, E2E, performance
4. **Docker Integration** - build, push, multi-platform
5. **Deployment Strategies** - staging, production, manual approval
6. **Security** - secrets management, vulnerability scanning
7. **Advanced Features** - reusable workflows, concurrency, caching
8. **Notifications** - Slack, release management

ในบทถัดไปเราจะเรียนรู้ **AWS Deployment** - EC2, RDS, S3, Lambda และการ deploy Node.js บน AWS
