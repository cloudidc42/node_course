# Part 18: Cookies และ Sessions
## ขั้นตอนที่ 18-8 จาก 1000

---

## สารบัญ
1. Cookies พื้นฐาน
2. cookie-parser
3. Sessions
4. express-session
5. Session Storage (Redis)
6. Flash Messages
7. Practical: Login System with Sessions
8. Exercise

---

## 1. Cookies พื้นฐาน

Cookie คือข้อมูลขนาดเล็กที่เก็บไว้ใน browser ของผู้ใช้ เมื่อ browser ส่ง request ไปยัง server จะแนบ cookies ไปด้วยทุกครั้ง

### ความสำคัญของ Cookies

```
Client (Browser)          Server
     |                       |
     |-- GET / ----------->  |
     |                       | 
     |<- 200 OK ------------ |
     |   Set-Cookie: name=val |
     |                       |
     | (Browser เก็บ cookie) |
     |                       |
     |-- GET /dashboard -->  |
     |   Cookie: name=val    |  <-- ส่ง cookie กลับมาเสมอ
     |                       |
```

### Cookie Properties

```javascript
// ตัวอย่าง Set-Cookie header:
// Set-Cookie: userId=123; Max-Age=3600; HttpOnly; Secure; SameSite=Strict; Path=/

// อธิบาย:
// userId=123         - ชื่อ=ค่า
// Max-Age=3600       - หมดอายุใน 3600 วินาที (1 ชั่วโมง)
// Expires=...        - วันหมดอายุแบบ absolute
// HttpOnly           - JavaScript ไม่สามารถอ่านได้ (ป้องกัน XSS)
// Secure             - ส่งได้เฉพาะ HTTPS
// SameSite=Strict    - ป้องกัน CSRF (Strict/Lax/None)
// Path=/             - cookie ใช้ได้ทุก path
// Domain=example.com - cookie ใช้ได้เฉพาะ domain นี้
```

### ข้อจำกัดของ Cookies

- **ขนาด**: สูงสุด 4KB ต่อ cookie
- **จำนวน**: ~50 cookies ต่อ domain
- **Security**: ไม่เหมาะเก็บข้อมูลสำคัญ (ต้องใช้ HttpOnly + Secure)

---

## 2. cookie-parser

```bash
npm install cookie-parser
```

### การใช้งาน

```javascript
const express = require('express');
const cookieParser = require('cookie-parser');
const app = express();

// ใช้ cookie secret สำหรับ signed cookies
const COOKIE_SECRET = process.env.COOKIE_SECRET || 'my-super-secret-key';
app.use(cookieParser(COOKIE_SECRET));

// ==============================
// Set Cookies
// ==============================
app.get('/set-cookies', (req, res) => {
  // Cookie พื้นฐาน
  res.cookie('name', 'สมชาย');
  
  // Cookie พร้อม options ครบ
  res.cookie('userId', '12345', {
    maxAge: 7 * 24 * 60 * 60 * 1000, // 7 วัน (ms)
    httpOnly: true,   // JavaScript เข้าไม่ถึง
    secure: process.env.NODE_ENV === 'production', // HTTPS only
    sameSite: 'strict' // ป้องกัน CSRF
  });
  
  // Signed cookie (ป้องกันการแก้ไข)
  res.cookie('token', 'abc123', {
    signed: true,
    httpOnly: true,
    maxAge: 3600000
  });
  
  // Cookie ชั่วคราว (หมดอายุเมื่อปิด browser)
  res.cookie('temp', 'data'); // ไม่กำหนด maxAge = session cookie
  
  res.send('Cookies ถูกตั้งค่าแล้ว');
});

// ==============================
// Read Cookies
// ==============================
app.get('/read-cookies', (req, res) => {
  // อ่าน unsigned cookies
  console.log(req.cookies);
  // { name: 'สมชาย', userId: '12345', temp: 'data' }
  
  // อ่าน signed cookies
  console.log(req.signedCookies);
  // { token: 'abc123' }
  
  // ตรวจสอบ signed cookie
  const token = req.signedCookies.token;
  if (token === false) {
    // ถ้า false แสดงว่า cookie ถูกแก้ไข
    return res.status(401).send('Cookie ไม่ถูกต้อง!');
  }
  
  res.json({
    cookies: req.cookies,
    signedCookies: req.signedCookies
  });
});

// ==============================
// Delete Cookies
// ==============================
app.get('/clear-cookies', (req, res) => {
  res.clearCookie('name');
  res.clearCookie('userId');
  res.clearCookie('token');
  
  res.send('Cookies ถูกลบแล้ว');
});
```

