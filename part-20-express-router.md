# Part 20: Express Router ขั้นสูง
## ขั้นตอนที่ 20-10 จาก 1000

---

## สารบัญ
1. Express Router
2. Router Mounting
3. Nested Routes
4. Parameter Inheritance
5. Router-level Middleware
6. Practical: Modular API Structure
7. Exercise

---

## 1. Express Router

Express Router คือ mini-application ที่สามารถ define routes, middleware ได้เหมือน app แต่ mount ที่ path ที่กำหนดได้

### Router พื้นฐาน

```javascript
const express = require('express');

// สร้าง Router instance
const router = express.Router();

// กำหนด routes บน router
router.get('/', (req, res) => {
  res.json({ message: 'Router home' });
});

router.get('/:id', (req, res) => {
  res.json({ id: req.params.id });
});

router.post('/', (req, res) => {
  res.status(201).json({ created: req.body });
});

// Export สำหรับ mount
module.exports = router;
```

```javascript
// app.js
const express = require('express');
const app = express();
app.use(express.json());

const userRouter = require('./routes/users');
const productRouter = require('./routes/products');

// Mount routers ที่ path prefix
app.use('/api/users', userRouter);
app.use('/api/products', productRouter);

// ผลลัพธ์:
// GET  /api/users           → userRouter GET /
// GET  /api/users/123       → userRouter GET /:id
// POST /api/users           → userRouter POST /
// GET  /api/products        → productRouter GET /
// ...

app.listen(3000);
```

### Router Options

```javascript
// Router options
const router = express.Router({
  caseSensitive: false, // /Users และ /users ต่างกัน? (default: false)
  strict: false,        // /users/ และ /users ต่างกัน? (default: false)
  mergeParams: true     // รับ params จาก parent router (default: false)
});
```

---

## 2. Router Mounting

Router mounting คือการ "ติด" router เข้ากับ path ของ application:

### Single Mount Point

```javascript
// routes/auth.js
const express = require('express');
const router = express.Router();

router.post('/login', (req, res) => {
  res.json({ action: 'login' });
});

router.post('/register', (req, res) => {
  res.json({ action: 'register' });
});

router.post('/logout', (req, res) => {
  res.json({ action: 'logout' });
});

module.exports = router;
```

```javascript
// app.js
const authRoutes = require('./routes/auth');

// Mount ที่ /auth
app.use('/auth', authRoutes);

// Accessible:
// POST /auth/login
// POST /auth/register
// POST /auth/logout
```

### Multiple Mount Points

```javascript
// Router เดียว mount หลาย paths
app.use('/api/v1/users', userRoutes);
app.use('/api/v2/users', userRoutes); // ใช้ router เดิม แต่คนละ path
```

### Conditional Mounting

```javascript
// Mount ตาม environment
if (process.env.NODE_ENV === 'development') {
  app.use('/debug', require('./routes/debug'));
}

// Mount ตาม feature flag
if (process.env.ENABLE_PAYMENTS === 'true') {
  app.use('/api/payments', require('./routes/payments'));
}
```

---

## 3. Nested Routes

Nested routes คือการ mount router ภายใน router:

### การสร้าง Nested Routes

```javascript
// routes/users/index.js
const express = require('express');
const router = express.Router();

// Import sub-routers
const userPostsRouter = require('./posts');
const userOrdersRouter = require('./orders');
const userProfileRouter = require('./profile');

// User base routes
router.get('/', getAllUsers);
router.get('/:userId', getUser);
router.post('/', createUser);
router.put('/:userId', updateUser);
router.delete('/:userId', deleteUser);

// Mount nested routers
// mergeParams: true ทำให้ sub-router เข้าถึง :userId ของ parent ได้
router.use('/:userId/posts', userPostsRouter);
router.use('/:userId/orders', userOrdersRouter);
router.use('/:userId/profile', userProfileRouter);

module.exports = router;
```

