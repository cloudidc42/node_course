# Part 19: REST API Basics
## ขั้นตอนที่ 19-9 จาก 1000

---

## สารบัญ
1. REST Principles
2. API Design Best Practices
3. Status Codes Usage
4. Request/Response Format
5. API Versioning
6. HATEOAS
7. Practical: Full CRUD REST API
8. Exercise

---

## 1. REST Principles

REST (Representational State Transfer) คือ architectural style สำหรับออกแบบ networked applications

### 6 หลักการของ REST

#### 1. Client-Server Separation
```
Client (React, Mobile, etc.)    Server (Express API)
         |                              |
         |--- HTTP Request ---------->  |
         |<-- HTTP Response ----------- |
         
// Client และ Server แยกกันโดยสิ้นเชิง
// Server ไม่รู้ว่า Client คืออะไร (Web? Mobile? CLI?)
// Client ไม่รู้ว่า Server เก็บข้อมูลยังไง (MySQL? MongoDB? Redis?)
```

#### 2. Stateless
```javascript
// ❌ Not Stateless: Server เก็บ state ของ client
app.get('/next-page', (req, res) => {
  // Server จำ "current page" ของ client แต่ละคน - ไม่ดี!
  const page = serverState[req.clientId].currentPage + 1;
  res.json(getPage(page));
});

// ✅ Stateless: ทุกข้อมูลอยู่ใน request
app.get('/pages/:pageNum', (req, res) => {
  // Client ส่ง page number มาเอง
  const page = parseInt(req.params.pageNum);
  res.json(getPage(page));
});
```

#### 3. Cacheable
```javascript
app.get('/products', (req, res) => {
  // บอก client ว่า cache ได้นาน 5 นาที
  res.set('Cache-Control', 'public, max-age=300');
  res.json(products);
});
```

#### 4. Uniform Interface
```
// ใช้ HTTP methods อย่างสม่ำเสมอ:
GET    /users          - ดูรายการ users
GET    /users/123      - ดู user เดียว
POST   /users          - สร้าง user ใหม่
PUT    /users/123      - อัปเดต user ทั้งหมด
PATCH  /users/123      - อัปเดต user บางส่วน
DELETE /users/123      - ลบ user
```

#### 5. Layered System
```
Client → Load Balancer → Cache Layer → API Server → Database
// Client ไม่รู้ว่าคุยกับ server จริง หรือ cache หรือ load balancer
```

#### 6. Code on Demand (Optional)
```javascript
// Server ส่ง executable code มาให้ client
// เช่น ส่ง JavaScript มา run ใน browser
// (ไม่ค่อยใช้ใน REST APIs)
```

### Resources คืออะไร?

ใน REST ทุกอย่างคือ "Resource":

```
// Resources ตัวอย่าง:
/users           - collection of users
/users/123       - specific user
/users/123/posts - posts ของ user 123
/products        - collection of products
/products/456    - specific product
/orders          - collection of orders
/orders/789/items - items ใน order 789
```

---

## 2. API Design Best Practices

### Naming Conventions

```
// ✅ ดี
GET    /users              - noun, plural, lowercase
GET    /users/123          - resource identifier
GET    /users/123/posts    - nested resource
GET    /product-categories - kebab-case สำหรับ multi-word

// ❌ ไม่ดี
GET    /getUsers           - ใช้ verb ใน URL (method บอก action แล้ว)
GET    /Users              - uppercase
GET    /user               - singular สำหรับ collection
POST   /createUser         - verb ใน URL
GET    /users_list         - underscore
```

### URL Structure

```
Base URL:  https://api.example.com
Version:   /v1
Resource:  /users
ID:        /123
Sub-resource: /posts
Sub-ID:    /456

Full URL: https://api.example.com/v1/users/123/posts/456

// ตัวอย่าง E-commerce API:
GET    /api/v1/products
GET    /api/v1/products?category=electronics&maxPrice=50000
GET    /api/v1/products/123
POST   /api/v1/products
PUT    /api/v1/products/123
PATCH  /api/v1/products/123
DELETE /api/v1/products/123

GET    /api/v1/users/456/orders
GET    /api/v1/users/456/orders/789
POST   /api/v1/users/456/orders
```

### Response Format

