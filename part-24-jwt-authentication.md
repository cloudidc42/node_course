# Part 24: JWT Authentication

> ขั้นตอนที่ 24-30 จาก 1000 — JSON Web Tokens สำหรับ Stateless Authentication

---

## สารบัญ

1. [JWT คืออะไร](#jwt-คืออะไร)
2. [สร้างและ Verify Tokens](#สร้างและ-verify-tokens)
3. [Access Tokens vs Refresh Tokens](#access-tokens-vs-refresh-tokens)
4. [JWT Middleware](#jwt-middleware)
5. [Blacklisting Tokens](#blacklisting-tokens)
6. [Best Practices](#best-practices)
7. [Practical: JWT API Authentication](#practical-jwt-api-authentication)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## JWT คืออะไร

JWT (JSON Web Token) เป็น open standard (RFC 7519) สำหรับส่งข้อมูลระหว่างฝ่ายต่างๆ อย่างปลอดภัยในรูปแบบ JSON

### โครงสร้างของ JWT

```
header.payload.signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Header** — algorithm และ type:
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload** — claims (ข้อมูล):
```json
{
  "sub": "user123",
  "name": "สมชาย ใจดี",
  "role": "admin",
  "iat": 1700000000,
  "exp": 1700003600
}
```

**Signature** — ลายเซ็นดิจิทัล:
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### JWT vs Session

| | JWT | Session |
|---|-----|---------|
| Storage | Client-side | Server-side |
| Scalability | ดี (stateless) | ต้องการ shared store |
| Revoke ทันที | ยาก | ง่าย |
| Payload size | ใหญ่กว่า | เล็ก (แค่ session ID) |
| Security | Signature verification | Session store |

---

## สร้างและ Verify Tokens

### ติดตั้ง

```bash
npm install jsonwebtoken
```

### Token Utils

```javascript
// utils/jwt.js
const jwt = require('jsonwebtoken');

const ACCESS_TOKEN_SECRET = process.env.JWT_ACCESS_SECRET;
const REFRESH_TOKEN_SECRET = process.env.JWT_REFRESH_SECRET;

if (!ACCESS_TOKEN_SECRET || !REFRESH_TOKEN_SECRET) {
  throw new Error('JWT secrets ไม่ได้ตั้งค่า');
}

/**
 * สร้าง Access Token (อายุสั้น)
 */
exports.generateAccessToken = (payload) => {
  return jwt.sign(payload, ACCESS_TOKEN_SECRET, {
    expiresIn: process.env.JWT_ACCESS_EXPIRES || '15m',
    issuer: 'myapp.com',
    audience: 'myapp-users',
  });
};

/**
 * สร้าง Refresh Token (อายุยาว)
 */
exports.generateRefreshToken = (payload) => {
  return jwt.sign(payload, REFRESH_TOKEN_SECRET, {
    expiresIn: process.env.JWT_REFRESH_EXPIRES || '7d',
    issuer: 'myapp.com',
    audience: 'myapp-users',
  });
};

/**
 * Verify Access Token
 */
exports.verifyAccessToken = (token) => {
  return jwt.verify(token, ACCESS_TOKEN_SECRET, {
    issuer: 'myapp.com',
    audience: 'myapp-users',
  });
};

/**
 * Verify Refresh Token
 */
exports.verifyRefreshToken = (token) => {
  return jwt.verify(token, REFRESH_TOKEN_SECRET, {
    issuer: 'myapp.com',
    audience: 'myapp-users',
  });
};

/**
 * Decode token โดยไม่ verify (ใช้ดู payload เท่านั้น)
 */
exports.decodeToken = (token) => {
  return jwt.decode(token, { complete: true });
};

// ตัวอย่างการใช้งาน
const example = () => {
  const payload = {
    sub: 'user123',
    email: 'user@example.com',
    role: 'user',
  };
  
  const accessToken = exports.generateAccessToken(payload);
  const refreshToken = exports.generateRefreshToken({ sub: payload.sub });
  
  console.log('Access Token:', accessToken);
  console.log('Refresh Token:', refreshToken);
  
  // Verify
  try {
    const decoded = exports.verifyAccessToken(accessToken);
    console.log('Decoded:', decoded);
    // { sub: 'user123', email: 'user@example.com', role: 'user', iat: ..., exp: ..., iss: '...', aud: '...' }
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      console.log('Token หมดอายุแล้ว');
    } else if (error.name === 'JsonWebTokenError') {
      console.log('Token ไม่ถูกต้อง');
    }
  }
};
```

---

## Access Tokens vs Refresh Tokens

### แนวคิด

```
Access Token:
- อายุสั้น (15 นาที - 1 ชั่วโมง)
- ใช้ authenticate requests
- ถ้า expire ต้องขอ token ใหม่

Refresh Token:
- อายุยาว (7-30 วัน)
- ใช้ขอ Access Token ใหม่
- เก็บอย่างปลอดภัย (httpOnly cookie หรือ secure storage)
```

### Refresh Token Flow

```
1. Login → ได้ access token + refresh token
2. ส่ง access token กับทุก request
3. Access token expire → ส่ง refresh token ไป /auth/refresh
4. Server verify refresh token → ออก access token ใหม่
5. ถ้า refresh token expire → ต้อง login ใหม่
```

### Auth Controller พร้อม Refresh Tokens

```javascript
// controllers/authController.js
const User = require('../models/User');
const RefreshToken = require('../models/RefreshToken');
const { hashPassword, comparePassword } = require('../utils/password');
const { 
  generateAccessToken, 
  generateRefreshToken, 
  verifyRefreshToken 
} = require('../utils/jwt');

// Helper: ส่ง tokens กลับ
const sendTokens = (res, user) => {
  const payload = {
    sub: user._id.toString(),
    email: user.email,
    role: user.role,
  };
  
  const accessToken = generateAccessToken(payload);
  const refreshToken = generateRefreshToken({ sub: payload.sub });
  
  // เก็บ refresh token ใน httpOnly cookie
  res.cookie('refreshToken', refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 7 * 24 * 60 * 60 * 1000,  // 7 วัน
    path: '/auth/refresh',              // เฉพาะ endpoint นี้เท่านั้น
  });
  
  return { accessToken, refreshToken };
};

// Login
exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email }).select('+password');
    if (!user || !(await comparePassword(password, user.password))) {
      return res.status(401).json({
        success: false,
        message: 'Email หรือรหัสผ่านไม่ถูกต้อง',
      });
    }
    
    if (!user.isActive) {
      return res.status(403).json({
        success: false,
        message: 'บัญชีถูกระงับ',
      });
    }
    
    // Update last login
    user.lastLoginAt = new Date();
    await user.save({ validateBeforeSave: false });
    
    const { accessToken } = sendTokens(res, user);
    
    res.json({
      success: true,
      accessToken,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role,
      },
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Refresh Access Token
exports.refresh = async (req, res) => {
  try {
    // รับ refresh token จาก cookie หรือ body
    const refreshToken = req.cookies.refreshToken || req.body.refreshToken;
    
    if (!refreshToken) {
      return res.status(401).json({
        success: false,
        message: 'ไม่มี refresh token',
      });
    }
    
    // Verify token
    let decoded;
    try {
      decoded = verifyRefreshToken(refreshToken);
    } catch (error) {
      return res.status(401).json({
        success: false,
        message: 'Refresh token ไม่ถูกต้องหรือหมดอายุ',
      });
    }
    
    // ตรวจสอบ token ไม่ได้ถูก blacklist
    const isBlacklisted = await RefreshToken.findOne({
      token: refreshToken,
      revoked: true,
    });
    
    if (isBlacklisted) {
      return res.status(401).json({
        success: false,
        message: 'Token ถูกยกเลิกแล้ว',
      });
    }
    
    // หา user
    const user = await User.findById(decoded.sub);
    if (!user || !user.isActive) {
      return res.status(401).json({
        success: false,
        message: 'ไม่พบผู้ใช้งาน',
      });
    }
    
    // Rotate: revoke old token, issue new one
    await RefreshToken.findOneAndUpdate(
      { token: refreshToken },
      { revoked: true, revokedAt: new Date() },
      { upsert: true }
    );
    
    const { accessToken } = sendTokens(res, user);
    
    res.json({ success: true, accessToken });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Logout
exports.logout = async (req, res) => {
  try {
    const refreshToken = req.cookies.refreshToken;
    
    if (refreshToken) {
      // Blacklist refresh token
      await RefreshToken.findOneAndUpdate(
        { token: refreshToken },
        { revoked: true, revokedAt: new Date() },
        { upsert: true }
      );
    }
    
    res.clearCookie('refreshToken', { path: '/auth/refresh' });
    
    res.json({ success: true, message: 'ออกจากระบบเรียบร้อยแล้ว' });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

---

## JWT Middleware

```javascript
// middleware/auth.js
const { verifyAccessToken } = require('../utils/jwt');
const User = require('../models/User');

/**
 * Verify JWT Access Token
 */
exports.authenticate = async (req, res, next) => {
  try {
    // ดึง token จาก Authorization header
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({
        success: false,
        message: 'กรุณาส่ง Authorization header',
      });
    }
    
    const token = authHeader.split(' ')[1];
    
    // Verify token
    let decoded;
    try {
      decoded = verifyAccessToken(token);
    } catch (error) {
      if (error.name === 'TokenExpiredError') {
        return res.status(401).json({
          success: false,
          message: 'Token หมดอายุแล้ว',
          code: 'TOKEN_EXPIRED',
        });
      }
      return res.status(401).json({
        success: false,
        message: 'Token ไม่ถูกต้อง',
        code: 'INVALID_TOKEN',
      });
    }
    
    // หา user จาก database (ตรวจสอบว่ายังมีอยู่และ active)
    const user = await User.findById(decoded.sub).select('-password');
    
    if (!user || !user.isActive) {
      return res.status(401).json({
        success: false,
        message: 'ไม่พบผู้ใช้งานหรือบัญชีถูกระงับ',
      });
    }
    
    // ตรวจสอบว่า password ไม่ได้เปลี่ยนหลัง token ถูกออก
    if (user.passwordChangedAt) {
      const passwordChangedTimestamp = parseInt(
        user.passwordChangedAt.getTime() / 1000,
        10
      );
      if (decoded.iat < passwordChangedTimestamp) {
        return res.status(401).json({
          success: false,
          message: 'รหัสผ่านถูกเปลี่ยนแล้ว กรุณาเข้าสู่ระบบใหม่',
        });
      }
    }
    
    req.user = user;
    req.tokenPayload = decoded;
    next();
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

/**
 * Optional authentication — ดึง user ถ้ามี token แต่ไม่ error ถ้าไม่มี
 */
exports.optionalAuth = async (req, res, next) => {
  try {
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return next();
    }
    
    const token = authHeader.split(' ')[1];
    
    try {
      const decoded = verifyAccessToken(token);
      const user = await User.findById(decoded.sub).select('-password');
      if (user && user.isActive) {
        req.user = user;
      }
    } catch {
      // ไม่ทำอะไรถ้า token ไม่ถูกต้อง
    }
    
    next();
  } catch (error) {
    next(error);
  }
};

/**
 * Require specific roles
 */
exports.requireRole = (...roles) => {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({
        success: false,
        message: 'ไม่ได้รับการยืนยันตัวตน',
      });
    }
    
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        message: `ต้องการสิทธิ์: ${roles.join(', ')}`,
      });
    }
    
    next();
  };
};

/**
 * Require resource ownership
 */
exports.requireOwnership = (getResourceUserId) => {
  return async (req, res, next) => {
    try {
      const resourceUserId = await getResourceUserId(req);
      
      if (
        req.user.role === 'admin' ||
        resourceUserId.toString() === req.user._id.toString()
      ) {
        return next();
      }
      
      res.status(403).json({
        success: false,
        message: 'คุณไม่มีสิทธิ์จัดการ resource นี้',
      });
    } catch (error) {
      next(error);
    }
  };
};
```

### การใช้ Middleware

```javascript
// routes/posts.js
const express = require('express');
const router = express.Router();
const { authenticate, requireRole, requireOwnership } = require('../middleware/auth');
const Post = require('../models/Post');

// Public routes
router.get('/', postController.getPosts);
router.get('/:id', postController.getPost);

// Auth required
router.post('/', authenticate, postController.createPost);

// Owner or admin
router.put(
  '/:id',
  authenticate,
  requireOwnership(async (req) => {
    const post = await Post.findById(req.params.id);
    return post?.author;
  }),
  postController.updatePost
);

// Admin only
router.delete('/:id', authenticate, requireRole('admin'), postController.deletePost);
```

---

## Blacklisting Tokens

### วิธีการ Blacklist

1. **Database blacklist** — เก็บ revoked tokens ใน database
2. **Redis blacklist** — เก็บใน Redis (เร็วกว่า)
3. **Token versioning** — เก็บ token version ใน user record

### Redis Blacklist

```javascript
// utils/tokenBlacklist.js
const redis = require('redis');

const client = redis.createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379',
});

client.on('error', (err) => console.error('Redis error:', err));
client.connect();

/**
 * เพิ่ม token เข้า blacklist
 * @param {string} token - JWT token
 * @param {number} expiresIn - วินาทีที่ token จะหมดอายุ
 */
exports.blacklistToken = async (token, expiresIn) => {
  const key = `blacklist:${token}`;
  await client.set(key, '1', { EX: expiresIn });
};

/**
 * ตรวจสอบว่า token อยู่ใน blacklist หรือไม่
 */
exports.isTokenBlacklisted = async (token) => {
  const key = `blacklist:${token}`;
  const result = await client.get(key);
  return result !== null;
};
```

```javascript
// middleware/auth.js (updated)
const { isTokenBlacklisted } = require('../utils/tokenBlacklist');

exports.authenticate = async (req, res, next) => {
  // ... extract token ...
  
  // ตรวจสอบ blacklist
  if (await isTokenBlacklisted(token)) {
    return res.status(401).json({
      success: false,
      message: 'Token ถูกยกเลิกแล้ว กรุณาเข้าสู่ระบบใหม่',
    });
  }
  
  // ... rest of verification ...
};
```

### Token Versioning (แนะนำ)

```javascript
// models/User.js — เพิ่ม tokenVersion
const userSchema = new mongoose.Schema({
  // ... other fields ...
  tokenVersion: {
    type: Number,
    default: 0,
  },
});

// Method สำหรับ invalidate tokens ทั้งหมด
userSchema.methods.incrementTokenVersion = function () {
  this.tokenVersion += 1;
  return this.save({ validateBeforeSave: false });
};
```

```javascript
// ใน generateAccessToken — ใส่ version เข้าไปด้วย
const generateAccessToken = (user) => {
  return jwt.sign(
    {
      sub: user._id.toString(),
      email: user.email,
      role: user.role,
      tokenVersion: user.tokenVersion,  // เพิ่ม version
    },
    ACCESS_TOKEN_SECRET,
    { expiresIn: '15m' }
  );
};

// ใน middleware — ตรวจสอบ version
exports.authenticate = async (req, res, next) => {
  // ... verify token ...
  
  const user = await User.findById(decoded.sub);
  
  // ตรวจสอบ token version
  if (decoded.tokenVersion !== user.tokenVersion) {
    return res.status(401).json({
      success: false,
      message: 'Token ถูกยกเลิกแล้ว',
    });
  }
  
  // ...
};

// Logout ทุก device
exports.logoutAll = async (req, res) => {
  await req.user.incrementTokenVersion();
  res.json({ success: true, message: 'ออกจากระบบทุก device แล้ว' });
};
```

---

## Best Practices

### 1. ใช้ Secret ที่แข็งแกร่ง

```bash
# สร้าง secret แบบ random
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

```env
JWT_ACCESS_SECRET=a3f5c8d9e2b4a7f1c3e6d8b0a2c4e7f9a1b3d5e7f9a0b2c4d6e8f0a2b4c6d8e0
JWT_REFRESH_SECRET=b4e6d8f0a2c4e6d8f0a2c4e6d8f0a2c4e6d8f0a2c4e6d8f0a2c4e6d8f0a2c4e6d8
```

### 2. เก็บ Token ใน Client อย่างปลอดภัย

```javascript
// BAD: เก็บใน localStorage (ถูก XSS ขโมยได้)
localStorage.setItem('token', accessToken);

// BETTER: Memory (ลบเมื่อ refresh page)
let accessToken = null;

// BEST: Access token ใน memory + Refresh token ใน httpOnly cookie
// ใน client
const { accessToken } = await fetch('/auth/login', {
  method: 'POST',
  credentials: 'include',  // ส่ง cookie ด้วย
  body: JSON.stringify({ email, password }),
}).then((r) => r.json());

// เก็บ access token ใน memory เท่านั้น
let memoryToken = accessToken;
```

### 3. Token Rotation

```javascript
// ทุกครั้งที่ refresh — ออก refresh token ใหม่และยกเลิกของเดิม
exports.refresh = async (req, res) => {
  const oldRefreshToken = req.cookies.refreshToken;
  
  // Verify old token
  const decoded = verifyRefreshToken(oldRefreshToken);
  
  // Revoke old
  await revokeToken(oldRefreshToken);
  
  // Issue new
  const user = await User.findById(decoded.sub);
  sendTokens(res, user);  // ออก tokens ใหม่ทั้งคู่
};
```

### 4. Payload ขนาดเล็ก

```javascript
// BAD: ใส่ข้อมูลมากเกินไป
const payload = {
  sub: userId,
  email: user.email,
  name: user.name,
  address: user.address,  // ไม่จำเป็น
  permissions: allPermissions,  // อาจใหญ่มาก
};

// GOOD: แค่ข้อมูลที่จำเป็น
const payload = {
  sub: userId,
  role: user.role,  // ดึง permissions จาก role ได้
};
```

---

## Practical: JWT API Authentication

### โครงสร้าง

```
jwt-auth-api/
├── config/
│   └── database.js
├── models/
│   ├── User.js
│   └── RefreshToken.js
├── controllers/
│   └── authController.js
├── middleware/
│   └── auth.js
├── routes/
│   ├── auth.js
│   └── users.js
├── utils/
│   ├── jwt.js
│   └── password.js
└── app.js
```

```javascript
// app.js
require('dotenv').config();
const express = require('express');
const cookieParser = require('cookie-parser');
const helmet = require('helmet');
const cors = require('cors');
const connectDB = require('./config/database');

const app = express();

connectDB();

// Security middleware
app.use(helmet());
app.use(cors({
  origin: process.env.CLIENT_URL || 'http://localhost:3001',
  credentials: true,  // อนุญาต cookies
}));
app.use(cookieParser());
app.use(express.json());

// Routes
app.use('/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/posts', require('./routes/posts'));

// Error handler
app.use((err, req, res, next) => {
  console.error(err);
  res.status(err.status || 500).json({
    success: false,
    message: err.message || 'เกิดข้อผิดพลาดภายในระบบ',
  });
});

app.listen(3000, () => console.log('Server started on port 3000'));
```

---

## แบบฝึกหัด

### Exercise 1: Refresh Token Rotation

1. สร้าง RefreshToken model ใน MongoDB
2. implement token rotation (เมื่อ refresh → ออก token ใหม่ + revoke เก่า)
3. จัดการ reuse detection (ถ้า revoked token ถูกใช้ → revoke ทั้ง family)

### Exercise 2: Device Management

1. เก็บ device info ใน refresh token (user agent, IP, device name)
2. สร้าง endpoint `/auth/sessions` ดู active sessions
3. สร้าง endpoint `/auth/sessions/:id` สำหรับ revoke session

### Exercise 3: Permission System

1. สร้าง Permission middleware ที่ละเอียดกว่า role
2. ตัวอย่าง: `posts:create`, `posts:update:own`, `users:read`
3. เก็บ permissions ใน JWT payload

---

## สรุป

- **JWT** — stateless authentication token
- **Access Token** อายุสั้น + **Refresh Token** อายุยาว
- **Blacklisting** ด้วย Redis หรือ Token Versioning
- **httpOnly Cookie** สำหรับ refresh token
- **Token Rotation** ป้องกัน token theft

> **บทถัดไป:** Part 25 — Security and Passwords