```javascript
// routes/users/posts.js
const express = require('express');
// ต้องใช้ mergeParams: true เพื่อรับ :userId จาก parent
const router = express.Router({ mergeParams: true });

router.get('/', (req, res) => {
  const { userId } = req.params; // ได้ userId จาก parent router
  res.json({
    message: `Posts ของ User ${userId}`,
    userId
  });
});

router.get('/:postId', (req, res) => {
  const { userId, postId } = req.params;
  res.json({
    message: `Post ${postId} ของ User ${userId}`,
    userId,
    postId
  });
});

router.post('/', (req, res) => {
  const { userId } = req.params;
  res.status(201).json({
    message: `สร้าง post สำหรับ User ${userId}`,
    userId,
    post: req.body
  });
});

module.exports = router;
```

```javascript
// app.js
app.use('/api/users', require('./routes/users'));

// ผลลัพธ์:
// GET    /api/users
// GET    /api/users/123
// GET    /api/users/123/posts
// GET    /api/users/123/posts/456
// POST   /api/users/123/posts
// GET    /api/users/123/orders
// GET    /api/users/123/profile
```

### ตัวอย่าง Nested Routes สำหรับ E-commerce

```javascript
// routes/shop/index.js
const express = require('express');
const router = express.Router();

// /shop
router.get('/', (req, res) => res.json({ page: 'Shop Home' }));

// /shop/categories - sub-router
router.use('/categories', require('./categories'));

// /shop/products - sub-router
router.use('/products', require('./products'));

// /shop/cart - sub-router
router.use('/cart', require('./cart'));

module.exports = router;
```

```javascript
// routes/shop/products.js
const express = require('express');
const router = express.Router({ mergeParams: true });

let products = [
  { id: 1, name: 'iPhone 15', categoryId: 1, price: 44900 },
  { id: 2, name: 'Galaxy S24', categoryId: 1, price: 35900 }
];

// GET /shop/products
router.get('/', (req, res) => {
  const { categoryId, minPrice, maxPrice, sort = 'id' } = req.query;
  
  let result = [...products];
  if (categoryId) result = result.filter(p => p.categoryId === parseInt(categoryId));
  if (minPrice) result = result.filter(p => p.price >= parseFloat(minPrice));
  if (maxPrice) result = result.filter(p => p.price <= parseFloat(maxPrice));
  
  res.json({ products: result });
});

// GET /shop/products/:productId
router.get('/:productId', (req, res) => {
  const product = products.find(p => p.id === parseInt(req.params.productId));
  if (!product) return res.status(404).json({ error: 'ไม่พบสินค้า' });
  res.json({ product });
});

// /shop/products/:productId/reviews - sub-router
router.use('/:productId/reviews', require('./product-reviews'));

module.exports = router;
```

```javascript
// routes/shop/product-reviews.js
const express = require('express');
const router = express.Router({ mergeParams: true });

const reviews = [
  { id: 1, productId: 1, userId: 1, rating: 5, comment: 'ดีมาก!' },
  { id: 2, productId: 1, userId: 2, rating: 4, comment: 'ใช้งานดี' }
];

// GET /shop/products/:productId/reviews
router.get('/', (req, res) => {
  const { productId } = req.params;
  const productReviews = reviews.filter(r => r.productId === parseInt(productId));
  
  const avgRating = productReviews.length
    ? productReviews.reduce((sum, r) => sum + r.rating, 0) / productReviews.length
    : 0;
  
  res.json({
    productId: parseInt(productId),
    reviews: productReviews,
    avgRating: avgRating.toFixed(1),
    count: productReviews.length
  });
});

// POST /shop/products/:productId/reviews
router.post('/', (req, res) => {
  const { productId } = req.params;
  const { rating, comment } = req.body;
  
  if (!rating || rating < 1 || rating > 5) {
    return res.status(400).json({ error: 'rating ต้องอยู่ระหว่าง 1-5' });
  }
  
  const newReview = {
    id: reviews.length + 1,
    productId: parseInt(productId),
    userId: 1, // จาก auth middleware
    rating,
    comment,
    createdAt: new Date()
  };
  
  reviews.push(newReview);
  res.status(201).json({ review: newReview });
});

module.exports = router;
```

