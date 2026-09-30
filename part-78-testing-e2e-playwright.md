# Part 78: E2E Testing กับ Playwright
## ขั้นตอนที่ 771-780 จาก 1000

---

## Playwright คืออะไร?

Playwright เป็น test automation library จาก Microsoft รองรับ Chromium, Firefox, และ WebKit ใช้สำหรับ E2E testing ทั้ง web UI และ API

---

## 1. Setup

```bash
npm install -D @playwright/test
npx playwright install

# สำหรับ API testing เท่านั้น (ไม่ต้องติดตั้ง browsers)
npm install -D @playwright/test
```

### playwright.config.ts

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  timeout: 30000,
  expect: { timeout: 5000 },
  
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  
  reporter: [
    ['html', { outputFolder: 'playwright-report' }],
    ['junit', { outputFile: 'test-results/results.xml' }]
  ],
  
  use: {
    baseURL: process.env.API_URL || 'http://localhost:3000',
    extraHTTPHeaders: {
      'Content-Type': 'application/json'
    },
    trace: 'on-first-retry',
    screenshot: 'only-on-failure'
  },

  projects: [
    { name: 'api', use: {} },
    { 
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'mobile',
      use: { ...devices['iPhone 12'] }
    }
  ],

  webServer: {
    command: 'npm run start:test',
    url: 'http://localhost:3000/health',
    reuseExistingServer: !process.env.CI,
    timeout: 120 * 1000
  }
});
```

---

## 2. Page Object Model

### Base Page

```typescript
// tests/e2e/pages/base.page.ts
import { Page, Locator } from '@playwright/test';

export abstract class BasePage {
  constructor(protected page: Page) {}

  async navigate(path: string): Promise<void> {
    await this.page.goto(path);
    await this.page.waitForLoadState('networkidle');
  }

  async getTitle(): Promise<string> {
    return this.page.title();
  }

  protected async fillInput(selector: string, value: string): Promise<void> {
    const input = this.page.locator(selector);
    await input.clear();
    await input.fill(value);
  }

  protected async clickButton(text: string): Promise<void> {
    await this.page.getByRole('button', { name: text }).click();
  }

  protected async waitForToast(message: string): Promise<void> {
    await this.page.getByText(message).waitFor({ state: 'visible' });
  }

  protected async getErrorMessage(): Promise<string | null> {
    const error = this.page.locator('[data-testid="error-message"]');
    if (await error.isVisible()) {
      return error.textContent();
    }
    return null;
  }
}
```

### Login Page

```typescript
// tests/e2e/pages/login.page.ts
import { Page, expect } from '@playwright/test';
import { BasePage } from './base.page';

export class LoginPage extends BasePage {
  private readonly emailInput: Locator;
  private readonly passwordInput: Locator;
  private readonly submitButton: Locator;
  private readonly errorMessage: Locator;

  constructor(page: Page) {
    super(page);
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Login' });
    this.errorMessage = page.getByRole('alert');
  }

  async goto(): Promise<void> {
    await this.navigate('/login');
  }

  async login(email: string, password: string): Promise<void> {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async expectSuccessfulLogin(): Promise<void> {
    await expect(this.page).toHaveURL('/dashboard');
  }

  async expectError(message: string): Promise<void> {
    await expect(this.errorMessage).toBeVisible();
    await expect(this.errorMessage).toContainText(message);
  }
}
```

### Dashboard Page

```typescript
// tests/e2e/pages/dashboard.page.ts
import { Page, Locator, expect } from '@playwright/test';
import { BasePage } from './base.page';

export class DashboardPage extends BasePage {
  constructor(page: Page) {
    super(page);
  }

  async goto(): Promise<void> {
    await this.navigate('/dashboard');
  }

  async getWelcomeMessage(): Promise<string | null> {
    return this.page.getByTestId('welcome-message').textContent();
  }

  async navigateToUsers(): Promise<void> {
    await this.page.getByRole('link', { name: 'Users' }).click();
    await this.page.waitForURL('/users');
  }

