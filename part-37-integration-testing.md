# Part 37: Integration Testing

> ขั้นตอนที่ 37-37 จาก 1000

---

## สารบัญ

1. [Integration Testing คืออะไร](#integration-testing-คืออะไร)
2. [Supertest สำหรับ Express](#supertest-สำหรับ-express)
3. [Database Testing](#database-testing)
4. [Test Fixtures](#test-fixtures)
5. [In-memory Databases](#in-memory-databases)
6. [CI Testing](#ci-testing)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Integration Testing คืออะไร

Integration Testing ทดสอบว่าหลาย components ทำงานร่วมกันได้ถูกต้อง

```
Unit Test:        [Component A]
Integration Test: [Component A] → [Component B] → [Database]
E2E Test:         [Browser] → [Frontend] → [API] → [Database]
```

### ทำไม Integration Tests สำคัญ

```javascript
// Unit tests ผ่านทั้งหมด แต่ integration test fail!
// เพราะ unit tests mock dependencies

// authService unit test (mock):
// authService.login() → OK (mock database)

// authService integration test (real):
// authService.login() → User.findOne() → MongoDB → ❌ FAIL
// เหตุ: query ไม่ correct หรือ connection string ผิด
```

---

## Supertest สำหรับ Express

Supertest เป็น library สำหรับ test HTTP endpoints ของ Express

### ติดตั้ง

```bash
npm install --save-dev supertest
```

### Basic Setup

```javascript
// tests/integration/auth.test.js
const request = require('supertest');
const app = require('../../src/app');
const mongoose = require('mongoose');
const { MongoMemoryServer } = require('mongodb-memory-server');
const User = require('../../src/models/User');

let mongoServer;

beforeAll(async () => {
  mongoServer = await MongoMemoryServer.create();
  await mongoose.connect(mongoServer.getUri());
});

afterAll(async () => {
  await mongoose.disconnect();
  await mongoServer.stop();
});

beforeEach(async () => {
  await User.deleteMany({});
});

describe('POST /api/auth/register', () => {
  it('registers new user successfully', async () => {
    const response = await request(app)
      .post('/api/auth/register')
      .send({
        name: 'Test User',
        email: 'test@example.com',
        password: 'Password123!',
      });
    
    expect(response.status).toBe(201);
    expect(response.body.status).toBe('success');
    expect(response.body.data.user).toBeDefined();
    expect(response.body.data.accessToken).toBeDefined();
    
    // ตรวจสอบว่าไม่มี sensitive data
    expect(response.body.data.user.password).toBeUndefined();
  });
  
  it('returns 400 for missing required fields', async () => {
    const response = await request(app)
      .post('/api/auth/register')
      .send({ email: 'test@example.com' }); // ขาด name และ password
    
    expect(response.status).toBe(400);
    expect(response.body.status).toBe('error');
    expect(response.body.code).toBe('VALIDATION_ERROR');
    expect(response.body.errors).toBeInstanceOf(Array);
  });
  
  it('returns 409 for duplicate email', async () => {
    // สร้าง user แรกก่อน
    await User.create({
      name: 'Existing User',
      email: 'existing@example.com',
      password: 'Password123!',
    });
    
    // พยายามสร้างด้วย email เดิม
    const response = await request(app)
      .post('/api/auth/register')
      .send({
        name: 'New User',
        email: 'existing@example.com',
        password: 'Password123!',
      });
    
    expect(response.status).toBe(409);
    expect(response.body.code).toBe('DUPLICATE_RESOURCE');
  });
  
  it('returns 400 for weak password', async () => {
    const response = await request(app)
      .post('/api/auth/register')
      .send({
        name: 'Test User',
        email: 'test@example.com',
        password: '123',  // weak password
      });
    
    expect(response.status).toBe(400);
  });
});

describe('POST /api/auth/login', () => {
  let testUser;
  
  beforeEach(async () => {
    testUser = await User.create({
      name: 'Test User',
      email: 'test@example.com',
      password: 'Password123!',
    });
  });
  
  it('logs in with valid credentials', async () => {
    const response = await request(app)
      .post('/api/auth/login')
      .send({
        email: 'test@example.com',
        password: 'Password123!',
      });
    
    expect(response.status).toBe(200);
    expect(response.body.data.accessToken).toBeDefined();
    expect(response.body.data.user.email).toBe('test@example.com');
  });
  
  it('returns 401 for wrong password', async () => {
    const response = await request(app)
      .post('/api/auth/login')
      .send({
        email: 'test@example.com',
        password: 'WrongPassword!',
      });
    
    expect(response.status).toBe(401);
  });
  
  it('returns 401 for non-existent email', async () => {
    const response = await request(app)
      .post('/api/auth/login')
      .send({
        email: 'notexist@example.com',
        password: 'Password123!',
      });
    
    expect(response.status).toBe(401);
  });
});
```

### Testing Authenticated Endpoints

```javascript
// tests/integration/posts.test.js
const request = require('supertest');
const app = require('../../src/app');
const jwt = require('jsonwebtoken');
const config = require('../../src/config');
const User = require('../../src/models/User');
const Post = require('../../src/models/Post');

describe('Posts API', () => {
  let authToken;
  let testUser;
  let adminToken;
  let adminUser;
  
  beforeEach(async () => {
    await User.deleteMany({});
    await Post.deleteMany({});
    
    testUser = await User.create({
      name: 'Test User',
      email: 'user@test.com',
      password: 'Password123!',
      role: 'user',
    });
    
    adminUser = await User.create({
      name: 'Admin',
      email: 'admin@test.com',
      password: 'Password123!',
      role: 'admin',
    });
    
    authToken = jwt.sign(
      { id: testUser._id, role: 'user' },
      config.auth.jwtSecret,
      { expiresIn: '1h' }
    );
    
    adminToken = jwt.sign(
      { id: adminUser._id, role: 'admin' },
      config.auth.jwtSecret,
      { expiresIn: '1h' }
    );
  });
  
  describe('GET /api/posts', () => {
    it('returns published posts', async () => {
      // สร้าง test data
      await Post.create([
        {
          title: 'Published Post 1',
          content: 'Content 1',
          author: testUser._id,
          status: 'published',
          publishedAt: new Date(),
        },
        {
          title: 'Draft Post',
          content: 'Content',
          author: testUser._id,
          status: 'draft',
        },
        {
          title: 'Published Post 2',
          content: 'Content 2',
          author: testUser._id,
          status: 'published',
          publishedAt: new Date(),
        },
      ]);
      
      const response = await request(app).get('/api/posts');
      
      expect(response.status).toBe(200);
      expect(response.body.data).toHaveLength(2); // ไม่รวม draft
      response.body.data.forEach(post => {
        expect(post.status).toBe('published');
      });
    });
    
    it('paginates results', async () => {
      // สร้าง 15 posts
      const posts = Array.from({ length: 15 }, (_, i) => ({
        title: `Post ${i + 1}`,
        content: 'Content',
        author: testUser._id,
        status: 'published',
        publishedAt: new Date(),
      }));
      await Post.insertMany(posts);
      
      const response = await request(app)
        .get('/api/posts')
        .query({ page: 1, limit: 10 });
      
      expect(response.status).toBe(200);
      expect(response.body.data).toHaveLength(10);
      expect(response.body.meta.pagination.total).toBe(15);
      expect(response.body.meta.pagination.totalPages).toBe(2);
      expect(response.body.meta.pagination.hasNextPage).toBe(true);
    });
    
    it('filters by tag', async () => {
      // TODO: สร้าง posts พร้อม tags แล้วทดสอบ filter
    });
  });
  
  describe('POST /api/posts', () => {
    it('requires authentication', async () => {
      const response = await request(app)
        .post('/api/posts')
        .send({ title: 'Test', content: 'Content' });
      
      expect(response.status).toBe(401);
    });
    
    it('creates post for authenticated user', async () => {
      const response = await request(app)
        .post('/api/posts')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          title: 'New Post Title',
          content: 'This is the content of the new post that is long enough',
          tags: ['nodejs', 'testing'],
        });
      
      expect(response.status).toBe(201);
      expect(response.body.data.title).toBe('New Post Title');
      expect(response.body.data.author.id).toBe(testUser._id.toString());
      
      // ตรวจสอบใน database
      const savedPost = await Post.findById(response.body.data.id);
      expect(savedPost).toBeDefined();
      expect(savedPost.title).toBe('New Post Title');
    });
    
    it('validates required fields', async () => {
      const response = await request(app)
        .post('/api/posts')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ title: 'Only Title' }); // ขาด content
      
      expect(response.status).toBe(400);
      expect(response.body.errors.some(e => e.field === 'content')).toBe(true);
    });
  });
  
  describe('DELETE /api/posts/:id', () => {
    let testPost;
    
    beforeEach(async () => {
      testPost = await Post.create({
        title: 'Test Post',
        content: 'Content',
        author: testUser._id,
        status: 'published',
      });
    });
    
    it('allows author to delete own post', async () => {
      const response = await request(app)
        .delete(`/api/posts/${testPost._id}`)
        .set('Authorization', `Bearer ${authToken}`);
      
      expect(response.status).toBe(204);
      
      const deletedPost = await Post.findById(testPost._id);
      expect(deletedPost).toBeNull();
    });
    
    it('prevents non-author from deleting post', async () => {
      const otherUser = await User.create({
        name: 'Other',
        email: 'other@test.com',
        password: 'Password123!',
      });
      
      const otherToken = jwt.sign(
        { id: otherUser._id, role: 'user' },
        config.auth.jwtSecret,
        { expiresIn: '1h' }
      );
      
      const response = await request(app)
        .delete(`/api/posts/${testPost._id}`)
        .set('Authorization', `Bearer ${otherToken}`);
      
      expect(response.status).toBe(403);
    });
    
    it('allows admin to delete any post', async () => {
      const response = await request(app)
        .delete(`/api/posts/${testPost._id}`)
        .set('Authorization', `Bearer ${adminToken}`);
      
      expect(response.status).toBe(204);
    });
  });
});
```

---

## Database Testing

### MongoDB Testing ด้วย MongoMemoryServer

```javascript
// tests/setup/mongodb.js
const mongoose = require('mongoose');
const { MongoMemoryServer } = require('mongodb-memory-server');

let mongoServer;

module.exports = {
  async connect() {
    mongoServer = await MongoMemoryServer.create({
      instance: {
        dbName: 'jest-test',
      },
    });
    
    const uri = mongoServer.getUri();
    await mongoose.connect(uri, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    
    console.log(`Connected to MongoDB at ${uri}`);
  },
  
  async clearDatabase() {
    const collections = mongoose.connection.collections;
    await Promise.all(
      Object.values(collections).map(col => col.deleteMany({}))
    );
  },
  
  async closeDatabase() {
    await mongoose.connection.dropDatabase();
    await mongoose.connection.close();
    await mongoServer.stop();
  },
};
```

```javascript
// jest.config.js - global setup
module.exports = {
  globalSetup: './tests/setup/globalSetup.js',
  globalTeardown: './tests/setup/globalTeardown.js',
  setupFilesAfterFramework: ['./tests/setup/jest.setup.js'],
};

// tests/setup/globalSetup.js
module.exports = async () => {
  // รันครั้งเดียวก่อน tests ทั้งหมด
};

// tests/setup/jest.setup.js
const mongodb = require('./mongodb');

beforeAll(async () => {
  await mongodb.connect();
});

afterEach(async () => {
  await mongodb.clearDatabase();
});

afterAll(async () => {
  await mongodb.closeDatabase();
});
```

### PostgreSQL Testing

```javascript
// ใช้ pg-mem สำหรับ in-memory PostgreSQL
const { newDb } = require('pg-mem');

describe('UserRepository with PostgreSQL', () => {
  let db;
  let userRepository;
  
  beforeAll(async () => {
    // สร้าง in-memory PostgreSQL
    db = newDb();
    
    // สร้าง tables
    db.public.none(`
      CREATE TABLE users (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        email VARCHAR(255) UNIQUE NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    `);
    
    const adapter = db.adapters.createPg();
    userRepository = new UserRepository(adapter);
  });
  
  beforeEach(async () => {
    await db.public.none('TRUNCATE users RESTART IDENTITY CASCADE');
  });
  
  it('creates user in database', async () => {
    const user = await userRepository.create({
      name: 'Test User',
      email: 'test@example.com',
    });
    
    expect(user.id).toBeDefined();
    expect(user.name).toBe('Test User');
  });
  
  it('finds user by email', async () => {
    await userRepository.create({ name: 'John', email: 'john@test.com' });
    
    const found = await userRepository.findByEmail('john@test.com');
    expect(found).toBeDefined();
    expect(found.name).toBe('John');
  });
});
```

---

## Test Fixtures

Fixtures คือข้อมูล test ที่เตรียมไว้ล่วงหน้า

### Fixture Files

```javascript
// tests/fixtures/users.js
const bcrypt = require('bcryptjs');

const hashedPassword = bcrypt.hashSync('Password123!', 10);

module.exports = {
  regularUser: {
    name: 'Regular User',
    email: 'user@test.com',
    password: hashedPassword,
    role: 'user',
  },
  
  adminUser: {
    name: 'Admin User',
    email: 'admin@test.com',
    password: hashedPassword,
    role: 'admin',
  },
  
  inactiveUser: {
    name: 'Inactive User',
    email: 'inactive@test.com',
    password: hashedPassword,
    role: 'user',
    isActive: false,
  },
};
```

```javascript
// tests/fixtures/posts.js
module.exports = {
  publishedPost: (authorId) => ({
    title: 'Published Test Post',
    slug: 'published-test-post',
    content: 'This is the content of the published post.',
    excerpt: 'A test post excerpt.',
    author: authorId,
    status: 'published',
    publishedAt: new Date('2024-01-01'),
  }),
  
  draftPost: (authorId) => ({
    title: 'Draft Test Post',
    slug: 'draft-test-post',
    content: 'This is draft content.',
    author: authorId,
    status: 'draft',
  }),
  
  multiplePosts: (authorId, count = 5) =>
    Array.from({ length: count }, (_, i) => ({
      title: `Post ${i + 1}`,
      slug: `post-${i + 1}`,
      content: `Content for post ${i + 1}`,
      author: authorId,
      status: 'published',
      publishedAt: new Date(`2024-01-${i + 1}`),
    })),
};
```

### Database Seeder

```javascript
// tests/helpers/seeder.js
const User = require('../../src/models/User');
const Post = require('../../src/models/Post');
const userFixtures = require('../fixtures/users');

const seeder = {
  async seedUsers() {
    const [user, admin] = await User.insertMany([
      userFixtures.regularUser,
      userFixtures.adminUser,
    ]);
    return { user, admin };
  },
  
  async seedPosts(authorId, count = 5) {
    const posts = Array.from({ length: count }, (_, i) => ({
      title: `Test Post ${i + 1}`,
      slug: `test-post-${i + 1}`,
      content: `Content for test post ${i + 1}. This is a longer content.`,
      author: authorId,
      status: i % 2 === 0 ? 'published' : 'draft',
      publishedAt: i % 2 === 0 ? new Date() : null,
    }));
    
    return Post.insertMany(posts);
  },
  
  async seed() {
    const { user, admin } = await this.seedUsers();
    const posts = await this.seedPosts(user._id, 10);
    
    return { user, admin, posts };
  },
  
  async clean() {
    await Promise.all([
      User.deleteMany({}),
      Post.deleteMany({}),
    ]);
  },
};

module.exports = seeder;
```

---

## In-memory Databases

### mongodb-memory-server

```bash
npm install --save-dev mongodb-memory-server
```

```javascript
// jest.config.js
module.exports = {
  // ดาวน์โหลด MongoDB binary อัตโนมัติ
  globalSetup: '@shelf/jest-mongodb/setup',
  globalTeardown: '@shelf/jest-mongodb/teardown',
  testEnvironment: '@shelf/jest-mongodb/environment',
};
```

### Redis Mock

```javascript
// ใช้ ioredis-mock สำหรับ test Redis
const RedisMock = require('ioredis-mock');

// แทนที่ redis client ใน test
jest.mock('ioredis', () => require('ioredis-mock'));

// หรือ inject dependency
const redis = new RedisMock();
const cacheService = new CacheService(redis);

describe('CacheService', () => {
  beforeEach(async () => {
    await redis.flushall(); // ล้าง cache ก่อนแต่ละ test
  });
  
  it('sets and gets value', async () => {
    await cacheService.set('key', 'value', 60);
    const result = await cacheService.get('key');
    expect(result).toBe('value');
  });
  
  it('expires after TTL', async () => {
    jest.useFakeTimers();
    
    await cacheService.set('key', 'value', 1); // 1 second TTL
    
    jest.advanceTimersByTime(2000); // เดิน timer 2 seconds
    
    const result = await cacheService.get('key');
    expect(result).toBeNull();
    
    jest.useRealTimers();
  });
});
```

---

## CI Testing

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [18.x, 20.x]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run unit tests
        run: npm run test:unit
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          NODE_ENV: test
          JWT_SECRET: test-secret-minimum-32-chars-long
          DATABASE_NAME: test_db
      
      - name: Run coverage
        run: npm run test:coverage
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          file: ./coverage/lcov.info
```

### Test Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:unit": "jest --testPathPattern='unit'",
    "test:integration": "jest --testPathPattern='integration'",
    "test:e2e": "jest --testPathPattern='e2e'",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage --coverageThreshold='{\"global\":{\"lines\":80}}'",
    "test:ci": "jest --ci --coverage --forceExit --detectOpenHandles"
  }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: API Integration Tests

สร้าง integration tests สำหรับ Todo API:

```javascript
// ต้องทดสอบ:
// GET  /api/todos          - ดึงรายการ
// POST /api/todos          - สร้าง
// PUT  /api/todos/:id      - อัพเดท
// DELETE /api/todos/:id    - ลบ
// GET  /api/todos?completed=true - filter

// Setup:
// - MongoMemoryServer
// - User authentication
// - Clear data ระหว่าง tests
```

### แบบฝึกหัดที่ 2: Error Handling Tests

```javascript
// ทดสอบ error scenarios:
// 1. Database connection error
// 2. Validation errors
// 3. Duplicate key error
// 4. JWT expired
// 5. Rate limit exceeded
// 6. File too large
```

### แบบฝึกหัดที่ 3: CI Setup

สร้าง GitHub Actions workflow ที่:
1. รัน tests บน Node 18 และ 20
2. Report coverage ไปยัง Codecov
3. Fail build ถ้า coverage < 80%
4. Cache npm dependencies

---

## สรุป

Integration Testing ทดสอบ components ที่ทำงานร่วมกัน

| เครื่องมือ | ใช้สำหรับ |
|-----------|---------|
| Supertest | HTTP endpoint testing |
| MongoMemoryServer | In-memory MongoDB |
| pg-mem | In-memory PostgreSQL |
| ioredis-mock | In-memory Redis |
| Fixtures | Test data ที่เตรียมไว้ |
| Seeders | เติมข้อมูลก่อน test |

**ถัดไป**: [Part 38: WebSocket & Socket.io →](./part-38-websocket-socketio.md)
