# Part 13: Express Middleware
## ขั้นตอนที่ 13-3 จาก 1000

---

## สารบัญ
1. Middleware คืออะไร?
2. Application-level Middleware
3. Router-level Middleware
4. Error-handling Middleware
5. Third-party Middleware (morgan, cors, helmet)
6. สร้าง Custom Middleware
7. Middleware Ordering
8. Practical: Authentication Middleware
9. Exercise

---

## 1. Middleware คืออะไร?

Middleware คือ function ที่ทำงานระหว่าง request และ response ใน Express pipeline:

```
Request → Middleware 1 → Middleware 2 → Middleware N → Route Handler → Response
```

### โครงสร้าง Middleware Function

```javascript
// Middleware function มี 3 parameters: req, res, next
function myMiddleware(req, res, next) {
  // 1. ทำงานบางอย่าง
  console.log('Middleware กำลังทำงาน');
  
  // 2. แก้ไข req หรือ res object (optional)
  req.customData = 'some data';
  
  // 3. เรียก next() เพื่อไปยัง middleware ถัดไป
  //    ถ้าไม่เรียก next() request จะหยุดที่นี่
  next();
  
  // หรือส่ง response เพื่อหยุด chain
  // res.json({ error: 'Something went wrong' });
}
```

### Middleware Chain

```javascript
const express = require('express');
const app = express();

// Middleware ทุกตัวทำงานตามลำดับ
app.use((req, res, next) => {
  console.log('Step 1: Request received');
  next(); // ไปต่อ
});

app.use((req, res, next) => {
  console.log('Step 2: Processing request');
  req.processedAt = new Date();
  next(); // ไปต่อ
});

app.use((req, res, next) => {
  console.log('Step 3: About to handle route');
  next(); // ไปต่อ
});

app.get('/', (req, res) => {
  console.log('Step 4: Route handler');
  res.json({
    message: 'Hello!',
    processedAt: req.processedAt
  });
});

// Output เมื่อมี request:
// Step 1: Request received
// Step 2: Processing request
// Step 3: About to handle route
// Step 4: Route handler
```

### ประเภทของ Middleware

| ประเภท | ใช้กับ | ตัวอย่าง |
|--------|--------|---------|
| Application-level | ทุก requests | logging, authentication |
| Router-level | routes ใน router | route-specific auth |
| Error-handling | จัดการ errors | error logging |
| Built-in | Express built-in | express.json(), express.static() |
| Third-party | npm packages | morgan, cors, helmet |

---

## 2. Application-level Middleware

Application-level middleware ใช้กับทุก requests ที่เข้ามา:

```javascript
const express = require('express');
const app = express();

// ==============================
// สำหรับทุก routes
// ==============================
app.use((req, res, next) => {
  console.log(`${new Date().toISOString()} - ${req.method} ${req.url}`);
  next();
});

// ==============================
// สำหรับ path เฉพาะ
// ==============================
app.use('/api', (req, res, next) => {
  console.log('API request:', req.method, req.path);
  next();
});

// ==============================
// หลาย middleware ใน use()
// ==============================
app.use('/admin',
  (req, res, next) => {
    console.log('Admin middleware 1');
    next();
  },
  (req, res, next) => {
    console.log('Admin middleware 2');
    next();
  }
);
```

### Built-in Middleware

```javascript
const express = require('express');
const path = require('path');
const app = express();

// แปลง JSON body (Content-Type: application/json)
app.use(express.json());

// แปลง URL-encoded form data (Content-Type: application/x-www-form-urlencoded)
app.use(express.urlencoded({ extended: true }));

// Serve static files จาก public folder
app.use(express.static(path.join(__dirname, 'public')));

// Serve static files ที่ virtual path
app.use('/static', express.static(path.join(__dirname, 'public')));
```

### Conditional Middleware