### Cookie vs LocalStorage vs SessionStorage

| | Cookie | LocalStorage | SessionStorage |
|--|--------|-------------|----------------|
| ส่งกับ request | ใช่ | ไม่ | ไม่ |
| ขนาด | ~4KB | ~5-10MB | ~5MB |
| หมดอายุ | กำหนดเองได้ | ไม่มี | ปิด tab |
| JavaScript | ถ้าไม่ HttpOnly | ใช่ | ใช่ |
| Server access | ใช่ | ไม่ | ไม่ |

---

## 3. Sessions

Session คือการเก็บข้อมูลผู้ใช้ฝั่ง server โดยใช้ cookie เพียงเก็บ session ID:

```
Client (Browser)          Server
     |                       |
     |-- POST /login ------>  |
     |   (email, password)   |
     |                       | ตรวจสอบ credentials
     |                       | สร้าง session: {userId: 1, role: 'admin'}
     |                       | เก็บ session ไว้ฝั่ง server
     |<- 200 OK ------------ |
     |   Set-Cookie: sid=abc  |  <- ส่งแค่ Session ID
     |                       |
     |-- GET /dashboard -->   |
     |   Cookie: sid=abc      |
     |                       | ค้นหา session จาก ID "abc"
     |                       | ได้ {userId: 1, role: 'admin'}
     |<- Dashboard ---------- |
```

### ทำไม Session ดีกว่าเก็บข้อมูลใน Cookie

```javascript
// ❌ ไม่ดี: เก็บข้อมูลทั้งหมดใน cookie
res.cookie('user', JSON.stringify({ id: 1, role: 'admin', permissions: [...] }));
// ปัญหา: 1. ขนาด จำกัด 4KB  2. ผู้ใช้แก้ไขได้  3. ข้อมูลถูก expose

// ✅ ดีกว่า: เก็บเฉพาะ session ID ใน cookie
req.session.userId = 1;
req.session.role = 'admin';
// ข้อมูลจริงเก็บไว้ฝั่ง server
```

---

## 4. express-session

```bash
npm install express-session
```

### การตั้งค่า Session

```javascript
const express = require('express');
const session = require('express-session');
const app = express();

app.use(session({
  // ==============================
  // Required options
  // ==============================
  secret: process.env.SESSION_SECRET || 'your-secret-key',
  
  // ==============================
  // Recommended options
  // ==============================
  resave: false,           // ไม่ save session ถ้าไม่มีการเปลี่ยนแปลง
  saveUninitialized: false, // ไม่ save session ที่ยังไม่ได้ใช้
  
  // ==============================
  // Cookie options
  // ==============================
  cookie: {
    maxAge: 24 * 60 * 60 * 1000, // 1 วัน
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict'
  },
  
  // ==============================
  // Session name (default: 'connect.sid')
  // ==============================
  name: 'my-session-id'
}));
```

### การใช้งาน Session

```javascript
// ==============================
// เก็บข้อมูลใน Session
// ==============================
app.post('/login', (req, res) => {
  const { email, password } = req.body;
  
  // ตรวจสอบ credentials (ตัวอย่าง)
  const user = findUserByEmail(email);
  if (!user || !checkPassword(password, user.password)) {
    return res.status(401).json({ error: 'Email หรือ Password ไม่ถูกต้อง' });
  }
  
  // เก็บข้อมูลใน session
  req.session.userId = user.id;
  req.session.userEmail = user.email;
  req.session.userRole = user.role;
  req.session.loginAt = new Date();
  
  res.json({ success: true, message: 'Login สำเร็จ' });
});

// ==============================
// อ่านข้อมูลจาก Session
// ==============================
app.get('/profile', (req, res) => {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'กรุณา login ก่อน' });
  }
  
  res.json({
    userId: req.session.userId,
    email: req.session.userEmail,
    role: req.session.userRole
  });
});

// ==============================
// ลบ Session (Logout)
// ==============================
app.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      return res.status(500).json({ error: 'เกิดข้อผิดพลาด' });
    }
    
    // ลบ cookie ด้วย
    res.clearCookie('my-session-id');
    res.json({ success: true, message: 'Logout สำเร็จ' });
  });
});

// ==============================
// Regenerate Session ID (Security)
// ==============================
app.post('/elevate', (req, res) => {
  // หลัง login ที่สำเร็จ ควร regenerate session ID
  // เพื่อป้องกัน session fixation attack
  req.session.regenerate((err) => {
    if (err) return next(err);
    
    req.session.userId = user.id;
    // ... set other session data
    
    res.json({ success: true });
  });
});
```

