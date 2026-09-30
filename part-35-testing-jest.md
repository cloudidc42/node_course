# Part 35: Testing ด้วย Jest

> ขั้นตอนที่ 35-35 จาก 1000

---

## สารบัญ

1. [Jest คืออะไร](#jest-คืออะไร)
2. [Jest Setup](#jest-setup)
3. [Unit Tests เบื้องต้น](#unit-tests-เบื้องต้น)
4. [describe/it/expect](#describeitexpect)
5. [Mocking](#mocking)
6. [Test Coverage](#test-coverage)
7. [Snapshot Testing](#snapshot-testing)
8. [Async Testing](#async-testing)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Jest คืออะไร

Jest เป็น JavaScript Testing Framework จาก Meta (Facebook) มีคุณสมบัติ:
- **Zero configuration**: ใช้ได้ทันทีโดยไม่ต้อง config มาก
- **Built-in assertions**: มี expect() API ครบครัน
- **Mocking**: จัดการ mocks และ spies ในตัว
- **Coverage**: ดู test coverage ได้ทันที
- **Fast**: รัน tests แบบ parallel
- **Watch mode**: re-run tests เมื่อ code เปลี่ยน

---

## Jest Setup

### ติดตั้ง

```bash
# ติดตั้ง Jest
npm install --save-dev jest

# สำหรับ ES Modules
npm install --save-dev jest @babel/core @babel/preset-env babel-jest

# สำหรับ TypeScript
npm install --save-dev jest @types/jest ts-jest typescript
```

### package.json Configuration

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --forceExit"
  },
  "jest": {
    "testEnvironment": "node",
    "testMatch": [
      "**/__tests__/**/*.test.js",
      "**/*.test.js",
      "**/*.spec.js"
    ],
    "collectCoverageFrom": [
      "src/**/*.js",
      "!src/**/*.test.js",
      "!src/index.js"
    ],
    "coverageThreshold": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      }
    },
    "setupFilesAfterFramework": ["./jest.setup.js"],
    "testTimeout": 10000
  }
}
```

### jest.config.js (แบบ file)

```javascript
// jest.config.js
module.exports = {
  testEnvironment: 'node',
  
  // ไฟล์ test pattern
  testMatch: ['**/__tests__/**/*.js', '**/*.test.js'],
  
  // ไม่ test ไฟล์พวกนี้
  testPathIgnorePatterns: ['/node_modules/', '/dist/'],
  
  // Setup ก่อน run tests
  setupFilesAfterFramework: ['<rootDir>/jest.setup.js'],
  
  // Coverage
  collectCoverage: false, // เปิดตอนรัน --coverage
  collectCoverageFrom: [
    'src/**/*.{js,ts}',
    '!src/**/*.test.{js,ts}',
    '!src/index.{js,ts}',
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  
  // Timeout
  testTimeout: 10000,
  
  // Module aliases
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@config/(.*)$': '<rootDir>/src/config/$1',
  },
  
  // Globals
  globals: {
    'ts-jest': {
      tsconfig: 'tsconfig.json',
    },
  },
};
```

### jest.setup.js

```javascript
// jest.setup.js
const mongoose = require('mongoose');

// ตั้งค่า environment variables สำหรับ testing
process.env.NODE_ENV = 'test';
process.env.JWT_SECRET = 'test-secret-key-minimum-32-characters-long';
process.env.PORT = '4000';

// Global setup
beforeAll(async () => {
  // Connect test database
});

afterAll(async () => {
  // Disconnect
  await mongoose.disconnect();
});

// เพิ่ม custom matchers
expect.extend({
  toBeWithinRange(received, floor, ceiling) {
    const pass = received >= floor && received <= ceiling;
    if (pass) {
      return {
        message: () => `expected ${received} not to be within range ${floor} - ${ceiling}`,
        pass: true,
      };
    } else {
      return {
        message: () => `expected ${received} to be within range ${floor} - ${ceiling}`,
        pass: false,
      };
    }
  },
});
```

---

## Unit Tests เบื้องต้น

### โครงสร้างไฟล์ Test

```
src/
├── utils/
│   ├── helpers.js
│   └── helpers.test.js      # test อยู่ข้างๆ source
├── services/
│   ├── authService.js
│   └── authService.test.js
└── __tests__/               # หรือรวมไว้ใน folder นี้
    ├── helpers.test.js
    └── authService.test.js
```

### Test ฟังก์ชัน Utility

```javascript
// src/utils/helpers.js
function calculateTax(price, taxRate = 0.07) {
  if (price < 0) throw new Error('Price cannot be negative');
  return Math.round(price * taxRate * 100) / 100;
}

function formatCurrency(amount, currency = 'THB') {
  return new Intl.NumberFormat('th-TH', {
    style: 'currency',
    currency,
  }).format(amount);
}

function slugify(text) {
  return text
    .toString()
    .toLowerCase()
    .trim()
    .replace(/\s+/g, '-')
    .replace(/[^\w-]+/g, '')
    .replace(/--+/g, '-');
}

function paginate(total, page = 1, limit = 10) {
  const totalPages = Math.ceil(total / limit);
  const offset = (page - 1) * limit;
  
  return {
    total,
    page,
    limit,
    totalPages,
    offset,
    hasNextPage: page < totalPages,
    hasPrevPage: page > 1,
  };
}

module.exports = { calculateTax, formatCurrency, slugify, paginate };
```

```javascript
// src/utils/helpers.test.js
const { calculateTax, slugify, paginate } = require('./helpers');

describe('calculateTax', () => {
  it('calculates 7% tax correctly', () => {
    expect(calculateTax(100)).toBe(7);
    expect(calculateTax(200)).toBe(14);
    expect(calculateTax(150)).toBe(10.5);
  });
  
  it('uses custom tax rate', () => {
    expect(calculateTax(100, 0.1)).toBe(10);
    expect(calculateTax(100, 0.15)).toBe(15);
  });
  
  it('rounds to 2 decimal places', () => {
    expect(calculateTax(333.33)).toBe(23.33);
    expect(calculateTax(100, 0.333)).toBe(33.3);
  });
  
  it('throws error for negative price', () => {
    expect(() => calculateTax(-100)).toThrow('Price cannot be negative');
    expect(() => calculateTax(-1)).toThrow(Error);
  });
  
  it('handles zero price', () => {
    expect(calculateTax(0)).toBe(0);
  });
});

describe('slugify', () => {
  it('converts text to slug', () => {
    expect(slugify('Hello World')).toBe('hello-world');
    expect(slugify('Node.js Tutorial')).toBe('nodejs-tutorial');
    expect(slugify('  spaces  ')).toBe('spaces');
  });
  
  it('handles special characters', () => {
    expect(slugify('Hello & World!')).toBe('hello-world');
    expect(slugify('100% Done')).toBe('100-done');
  });
  
  it('handles multiple spaces/dashes', () => {
    expect(slugify('hello   world')).toBe('hello-world');
    expect(slugify('hello---world')).toBe('hello-world');
  });
  
  it('handles Thai text', () => {
    expect(slugify('สวัสดี World')).toBe('world');
  });
});

describe('paginate', () => {
  it('calculates pagination correctly', () => {
    const result = paginate(100, 1, 10);
    
    expect(result).toEqual({
      total: 100,
      page: 1,
      limit: 10,
      totalPages: 10,
      offset: 0,
      hasNextPage: true,
      hasPrevPage: false,
    });
  });
  
  it('calculates middle page correctly', () => {
    const result = paginate(100, 5, 10);
    
    expect(result.offset).toBe(40);
    expect(result.hasNextPage).toBe(true);
    expect(result.hasPrevPage).toBe(true);
  });
  
  it('calculates last page correctly', () => {
    const result = paginate(100, 10, 10);
    
    expect(result.hasNextPage).toBe(false);
    expect(result.hasPrevPage).toBe(true);
  });
  
  it('handles partial last page', () => {
    const result = paginate(25, 1, 10);
    expect(result.totalPages).toBe(3);
  });
  
  it('uses defaults', () => {
    const result = paginate(50);
    expect(result.page).toBe(1);
    expect(result.limit).toBe(10);
  });
});
```

---

## describe/it/expect

### Describe Blocks

```javascript
// describe ใช้จัดกลุ่ม tests ที่เกี่ยวข้องกัน
describe('UserService', () => {
  
  describe('createUser', () => {
    it('creates user with valid data', async () => { /* ... */ });
    it('throws error for duplicate email', async () => { /* ... */ });
    it('hashes password before saving', async () => { /* ... */ });
  });
  
  describe('findUser', () => {
    it('finds user by id', async () => { /* ... */ });
    it('returns null for non-existent user', async () => { /* ... */ });
  });
  
  // Nested describe
  describe('authentication', () => {
    describe('login', () => {
      it('returns token for valid credentials', async () => { /* ... */ });
      it('throws error for wrong password', async () => { /* ... */ });
    });
  });
});
```

### Lifecycle Hooks

```javascript
describe('Database Tests', () => {
  let connection;
  let testUserId;
  
  // รันครั้งเดียวก่อน tests ทั้งหมดใน describe นี้
  beforeAll(async () => {
    connection = await connectToTestDB();
    await seedTestData();
  });
  
  // รันครั้งเดียวหลัง tests ทั้งหมด
  afterAll(async () => {
    await cleanupTestData();
    await connection.close();
  });
  
  // รันก่อนทุก test
  beforeEach(async () => {
    testUserId = await createTestUser();
  });
  
  // รันหลังทุก test
  afterEach(async () => {
    await deleteTestUser(testUserId);
    jest.clearAllMocks(); // ล้าง mocks
  });
  
  it('test 1', async () => { /* ... */ });
  it('test 2', async () => { /* ... */ });
});
```

### Expect Matchers ทั้งหมด

```javascript
// ตัวเลข
expect(2 + 2).toBe(4);                    // ===
expect(0.1 + 0.2).toBeCloseTo(0.3, 5);   // floating point
expect(10).toBeGreaterThan(5);
expect(10).toBeGreaterThanOrEqual(10);
expect(5).toBeLessThan(10);
expect(5).toBeLessThanOrEqual(5);

// String
expect('hello world').toMatch(/world/);
expect('hello world').toContain('world');
expect('hello').toHaveLength(5);

// Objects
expect({a: 1, b: 2}).toEqual({a: 1, b: 2});         // deep equal
expect({a: 1, b: 2}).toMatchObject({a: 1});          // partial match
expect({a: 1}).toHaveProperty('a');
expect({a: 1}).toHaveProperty('a', 1);
expect({a: {b: 2}}).toHaveProperty('a.b', 2);

// Arrays
expect([1, 2, 3]).toContain(2);
expect([1, 2, 3]).toHaveLength(3);
expect([{a: 1}]).toContainEqual({a: 1});
expect([1, 2, 3]).toEqual(expect.arrayContaining([1, 3]));

// Booleans
expect(true).toBe(true);
expect(null).toBeNull();
expect(undefined).toBeUndefined();
expect('value').toBeDefined();
expect(null).toBeFalsy();
expect('text').toBeTruthy();

// Errors
expect(() => { throw new Error('oops') }).toThrow('oops');
expect(() => { throw new Error('oops') }).toThrow(Error);
expect(() => { throw new TypeError('type') }).toThrowError(TypeError);

// Promises/Async
await expect(Promise.resolve(42)).resolves.toBe(42);
await expect(Promise.reject(new Error('fail'))).rejects.toThrow('fail');
await expect(asyncFn()).resolves.toMatchObject({ status: 'ok' });

// NOT
expect(5).not.toBe(6);
expect(null).not.toBeDefined();
```

---

## Mocking

### Manual Mocks

```javascript
// __mocks__/emailService.js - Mock module
module.exports = {
  sendEmail: jest.fn().mockResolvedValue({ messageId: 'test-id' }),
  sendVerificationEmail: jest.fn().mockResolvedValue(true),
  sendPasswordResetEmail: jest.fn().mockResolvedValue(true),
};
```

```javascript
// ใน test file
jest.mock('../services/emailService');

const emailService = require('../services/emailService');
const authService = require('../services/authService');

describe('AuthService', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });
  
  it('sends verification email on register', async () => {
    await authService.register({
      name: 'Test',
      email: 'test@example.com',
      password: 'password123',
    });
    
    expect(emailService.sendVerificationEmail).toHaveBeenCalledTimes(1);
    expect(emailService.sendVerificationEmail).toHaveBeenCalledWith(
      'test@example.com',
      expect.any(String) // verification token
    );
  });
});
```

### jest.fn() - Function Mocks

```javascript
describe('Mock Functions', () => {
  it('creates mock function', () => {
    const mockFn = jest.fn();
    
    mockFn(1);
    mockFn(2, 3);
    
    expect(mockFn).toHaveBeenCalled();
    expect(mockFn).toHaveBeenCalledTimes(2);
    expect(mockFn).toHaveBeenCalledWith(1);
    expect(mockFn).toHaveBeenLastCalledWith(2, 3);
    expect(mockFn).toHaveBeenNthCalledWith(1, 1);
  });
  
  it('mock return values', () => {
    const mockFn = jest.fn()
      .mockReturnValueOnce(1)
      .mockReturnValueOnce(2)
      .mockReturnValue(0); // default
    
    expect(mockFn()).toBe(1);
    expect(mockFn()).toBe(2);
    expect(mockFn()).toBe(0);
    expect(mockFn()).toBe(0);
  });
  
  it('mock async function', async () => {
    const mockFetch = jest.fn()
      .mockResolvedValueOnce({ data: 'first' })
      .mockRejectedValueOnce(new Error('Network error'));
    
    const first = await mockFetch();
    expect(first).toEqual({ data: 'first' });
    
    await expect(mockFetch()).rejects.toThrow('Network error');
  });
  
  it('mock implementation', () => {
    const mockAdd = jest.fn().mockImplementation((a, b) => a + b);
    expect(mockAdd(2, 3)).toBe(5);
  });
});
```

### Mocking Modules

```javascript
// Mock entire module
jest.mock('axios');
const axios = require('axios');

// กำหนด mock behavior
axios.get.mockResolvedValue({
  data: { users: [{ id: 1, name: 'John' }] },
  status: 200,
});

// Mock partial module
jest.mock('../config', () => ({
  ...jest.requireActual('../config'), // ใช้ค่าจริง
  database: {
    host: 'localhost',
    name: 'test_db',
  },
}));

// Mock third-party
jest.mock('nodemailer', () => ({
  createTransport: jest.fn().mockReturnValue({
    sendMail: jest.fn().mockResolvedValue({ messageId: 'test' }),
  }),
}));
```

### Spy on Methods

```javascript
const userService = require('../services/userService');

describe('Spying', () => {
  it('spies on method', async () => {
    const spy = jest.spyOn(userService, 'findById')
      .mockResolvedValue({ id: '123', name: 'Test' });
    
    // ทดสอบ code ที่เรียกใช้ userService.findById
    const result = await someFunction('123');
    
    expect(spy).toHaveBeenCalledWith('123');
    expect(result).toBeDefined();
    
    spy.mockRestore(); // คืนค่า original
  });
  
  it('spies without mocking', () => {
    const consoleSpy = jest.spyOn(console, 'log');
    
    someFunction(); // function ที่เรียก console.log
    
    expect(consoleSpy).toHaveBeenCalled();
    consoleSpy.mockRestore();
  });
});
```

---

## Test Coverage

### รัน Coverage

```bash
# รัน tests พร้อม coverage report
npm test -- --coverage

# Coverage threshold - fail ถ้า coverage ต่ำกว่าที่กำหนด
jest --coverage --coverageThreshold='{"global":{"lines":80}}'
```

### Coverage Report

```
PASS  src/utils/helpers.test.js
PASS  src/services/authService.test.js

--------------------|---------|----------|---------|---------|
File                | % Stmts | % Branch | % Funcs | % Lines |
--------------------|---------|----------|---------|---------|
All files           |   85.71 |    72.22 |   88.89 |   86.67 |
 utils/             |         |          |         |         |
  helpers.js        |   94.74 |    83.33 |     100 |   94.74 |
 services/          |         |          |         |         |
  authService.js    |   78.57 |    66.67 |   77.78 |   80.00 |
--------------------|---------|----------|---------|---------|
```

### Coverage Configuration

```javascript
// jest.config.js
module.exports = {
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/*.test.js',
    '!src/index.js',
    '!src/config/**',      // ไม่ต้อง test config
    '!src/**/*.d.ts',
  ],
  
  coverageDirectory: 'coverage',
  coverageReporters: [
    'text',        // console output
    'lcov',        // สำหรับ CI tools (SonarQube, Codecov)
    'html',        // เปิดใน browser
    'json',        // JSON format
    'json-summary', // summary JSON
  ],
  
  coverageThreshold: {
    global: {
      branches: 75,
      functions: 80,
      lines: 80,
      statements: 80,
    },
    // Per-file thresholds
    './src/services/': {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90,
    },
  },
};
```

---

## Snapshot Testing

Snapshot testing เหมาะสำหรับ test ว่า output ไม่เปลี่ยนแปลง

```javascript
// ตัวอย่าง snapshot test
const { formatUserResponse } = require('../utils/responseFormatter');