```javascript
// ใช้ middleware เฉพาะ environment บางอย่าง
if (process.env.NODE_ENV === 'development') {
  app.use((req, res, next) => {
    console.log('DEBUG:', req.method, req.url, req.body);
    next();
  });
}

// ใช้ middleware เฉพาะบาง methods
app.use((req, res, next) => {
  if (req.method === 'POST' || req.method === 'PUT') {
    console.log('Mutation request:', req.body);
  }
  next();
});
```

---

## 3. Router-level Middleware

Router-level middleware ทำงานเหมือน Application-level แต่ bind กับ `express.Router()`:

```javascript
const express = require('express');
const router = express.Router();

// Middleware สำหรับทุก routes ใน router นี้
router.use((req, res, next) => {
  console.log('Router-level middleware');
  next();
});

// Middleware สำหรับ path เฉพาะ
router.use('/protected', (req, res, next) => {
  const token = req.headers.authorization;
  if (!token) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
});

router.get('/', (req, res) => {
  res.json({ message: 'Router home' });
});

router.get('/protected/data', (req, res) => {
  res.json({ message: 'Protected data' });
});

module.exports = router;
```

### ตัวอย่าง Router Middleware สำหรับ API

```javascript
// routes/api.js
const express = require('express');
const router = express.Router();

// Rate limiting middleware (แบบง่าย)
const requestCounts = {};
router.use((req, res, next) => {
  const ip = req.ip;
  const now = Date.now();
  const window = 60 * 1000; // 1 นาที
  const limit = 100; // 100 requests ต่อนาที
  
  if (!requestCounts[ip]) {
    requestCounts[ip] = { count: 0, resetAt: now + window };
  }
  
  if (now > requestCounts[ip].resetAt) {
    requestCounts[ip] = { count: 0, resetAt: now + window };
  }
  
  requestCounts[ip].count++;
  
  if (requestCounts[ip].count > limit) {
    return res.status(429).json({
      error: 'Too Many Requests',
      retryAfter: Math.ceil((requestCounts[ip].resetAt - now) / 1000)
    });
  }
  
  // เพิ่ม rate limit headers
  res.set('X-RateLimit-Limit', limit);
  res.set('X-RateLimit-Remaining', limit - requestCounts[ip].count);
  
  next();
});

// Content-type validation middleware
router.use((req, res, next) => {
  if (['POST', 'PUT', 'PATCH'].includes(req.method)) {
    if (!req.is('application/json')) {
      return res.status(415).json({
        error: 'Content-Type ต้องเป็น application/json'
      });
    }
  }
  next();
});

router.get('/data', (req, res) => {
  res.json({ message: 'API data' });
});

module.exports = router;
```

---

## 4. Error-handling Middleware

Error-handling middleware มี **4 parameters**: `err, req, res, next`

```javascript
const express = require('express');
const app = express();

// ==============================
// สร้าง custom error
// ==============================
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
  }
}

// ==============================
// Route ที่อาจเกิด error
// ==============================
app.get('/risky', (req, res, next) => {
  try {
    // โค้ดที่อาจเกิด error
    const result = JSON.parse('invalid json');
    res.json(result);
  } catch (err) {
    next(err); // ส่ง error ไปยัง error handler
  }
});

app.get('/not-found/:id', (req, res, next) => {
  const id = req.params.id;
  const item = null; // ไม่พบข้อมูล
  
  if (!item) {
    return next(new AppError(`ไม่พบ item ID: ${id}`, 404));
  }
  
  res.json(item);
});

// Async error handling
app.get('/async', async (req, res, next) => {
  try {
    const data = await fetchSomeData(); // อาจ throw error
    res.json(data);
  } catch (err) {
    next(err);
  }
});

// ==============================
// Error Handler (ต้องอยู่หลัง routes)
// ==============================

// 404 handler
app.use((req, res, next) => {
  next(new AppError(`ไม่พบ route: ${req.method} ${req.url}`, 404));
});

// Global error handler (4 parameters!)
app.use((err, req, res, next) => {
  // กำหนด default values
  err.statusCode = err.statusCode || 500;
  err.message = err.message || 'Internal Server Error';
  
  // Log error
  if (err.statusCode >= 500) {
    console.error('SERVER ERROR:', err);
  }
  
  // Development mode: ส่ง stack trace
  if (process.env.NODE_ENV === 'development') {
    return res.status(err.statusCode).json({
      success: false,
      error: err.message,
      stack: err.stack
    });
  }
  
  // Production mode: ส่งเฉพาะ operational errors
  if (err.isOperational) {
    return res.status(err.statusCode).json({
      success: false,
      error: err.message
    });
  }
  
  // Programming errors ใน production: ไม่เปิดเผยรายละเอียด
  res.status(500).json({
    success: false,
    error: 'เกิดข้อผิดพลาดภายใน Server'
  });
});

app.listen(3000);
```