```javascript
// ✅ Consistent response format
// Success
{
  "success": true,
  "data": { /* ข้อมูล */ },
  "meta": { /* pagination, etc. */ },
  "message": "สำเร็จ"
}

// Error
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "ข้อมูลไม่ถูกต้อง",
    "details": [
      { "field": "email", "message": "รูปแบบอีเมลไม่ถูกต้อง" }
    ]
  }
}

// Collection
{
  "success": true,
  "data": [...],
  "meta": {
    "total": 100,
    "page": 1,
    "limit": 10,
    "totalPages": 10,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```

---

## 3. Status Codes Usage

### ตาราง HTTP Status Codes

```
// 2xx - Success
200 OK           - GET, PUT, PATCH สำเร็จ
201 Created      - POST สร้าง resource ใหม่
204 No Content   - DELETE สำเร็จ (ไม่มี body)

// 3xx - Redirection
301 Moved Permanently  - URL เปลี่ยนถาวร
302 Found              - Redirect ชั่วคราว
304 Not Modified       - Cache ยังใช้ได้

// 4xx - Client Errors
400 Bad Request        - ข้อมูลไม่ถูกต้อง (validation error)
401 Unauthorized       - ยังไม่ได้ authenticate
403 Forbidden          - Authenticate แล้ว แต่ไม่มีสิทธิ์
404 Not Found          - ไม่พบ resource
405 Method Not Allowed - HTTP method ไม่รองรับ
409 Conflict           - ข้อมูลซ้ำ (เช่น email ซ้ำ)
410 Gone               - Resource ถูกลบถาวร
422 Unprocessable Entity - ข้อมูลถูกรูปแบบ แต่ validation ไม่ผ่าน
429 Too Many Requests  - Rate limit เกิน

// 5xx - Server Errors
500 Internal Server Error - เกิดข้อผิดพลาดใน server
502 Bad Gateway         - ปัญหา upstream
503 Service Unavailable - Server ไม่พร้อมใช้งาน
```

### ตัวอย่างการใช้งาน Status Codes

```javascript
const express = require('express');
const app = express();
app.use(express.json());

const users = [];

// 200 OK - ดึงข้อมูลสำเร็จ
app.get('/api/users', (req, res) => {
  res.status(200).json({
    success: true,
    data: users
  });
});

// 201 Created - สร้างสำเร็จ
app.post('/api/users', (req, res) => {
  const { name, email } = req.body;
  
  // 400 Bad Request - validation error
  if (!name || !email) {
    return res.status(400).json({
      success: false,
      error: { message: 'กรุณาระบุ name และ email' }
    });
  }
  
  // 409 Conflict - email ซ้ำ
  if (users.find(u => u.email === email)) {
    return res.status(409).json({
      success: false,
      error: { message: 'Email นี้ถูกใช้แล้ว' }
    });
  }
  
  const newUser = { id: Date.now(), name, email };
  users.push(newUser);
  
  res.status(201)
    .location(`/api/users/${newUser.id}`)  // บอก URL ของ resource ใหม่
    .json({ success: true, data: newUser });
});

// 404 Not Found
app.get('/api/users/:id', (req, res) => {
  const user = users.find(u => u.id === parseInt(req.params.id));
  
  if (!user) {
    return res.status(404).json({
      success: false,
      error: { message: `ไม่พบ user ID: ${req.params.id}` }
    });
  }
  
  res.json({ success: true, data: user });
});

// 204 No Content - ลบสำเร็จ
app.delete('/api/users/:id', (req, res) => {
  const idx = users.findIndex(u => u.id === parseInt(req.params.id));
  
  if (idx === -1) {
    return res.status(404).json({
      success: false,
      error: { message: 'ไม่พบ user' }
    });
  }
  
  users.splice(idx, 1);
  res.status(204).end(); // ไม่มี body
});

// 401 Unauthorized
app.get('/api/secret', (req, res) => {
  const token = req.headers.authorization;
  if (!token) {
    return res.status(401).json({
      success: false,
      error: {
        message: 'กรุณา authenticate ก่อน',
        hint: 'ส่ง Authorization: Bearer <token> ใน header'
      }
    });
  }
  res.json({ secret: 'data' });
});

// 403 Forbidden
app.delete('/api/admin/users/:id', (req, res) => {
  if (req.user?.role !== 'admin') {
    return res.status(403).json({
      success: false,
      error: { message: 'ต้องการสิทธิ์ admin' }
    });
  }
  // ลบ user...
  res.status(204).end();
});
```

---

## 4. Request/Response Format

### Request Headers สำคัญ