### Session Middleware ตรวจสอบ Authentication

```javascript
// middleware/requireAuth.js
const requireAuth = (req, res, next) => {
  if (!req.session || !req.session.userId) {
    if (req.accepts('html')) {
      return res.redirect('/login');
    }
    return res.status(401).json({
      error: 'กรุณา login ก่อน',
      redirectTo: '/login'
    });
  }
  next();
};

const requireAdmin = (req, res, next) => {
  if (!req.session || req.session.userRole !== 'admin') {
    return res.status(403).json({ error: 'ไม่มีสิทธิ์เข้าถึง' });
  }
  next();
};

// ใช้งาน
app.get('/dashboard', requireAuth, (req, res) => {
  res.render('dashboard');
});

app.get('/admin', requireAuth, requireAdmin, (req, res) => {
  res.render('admin');
});
```

---

## 5. Session Storage (Redis)

การเก็บ session ใน memory ของ server มีปัญหาเมื่อ:
- Server restart → session หาย
- หลาย server instances → session ไม่ sync

Redis แก้ปัญหาเหล่านี้:

```bash
npm install connect-redis redis
```

### Redis Session Store

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');

// สร้าง Redis client
const redisClient = createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379'
});

redisClient.connect().catch(console.error);

// ตั้งค่า Session ด้วย Redis Store
app.use(session({
  store: new RedisStore({
    client: redisClient,
    prefix: 'sess:',    // prefix ใน Redis key
    ttl: 86400          // TTL 1 วัน (วินาที)
  }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 24 * 60 * 60 * 1000
  }
}));
```

### ทางเลือกอื่น สำหรับ Session Store

```javascript
// MongoDB (connect-mongo)
const MongoStore = require('connect-mongo');
app.use(session({
  store: MongoStore.create({ mongoUrl: 'mongodb://localhost/mydb' }),
  secret: 'secret',
  resave: false,
  saveUninitialized: false
}));

// SQLite/PostgreSQL (connect-pg-simple)
const pgSession = require('connect-pg-simple')(session);
app.use(session({
  store: new pgSession({ conString: 'postgresql://localhost/mydb' }),
  secret: 'secret',
  resave: false,
  saveUninitialized: false
}));

// File system (session-file-store) - ง่าย แต่ไม่ scale
const FileStore = require('session-file-store')(session);
app.use(session({
  store: new FileStore({ path: './sessions' }),
  secret: 'secret',
  resave: false,
  saveUninitialized: false
}));
```

---

## 6. Flash Messages

Flash messages คือ messages ที่แสดงครั้งเดียวแล้วหาย (เช่น "บันทึกสำเร็จ" หรือ "เกิดข้อผิดพลาด"):

```bash
npm install connect-flash
```

```javascript
const session = require('express-session');
const flash = require('connect-flash');

app.use(session({ secret: 'secret', resave: false, saveUninitialized: false }));
app.use(flash());

// ==============================
// Middleware ส่ง flash messages ไปทุก template
// ==============================
app.use((req, res, next) => {
  res.locals.success = req.flash('success');
  res.locals.error = req.flash('error');
  res.locals.info = req.flash('info');
  res.locals.warning = req.flash('warning');
  next();
});

// ==============================
// ตั้งค่า flash message
// ==============================
app.post('/create', (req, res) => {
  // หลัง create สำเร็จ
  req.flash('success', 'สร้างข้อมูลสำเร็จ!');
  res.redirect('/list');
});

app.post('/delete/:id', (req, res) => {
  // หลัง delete
  req.flash('success', `ลบรายการ #${req.params.id} สำเร็จ`);
  res.redirect('/list');
});