### Async Error Wrapper

```javascript
// utils/asyncHandler.js
// Helper สำหรับ async route handlers
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

module.exports = asyncHandler;
```

```javascript
// ใช้ asyncHandler
const asyncHandler = require('./utils/asyncHandler');

app.get('/users', asyncHandler(async (req, res) => {
  const users = await User.findAll(); // ถ้า error จะถูกส่งไป error handler อัตโนมัติ
  res.json({ users });
}));

// ไม่ต้องใช้ try-catch แล้ว!
```

---

## 5. Third-party Middleware

### morgan - HTTP Request Logger

```javascript
const express = require('express');
const morgan = require('morgan');
const app = express();

// ==============================
// Morgan Formats
// ==============================

// 'dev' - colorized output สำหรับ development
// GET /users 200 5.234 ms - 142
app.use(morgan('dev'));

// 'combined' - Apache combined format สำหรับ production
// ::1 - - [01/Jan/2024:00:00:00 +0000] "GET / HTTP/1.1" 200 123
app.use(morgan('combined'));

// 'short' - สั้นกว่า default
app.use(morgan('short'));

// 'tiny' - สั้นมาก
app.use(morgan('tiny'));

// Custom format
app.use(morgan(':method :url :status :res[content-length] - :response-time ms'));

// ==============================
// Log เฉพาะ errors
// ==============================
app.use(morgan('combined', {
  skip: (req, res) => res.statusCode < 400
}));

// ==============================
// บันทึก log ลงไฟล์
// ==============================
const fs = require('fs');
const path = require('path');

const accessLogStream = fs.createWriteStream(
  path.join(__dirname, 'logs', 'access.log'),
  { flags: 'a' }
);

app.use(morgan('combined', { stream: accessLogStream }));
```

### cors - Cross-Origin Resource Sharing

```javascript
const express = require('express');
const cors = require('cors');
const app = express();

// ==============================
// อนุญาตทุก origins (ไม่แนะนำใน production)
// ==============================
app.use(cors());

// ==============================
// กำหนด origins ที่อนุญาต
// ==============================
const corsOptions = {
  origin: ['http://localhost:3000', 'https://myapp.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true, // อนุญาต cookies
  maxAge: 86400 // cache preflight 24 ชั่วโมง
};

app.use(cors(corsOptions));

// ==============================
// CORS แบบ dynamic
// ==============================
const allowedOrigins = ['http://localhost:3000', 'https://myapp.com'];

app.use(cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  }
}));

// ==============================
// CORS สำหรับ specific routes
// ==============================
app.get('/public', cors(), (req, res) => {
  res.json({ message: 'Public data' });
});

app.post('/private', cors(corsOptions), (req, res) => {
  res.json({ message: 'Private data' });
});
```

### helmet - Security Headers

