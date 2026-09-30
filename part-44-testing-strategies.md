# Part 44 | ขั้นตอนที่ 781-800 จาก 1000

## Testing Strategies - กลยุทธ์การทดสอบ

---

## สารบัญ

1. [Testing Pyramid](#testing-pyramid)
2. [Unit Testing กับ Jest](#unit-testing-กับ-jest)
3. [Mocking และ Stubbing](#mocking-และ-stubbing)
4. [Integration Testing](#integration-testing)
5. [API Testing](#api-testing)
6. [E2E Testing กับ Playwright](#e2e-testing-กับ-playwright)
7. [TDD - Test Driven Development](#tdd---test-driven-development)
8. [BDD - Behavior Driven Development](#bdd---behavior-driven-development)
9. [Test Coverage](#test-coverage)
10. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Testing Pyramid

### ขั้นตอนที่ 781: ทำความเข้าใจ Testing Pyramid

```
Testing Pyramid:

         /\
        /  \
       / E2E \     ← น้อยที่สุด, ช้า, แพง
      /--------\
     / Integration\ ← ปานกลาง
    /-------------\
   /   Unit Tests   \ ← มากที่สุด, เร็ว, ถูก
  /-------------------\

Rule of Thumb:
  Unit Tests: 70%
  Integration: 20%
  E2E: 10%
```

### ขั้นตอนที่ 782: โครงสร้าง Test Project

```
src/
├── __tests__/
│   ├── unit/
│   │   ├── services/
│   │   │   ├── userService.test.js
│   │   │   └── postService.test.js
│   │   ├── utils/
│   │   │   └── validators.test.js
│   │   └── middleware/
│   │       └── auth.test.js
│   ├── integration/
│   │   ├── routes/
│   │   │   ├── auth.test.js
│   │   │   └── posts.test.js
│   │   └── database/
│   │       └── models.test.js
│   └── e2e/
│       ├── auth.spec.ts
│       └── posts.spec.ts
├── jest.config.js
└── jest.setup.js
```

---

## Unit Testing กับ Jest

### ขั้นตอนที่ 783: ติดตั้งและตั้งค่า Jest

```bash
npm install --save-dev jest @types/jest babel-jest @babel/core @babel/preset-env
npm install --save-dev jest-environment-node
```

```javascript
// jest.config.js
module.exports = {
  // ใช้ environment ตาม type ของ test
  testEnvironment: 'node',

  // รูปแบบ test files
  testMatch: [
    '**/__tests__/**/*.test.js',
    '**/?(*.)+(spec|test).js',
  ],

  // Setup files
  setupFilesAfterFramework: ['./jest.setup.js'],

  // Coverage
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/*.test.js',
    '!src/index.js',
  ],

  coverageThresholds: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },

  coverageReporters: ['text', 'lcov', 'html'],

  // Transform
  transform: {
    '^.+\\.js$': 'babel-jest',
  },

  // Module aliases
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },

  // Timeout
  testTimeout: 10000,

  // Verbose
  verbose: true,

  // Projects สำหรับ run unit/integration แยกกัน
  projects: [
    {
      displayName: 'unit',
      testMatch: ['<rootDir>/src/__tests__/unit/**/*.test.js'],
    },
    {
      displayName: 'integration',
      testMatch: ['<rootDir>/src/__tests__/integration/**/*.test.js'],
      globalSetup: '<rootDir>/jest.globalSetup.js',
      globalTeardown: '<rootDir>/jest.globalTeardown.js',
    },
  ],
};
```

```javascript
// jest.setup.js
// ตั้งค่า global ที่ใช้ทุก tests

// Timeout
jest.setTimeout(10000);

// Global mocks
jest.mock('../src/config/logger', () => ({
  info: jest.fn(),
  warn: jest.fn(),
  error: jest.fn(),
  debug: jest.fn(),
}));

// Custom matchers
expect.extend({
  toBeWithinRange(received, floor, ceiling) {
    const pass = received >= floor && received <= ceiling;
    if (pass) {
      return {
        message: () => `expected ${received} not to be within range ${floor} - ${ceiling}`,
        pass: true,
      };
    }
    return {
      message: () => `expected ${received} to be within range ${floor} - ${ceiling}`,
      pass: false,
    };
  },
});
```

### ขั้นตอนที่ 784: Unit Tests พื้นฐาน

```javascript
// src/__tests__/unit/services/userService.test.js
const userService = require('../../../services/userService');
const User = require('../../../models/User');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');

// Mock dependencies
jest.mock('../../../models/User');
jest.mock('bcrypt');
jest.mock('jsonwebtoken');

describe('UserService', () => {
  // Setup ก่อนแต่ละ test
  beforeEach(() => {
    jest.clearAllMocks();
    process.env.JWT_SECRET = 'test-secret';
  });

  describe('register', () => {
    const validUserData = {
      username: 'testuser',
      email: 'test@example.com',
      password: 'password123',
    };

    it('should create a new user successfully', async () => {
      // Arrange
      const mockUser = {
        _id: 'user123',
        ...validUserData,
        password: 'hashedpassword',
      };

      User.findOne.mockResolvedValue(null);   // ไม่มี user อยู่แล้ว
      User.prototype.save = jest.fn().mockResolvedValue(mockUser);

      // Act
      const result = await userService.register(validUserData);

      // Assert
      expect(User.findOne).toHaveBeenCalledWith({
        $or: [{ email: validUserData.email }, { username: validUserData.username }],
      });
      expect(result).toHaveProperty('token');
      expect(result.user.email).toBe(validUserData.email);
    });

    it('should throw error if user already exists', async () => {
      // Arrange
      User.findOne.mockResolvedValue({ _id: 'existing', email: validUserData.email });

      // Act & Assert
      await expect(userService.register(validUserData))
        .rejects
        .toThrow('User already exists');
    });

    it('should hash password before saving', async () => {
      User.findOne.mockResolvedValue(null);
      bcrypt.hash.mockResolvedValue('hashed123');
      User.prototype.save = jest.fn().mockResolvedValue({});
      jwt.sign.mockReturnValue('token123');

      await userService.register(validUserData);

      expect(bcrypt.hash).toHaveBeenCalledWith(validUserData.password, 12);
    });
  });

  describe('login', () => {
    it('should return token for valid credentials', async () => {
      // Arrange
      const mockUser = {
        _id: 'user123',
        email: 'test@example.com',
        password: 'hashedpassword',
        role: 'USER',
      };

      User.findOne.mockResolvedValue(mockUser);
      bcrypt.compare.mockResolvedValue(true);
      jwt.sign.mockReturnValue('jwt-token');

      // Act
      const result = await userService.login('test@example.com', 'password123');

      // Assert
      expect(result).toHaveProperty('token', 'jwt-token');
      expect(bcrypt.compare).toHaveBeenCalledWith('password123', 'hashedpassword');
    });

    it('should throw AuthenticationError for wrong password', async () => {
      User.findOne.mockResolvedValue({ password: 'hashedpassword' });
      bcrypt.compare.mockResolvedValue(false);

      await expect(userService.login('test@example.com', 'wrongpassword'))
        .rejects
        .toThrow('Invalid credentials');
    });

    it('should throw AuthenticationError for non-existent user', async () => {
      User.findOne.mockResolvedValue(null);

      await expect(userService.login('notexist@example.com', 'password'))
        .rejects
        .toThrow('Invalid credentials');
    });
  });

  describe('getUser', () => {
    it('should return user by ID', async () => {
      const mockUser = { _id: 'user123', username: 'testuser' };
      User.findById.mockReturnValue({
        select: jest.fn().mockResolvedValue(mockUser),
      });

      const result = await userService.getUser('user123');
      expect(result).toEqual(mockUser);
    });

    it('should throw NotFoundError for non-existent user', async () => {
      User.findById.mockReturnValue({
        select: jest.fn().mockResolvedValue(null),
      });

      await expect(userService.getUser('nonexistent'))
        .rejects
        .toThrow('User not found');
    });
  });
});
```

### ขั้นตอนที่ 785: Testing Utility Functions

```javascript
// src/__tests__/unit/utils/validators.test.js
const {
  validateEmail,
  validatePassword,
  sanitizeInput,
  validatePagination,
} = require('../../../utils/validators');

describe('Validators', () => {
  describe('validateEmail', () => {
    // Valid emails
    it.each([
      'user@example.com',
      'user.name@example.co.th',
      'user+tag@gmail.com',
    ])('should accept valid email: %s', (email) => {
      expect(validateEmail(email)).toBe(true);
    });

    // Invalid emails
    it.each([
      'not-an-email',
      '@example.com',
      'user@',
      '',
      null,
      undefined,
    ])('should reject invalid email: %s', (email) => {
      expect(validateEmail(email)).toBe(false);
    });
  });

  describe('validatePassword', () => {
    it('should accept strong passwords', () => {
      expect(validatePassword('StrongP@ss123')).toEqual({ valid: true });
    });

    it('should reject short passwords', () => {
      const result = validatePassword('Sh0rt!');
      expect(result.valid).toBe(false);
      expect(result.errors).toContain('ต้องมีความยาวอย่างน้อย 8 ตัวอักษร');
    });

    it('should require uppercase letter', () => {
      const result = validatePassword('lowercase123!');
      expect(result.valid).toBe(false);
      expect(result.errors).toContain('ต้องมีตัวอักษรพิมพ์ใหญ่อย่างน้อย 1 ตัว');
    });

    it('should require number', () => {
      const result = validatePassword('NoNumbers!');
      expect(result.valid).toBe(false);
      expect(result.errors).toContain('ต้องมีตัวเลขอย่างน้อย 1 ตัว');
    });
  });

  describe('sanitizeInput', () => {
    it('should remove HTML tags', () => {
      expect(sanitizeInput('<script>alert("xss")</script>Hello')).toBe('Hello');
    });

    it('should trim whitespace', () => {
      expect(sanitizeInput('  hello  ')).toBe('hello');
    });

    it('should handle null/undefined', () => {
      expect(sanitizeInput(null)).toBe('');
      expect(sanitizeInput(undefined)).toBe('');
    });
  });

  describe('validatePagination', () => {
    it('should return valid pagination params', () => {
      const result = validatePagination({ page: '2', limit: '20' });
      expect(result).toEqual({ page: 2, limit: 20, skip: 20 });
    });

    it('should use defaults for missing params', () => {
      const result = validatePagination({});
      expect(result).toEqual({ page: 1, limit: 10, skip: 0 });
    });

    it('should cap limit at maximum', () => {
      const result = validatePagination({ limit: '1000' });
      expect(result.limit).toBe(100); // max limit
    });
  });
});
```

---

## Mocking และ Stubbing

### ขั้นตอนที่ 786: Jest Mocking Techniques

```javascript
// src/__tests__/unit/middleware/auth.test.js
const jwt = require('jsonwebtoken');
const { authenticate, requireAuth, requireRole } = require('../../../middleware/auth');

jest.mock('jsonwebtoken');

describe('Auth Middleware', () => {
  let req, res, next;

  beforeEach(() => {
    req = {
      headers: {},
      user: null,
    };
    res = {
      status: jest.fn().mockReturnThis(),
      json: jest.fn().mockReturnThis(),
    };
    next = jest.fn();
  });

  describe('authenticate', () => {
    it('should set user when valid token provided', () => {
      const mockUser = { id: 'user123', role: 'USER' };
      req.headers.authorization = 'Bearer valid-token';
      jwt.verify.mockReturnValue(mockUser);

      authenticate(req, res, next);

      expect(req.user).toEqual(mockUser);
      expect(next).toHaveBeenCalledTimes(1);
      expect(next).not.toHaveBeenCalledWith(expect.any(Error));
    });

    it('should set user to null when no token', () => {
      authenticate(req, res, next);

      expect(req.user).toBeNull();
      expect(next).toHaveBeenCalled();
    });

    it('should set user to null for invalid token', () => {
      req.headers.authorization = 'Bearer invalid';
      jwt.verify.mockImplementation(() => { throw new Error('invalid token'); });

      authenticate(req, res, next);

      expect(req.user).toBeNull();
      expect(next).toHaveBeenCalled();
    });
  });

  describe('requireAuth', () => {
    it('should call next() when user is authenticated', () => {
      req.user = { id: 'user123' };
      requireAuth(req, res, next);
      expect(next).toHaveBeenCalled();
    });

    it('should return 401 when user is not authenticated', () => {
      req.user = null;
      requireAuth(req, res, next);

      expect(res.status).toHaveBeenCalledWith(401);
      expect(res.json).toHaveBeenCalledWith({ error: expect.any(String) });
      expect(next).not.toHaveBeenCalled();
    });
  });

  describe('requireRole', () => {
    it('should allow access for correct role', () => {
      req.user = { id: 'user123', role: 'ADMIN' };
      requireRole('ADMIN')(req, res, next);
      expect(next).toHaveBeenCalled();
    });

    it('should deny access for wrong role', () => {
      req.user = { id: 'user123', role: 'USER' };
      requireRole('ADMIN')(req, res, next);

      expect(res.status).toHaveBeenCalledWith(403);
      expect(next).not.toHaveBeenCalled();
    });
  });
});
```

### ขั้นตอนที่ 787: Mock External Services

```javascript
// src/__tests__/unit/services/emailService.test.js
const nodemailer = require('nodemailer');
const emailService = require('../../../services/emailService');

// Mock nodemailer
jest.mock('nodemailer', () => ({
  createTransporter: jest.fn().mockReturnValue({
    sendMail: jest.fn().mockResolvedValue({ messageId: 'mock-id-123' }),
    verify: jest.fn().mockResolvedValue(true),
  }),
}));

describe('EmailService', () => {
  let transporter;

  beforeEach(() => {
    transporter = nodemailer.createTransporter();
    jest.clearAllMocks();
  });

  it('should send welcome email', async () => {
    await emailService.sendWelcomeEmail('user@example.com', { name: 'John' });

    expect(transporter.sendMail).toHaveBeenCalledWith(
      expect.objectContaining({
        to: 'user@example.com',
        subject: expect.stringContaining('ยินดีต้อนรับ'),
      })
    );
  });

  it('should throw error if email sending fails', async () => {
    transporter.sendMail.mockRejectedValue(new Error('SMTP error'));

    await expect(
      emailService.sendWelcomeEmail('user@example.com', { name: 'John' })
    ).rejects.toThrow('SMTP error');
  });
});
```

---

## Integration Testing

### ขั้นตอนที่ 788: Database Integration Tests

```javascript
// jest.globalSetup.js
const { MongoMemoryServer } = require('mongodb-memory-server');

module.exports = async () => {
  const mongod = await MongoMemoryServer.create();
  process.env.MONGODB_URI = mongod.getUri();
  global.__MONGOD__ = mongod;
};
```

```javascript
// jest.globalTeardown.js
module.exports = async () => {
  await global.__MONGOD__.stop();
};
```

```javascript
// src/__tests__/integration/models/User.test.js
const mongoose = require('mongoose');
const User = require('../../../models/User');

describe('User Model Integration Tests', () => {
  beforeAll(async () => {
    await mongoose.connect(process.env.MONGODB_URI);
  });

  afterAll(async () => {
    await mongoose.disconnect();
  });

  beforeEach(async () => {
    await User.deleteMany({});
  });

  describe('User Creation', () => {
    it('should create user with valid data', async () => {
      const user = await User.create({
        username: 'testuser',
        email: 'test@example.com',
        password: 'password123',
      });

      expect(user._id).toBeDefined();
      expect(user.username).toBe('testuser');
      expect(user.email).toBe('test@example.com');
      expect(user.password).not.toBe('password123'); // ถูก hash แล้ว
      expect(user.role).toBe('USER');
    });

    it('should not create user with duplicate email', async () => {
      await User.create({
        username: 'user1',
        email: 'test@example.com',
        password: 'password123',
      });

      await expect(User.create({
        username: 'user2',
        email: 'test@example.com',
        password: 'password456',
      })).rejects.toThrow(/duplicate key/i);
    });

    it('should fail validation for missing required fields', async () => {
      await expect(User.create({ email: 'test@example.com' }))
        .rejects
        .toThrow(mongoose.Error.ValidationError);
    });
  });

  describe('Password Operations', () => {
    it('should hash password on save', async () => {
      const user = await User.create({
        username: 'testuser',
        email: 'test@example.com',
        password: 'plainpassword',
      });

      expect(user.password).not.toBe('plainpassword');
      expect(user.password).toMatch(/^\$2b\$/);  // bcrypt hash
    });

    it('should compare password correctly', async () => {
      const user = await User.create({
        username: 'testuser',
        email: 'test@example.com',
        password: 'mypassword',
      });

      expect(await user.comparePassword('mypassword')).toBe(true);
      expect(await user.comparePassword('wrongpassword')).toBe(false);
    });
  });
});
```

---

## API Testing

### ขั้นตอนที่ 789: API Integration Tests กับ Supertest

```bash
npm install --save-dev supertest
```

```javascript
// src/__tests__/integration/routes/auth.test.js
const request = require('supertest');
const mongoose = require('mongoose');
const app = require('../../../app');
const User = require('../../../models/User');

describe('Auth Routes', () => {
  beforeAll(async () => {
    await mongoose.connect(process.env.MONGODB_URI);
  });

  afterAll(async () => {
    await mongoose.disconnect();
  });

  beforeEach(async () => {
    await User.deleteMany({});
  });

  describe('POST /api/auth/register', () => {
    const validData = {
      username: 'testuser',
      email: 'test@example.com',
      password: 'Password123!',
    };

    it('should register a new user', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send(validData)
        .expect('Content-Type', /json/)
        .expect(201);

      expect(response.body).toMatchObject({
        user: {
          username: validData.username,
          email: validData.email,
        },
        token: expect.any(String),
      });

      // ตรวจสอบว่า password ไม่ถูกส่งออกมา
      expect(response.body.user.password).toBeUndefined();
    });

    it('should return 400 for invalid email', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({ ...validData, email: 'invalid-email' })
        .expect(400);

      expect(response.body).toHaveProperty('errors');
    });

    it('should return 409 for duplicate email', async () => {
      await User.create(validData);

      const response = await request(app)
        .post('/api/auth/register')
        .send(validData)
        .expect(409);

      expect(response.body.error).toMatch(/already exists/i);
    });
  });

  describe('POST /api/auth/login', () => {
    let user;

    beforeEach(async () => {
      user = await User.create({
        username: 'testuser',
        email: 'test@example.com',
        password: 'Password123!',
      });
    });

    it('should login with valid credentials', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({ email: 'test@example.com', password: 'Password123!' })
        .expect(200);

      expect(response.body.token).toBeDefined();
      expect(response.body.user.email).toBe('test@example.com');
    });

    it('should return 401 for wrong password', async () => {
      await request(app)
        .post('/api/auth/login')
        .send({ email: 'test@example.com', password: 'wrongpassword' })
        .expect(401);
    });

    it('should rate limit login attempts', async () => {
      const attempts = Array(6).fill(null).map(() =>
        request(app)
          .post('/api/auth/login')
          .send({ email: 'test@example.com', password: 'wrong' })
      );

      const responses = await Promise.all(attempts);
      const lastResponse = responses[responses.length - 1];

      expect(lastResponse.status).toBe(429);
    });
  });

  describe('Protected Routes', () => {
    let token;

    beforeEach(async () => {
      const user = await User.create({
        username: 'testuser',
        email: 'test@example.com',
        password: 'Password123!',
      });

      const response = await request(app)
        .post('/api/auth/login')
        .send({ email: 'test@example.com', password: 'Password123!' });

      token = response.body.token;
    });

    it('should access protected route with valid token', async () => {
      await request(app)
        .get('/api/users/me')
        .set('Authorization', `Bearer ${token}`)
        .expect(200);
    });

    it('should reject access without token', async () => {
      await request(app)
        .get('/api/users/me')
        .expect(401);
    });

    it('should reject access with invalid token', async () => {
      await request(app)
        .get('/api/users/me')
        .set('Authorization', 'Bearer invalid-token')
        .expect(401);
    });
  });
});
```

---

## E2E Testing กับ Playwright

### ขั้นตอนที่ 790: ติดตั้งและตั้งค่า Playwright

```bash
npm install --save-dev @playwright/test
npx playwright install
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  timeout: 30000,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,

  reporter: [
    ['html', { outputFolder: 'playwright-report' }],
    ['json', { outputFile: 'test-results.json' }],
    process.env.CI ? ['github'] : ['list'],
  ],

  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    trace: 'on-first-retry',
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
  ],
});
```

### ขั้นตอนที่ 791: E2E Test Examples

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/');
  });

  test('should display login form', async ({ page }) => {
    await page.goto('/login');
    
    await expect(page.locator('h1')).toContainText('เข้าสู่ระบบ');
    await expect(page.locator('#email')).toBeVisible();
    await expect(page.locator('#password')).toBeVisible();
    await expect(page.locator('button[type="submit"]')).toBeEnabled();
  });

  test('should login successfully', async ({ page }) => {
    await page.goto('/login');

    await page.fill('#email', 'test@example.com');
    await page.fill('#password', 'password123');
    await page.click('button[type="submit"]');

    // รอ redirect
    await page.waitForURL('/dashboard');

    // ตรวจสอบว่า login สำเร็จ
    await expect(page.locator('.user-name')).toContainText('test');
  });

  test('should show error for invalid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.fill('#email', 'wrong@example.com');
    await page.fill('#password', 'wrongpassword');
    await page.click('button[type="submit"]');

    const error = page.locator('.error-message');
    await expect(error).toBeVisible();
    await expect(error).toContainText('อีเมลหรือรหัสผ่านไม่ถูกต้อง');
  });

  test('should register new user', async ({ page }) => {
    await page.goto('/register');

    await page.fill('#username', `user${Date.now()}`);
    await page.fill('#email', `user${Date.now()}@example.com`);
    await page.fill('#password', 'Password123!');
    await page.fill('#confirmPassword', 'Password123!');

    await page.click('button[type="submit"]');

    await page.waitForURL('/dashboard');
    await expect(page.locator('.welcome-message')).toBeVisible();
  });
});
```

```typescript
// e2e/posts.spec.ts
import { test, expect, Page } from '@playwright/test';

// Helper function
async function loginUser(page: Page) {
  await page.goto('/login');
  await page.fill('#email', 'test@example.com');
  await page.fill('#password', 'password123');
  await page.click('button[type="submit"]');
  await page.waitForURL('/dashboard');
}

test.describe('Posts', () => {
  test.beforeEach(async ({ page }) => {
    await loginUser(page);
  });

  test('should create a new post', async ({ page }) => {
    await page.goto('/posts/create');

    await page.fill('#title', 'Test Post Title');
    await page.fill('#content', 'This is the post content');
    await page.selectOption('#status', 'PUBLISHED');

    // เพิ่ม tags
    await page.fill('#tags', 'nodejs');
    await page.keyboard.press('Enter');
    await page.fill('#tags', 'testing');
    await page.keyboard.press('Enter');

    await page.click('#submit-post');

    // ตรวจสอบ success message
    await expect(page.locator('.success-toast')).toContainText('สร้างบทความเรียบร้อย');

    // Verify post appears in list
    await page.goto('/posts');
    await expect(page.locator('text=Test Post Title')).toBeVisible();
  });

  test('should edit a post', async ({ page }) => {
    await page.goto('/posts');

    // Click edit on first post
    await page.locator('.post-card').first().locator('.edit-button').click();

    // Update title
    await page.fill('#title', 'Updated Post Title');
    await page.click('#submit-post');

    await expect(page.locator('.success-toast')).toContainText('แก้ไขบทความเรียบร้อย');
  });

  test('should search for posts', async ({ page }) => {
    await page.goto('/posts');

    await page.fill('#search', 'nodejs');
    await page.keyboard.press('Enter');

    // ตรวจสอบว่า results มี keyword
    const results = page.locator('.post-card');
    await expect(results).not.toHaveCount(0);
  });
});
```

---

## TDD - Test Driven Development

### ขั้นตอนที่ 792: TDD Workflow

```
TDD Cycle:
  1. RED   → เขียน test ที่ fail ก่อน
  2. GREEN → เขียน code ให้ test pass
  3. REFACTOR → ปรับปรุง code โดยไม่ทำ tests break
```

```javascript
// TDD Example - สร้าง PasswordValidator

// Step 1: RED - เขียน tests ก่อน
// src/__tests__/unit/utils/passwordValidator.test.js
const PasswordValidator = require('../../../utils/passwordValidator');

describe('PasswordValidator', () => {
  let validator;

  beforeEach(() => {
    validator = new PasswordValidator({
      minLength: 8,
      requireUppercase: true,
      requireLowercase: true,
      requireNumbers: true,
      requireSpecial: true,
    });
  });

  test('should pass for strong password', () => {
    expect(validator.validate('StrongP@ss1')).toEqual({ valid: true, errors: [] });
  });

  test('should fail for too short password', () => {
    const result = validator.validate('Sh0!');
    expect(result.valid).toBe(false);
    expect(result.errors).toContain('ต้องมีความยาวอย่างน้อย 8 ตัวอักษร');
  });

  test('should fail when missing uppercase', () => {
    const result = validator.validate('lowercase1@');
    expect(result.valid).toBe(false);
    expect(result.errors).toContain('ต้องมีตัวพิมพ์ใหญ่');
  });

  test('should fail when missing special character', () => {
    const result = validator.validate('Password123');
    expect(result.valid).toBe(false);
    expect(result.errors).toContain('ต้องมีอักขระพิเศษ');
  });

  test('should return multiple errors', () => {
    const result = validator.validate('weak');
    expect(result.errors.length).toBeGreaterThan(1);
  });
});

// Step 2: GREEN - เขียน code ให้ผ่าน tests
// src/utils/passwordValidator.js
class PasswordValidator {
  constructor(rules = {}) {
    this.rules = {
      minLength: rules.minLength || 8,
      requireUppercase: rules.requireUppercase !== false,
      requireLowercase: rules.requireLowercase !== false,
      requireNumbers: rules.requireNumbers !== false,
      requireSpecial: rules.requireSpecial !== false,
    };
  }

  validate(password) {
    const errors = [];

    if (password.length < this.rules.minLength) {
      errors.push(`ต้องมีความยาวอย่างน้อย ${this.rules.minLength} ตัวอักษร`);
    }

    if (this.rules.requireUppercase && !/[A-Z]/.test(password)) {
      errors.push('ต้องมีตัวพิมพ์ใหญ่');
    }

    if (this.rules.requireLowercase && !/[a-z]/.test(password)) {
      errors.push('ต้องมีตัวพิมพ์เล็ก');
    }

    if (this.rules.requireNumbers && !/\d/.test(password)) {
      errors.push('ต้องมีตัวเลข');
    }

    if (this.rules.requireSpecial && !/[!@#$%^&*(),.?":{}|<>]/.test(password)) {
      errors.push('ต้องมีอักขระพิเศษ');
    }

    return { valid: errors.length === 0, errors };
  }
}

module.exports = PasswordValidator;

// Step 3: REFACTOR - ปรับปรุงให้ดีขึ้น
```

---

## BDD - Behavior Driven Development

### ขั้นตอนที่ 793: BDD กับ Jest

```javascript
// BDD style - ใช้ describe/it ที่อ่านง่ายเหมือนภาษาธรรมชาติ

// src/__tests__/bdd/shopping-cart.test.js
describe('ตะกร้าสินค้า', () => {
  describe('เมื่อเพิ่มสินค้า', () => {
    it('ควรเพิ่มสินค้าใหม่ลงในตะกร้า', () => {
      const cart = new ShoppingCart();
      cart.addItem({ id: 1, name: 'สินค้า A', price: 100 });
      
      expect(cart.items).toHaveLength(1);
      expect(cart.items[0].name).toBe('สินค้า A');
    });

    it('ควรอัปเดตจำนวนถ้าสินค้าอยู่ในตะกร้าแล้ว', () => {
      const cart = new ShoppingCart();
      cart.addItem({ id: 1, name: 'สินค้า A', price: 100 });
      cart.addItem({ id: 1, name: 'สินค้า A', price: 100 });

      expect(cart.items).toHaveLength(1);
      expect(cart.items[0].quantity).toBe(2);
    });
  });

  describe('เมื่อคำนวณราคา', () => {
    it('ควรคำนวณราคารวมอย่างถูกต้อง', () => {
      const cart = new ShoppingCart();
      cart.addItem({ id: 1, name: 'สินค้า A', price: 100, quantity: 2 });
      cart.addItem({ id: 2, name: 'สินค้า B', price: 50, quantity: 3 });

      expect(cart.total).toBe(350);
    });

    it('ควรใช้ส่วนลดถ้ามีโค้ด', () => {
      const cart = new ShoppingCart();
      cart.addItem({ id: 1, name: 'สินค้า A', price: 1000 });
      cart.applyDiscount('SAVE10', 10); // 10% off

      expect(cart.total).toBe(900);
    });
  });

  describe('เมื่อ checkout', () => {
    it('ควร throw error ถ้าตะกร้าว่าง', () => {
      const cart = new ShoppingCart();
      expect(() => cart.checkout()).toThrow('ตะกร้าสินค้าว่าง');
    });

    it('ควรสร้าง order เมื่อ checkout สำเร็จ', async () => {
      const cart = new ShoppingCart();
      cart.addItem({ id: 1, name: 'สินค้า A', price: 100 });

      const order = await cart.checkout({ userId: 'user123' });

      expect(order).toMatchObject({
        userId: 'user123',
        items: expect.any(Array),
        totalAmount: 100,
        status: 'PENDING',
      });
    });
  });
});
```

---

## Test Coverage

### ขั้นตอนที่ 794: Coverage Configuration

```javascript
// jest.config.js - Coverage settings
module.exports = {
  collectCoverage: process.env.COVERAGE === 'true',
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/index.js',
    '!src/config/**',
    '!src/**/__tests__/**',
    '!src/**/*.test.js',
  ],
  coverageThresholds: {
    global: {
      branches: 80,
      functions: 85,
      lines: 85,
      statements: 85,
    },
    // เพิ่ม threshold สำหรับ critical modules
    './src/services/': {
      branches: 90,
      functions: 95,
      lines: 90,
    },
  },
  coverageReporters: [
    'text',
    'text-summary',
    'lcov',
    'html',
    'json',
    'cobertura',  // สำหรับ CI/CD
  ],
  coverageDirectory: 'coverage',
};
```

### ขั้นตอนที่ 795: Coverage Reports

```bash
# Run tests พร้อม coverage
npm test -- --coverage

# ดู HTML report
open coverage/index.html

# Coverage ใน CI
npm test -- --coverage --coverageReporters=json-summary

# Check coverage threshold
cat coverage/coverage-summary.json | \
  node -e "
    const data = require('/dev/stdin');
    const total = data.total;
    const failed = [];
    ['lines', 'statements', 'functions', 'branches'].forEach(metric => {
      if (total[metric].pct < 80) {
        failed.push(\`\${metric}: \${total[metric].pct}% (< 80%)\`);
      }
    });
    if (failed.length > 0) {
      console.error('Coverage thresholds not met:', failed);
      process.exit(1);
    }
    console.log('Coverage thresholds met!');
  "
```

---

## Testing Async Code

### ขั้นตอนที่ 796: Async Testing Patterns

```javascript
// Testing Promises
it('should resolve with user data', async () => {
  const user = await userService.getUser('user123');
  expect(user.email).toBe('test@example.com');
});

// Testing Rejections
it('should reject for invalid ID', async () => {
  await expect(userService.getUser('invalid'))
    .rejects
    .toThrow('User not found');
});

// Testing with done callback (old style)
it('should call callback with data', (done) => {
  getData((err, data) => {
    if (err) return done(err);
    expect(data).toBeDefined();
    done();
  });
});

// Testing timers
it('should call function after delay', () => {
  jest.useFakeTimers();
  const callback = jest.fn();

  setTimeout(callback, 1000);
  expect(callback).not.toHaveBeenCalled();

  jest.runAllTimers();
  expect(callback).toHaveBeenCalledTimes(1);

  jest.useRealTimers();
});

// Testing observables/streams
it('should emit values', (done) => {
  const stream = createStream();
  const values = [];

  stream.on('data', (value) => values.push(value));
  stream.on('end', () => {
    expect(values).toEqual([1, 2, 3]);
    done();
  });
});
```

---

## Snapshot Testing

### ขั้นตอนที่ 797: Jest Snapshots

```javascript
// src/__tests__/unit/utils/formatter.test.js
const { formatUser, formatPost, formatError } = require('../../../utils/formatter');

describe('Formatter', () => {
  it('should format user correctly', () => {
    const user = {
      _id: 'user123',
      username: 'testuser',
      email: 'test@example.com',
      password: 'hashedpassword',
      role: 'USER',
      createdAt: new Date('2024-01-01'),
    };

    const formatted = formatUser(user);

    // ตรวจสอบ structure
    expect(formatted).toMatchSnapshot();

    // ตรวจสอบว่า password ไม่อยู่ใน output
    expect(formatted).not.toHaveProperty('password');
  });

  it('should format error response', () => {
    const error = {
      message: 'Validation failed',
      errors: ['Email is required', 'Password too short'],
    };

    expect(formatError(error)).toMatchSnapshot();
  });
});
```

---

## Test Fixtures และ Factories

### ขั้นตอนที่ 798: Test Data Factories

```javascript
// src/__tests__/factories/userFactory.js
const { faker } = require('@faker-js/faker');
const User = require('../../models/User');

class UserFactory {
  static build(overrides = {}) {
    return {
      username: faker.internet.username(),
      email: faker.internet.email(),
      password: 'Password123!',
      role: 'USER',
      profile: {
        bio: faker.lorem.sentence(),
        avatar: faker.image.avatar(),
      },
      ...overrides,
    };
  }

  static async create(overrides = {}) {
    const data = this.build(overrides);
    return User.create(data);
  }

  static async createMany(count = 5, overrides = {}) {
    return Promise.all(
      Array(count).fill(null).map(() => this.create(overrides))
    );
  }

  static buildAdmin(overrides = {}) {
    return this.build({ role: 'ADMIN', ...overrides });
  }
}

module.exports = UserFactory;
```

```javascript
// ใช้งาน factory ใน tests
const UserFactory = require('../factories/userFactory');

describe('User Service', () => {
  it('should get all users', async () => {
    await UserFactory.createMany(10);
    const users = await userService.getAll();
    expect(users.length).toBe(10);
  });

  it('should not allow regular user to access admin features', async () => {
    const user = await UserFactory.create();
    // test with user data...
  });
});
```

---

## Testing แบบ Pattern

### ขั้นตอนที่ 799: AAA Pattern

```javascript
// Arrange, Act, Assert pattern
test('should calculate order total with discount', () => {
  // ARRANGE - เตรียม data
  const cart = new Cart();
  const items = [
    { id: 1, price: 100, quantity: 2 },
    { id: 2, price: 50, quantity: 1 },
  ];
  const discount = 10; // 10%

  items.forEach(item => cart.addItem(item));
  cart.setDiscount(discount);

  // ACT - ทำ action ที่ต้องการ test
  const total = cart.calculateTotal();

  // ASSERT - ตรวจสอบผลลัพธ์
  expect(total).toBe(225); // (100*2 + 50*1) * 0.9
});
```

### ขั้นตอนที่ 800: Testing Best Practices Summary

```javascript
// ✅ Good Testing Practices

// 1. Test behavior ไม่ใช่ implementation
it('should calculate correct total', () => {
  const cart = createCart([{ price: 100 }, { price: 50 }]);
  expect(cart.total).toBe(150);  // ✅ ทดสอบ behavior
  // ❌ expect(cart._internalTotal).toBe(150);  // ไม่ดี - ทดสอบ implementation
});

// 2. One assertion per test (ถ้าเป็นไปได้)
it('should return user with correct email', async () => {
  const user = await createUser({ email: 'test@example.com' });
  expect(user.email).toBe('test@example.com');
});

// 3. Test edge cases
it('should handle empty array', () => {
  expect(sum([])).toBe(0);
});

it('should handle large numbers', () => {
  expect(sum([Number.MAX_SAFE_INTEGER, 1])).toBeDefined();
});

// 4. Descriptive test names
it('should return 401 when accessing protected route without token', async () => {
  // ชัดเจนว่า test อะไร
});

// 5. Setup/Teardown ที่ดี
describe('User operations', () => {
  let db;

  beforeAll(async () => {
    db = await createTestDatabase();
  });

  afterAll(async () => {
    await db.close();
  });

  afterEach(async () => {
    await db.clear();  // Clean state ระหว่าง tests
  });
});

// 6. Avoid test interdependencies
// ❌ Tests ต้องรันตามลำดับ
// ✅ แต่ละ test รันได้อิสระ

// 7. Mock external dependencies
jest.mock('../services/emailService');  // ไม่ส่ง emails จริงๆ ใน tests
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: TDD Implementation

```javascript
// TODO: ใช้ TDD สร้าง BankAccount class
// เขียน tests ก่อน:
// - deposit(amount)
// - withdraw(amount)
// - getBalance()
// - transfer(amount, targetAccount)
// - ประวัติ transactions
```

### แบบฝึกหัดที่ 2: E2E Test Suite

```typescript
// TODO: สร้าง Playwright E2E tests สำหรับ:
// - ลงทะเบียนผู้ใช้ใหม่
// - เข้าสู่ระบบ
// - สร้างบทความ
// - แก้ไขบทความ
// - ลบบทความ
// - ค้นหาบทความ
```

### แบบฝึกหัดที่ 3: Test Coverage

```javascript
// TODO: เขียน tests ให้ครอบคลุม 90%+ coverage
// สำหรับ modules เหล่านี้:
// - userService.js
// - postService.js
// - authMiddleware.js
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Testing Pyramid** - unit, integration, E2E
2. **Jest** - configuration, matchers, async testing
3. **Mocking** - jest.mock, spies, stubs
4. **Integration Testing** - database, API testing
5. **Supertest** - HTTP endpoint testing
6. **Playwright** - E2E browser testing
7. **TDD** - Red-Green-Refactor cycle
8. **BDD** - Behavior Driven Development
9. **Coverage** - configuration, thresholds
10. **Best Practices** - AAA pattern, test isolation

ในบทถัดไปเราจะเรียนรู้ **Advanced Security** - OWASP Top 10, penetration testing, security headers
