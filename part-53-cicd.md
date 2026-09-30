# Part 53: CI/CD (Continuous Integration / Continuous Deployment)
## ขั้นตอนที่ 53-53 จาก 1000

---

## บทนำ

CI/CD คือแนวปฏิบัติที่ช่วยให้ทีมพัฒนาสามารถส่ง code ไปยัง production ได้อย่างรวดเร็วและมั่นใจ

---

## 53.1 CI/CD Concepts

### Continuous Integration (CI)

```
Developer pushes code
    ↓
CI System picks up change
    ↓
Run automated tests
    ↓
Code quality checks
    ↓
Build application
    ↓
Report results
```

### Continuous Deployment (CD)

```
CI passes
    ↓
Deploy to Staging
    ↓
Integration tests
    ↓
Deploy to Production
    ↓
Monitor
```

---

## 53.2 GitHub Actions

### โครงสร้าง GitHub Actions

```
.github/
└── workflows/
    ├── ci.yml          # Continuous Integration
    ├── cd.yml          # Continuous Deployment
    ├── security.yml    # Security scanning
    └── release.yml     # Release management
```

### CI Workflow พื้นฐาน

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '20'

jobs:
  # Job 1: Lint และ Type Check
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run ESLint
        run: npm run lint
      
      - name: Run TypeScript check
        run: npm run type-check

  # Job 2: Unit Tests
  test:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: lint
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    env:
      NODE_ENV: test
      DB_HOST: localhost
      DB_PORT: 5432
      DB_NAME: testdb
      DB_USER: testuser
      DB_PASSWORD: testpass
      REDIS_URL: redis://localhost:6379
      JWT_SECRET: test-secret-key
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run database migrations
        run: npm run db:migrate
      
      - name: Run unit tests
        run: npm run test:unit -- --coverage
      
      - name: Upload coverage report
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          file: ./coverage/lcov.info

  # Job 3: Integration Tests
  integration-test:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Start services
        run: docker-compose -f docker-compose.test.yml up -d
      
      - name: Wait for services
        run: |
          sleep 10
          docker-compose -f docker-compose.test.yml ps
      
      - name: Run integration tests
        run: npm run test:integration
      
      - name: Stop services
        if: always()
        run: docker-compose -f docker-compose.test.yml down

  # Job 4: Build Docker Image
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [test, integration-test]
    if: github.event_name == 'push'
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ secrets.DOCKER_USERNAME }}/myapp
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            type=semver,pattern={{version}}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 53.3 Testing in CI

### Jest Configuration

```javascript
// jest.config.js
module.exports = {
  testEnvironment: 'node',
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  coverageReporters: ['text', 'lcov', 'html'],
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/*.test.js',
    '!src/config/**',
    '!src/database/migrations/**'
  ],
  testMatch: [
    '**/__tests__/**/*.js',
    '**/*.test.js'
  ],
  globalSetup: './tests/setup.js',
  globalTeardown: './tests/teardown.js'
};
```

### Test Setup

```javascript
// tests/setup.js
const { sequelize } = require('../src/config/database');

module.exports = async () => {
  // รัน migrations
  await sequelize.sync({ force: true });
  
  // Seed test data
  await seedTestData();
  
  console.log('✅ Test database ready');
};

// tests/teardown.js
module.exports = async () => {
  const { sequelize } = require('../src/config/database');
  await sequelize.close();
  console.log('✅ Test database closed');
};
```

### Unit Test Example

```javascript
// src/__tests__/userService.test.js
const { createUser, getUserById } = require('../services/userService');
const User = require('../models/User');

describe('UserService', () => {
  beforeEach(async () => {
    await User.destroy({ where: {}, truncate: true });
  });
  
  describe('createUser', () => {
    it('should create a new user', async () => {
      const userData = {
        name: 'สมชาย ใจดี',
        email: 'somchai@test.com',
        password: 'Password123!'
      };
      
      const user = await createUser(userData);
      
      expect(user).toBeDefined();
      expect(user.id).toBeDefined();
      expect(user.name).toBe(userData.name);
      expect(user.email).toBe(userData.email);
      expect(user.password).not.toBe(userData.password);  // ต้อง hash แล้ว
    });
    
    it('should throw error for duplicate email', async () => {
      const userData = {
        name: 'Test User',
        email: 'duplicate@test.com',
        password: 'Password123!'
      };
      
      await createUser(userData);
      
      await expect(createUser(userData)).rejects.toThrow(
        'Email already exists'
      );
    });
  });
  
  describe('getUserById', () => {
    it('should return user by id', async () => {
      const created = await createUser({
        name: 'Test',
        email: 'test@test.com',
        password: 'Password123!'
      });
      
      const user = await getUserById(created.id);
      
      expect(user.id).toBe(created.id);
      expect(user.email).toBe('test@test.com');
    });
    
    it('should return null for non-existent user', async () => {
      const user = await getUserById(99999);
      expect(user).toBeNull();
    });
  });
});
```

---

## 53.4 Deployment Pipelines

### CD Workflow

```yaml
# .github/workflows/cd.yml
name: CD

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /app
            docker-compose pull
            docker-compose up -d --no-deps app
            docker-compose exec -T app npm run db:migrate
  
  smoke-test:
    name: Smoke Tests
    needs: deploy-staging
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run smoke tests
        run: |
          npm ci
          STAGING_URL=${{ secrets.STAGING_URL }} npm run test:smoke
  
  deploy-production:
    name: Deploy to Production
    needs: smoke-test
    runs-on: ubuntu-latest
    environment: production  # ต้องการ approval
    
    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd /app
            
            # Backup database
            docker-compose exec -T postgres pg_dump -U $DB_USER $DB_NAME > backup_$(date +%Y%m%d_%H%M%S).sql
            
            # Rolling update
            docker-compose pull app
            docker-compose up -d --no-deps --scale app=2 app
            sleep 30
            
            # Health check
            curl -f http://localhost/health || exit 1
            
            echo "✅ Deployment successful"
      
      - name: Notify Slack
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: "Production deployment ${{ job.status }}"
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 53.5 Security Scanning

```yaml
# .github/workflows/security.yml
name: Security

on:
  schedule:
    - cron: '0 2 * * 1'  # ทุกวันจันทร์ 02:00
  push:
    branches: [main]

jobs:
  npm-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm audit --audit-level=high
  
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Semgrep
        uses: semgrep/semgrep-action@v1
        with:
          config: p/nodejs
  
  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t scan-target .
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'scan-target'
          severity: 'CRITICAL,HIGH'
```

---

## แบบฝึกหัดที่ 53

### แบบฝึกหัดพื้นฐาน

**1. สร้าง CI Pipeline**

สร้าง GitHub Actions workflow ที่:
- Lint code
- Run unit tests
- Check coverage > 80%
- Build Docker image

**2. สร้าง CD Pipeline**

สร้าง deployment pipeline ที่:
- Deploy to staging automatically
- Wait for approval ก่อน production
- Rollback ถ้า health check ล้มเหลว

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **CI/CD Concepts** - integration, deployment pipeline
2. **GitHub Actions** - workflows, jobs, steps
3. **Testing in CI** - unit, integration tests
4. **Deployment** - staging, production, rollback
5. **Security** - npm audit, container scanning

**ถัดไป:** Part 54 - AWS Deployment