```javascript
const express = require('express');
const helmet = require('helmet');
const app = express();

// ==============================
// Helmet พื้นฐาน (แนะนำ)
// ==============================
app.use(helmet());

// helmet ตั้งค่า headers ต่อไปนี้:
// - Content-Security-Policy
// - X-DNS-Prefetch-Control
// - X-Frame-Options
// - X-Powered-By (removed)
// - Strict-Transport-Security
// - X-Download-Options
// - X-Content-Type-Options
// - X-XSS-Protection

// ==============================
// กำหนดค่าเอง
// ==============================
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https://res.cloudinary.com"],
      fontSrc: ["'self'", "https://fonts.gstatic.com"],
    }
  },
  crossOriginEmbedderPolicy: false, // ถ้าใช้ iframe
  frameguard: { action: 'deny' } // ป้องกัน clickjacking
}));

// ==============================
// ปิดบาง middleware
// ==============================
app.use(helmet({
  contentSecurityPolicy: false, // ปิด CSP (ถ้าจำเป็น)
}));
```

### express-rate-limit

```javascript
const rateLimit = require('express-rate-limit');

// Global rate limit
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 นาที
  max: 100, // 100 requests ต่อ window
  message: {
    error: 'คำขอมากเกินไป กรุณารอสักครู่'
  },
  standardHeaders: true, // Return rate limit info ใน headers
  legacyHeaders: false
});

app.use(globalLimiter);

// Strict limit สำหรับ auth routes
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5, // เพียง 5 ครั้งต่อ 15 นาที
  message: { error: 'พยายาม login มากเกินไป' }
});

app.use('/api/auth', authLimiter);
```

---

## 6. สร้าง Custom Middleware

### Request Logger

```javascript
// middleware/logger.js
const logger = (req, res, next) => {
  const start = Date.now();
  const { method, url, ip } = req;
  
  // Override res.json เพื่อ log response
  const originalJson = res.json.bind(res);
  res.json = (body) => {
    const duration = Date.now() - start;
    const statusCode = res.statusCode;
    
    const logEntry = {
      timestamp: new Date().toISOString(),
      method,
      url,
      ip,
      statusCode,
      duration: `${duration}ms`
    };
    
    console.log(JSON.stringify(logEntry));
    return originalJson(body);
  };
  
  next();
};

module.exports = logger;
```

### Request Validator

```javascript
// middleware/validate.js
const validate = (schema) => (req, res, next) => {
  const { error } = schema.validate(req.body);
  
  if (error) {
    return res.status(400).json({
      success: false,
      errors: error.details.map(d => d.message)
    });
  }
  
  next();
};

// ใช้กับ Joi schema
// const Joi = require('joi');
// const userSchema = Joi.object({
//   name: Joi.string().required(),
//   email: Joi.string().email().required()
// });
// app.post('/users', validate(userSchema), createUser);

module.exports = validate;
```

### API Key Middleware

```javascript
// middleware/apiKey.js
const VALID_API_KEYS = new Set([
  'key_123abc',
  'key_456def',
  'key_789ghi'
]);

const requireApiKey = (req, res, next) => {
  const apiKey = req.headers['x-api-key'];
  
  if (!apiKey) {
    return res.status(401).json({
      error: 'กรุณาระบุ API Key ใน header X-API-Key'
    });
  }
  
  if (!VALID_API_KEYS.has(apiKey)) {
    return res.status(403).json({
      error: 'API Key ไม่ถูกต้อง'
    });
  }
  
  next();
};

module.exports = requireApiKey;
```

### Response Time Middleware

```javascript
// middleware/responseTime.js
const responseTime = (req, res, next) => {
  const start = process.hrtime.bigint();
  
  res.on('finish', () => {
    const end = process.hrtime.bigint();
    const duration = Number(end - start) / 1e6; // แปลงเป็น milliseconds
    console.log(`${req.method} ${req.url} - ${duration.toFixed(2)}ms`);
  });
  
  next();
};

module.exports = responseTime;
```

### Request ID Middleware