```javascript
app.use((req, res, next) => {
  // Content-Type - บอก server ว่า body เป็นอะไร
  // Content-Type: application/json
  // Content-Type: multipart/form-data
  
  // Accept - บอก server ว่า client ต้องการ response แบบไหน
  // Accept: application/json
  // Accept: application/json, text/html
  
  // Authorization - authentication token
  // Authorization: Bearer eyJhbG...
  // Authorization: Basic dXNlcjpwYXNz
  
  // API Key
  // X-API-Key: your-api-key
  
  // Request ID (สำหรับ tracing)
  // X-Request-ID: uuid-here
  
  next();
});
```

### Response Headers สำคัญ

```javascript
app.get('/api/data', (req, res) => {
  res
    // Content-Type
    .set('Content-Type', 'application/json; charset=utf-8')
    
    // Cache
    .set('Cache-Control', 'public, max-age=300')
    .set('ETag', '"abc123"')
    
    // CORS
    .set('Access-Control-Allow-Origin', '*')
    
    // Rate Limiting Info
    .set('X-RateLimit-Limit', '100')
    .set('X-RateLimit-Remaining', '95')
    .set('X-RateLimit-Reset', Math.floor(Date.now() / 1000) + 3600)
    
    // Request tracing
    .set('X-Request-ID', req.headers['x-request-id'] || 'auto-generated')
    
    // API Version
    .set('X-API-Version', '1.0.0')
    
    .json({ data: 'response' });
});
```

### Pagination Response

```javascript
// Standard pagination format
const createPaginatedResponse = (data, total, page, limit) => {
  const totalPages = Math.ceil(total / limit);
  
  return {
    success: true,
    data,
    meta: {
      pagination: {
        total,
        page,
        limit,
        totalPages,
        hasNextPage: page < totalPages,
        hasPrevPage: page > 1,
        nextPage: page < totalPages ? page + 1 : null,
        prevPage: page > 1 ? page - 1 : null
      }
    }
  };
};

app.get('/api/posts', (req, res) => {
  const page = Math.max(1, parseInt(req.query.page) || 1);
  const limit = Math.min(100, Math.max(1, parseInt(req.query.limit) || 10));
  
  const total = posts.length;
  const data = posts.slice((page - 1) * limit, page * limit);
  
  res.json(createPaginatedResponse(data, total, page, limit));
});
```

---

## 5. API Versioning

### Version ใน URL Path (แนะนำ)

```javascript
const express = require('express');
const app = express();
app.use(express.json());

// V1 Routes
const v1Router = express.Router();
v1Router.get('/users', (req, res) => {
  res.json({
    users: [{ id: 1, name: 'สมชาย' }] // V1 format (เรียบง่าย)
  });
});

// V2 Routes (ปรับปรุงแล้ว)
const v2Router = express.Router();
v2Router.get('/users', (req, res) => {
  res.json({
    success: true,
    data: [{
      id: 1,
      name: 'สมชาย',
      profile: { avatar: '/img/avatar.jpg' },
      stats: { posts: 10, followers: 50 }
    }],
    meta: { total: 1, version: '2.0' }
  });
});

app.use('/api/v1', v1Router);
app.use('/api/v2', v2Router);

// Current version alias
app.use('/api', v2Router); // หรือ redirect
```

### Version ใน Header

```javascript
app.use('/api/users', (req, res, next) => {
  const version = req.headers['api-version'] || '1';
  
  if (version === '2') {
    return getUsersV2(req, res);
  }
  
  return getUsersV1(req, res);
});

function getUsersV1(req, res) {
  res.json({ users: [] });
}

function getUsersV2(req, res) {
  res.json({ success: true, data: [], meta: {} });
}
```

---

## 6. HATEOAS

HATEOAS (Hypermedia As The Engine Of Application State) คือ REST constraint ที่ response มี links บอกว่าทำอะไรได้ต่อ:

```javascript
// ตัวอย่าง HATEOAS response
{
  "success": true,
  "data": {
    "id": 123,
    "name": "สมชาย ใจดี",
    "email": "somchai@example.com",
    "status": "active"
  },
  "_links": {
    "self": { "href": "/api/users/123", "method": "GET" },
    "update": { "href": "/api/users/123", "method": "PUT" },
    "delete": { "href": "/api/users/123", "method": "DELETE" },
    "posts": { "href": "/api/users/123/posts", "method": "GET" },
    "orders": { "href": "/api/users/123/orders", "method": "GET" }
  }
}
```

