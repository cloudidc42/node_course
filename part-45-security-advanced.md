# Part 45 | ขั้นตอนที่ 801-820 จาก 1000

## Advanced Security - ความปลอดภัยขั้นสูง

---

## สารบัญ

1. [OWASP Top 10](#owasp-top-10)
2. [Injection Attacks](#injection-attacks)
3. [Broken Authentication](#broken-authentication)
4. [Security Headers](#security-headers)
5. [CSRF Protection](#csrf-protection)
6. [Rate Limiting Advanced](#rate-limiting-advanced)
7. [Input Validation](#input-validation)
8. [Dependency Security](#dependency-security)
9. [Penetration Testing](#penetration-testing)
10. [แบบฝึกหัด](#แบบฝึกหัด)

---

## OWASP Top 10

### ขั้นตอนที่ 801: ทำความเข้าใจ OWASP Top 10

```
OWASP Top 10 (2021):
  A01: Broken Access Control
  A02: Cryptographic Failures
  A03: Injection
  A04: Insecure Design
  A05: Security Misconfiguration
  A06: Vulnerable and Outdated Components
  A07: Identification and Authentication Failures
  A08: Software and Data Integrity Failures
  A09: Security Logging and Monitoring Failures
  A10: Server-Side Request Forgery (SSRF)
```

### ขั้นตอนที่ 802: Security Checklist

```javascript
// security-checklist.js - ตรวจสอบ configuration ที่จำเป็น
const securityChecklist = {
  // HTTP Headers
  headers: [
    'X-Content-Type-Options',
    'X-Frame-Options',
    'Strict-Transport-Security',
    'Content-Security-Policy',
    'X-XSS-Protection',
    'Referrer-Policy',
  ],

  // Authentication
  auth: [
    'JWT secret is strong and unique',
    'Passwords are hashed with bcrypt/argon2',
    'Login rate limiting is enabled',
    'Account lockout policy exists',
    'Token expiry is set',
    'Refresh token rotation is implemented',
  ],

  // Input Validation
  validation: [
    'All input is validated and sanitized',
    'SQL queries use parameterized statements',
    'File upload types are restricted',
    'File size limits are enforced',
  ],

  // Data Protection
  data: [
    'Sensitive data is encrypted at rest',
    'HTTPS is enforced',
    'PII is handled according to regulations',
    'Passwords are never logged',
  ],

  // Dependencies
  deps: [
    'No known vulnerable dependencies',
    'Regular security updates',
    'Dependency audit runs in CI/CD',
  ],
};

module.exports = securityChecklist;
```

---

## Injection Attacks

### ขั้นตอนที่ 803: ป้องกัน NoSQL Injection

```javascript
// ❌ Vulnerable to NoSQL Injection
app.post('/login', async (req, res) => {
  const { email, password } = req.body;

  // อันตราย! ถ้า email = { "$gt": "" }
  const user = await User.findOne({ email });
  // ...
});

// ✅ Protected
const express = require('express');
const mongoSanitize = require('express-mongo-sanitize');
const { body, validationResult } = require('express-validator');

app.use(mongoSanitize({
  replaceWith: '_',  // แทนที่ $ และ . ด้วย _
  onSanitize: ({ req, key }) => {
    console.warn(`NoSQL injection attempt on field: ${key}`);
    // log security event
  },
}));

// Validate input types
app.post('/login',
  body('email').isEmail().normalizeEmail(),
  body('password').isString().isLength({ min: 6, max: 100 }),
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    const { email, password } = req.body;
    // email และ password เป็น string เสมอ
    const user = await User.findOne({ email: email.toLowerCase() });
    // ...
  }
);
```

### ขั้นตอนที่ 804: ป้องกัน SQL Injection (ถ้าใช้ SQL)

```javascript
// ❌ Vulnerable SQL
const getUserById = (id) => {
  return db.query(`SELECT * FROM users WHERE id = ${id}`);
};

// ✅ Parameterized Queries
const getUserById = (id) => {
  return db.query('SELECT * FROM users WHERE id = $1', [id]);
};

// ✅ Using ORM (Sequelize)
const user = await User.findByPk(id); // safe

// ✅ Using raw query safely
const users = await User.findAll({
  where: {
    email: { [Op.like]: `%${searchTerm}%` },  // Sequelize handles escaping
  },
});
```

### ขั้นตอนที่ 805: XSS Prevention

```javascript
// dependencies
// npm install dompurify jsdom helmet

const createDOMPurify = require('dompurify');
const { JSDOM } = require('jsdom');
const window = new JSDOM('').window;
const DOMPurify = createDOMPurify(window);

// Sanitize HTML content
function sanitizeHtml(dirty) {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'ul', 'ol', 'li', 'a'],
    ALLOWED_ATTR: ['href', 'title'],
    FORBID_TAGS: ['script', 'object', 'embed', 'iframe'],
    FORBID_ATTR: ['onerror', 'onload', 'onclick'],
  });
}

// Example usage
app.post('/posts', async (req, res) => {
  const { title, content } = req.body;

  const sanitizedContent = sanitizeHtml(content);

  const post = await Post.create({
    title: validator.escape(title),  // Escape special chars
    content: sanitizedContent,
  });

  res.json({ post });
});
```

---

## Broken Authentication

### ขั้นตอนที่ 806: Secure Authentication

```javascript
// services/authService.js
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const crypto = require('crypto');

const BCRYPT_ROUNDS = 12;
const JWT_ACCESS_EXPIRY = '15m';
const JWT_REFRESH_EXPIRY = '7d';
const MAX_LOGIN_ATTEMPTS = 5;
const LOCKOUT_DURATION = 15 * 60 * 1000; // 15 minutes

class AuthService {
  async register(userData) {
    const { email, password, username } = userData;

    // ตรวจสอบ password strength
    if (!isStrongPassword(password)) {
      throw new Error('Password does not meet requirements');
    }

    // Hash password ด้วย bcrypt
    const hashedPassword = await bcrypt.hash(password, BCRYPT_ROUNDS);

    const user = await User.create({
      email: email.toLowerCase(),
      password: hashedPassword,
      username,
      emailVerified: false,
      emailVerifyToken: crypto.randomBytes(32).toString('hex'),
    });

    // ส่ง verification email
    await this.sendVerificationEmail(user);

    return this.generateTokens(user);
  }

  async login(email, password, ipAddress) {
    const user = await User.findOne({ email: email.toLowerCase() });

    // ตรวจสอบว่า account ถูก lock หรือไม่
    if (user?.lockUntil && user.lockUntil > Date.now()) {
      const minutesLeft = Math.ceil((user.lockUntil - Date.now()) / 60000);
      throw new Error(`บัญชีถูกล็อค กรุณารออีก ${minutesLeft} นาที`);
    }

    // generic error message เพื่อไม่เปิดเผยว่า email มีอยู่หรือไม่
    if (!user) {
      throw new Error('อีเมลหรือรหัสผ่านไม่ถูกต้อง');
    }

    const isPasswordValid = await bcrypt.compare(password, user.password);

    if (!isPasswordValid) {
      // เพิ่ม login attempt
      user.loginAttempts = (user.loginAttempts || 0) + 1;

      if (user.loginAttempts >= MAX_LOGIN_ATTEMPTS) {
        user.lockUntil = Date.now() + LOCKOUT_DURATION;
        await user.save();
        throw new Error('บัญชีถูกล็อคชั่วคราว เนื่องจากล็อกอินผิดหลายครั้ง');
      }

      await user.save();
      throw new Error('อีเมลหรือรหัสผ่านไม่ถูกต้อง');
    }

    // Reset login attempts on success
    user.loginAttempts = 0;
    user.lockUntil = null;
    user.lastLoginAt = new Date();
    user.lastLoginIp = ipAddress;
    await user.save();

    // Log security event
    await SecurityLog.create({
      userId: user._id,
      event: 'LOGIN_SUCCESS',
      ipAddress,
    });

    return this.generateTokens(user);
  }

  generateTokens(user) {
    const payload = { id: user._id, role: user.role };

    const accessToken = jwt.sign(payload, process.env.JWT_SECRET, {
      expiresIn: JWT_ACCESS_EXPIRY,
      issuer: 'myapp.com',
      audience: 'myapp-users',
    });

    const refreshToken = jwt.sign(
      { id: user._id },
      process.env.JWT_REFRESH_SECRET,
      { expiresIn: JWT_REFRESH_EXPIRY }
    );

    return { accessToken, refreshToken };
  }

  async refreshAccessToken(refreshToken) {
    try {
      const payload = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET);

      // ตรวจสอบว่า refresh token ยังใช้ได้ใน database
      const storedToken = await RefreshToken.findOne({
        token: refreshToken,
        userId: payload.id,
        expiresAt: { $gt: new Date() },
        revoked: false,
      });

      if (!storedToken) {
        throw new Error('Invalid refresh token');
      }

      const user = await User.findById(payload.id);

      // Rotate refresh token
      storedToken.revoked = true;
      await storedToken.save();

      return this.generateTokens(user);
    } catch (err) {
      throw new Error('Invalid or expired refresh token');
    }
  }
}
```

---

## Security Headers

### ขั้นตอนที่ 807: Helmet.js Security Headers

```javascript
const helmet = require('helmet');

app.use(helmet({
  // Prevent clickjacking
  frameguard: { action: 'deny' },

  // Prevent MIME type sniffing
  noSniff: true,

  // Force HTTPS (HSTS)
  hsts: {
    maxAge: 31536000,  // 1 year in seconds
    includeSubDomains: true,
    preload: true,
  },

  // Content Security Policy
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'strict-dynamic'"],
      styleSrc: ["'self'", "'unsafe-inline'", 'https://fonts.googleapis.com'],
      fontSrc: ["'self'", 'https://fonts.gstatic.com'],
      imgSrc: ["'self'", 'data:', 'https://res.cloudinary.com'],
      connectSrc: ["'self'", 'https://api.myapp.com'],
      frameSrc: ["'none'"],
      objectSrc: ["'none'"],
      upgradeInsecureRequests: [],
    },
  },

  // Referrer Policy
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },

  // Permissions Policy
  permissionsPolicy: {
    features: {
      geolocation: [],  // ปิดกล้อง
      microphone: [],   // ปิดไมค์
      camera: [],       // ปิดกล้อง
    },
  },
}));

// Custom headers
app.use((req, res, next) => {
  // ซ่อน technology fingerprint
  res.removeHeader('X-Powered-By');
  res.setHeader('Server', 'MyApp');

  // Cross-Origin-Opener-Policy
  res.setHeader('Cross-Origin-Opener-Policy', 'same-origin');

  // Cross-Origin-Embedder-Policy
  res.setHeader('Cross-Origin-Embedder-Policy', 'require-corp');

  // Cross-Origin-Resource-Policy
  res.setHeader('Cross-Origin-Resource-Policy', 'same-origin');

  next();
});
```

---

## CSRF Protection

### ขั้นตอนที่ 808: CSRF Protection

```javascript
// npm install csurf cookie-parser
const csrf = require('csurf');
const cookieParser = require('cookie-parser');

app.use(cookieParser());

// CSRF protection สำหรับ web forms
const csrfProtection = csrf({
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
  },
});

// Apply CSRF to state-changing routes
app.post('/api/posts', csrfProtection, (req, res) => {
  // CSRF token ถูก validate แล้ว
  res.json({ success: true });
});

// Provide CSRF token to frontend
app.get('/api/csrf-token', csrfProtection, (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});

// Frontend เอา token จาก cookie และส่งใน header
// axios.defaults.headers.common['X-CSRF-Token'] = csrfToken;

// สำหรับ REST API ที่ใช้ JWT
// CSRF ป้องกันด้วย:
// 1. SameSite cookie attribute
// 2. Custom request headers (X-Requested-With)
// 3. Double submit cookie pattern

// SameSite=Strict ใน JWT cookie
res.cookie('token', jwt, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',  // ป้องกัน CSRF โดย browser จะไม่ส่ง cookie cross-site
  maxAge: 15 * 60 * 1000,
});
```

---

## Rate Limiting Advanced

### ขั้นตอนที่ 809: Advanced Rate Limiting Strategies

```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const Redis = require('ioredis');

const redis = new Redis(process.env.REDIS_URL);

// Sliding window rate limiter ด้วย Redis
class SlidingWindowRateLimiter {
  constructor(redis, options = {}) {
    this.redis = redis;
    this.windowMs = options.windowMs || 60000;   // 1 minute
    this.max = options.max || 100;
  }

  async isAllowed(key) {
    const now = Date.now();
    const windowStart = now - this.windowMs;

    const pipeline = this.redis.pipeline();
    pipeline.zremrangebyscore(key, 0, windowStart);     // ลบ requests เก่า
    pipeline.zadd(key, now, `${now}-${Math.random()}`); // เพิ่ม request ใหม่
    pipeline.zcard(key);                                 // นับ requests ใน window
    pipeline.pexpire(key, this.windowMs);                // ตั้ง TTL

    const results = await pipeline.exec();
    const count = results[2][1];

    return {
      allowed: count <= this.max,
      remaining: Math.max(0, this.max - count),
      resetAt: now + this.windowMs,
    };
  }
}

// Rate limiting per user/IP
const createApiLimiter = (max, windowMs) => rateLimit({
  windowMs,
  max,
  standardHeaders: true,   // Return rate limit info in headers
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args) => redis.call(...args),
  }),
  keyGenerator: (req) => {
    // Rate limit by user ID if authenticated, IP if not
    return req.user?.id || req.ip;
  },
  handler: (req, res) => {
    res.status(429).json({
      error: 'Too many requests',
      retryAfter: Math.ceil(req.rateLimit.resetTime / 1000),
    });
  },
});

// Different limits for different endpoints
app.use('/api/auth/login', createApiLimiter(5, 15 * 60 * 1000));  // 5 per 15min
app.use('/api/auth/register', createApiLimiter(3, 60 * 60 * 1000)); // 3 per hour
app.use('/api/', createApiLimiter(100, 60 * 1000));  // 100 per minute

// Token bucket algorithm
class TokenBucket {
  constructor(redis, capacity, refillRate) {
    this.redis = redis;
    this.capacity = capacity;
    this.refillRate = refillRate; // tokens per second
  }

  async consume(key, tokens = 1) {
    const now = Date.now() / 1000;  // Unix timestamp in seconds

    const script = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local refillRate = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])
      local requested = tonumber(ARGV[4])

      local bucket = redis.call('HMGET', key, 'tokens', 'lastRefill')
      local tokens = tonumber(bucket[1]) or capacity
      local lastRefill = tonumber(bucket[2]) or now

      -- เติม tokens
      local elapsed = now - lastRefill
      tokens = math.min(capacity, tokens + elapsed * refillRate)

      -- ตรวจสอบว่ามี tokens พอไหม
      if tokens < requested then
        redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
        return 0
      end

      -- หัก tokens
      tokens = tokens - requested
      redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
      redis.call('EXPIRE', key, 3600)
      return 1
    `;

    const result = await this.redis.eval(
      script, 1, key,
      this.capacity, this.refillRate, now, tokens
    );

    return result === 1;
  }
}
```

---

## Input Validation

### ขั้นตอนที่ 810: Advanced Input Validation

```javascript
// middleware/validation.js
const { body, query, param, validationResult } = require('express-validator');
const validator = require('validator');

// Custom validators
const customValidators = {
  isThaiPhoneNumber: (value) => {
    if (!value) return true; // optional
    return /^(0[689]\d{8}|0[2-9]\d{7})$/.test(value);
  },

  isStrongPassword: (value) => {
    return validator.isStrongPassword(value, {
      minLength: 8,
      minLowercase: 1,
      minUppercase: 1,
      minNumbers: 1,
      minSymbols: 1,
    });
  },

  isSafeFilename: (value) => {
    // ป้องกัน path traversal
    return !(/[<>:"/\\|?*\x00-\x1F]/.test(value)) && 
           value !== '..' && 
           value !== '.';
  },

  isValidMongoId: (value) => {
    return /^[0-9a-fA-F]{24}$/.test(value);
  },
};

// Validation chains
const registerValidation = [
  body('email')
    .isEmail().withMessage('อีเมลไม่ถูกต้อง')
    .normalizeEmail()
    .isLength({ max: 255 }).withMessage('อีเมลยาวเกินไป'),

  body('password')
    .custom(customValidators.isStrongPassword)
    .withMessage('รหัสผ่านต้องมีตัวพิมพ์ใหญ่ พิมพ์เล็ก ตัวเลข และอักขระพิเศษ'),

  body('username')
    .trim()
    .isAlphanumeric('en-US', { ignore: '_-' })
    .withMessage('ชื่อผู้ใช้ต้องเป็นตัวอักษรและตัวเลขเท่านั้น')
    .isLength({ min: 3, max: 30 })
    .withMessage('ชื่อผู้ใช้ต้องมี 3-30 ตัวอักษร'),

  body('phone')
    .optional()
    .custom(customValidators.isThaiPhoneNumber)
    .withMessage('เบอร์โทรศัพท์ไทยไม่ถูกต้อง'),
];

const paginationValidation = [
  query('page')
    .optional()
    .isInt({ min: 1 }).withMessage('page ต้องเป็นตัวเลขบวก')
    .toInt(),

  query('limit')
    .optional()
    .isInt({ min: 1, max: 100 }).withMessage('limit ต้องอยู่ระหว่าง 1-100')
    .toInt(),

  query('sort')
    .optional()
    .isIn(['createdAt', '-createdAt', 'title', '-title']).withMessage('sort ไม่ถูกต้อง'),
];

// Middleware to check validation results
const validate = (req, res, next) => {
  const errors = validationResult(req);

  if (!errors.isEmpty()) {
    return res.status(400).json({
      error: 'Validation failed',
      errors: errors.array().map(err => ({
        field: err.path,
        message: err.msg,
        value: err.value,
      })),
    });
  }

  next();
};

// File upload validation
const { MulterError } = require('multer');

const fileValidation = (req, res, next) => {
  const allowedMimeTypes = ['image/jpeg', 'image/png', 'image/webp'];
  const maxSize = 5 * 1024 * 1024; // 5MB

  if (!req.file) return next();

  // ตรวจสอบ MIME type จาก file content ไม่ใช่ extension
  const fileType = require('file-type');
  const type = fileType.fromBuffer(req.file.buffer);

  if (!type || !allowedMimeTypes.includes(type.mime)) {
    return res.status(400).json({ error: 'ประเภทไฟล์ไม่ได้รับอนุญาต' });
  }

  if (req.file.size > maxSize) {
    return res.status(400).json({ error: 'ไฟล์มีขนาดเกิน 5MB' });
  }

  next();
};

module.exports = {
  registerValidation,
  paginationValidation,
  validate,
  fileValidation,
};
```

---

## Dependency Security

### ขั้นตอนที่ 811: Dependency Scanning

```bash
# ตรวจสอบ vulnerable dependencies
npm audit

# Fix automatically
npm audit fix

# Force fix (breaking changes)
npm audit fix --force

# ดู audit ในรูปแบบ JSON
npm audit --json | node -e "
  const report = require('/dev/stdin');
  const criticals = Object.values(report.vulnerabilities)
    .filter(v => v.severity === 'critical');
  console.log('Critical:', criticals.length);
"
```

```javascript
// .github/workflows/security.yml
name: Security Audit
on:
  push:
  schedule:
    - cron: '0 8 * * 1'  # ทุกวันจันทร์ 8 โมง

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4

      - name: NPM Audit
        run: |
          npm audit --audit-level=high
          if [ $? -ne 0 ]; then
            echo "::error::High/Critical vulnerabilities found!"
            exit 1
          fi

      - name: Snyk Security Scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'myapp'
          path: '.'
          format: 'HTML'
          args: --failOnCVSS 7
```

---

## Penetration Testing

### ขั้นตอนที่ 812: Security Testing กับ OWASP ZAP

```javascript
// scripts/security-scan.js
const ZapClient = require('zaproxy');

const zapOptions = {
  apiKey: process.env.ZAP_API_KEY,
  proxy: { host: 'localhost', port: 8080 },
};

const zap = new ZapClient(zapOptions);

async function runSecurityScan(targetUrl) {
  console.log(`🔍 Starting security scan for ${targetUrl}`);

  // Spider - ค้นหา endpoints ทั้งหมด
  const spiderId = await zap.spider.scan(targetUrl);
  let spiderStatus = 0;

  while (spiderStatus < 100) {
    spiderStatus = await zap.spider.status(spiderId);
    console.log(`Spider progress: ${spiderStatus}%`);
    await new Promise(r => setTimeout(r, 2000));
  }

  // Active Scan
  const scanId = await zap.ascan.scan(targetUrl);
  let scanStatus = 0;

  while (scanStatus < 100) {
    scanStatus = await zap.ascan.status(scanId);
    console.log(`Active scan progress: ${scanStatus}%`);
    await new Promise(r => setTimeout(r, 5000));
  }

  // Get results
  const alerts = await zap.core.alerts(targetUrl);

  // ผลการ scan
  const summary = {
    high: alerts.filter(a => a.risk === 'High').length,
    medium: alerts.filter(a => a.risk === 'Medium').length,
    low: alerts.filter(a => a.risk === 'Low').length,
  };

  console.log('Security scan results:', summary);

  if (summary.high > 0) {
    throw new Error(`Found ${summary.high} high risk vulnerabilities!`);
  }

  return alerts;
}
```

### ขั้นตอนที่ 813: Manual Security Testing

```javascript
// test/security/auth.security.test.js
const request = require('supertest');
const app = require('../../src/app');

describe('Security Tests', () => {
  describe('SQL/NoSQL Injection', () => {
    const injectionPayloads = [
      '{"$gt":""}',
      '{"$where":"function() { return true; }"}',
      "'; DROP TABLE users; --",
      '<script>alert("xss")</script>',
      '../../../etc/passwd',
    ];

    it.each(injectionPayloads)('should block injection: %s', async (payload) => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({ email: payload, password: 'test' });

      // ต้อง reject input ที่ไม่ถูกต้อง
      expect([400, 401, 422]).toContain(response.status);
    });
  });

  describe('Authentication Bypass', () => {
    it('should not accept expired token', async () => {
      const expiredToken = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6InVzZXIxMjMiLCJpYXQiOjE1MTYyMzkwMjIsImV4cCI6MTUxNjIzOTAyM30.invalid';

      const response = await request(app)
        .get('/api/users/me')
        .set('Authorization', `Bearer ${expiredToken}`);

      expect(response.status).toBe(401);
    });

    it('should not accept modified token', async () => {
      // สร้าง valid token แล้วแก้ไข payload
      const token = 'valid.tampered.signature';

      const response = await request(app)
        .get('/api/users/me')
        .set('Authorization', `Bearer ${token}`);

      expect(response.status).toBe(401);
    });
  });

  describe('Access Control', () => {
    it('regular user cannot access admin endpoints', async () => {
      const userToken = await getRegularUserToken();

      const response = await request(app)
        .get('/api/admin/users')
        .set('Authorization', `Bearer ${userToken}`);

      expect(response.status).toBe(403);
    });

    it('user cannot access another users data', async () => {
      const userToken = await getRegularUserToken('user1');
      const otherUserId = 'user2-id';

      const response = await request(app)
        .get(`/api/users/${otherUserId}/private-data`)
        .set('Authorization', `Bearer ${userToken}`);

      expect(response.status).toBe(403);
    });
  });

  describe('Rate Limiting', () => {
    it('should rate limit login attempts', async () => {
      const attempts = [];

      for (let i = 0; i < 10; i++) {
        attempts.push(
          request(app)
            .post('/api/auth/login')
            .send({ email: 'test@example.com', password: 'wrong' })
        );
      }

      const responses = await Promise.all(attempts);
      const rateLimited = responses.filter(r => r.status === 429);

      expect(rateLimited.length).toBeGreaterThan(0);
    });
  });
});
```

---

## Secrets Management

### ขั้นตอนที่ 814: Secure Configuration Management

```javascript
// config/secrets.js
const AWS = require('aws-sdk');
const ssm = new AWS.SSM();

class SecretsManager {
  constructor() {
    this.cache = new Map();
    this.cacheTTL = 5 * 60 * 1000; // 5 minutes
  }

  async getSecret(name) {
    // ตรวจสอบ cache
    const cached = this.cache.get(name);
    if (cached && cached.expires > Date.now()) {
      return cached.value;
    }

    // ดึงจาก AWS Parameter Store
    const response = await ssm.getParameter({
      Name: name,
      WithDecryption: true,
    }).promise();

    const value = response.Parameter.Value;

    // บันทึก cache
    this.cache.set(name, {
      value,
      expires: Date.now() + this.cacheTTL,
    });

    return value;
  }

  async loadConfig() {
    const [jwtSecret, dbPassword, redisPassword] = await Promise.all([
      this.getSecret('/myapp/prod/jwt-secret'),
      this.getSecret('/myapp/prod/db-password'),
      this.getSecret('/myapp/prod/redis-password'),
    ]);

    return {
      jwtSecret,
      dbPassword,
      redisPassword,
    };
  }
}

// .env file validation
const Joi = require('joi');

const envSchema = Joi.object({
  NODE_ENV: Joi.string().valid('development', 'production', 'test').required(),
  PORT: Joi.number().integer().min(1000).max(65535).default(3000),
  MONGODB_URI: Joi.string().uri().required(),
  JWT_SECRET: Joi.string().min(32).required(),
  JWT_REFRESH_SECRET: Joi.string().min(32).required(),
  REDIS_URL: Joi.string().uri().required(),
}).unknown(true);

const { error, value } = envSchema.validate(process.env);

if (error) {
  throw new Error(`Environment validation error: ${error.message}`);
}

// ห้ามใช้ default secrets ใน production
if (process.env.NODE_ENV === 'production') {
  if (process.env.JWT_SECRET === 'default-secret' || 
      process.env.JWT_SECRET?.length < 32) {
    throw new Error('Insecure JWT_SECRET in production!');
  }
}
```

---

## Security Logging

### ขั้นตอนที่ 815: Security Event Logging

```javascript
// middleware/securityLogger.js
const winston = require('winston');

const securityLogger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json(),
  ),
  transports: [
    new winston.transports.File({ filename: 'logs/security.log' }),
  ],
});

const SECURITY_EVENTS = {
  LOGIN_SUCCESS: 'login_success',
  LOGIN_FAILED: 'login_failed',
  ACCOUNT_LOCKED: 'account_locked',
  PASSWORD_CHANGED: 'password_changed',
  UNAUTHORIZED_ACCESS: 'unauthorized_access',
  SUSPICIOUS_ACTIVITY: 'suspicious_activity',
  RATE_LIMIT_EXCEEDED: 'rate_limit_exceeded',
  INJECTION_ATTEMPT: 'injection_attempt',
};

function logSecurityEvent(event, data = {}) {
  securityLogger.info({
    event,
    timestamp: new Date().toISOString(),
    ...data,
  });
}

// Middleware สำหรับ detect suspicious patterns
const detectSuspiciousActivity = (req, res, next) => {
  const suspiciousPatterns = [
    /(\.\.\/)|(\.\.\\)/,           // Path traversal
    /<script[\s\S]*?>[\s\S]*?<\/script>/i,  // XSS
    /(\$where|\$gt|\$or|\$and)/,   // NoSQL injection
    /(select|insert|update|delete|drop|union)[\s\S]*?(from|into|table)/i,  // SQL injection
  ];

  const requestBody = JSON.stringify(req.body);
  const requestQuery = JSON.stringify(req.query);

  for (const pattern of suspiciousPatterns) {
    if (pattern.test(requestBody) || pattern.test(requestQuery)) {
      logSecurityEvent(SECURITY_EVENTS.INJECTION_ATTEMPT, {
        ip: req.ip,
        method: req.method,
        path: req.path,
        userAgent: req.get('User-Agent'),
      });

      return res.status(400).json({ error: 'Invalid input detected' });
    }
  }

  next();
};

module.exports = { logSecurityEvent, detectSuspiciousActivity, SECURITY_EVENTS };
```

---

## HTTPS และ SSL/TLS

### ขั้นตอนที่ 816: HTTPS Configuration

```javascript
// server.js
const https = require('https');
const http = require('http');
const fs = require('fs');

if (process.env.NODE_ENV === 'production') {
  // Production: ใช้ certificates จาก Let's Encrypt
  const tlsOptions = {
    cert: fs.readFileSync('/etc/letsencrypt/live/myapp.com/fullchain.pem'),
    key: fs.readFileSync('/etc/letsencrypt/live/myapp.com/privkey.pem'),
    // Security settings
    secureOptions: require('constants').SSL_OP_NO_SSLv3 |
                   require('constants').SSL_OP_NO_TLSv1 |
                   require('constants').SSL_OP_NO_TLSv1_1,
    ciphers: [
      'ECDHE-ECDSA-AES128-GCM-SHA256',
      'ECDHE-RSA-AES128-GCM-SHA256',
      'ECDHE-ECDSA-AES256-GCM-SHA384',
    ].join(':'),
    honorCipherOrder: true,
    minVersion: 'TLSv1.2',
  };

  https.createServer(tlsOptions, app).listen(443);

  // Redirect HTTP to HTTPS
  http.createServer((req, res) => {
    res.writeHead(301, { Location: `https://${req.headers.host}${req.url}` });
    res.end();
  }).listen(80);

} else {
  app.listen(3000);
}
```

---

## Advanced Authorization

### ขั้นตอนที่ 817: RBAC Implementation

```javascript
// middleware/rbac.js
const permissions = {
  USER: {
    post: ['create', 'read', 'update:own', 'delete:own'],
    comment: ['create', 'read', 'update:own', 'delete:own'],
    profile: ['read', 'update:own'],
  },
  MODERATOR: {
    post: ['create', 'read', 'update:own', 'delete:own', 'delete:any'],
    comment: ['create', 'read', 'update:own', 'delete:any'],
    profile: ['read', 'update:own'],
    user: ['read'],
  },
  ADMIN: {
    post: ['*'],
    comment: ['*'],
    profile: ['*'],
    user: ['*'],
    settings: ['*'],
  },
};

function hasPermission(role, resource, action, userId, resourceUserId) {
  const rolePermissions = permissions[role]?.[resource];
  if (!rolePermissions) return false;

  if (rolePermissions.includes('*')) return true;
  if (rolePermissions.includes(action)) return true;

  // ตรวจสอบ :own permission
  if (action === 'update' || action === 'delete') {
    if (rolePermissions.includes(`${action}:own`) && userId === resourceUserId) {
      return true;
    }
    if (rolePermissions.includes(`${action}:any`)) {
      return true;
    }
  }

  return false;
}

// Middleware factory
const authorize = (resource, action) => async (req, res, next) => {
  if (!req.user) {
    return res.status(401).json({ error: 'Authentication required' });
  }

  const { id: userId, role } = req.user;
  const resourceUserId = req.params.userId || req.resource?.userId;

  if (!hasPermission(role, resource, action, userId, resourceUserId)) {
    return res.status(403).json({ error: 'Insufficient permissions' });
  }

  next();
};

// Usage
app.put('/posts/:id',
  authenticate,
  async (req, res, next) => {
    // โหลด resource สำหรับ :own check
    req.resource = await Post.findById(req.params.id);
    if (!req.resource) return res.status(404).json({ error: 'Not found' });
    next();
  },
  authorize('post', 'update'),
  updatePostHandler
);
```

---

## Security Misconfiguration

### ขั้นตอนที่ 818: Hardening Configuration

```javascript
// config/security.js

module.exports = {
  // CORS Configuration
  cors: {
    origin: process.env.CORS_ORIGINS?.split(',') || ['http://localhost:3000'],
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-Requested-With'],
    exposedHeaders: ['X-RateLimit-Limit', 'X-RateLimit-Remaining'],
    maxAge: 86400,  // 24 hours
  },

  // Cookie Settings
  session: {
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 24 * 60 * 60 * 1000,  // 24 hours
    },
  },

  // Password Policy
  password: {
    minLength: 8,
    requireUppercase: true,
    requireLowercase: true,
    requireNumbers: true,
    requireSpecial: true,
    bcryptRounds: 12,
  },

  // Account Lockout
  lockout: {
    maxAttempts: 5,
    durationMs: 15 * 60 * 1000,  // 15 minutes
  },

  // JWT
  jwt: {
    accessTokenExpiry: '15m',
    refreshTokenExpiry: '7d',
    algorithm: 'HS256',
    issuer: process.env.APP_URL,
  },
};
```

---

## Error Handling Security

### ขั้นตอนที่ 819: Secure Error Handling

```javascript
// middleware/errorHandler.js
const logger = require('../config/logger');

const errorHandler = (err, req, res, next) => {
  // Log full error internally
  logger.error({
    message: err.message,
    stack: err.stack,
    url: req.originalUrl,
    method: req.method,
    ip: req.ip,
    userId: req.user?.id,
  });

  // ส่ง error response ที่ปลอดภัย (ไม่เปิดเผย internal details)
  const statusCode = err.statusCode || 500;

  // Production: ซ่อน internal errors
  if (process.env.NODE_ENV === 'production' && statusCode === 500) {
    return res.status(500).json({
      error: 'Internal server error',
      requestId: req.id,  // สำหรับ tracking ใน logs
    });
  }

  // Development: แสดง details
  res.status(statusCode).json({
    error: err.message,
    ...(process.env.NODE_ENV !== 'production' && { stack: err.stack }),
  });
};

// ป้องกัน unhandled rejections
process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled Rejection:', { reason, promise });
  // ไม่ crash server แต่ log ไว้
});

process.on('uncaughtException', (err) => {
  logger.error('Uncaught Exception:', err);
  // Graceful shutdown
  process.exit(1);
});

module.exports = errorHandler;
```

---

## Security Audit Script

### ขั้นตอนที่ 820: Security Audit Checklist

```javascript
// scripts/security-audit.js
const fs = require('fs');
const path = require('path');

class SecurityAuditor {
  constructor(projectPath) {
    this.projectPath = projectPath;
    this.issues = [];
  }

  checkHardcodedSecrets() {
    const patterns = [
      { name: 'API Key', regex: /api[_-]?key\s*=\s*['"][^'"]{16,}['"]/gi },
      { name: 'Password', regex: /password\s*=\s*['"][^'"]{4,}['"]/gi },
      { name: 'Secret', regex: /secret\s*=\s*['"][^'"]{8,}['"]/gi },
      { name: 'Token', regex: /token\s*=\s*['"][^'"]{16,}['"]/gi },
    ];

    const jsFiles = this.getJsFiles(this.projectPath);

    for (const file of jsFiles) {
      const content = fs.readFileSync(file, 'utf8');
      for (const { name, regex } of patterns) {
        if (regex.test(content)) {
          this.issues.push({
            severity: 'HIGH',
            type: 'Hardcoded Secret',
            file,
            description: `Possible hardcoded ${name}`,
          });
        }
      }
    }
  }

  checkMissingHelmet() {
    const appFiles = ['app.js', 'server.js', 'index.js'].map(f =>
      path.join(this.projectPath, 'src', f)
    );

    for (const file of appFiles) {
      if (!fs.existsSync(file)) continue;
      const content = fs.readFileSync(file, 'utf8');

      if (!content.includes('helmet')) {
        this.issues.push({
          severity: 'MEDIUM',
          type: 'Missing Security Headers',
          file,
          description: 'helmet middleware not found',
        });
      }
    }
  }

  checkUnsafeRegex() {
    const jsFiles = this.getJsFiles(this.projectPath);

    for (const file of jsFiles) {
      const content = fs.readFileSync(file, 'utf8');
      // ค้นหา regex ที่อาจ vulnerable to ReDoS
      const reDoSPattern = /\(.*\+\).*\+|\(.*\*\).*\+|\(.*\+\).*\*/g;
      if (reDoSPattern.test(content)) {
        this.issues.push({
          severity: 'LOW',
          type: 'Potential ReDoS',
          file,
          description: 'Potentially vulnerable regex detected',
        });
      }
    }
  }

  generateReport() {
    const report = {
      timestamp: new Date().toISOString(),
      project: this.projectPath,
      summary: {
        total: this.issues.length,
        high: this.issues.filter(i => i.severity === 'HIGH').length,
        medium: this.issues.filter(i => i.severity === 'MEDIUM').length,
        low: this.issues.filter(i => i.severity === 'LOW').length,
      },
      issues: this.issues,
    };

    fs.writeFileSync(
      'security-audit-report.json',
      JSON.stringify(report, null, 2)
    );

    console.log('Security Audit Results:');
    console.log(`High: ${report.summary.high}`);
    console.log(`Medium: ${report.summary.medium}`);
    console.log(`Low: ${report.summary.low}`);

    if (report.summary.high > 0) {
      process.exit(1);
    }

    return report;
  }

  getJsFiles(dir) {
    const files = [];
    const items = fs.readdirSync(dir, { withFileTypes: true });

    for (const item of items) {
      if (item.name === 'node_modules') continue;
      const fullPath = path.join(dir, item.name);

      if (item.isDirectory()) {
        files.push(...this.getJsFiles(fullPath));
      } else if (item.name.endsWith('.js')) {
        files.push(fullPath);
      }
    }

    return files;
  }
}

// Run audit
const auditor = new SecurityAuditor(process.cwd());
auditor.checkHardcodedSecrets();
auditor.checkMissingHelmet();
auditor.checkUnsafeRegex();
auditor.generateReport();
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Secure API

```javascript
// TODO: สร้าง secure REST API ที่มี:
// 1. Authentication ด้วย JWT
// 2. Rate limiting per user
// 3. Input validation ทุก endpoint
// 4. Security headers ด้วย Helmet
// 5. CORS configuration ที่เหมาะสม
// 6. Error handling ที่ปลอดภัย
// 7. Security event logging
```

### แบบฝึกหัดที่ 2: Penetration Test

```javascript
// TODO: เขียน security tests สำหรับ:
// 1. SQL/NoSQL Injection attempts
// 2. XSS attacks
// 3. Authentication bypass
// 4. Privilege escalation
// 5. Rate limit bypass attempts
```

### แบบฝึกหัดที่ 3: Security Audit

```javascript
// TODO: Run security audit บน project ของคุณ:
// 1. npm audit
// 2. ตรวจสอบ OWASP Top 10
// 3. สร้าง security report
// 4. Fix vulnerabilities ที่พบ
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **OWASP Top 10** - vulnerability categories หลัก
2. **Injection Prevention** - NoSQL, SQL, XSS
3. **Secure Authentication** - account lockout, token rotation
4. **Security Headers** - helmet, CSP, HSTS
5. **CSRF Protection** - csrf tokens, SameSite cookies
6. **Rate Limiting** - sliding window, token bucket
7. **Input Validation** - express-validator, file validation
8. **Dependency Security** - npm audit, Snyk
9. **Penetration Testing** - OWASP ZAP, security test cases
10. **Secrets Management** - AWS Parameter Store, env validation
11. **RBAC** - role-based access control
12. **Security Logging** - event logging, monitoring

ยินดีด้วย! คุณได้เรียนรู้ Node.js และ Express.js ในระดับ Advanced แล้ว ตั้งแต่ GraphQL, WebSockets, Redis, Message Queues, Microservices, Docker, CI/CD, AWS, Monitoring, Testing จนถึง Security ขั้นสูง