---

## 4. Parameter Inheritance

Parameter inheritance ช่วยให้ nested routers เข้าถึง parameters ของ parent router:

```javascript
// ปัญหา: โดย default child router ไม่เห็น parent params
// routes/users.js
const router = express.Router();

router.use('/:userId/posts', (req, res, next) => {
  console.log(req.params.userId); // undefined! เพราะ default mergeParams: false
  next();
}, postsRouter);
```

```javascript
// แก้ปัญหา: ใช้ mergeParams: true

// routes/posts.js (child router)
const router = express.Router({ mergeParams: true });

router.get('/', (req, res) => {
  console.log(req.params.userId); // ✅ ได้ userId จาก parent
  res.json({ userId: req.params.userId });
});

module.exports = router;
```

### ตัวอย่าง Parameter Inheritance แบบลึก

```javascript
// 3 ระดับ: /orgs/:orgId/teams/:teamId/members/:memberId

// routes/orgs/teams/members.js
const express = require('express');
const router = express.Router({ mergeParams: true });

router.get('/', (req, res) => {
  const { orgId, teamId } = req.params;
  res.json({
    message: `Members ของ Team ${teamId} ใน Org ${orgId}`,
    params: req.params
  });
});

router.get('/:memberId', (req, res) => {
  const { orgId, teamId, memberId } = req.params;
  res.json({
    message: `Member ${memberId} ใน Team ${teamId} ใน Org ${orgId}`
  });
});

module.exports = router;
```

```javascript
// routes/orgs/teams.js
const express = require('express');
const router = express.Router({ mergeParams: true });
const membersRouter = require('./members');

router.get('/', (req, res) => {
  const { orgId } = req.params; // ได้จาก parent
  res.json({ orgId, teams: [] });
});

router.use('/:teamId/members', membersRouter);

module.exports = router;
```

```javascript
// routes/orgs.js
const express = require('express');
const router = express.Router();
const teamsRouter = require('./orgs/teams');

router.get('/', (req, res) => res.json({ orgs: [] }));
router.use('/:orgId/teams', teamsRouter);

module.exports = router;
```

```javascript
// app.js
app.use('/api/orgs', require('./routes/orgs'));

// ผลลัพธ์:
// GET /api/orgs
// GET /api/orgs/:orgId/teams
// GET /api/orgs/:orgId/teams/:teamId/members
// GET /api/orgs/:orgId/teams/:teamId/members/:memberId
```

---

## 5. Router-level Middleware

```javascript
const express = require('express');
const router = express.Router();

// ==============================
// Middleware สำหรับทุก routes ใน router
// ==============================
router.use((req, res, next) => {
  console.log('Router middleware:', req.method, req.path);
  next();
});

// ==============================
// Middleware สำหรับ path เฉพาะ
// ==============================
router.use('/admin', (req, res, next) => {
  if (!req.user?.isAdmin) {
    return res.status(403).json({ error: 'Admin only' });
  }
  next();
});

// ==============================
// Middleware บน route เฉพาะ
// ==============================
const validate = (schema) => (req, res, next) => {
  // validate req.body ตาม schema
  next();
};

router.post('/users', validate(userSchema), createUser);

// ==============================
// Middleware Chain
// ==============================
router.get('/secure',
  authenticate,    // ตรวจสอบ token
  authorize('admin'), // ตรวจสอบ role
  rateLimit(10),   // rate limit
  (req, res) => {  // handler
    res.json({ data: 'secure data' });
  }
);

module.exports = router;
```

### Shared Middleware Pattern

```javascript
// middleware/index.js
const jwt = require('jsonwebtoken');

exports.authenticate = (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Unauthorized' });
  
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
};

exports.authorize = (...roles) => (req, res, next) => {
  if (!roles.includes(req.user?.role)) {
    return res.status(403).json({ error: 'Forbidden' });
  }
  next();
};

exports.validate = (schema) => async (req, res, next) => {
  try {
    req.body = await schema.validateAsync(req.body, { abortEarly: false });
    next();
  } catch (err) {
    res.status(422).json({ errors: err.details.map(d => d.message) });
  }
};
```