```javascript
// Helper function
const addLinks = (resource, resourceType, id, extra = {}) => {
  const baseUrl = `/api/${resourceType}/${id}`;
  return {
    ...resource,
    _links: {
      self: { href: baseUrl, method: 'GET' },
      update: { href: baseUrl, method: 'PUT' },
      patch: { href: baseUrl, method: 'PATCH' },
      delete: { href: baseUrl, method: 'DELETE' },
      ...extra
    }
  };
};

app.get('/api/users/:id', (req, res) => {
  const user = { id: 1, name: 'สมชาย' };
  
  res.json({
    success: true,
    data: addLinks(user, 'users', user.id, {
      posts: { href: `/api/users/${user.id}/posts`, method: 'GET' }
    })
  });
});
```

---

## 7. Practical: Full CRUD REST API

### สร้าง Blog API สมบูรณ์

```javascript
// app.js - Full Blog REST API
const express = require('express');
const app = express();
app.use(express.json());

// ==============================
// In-memory Data Store
// ==============================
let posts = [
  {
    id: 1,
    title: 'เริ่มต้นกับ Node.js',
    content: 'Node.js เป็น runtime สำหรับ JavaScript บน server...',
    author: 'สมชาย ใจดี',
    authorId: 1,
    category: 'Node.js',
    tags: ['nodejs', 'javascript'],
    status: 'published',
    views: 100,
    createdAt: new Date('2024-01-01'),
    updatedAt: new Date('2024-01-01')
  },
  {
    id: 2,
    title: 'Express.js ขั้นสูง',
    content: 'ในบทความนี้เราจะเรียนรู้ Express.js...',
    author: 'สมหญิง ใจงาม',
    authorId: 2,
    category: 'Express.js',
    tags: ['express', 'nodejs', 'api'],
    status: 'published',
    views: 75,
    createdAt: new Date('2024-01-05'),
    updatedAt: new Date('2024-01-06')
  }
];

let nextId = 3;

// ==============================
// Helper Functions
// ==============================
const createResponse = (data, meta = null) => ({
  success: true,
  data,
  ...(meta && { meta })
});

const createError = (message, code = null, details = null) => ({
  success: false,
  error: {
    message,
    ...(code && { code }),
    ...(details && { details })
  }
});

const paginate = (array, page, limit) => {
  const total = array.length;
  const totalPages = Math.ceil(total / limit);
  const offset = (page - 1) * limit;
  const data = array.slice(offset, offset + limit);
  
  return {
    data,
    meta: {
      pagination: {
        total,
        page,
        limit,
        totalPages,
        hasNextPage: page < totalPages,
        hasPrevPage: page > 1
      }
    }
  };
};

// ==============================
// Validation
// ==============================
const validatePost = (data, partial = false) => {
  const errors = [];
  
  if (!partial || data.title !== undefined) {
    if (!data.title || data.title.trim().length < 3) {
      errors.push({ field: 'title', message: 'title ต้องมีอย่างน้อย 3 ตัวอักษร' });
    }
  }
  
  if (!partial || data.content !== undefined) {
    if (!data.content || data.content.trim().length < 10) {
      errors.push({ field: 'content', message: 'content ต้องมีอย่างน้อย 10 ตัวอักษร' });
    }
  }
  
  if (!partial || data.status !== undefined) {
    if (data.status && !['draft', 'published', 'archived'].includes(data.status)) {
      errors.push({ field: 'status', message: 'status ต้องเป็น draft, published, หรือ archived' });
    }
  }
  
  return errors;
};

// ==============================
// API Routes
// ==============================

// ===== GET /api/posts - ดูทุก posts =====
app.get('/api/posts', (req, res) => {
  const {
    search, category, tag, status,
    sort = 'createdAt', order = 'desc',
    page = 1, limit = 10
  } = req.query;
  
  let result = [...posts];
  
  // Filter
  if (search) {
    const q = search.toLowerCase();
    result = result.filter(p =>
      p.title.toLowerCase().includes(q) ||
      p.content.toLowerCase().includes(q)
    );
  }
  
  if (category) result = result.filter(p => p.category === category);
  if (tag) result = result.filter(p => p.tags.includes(tag));
  if (status) result = result.filter(p => p.status === status);
  
  // Sort
  const sortOrder = order === 'desc' ? -1 : 1;
  result.sort((a, b) => {
    if (a[sort] < b[sort]) return -1 * sortOrder;
    if (a[sort] > b[sort]) return 1 * sortOrder;
    return 0;
  });
  
  // Paginate
  const pageNum = parseInt(page);
  const limitNum = parseInt(limit);
  const { data, meta } = paginate(result, pageNum, limitNum);
  
  res.json(createResponse(data, meta));
});

// ===== GET /api/posts/:id - ดู post เดียว =====
app.get('/api/posts/:id', (req, res) => {
  const id = parseInt(req.params.id);
  if (isNaN(id)) {
    return res.status(400).json(createError('ID ต้องเป็นตัวเลข'));
  }
  
  const post = posts.find(p => p.id === id);
  if (!post) {
    return res.status(404).json(createError(`ไม่พบ post ID: ${id}`));
  }
  
  post.views++;
  
  res.json(createResponse(post));
});

// ===== POST /api/posts - สร้าง post ใหม่ =====
app.post('/api/posts', (req, res) => {
  const errors = validatePost(req.body);
  if (errors.length > 0) {
    return res.status(422).json(
      createError('ข้อมูลไม่ถูกต้อง', 'VALIDATION_ERROR', errors)
    );
  }
  
  const now = new Date();
  const newPost = {
    id: nextId++,
    title: req.body.title.trim(),
    content: req.body.content.trim(),
    author: req.body.author || 'Anonymous',
    authorId: req.body.authorId || null,
    category: req.body.category || 'Uncategorized',
    tags: Array.isArray(req.body.tags) ? req.body.tags : [],
    status: req.body.status || 'draft',
    views: 0,
    createdAt: now,
    updatedAt: now
  };
  
  posts.push(newPost);
  
  res.status(201)
    .location(`/api/posts/${newPost.id}`)
    .json(createResponse(newPost));
});

// ===== PUT /api/posts/:id - อัปเดตทั้งหมด =====
app.put('/api/posts/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const postIndex = posts.findIndex(p => p.id === id);
  
  if (postIndex === -1) {
    return res.status(404).json(createError(`ไม่พบ post ID: ${id}`));
  }
  
  const errors = validatePost(req.body);
  if (errors.length > 0) {
    return res.status(422).json(
      createError('ข้อมูลไม่ถูกต้อง', 'VALIDATION_ERROR', errors)
    );
  }
  
  const oldPost = posts[postIndex];
  posts[postIndex] = {
    id: oldPost.id,
    title: req.body.title.trim(),
    content: req.body.content.trim(),
    author: req.body.author || oldPost.author,
    authorId: req.body.authorId || oldPost.authorId,
    category: req.body.category || 'Uncategorized',
    tags: Array.isArray(req.body.tags) ? req.body.tags : [],
    status: req.body.status || 'draft',
    views: oldPost.views,
    createdAt: oldPost.createdAt,
    updatedAt: new Date()
  };
  
  res.json(createResponse(posts[postIndex]));
});

// ===== PATCH /api/posts/:id - อัปเดตบางส่วน =====
app.patch('/api/posts/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const postIndex = posts.findIndex(p => p.id === id);
  
  if (postIndex === -1) {
    return res.status(404).json(createError(`ไม่พบ post ID: ${id}`));
  }
  
  const errors = validatePost(req.body, true); // partial = true
  if (errors.length > 0) {
    return res.status(422).json(
      createError('ข้อมูลไม่ถูกต้อง', 'VALIDATION_ERROR', errors)
    );
  }
  
  // อัปเดตเฉพาะ fields ที่ส่งมา
  const allowedFields = ['title', 'content', 'author', 'category', 'tags', 'status'];
  allowedFields.forEach(field => {
    if (req.body[field] !== undefined) {
      posts[postIndex][field] = req.body[field];
    }
  });
  posts[postIndex].updatedAt = new Date();
  
  res.json(createResponse(posts[postIndex]));
});

// ===== DELETE /api/posts/:id - ลบ post =====
app.delete('/api/posts/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const postIndex = posts.findIndex(p => p.id === id);
  
  if (postIndex === -1) {
    return res.status(404).json(createError(`ไม่พบ post ID: ${id}`));
  }
  
  posts.splice(postIndex, 1);
  res.status(204).end();
});

// ===== Special Endpoints =====

// GET /api/posts/stats - สถิติ
app.get('/api/posts/stats', (req, res) => {
  const stats = {
    total: posts.length,
    byStatus: {
      draft: posts.filter(p => p.status === 'draft').length,
      published: posts.filter(p => p.status === 'published').length,
      archived: posts.filter(p => p.status === 'archived').length
    },
    byCategory: posts.reduce((acc, p) => {
      acc[p.category] = (acc[p.category] || 0) + 1;
      return acc;
    }, {}),
    totalViews: posts.reduce((sum, p) => sum + p.views, 0),
    mostViewed: [...posts].sort((a, b) => b.views - a.views).slice(0, 5)
  };
  
  res.json(createResponse(stats));
});

// POST /api/posts/:id/publish - publish post
app.post('/api/posts/:id/publish', (req, res) => {
  const post = posts.find(p => p.id === parseInt(req.params.id));
  if (!post) return res.status(404).json(createError('ไม่พบ post'));
  
  post.status = 'published';
  post.publishedAt = new Date();
  post.updatedAt = new Date();
  
  res.json(createResponse(post));
});

// ==============================
// Error Handlers
// ==============================
app.use((req, res) => {
  res.status(404).json(createError(
    `ไม่พบ endpoint: ${req.method} ${req.url}`,
    'NOT_FOUND'
  ));
});

app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json(createError(
    'เกิดข้อผิดพลาดภายใน Server',
    'INTERNAL_ERROR'
  ));
});

app.listen(3000, () => {
  console.log('Blog REST API: http://localhost:3000');
  console.log('\nEndpoints:');
  console.log('GET    /api/posts');
  console.log('GET    /api/posts/stats');
  console.log('GET    /api/posts/:id');
  console.log('POST   /api/posts');
  console.log('PUT    /api/posts/:id');
  console.log('PATCH  /api/posts/:id');
  console.log('DELETE /api/posts/:id');
  console.log('POST   /api/posts/:id/publish');
});
```