```javascript
// middleware/requestId.js
const { v4: uuidv4 } = require('uuid');

const requestId = (req, res, next) => {
  const id = req.headers['x-request-id'] || uuidv4();
  req.id = id;
  res.set('X-Request-ID', id);
  next();
};

module.exports = requestId;
```

---

## 7. Middleware Ordering

ลำดับของ middleware สำคัญมาก:

```javascript
const express = require('express');
const morgan = require('morgan');
const helmet = require('helmet');
const cors = require('cors');
const app = express();

// ==============================
// ลำดับที่แนะนำ
// ==============================

// 1. Security (ต้องมาก่อน)
app.use(helmet());
app.use(cors());

// 2. Logging (ต้องมาก่อน route handlers)
app.use(morgan('dev'));

// 3. Body Parsing (ต้องมาก่อน routes ที่ต้องการ body)
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// 4. Static Files
app.use(express.static('public'));

// 5. Custom middleware
app.use(requestIdMiddleware);
app.use(authenticationMiddleware);

// 6. Routes
app.use('/api/v1', apiRoutes);

// 7. 404 Handler (ต้องอยู่หลัง routes)
app.use((req, res) => {
  res.status(404).json({ error: 'Not Found' });
});

// 8. Error Handler (ต้องอยู่สุดท้าย)
app.use((err, req, res, next) => {
  res.status(err.status || 500).json({ error: err.message });
});
```

### ตัวอย่างที่ผิด vs ถูก

```javascript
// ❌ ผิด: Express.json มาหลัง routes
app.post('/users', (req, res) => {
  console.log(req.body); // undefined! เพราะยังไม่ parse
  res.json({ received: req.body });
});
app.use(express.json()); // สาย!

// ✅ ถูก: Express.json มาก่อน routes
app.use(express.json()); // parse body ก่อน
app.post('/users', (req, res) => {
  console.log(req.body); // { name: 'John', email: 'john@example.com' }
  res.json({ received: req.body });
});
```

```javascript
// ❌ ผิด: Error handler ไม่ถูกต้อง (ขาด parameter)
app.use((err, res) => { // ขาด req และ next
  res.status(500).json({ error: err.message });
});

// ✅ ถูก: Error handler ต้องมี 4 parameters
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});
```

---

## 8. Practical: Authentication Middleware

สร้าง Authentication System ด้วย JWT:

### โครงสร้างไฟล์

```
auth-app/
├── middleware/
│   ├── auth.js
│   ├── authorize.js
│   └── validate.js
├── routes/
│   ├── auth.js
│   └── protected.js
└── app.js
```

### middleware/auth.js

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key-change-in-production';

// Middleware ตรวจสอบ JWT token
const authenticate = (req, res, next) => {
  // ดึง token จาก header
  const authHeader = req.headers.authorization;
  
  if (!authHeader) {
    return res.status(401).json({
      success: false,
      error: 'ไม่พบ Authorization header'
    });
  }
  
  // ตรวจสอบรูปแบบ "Bearer <token>"
  const parts = authHeader.split(' ');
  if (parts.length !== 2 || parts[0] !== 'Bearer') {
    return res.status(401).json({
      success: false,
      error: 'รูปแบบ Authorization header ไม่ถูกต้อง (ต้องเป็น: Bearer <token>)'
    });
  }
  
  const token = parts[1];
  
  try {
    // ตรวจสอบและ decode token
    const decoded = jwt.verify(token, JWT_SECRET);
    
    // เพิ่มข้อมูล user ใน req
    req.user = {
      id: decoded.id,
      email: decoded.email,
      role: decoded.role
    };
    
    next();
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({
        success: false,
        error: 'Token หมดอายุแล้ว กรุณา login ใหม่'
      });
    }
    
    if (err.name === 'JsonWebTokenError') {
      return res.status(401).json({
        success: false,
        error: 'Token ไม่ถูกต้อง'
      });
    }
    
    next(err);
  }
};

