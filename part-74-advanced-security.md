# Part 74: Advanced Security
## ขั้นตอนที่ 731-740 จาก 1000

---

## Security เป็นเรื่องสำคัญที่สุด

การรักษาความปลอดภัยของ application ต้องทำตั้งแต่ design phase ไม่ใช่แค่ add-on ทีหลัง

---

## 1. OWASP Top 10 สำหรับ Node.js

### A01: Broken Access Control

```javascript
// ปัญหา: ไม่ตรวจสอบ ownership
app.delete('/api/posts/:id', async (req, res) => {
  await Post.findByIdAndDelete(req.params.id); // ใครก็ลบได้!
});

// แก้ไข: ตรวจสอบ ownership เสมอ
app.delete('/api/posts/:id', authenticate, async (req, res) => {
  const post = await Post.findOne({
    _id: req.params.id,
    author: req.user.id  // ตรวจสอบว่าเป็น owner
  });

  if (!post) {
    return res.status(404).json({ error: 'Post not found or access denied' });
  }

  await post.deleteOne();
  res.json({ success: true });
});

// Role-based access control
function authorize(...roles) {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}

app.get('/api/admin/users', authenticate, authorize('admin'), adminController.getUsers);
```

### A02: Cryptographic Failures

```javascript
// ปัญหา: เก็บ passwords แบบ plain text หรือ hash ที่อ่อนแอ
user.password = md5(password); // อันตราย!
user.password = sha256(password); // ยังอันตราย!

// แก้ไข: ใช้ bcrypt หรือ argon2
const bcrypt = require('bcrypt');
const argon2 = require('argon2');

// bcrypt
const SALT_ROUNDS = 12;
const hashed = await bcrypt.hash(password, SALT_ROUNDS);
const isValid = await bcrypt.compare(password, hashed);

// argon2 (แนะนำกว่า)
const hashed = await argon2.hash(password, {
  type: argon2.argon2id,
  memoryCost: 2 ** 16,
  timeCost: 3,
  parallelism: 1
});
const isValid = await argon2.verify(hashed, password);

// เข้ารหัสข้อมูลสำคัญ
const crypto = require('crypto');

function encrypt(text, key) {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-gcm', Buffer.from(key, 'hex'), iv);
  
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  
  const authTag = cipher.getAuthTag();
  
  return {
    encrypted,
    iv: iv.toString('hex'),
    authTag: authTag.toString('hex')
  };
}

function decrypt(encrypted, iv, authTag, key) {
  const decipher = crypto.createDecipheriv(
    'aes-256-gcm',
    Buffer.from(key, 'hex'),
    Buffer.from(iv, 'hex')
  );
  
  decipher.setAuthTag(Buffer.from(authTag, 'hex'));
  
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  
  return decrypted;
}
```

### A03: Injection

```javascript
// SQL Injection
// ปัญหา
const query = `SELECT * FROM users WHERE email = '${email}'`;
// เมื่อ email = "admin@test.com' OR '1'='1"

// แก้ไข: ใช้ parameterized queries
const result = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);

// NoSQL Injection (MongoDB)
// ปัญหา: ไม่ validate input
User.findOne({ username: req.body.username });
// เมื่อ username = { "$gt": "" } จะ return user แรกที่เจอ

// แก้ไข: validate และ sanitize input
const mongoSanitize = require('express-mongo-sanitize');
app.use(mongoSanitize()); // ลบ $ และ . จาก request

// หรือ validate แบบ manual
function sanitizeQuery(input) {
  if (typeof input !== 'string') {
    throw new Error('Invalid input type');
  }
  return input.replace(/[${}]/g, '');
}

// XSS Prevention
const createDOMPurify = require('isomorphic-dompurify');
const sanitizeHtml = require('sanitize-html');

function sanitizeInput(html) {
  return sanitizeHtml(html, {
    allowedTags: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
    allowedAttributes: { 'a': ['href'] },
    allowedSchemes: ['https']
  });
}
```

### A05: Security Misconfiguration

```javascript
// Security Headers
const helmet = require('helmet');

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],
      styleSrc: ["'self'", "https://fonts.googleapis.com"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'"],
      fontSrc: ["'self'", "https://fonts.gstatic.com"],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"]
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  },
  noSniff: true,
  xssFilter: true,
  referrerPolicy: { policy: 'same-origin' }
}));

// ป้องกัน information leakage
app.use((err, req, res, next) => {
  // ไม่ส่ง stack trace ใน production
  const error = {
    message: err.message || 'Internal Server Error',
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
  };
  
  res.status(err.status || 500).json({ error });
});

// ซ่อน server information
app.disable('x-powered-by');
```

### A07: Identification and Authentication Failures