---

## 6. Practical: Modular API Structure

### โครงสร้างที่สมบูรณ์

```
api-project/
├── src/
│   ├── routes/
│   │   ├── index.js          (main router)
│   │   ├── auth/
│   │   │   ├── index.js
│   │   │   ├── login.js
│   │   │   └── register.js
│   │   ├── users/
│   │   │   ├── index.js
│   │   │   ├── profile.js
│   │   │   └── orders.js
│   │   ├── products/
│   │   │   ├── index.js
│   │   │   ├── reviews.js
│   │   │   └── images.js
│   │   └── admin/
│   │       ├── index.js
│   │       ├── users.js
│   │       └── reports.js
│   ├── middleware/
│   │   ├── auth.js
│   │   ├── validate.js
│   │   ├── rateLimit.js
│   │   ├── cache.js
│   │   └── errorHandler.js
│   ├── controllers/
│   │   ├── userController.js
│   │   ├── productController.js
│   │   └── orderController.js
│   └── utils/
│       ├── response.js
│       ├── pagination.js
│       └── asyncHandler.js
└── app.js
```

### src/utils/response.js

```javascript
// src/utils/response.js
const successResponse = (res, data, statusCode = 200, meta = null) => {
  const response = { success: true, data };
  if (meta) response.meta = meta;
  return res.status(statusCode).json(response);
};

const errorResponse = (res, message, statusCode = 400, code = null, details = null) => {
  const response = {
    success: false,
    error: { message }
  };
  if (code) response.error.code = code;
  if (details) response.error.details = details;
  return res.status(statusCode).json(response);
};

const paginatedResponse = (res, data, total, page, limit) => {
  const totalPages = Math.ceil(total / limit);
  return successResponse(res, data, 200, {
    pagination: {
      total,
      page,
      limit,
      totalPages,
      hasNextPage: page < totalPages,
      hasPrevPage: page > 1
    }
  });
};

module.exports = { successResponse, errorResponse, paginatedResponse };
```

### src/utils/asyncHandler.js

```javascript
// src/utils/asyncHandler.js
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

module.exports = asyncHandler;
```

### src/middleware/auth.js

```javascript
// src/middleware/auth.js
const jwt = require('jsonwebtoken');
const { errorResponse } = require('../utils/response');

const authenticate = (req, res, next) => {
  const authHeader = req.headers.authorization;
  
  if (!authHeader?.startsWith('Bearer ')) {
    return errorResponse(res, 'กรุณา authenticate ก่อน', 401, 'UNAUTHORIZED');
  }
  
  try {
    const token = authHeader.substring(7);
    req.user = jwt.verify(token, process.env.JWT_SECRET || 'secret');
    next();
  } catch (err) {
    const message = err.name === 'TokenExpiredError'
      ? 'Token หมดอายุ' : 'Token ไม่ถูกต้อง';
    errorResponse(res, message, 401, 'INVALID_TOKEN');
  }
};

const authorize = (...roles) => (req, res, next) => {
  if (!req.user) return errorResponse(res, 'กรุณา authenticate ก่อน', 401);
  if (!roles.includes(req.user.role)) {
    return errorResponse(res, 'ไม่มีสิทธิ์เข้าถึง', 403, 'FORBIDDEN');
  }
  next();
};

module.exports = { authenticate, authorize };
```

### src/controllers/userController.js