// Middleware สำหรับ optional authentication
const optionalAuth = (req, res, next) => {
  const authHeader = req.headers.authorization;
  
  if (!authHeader) {
    return next(); // ไม่มี token ก็ผ่าน
  }
  
  authenticate(req, res, next);
};

module.exports = { authenticate, optionalAuth };
```

### middleware/authorize.js

```javascript
// middleware/authorize.js

// สร้าง middleware ที่ต้องการ roles เฉพาะ
const authorize = (...roles) => {
  return (req, res, next) => {
    // ต้องผ่าน authenticate ก่อน
    if (!req.user) {
      return res.status(401).json({
        success: false,
        error: 'กรุณา login ก่อน'
      });
    }
    
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        error: `สิทธิ์ไม่เพียงพอ ต้องการ role: ${roles.join(' หรือ ')}`
      });
    }
    
    next();
  };
};

// ตัวอย่างการใช้งาน:
// app.get('/admin', authenticate, authorize('admin'), handler)
// app.get('/data', authenticate, authorize('admin', 'editor'), handler)

module.exports = authorize;
```

### routes/auth.js

```javascript
// routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');

const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';

// ข้อมูลผู้ใช้จำลอง
const users = [
  {
    id: 1,
    name: 'สมชาย ใจดี',
    email: 'admin@example.com',
    password: '$2a$10$...', // bcrypt hash
    role: 'admin'
  },
  {
    id: 2,
    name: 'สมหญิง ใจงาม',
    email: 'user@example.com',
    password: '$2a$10$...',
    role: 'user'
  }
];

// สร้าง token
const generateToken = (user) => {
  return jwt.sign(
    {
      id: user.id,
      email: user.email,
      role: user.role
    },
    JWT_SECRET,
    { expiresIn: '7d' }
  );
};

// POST /api/auth/register
router.post('/register', async (req, res) => {
  try {
    const { name, email, password } = req.body;
    
    if (!name || !email || !password) {
      return res.status(400).json({
        success: false,
        error: 'กรุณากรอกข้อมูลให้ครบ'
      });
    }
    
    // ตรวจสอบ email ซ้ำ
    const existingUser = users.find(u => u.email === email);
    if (existingUser) {
      return res.status(400).json({
        success: false,
        error: 'Email นี้ถูกใช้แล้ว'
      });
    }
    
    // Hash password
    const hashedPassword = await bcrypt.hash(password, 10);
    
    const newUser = {
      id: users.length + 1,
      name,
      email,
      password: hashedPassword,
      role: 'user'
    };
    
    users.push(newUser);
    
    const token = generateToken(newUser);
    
    res.status(201).json({
      success: true,
      message: 'สมัครสมาชิกสำเร็จ',
      token,
      user: {
        id: newUser.id,
        name: newUser.name,
        email: newUser.email,
        role: newUser.role
      }
    });
  } catch (err) {
    res.status(500).json({ success: false, error: 'Server Error' });
  }
});