### ทดสอบ API

```bash
# ดูทุก posts
curl http://localhost:3000/api/posts

# ค้นหา
curl "http://localhost:3000/api/posts?search=nodejs&page=1&limit=5"

# ดู post เดียว
curl http://localhost:3000/api/posts/1

# สร้างใหม่
curl -X POST http://localhost:3000/api/posts \
  -H "Content-Type: application/json" \
  -d '{
    "title": "บทความใหม่",
    "content": "เนื้อหาบทความที่น่าสนใจมาก",
    "author": "นักเขียน",
    "category": "Node.js",
    "tags": ["nodejs", "tutorial"],
    "status": "published"
  }'

# อัปเดตทั้งหมด (PUT)
curl -X PUT http://localhost:3000/api/posts/1 \
  -H "Content-Type: application/json" \
  -d '{"title": "ชื่อใหม่", "content": "เนื้อหาใหม่"}'

# อัปเดตบางส่วน (PATCH)
curl -X PATCH http://localhost:3000/api/posts/1 \
  -H "Content-Type: application/json" \
  -d '{"status": "archived"}'

# ลบ
curl -X DELETE http://localhost:3000/api/posts/1

# สถิติ
curl http://localhost:3000/api/posts/stats
```

---

## 8. Exercise

### Exercise 1: E-commerce Product API