  async logout(): Promise<void> {
    await this.page.getByTestId('user-menu').click();
    await this.page.getByRole('menuitem', { name: 'Logout' }).click();
    await this.page.waitForURL('/login');
  }
}
```

---

## 3. API Testing

```typescript
// tests/e2e/api/auth.api.test.ts
import { test, expect, APIRequestContext, request } from '@playwright/test';

let apiContext: APIRequestContext;
let authToken: string;

test.beforeAll(async () => {
  apiContext = await request.newContext({
    baseURL: process.env.API_URL || 'http://localhost:3000',
    extraHTTPHeaders: { 'Content-Type': 'application/json' }
  });
});

test.afterAll(async () => {
  await apiContext.dispose();
});

test.describe('Auth API', () => {
  test('POST /api/auth/register - should register new user', async () => {
    const response = await apiContext.post('/api/auth/register', {
      data: {
        name: 'Test User',
        email: `test-${Date.now()}@example.com`,
        password: 'SecurePass123!'
      }
    });

    expect(response.status()).toBe(201);
    
    const body = await response.json();
    expect(body).toMatchObject({
      success: true,
      data: expect.objectContaining({
        id: expect.any(String),
        email: expect.any(String),
        name: 'Test User'
      }),
      token: expect.any(String)
    });
    
    expect(body.data.password).toBeUndefined();
  });

  test('POST /api/auth/login - should login with valid credentials', async () => {
    const email = `test-${Date.now()}@example.com`;
    
    // Register first
    await apiContext.post('/api/auth/register', {
      data: { name: 'Test', email, password: 'SecurePass123!' }
    });

    // Login
    const response = await apiContext.post('/api/auth/login', {
      data: { email, password: 'SecurePass123!' }
    });

    expect(response.status()).toBe(200);
    const body = await response.json();
    
    expect(body.token).toBeTruthy();
    authToken = body.token;
  });

  test('POST /api/auth/login - should reject invalid password', async () => {
    const response = await apiContext.post('/api/auth/login', {
      data: { email: 'test@example.com', password: 'WrongPassword' }
    });

    expect(response.status()).toBe(401);
    const body = await response.json();
    expect(body.error).toBeTruthy();
  });

  test('GET /api/auth/me - should return user with valid token', async () => {
    const response = await apiContext.get('/api/auth/me', {
      headers: { Authorization: `Bearer ${authToken}` }
    });

    expect(response.status()).toBe(200);
    const body = await response.json();
    expect(body.data).toHaveProperty('email');
  });

  test('GET /api/auth/me - should reject without token', async () => {
    const response = await apiContext.get('/api/auth/me');
    expect(response.status()).toBe(401);
  });
});
```

```typescript
// tests/e2e/api/users.api.test.ts
import { test, expect, APIRequestContext, request } from '@playwright/test';

test.describe('Users CRUD API', () => {
  let apiContext: APIRequestContext;
  let adminToken: string;
  let createdUserId: string;

  test.beforeAll(async () => {
    apiContext = await request.newContext({
      baseURL: process.env.API_URL || 'http://localhost:3000'
    });

    // Login as admin
    const loginRes = await apiContext.post('/api/auth/login', {
      data: {
        email: process.env.ADMIN_EMAIL || 'admin@example.com',
        password: process.env.ADMIN_PASSWORD || 'AdminPass123!'
      }
    });
    
    const loginBody = await loginRes.json();
    adminToken = loginBody.token;
  });

  test.afterAll(async () => {
    await apiContext.dispose();
  });

  test('GET /api/users - should return paginated users', async () => {
    const response = await apiContext.get('/api/users?page=1&limit=10', {
      headers: { Authorization: `Bearer ${adminToken}` }
    });

    expect(response.status()).toBe(200);
    const body = await response.json();
    
    expect(body).toMatchObject({
      success: true,
      data: expect.any(Array),
      pagination: expect.objectContaining({
        page: 1,
        limit: 10,
        total: expect.any(Number)
      })
    });
  });

  test('POST /api/users - should create user', async () => {
    const userData = {
      name: 'New User',
      email: `new-user-${Date.now()}@example.com`,
      password: 'TestPass123!'
    };

    const response = await apiContext.post('/api/users', {
      headers: { Authorization: `Bearer ${adminToken}` },
      data: userData
    });

    expect(response.status()).toBe(201);
    const body = await response.json();
    
    createdUserId = body.data.id;
    expect(body.data.name).toBe(userData.name);
    expect(body.data.email).toBe(userData.email.toLowerCase());
  });

  test('GET /api/users/:id - should return user', async () => {
    const response = await apiContext.get(`/api/users/${createdUserId}`, {
      headers: { Authorization: `Bearer ${adminToken}` }
    });

    expect(response.status()).toBe(200);
    const body = await response.json();
    expect(body.data.id).toBe(createdUserId);
  });

  test('PUT /api/users/:id - should update user', async () => {
    const response = await apiContext.put(`/api/users/${createdUserId}`, {
      headers: { Authorization: `Bearer ${adminToken}` },
      data: { name: 'Updated Name' }
    });

    expect(response.status()).toBe(200);
    const body = await response.json();
    expect(body.data.name).toBe('Updated Name');
  });

  test('DELETE /api/users/:id - should delete user', async () => {
    const response = await apiContext.delete(`/api/users/${createdUserId}`, {
      headers: { Authorization: `Bearer ${adminToken}` }
    });

    expect(response.status()).toBe(200);
    
    // Verify deleted
    const getResponse = await apiContext.get(`/api/users/${createdUserId}`, {
      headers: { Authorization: `Bearer ${adminToken}` }
    });
    expect(getResponse.status()).toBe(404);
  });
});
```

---

## 4. UI Testing

```typescript
// tests/e2e/ui/login.test.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/login.page';
import { DashboardPage } from '../pages/dashboard.page';

test.describe('Login Flow', () => {
  test('should login successfully', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    
    await loginPage.login('admin@example.com', 'AdminPass123!');
    await loginPage.expectSuccessfulLogin();
    
    const dashboardPage = new DashboardPage(page);
    const welcomeMessage = await dashboardPage.getWelcomeMessage();
    expect(welcomeMessage).toContain('Welcome');
  });

  test('should show error on invalid credentials', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    
    await loginPage.login('wrong@example.com', 'wrongpassword');
    await loginPage.expectError('Invalid credentials');
  });