```javascript
// src/controllers/userController.js
const asyncHandler = require('../utils/asyncHandler');
const { successResponse, errorResponse, paginatedResponse } = require('../utils/response');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

// ข้อมูลจำลอง
let users = [
  { id: 1, name: 'สมชาย ใจดี', email: 'admin@test.com', role: 'admin', createdAt: new Date() },
  { id: 2, name: 'สมหญิง ใจงาม', email: 'user@test.com', role: 'user', createdAt: new Date() }
];

exports.getAllUsers = asyncHandler(async (req, res) => {
  const { search, role, page = 1, limit = 10 } = req.query;
  
  let result = [...users];
  if (search) result = result.filter(u =>
    u.name.includes(search) || u.email.includes(search)
  );
  if (role) result = result.filter(u => u.role === role);
  
  const total = result.length;
  const pageNum = parseInt(page);
  const limitNum = parseInt(limit);
  const data = result.slice((pageNum - 1) * limitNum, pageNum * limitNum);
  
  paginatedResponse(res, data, total, pageNum, limitNum);
});

exports.getUserById = asyncHandler(async (req, res) => {
  const user = users.find(u => u.id === parseInt(req.params.userId));
  if (!user) return errorResponse(res, 'ไม่พบผู้ใช้', 404, 'NOT_FOUND');
  successResponse(res, user);
});

exports.createUser = asyncHandler(async (req, res) => {
  const { name, email, password, role = 'user' } = req.body;
  
  if (users.find(u => u.email === email)) {
    return errorResponse(res, 'Email นี้ถูกใช้แล้ว', 409, 'DUPLICATE_EMAIL');
  }
  
  const newUser = {
    id: Math.max(...users.map(u => u.id), 0) + 1,
    name,
    email,
    role,
    createdAt: new Date()
  };
  users.push(newUser);
  
  successResponse(res, newUser, 201);
});

exports.updateUser = asyncHandler(async (req, res) => {
  const idx = users.findIndex(u => u.id === parseInt(req.params.userId));
  if (idx === -1) return errorResponse(res, 'ไม่พบผู้ใช้', 404);
  
  const allowedFields = ['name', 'email', 'role'];
  allowedFields.forEach(field => {
    if (req.body[field] !== undefined) users[idx][field] = req.body[field];
  });
  users[idx].updatedAt = new Date();
  
  successResponse(res, users[idx]);
});

exports.deleteUser = asyncHandler(async (req, res) => {
  const idx = users.findIndex(u => u.id === parseInt(req.params.userId));
  if (idx === -1) return errorResponse(res, 'ไม่พบผู้ใช้', 404);
  
  users.splice(idx, 1);
  res.status(204).end();
});

exports.login = asyncHandler(async (req, res) => {
  const { email, password } = req.body;
  
  const user = users.find(u => u.email === email);
  if (!user) return errorResponse(res, 'Email หรือ Password ไม่ถูกต้อง', 401);
  
  // จำลองการตรวจสอบ password
  if (password !== 'password123') {
    return errorResponse(res, 'Email หรือ Password ไม่ถูกต้อง', 401);
  }
  
  const token = jwt.sign(
    { id: user.id, email: user.email, role: user.role },
    process.env.JWT_SECRET || 'secret',
    { expiresIn: '7d' }
  );
  
  successResponse(res, { token, user: { id: user.id, name: user.name, role: user.role } });
});
```

### src/routes/users/index.js

```javascript
// src/routes/users/index.js
const express = require('express');
const router = express.Router();
const userController = require('../../controllers/userController');
const { authenticate, authorize } = require('../../middleware/auth');

// Logging middleware สำหรับ user routes
router.use((req, res, next) => {
  console.log(`[Users] ${req.method} ${req.originalUrl}`);
  next();
});

// Public routes
router.post('/login', userController.login);

// Protected routes (ต้อง authenticate)
router.use(authenticate);

// GET /api/users - ดูทุก users (admin เท่านั้น)
router.get('/', authorize('admin'), userController.getAllUsers);

// GET /api/users/me - ดูข้อมูลตัวเอง
router.get('/me', (req, res) => {
  const { successResponse } = require('../../utils/response');
  successResponse(res, req.user);
});

// GET /api/users/:userId
router.get('/:userId', userController.getUserById);

// POST /api/users - สร้าง user (admin เท่านั้น)
router.post('/', authorize('admin'), userController.createUser);

// PATCH /api/users/:userId
router.patch('/:userId', userController.updateUser);

// DELETE /api/users/:userId (admin เท่านั้น)
router.delete('/:userId', authorize('admin'), userController.deleteUser);

// Nested routes
router.use('/:userId/orders', require('./orders'));

module.exports = router;
```

