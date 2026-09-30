# Part 23: Authentication Basics

> ขั้นตอนที่ 23-30 จาก 1000 — ระบบ Authentication พื้นฐานสำหรับ Web Application

---

## สารบัญ

1. [Authentication Concepts](#authentication-concepts)
2. [Username/Password Authentication](#usernamepassword-authentication)
3. [Hashing Passwords ด้วย bcrypt](#hashing-passwords-ด้วย-bcrypt)
4. [Express Sessions](#express-sessions)
5. [Remember Me](#remember-me)
6. [OAuth2 Overview](#oauth2-overview)
7. [Practical: Full Auth System](#practical-full-auth-system)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Authentication Concepts

### Authentication vs Authorization

| Authentication | Authorization |
|----------------|---------------|
| "คุณเป็นใคร?" | "คุณทำอะไรได้บ้าง?" |
| ตรวจสอบตัวตน | ตรวจสอบสิทธิ์ |
| Login/Logout | Role-based access |

### ประเภทของ Authentication

1. **Knowledge-based** — สิ่งที่คุณรู้ (password, PIN)
2. **Possession-based** — สิ่งที่คุณมี (OTP, hardware token)
3. **Inherence-based** — สิ่งที่คุณเป็น (fingerprint, face)
4. **Multi-factor (MFA)** — รวมหลายประเภท

### Authentication Flows

```
Session-based:
User → Login → Server creates session → Cookie → User requests with cookie

Token-based (JWT):
User → Login → Server creates token → User stores token → User sends token in header

OAuth2:
User → "Login with Google" → Redirect to Google → Google authenticates → Callback with code → Exchange for token
```

---

## Username/Password Authentication

### การตรวจสอบ Input

```javascript
// middleware/validators.js
const { body, validationResult } = require('express-validator');

exports.validateRegister = [
  body('name')
    .notEmpty().withMessage('กรุณาระบุชื่อ')
    .isLength({ min: 2, max: 100 }).withMessage('ชื่อต้องมี 2-100 ตัวอักษร')
    .trim(),
  
  body('email')
    .notEmpty().withMessage('กรุณาระบุ email')
    .isEmail().withMessage('รูปแบบ email ไม่ถูกต้อง')
    .normalizeEmail(),
  
  body('password')
    .notEmpty().withMessage('กรุณาระบุรหัสผ่าน')
    .isLength({ min: 8 }).withMessage('รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร')
    .matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
    .withMessage('รหัสผ่านต้องมีตัวพิมพ์เล็ก พิมพ์ใหญ่ และตัวเลข'),
  
  body('confirmPassword')
    .custom((value, { req }) => {
      if (value !== req.body.password) {
        throw new Error('รหัสผ่านไม่ตรงกัน');
      }
      return true;
    }),
  
  (req, res, next) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(422).json({
        success: false,
        errors: errors.array().map((e) => ({
          field: e.path,
          message: e.msg,
        })),
      });
    }
    next();
  },
];
```

---

## Hashing Passwords ด้วย bcrypt

### ทำไมต้อง Hash?

ไม่ควรเก็บ password ในรูปแบบ plaintext เนื่องจาก:
- หาก database ถูก breach ผู้ใช้ทุกคนมีความเสี่ยง
- ผู้ดูแล database ไม่ควรเห็น password ของผู้ใช้

### bcrypt ทำงานอย่างไร

```
password + salt → bcrypt hash

"myPassword" + "$2b$12$randomSalt..." → "$2b$12$randomSalt...hashedValue"
```

- **Salt** — random string ที่ใส่เข้าไปก่อน hash ป้องกัน rainbow table attacks
- **Cost factor** — จำนวนรอบ (rounds) ของการ hash (ยิ่งมาก ยิ่งช้า ยิ่งปลอดภัย)

### การใช้ bcrypt

```bash
npm install bcryptjs
```

```javascript
// utils/password.js
const bcrypt = require('bcryptjs');

const SALT_ROUNDS = 12;  // ค่าแนะนำสำหรับ production

/**
 * Hash password
 * @param {string} password - plaintext password
 * @returns {Promise<string>} hashed password
 */
exports.hashPassword = async (password) => {
  return bcrypt.hash(password, SALT_ROUNDS);
};

/**
 * Compare password กับ hash
 * @param {string} candidatePassword - password ที่ผู้ใช้ส่งมา
 * @param {string} hashedPassword - hash ที่เก็บใน database
 * @returns {Promise<boolean>}
 */
exports.comparePassword = async (candidatePassword, hashedPassword) => {
  return bcrypt.compare(candidatePassword, hashedPassword);
};

// ตัวอย่างการใช้งาน
const demonstrateBcrypt = async () => {
  const password = 'MyPassword123!';
  
  // Hash
  const hash = await bcrypt.hash(password, 12);
  console.log('Hash:', hash);
  // $2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewdBPj/RdFCi2EQC
  
  // เปรียบเทียบ
  const isMatch = await bcrypt.compare(password, hash);
  console.log('Match:', isMatch);  // true
  
  const notMatch = await bcrypt.compare('wrongPassword', hash);
  console.log('Wrong:', notMatch);  // false
  
  // สร้าง salt แยก (ไม่ค่อยใช้)
  const salt = await bcrypt.genSalt(12);
  const hash2 = await bcrypt.hash(password, salt);
};
```

### ทดสอบ Performance

```javascript
// benchmark salt rounds
const benchmarkBcrypt = async () => {
  const password = 'TestPassword123';
  
  for (let rounds = 8; rounds <= 14; rounds++) {
    const start = Date.now();
    await bcrypt.hash(password, rounds);
    const time = Date.now() - start;
    console.log(`Rounds: ${rounds}, Time: ${time}ms`);
  }
};

// ผลลัพธ์ประมาณ:
// Rounds: 8,  Time: 40ms
// Rounds: 10, Time: 150ms
// Rounds: 12, Time: 600ms   ← แนะนำ
// Rounds: 14, Time: 2400ms
```

---

## Express Sessions

Session เก็บข้อมูล user ไว้บน server และส่ง session ID ผ่าน cookie

### ติดตั้งและตั้งค่า

```bash
npm install express-session connect-redis redis
# หรือใช้ MongoDB store
npm install connect-mongodb-session
```

```javascript
// config/session.js
const session = require('express-session');
const MongoStore = require('connect-mongodb-session')(session);
// หรือ RedisStore สำหรับ production

const sessionStore = new MongoStore({
  uri: process.env.MONGODB_URI,
  collection: 'sessions',
  expires: 1000 * 60 * 60 * 24 * 7,  // 7 วัน
});

sessionStore.on('error', (error) => {
  console.error('Session store error:', error);
});

const sessionMiddleware = session({
  secret: process.env.SESSION_SECRET || 'your-super-secret-key',
  resave: false,            // ไม่ save session ถ้าไม่มีการเปลี่ยนแปลง
  saveUninitialized: false, // ไม่ save session ที่ยังไม่มีข้อมูล
  store: sessionStore,
  
  cookie: {
    httpOnly: true,         // JavaScript ไม่สามารถอ่าน cookie ได้
    secure: process.env.NODE_ENV === 'production',  // HTTPS only ใน production
    sameSite: 'lax',        // ป้องกัน CSRF
    maxAge: 1000 * 60 * 60 * 24 * 7,  // 7 วัน
  },
  
  name: 'sessionId',        // เปลี่ยนชื่อจาก default 'connect.sid'
});

module.exports = sessionMiddleware;
```

### Session-based Auth Controller

```javascript
// controllers/authController.js
const User = require('../models/User');
const { hashPassword, comparePassword } = require('../utils/password');

// Register
exports.register = async (req, res) => {
  try {
    const { name, email, password } = req.body;
    
    // ตรวจสอบว่า email มีอยู่แล้วหรือไม่
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({
        success: false,
        message: 'Email นี้มีผู้ใช้แล้ว',
      });
    }
    
    // Hash password
    const hashedPassword = await hashPassword(password);
    
    // สร้าง user
    const user = await User.create({
      name,
      email,
      password: hashedPassword,
    });
    
    // เริ่ม session
    req.session.userId = user._id;
    req.session.role = user.role;
    
    res.status(201).json({
      success: true,
      message: 'สมัครสมาชิกเรียบร้อยแล้ว',
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

// Login
exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;
    
    // หา user
    const user = await User.findOne({ email }).select('+password');
    if (!user) {
      // ใช้ข้อความเดียวกันเพื่อไม่ให้รู้ว่า email มีอยู่หรือไม่
      return res.status(401).json({
        success: false,
        message: 'Email หรือรหัสผ่านไม่ถูกต้อง',
      });
    }
    
    // ตรวจสอบ account lock
    if (user.lockUntil && user.lockUntil > Date.now()) {
      const minutesLeft = Math.ceil((user.lockUntil - Date.now()) / 60000);
      return res.status(423).json({
        success: false,
        message: `บัญชีถูกล็อก กรุณารอ ${minutesLeft} นาที`,
      });
    }
    
    // ตรวจสอบ password
    const isMatch = await comparePassword(password, user.password);
    if (!isMatch) {
      // เพิ่ม failed attempt count
      await User.findByIdAndUpdate(user._id, {
        $inc: { loginAttempts: 1 },
        ...(user.loginAttempts >= 4 && {
          lockUntil: new Date(Date.now() + 30 * 60 * 1000),  // lock 30 นาที
        }),
      });
      
      return res.status(401).json({
        success: false,
        message: 'Email หรือรหัสผ่านไม่ถูกต้อง',
      });
    }
    
    // Reset failed attempts
    await User.findByIdAndUpdate(user._id, {
      loginAttempts: 0,
      lockUntil: null,
      lastLoginAt: new Date(),
    });
    
    // สร้าง session
    req.session.userId = user._id;
    req.session.role = user.role;
    
    // Regenerate session ID ป้องกัน session fixation
    req.session.regenerate((err) => {
      if (err) throw err;
      
      req.session.userId = user._id;
      req.session.role = user.role;
      
      res.json({
        success: true,
        message: 'เข้าสู่ระบบเรียบร้อยแล้ว',
        user: {
          id: user._id,
          name: user.name,
          email: user.email,
          role: user.role,
        },
      });
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Logout
exports.logout = (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      return res.status(500).json({
        success: false,
        message: 'ออกจากระบบไม่สำเร็จ',
      });
    }
    
    res.clearCookie('sessionId');
    res.json({
      success: true,
      message: 'ออกจากระบบเรียบร้อยแล้ว',
    });
  });
};

// Get current user
exports.getMe = async (req, res) => {
  try {
    const user = await User.findById(req.session.userId).select('-password');
    
    if (!user) {
      return res.status(401).json({
        success: false,
        message: 'ไม่ได้รับการยืนยันตัวตน',
      });
    }
    
    res.json({ success: true, user });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

### Auth Middleware

```javascript
// middleware/auth.js

// ตรวจสอบว่า login อยู่หรือไม่
exports.requireAuth = async (req, res, next) => {
  if (!req.session.userId) {
    return res.status(401).json({
      success: false,
      message: 'กรุณาเข้าสู่ระบบก่อน',
    });
  }
  
  try {
    const user = await User.findById(req.session.userId).select('-password');
    
    if (!user || !user.isActive) {
      req.session.destroy();
      return res.status(401).json({
        success: false,
        message: 'Session ไม่ถูกต้อง',
      });
    }
    
    req.user = user;
    next();
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// ตรวจสอบ roles
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
        message: 'คุณไม่มีสิทธิ์ในการดำเนินการนี้',
      });
    }
    
    next();
  };
};
```

---

## Remember Me

```javascript
// Login with Remember Me
exports.login = async (req, res) => {
  const { email, password, rememberMe } = req.body;
  
  // ... ตรวจสอบ credentials ...
  
  // ปรับ session expiry ตาม rememberMe
  if (rememberMe) {
    req.session.cookie.maxAge = 30 * 24 * 60 * 60 * 1000;  // 30 วัน
  } else {
    req.session.cookie.expires = false;  // session cookie (ปิด browser = หมดอายุ)
  }
  
  req.session.userId = user._id;
  
  res.json({ success: true, message: 'เข้าสู่ระบบเรียบร้อย' });
};
```

---

## OAuth2 Overview

### OAuth2 Flow

```
1. User คลิก "Login with Google"
2. App redirect ไป Google OAuth endpoint
3. User อนุญาต (grant permission)
4. Google redirect กลับมาพร้อม authorization code
5. Server แลก code กับ access token
6. Server ใช้ access token ดึงข้อมูล user จาก Google
7. สร้าง/อัปเดต user ใน database
8. สร้าง session สำหรับ user
```

### Passport.js สำหรับ OAuth

```bash
npm install passport passport-google-oauth20 passport-local
```

```javascript
// config/passport.js
const passport = require('passport');
const LocalStrategy = require('passport-local').Strategy;
const GoogleStrategy = require('passport-google-oauth20').Strategy;
const User = require('../models/User');
const { comparePassword } = require('../utils/password');

// Local Strategy (username/password)
passport.use(
  new LocalStrategy(
    { usernameField: 'email', passwordField: 'password' },
    async (email, password, done) => {
      try {
        const user = await User.findOne({ email }).select('+password');
        
        if (!user) {
          return done(null, false, { message: 'Email หรือรหัสผ่านไม่ถูกต้อง' });
        }
        
        const isMatch = await comparePassword(password, user.password);
        if (!isMatch) {
          return done(null, false, { message: 'Email หรือรหัสผ่านไม่ถูกต้อง' });
        }
        
        return done(null, user);
      } catch (error) {
        return done(error);
      }
    }
  )
);

// Google Strategy
passport.use(
  new GoogleStrategy(
    {
      clientID: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
      callbackURL: `${process.env.APP_URL}/auth/google/callback`,
    },
    async (accessToken, refreshToken, profile, done) => {
      try {
        // หา user ด้วย Google ID
        let user = await User.findOne({ googleId: profile.id });
        
        if (user) {
          return done(null, user);
        }
        
        // หา user ด้วย email (กรณีที่เคย register ด้วย email แล้ว)
        user = await User.findOne({ email: profile.emails[0].value });
        
        if (user) {
          // เชื่อม Google account กับ user ที่มีอยู่
          user.googleId = profile.id;
          await user.save();
          return done(null, user);
        }
        
        // สร้าง user ใหม่
        user = await User.create({
          googleId: profile.id,
          name: profile.displayName,
          email: profile.emails[0].value,
          avatar: profile.photos[0]?.value,
          isEmailVerified: true,  // Google ยืนยัน email แล้ว
        });
        
        return done(null, user);
      } catch (error) {
        return done(error);
      }
    }
  )
);

// Serialize/Deserialize
passport.serializeUser((user, done) => {
  done(null, user._id);
});

passport.deserializeUser(async (id, done) => {
  try {
    const user = await User.findById(id).select('-password');
    done(null, user);
  } catch (error) {
    done(error);
  }
});

module.exports = passport;
```

```javascript
// routes/auth.js
const express = require('express');
const passport = require('../config/passport');
const router = express.Router();

// Google OAuth
router.get(
  '/google',
  passport.authenticate('google', {
    scope: ['profile', 'email'],
  })
);

router.get(
  '/google/callback',
  passport.authenticate('google', { failureRedirect: '/login?error=oauth' }),
  (req, res) => {
    res.redirect('/dashboard');
  }
);
```

---

## Practical: Full Auth System

### โครงสร้าง

```
auth-system/
├── config/
│   ├── database.js
│   ├── session.js
│   └── passport.js
├── models/
│   └── User.js (with login attempts, lock, email verification)
├── controllers/
│   └── authController.js
├── routes/
│   └── auth.js
├── middleware/
│   ├── auth.js
│   └── validators.js
├── utils/
│   ├── password.js
│   └── email.js (ส่ง verification email)
└── app.js
```

### User Model (เต็มรูปแบบ)

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const crypto = require('crypto');

const userSchema = new mongoose.Schema(
  {
    name: { type: String, required: true, trim: true },
    email: { type: String, required: true, unique: true, lowercase: true },
    password: { type: String, select: false },  // ไม่ดึงโดย default
    
    // OAuth
    googleId: String,
    githubId: String,
    avatar: String,
    
    // Email verification
    isEmailVerified: { type: Boolean, default: false },
    emailVerificationToken: String,
    emailVerificationExpires: Date,
    
    // Password reset
    passwordResetToken: String,
    passwordResetExpires: Date,
    
    // Account security
    loginAttempts: { type: Number, default: 0 },
    lockUntil: Date,
    lastLoginAt: Date,
    lastLoginIP: String,
    
    role: {
      type: String,
      enum: ['user', 'admin'],
      default: 'user',
    },
    isActive: { type: Boolean, default: true },
  },
  { timestamps: true }
);

// Hash password ก่อน save
userSchema.pre('save', async function (next) {
  if (!this.isModified('password') || !this.password) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

// Generate email verification token
userSchema.methods.generateEmailVerificationToken = function () {
  const token = crypto.randomBytes(32).toString('hex');
  this.emailVerificationToken = crypto
    .createHash('sha256')
    .update(token)
    .digest('hex');
  this.emailVerificationExpires = Date.now() + 24 * 60 * 60 * 1000;  // 24 ชั่วโมง
  return token;  // ส่ง raw token ทาง email
};

// Generate password reset token
userSchema.methods.generatePasswordResetToken = function () {
  const token = crypto.randomBytes(32).toString('hex');
  this.passwordResetToken = crypto
    .createHash('sha256')
    .update(token)
    .digest('hex');
  this.passwordResetExpires = Date.now() + 60 * 60 * 1000;  // 1 ชั่วโมง
  return token;
};

module.exports = mongoose.model('User', userSchema);
```

### Auth Routes

```javascript
// routes/auth.js
const express = require('express');
const router = express.Router();
const authController = require('../controllers/authController');
const { requireAuth } = require('../middleware/auth');
const { validateRegister, validateLogin } = require('../middleware/validators');
const rateLimit = require('express-rate-limit');

// Rate limiting สำหรับ auth routes
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 นาที
  max: 10,                    // 10 requests
  message: 'คำขอมากเกินไป กรุณารอสักครู่',
});

router.post('/register', authLimiter, validateRegister, authController.register);
router.post('/login', authLimiter, validateLogin, authController.login);
router.post('/logout', requireAuth, authController.logout);
router.get('/me', requireAuth, authController.getMe);
router.post('/verify-email/:token', authController.verifyEmail);
router.post('/forgot-password', authLimiter, authController.forgotPassword);
router.post('/reset-password/:token', authLimiter, authController.resetPassword);
router.post('/change-password', requireAuth, authController.changePassword);

module.exports = router;
```

---

## แบบฝึกหัด

### Exercise 1: Email Verification

1. ส่ง verification email เมื่อ register
2. สร้าง endpoint `/verify-email/:token`
3. หลัง verify แล้วให้ redirect ไปหน้า login
4. ส่ง email ใหม่ได้ถ้า token หมดอายุ

### Exercise 2: Password Reset

1. POST `/forgot-password` — ส่ง reset link ทาง email
2. GET `/reset-password/:token` — ตรวจสอบ token validity
3. POST `/reset-password/:token` — บันทึก password ใหม่
4. Invalidate token หลังใช้แล้ว

### Exercise 3: Account Lockout

1. Lock account หลัง login ผิด 5 ครั้ง
2. Lock เป็นเวลา 30 นาที
3. Unlock อัตโนมัติหลัง 30 นาที
4. Admin สามารถ unlock ได้

---

## สรุป

- **bcrypt** สำหรับ hash passwords อย่างปลอดภัย
- **Express Sessions** สำหรับ session-based authentication
- **Account Security** — login attempts, lockout
- **OAuth2** สำหรับ social login
- **Email Verification** และ **Password Reset**

> **บทถัดไป:** Part 24 — JWT Authentication