app.post('/update', (req, res) => {
  // เกิดข้อผิดพลาด
  req.flash('error', 'ไม่สามารถบันทึกข้อมูลได้ กรุณาลองใหม่');
  res.redirect('/form');
});

// ==============================
// ใช้ flash ใน template (EJS)
// ==============================
```

```html
<!-- views/partials/flash.ejs -->
<% if (typeof success !== 'undefined' && success.length > 0) { %>
  <% success.forEach(msg => { %>
    <div class="alert alert-success" role="alert">
      <i class="fas fa-check-circle"></i> <%= msg %>
      <button type="button" class="close" data-dismiss="alert">&times;</button>
    </div>
  <% }) %>
<% } %>

<% if (typeof error !== 'undefined' && error.length > 0) { %>
  <% error.forEach(msg => { %>
    <div class="alert alert-danger" role="alert">
      <i class="fas fa-exclamation-circle"></i> <%= msg %>
      <button type="button" class="close" data-dismiss="alert">&times;</button>
    </div>
  <% }) %>
<% } %>

<% if (typeof info !== 'undefined' && info.length > 0) { %>
  <% info.forEach(msg => { %>
    <div class="alert alert-info" role="alert">
      <i class="fas fa-info-circle"></i> <%= msg %>
    </div>
  <% }) %>
<% } %>
```

---

## 7. Practical: Login System with Sessions

### โครงสร้างไฟล์

```
login-app/
├── views/
│   ├── partials/
│   │   └── flash.ejs
│   ├── login.ejs
│   ├── register.ejs
│   ├── dashboard.ejs
│   └── profile.ejs
├── middleware/
│   └── auth.js
├── routes/
│   ├── auth.js
│   └── protected.js
└── app.js
```

### views/login.ejs

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>เข้าสู่ระบบ</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Sarabun', sans-serif; background: #f0f2f5; min-height: 100vh; display: flex; align-items: center; justify-content: center; }
    .card { background: white; padding: 2.5rem; border-radius: 1rem; box-shadow: 0 4px 20px rgba(0,0,0,0.1); width: 100%; max-width: 400px; }
    h1 { color: #1e40af; text-align: center; margin-bottom: 2rem; }
    .form-group { margin-bottom: 1.25rem; }
    label { display: block; margin-bottom: 0.5rem; font-weight: 600; }
    input { width: 100%; padding: 0.75rem; border: 1px solid #d1d5db; border-radius: 0.5rem; font-size: 1rem; }
    input:focus { outline: none; border-color: #2563eb; }
    .btn { width: 100%; padding: 0.875rem; background: #2563eb; color: white; border: none; border-radius: 0.5rem; font-size: 1rem; font-weight: 600; cursor: pointer; margin-top: 0.5rem; }
    .btn:hover { background: #1d4ed8; }
    .alert { padding: 0.75rem 1rem; border-radius: 0.5rem; margin-bottom: 1rem; }
    .alert-error { background: #fef2f2; border: 1px solid #fecaca; color: #dc2626; }
    .links { text-align: center; margin-top: 1rem; }
    a { color: #2563eb; }
  </style>
</head>
<body>
  <div class="card">
    <h1>🔐 เข้าสู่ระบบ</h1>
    
    <% if (typeof error !== 'undefined' && error.length > 0) { %>
      <% error.forEach(msg => { %>
        <div class="alert alert-error"><%= msg %></div>
      <% }) %>
    <% } %>
    
    <form action="/auth/login" method="POST">
      <div class="form-group">
        <label for="email">อีเมล</label>
        <input type="email" id="email" name="email" 
               value="<%= typeof formData !== 'undefined' ? formData.email || '' : '' %>"
               required autofocus>
      </div>
      
      <div class="form-group">
        <label for="password">รหัสผ่าน</label>
        <input type="password" id="password" name="password" required>
      </div>
      
      <div class="form-group">
        <label>
          <input type="checkbox" name="remember" value="yes"> จดจำฉัน 30 วัน
        </label>
      </div>
      
      <button type="submit" class="btn">เข้าสู่ระบบ</button>
    </form>
    
    <div class="links">
      <p>ยังไม่มีบัญชี? <a href="/auth/register">สมัครสมาชิก</a></p>
      <p><a href="/auth/forgot-password">ลืมรหัสผ่าน?</a></p>
    </div>
  </div>
</body>
</html>
```