### src/routes/users/orders.js

```javascript
// src/routes/users/orders.js
const express = require('express');
const router = express.Router({ mergeParams: true }); // รับ :userId จาก parent
const asyncHandler = require('../../utils/asyncHandler');
const { successResponse, errorResponse } = require('../../utils/response');

const orders = [
  { id: 1, userId: 1, total: 44900, status: 'completed', items: [] },
  { id: 2, userId: 1, total: 35900, status: 'pending', items: [] },
  { id: 3, userId: 2, total: 1200, status: 'shipped', items: [] }
];

// GET /api/users/:userId/orders
router.get('/', asyncHandler(async (req, res) => {
  const { userId } = req.params;
  const { status } = req.query;
  
  let userOrders = orders.filter(o => o.userId === parseInt(userId));
  if (status) userOrders = userOrders.filter(o => o.status === status);
  
  successResponse(res, userOrders);
}));

// GET /api/users/:userId/orders/:orderId
router.get('/:orderId', asyncHandler(async (req, res) => {
  const { userId, orderId } = req.params;
  
  const order = orders.find(
    o => o.userId === parseInt(userId) && o.id === parseInt(orderId)
  );
  
  if (!order) return errorResponse(res, 'ไม่พบคำสั่งซื้อ', 404);
  
  successResponse(res, order);
}));

// POST /api/users/:userId/orders
router.post('/', asyncHandler(async (req, res) => {
  const { userId } = req.params;
  const { items } = req.body;
  
  if (!items || items.length === 0) {
    return errorResponse(res, 'กรุณาระบุรายการสินค้า', 400);
  }
  
  const total = items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
  
  const newOrder = {
    id: orders.length + 1,
    userId: parseInt(userId),
    items,
    total,
    status: 'pending',
    createdAt: new Date()
  };
  
  orders.push(newOrder);
  successResponse(res, newOrder, 201);
}));

module.exports = router;
```

### src/routes/index.js - Main Router

```javascript
// src/routes/index.js
const express = require('express');
const router = express.Router();

// API Info
router.get('/', (req, res) => {
  res.json({
    name: 'My API',
    version: '1.0.0',
    endpoints: {
      auth: '/api/v1/auth',
      users: '/api/v1/users',
      products: '/api/v1/products',
      orders: '/api/v1/orders'
    },
    docs: '/api/docs'
  });
});

// Mount all route modules
router.use('/auth', require('./auth'));
router.use('/users', require('./users'));
router.use('/products', require('./products'));

module.exports = router;
```

### app.js - Main Application

```javascript
// app.js
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const morgan = require('morgan');
const path = require('path');

const app = express();
const PORT = process.env.PORT || 3000;

// ==============================
// Security & Utils Middleware
// ==============================
app.use(helmet());
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true
}));
app.use(morgan('dev'));

// ==============================
// Body Parsing
// ==============================
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// ==============================
// Static Files
// ==============================
app.use(express.static(path.join(__dirname, 'public')));

// ==============================
// Request Tracking
// ==============================
app.use((req, res, next) => {
  req.requestId = `req-${Date.now()}-${Math.random().toString(36).substring(2)}`;
  res.set('X-Request-ID', req.requestId);
  next();
});

// ==============================
// API Routes
// ==============================
const apiRoutes = require('./src/routes');
app.use('/api/v1', apiRoutes);

// ==============================
// 404 Handler
// ==============================
app.use((req, res) => {
  res.status(404).json({
    success: false,
    error: {
      message: `ไม่พบ route: ${req.method} ${req.url}`,
      code: 'NOT_FOUND'
    }
  });
});

// ==============================
// Global Error Handler
// ==============================
app.use((err, req, res, next) => {
  console.error(`[${req.requestId}] Error:`, err.message);
  
  if (process.env.NODE_ENV === 'development') {
    return res.status(err.status || 500).json({
      success: false,
      error: {
        message: err.message,
        stack: err.stack,
        requestId: req.requestId
      }
    });
  }
  
  res.status(err.status || 500).json({
    success: false,
    error: {
      message: err.isOperational ? err.message : 'Internal Server Error',
      requestId: req.requestId
    }
  });
});

// ==============================
// Start Server
// ==============================
const server = app.listen(PORT, () => {
  console.log(`\n🚀 API Server started`);
  console.log(`📡 URL: http://localhost:${PORT}`);
  console.log(`🌍 Environment: ${process.env.NODE_ENV || 'development'}`);
  console.log('\n📋 Available Routes:');
  console.log('  GET    /api/v1');
  console.log('  POST   /api/v1/users/login');
  console.log('  GET    /api/v1/users (auth required)');
  console.log('  GET    /api/v1/users/:id');
  console.log('  GET    /api/v1/users/:userId/orders');
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received. Shutting down gracefully...');
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});