  test('should logout successfully', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('admin@example.com', 'AdminPass123!');
    
    const dashboardPage = new DashboardPage(page);
    await dashboardPage.logout();
    
    await expect(page).toHaveURL('/login');
  });
});
```

---

## 5. CI Integration

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on: [push, pull_request]

jobs:
  e2e:
    runs-on: ubuntu-latest
    
    services:
      mongodb:
        image: mongo:6
        ports: ['27017:27017']
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: npm
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium
      
      - name: Start server
        run: |
          npm run seed:test
          npm run start:test &
          sleep 5
        env:
          DATABASE_URL: mongodb://localhost:27017/test
          NODE_ENV: test
      
      - name: Run E2E tests
        run: npx playwright test
        env:
          API_URL: http://localhost:3000
          ADMIN_EMAIL: admin@example.com
          ADMIN_PASSWORD: AdminPass123!
      
      - name: Upload test report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
- ตั้งค่า Playwright
- เขียน API tests สำหรับ auth endpoints
- Run tests และดู report

### ระดับ 2: กลาง
- Page Object Model
- UI tests สำหรับ login flow
- Test data management

### ระดับ 3: ขั้นสูง
- CI/CD integration
- Visual regression testing
- Performance assertions

---

## สรุป

Playwright เป็น tool ที่ทรงพลังสำหรับ E2E testing รองรับทั้ง API และ UI testing ใน framework เดียว Page Object Model ช่วย organize tests ให้ maintainable การรวม E2E tests เข้ากับ CI/CD pipeline ช่วยให้มั่นใจว่า regression ไม่เกิดขึ้นก่อน deploy

> ขั้นตอนต่อไป: Part 79 - Apache Kafka