สร้าง REST API สำหรับ E-commerce ที่มี:
- Products CRUD
- Categories CRUD
- ความสัมพันธ์ระหว่าง Product ↔ Category
- Filter, search, pagination
- Stock management (in/out)

### Exercise 2: Task Management API

```
GET    /api/tasks               - ดูทุก tasks
GET    /api/tasks/:id           - ดู task เดียว
POST   /api/tasks               - สร้าง task
PATCH  /api/tasks/:id           - อัปเดต task
DELETE /api/tasks/:id           - ลบ task
POST   /api/tasks/:id/complete  - mark เป็น complete
POST   /api/tasks/:id/assign    - assign ให้ user
GET    /api/tasks/stats         - สถิติ tasks
```

### Exercise 3: API Documentation

สร้าง documentation endpoint ที่แสดง API spec:
```
GET /api/docs - แสดง HTML documentation
GET /api/docs.json - แสดง OpenAPI/Swagger spec
```

---

## สรุป Part 19

ในบทนี้เราได้เรียนรู้:

1. **REST Principles** - 6 หลักการสำคัญ
2. **API Design** - naming conventions, URL structure
3. **Status Codes** - การใช้งานที่ถูกต้อง
4. **Response Format** - consistent format, pagination
5. **API Versioning** - URL path, header
6. **HATEOAS** - hypermedia links
7. **Practical** - Full CRUD Blog API

---

*Part 19 | Node.js/Express.js Course | ขั้นตอนที่ 19 จาก 1000*