module.exports = app;
```

### ทดสอบ API

```bash
# ดู API info
curl http://localhost:3000/api/v1

# Login
TOKEN=$(curl -s -X POST http://localhost:3000/api/v1/users/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@test.com","password":"password123"}' | \
  jq -r '.data.token')

# ดู users (ต้อง token)
curl http://localhost:3000/api/v1/users \
  -H "Authorization: Bearer $TOKEN"

# ดู orders ของ user
curl http://localhost:3000/api/v1/users/1/orders \
  -H "Authorization: Bearer $TOKEN"

# สร้าง order
curl -X POST http://localhost:3000/api/v1/users/1/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"items": [{"productId": 1, "price": 44900, "quantity": 1}]}'
```

---

## 7. Exercise

### Exercise 1: Complete API Structure

สร้าง API โปรเจกต์ที่มีโครงสร้างครบถ้วน:
```
/api/v1/auth - login, register, refresh token
/api/v1/users - CRUD + nested orders, reviews
/api/v1/products - CRUD + nested reviews, images
/api/v1/orders - CRUD + nested items, tracking
/api/v1/admin - manage users, products, reports
```

### Exercise 2: Router Testing

เขียน tests สำหรับ routes:
- Unit test สำหรับ route handlers
- Integration test สำหรับ route chains
- Test middleware (auth, validation)

### Exercise 3: API Documentation Router

สร้าง Router ที่ generate API documentation อัตโนมัติ:
- Scan ทุก routes
- สร้าง HTML documentation page
- Export เป็น OpenAPI/Swagger format

---

## สรุป Part 20

ในบทนี้เราได้เรียนรู้:

1. **Express Router** - mini-application สำหรับ organize routes
2. **Router Mounting** - mount routers ที่ path ต่างๆ
3. **Nested Routes** - routes ภายใน routes สำหรับ resources ที่ซ้อนกัน
4. **Parameter Inheritance** - mergeParams ให้ child router เข้าถึง parent params
5. **Router Middleware** - middleware เฉพาะใน router
6. **Practical** - Modular API structure ที่สมบูรณ์

---

## สรุปทั้ง Part 11-20

ตลอด 10 บทที่ผ่านมา เราได้เรียนรู้:

| Part | หัวข้อ |
|------|--------|
| 11 | Express.js เบื้องต้น |
| 12 | Express Routing |
| 13 | Express Middleware |
| 14 | Request/Response Objects |
| 15 | Template Engines (EJS, Pug, Handlebars) |
| 16 | Static Files |
| 17 | Form Handling |
| 18 | Cookies & Sessions |
| 19 | REST API Basics |
| 20 | Express Router ขั้นสูง |

---

## ขั้นตอนถัดไป

ใน Part 21-30 เราจะเรียนรู้เรื่อง:
- Database Integration (MongoDB, MySQL)
- Authentication (JWT, OAuth)
- Testing (Jest, Supertest)
- Deployment (Docker, PM2, Cloud)
- Performance Optimization

---

*Part 20 | Node.js/Express.js Course | ขั้นตอนที่ 20 จาก 1000*