// POST /api/auth/login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    if (!email || !password) {
      return res.status(400).json({
        success: false,
        error: 'กรุณาระบุ email และ password'
      });
    }
    
    const user = users.find(u => u.email === email);
    if (!user) {
      return res.status(401).json({
        success: false,
        error: 'Email หรือ Password ไม่ถูกต้อง'
      });
    }
    
    // ตรวจสอบ password
    const isPasswordValid = await bcrypt.compare(password, user.password);
    if (!isPasswordValid) {
      return res.status(401).json({
        success: false,
        error: 'Email หรือ Password ไม่ถูกต้อง'
      });
    }
    
    const token = generateToken(user);
    
    res.json({
      success: true,
      message: 'Login สำเร็จ',
      token,
      user: {
        id: user.id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (err) {
    res.status(500).json({ success: false, error: 'Server Error' });
  }
});

// GET /api/auth/me (ต้อง authenticate)
const { authenticate } = require('../middleware/auth');
router.get('/me', authenticate, (req, res) => {
  const user = users.find(u => u.id === req.user.id);
  if (!user) {
    return res.status(404).json({ success: false, error: 'ไม่พบผู้ใช้' });
  }
  
  res.json({
    success: true,
    user: {
      id: user.id,
      name: user.name,
      email: user.email,
      role: user.role
    }
  });
});

module.exports = router;
```

### routes/protected.js

```javascript
// routes/protected.js
const express = require('express');
const router = express.Router();
const { authenticate } = require('../middleware/auth');
const authorize = require('../middleware/authorize');

// ทุก routes ใน router นี้ต้อง authenticate
router.use(authenticate);

// GET /api/profile - ทุก user ที่ login
router.get('/profile', (req, res) => {
  res.json({
    success: true,
    message: 'ข้อมูลโปรไฟล์',
    user: req.user
  });
});

// GET /api/admin - เฉพาะ admin
router.get('/admin', authorize('admin'), (req, res) => {
  res.json({
    success: true,
    message: 'Admin panel',
    data: { secret: 'admin-only-data' }
  });
});

// GET /api/editor - admin หรือ editor
router.get('/editor', authorize('admin', 'editor'), (req, res) => {
  res.json({
    success: true,
    message: 'Editor area'
  });
});

module.exports = router;
```

### app.js สำหรับ Auth App

```javascript
// app.js
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const morgan = require('morgan');

const app = express();

// Middleware
app.use(helmet());
app.use(cors());
app.use(morgan('dev'));
app.use(express.json());

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api', require('./routes/protected'));

// Error Handler
app.use((err, req, res, next) => {
  console.error(err);
  res.status(err.status || 500).json({
    success: false,
    error: err.message || 'Internal Server Error'
  });
});

app.listen(3000, () => {
  console.log('Auth API: http://localhost:3000');
});
```

### ทดสอบ Authentication

```bash
# Register
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name": "ทดสอบ", "email": "test@example.com", "password": "password123"}'

# Login
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com", "password": "password123"}'

# ดูโปรไฟล์ (ต้องใส่ token)
curl http://localhost:3000/api/profile \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# เข้า Admin (เฉพาะ admin)
curl http://localhost:3000/api/admin \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN"
```

---

## 9. Exercise

### Exercise 1: Logging Middleware

สร้าง middleware ที่:
- บันทึก request ทุกตัวลงในไฟล์ `logs/requests.log`
- Format: `[timestamp] METHOD /path STATUS duration`
- ข้ามการ log static files (path ที่ขึ้นต้นด้วย `/public`)

### Exercise 2: Request Validation Middleware

สร้าง middleware factory ที่ validate request body:

```javascript
// ใช้แบบนี้
app.post('/users', 
  validateBody({
    name: { required: true, minLength: 2 },
    email: { required: true, isEmail: true },
    age: { type: 'number', min: 0, max: 150 }
  }),
  createUser
);
```

### Exercise 3: Complete Middleware Stack

สร้าง Express app ที่มี middleware stack ครบ:
1. helmet สำหรับ security
2. cors กำหนด specific origins
3. morgan บันทึก logs
4. rate limiting (100 requests/15min)
5. express.json() พร้อม request size limit
6. custom request ID middleware
7. JWT authentication (optional)
8. Error handler

---

## สรุป Part 13

ในบทนี้เราได้เรียนรู้:

1. **Middleware คืออะไร** - functions ที่ทำงานระหว่าง request และ response
2. **Application-level** - `app.use()` สำหรับทุก requests
3. **Router-level** - `router.use()` สำหรับ router เฉพาะ
4. **Error-handling** - middleware 4 parameters สำหรับจัดการ errors
5. **Third-party** - morgan, cors, helmet ที่ใช้บ่อย
6. **Custom Middleware** - logger, validator, API key
7. **Ordering** - ลำดับที่สำคัญมาก

---

*Part 13 | Node.js/Express.js Course | ขั้นตอนที่ 13 จาก 1000*