```javascript
// Rate limiting สำหรับ auth endpoints
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const redis = require('./redis');

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: { error: 'Too many login attempts. Please try again after 15 minutes.' },
  standardHeaders: true,
  store: new RedisStore({
    sendCommand: (...args) => redis.call(...args),
    prefix: 'rl:login:'
  })
});

// Account lockout
async function handleFailedLogin(userId) {
  const key = `failed_login:${userId}`;
  const attempts = await redis.incr(key);
  
  if (attempts === 1) {
    await redis.expire(key, 15 * 60); // Reset หลัง 15 นาที
  }
  
  if (attempts >= 5) {
    await User.findByIdAndUpdate(userId, {
      lockedUntil: new Date(Date.now() + 30 * 60 * 1000)
    });
    throw new Error('Account locked for 30 minutes');
  }
}

// Password strength validation
const zxcvbn = require('zxcvbn');

function validatePasswordStrength(password) {
  const result = zxcvbn(password);
  
  if (result.score < 3) {
    const suggestions = result.feedback.suggestions.join('. ');
    throw new Error(`Password too weak. ${suggestions}`);
  }
  
  return true;
}

// MFA (Two-Factor Authentication)
const speakeasy = require('speakeasy');
const qrcode = require('qrcode');

async function setupMFA(userId) {
  const secret = speakeasy.generateSecret({
    name: `MyApp (${userId})`,
    length: 20
  });
  
  const qrCodeUrl = await qrcode.toDataURL(secret.otpauth_url);
  
  // เก็บ secret ชั่วคราว (ยังไม่ confirm)
  await redis.set(`mfa_setup:${userId}`, secret.base32, 'EX', 600);
  
  return { secret: secret.base32, qrCode: qrCodeUrl };
}

async function verifyMFAToken(userId, token) {
  const user = await User.findById(userId).select('mfaSecret');
  
  return speakeasy.totp.verify({
    secret: user.mfaSecret,
    encoding: 'base32',
    token,
    window: 1  // อนุญาต 30 วินาที tolerance
  });
}
```

---

## 2. Security Audit Tools

### npm audit

```bash
# ตรวจสอบ vulnerabilities
npm audit

# แก้ไขอัตโนมัติ
npm audit fix

# รายงานละเอียด
npm audit --json

# ใช้ใน CI/CD
npm audit --audit-level=high
```

### OWASP Dependency Check

```bash
# ติดตั้ง
npm install -g owasp-dependency-check

# ตรวจสอบ
dependency-check --project "MyApp" --scan . --format HTML --out reports
```

### Helmet Configuration Audit

```javascript
// ตรวจสอบ security headers
const checkSecurityHeaders = async (url) => {
  const response = await fetch(url);
  const headers = response.headers;
  
  const checks = {
    'strict-transport-security': headers.get('strict-transport-security'),
    'x-content-type-options': headers.get('x-content-type-options'),
    'x-frame-options': headers.get('x-frame-options'),
    'content-security-policy': headers.get('content-security-policy'),
    'x-xss-protection': headers.get('x-xss-protection')
  };
  
  return checks;
};
```

---

## 3. Secrets Management

```javascript
// ปัญหา: เก็บ secrets ใน code
const DB_PASSWORD = 'my-super-secret'; // อันตราย!

// แก้ไข: ใช้ environment variables
require('dotenv').config();
const DB_PASSWORD = process.env.DB_PASSWORD;

// ตรวจสอบว่า required env vars มีครบ
function validateEnv() {
  const required = [
    'DATABASE_URL',
    'JWT_SECRET',
    'REDIS_URL',
    'STRIPE_SECRET_KEY'
  ];
  
  const missing = required.filter(key => !process.env[key]);
  
  if (missing.length > 0) {
    throw new Error(`Missing required environment variables: ${missing.join(', ')}`);
  }
}

validateEnv();

// Secrets Scanning
// .gitignore
// .env
// .env.*
// *.pem
// *.key
// config/secrets.json
```

---

## 4. SQL/NoSQL Injection Prevention

```javascript
// Comprehensive input validation
const { body, param, query } = require('express-validator');
const { validationResult } = require('express-validator');

const validateCreateUser = [
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 8 }).trim(),
  body('name').notEmpty().isLength({ max: 100 }).trim().escape(),
  body('age').optional().isInt({ min: 0, max: 120 }),
  
  (req, res, next) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    next();
  }
];

// Prevent prototype pollution
const deepFreeze = require('deep-freeze');

app.use((req, res, next) => {
  // Freeze request body ป้องกัน prototype pollution
  if (req.body) {
    Object.keys(req.body).forEach(key => {
      if (key === '__proto__' || key === 'constructor' || key === 'prototype') {
        delete req.body[key];
      }
    });
  }
  next();
});
```

---

## 5. Security Logging

```javascript
// audit.logger.js
const winston = require('winston');

const auditLogger = winston.createLogger({
  transports: [
    new winston.transports.File({
      filename: 'logs/audit.log',
      format: winston.format.json()
    })
  ]
});

function logSecurityEvent(event) {
  auditLogger.info({
    timestamp: new Date().toISOString(),
    ...event
  });
}

// ใช้งาน
app.post('/auth/login', async (req, res) => {
  const { email } = req.body;
  
  try {
    const user = await authenticateUser(email, req.body.password);
    
    logSecurityEvent({
      type: 'login_success',
      userId: user.id,
      email,
      ip: req.ip,
      userAgent: req.get('user-agent')
    });
    
    res.json({ token: generateToken(user) });
  } catch (error) {
    logSecurityEvent({
      type: 'login_failure',
      email,
      ip: req.ip,
      userAgent: req.get('user-agent'),
      reason: error.message
    });
    
    res.status(401).json({ error: 'Invalid credentials' });
  }
});
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
- ตั้งค่า Helmet
- เพิ่ม rate limiting
- Validate ทุก inputs

### ระดับ 2: กลาง
- Implement MFA
- Security audit logging
- Input sanitization

### ระดับ 3: ขั้นสูง
- Penetration testing
- SAST/DAST integration
- Security CI/CD pipeline

---

## สรุป

Security ต้องทำเป็น layered approach ตั้งแต่ authentication, authorization, input validation, encryption, logging ไปจนถึง infrastructure security การทำ security audit เป็นประจำและ keep dependencies updated เป็นสิ่งจำเป็น

> ขั้นตอนต่อไป: Part 75 - Interview Preparation