### routes/auth.js

```javascript
// routes/auth.js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');

// ข้อมูลผู้ใช้ (จำลอง)
const users = [
  {
    id: 1,
    name: 'สมชาย ใจดี',
    email: 'admin@example.com',
    password: bcrypt.hashSync('Admin1234', 10),
    role: 'admin',
    active: true
  },
  {
    id: 2,
    name: 'สมหญิง ใจงาม',
    email: 'user@example.com',
    password: bcrypt.hashSync('User1234', 10),
    role: 'user',
    active: true
  }
];

// GET /auth/login
router.get('/login', (req, res) => {
  if (req.session.userId) {
    return res.redirect('/dashboard');
  }
  res.render('login', { title: 'เข้าสู่ระบบ' });
});

// POST /auth/login
router.post('/login', async (req, res) => {
  const { email, password, remember } = req.body;
  
  // หา user
  const user = users.find(u => u.email === email);
  
  if (!user) {
    req.flash('error', 'Email หรือ Password ไม่ถูกต้อง');
    return res.redirect('/auth/login');
  }
  
  if (!user.active) {
    req.flash('error', 'บัญชีนี้ถูกระงับการใช้งาน');
    return res.redirect('/auth/login');
  }
  
  // ตรวจสอบ password
  const isMatch = await bcrypt.compare(password, user.password);
  if (!isMatch) {
    req.flash('error', 'Email หรือ Password ไม่ถูกต้อง');
    return res.redirect('/auth/login');
  }
  
  // Regenerate session ID เพื่อความปลอดภัย
  req.session.regenerate((err) => {
    if (err) return next(err);
    
    req.session.userId = user.id;
    req.session.userEmail = user.email;
    req.session.userName = user.name;
    req.session.userRole = user.role;
    req.session.loginAt = new Date();
    
    // ถ้าเลือก "จดจำฉัน"
    if (remember === 'yes') {
      req.session.cookie.maxAge = 30 * 24 * 60 * 60 * 1000; // 30 วัน
    }
    
    req.flash('success', `ยินดีต้อนรับ ${user.name}!`);
    res.redirect('/dashboard');
  });
});

// GET /auth/register
router.get('/register', (req, res) => {
  res.render('register', { title: 'สมัครสมาชิก' });
});

// POST /auth/register
router.post('/register', async (req, res) => {
  const { name, email, password, confirmPassword } = req.body;
  
  const errors = {};
  
  if (!name || name.trim().length < 2) errors.name = 'กรุณาระบุชื่อ';
  if (!email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) errors.email = 'อีเมลไม่ถูกต้อง';
  if (!password || password.length < 8) errors.password = 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
  if (password !== confirmPassword) errors.confirmPassword = 'รหัสผ่านไม่ตรงกัน';
  
  if (users.find(u => u.email === email)) {
    errors.email = 'Email นี้ถูกใช้งานแล้ว';
  }
  
  if (Object.keys(errors).length > 0) {
    Object.values(errors).forEach(err => req.flash('error', err));
    return res.redirect('/auth/register');
  }
  
  const hashedPassword = await bcrypt.hash(password, 10);
  const newUser = {
    id: users.length + 1,
    name: name.trim(),
    email: email.toLowerCase(),
    password: hashedPassword,
    role: 'user',
    active: true,
    createdAt: new Date()
  };
  users.push(newUser);
  
  req.flash('success', 'สมัครสมาชิกสำเร็จ! กรุณาเข้าสู่ระบบ');
  res.redirect('/auth/login');
});

// POST /auth/logout
router.post('/logout', (req, res) => {
  const userName = req.session.userName;
  
  req.session.destroy((err) => {
    if (err) {
      console.error('Session destroy error:', err);
    }
    res.clearCookie('session-id');
    res.redirect('/auth/login');
  });
});

module.exports = router;
```

### routes/protected.js

