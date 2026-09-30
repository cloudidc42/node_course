# Part 45: CSRF Protection

> ขั้นตอนที่ 45-45 จาก 1000

---

## สารบัญ

1. [CSRF Attack คืออะไร](#csrf-attack-คืออะไร)
2. [CSRF Tokens](#csrf-tokens)
3. [csurf Middleware](#csurf-middleware)
4. [SameSite Cookies](#samesite-cookies)
5. [Double Submit Pattern](#double-submit-pattern)
6. [Modern Approaches](#modern-approaches)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## CSRF Attack คืออะไร

CSRF (Cross-Site Request Forgery) หรือ "Sea Surf" คือ attack ที่หลอกให้ user ทำ action โดยไม่ตั้งใจ

### Attack Flow

```
1. User login ที่ bank.com → browser เก็บ session cookie

2. User เข้า evil.com (ยังเปิด bank.com อยู่)

3. evil.com มี HTML ที่ส่ง request ไป bank.com:
   <form action="https://bank.com/transfer" method="POST">
     <input type="hidden" name="to" value="hacker">
     <input type="hidden" name="amount" value="10000">
   </form>
   <script>document.forms[0].submit()</script>

4. Browser ส่ง POST ไป bank.com พร้อม cookie ของ user!
   → bank.com ทำ transfer เพราะ cookie valid

5. เงินถูกโอน โดย user ไม่รู้ตัว
```

### ทำไม CSRF ใช้งานได้

```
Browser พฤติกรรม default:
- ส่ง cookies ทุก request ไปยัง domain นั้น
- ไม่สนว่า request มาจาก origin ไหน

ถ้า bank.com ใช้แค่ session cookie verify:
- evil.com สามารถ forge request ได้
- เพราะ cookie ถูกส่งอัตโนมัติ
```

### CSRF ที่พบในชีวิตจริง

```html
<!-- GET-based CSRF -->
<img src="https://bank.com/transfer?to=hacker&amount=1000" style="display:none">

<!-- POST-based CSRF ด้วย form -->
<form action="https://bank.com/transfer" method="POST" id="csrf-form">
  <input type="hidden" name="to" value="hacker">
  <input type="hidden" name="amount" value="1000">
</form>
<script>document.getElementById('csrf-form').submit();</script>

<!-- JSON-based CSRF (เฉพาะ Content-Type: text/plain) -->
<form action="https://api.com/update" method="POST" enctype="text/plain">
  <input name='{"username":"hacker","admin":true,"ignore":"' value='"}'>
</form>
```

---

## CSRF Tokens

CSRF Token เป็น unique, unpredictable token ที่เพิ่มใน form/header เพื่อ verify ว่า request มาจาก site เราเอง

### วิธีทำงาน

```
Server สร้าง CSRF token และเก็บไว้ใน session:
  token = generateRandomToken()
  session.csrfToken = token

Server ส่ง token ไปยัง client (ใน HTML หรือ cookie):
  <input type="hidden" name="_csrf" value="abc123xyz">

Client ส่ง token กลับมาใน request:
  POST /transfer
  _csrf: abc123xyz

Server verify token:
  if request._csrf !== session.csrfToken → reject!
  
Evil site ไม่รู้ค่า token → request ถูก reject
```

### สร้าง CSRF Token เอง

```javascript
// utils/csrf.js
const crypto = require('crypto');

function generateCsrfToken() {
  return crypto.randomBytes(32).toString('hex');
}

function createCsrfMiddleware() {
  return {
    generateToken(req, res) {
      if (!req.session.csrfToken) {
        req.session.csrfToken = generateCsrfToken();
      }
      return req.session.csrfToken;
    },
    
    validateRequest(req, res, next) {
      // Skip สำหรับ GET, HEAD, OPTIONS
      if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
        return next();
      }
      
      const token = req.body._csrf
        || req.headers['x-csrf-token']
        || req.headers['x-xsrf-token'];
      
      const sessionToken = req.session?.csrfToken;
      
      if (!token || !sessionToken || token !== sessionToken) {
        return res.status(403).json({
          error: 'Invalid CSRF token',
          code: 'CSRF_TOKEN_INVALID',
        });
      }
      
      next();
    },
  };
}

module.exports = { createCsrfMiddleware, generateCsrfToken };
```

---

## csurf Middleware

> ⚠️ หมายเหตุ: `csurf` ถูก deprecated แล้ว แนะนำให้ใช้ `csrf-csrf` หรือ `lusca` แทน

### csrf-csrf (แนะนำ)

```bash
npm install csrf-csrf
```

```javascript
// config/csrf.js
const { doubleCsrf } = require('csrf-csrf');

const {
  generateToken,    // สร้าง CSRF token สำหรับใส่ใน response
  doubleCsrfProtection, // middleware ป้องกัน CSRF
} = doubleCsrf({
  getSecret: (req) => process.env.CSRF_SECRET || 'fallback-secret-change-this',
  
  cookieName: 'x-csrf-token',
  cookieOptions: {
    httpOnly: true,
    sameSite: 'strict',
    secure: process.env.NODE_ENV === 'production',
    path: '/',
  },
  
  getTokenFromRequest: (req) =>
    req.headers['x-csrf-token'] || req.body?._csrf,
  
  size: 64,      // token size in bytes
  
  // ไม่ validate สำหรับ methods เหล่านี้
  ignoredMethods: ['GET', 'HEAD', 'OPTIONS'],
  
  // Error handling
  errorConfig: {
    statusCode: 403,
    message: 'CSRF validation failed',
  },
});

module.exports = { generateToken, doubleCsrfProtection };
```

```javascript
// app.js
const express = require('express');
const session = require('express-session');
const { doubleCsrfProtection, generateToken } = require('./config/csrf');
const cookieParser = require('cookie-parser');

const app = express();

app.use(cookieParser());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
}));

// Apply CSRF protection
app.use(doubleCsrfProtection);

// Endpoint ส่ง CSRF token ไปยัง client (SPA)
app.get('/api/csrf-token', (req, res) => {
  const token = generateToken(req, res);
  res.json({ csrfToken: token });
});
```

```javascript
// Frontend (React/Vue SPA)
// 1. ดึง CSRF token เมื่อโหลด app
async function getCsrfToken() {
  const response = await fetch('/api/csrf-token', { credentials: 'include' });
  const { csrfToken } = await response.json();
  return csrfToken;
}

// 2. ส่ง token ใน header ทุก mutating request
const csrfToken = await getCsrfToken();

await fetch('/api/transfer', {
  method: 'POST',
  credentials: 'include',  // ส่ง cookies
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': csrfToken,  // CSRF token ใน header
  },
  body: JSON.stringify({ to: 'recipient', amount: 100 }),
});
```

### Traditional Forms (Server-rendered)

```javascript
// สำหรับ server-rendered HTML forms

app.get('/transfer', (req, res) => {
  const csrfToken = generateToken(req, res);
  
  res.render('transfer', { csrfToken });
});

app.post('/transfer', doubleCsrfProtection, async (req, res) => {
  // CSRF validated โดย middleware แล้ว
  await processTransfer(req.body);
  res.redirect('/success');
});
```

```html
<!-- transfer.html/ejs -->
<form method="POST" action="/transfer">
  <!-- CSRF token ใน hidden field -->
  <input type="hidden" name="_csrf" value="<%= csrfToken %>">
  
  <label>Transfer to:</label>
  <input type="text" name="recipient">
  
  <label>Amount:</label>
  <input type="number" name="amount">
  
  <button type="submit">Transfer</button>
</form>
```

---

## SameSite Cookies

SameSite cookie attribute เป็นวิธีง่ายที่สุดในการป้องกัน CSRF (browsers สมัยใหม่)

```javascript
// Session cookie ด้วย SameSite
app.use(session({
  secret: process.env.SESSION_SECRET,
  cookie: {
    httpOnly: true,
    secure: true,       // HTTPS only
    sameSite: 'strict', // ❶ หรือ 'lax'
  },
}));

// Auth cookie
res.cookie('token', jwtToken, {
  httpOnly: true,
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'strict',
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
});
```

### SameSite Values

```
SameSite=Strict:
  - ไม่ส่ง cookie สำหรับ cross-site requests ทั้งหมด
  - ป้องกัน CSRF ได้ 100%
  - UX: user ต้อง login ใหม่ถ้า navigate มาจาก link ภายนอก
  
SameSite=Lax (default ใน Chrome):
  - ส่ง cookie สำหรับ top-level navigation (GET)
  - ไม่ส่ง cookie สำหรับ POST, iframe, img, script requests
  - ป้องกัน CSRF POST attacks
  - UX ดีกว่า Strict
  
SameSite=None:
  - ส่ง cookie ทุก cross-site requests
  - ต้องมี Secure attribute
  - ต้องใช้ CSRF token ร่วมด้วย
  - เหมาะสำหรับ third-party cookies
```

---

## Double Submit Pattern

Double Submit Cookie เป็น stateless CSRF protection

```javascript
// วิธีทำงาน:
// 1. Server ส่ง CSRF token ใน cookie (SameSite=None, Secure)
// 2. Client ส่ง token เดียวกันใน request header/body
// 3. Server verify ว่า cookie == header value

const crypto = require('crypto');

function generateDoubleSubmitCookie(res) {
  const token = crypto.randomBytes(32).toString('hex');
  
  // ส่ง token ใน cookie (readable by JavaScript)
  res.cookie('csrf-token', token, {
    secure: true,
    sameSite: 'none',
    // ไม่ต้องการ httpOnly เพราะ JS ต้องอ่านได้
  });
  
  return token;
}

function validateDoubleSubmit(req, res, next) {
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
    return next();
  }
  
  const cookieToken = req.cookies['csrf-token'];
  const headerToken = req.headers['x-csrf-token'];
  
  if (!cookieToken || !headerToken || cookieToken !== headerToken) {
    return res.status(403).json({ error: 'CSRF validation failed' });
  }
  
  // Rotate token after use
  const newToken = generateDoubleSubmitCookie(res);
  
  next();
}

// ใช้งาน
app.use(validateDoubleSubmit);

// Client-side:
// 1. อ่าน token จาก cookie
const csrfToken = document.cookie
  .split('; ')
  .find(row => row.startsWith('csrf-token='))
  ?.split('=')[1];

// 2. ส่งใน header
fetch('/api/data', {
  method: 'POST',
  headers: { 'X-CSRF-Token': csrfToken },
});
```

---

## Modern Approaches

### JWT ใน Authorization Header (Stateless APIs)

```javascript
// APIs ที่ใช้ JWT ใน Authorization header ไม่ต้องกลัว CSRF
// เพราะ browser ไม่ส่ง Authorization header อัตโนมัติ

// Client ต้อง set header เอง
fetch('/api/data', {
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
  },
});

// ❌ CSRF attack:
// evil.com ไม่รู้ JWT token ใน localStorage
// browser ไม่ส่ง Authorization header อัตโนมัติ
// → CSRF ไม่ได้ผล!

// ⚠️ แต่ถ้า JWT อยู่ใน cookie:
res.cookie('token', jwt, { httpOnly: true }); // cookie-based JWT
// → ต้องมี CSRF protection!
```

### Origin/Referer Checking

```javascript
function checkOrigin(req, res, next) {
  // ตรวจสอบ Origin หรือ Referer header
  const origin = req.headers.origin || req.headers.referer;
  
  if (!origin) {
    // ไม่มี origin (อาจเป็น same-origin form submit หรือ curl)
    return next();
  }
  
  const allowedOrigins = [
    'https://myapp.com',
    'https://staging.myapp.com',
    ...(process.env.NODE_ENV === 'development' ? ['http://localhost:3000', 'http://localhost:5173'] : []),
  ];
  
  const originUrl = new URL(origin);
  const isAllowed = allowedOrigins.some(allowed => {
    const allowedUrl = new URL(allowed);
    return originUrl.origin === allowedUrl.origin;
  });
  
  if (!isAllowed) {
    console.warn(`CSRF: blocked request from ${origin}`);
    return res.status(403).json({ error: 'Origin not allowed' });
  }
  
  next();
}

// ใช้กับ state-changing operations
app.use(['/api/users', '/api/posts', '/api/payments'], checkOrigin);
```

### Complete CSRF Strategy

```javascript
// security/csrf.js - Complete Strategy

const { doubleCsrf } = require('csrf-csrf');

// ตั้งค่า CSRF protection
const { doubleCsrfProtection, generateToken } = doubleCsrf({
  getSecret: () => process.env.CSRF_SECRET,
  cookieOptions: {
    sameSite: 'strict',
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
  },
});

// Middleware หลัก
function csrfProtection(app) {
  // 1. Apply CSRF token validation
  app.use(doubleCsrfProtection);
  
  // 2. Endpoint สำหรับ SPA
  app.get('/api/csrf', (req, res) => {
    res.json({ token: generateToken(req, res) });
  });
}

// Export
module.exports = {
  csrfProtection,
  generateToken,
  doubleCsrfProtection,
};
```

```javascript
// app.js
const { csrfProtection } = require('./security/csrf');

const app = express();
// ... other middleware ...

csrfProtection(app);

// Routes ...
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: CSRF Demo

สร้าง demo ที่แสดงให้เห็น CSRF attack:
1. สร้าง bank.com (Express app)
2. สร้าง evil.com (static HTML)
3. แสดงว่า evil.com สามารถทำ request แทน user
4. เพิ่ม CSRF protection และแสดงว่า attack ไม่ได้ผล

### แบบฝึกหัดที่ 2: REST API CSRF

สร้าง REST API ที่:
1. ใช้ JWT ใน Authorization header (ไม่ต้อง CSRF)
2. ใช้ JWT ใน cookie (ต้องมี CSRF)
3. เพิ่ม CSRF protection สำหรับ case 2
4. Test ทั้งสอง cases

### แบบฝึกหัดที่ 3: SameSite Testing

```javascript
// ทดสอบ SameSite cookie:
// 1. Set cookie ด้วย SameSite=Strict
// 2. พยายาม access จาก different origin
// 3. ดูว่า cookie ถูก sent หรือไม่
// 4. เปรียบเทียบ Strict, Lax, None
```

---

## สรุป

CSRF ป้องกันได้ด้วยหลายวิธีร่วมกัน

| วิธี | เหมาะสำหรับ |
|------|------------|
| CSRF Token | Traditional forms |
| SameSite=Strict | Cookies สมัยใหม่ |
| Double Submit | Stateless CSRF protection |
| JWT in Header | REST APIs ที่ใช้ header-based auth |
| Origin Check | Additional layer |

### เมื่อไหรต้องมี CSRF Protection

```
✅ ต้องมี:
- Cookie-based authentication
- Session-based authentication
- Form submissions
- State-changing operations (POST, PUT, DELETE)

❌ ไม่จำเป็น:
- JWT ใน Authorization header
- API keys ใน header
- Public APIs (GET only)
- Read-only endpoints
```

---

**ยินดีด้วย! คุณเรียนจบ Part 31-45 แล้ว** 

### สรุปสิ่งที่เรียนในส่วนนี้

| Part | หัวข้อ |
|------|--------|
| 31 | Environment Variables & Config Management |
| 32 | MVC Architecture |
| 33 | RESTful API Design |
| 34 | API Documentation (Swagger/OpenAPI) |
| 35 | Testing ด้วย Jest |
| 36 | Unit Testing |
| 37 | Integration Testing |
| 38 | WebSocket & Socket.io |
| 39 | Caching ด้วย Redis |
| 40 | Rate Limiting |
| 41 | CORS |
| 42 | Security Headers (Helmet.js) |
| 43 | SQL Injection Prevention |
| 44 | XSS Prevention |
| 45 | CSRF Protection |

**ถัดไป**: Part 46+ - Advanced Topics (Microservices, GraphQL, Docker, etc.)
