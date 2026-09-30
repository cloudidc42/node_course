# Part 33: RESTful API Design

> ขั้นตอนที่ 33-33 จาก 1000

---

## สารบัญ

1. [REST Constraints](#rest-constraints)
2. [Resource Naming](#resource-naming)
3. [Versioning Strategies](#versioning-strategies)
4. [HATEOAS](#hateoas)
5. [Pagination Patterns](#pagination-patterns)
6. [Filtering and Sorting](#filtering-and-sorting)
7. [Response Format Standards](#response-format-standards)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## REST Constraints

REST (Representational State Transfer) เป็น architectural style ที่มีข้อกำหนด 6 ข้อ:

### 1. Client-Server

```
Client (Browser/App)  ←→  Server (API)
      ↑                        ↑
   UI Logic             Business Logic
   State Management     Data Storage
```

### 2. Stateless

Server ไม่เก็บ state ของ client ทุก request ต้องมีข้อมูลครบในตัวเอง

```javascript
// ❌ Stateful - server จำ session
app.post('/login', (req, res) => {
  req.session.userId = user.id; // เก็บใน session บน server
});

// ✅ Stateless - token ใน request
app.get('/profile', (req, res) => {
  const token = req.headers.authorization; // client ส่งมาทุก request
  const user = verifyToken(token);
  res.json(user);
});
```

### 3. Cacheable

Response ต้องบอกว่า cache ได้หรือไม่

```javascript
// ตั้งค่า cache headers
app.get('/api/products', (req, res) => {
  res.set({
    'Cache-Control': 'public, max-age=3600',    // cache 1 hour
    'ETag': '"product-list-v1"',
    'Last-Modified': new Date().toUTCString(),
  });
  
  res.json(products);
});

// Conditional request handling
app.get('/api/products/:id', (req, res) => {
  const product = getProduct(req.params.id);
  const etag = `"${product.id}-${product.updatedAt.getTime()}"`;
  
  // ถ้า client มี version ล่าสุดแล้ว
  if (req.headers['if-none-match'] === etag) {
    return res.status(304).end(); // Not Modified
  }
  
  res.set('ETag', etag);
  res.json(product);
});
```

### 4. Uniform Interface

ใช้ HTTP methods อย่างถูกต้อง

```
GET    /users          → ดึงรายการ users ทั้งหมด
GET    /users/123      → ดึง user id 123
POST   /users          → สร้าง user ใหม่
PUT    /users/123      → อัพเดท user 123 ทั้งหมด (replace)
PATCH  /users/123      → อัพเดท user 123 บางส่วน (partial update)
DELETE /users/123      → ลบ user 123
```

### 5. Layered System

Client ไม่รู้ว่า request ผ่าน layers กี่ชั้น

```
Client → Load Balancer → CDN → API Gateway → Service → Database
```

### 6. Code on Demand (Optional)

Server สามารถส่ง executable code กลับมาได้ (เช่น JavaScript)

---

## Resource Naming

### หลักการตั้งชื่อ Resources

```
✅ ดี:
GET  /users              - รายการ users
GET  /users/123          - user คนเดียว
GET  /users/123/posts    - posts ของ user 123
POST /users/123/posts    - สร้าง post ของ user 123

❌ ไม่ดี:
GET  /getUsers           - ใช้ verb ใน URL
GET  /user/123           - singular แทน plural
GET  /Users              - uppercase
POST /createUser         - verb ใน URL
GET  /users/123/getPosts - verb ใน URL
```

### Naming Conventions

```javascript
// ✅ Plural nouns
/users, /products, /orders, /categories

// ✅ Lowercase with hyphens
/blog-posts, /user-profiles, /api-keys

// ✅ Hierarchical relationships
/users/:userId/orders          - orders ของ user คนนั้น
/orders/:orderId/items         - items ใน order นั้น
/categories/:id/products       - products ใน category นั้น

// ✅ Actions ที่ไม่ใช่ CRUD
POST /auth/login
POST /auth/logout
POST /auth/refresh
POST /users/:id/ban
POST /users/:id/unban
POST /posts/:id/publish
POST /payments/:id/capture
POST /emails/verify

// ✅ Search/Filter เป็น query parameters
GET /users?role=admin&status=active
GET /products?category=electronics&minPrice=100
GET /posts?search=nodejs&tag=tutorial
```

### Nested Resources (Max 2-3 levels)

```javascript
// ✅ 2 levels - OK
GET /users/:id/posts
GET /posts/:id/comments

// ✅ 3 levels - ยังพอรับได้
GET /users/:id/posts/:postId/comments

// ❌ 4+ levels - ยากเกินไป
GET /users/:id/posts/:postId/comments/:commentId/replies

// ✅ แก้โดย flatten ออกมา
GET /comments/:commentId/replies
```

---

## Versioning Strategies

### 1. URL Versioning (แนะนำ)

```javascript
// routes/v1/users.js
const router = express.Router();
router.get('/', usersV1Controller.getAll);

// routes/v2/users.js
const router = express.Router();
router.get('/', usersV2Controller.getAll); // v2 format ต่างออกไป

// app.js
app.use('/api/v1', require('./routes/v1'));
app.use('/api/v2', require('./routes/v2'));

// Endpoints:
// GET /api/v1/users
// GET /api/v2/users
```

```javascript
// ตัวอย่าง response format ที่ต่างกัน
// v1 response
{
  "users": [...],
  "total": 100
}

// v2 response (มี pagination รายละเอียดมากกว่า)
{
  "data": [...],
  "meta": {
    "pagination": {
      "total": 100,
      "page": 1,
      "perPage": 10,
      "totalPages": 10
    }
  }
}
```

### 2. Header Versioning

```javascript
// Middleware
const versionMiddleware = (req, res, next) => {
  const version = req.headers['api-version'] || 'v1';
  req.apiVersion = version;
  next();
};

app.use(versionMiddleware);

app.get('/api/users', (req, res) => {
  if (req.apiVersion === 'v2') {
    // v2 logic
  } else {
    // v1 logic
  }
});

// Client request:
// GET /api/users
// Headers: Api-Version: v2
```

### 3. Query Parameter Versioning

```javascript
// GET /api/users?version=2
app.get('/api/users', (req, res) => {
  const version = req.query.version || '1';
  
  switch (version) {
    case '2':
      return res.json(formatV2Response(users));
    default:
      return res.json(formatV1Response(users));
  }
});
```

### Deprecation Handling

```javascript
// Middleware สำหรับ deprecated endpoints
const deprecationWarning = (newEndpoint) => (req, res, next) => {
  res.set({
    'Deprecation': 'true',
    'Sunset': new Date('2025-12-31').toUTCString(),
    'Link': `<${newEndpoint}>; rel="successor-version"`,
  });
  
  console.warn(`Deprecated endpoint called: ${req.path}`);
  next();
};

// ใช้งาน
app.use('/api/v1/users', deprecationWarning('/api/v2/users'));
```

---

## HATEOAS

HATEOAS (Hypermedia as the Engine of Application State) คือการใส่ links ใน response เพื่อให้ client รู้ว่าทำอะไรได้บ้าง

```javascript
// ตัวอย่าง HATEOAS response
{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com",
  "status": "active",
  "_links": {
    "self": {
      "href": "/api/users/123",
      "method": "GET"
    },
    "update": {
      "href": "/api/users/123",
      "method": "PUT"
    },
    "delete": {
      "href": "/api/users/123",
      "method": "DELETE"
    },
    "posts": {
      "href": "/api/users/123/posts",
      "method": "GET"
    },
    "deactivate": {
      "href": "/api/users/123/deactivate",
      "method": "POST"
    }
  }
}
```

```javascript
// HATEOAS helper
class HateoasBuilder {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
  }
  
  buildUserLinks(user, requestUser) {
    const links = {
      self: { href: `${this.baseUrl}/users/${user.id}`, method: 'GET' },
      posts: { href: `${this.baseUrl}/users/${user.id}/posts`, method: 'GET' },
    };
    
    // เพิ่ม links ตาม permissions
    if (requestUser && (requestUser.id === user.id || requestUser.role === 'admin')) {
      links.update = { href: `${this.baseUrl}/users/${user.id}`, method: 'PUT' };
      links.delete = { href: `${this.baseUrl}/users/${user.id}`, method: 'DELETE' };
    }
    
    if (requestUser?.role === 'admin' && user.status === 'active') {
      links.ban = { href: `${this.baseUrl}/users/${user.id}/ban`, method: 'POST' };
    }
    
    return links;
  }
  
  buildListLinks(resource, page, limit, total) {
    const totalPages = Math.ceil(total / limit);
    const links = {
      self: { href: `${this.baseUrl}/${resource}?page=${page}&limit=${limit}` },
    };
    
    if (page > 1) {
      links.prev = { href: `${this.baseUrl}/${resource}?page=${page - 1}&limit=${limit}` };
      links.first = { href: `${this.baseUrl}/${resource}?page=1&limit=${limit}` };
    }
    
    if (page < totalPages) {
      links.next = { href: `${this.baseUrl}/${resource}?page=${page + 1}&limit=${limit}` };
      links.last = { href: `${this.baseUrl}/${resource}?page=${totalPages}&limit=${limit}` };
    }
    
    return links;
  }
}

// ใช้งาน
const hateoas = new HateoasBuilder('https://api.example.com');

app.get('/api/users/:id', (req, res) => {
  const user = getUser(req.params.id);
  
  res.json({
    ...user,
    _links: hateoas.buildUserLinks(user, req.user),
  });
});
```

---

## Pagination Patterns

### 1. Offset-based Pagination

```javascript
// GET /api/posts?page=2&limit=10
app.get('/api/posts', async (req, res) => {
  const page = Math.max(1, parseInt(req.query.page) || 1);
  const limit = Math.min(100, parseInt(req.query.limit) || 10);
  const offset = (page - 1) * limit;
  
  const [posts, total] = await Promise.all([
    Post.find().skip(offset).limit(limit),
    Post.countDocuments(),
  ]);
  
  const totalPages = Math.ceil(total / limit);
  
  res.json({
    data: posts,
    pagination: {
      page,
      limit,
      total,
      totalPages,
      hasNextPage: page < totalPages,
      hasPrevPage: page > 1,
    },
    links: {
      first: `/api/posts?page=1&limit=${limit}`,
      prev: page > 1 ? `/api/posts?page=${page - 1}&limit=${limit}` : null,
      next: page < totalPages ? `/api/posts?page=${page + 1}&limit=${limit}` : null,
      last: `/api/posts?page=${totalPages}&limit=${limit}`,
    },
  });
});
```

### 2. Cursor-based Pagination (แนะนำสำหรับ large datasets)

```javascript
// GET /api/posts?cursor=eyJpZCI6IjEyMyIsImNyZWF0ZWRBdCI6IjIwMjQifQ==&limit=10
app.get('/api/posts', async (req, res) => {
  const limit = Math.min(100, parseInt(req.query.limit) || 10);
  const cursor = req.query.cursor;
  
  let filter = {};
  let decodedCursor = null;
  
  if (cursor) {
    try {
      decodedCursor = JSON.parse(Buffer.from(cursor, 'base64').toString());
      filter = {
        $or: [
          { createdAt: { $lt: new Date(decodedCursor.createdAt) } },
          {
            createdAt: new Date(decodedCursor.createdAt),
            _id: { $lt: decodedCursor.id },
          },
        ],
      };
    } catch (e) {
      return res.status(400).json({ error: 'Invalid cursor' });
    }
  }
  
  const posts = await Post.find(filter)
    .sort({ createdAt: -1, _id: -1 })
    .limit(limit + 1); // fetch 1 extra to check hasNextPage
  
  const hasNextPage = posts.length > limit;
  if (hasNextPage) posts.pop();
  
  let nextCursor = null;
  if (hasNextPage && posts.length > 0) {
    const lastPost = posts[posts.length - 1];
    nextCursor = Buffer.from(JSON.stringify({
      id: lastPost._id,
      createdAt: lastPost.createdAt,
    })).toString('base64');
  }
  
  res.json({
    data: posts,
    pagination: {
      limit,
      hasNextPage,
      cursor: nextCursor,
    },
  });
});
```

### 3. Keyset Pagination

```javascript
// GET /api/posts?after_id=507f1f77bcf86cd799439011&limit=10
app.get('/api/posts', async (req, res) => {
  const limit = parseInt(req.query.limit) || 10;
  const afterId = req.query.after_id;
  
  let query = { status: 'published' };
  
  if (afterId) {
    const lastPost = await Post.findById(afterId).select('createdAt');
    if (lastPost) {
      query.createdAt = { $lt: lastPost.createdAt };
    }
  }
  
  const posts = await Post.find(query)
    .sort({ createdAt: -1 })
    .limit(limit);
  
  res.json({
    data: posts,
    pagination: {
      limit,
      hasNextPage: posts.length === limit,
      lastId: posts.length > 0 ? posts[posts.length - 1]._id : null,
    },
  });
});
```

---

## Filtering and Sorting

### Query Builder Pattern

```javascript
// utils/queryBuilder.js
class QueryBuilder {
  constructor(model, queryString) {
    this.query = model.find();
    this.queryString = queryString;
    this.model = model;
  }
  
  filter() {
    const queryObj = { ...this.queryString };
    
    // ลบ fields ที่ไม่ใช่ filter
    const excludeFields = ['page', 'limit', 'sort', 'fields', 'search'];
    excludeFields.forEach(field => delete queryObj[field]);
    
    // แปลง operators: gte, gt, lte, lt
    // ?price[gte]=100&price[lte]=500
    let queryStr = JSON.stringify(queryObj);
    queryStr = queryStr.replace(/\b(gte|gt|lte|lt|ne|in|nin)\b/g, match => `$${match}`);
    
    this.query = this.query.find(JSON.parse(queryStr));
    return this;
  }
  
  search(fields) {
    if (this.queryString.search) {
      const searchRegex = new RegExp(this.queryString.search, 'i');
      const searchConditions = fields.map(field => ({ [field]: searchRegex }));
      this.query = this.query.find({ $or: searchConditions });
    }
    return this;
  }
  
  sort() {
    if (this.queryString.sort) {
      // ?sort=price,-createdAt  (- หมายถึง descending)
      const sortBy = this.queryString.sort.split(',').join(' ');
      this.query = this.query.sort(sortBy);
    } else {
      this.query = this.query.sort('-createdAt');
    }
    return this;
  }
  
  selectFields() {
    if (this.queryString.fields) {
      // ?fields=name,email,createdAt
      const fields = this.queryString.fields.split(',').join(' ');
      this.query = this.query.select(fields);
    } else {
      this.query = this.query.select('-__v');
    }
    return this;
  }
  
  paginate() {
    const page = parseInt(this.queryString.page) || 1;
    const limit = Math.min(parseInt(this.queryString.limit) || 10, 100);
    const skip = (page - 1) * limit;
    
    this.query = this.query.skip(skip).limit(limit);
    this._page = page;
    this._limit = limit;
    return this;
  }
  
  async execute() {
    const data = await this.query;
    const total = await this.model.countDocuments(this.query.getFilter());
    
    return {
      data,
      total,
      page: this._page,
      limit: this._limit,
      totalPages: Math.ceil(total / this._limit),
    };
  }
}

module.exports = QueryBuilder;
```

```javascript
// ใช้งาน QueryBuilder
const QueryBuilder = require('../utils/queryBuilder');
const Product = require('../models/Product');

app.get('/api/products', async (req, res) => {
  // ?category=electronics&price[gte]=100&price[lte]=1000
  // &sort=-price,name
  // &fields=name,price,category
  // &page=1&limit=20
  // &search=laptop
  
  const result = await new QueryBuilder(Product, req.query)
    .filter()
    .search(['name', 'description'])
    .sort()
    .selectFields()
    .paginate()
    .execute();
  
  res.json({
    status: 'success',
    ...result,
  });
});
```

### Advanced Filtering Examples

```javascript
// ตัวอย่าง query parameters ที่รองรับ

// Range filter
// GET /api/products?price[gte]=100&price[lte]=500
// GET /api/posts?createdAt[gte]=2024-01-01

// Array filter  
// GET /api/products?category[in]=electronics,clothing
// GET /api/users?role[in]=admin,moderator

// Null filter
// GET /api/users?deletedAt[exists]=false

// Text search
// GET /api/products?search=laptop gaming

// Sorting
// GET /api/products?sort=-price,name    (price desc, name asc)
// GET /api/posts?sort=-publishedAt      (newest first)

// Field selection
// GET /api/users?fields=name,email,createdAt

// Combining all
// GET /api/products?category=electronics
//   &price[gte]=500
//   &sort=-rating,price
//   &fields=name,price,rating,image
//   &page=1&limit=12
```

---

## Response Format Standards

### Standard Response Structure

```javascript
// Success responses
// 200 OK - GET, PUT, PATCH
{
  "status": "success",
  "data": { /* resource */ },
  "meta": { /* optional metadata */ }
}

// 201 Created - POST
{
  "status": "success",
  "message": "Resource created successfully",
  "data": { /* created resource */ }
}

// 204 No Content - DELETE
// No body

// List response
{
  "status": "success",
  "data": [ /* array of resources */ ],
  "meta": {
    "pagination": {
      "total": 100,
      "page": 1,
      "perPage": 10,
      "totalPages": 10,
      "hasNextPage": true,
      "hasPrevPage": false
    }
  },
  "links": {
    "first": "/api/users?page=1",
    "prev": null,
    "next": "/api/users?page=2",
    "last": "/api/users?page=10"
  }
}
```

### Error Response Format

```javascript
// Error responses
// 400 Bad Request
{
  "status": "error",
  "code": "VALIDATION_ERROR",
  "message": "Validation failed",
  "errors": [
    {
      "field": "email",
      "message": "Must be a valid email address",
      "value": "not-an-email"
    },
    {
      "field": "password",
      "message": "Must be at least 8 characters"
    }
  ]
}

// 401 Unauthorized
{
  "status": "error",
  "code": "UNAUTHORIZED",
  "message": "Authentication required"
}

// 403 Forbidden
{
  "status": "error",
  "code": "FORBIDDEN",
  "message": "You don't have permission to access this resource"
}

// 404 Not Found
{
  "status": "error",
  "code": "NOT_FOUND",
  "message": "User with id '123' not found"
}

// 409 Conflict
{
  "status": "error",
  "code": "DUPLICATE_RESOURCE",
  "message": "Email already registered"
}

// 429 Too Many Requests
{
  "status": "error",
  "code": "RATE_LIMIT_EXCEEDED",
  "message": "Too many requests",
  "retryAfter": 60
}

// 500 Internal Server Error
{
  "status": "error",
  "code": "INTERNAL_ERROR",
  "message": "An unexpected error occurred",
  "requestId": "req_abc123"  // สำหรับ debugging
}
```

### Response Helpers

```javascript
// utils/response.js
class ApiResponse {
  static success(data, message = null, meta = null) {
    const response = { status: 'success' };
    if (message) response.message = message;
    if (data !== null && data !== undefined) response.data = data;
    if (meta) response.meta = meta;
    return response;
  }
  
  static created(data, message = 'Created successfully') {
    return {
      status: 'success',
      message,
      data,
    };
  }
  
  static list(data, pagination) {
    return {
      status: 'success',
      data,
      meta: { pagination },
      links: ApiResponse.buildPaginationLinks(pagination),
    };
  }
  
  static error(message, code, statusCode = 400, errors = null) {
    const response = {
      status: 'error',
      code,
      message,
    };
    if (errors) response.errors = errors;
    return { response, statusCode };
  }
  
  static buildPaginationLinks(pagination, baseUrl = '') {
    const { page, perPage, totalPages } = pagination;
    return {
      first: `${baseUrl}?page=1&limit=${perPage}`,
      prev: page > 1 ? `${baseUrl}?page=${page - 1}&limit=${perPage}` : null,
      next: page < totalPages ? `${baseUrl}?page=${page + 1}&limit=${perPage}` : null,
      last: `${baseUrl}?page=${totalPages}&limit=${perPage}`,
    };
  }
}

module.exports = ApiResponse;
```

```javascript
// Error handler middleware
const errorHandler = (err, req, res, next) => {
  err.statusCode = err.statusCode || 500;
  err.code = err.code || 'INTERNAL_ERROR';
  
  // Log error (ไม่แสดงใน production)
  if (process.env.NODE_ENV === 'development') {
    console.error(err.stack);
  }
  
  // Mongoose validation error
  if (err.name === 'ValidationError') {
    const errors = Object.values(err.errors).map(e => ({
      field: e.path,
      message: e.message,
    }));
    
    return res.status(400).json({
      status: 'error',
      code: 'VALIDATION_ERROR',
      message: 'Validation failed',
      errors,
    });
  }
  
  // JWT errors
  if (err.name === 'JsonWebTokenError') {
    return res.status(401).json({
      status: 'error',
      code: 'INVALID_TOKEN',
      message: 'Invalid authentication token',
    });
  }
  
  res.status(err.statusCode).json({
    status: 'error',
    code: err.code,
    message: err.message,
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
  });
};

module.exports = errorHandler;
```

### HTTP Status Codes Reference

```javascript
// 2xx Success
200 OK             - GET, PUT, PATCH (ส่งข้อมูลคืน)
201 Created        - POST (สร้างสำเร็จ)
204 No Content     - DELETE (ลบสำเร็จ ไม่มี body)
206 Partial Content - GET range request

// 3xx Redirection
301 Moved Permanently - endpoint เปลี่ยน URL ถาวร
302 Found           - redirect ชั่วคราว
304 Not Modified    - cache ยังใช้ได้

// 4xx Client Errors
400 Bad Request     - request ผิด syntax หรือ validation
401 Unauthorized    - ต้อง authenticate ก่อน
403 Forbidden       - authenticate แล้ว แต่ไม่มี permission
404 Not Found       - resource ไม่พบ
405 Method Not Allowed - HTTP method ไม่รองรับ
409 Conflict        - ข้อมูลซ้ำ
410 Gone            - resource ถูกลบถาวรแล้ว
422 Unprocessable Entity - validation error
429 Too Many Requests - rate limit

// 5xx Server Errors
500 Internal Server Error - server error ทั่วไป
502 Bad Gateway     - upstream error
503 Service Unavailable - server ไม่พร้อม
504 Gateway Timeout - upstream timeout
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: RESTful Routes

ออกแบบ URL structure สำหรับ E-commerce API:
- Products, Categories, Orders, Reviews, Users
- ต้องมี nested resources ที่เหมาะสม
- ต้องมี action endpoints ที่ถูกต้อง

```javascript
// TODO: ออกแบบ endpoints ต่อไปนี้
// - รายการสินค้า พร้อม filter และ sort
// - รายละเอียดสินค้า
// - สินค้าใน category
// - สร้าง order
// - ดู order status
// - เพิ่ม review ให้สินค้า
// - ยกเลิก order
```

### แบบฝึกหัดที่ 2: Pagination

สร้าง API endpoint ที่รองรับทั้ง offset และ cursor pagination:

```javascript
// GET /api/posts?type=offset&page=2&limit=10
// GET /api/posts?type=cursor&cursor=xxx&limit=10
// สลับระหว่าง 2 modes ตาม query parameter
```

### แบบฝึกหัดที่ 3: Advanced Filtering

สร้าง Product search API:

```bash
# ตัวอย่าง queries ที่ต้องรองรับ
GET /api/products?category=electronics&brand[in]=Apple,Samsung
GET /api/products?price[gte]=100&price[lte]=500&rating[gte]=4
GET /api/products?search=wireless+headphones&sort=-rating,price
GET /api/products?fields=name,price,image,rating&page=1&limit=24
GET /api/products?inStock=true&discount[gt]=0
```

### แบบฝึกหัดที่ 4: HATEOAS

เพิ่ม HATEOAS links ให้กับ User API:

```json
// Expected response
{
  "id": "123",
  "name": "John",
  "status": "active",
  "_links": {
    "self": { "href": "/api/users/123", "method": "GET" },
    "update": { "href": "/api/users/123", "method": "PUT" },
    "orders": { "href": "/api/users/123/orders", "method": "GET" },
    "deactivate": { "href": "/api/users/123/deactivate", "method": "POST" }
  }
}
```

---

## สรุป

RESTful API Design ที่ดีช่วยให้ API ใช้งานง่ายและ predictable:

| หัวข้อ | Best Practice |
|--------|--------------|
| Naming | Plural nouns, lowercase, hyphens |
| Methods | ใช้ HTTP methods ให้ถูกต้อง |
| Versioning | URL versioning ง่ายที่สุด |
| Pagination | Cursor-based สำหรับ large datasets |
| Filtering | Query parameters, operators |
| Response | Consistent structure เสมอ |
| Errors | Error codes ที่ชัดเจน |

**ถัดไป**: [Part 34: API Documentation →](./part-34-api-documentation.md)