```javascript
// routes/protected.js
const express = require('express');
const router = express.Router();

// Middleware ตรวจสอบ login
const requireAuth = (req, res, next) => {
  if (!req.session.userId) {
    req.flash('error', 'กรุณาเข้าสู่ระบบก่อน');
    return res.redirect('/auth/login');
  }
  next();
};

const requireAdmin = (req, res, next) => {
  if (req.session.userRole !== 'admin') {
    return res.status(403).render('error', {
      title: 'ไม่มีสิทธิ์',
      message: 'คุณไม่มีสิทธิ์เข้าถึงหน้านี้'
    });
  }
  next();
};

// Dashboard - ต้อง login
router.get('/dashboard', requireAuth, (req, res) => {
  res.render('dashboard', {
    title: 'Dashboard',
    user: {
      id: req.session.userId,
      name: req.session.userName,
      email: req.session.userEmail,
      role: req.session.userRole,
      loginAt: req.session.loginAt
    }
  });
});

// Profile
router.get('/profile', requireAuth, (req, res) => {
  res.render('profile', {
    title: 'โปรไฟล์',
    user: {
      id: req.session.userId,
      name: req.session.userName,
      email: req.session.userEmail
    }
  });
});

// Admin Panel - ต้องเป็น admin
router.get('/admin', requireAuth, requireAdmin, (req, res) => {
  res.render('admin', {
    title: 'Admin Panel'
  });
});

module.exports = router;
```

### app.js

```javascript
// app.js
const express = require('express');
const session = require('express-session');
const flash = require('connect-flash');
const expressLayouts = require('express-ejs-layouts');
const path = require('path');
const app = express();

app.set('view engine', 'ejs');
app.set('views', path.join(__dirname, 'views'));
app.use(expressLayouts);
app.set('layout', 'layouts/main');

app.use(express.urlencoded({ extended: true }));
app.use(express.static('public'));

// Session
app.use(session({
  secret: process.env.SESSION_SECRET || 'dev-secret-key',
  name: 'session-id',
  resave: false,
  saveUninitialized: false,
  cookie: {
    maxAge: 24 * 60 * 60 * 1000,
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production'
  }
}));

// Flash messages
app.use(flash());

// Global variables
app.use((req, res, next) => {
  res.locals.success = req.flash('success');
  res.locals.error = req.flash('error');
  res.locals.info = req.flash('info');
  res.locals.currentUser = req.session.userId ? {
    id: req.session.userId,
    name: req.session.userName,
    role: req.session.userRole
  } : null;
  next();
});

// Routes
app.get('/', (req, res) => res.redirect('/dashboard'));
app.use('/auth', require('./routes/auth'));
app.use('/', require('./routes/protected'));

app.listen(3000, () => {
  console.log('Login System: http://localhost:3000/auth/login');
  console.log('Test accounts:');
  console.log('  Admin: admin@example.com / Admin1234');
  console.log('  User:  user@example.com  / User1234');
});
```

---

## 8. Exercise

### Exercise 1: Shopping Cart with Session

สร้าง Shopping Cart ที่ใช้ session เก็บข้อมูล:
- เพิ่มสินค้าใน cart
- ลบสินค้าออก
- เปลี่ยนจำนวน
- ล้าง cart
- แสดง total price

### Exercise 2: Remember Me Feature

ปรับปรุง login system ให้รองรับ "Remember Me":
- ถ้าไม่เลือก Remember Me: session หมดเมื่อปิด browser
- ถ้าเลือก Remember Me: จำไว้ 30 วัน
- Track last login time และ device

### Exercise 3: Session Security

เพิ่ม security features:
- ตรวจจับการล็อกอินจาก IP ต่างกัน
- Limit failed login attempts (lockout หลัง 5 ครั้ง)
- Session timeout เมื่อ idle 30 นาที
- แสดง active sessions ที่ login อยู่

---

## สรุป Part 18

ในบทนี้เราได้เรียนรู้:

1. **Cookies** - การเก็บข้อมูลใน browser, properties, security
2. **cookie-parser** - อ่าน/เขียน/ลบ cookies รวม signed cookies
3. **Sessions** - concept และข้อดีเทียบกับ cookies
4. **express-session** - ตั้งค่าและใช้งาน session
5. **Redis** - Session store สำหรับ production
6. **Flash Messages** - messages ชั่วคราวสำหรับ feedback
7. **Practical** - Login system ที่สมบูรณ์

---

*Part 18 | Node.js/Express.js Course | ขั้นตอนที่ 18 จาก 1000*