describe('Response Formatter', () => {
  const mockUser = {
    _id: '507f1f77bcf86cd799439011',
    name: 'John Doe',
    email: 'john@example.com',
    password: 'hashedpassword',
    role: 'user',
    createdAt: new Date('2024-01-01'),
    __v: 0,
  };
  
  it('formats user response correctly', () => {
    const formatted = formatUserResponse(mockUser);
    
    // ครั้งแรกที่รัน: สร้าง snapshot
    // ครั้งต่อไป: เปรียบเทียบกับ snapshot
    expect(formatted).toMatchSnapshot();
  });
  
  it('excludes sensitive fields', () => {
    const formatted = formatUserResponse(mockUser);
    
    expect(formatted).toMatchSnapshot({
      createdAt: expect.any(Date), // ค่าที่เปลี่ยนทุกครั้งใช้ matcher
    });
    
    expect(formatted.password).toBeUndefined();
    expect(formatted.__v).toBeUndefined();
  });
  
  // Inline snapshot
  it('has correct structure', () => {
    const result = formatUserResponse(mockUser);
    
    expect(result).toMatchInlineSnapshot(`
      Object {
        "email": "john@example.com",
        "id": "507f1f77bcf86cd799439011",
        "name": "John Doe",
        "role": "user",
      }
    `);
  });
});
```

```bash
# อัพเดต snapshots เมื่อ format เปลี่ยนจริงๆ
jest --updateSnapshot
# หรือ
jest -u
```

---

## Async Testing

```javascript
// ทดสอบ Promises
describe('Async Tests', () => {
  // วิธีที่ 1: return Promise
  it('resolves with data', () => {
    return fetchUser(1).then(user => {
      expect(user.name).toBe('John');
    });
  });
  
  // วิธีที่ 2: async/await (แนะนำ)
  it('fetches user data', async () => {
    const user = await fetchUser(1);
    expect(user.name).toBe('John');
  });
  
  // วิธีที่ 3: resolves/rejects
  it('resolves', async () => {
    await expect(fetchUser(1)).resolves.toMatchObject({ name: 'John' });
  });
  
  it('rejects with error', async () => {
    await expect(fetchUser(999)).rejects.toThrow('User not found');
    await expect(fetchUser(999)).rejects.toMatchObject({
      message: 'User not found',
      statusCode: 404,
    });
  });
  
  // Test timeout
  it('completes within time limit', async () => {
    jest.setTimeout(5000);
    const result = await longRunningOperation();
    expect(result).toBeDefined();
  }, 5000); // timeout สำหรับ test นี้
});
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic Unit Tests

