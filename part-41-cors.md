# Part 41: CORS (Cross-Origin Resource Sharing)

> ขั้นตอนที่ 41-41 จาก 1000

---

## สารบัญ

1. [CORS คืออะไร](#cors-คืออะไร)
2. [cors Middleware](#cors-middleware)
3. [Preflight Requests](#preflight-requests)
4. [Credentials](#credentials)
5. [Dynamic Origins](#dynamic-origins)
6. [Common CORS Problems](#common-cors-problems)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## CORS คืออะไร

CORS (Cross-Origin Resource Sharing) เป็น security mechanism ที่ browser ใช้ ป้องกัน JavaScript จาก domain หนึ่งเข้าถึง resources ของอีก domain หนึ่งโดยไม่ได้รับอนุญาต

### Same-Origin Policy

```
Origin = Protocol + Host + Port

https://example.com:443

https://example.com/page1      → Same Origin ✅
https://example.com/api        → Same Origin ✅  
http://example.com             → Different protocol ❌
https://sub.example.com        → Different host ❌
https://example.com:8080       → Different port ❌
https://other-site.com         → Different host ❌
```

### CORS Flow

```
Browser (http://frontend.com)    Server (http://api.com)

1. JavaScript ทำ request ไป api.com
   GET /api/users
   Origin: http://frontend.com

2. Server ตอบกลับพร้อม CORS headers
   Access-Control-Allow-Origin: http://frontend.com
   (หรือ * สำหรับ public API)

3. Browser อ่าน response ได้
```

### ทำไม CORS จำเป็น

```javascript
// ถ้าไม่มี CORS:
// evil-site.com สามารถทำ request ไป bank.com
// โดยใช้ credentials ของ user (cookies)

// <script src="evil-site.com">
fetch('https://bank.com/api/transfer', {
  method: 'POST',
  credentials: 'include',  // ส่ง cookies ของ bank.com
  body: JSON.stringify({ to: 'hacker', amount: 10000 }),
});
// Browser blocks นี้ด้วย CORS
```

---

## cors Middleware

```bash
npm install cors
```

### Basic Setup

```javascript
const express = require('express');
const cors = require('cors');

const app = express();

// 1. Allow all origins (สำหรับ public API)
app.use(cors());

// 2. Allow specific origin
app.use(cors({
  origin: 'http://localhost:5173',
}));

// 3. Allow multiple origins (array)
app.use(cors({
  origin: ['http://localhost:3000', 'http://localhost:5173', 'https://myapp.com'],
}));
```

### Full Configuration

```javascript
const corsOptions = {
  // Origins ที่อนุญาต
  origin: 'https://myapp.com',
  
  // HTTP methods ที่อนุญาต
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  
  // Headers ที่ client ส่งได้
  allowedHeaders: [
    'Content-Type',
    'Authorization',
    'X-Requested-With',
    'Accept',
    'X-API-Key',
  ],
  
  // Headers ที่ client อ่านได้จาก response
  exposedHeaders: [
    'X-Total-Count',
    'X-Page',
    'RateLimit-Limit',
    'RateLimit-Remaining',
  ],
  
  // อนุญาต cookies/credentials
  credentials: true,
  
  // Cache preflight request (seconds)
  maxAge: 86400, // 24 hours
  
  // ส่ง 204 สำหรับ OPTIONS request
  optionsSuccessStatus: 204,
};

app.use(cors(corsOptions));
```

### CORS สำหรับ Specific Routes

```javascript
// Route-specific CORS
const publicCors = cors(); // allow all
const privateCors = cors({ origin: 'https://admin.myapp.com', credentials: true });

// Public API
app.get('/api/public/data', publicCors, controller.getPublicData);

// Admin API
app.use('/api/admin', privateCors, adminRoutes);

// Handle preflight manually สำหรับ specific route
app.options('/api/admin/*', privateCors);
app.use('/api/admin', privateCors, adminRoutes);
```

---

## Preflight Requests

Browser ส่ง preflight request (OPTIONS) ก่อน actual request สำหรับ non-simple requests

### Simple Requests (ไม่มี Preflight)

```
Method: GET, POST, HEAD

Headers เฉพาะ:
- Accept
- Accept-Language
- Content-Type: application/x-www-form-urlencoded, multipart/form-data, text/plain
```

### Non-Simple Requests (มี Preflight)

```
Method: PUT, DELETE, PATCH, หรือ
Content-Type: application/json หรือ
Custom headers: Authorization, X-Custom-Header
```

### Preflight Flow

```
Browser                          Server
  |                                |
  |-- OPTIONS /api/users -------->|
  |   Origin: https://app.com     |
  |   Access-Control-Request-Method: POST
  |   Access-Control-Request-Headers: Content-Type, Authorization
  |                                |
  |<-- 204 No Content ------------|
  |   Access-Control-Allow-Origin: https://app.com
  |   Access-Control-Allow-Methods: GET, POST, PUT, DELETE
  |   Access-Control-Allow-Headers: Content-Type, Authorization
  |   Access-Control-Max-Age: 86400
  |                                |
  |-- POST /api/users ----------->|
  |   Origin: https://app.com     |
  |   Authorization: Bearer xxx   |
  |   Content-Type: application/json
  |                                |
  |<-- 201 Created ----------------|
```

### Handle Preflight ใน Express

```javascript
// cors() จัดการ preflight ให้อัตโนมัติ
app.use(cors(corsOptions));

// แต่ถ้าต้องการ explicit handle:
app.options('*', cors(corsOptions)); // ตอบ OPTIONS requests ทุก path

// หรือสำหรับ specific route:
app.options('/api/users', cors(corsOptions));
app.post('/api/users', cors(corsOptions), createUser);
```

---

## Credentials

เมื่อต้องการส่ง cookies หรือ HTTP authentication:

```javascript
// Server
app.use(cors({
  origin: 'https://app.com',  // ต้องระบุ origin เฉพาะ (ไม่ใช่ *)
  credentials: true,           // อนุญาต credentials
}));

// Client (JavaScript)
fetch('https://api.com/user', {
  credentials: 'include',     // ส่ง cookies
});

// หรือด้วย axios
axios.get('https://api.com/user', {
  withCredentials: true,
});
```

### ข้อผิดพลาดที่พบบ่อย

```
❌ Error:
"The value of the 'Access-Control-Allow-Origin' header in the response
must not be the wildcard '*' when the request's credentials mode is 'include'"

✅ Solution:
ถ้าใช้ credentials: true ต้องระบุ origin เฉพาะ ไม่ใช่ *

app.use(cors({
  origin: 'https://app.com',  // NOT '*'
  credentials: true,
}));
```

---

## Dynamic Origins

เมื่อต้องการ allow หลาย origins หรือ origins แบบ dynamic:

```javascript
// ใช้ function เป็น origin
const allowedOrigins = [
  'http://localhost:3000',
  'http://localhost:5173',
  'https://myapp.com',
  'https://staging.myapp.com',
];

app.use(cors({
  origin: function(origin, callback) {
    // อนุญาต requests ที่ไม่มี origin (เช่น mobile apps, Postman)
    if (!origin) return callback(null, true);
    
    if (allowedOrigins.includes(origin)) {
      return callback(null, true);
    }
    
    callback(new Error(`CORS blocked: ${origin} not allowed`));
  },
  credentials: true,
}));
```

### Pattern-based Origins

```javascript
// อนุญาต subdomains ทั้งหมดของ myapp.com
app.use(cors({
  origin: function(origin, callback) {
    if (!origin) return callback(null, true);
    
    // อนุญาต *.myapp.com
    if (/^https:\/\/([\w-]+\.)?myapp\.com$/.test(origin)) {
      return callback(null, true);
    }
    
    // Development
    if (process.env.NODE_ENV === 'development') {
      if (/^http:\/\/localhost:\d+$/.test(origin)) {
        return callback(null, true);
      }
    }
    
    callback(new Error(`CORS: origin ${origin} not allowed`));
  },
  credentials: true,
}));
```

### Database-driven Origins

```javascript
// โหลด allowed origins จาก database
async function getDynamicCorsOptions() {
  const allowedOrigins = await db.query(
    'SELECT origin FROM cors_whitelist WHERE active = true'
  );
  
  const origins = allowedOrigins.rows.map(r => r.origin);
  
  return {
    origin: function(origin, callback) {
      if (!origin || origins.includes(origin)) {
        return callback(null, true);
      }
      callback(new Error('CORS not allowed'));
    },
    credentials: true,
  };
}

// Cache origins และ refresh เป็นระยะ
let cachedCorsOptions;
let lastRefresh = 0;

async function getCorsOptions() {
  if (Date.now() - lastRefresh > 5 * 60 * 1000) { // refresh ทุก 5 นาที
    cachedCorsOptions = await getDynamicCorsOptions();
    lastRefresh = Date.now();
  }
  return cachedCorsOptions;
}

app.use(async (req, res, next) => {
  const options = await getCorsOptions();
  cors(options)(req, res, next);
});
```

---

## Common CORS Problems

### CORS Errors ที่พบบ่อย

```javascript
// 1. Missing CORS headers
// Error: No 'Access-Control-Allow-Origin' header is present
// Solution: เพิ่ม cors() middleware

// 2. Wrong origin
// Error: Origin 'http://localhost:3000' is not allowed
// Solution: เพิ่ม origin ใน allowed list

// 3. Credentials with wildcard
// Error: Cannot use wildcard with credentials
// Solution: ระบุ origin เฉพาะเมื่อใช้ credentials: true

// 4. Missing preflight handler
// Error: Method 'PUT' is not allowed
// Solution: เพิ่ม app.options('*', cors()) สำหรับ preflight

// 5. Request header not allowed
// Error: Header 'X-Custom-Header' not allowed
// Solution: เพิ่ม header ใน allowedHeaders
```

### CORS Debugging

```javascript
// Debug CORS issues
app.use((req, res, next) => {
  if (process.env.NODE_ENV === 'development') {
    console.log('CORS Debug:', {
      origin: req.headers.origin,
      method: req.method,
      path: req.path,
    });
  }
  next();
});

// Test CORS headers ด้วย curl
// curl -v -X OPTIONS http://localhost:3000/api/users \
//   -H "Origin: http://localhost:5173" \
//   -H "Access-Control-Request-Method: POST" \
//   -H "Access-Control-Request-Headers: Content-Type,Authorization"
```

### Production CORS Configuration

```javascript
// config/cors.js
const config = require('./index');

const corsConfig = {
  development: {
    origin: ['http://localhost:3000', 'http://localhost:5173', 'http://127.0.0.1:5173'],
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-Requested-With'],
    exposedHeaders: ['X-Total-Count', 'RateLimit-Remaining'],
    maxAge: 600, // 10 minutes (shorter for dev)
  },
  
  production: {
    origin: function(origin, callback) {
      const allowed = config.app.allowedOrigins; // จาก env
      
      if (!origin || allowed.includes(origin)) {
        return callback(null, true);
      }
      
      console.error(`CORS blocked: ${origin}`);
      callback(new Error('Not allowed by CORS'));
    },
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization'],
    exposedHeaders: ['X-Total-Count'],
    maxAge: 86400, // 24 hours
    optionsSuccessStatus: 204,
  },
  
  test: {
    origin: '*', // ง่ายที่สุดสำหรับ tests
  },
};

module.exports = corsConfig[process.env.NODE_ENV || 'development'];
```

```javascript
// app.js
const cors = require('cors');
const corsConfig = require('./config/cors');

app.use(cors(corsConfig));
app.options('*', cors(corsConfig)); // Preflight
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic CORS Setup

ตั้งค่า CORS สำหรับ API:
1. Allow: `http://localhost:5173` และ `https://myapp.com`
2. Methods: GET, POST, PUT, DELETE
3. Headers: Content-Type, Authorization
4. Credentials: true
5. Preflight cache: 1 hour

### แบบฝึกหัดที่ 2: Dynamic Origins

สร้าง CORS config ที่:
1. อ่าน allowed origins จาก database
2. Cache origins ไว้ 5 นาที
3. อนุญาต `*.myapp.com` patterns
4. Log ทุก CORS blocked request

### แบบฝึกหัดที่ 3: Testing CORS

เขียน tests สำหรับ CORS:
1. ทดสอบ allowed origin
2. ทดสอบ blocked origin
3. ทดสอบ preflight request
4. ทดสอบ credentials ด้วย cookies

---

## สรุป

CORS ป้องกัน cross-origin attacks แต่ต้องตั้งค่าให้ถูกต้อง

| Setting | ค่า |
|---------|-----|
| Public API | `origin: '*'`, `credentials: false` |
| Authenticated API | `origin: [specific origins]`, `credentials: true` |
| Development | อนุญาต localhost origins |
| Production | เฉพาะ production domains |

**ถัดไป**: [Part 42: Security Headers →](./part-42-security-headers.md)
