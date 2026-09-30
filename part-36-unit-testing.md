# Part 36: Unit Testing เชิงลึก

> ขั้นตอนที่ 36-36 จาก 1000

---

## สารบัญ

1. [Test Pyramid](#test-pyramid)
2. [Unit vs Integration Tests](#unit-vs-integration-tests)
3. [Test Doubles (Mock, Stub, Spy)](#test-doubles)
4. [Testing Utilities](#testing-utilities)
5. [TDD Approach](#tdd-approach)
6. [Best Practices](#best-practices)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Test Pyramid

Test Pyramid เป็นแนวคิดที่แสดงสัดส่วนของ tests ประเภทต่างๆ ที่ควรมี

```
           /\
          /  \
         / E2E\        10% - 느린, ค่าแพง, fragile
        /------\
       / Integ. \      30% - ทดสอบหลาย components ร่วมกัน
      /----------\
     /    Unit    \    60% - เร็ว, ถูก, isolated
    /--------------\

Unit Tests (ฐาน):
  - เร็วมาก (milliseconds)
  - ทดสอบ function/class เดียว
  - ไม่ต้องการ external dependencies
  - ควรมีมากที่สุด

Integration Tests (กลาง):
  - ช้ากว่า
  - ทดสอบหลาย components ร่วมกัน
  - อาจต้องการ DB หรือ service จริง

E2E Tests (ยอด):
  - ช้าที่สุด
  - ทดสอบ full flow จาก user perspective
  - ควรมีน้อยที่สุด แต่ครอบ critical paths
```

### ทำไม Unit Tests สำคัญ

```javascript
// สมมติมี function นี้
function calculateDiscount(price, userType, quantity) {
  let discount = 0;
  
  if (userType === 'premium') {
    discount += 0.1;
  }
  
  if (quantity >= 10) {
    discount += 0.05;
  }
  
  if (quantity >= 50) {
    discount += 0.1; // additional discount
  }
  
  if (price > 1000) {
    discount += 0.05;
  }
  
  return Math.round(price * (1 - discount) * 100) / 100;
}

// Unit tests ช่วยให้:
// 1. มั่นใจว่า logic ถูกต้องทุก case
// 2. Document behavior ของ function
// 3. ตรวจจับ regression เมื่อแก้ code
// 4. Refactor ได้อย่างมั่นใจ
```

---

## Unit vs Integration Tests

### Unit Tests

```javascript
// unit test - test แยก component เดียว
// ใช้ mocks แทน dependencies ทั้งหมด

const orderService = require('../services/orderService');

// Mock dependencies
jest.mock('../repositories/orderRepository');
jest.mock('../services/paymentService');
jest.mock('../services/emailService');
jest.mock('../services/inventoryService');

const orderRepository = require('../repositories/orderRepository');
const paymentService = require('../services/paymentService');
const emailService = require('../services/emailService');
const inventoryService = require('../services/inventoryService');

describe('OrderService.createOrder - Unit Tests', () => {
  const mockUser = { id: 'user123', email: 'user@test.com' };
  const mockItems = [
    { productId: 'prod1', quantity: 2, price: 100 },
    { productId: 'prod2', quantity: 1, price: 200 },
  ];
  
  beforeEach(() => {
    jest.clearAllMocks();
    
    // Setup default mocks
    inventoryService.checkAvailability.mockResolvedValue(true);
    paymentService.charge.mockResolvedValue({ id: 'charge123', status: 'success' });
    orderRepository.create.mockResolvedValue({
      id: 'order123',
      userId: mockUser.id,
      items: mockItems,
      total: 400,
      status: 'pending',
    });
    emailService.sendOrderConfirmation.mockResolvedValue(true);
  });
  
  it('creates order successfully', async () => {
    const order = await orderService.createOrder(mockUser, mockItems);
    
    expect(order).toBeDefined();
    expect(order.id).toBe('order123');
    expect(order.total).toBe(400);
  });
  
  it('checks inventory before creating order', async () => {
    await orderService.createOrder(mockUser, mockItems);
    
    expect(inventoryService.checkAvailability).toHaveBeenCalledWith(
      mockItems.map(item => item.productId)
    );
  });
  
  it('charges payment', async () => {
    await orderService.createOrder(mockUser, mockItems);
    
    expect(paymentService.charge).toHaveBeenCalledWith({
      userId: mockUser.id,
      amount: 400,
      currency: 'THB',
    });
  });
  
  it('sends confirmation email', async () => {
    await orderService.createOrder(mockUser, mockItems);
    
    expect(emailService.sendOrderConfirmation).toHaveBeenCalledWith(
      mockUser.email,
      expect.objectContaining({ id: 'order123' })
    );
  });
  
  it('throws error when inventory unavailable', async () => {
    inventoryService.checkAvailability.mockResolvedValue(false);
    
    await expect(orderService.createOrder(mockUser, mockItems))
      .rejects.toThrow('Items out of stock');
    
    expect(paymentService.charge).not.toHaveBeenCalled();
    expect(orderRepository.create).not.toHaveBeenCalled();
  });
  
  it('rolls back on payment failure', async () => {
    paymentService.charge.mockRejectedValue(new Error('Payment failed'));
    
    await expect(orderService.createOrder(mockUser, mockItems))
      .rejects.toThrow('Payment failed');
    
    // ตรวจสอบว่า order ไม่ถูกสร้าง
    expect(orderRepository.create).not.toHaveBeenCalled();
  });
});
```

### Integration Tests

```javascript
// integration test - ทดสอบหลาย components ร่วมกัน
// อาจใช้ in-memory database หรือ test database จริง

const mongoose = require('mongoose');
const { MongoMemoryServer } = require('mongodb-memory-server');
const orderService = require('../services/orderService');
const Order = require('../models/Order');
const Product = require('../models/Product');
const User = require('../models/User');

// Mock เฉพาะ external services
jest.mock('../services/paymentService');
jest.mock('../services/emailService');

describe('OrderService - Integration Tests', () => {
  let mongoServer;
  let testUser;
  let testProduct;
  
  beforeAll(async () => {
    mongoServer = await MongoMemoryServer.create();
    await mongoose.connect(mongoServer.getUri());
  });
  
  afterAll(async () => {
    await mongoose.disconnect();
    await mongoServer.stop();
  });
  
  beforeEach(async () => {
    await Order.deleteMany({});
    await Product.deleteMany({});
    await User.deleteMany({});
    
    testUser = await User.create({
      name: 'Test User',
      email: 'test@example.com',
      password: 'password123',
    });
    
    testProduct = await Product.create({
      name: 'Test Product',
      price: 100,
      stock: 10,
    });
  });
  
  it('creates and saves order to database', async () => {
    const paymentService = require('../services/paymentService');
    paymentService.charge.mockResolvedValue({ id: 'charge123' });
    
    const order = await orderService.createOrder(testUser, [
      { productId: testProduct._id, quantity: 2, price: testProduct.price },
    ]);
    
    // ตรวจสอบ order ใน database จริง
    const savedOrder = await Order.findById(order._id);
    expect(savedOrder).toBeDefined();
    expect(savedOrder.total).toBe(200);
    expect(savedOrder.userId.toString()).toBe(testUser._id.toString());
  });
  
  it('updates product stock after order', async () => {
    const paymentService = require('../services/paymentService');
    paymentService.charge.mockResolvedValue({ id: 'charge123' });
    
    await orderService.createOrder(testUser, [
      { productId: testProduct._id, quantity: 3, price: testProduct.price },
    ]);
    
    const updatedProduct = await Product.findById(testProduct._id);
    expect(updatedProduct.stock).toBe(7); // 10 - 3
  });
});
```

---

## Test Doubles

### 1. Dummy Objects

ส่งเป็น argument แต่ไม่ได้ใช้จริง

```javascript
it('logs user registration', () => {
  const dummyEmail = 'dummy@test.com'; // ไม่ได้ใช้จริงใน assertion
  const logger = { log: jest.fn() };
  
  registerUser({ email: dummyEmail, name: 'Test' }, logger);
  
  expect(logger.log).toHaveBeenCalled();
});
```

### 2. Stubs

แทนที่ implementation ด้วยค่าที่กำหนดไว้

```javascript
// Stub - กำหนด return value
const userRepositoryStub = {
  findById: jest.fn().mockResolvedValue({
    id: '123',
    name: 'John',
    email: 'john@test.com',
    role: 'user',
  }),
  save: jest.fn().mockResolvedValue(true),
};

// ทดสอบ service ด้วย stub
describe('UserService with stubs', () => {
  const userService = new UserService(userRepositoryStub);
  
  it('gets user profile', async () => {
    const profile = await userService.getProfile('123');
    
    expect(profile.name).toBe('John');
    expect(profile.email).toBe('john@test.com');
  });
  
  it('stub returns different values', async () => {
    userRepositoryStub.findById
      .mockResolvedValueOnce({ id: '1', name: 'Alice' })
      .mockResolvedValueOnce({ id: '2', name: 'Bob' });
    
    const alice = await userService.getProfile('1');
    const bob = await userService.getProfile('2');
    
    expect(alice.name).toBe('Alice');
    expect(bob.name).toBe('Bob');
  });
});
```

### 3. Spies

ติดตาม calls โดยไม่เปลี่ยน behavior

```javascript
// Spy - ดูว่า method ถูกเรียกหรือไม่
describe('Spies', () => {
  it('logs when user is created', async () => {
    const logger = {
      info: jest.fn(),
      error: jest.fn(),
    };
    const service = new UserService({ logger });
    
    await service.createUser({ name: 'Test', email: 'test@test.com' });
    
    // ดูว่า logger.info ถูกเรียกด้วย correct message
    expect(logger.info).toHaveBeenCalledWith(
      expect.stringContaining('User created')
    );
  });
  
  it('spies on real method', () => {
    const obj = {
      calculate: (a, b) => a + b,
    };
    
    const spy = jest.spyOn(obj, 'calculate');
    
    const result = obj.calculate(2, 3);
    
    expect(result).toBe(5);         // ยังทำงานปกติ
    expect(spy).toHaveBeenCalledWith(2, 3);
    
    spy.mockRestore();
  });
  
  it('spies and changes behavior', () => {
    const Math2 = {
      random: () => Math.random(),
    };
    
    const spy = jest.spyOn(Math2, 'random')
      .mockReturnValue(0.5); // override
    
    expect(Math2.random()).toBe(0.5);
    
    spy.mockRestore();
    // ตอนนี้ Math2.random() ทำงานปกติอีกครั้ง
  });
});
```

### 4. Mocks

กำหนด behavior ครบถ้วน พร้อม verify calls

```javascript
// Mock - full replacement with behavior verification
const emailTransportMock = {
  sendMail: jest.fn().mockImplementation(async (options) => {
    if (!options.to || !options.subject) {
      throw new Error('Invalid email options');
    }
    return { messageId: `mock-${Date.now()}` };
  }),
};

describe('EmailService with mock', () => {
  const emailService = new EmailService(emailTransportMock);
  
  it('sends email with correct options', async () => {
    await emailService.sendWelcomeEmail('user@test.com', 'John');
    
    expect(emailTransportMock.sendMail).toHaveBeenCalledWith(
      expect.objectContaining({
        to: 'user@test.com',
        subject: expect.stringContaining('Welcome'),
        html: expect.stringContaining('John'),
      })
    );
  });
  
  it('handles send failure', async () => {
    emailTransportMock.sendMail.mockRejectedValueOnce(
      new Error('SMTP connection failed')
    );
    
    await expect(emailService.sendWelcomeEmail('user@test.com', 'John'))
      .rejects.toThrow('SMTP connection failed');
  });
});
```

### 5. Fakes

Implementation จริงแต่ simplified

```javascript
// Fake - ทำงานจริงแต่เร็วกว่า
class FakeUserRepository {
  constructor() {
    this.users = new Map();
    this.nextId = 1;
  }
  
  async findById(id) {
    return this.users.get(id) || null;
  }
  
  async findByEmail(email) {
    return Array.from(this.users.values()).find(u => u.email === email) || null;
  }
  
  async create(data) {
    const user = { ...data, id: String(this.nextId++), createdAt: new Date() };
    this.users.set(user.id, user);
    return user;
  }
  
  async update(id, data) {
    const user = this.users.get(id);
    if (!user) return null;
    const updated = { ...user, ...data, updatedAt: new Date() };
    this.users.set(id, updated);
    return updated;
  }
  
  async delete(id) {
    return this.users.delete(id);
  }
  
  async count() {
    return this.users.size;
  }
  
  // Helper สำหรับ tests
  clear() {
    this.users.clear();
    this.nextId = 1;
  }
}

// ใช้งาน
describe('UserService with FakeRepository', () => {
  let fakeRepo;
  let userService;
  
  beforeEach(() => {
    fakeRepo = new FakeUserRepository();
    userService = new UserService(fakeRepo);
  });
  
  it('creates user', async () => {
    const user = await userService.createUser({
      name: 'Test',
      email: 'test@example.com',
    });
    
    expect(user.id).toBeDefined();
    expect(await fakeRepo.count()).toBe(1);
  });
  
  it('prevents duplicate emails', async () => {
    await userService.createUser({ email: 'test@example.com' });
    
    await expect(
      userService.createUser({ email: 'test@example.com' })
    ).rejects.toThrow('Email already exists');
  });
});
```

---

## Testing Utilities

### Test Factory

```javascript
// tests/factories/userFactory.js
const { faker } = require('@faker-js/faker');

const userFactory = {
  build: (overrides = {}) => ({
    id: faker.database.mongodbObjectId(),
    name: faker.person.fullName(),
    email: faker.internet.email().toLowerCase(),
    password: 'Password123!',
    role: 'user',
    isActive: true,
    createdAt: faker.date.past(),
    ...overrides,
  }),
  
  buildMany: (count, overrides = {}) =>
    Array.from({ length: count }, () => userFactory.build(overrides)),
  
  buildAdmin: (overrides = {}) =>
    userFactory.build({ role: 'admin', ...overrides }),
};

module.exports = { userFactory };
```

```javascript
// tests/factories/postFactory.js
const { faker } = require('@faker-js/faker');

const postFactory = {
  build: (overrides = {}) => ({
    id: faker.database.mongodbObjectId(),
    title: faker.lorem.sentence(),
    slug: faker.helpers.slugify(faker.lorem.sentence()).toLowerCase(),
    content: faker.lorem.paragraphs(3),
    excerpt: faker.lorem.sentence(),
    status: 'draft',
    author: faker.database.mongodbObjectId(),
    tags: [],
    viewCount: 0,
    createdAt: faker.date.past(),
    ...overrides,
  }),
  
  buildPublished: (overrides = {}) =>
    postFactory.build({
      status: 'published',
      publishedAt: faker.date.past(),
      ...overrides,
    }),
};

module.exports = { postFactory };
```

### Test Helpers

```javascript
// tests/helpers/dbHelper.js
const mongoose = require('mongoose');
const { MongoMemoryServer } = require('mongodb-memory-server');

let mongoServer;

const dbHelper = {
  async connect() {
    mongoServer = await MongoMemoryServer.create();
    await mongoose.connect(mongoServer.getUri());
  },
  
  async disconnect() {
    await mongoose.disconnect();
    await mongoServer?.stop();
  },
  
  async clearAll() {
    const collections = mongoose.connection.collections;
    await Promise.all(
      Object.values(collections).map(collection => collection.deleteMany({}))
    );
  },
  
  async clearCollection(name) {
    await mongoose.connection.collections[name]?.deleteMany({});
  },
};

module.exports = dbHelper;
```

```javascript
// tests/helpers/authHelper.js
const jwt = require('jsonwebtoken');
const config = require('../../src/config');

const authHelper = {
  createToken(userId, role = 'user') {
    return jwt.sign(
      { id: userId, role },
      config.auth.jwtSecret,
      { expiresIn: '1h' }
    );
  },
  
  createExpiredToken(userId) {
    return jwt.sign(
      { id: userId },
      config.auth.jwtSecret,
      { expiresIn: '0s' }
    );
  },
  
  createAdminToken(userId) {
    return this.createToken(userId, 'admin');
  },
  
  authHeader(token) {
    return { Authorization: `Bearer ${token}` };
  },
};

module.exports = authHelper;
```

---

## TDD Approach

TDD (Test-Driven Development): เขียน test ก่อน แล้วค่อยเขียน code

### Red-Green-Refactor Cycle

```
1. RED   - เขียน test ที่ fail ก่อน
2. GREEN - เขียน code ให้ test ผ่าน (simplest possible)
3. REFACTOR - ปรับปรุง code โดยไม่ทำ tests fail
```

### ตัวอย่าง TDD สร้าง DiscountCalculator

```javascript
// STEP 1: RED - เขียน tests ก่อน
// tests/discountCalculator.test.js

const DiscountCalculator = require('../src/utils/discountCalculator');

describe('DiscountCalculator', () => {
  let calculator;
  
  beforeEach(() => {
    calculator = new DiscountCalculator();
  });
  
  describe('calculateDiscount', () => {
    it('gives no discount for regular user with small order', () => {
      const result = calculator.calculate({
        userType: 'regular',
        price: 100,
        quantity: 1,
      });
      
      expect(result).toEqual({
        originalPrice: 100,
        discountPercent: 0,
        discountAmount: 0,
        finalPrice: 100,
      });
    });
    
    it('gives 10% discount for premium user', () => {
      const result = calculator.calculate({
        userType: 'premium',
        price: 100,
        quantity: 1,
      });
      
      expect(result.discountPercent).toBe(10);
      expect(result.finalPrice).toBe(90);
    });
    
    it('gives 5% discount for quantity >= 10', () => {
      const result = calculator.calculate({
        userType: 'regular',
        price: 100,
        quantity: 10,
      });
      
      expect(result.discountPercent).toBe(5);
      expect(result.finalPrice).toBe(95);
    });
    
    it('stacks discounts for premium user buying in bulk', () => {
      const result = calculator.calculate({
        userType: 'premium',
        price: 100,
        quantity: 10,
      });
      
      expect(result.discountPercent).toBe(15); // 10% + 5%
      expect(result.finalPrice).toBe(85);
    });
    
    it('caps discount at 30%', () => {
      const result = calculator.calculate({
        userType: 'premium',
        price: 1000,
        quantity: 50,
      });
      
      // 10% + 15% + 10% = 35% -> capped at 30%
      expect(result.discountPercent).toBe(30);
      expect(result.finalPrice).toBe(700);
    });
    
    it('throws error for invalid price', () => {
      expect(() => calculator.calculate({
        userType: 'regular',
        price: -100,
        quantity: 1,
      })).toThrow('Price must be positive');
    });
    
    it('throws error for invalid quantity', () => {
      expect(() => calculator.calculate({
        userType: 'regular',
        price: 100,
        quantity: 0,
      })).toThrow('Quantity must be at least 1');
    });
  });
});
```

```javascript
// STEP 2: GREEN - เขียน code ให้ tests ผ่าน
// src/utils/discountCalculator.js

class DiscountCalculator {
  calculate({ userType, price, quantity }) {
    // Validate
    if (price <= 0) throw new Error('Price must be positive');
    if (quantity < 1) throw new Error('Quantity must be at least 1');
    
    let discountPercent = 0;
    
    // Premium user discount
    if (userType === 'premium') {
      discountPercent += 10;
    }
    
    // Bulk discount
    if (quantity >= 10) {
      discountPercent += 5;
    }
    if (quantity >= 50) {
      discountPercent += 10;
    }
    
    // High value discount
    if (price > 1000) {
      discountPercent += 5;
    }
    
    // Cap at 30%
    discountPercent = Math.min(discountPercent, 30);
    
    const discountAmount = Math.round(price * discountPercent) / 100;
    const finalPrice = price - discountAmount;
    
    return {
      originalPrice: price,
      discountPercent,
      discountAmount,
      finalPrice,
    };
  }
}

module.exports = DiscountCalculator;
```

```javascript
// STEP 3: REFACTOR - ปรับปรุง code ให้ดีขึ้น
// (tests ยังผ่านอยู่)

class DiscountCalculator {
  static DISCOUNT_RULES = [
    {
      name: 'premium_user',
      condition: ({ userType }) => userType === 'premium',
      discount: 10,
    },
    {
      name: 'bulk_10',
      condition: ({ quantity }) => quantity >= 10,
      discount: 5,
    },
    {
      name: 'bulk_50',
      condition: ({ quantity }) => quantity >= 50,
      discount: 10,
    },
    {
      name: 'high_value',
      condition: ({ price }) => price > 1000,
      discount: 5,
    },
  ];
  
  static MAX_DISCOUNT = 30;
  
  validate({ price, quantity }) {
    if (price <= 0) throw new Error('Price must be positive');
    if (quantity < 1) throw new Error('Quantity must be at least 1');
  }
  
  calculateDiscountPercent(params) {
    const total = DiscountCalculator.DISCOUNT_RULES
      .filter(rule => rule.condition(params))
      .reduce((sum, rule) => sum + rule.discount, 0);
    
    return Math.min(total, DiscountCalculator.MAX_DISCOUNT);
  }
  
  calculate(params) {
    this.validate(params);
    
    const { price } = params;
    const discountPercent = this.calculateDiscountPercent(params);
    const discountAmount = Math.round(price * discountPercent) / 100;
    
    return {
      originalPrice: price,
      discountPercent,
      discountAmount,
      finalPrice: price - discountAmount,
    };
  }
}

module.exports = DiscountCalculator;
```

---

## Best Practices

### 1. FIRST Principles

```
F - Fast     : Tests ต้องรวดเร็ว
I - Isolated : แต่ละ test ต้องไม่ขึ้นกัน
R - Repeatable: ผลลัพธ์ต้องเหมือนกันทุกครั้ง
S - Self-validating: ผ่าน/ไม่ผ่านต้องชัดเจน
T - Timely   : เขียน test พร้อมกับ code
```

### 2. AAA Pattern

```javascript
it('should update user name', async () => {
  // ARRANGE - เตรียมข้อมูล
  const userId = '123';
  const newName = 'Updated Name';
  userRepo.findById.mockResolvedValue({ id: userId, name: 'Old Name' });
  userRepo.update.mockResolvedValue({ id: userId, name: newName });
  
  // ACT - ทำ action
  const result = await userService.updateName(userId, newName);
  
  // ASSERT - ตรวจสอบผลลัพธ์
  expect(result.name).toBe(newName);
  expect(userRepo.update).toHaveBeenCalledWith(userId, { name: newName });
});
```

### 3. Test ชื่อที่ดี

```javascript
// ❌ ไม่ดี
it('works', () => {});
it('test1', () => {});
it('should work', () => {});

// ✅ ดี - format: "should [expected behavior] when [condition]"
it('returns 404 when user not found', () => {});
it('throws error when email already exists', () => {});
it('sends welcome email after successful registration', () => {});
it('calculates 10% discount for premium users', () => {});
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: TDD สร้าง PasswordValidator

```javascript
// เขียน tests ก่อน แล้วค่อย implement

// PasswordValidator ต้องตรวจสอบ:
// - ความยาว >= 8 characters
// - มีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว
// - มีตัวเลขอย่างน้อย 1 ตัว
// - มี special character อย่างน้อย 1 ตัว
// - คืน {valid: true/false, errors: [...]}

// เขียน tests ก่อน:
describe('PasswordValidator', () => {
  it('validates strong password', () => { /* TODO */ });
  it('fails for short password', () => { /* TODO */ });
  it('fails for no uppercase', () => { /* TODO */ });
  // ...
});
```

### แบบฝึกหัดที่ 2: Test Doubles

สร้าง tests สำหรับ NotificationService ที่ใช้:
- Stub สำหรับ UserRepository
- Mock สำหรับ EmailService
- Fake สำหรับ SMS Provider (ที่เก็บข้อความไว้ใน memory)

### แบบฝึกหัดที่ 3: Refactoring with Tests

มี code ต่อไปนี้:

```javascript
// calculateShipping.js - code ที่ไม่มี tests
function calculateShipping(weight, distance, express) {
  let cost = 0;
  if (weight <= 1) cost = 30;
  else if (weight <= 5) cost = 50;
  else cost = 100;
  
  if (distance > 500) cost *= 1.5;
  if (express) cost *= 2;
  
  return cost;
}
```

1. เขียน tests ก่อน
2. Refactor code ให้ดีขึ้น
3. ตรวจสอบว่า tests ยังผ่านอยู่

---

## สรุป

Unit Testing ที่ดีช่วยให้ code มีคุณภาพสูง

| หัวข้อ | Key Points |
|--------|-----------|
| Test Pyramid | Unit > Integration > E2E |
| Test Doubles | Dummy, Stub, Spy, Mock, Fake |
| TDD | Red → Green → Refactor |
| FIRST | Fast, Isolated, Repeatable, Self-validating, Timely |
| AAA | Arrange, Act, Assert |

**ถัดไป**: [Part 37: Integration Testing →](./part-37-integration-testing.md)