เขียน tests สำหรับ utility functions:

```javascript
// src/utils/validators.js - เขียน test ให้ functions เหล่านี้

function isValidEmail(email) {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

function isStrongPassword(password) {
  return password.length >= 8 &&
    /[A-Z]/.test(password) &&
    /[a-z]/.test(password) &&
    /[0-9]/.test(password);
}

function parseQueryParams(queryString) {
  // TODO: implement
}

module.exports = { isValidEmail, isStrongPassword, parseQueryParams };
```

### แบบฝึกหัดที่ 2: Mocking Dependencies

เขียน test สำหรับ UserService ที่ mock:
- Database model
- Email service
- JWT library

```javascript
// src/services/userService.js
const User = require('../models/User');
const emailService = require('./emailService');

class UserService {
  async register(data) {
    const user = await User.create(data);
    await emailService.sendWelcomeEmail(user.email);
    return user;
  }
  // ...
}

// tests/userService.test.js - TODO
```

### แบบฝึกหัดที่ 3: Coverage Report

รัน coverage report และเพิ่ม coverage ให้ถึง 80%:
1. รัน `npm test -- --coverage`
2. ดูว่า branches ไหน uncovered
3. เขียน tests เพิ่มให้ครอบคลุม edge cases

---

## สรุป

Jest เป็น testing framework ที่สมบูรณ์สำหรับ Node.js

| Feature | ใช้งาน |
|---------|--------|
| describe/it | จัดกลุ่มและ define tests |
| expect | assert ผลลัพธ์ |
| jest.fn() | mock functions |
| jest.mock() | mock modules |
| jest.spyOn() | spy on methods |
| Snapshots | test output ไม่เปลี่ยน |
| Coverage | วัดความครอบคลุมของ tests |

**ถัดไป**: [Part 36: Unit Testing →](./part-36-unit-testing.md)
