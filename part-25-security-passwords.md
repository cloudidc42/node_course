# Part 25: Security และ Passwords

> ขั้นตอนที่ 25-30 จาก 1000 — ความปลอดภัยของรหัสผ่านและการยืนยันตัวตนแบบ 2 ชั้น

---

## สารบัญ

1. [Password Security Best Practices](#password-security-best-practices)
2. [bcrypt/argon2](#bcryptargon2)
3. [Salt Rounds](#salt-rounds)
4. [Password Strength Validation](#password-strength-validation)
5. [Account Lockout](#account-lockout)
6. [Password Reset Flow](#password-reset-flow)
7. [Two-Factor Authentication](#two-factor-authentication)
8. [Practical: Secure User Auth](#practical-secure-user-auth)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Password Security Best Practices

### หลักการพื้นฐาน

1. **ไม่เก็บ plaintext passwords** — ต้อง hash เสมอ
2. **ใช้ algorithm ที่ออกแบบสำหรับ password** — bcrypt, argon2, scrypt
3. **ใช้ salt** — ป้องกัน rainbow table และ precomputed attacks
4. **ปรับ cost factor** — ให้ช้าพอที่จะป้องกัน brute force
5. **ไม่ใช้ MD5/SHA** สำหรับ password (เร็วเกินไป)

### การโจมตีที่ต้องป้องกัน

| การโจมตี | การป้องกัน |
|---------|----------|
| Brute force | Rate limiting + Account lockout |
| Dictionary attack | Strong password policy |
| Rainbow table | Unique salt per password |
| Timing attack | Constant-time comparison |
| Credential stuffing | MFA + anomaly detection |
| Phishing | Security awareness |

---

## bcrypt/argon2

### bcrypt

```bash
npm install bcryptjs
```

```javascript
// utils/bcryptPassword.js
const bcrypt = require('bcryptjs');

const BCRYPT_ROUNDS = 12;

exports.hash = (password) => bcrypt.hash(password, BCRYPT_ROUNDS);
exports.compare = (plain, hashed) => bcrypt.compare(plain, hashed);
```

### argon2 (แนะนำมากกว่า bcrypt)

argon2 ชนะ Password Hashing Competition (PHC) 2015 และถือว่าปลอดภัยกว่า bcrypt

```bash
npm install argon2
```

```javascript
// utils/argon2Password.js
const argon2 = require('argon2');

const options = {
  type: argon2.argon2id,  // แนะนำสำหรับ password hashing
  memoryCost: 65536,       // 64 MB memory
  timeCost: 3,             // จำนวน iterations
  parallelism: 4,          // threads
};

exports.hash = (password) => argon2.hash(password, options);
exports.compare = (hashed, plain) => argon2.verify(hashed, plain);
exports.needsRehash = (hash) => argon2.needsRehash(hash, options);
```

### เปรียบเทียบ

| | bcrypt | argon2id |
|--|--------|----------|
| Memory hardness | ไม่มี | มี (ป้องกัน ASIC) |
| Parallelism | ไม่มี | มี |
| PHC winner | ไม่ใช่ | ใช่ |
| Node.js support | ดี (pure JS) | ดี แต่ต้องใช้ native |
| Security | ดี | ดีกว่า |

### Password Hashing Service

```javascript
// services/passwordService.js
const argon2 = require('argon2');

class PasswordService {
  constructor() {
    this.options = {
      type: argon2.argon2id,
      memoryCost: process.env.NODE_ENV === 'test' ? 8192 : 65536,
      timeCost: process.env.NODE_ENV === 'test' ? 1 : 3,
      parallelism: 4,
    };
  }
  
  async hash(password) {
    return argon2.hash(password, this.options);
  }
  
  async verify(hashedPassword, password) {
    return argon2.verify(hashedPassword, password);
  }
  
  needsRehash(hashedPassword) {
    return argon2.needsRehash(hashedPassword, this.options);
  }
  
  // rehash เมื่อ login ถ้าจำเป็น (เช่น เพิ่ม cost factor)
  async rehashOnLogin(user, plainPassword) {
    if (this.needsRehash(user.password)) {
      const newHash = await this.hash(plainPassword);
      await User.findByIdAndUpdate(user._id, { password: newHash });
    }
  }
}

module.exports = new PasswordService();
```

---

## Salt Rounds

### ทำความเข้าใจ Salt

```javascript
// bcrypt สร้าง salt อัตโนมัติ — ต่างกันทุกครั้งแม้ password เดียวกัน
const hash1 = await bcrypt.hash('password123', 12);
const hash2 = await bcrypt.hash('password123', 12);

console.log(hash1 === hash2);  // false — salt ต่างกัน
// hash1: $2b$12$LQv3c1yqBWVHxkd0LHAkCO...
// hash2: $2b$12$8SIk3Nda7zOUQeX4QKJK5O...

// แต่ทั้งคู่ match กับ 'password123'
console.log(await bcrypt.compare('password123', hash1));  // true
console.log(await bcrypt.compare('password123', hash2));  // true
```

### เลือก Cost Factor ที่เหมาะสม

```javascript
// ทดสอบ performance บน server จริง
const benchmarkCostFactor = async () => {
  const password = 'BenchmarkPassword123!';
  const targetTime = 250;  // ms (แนะนำ 200-500ms)
  
  console.log('Testing bcrypt cost factors...');
  
  for (let rounds = 10; rounds <= 14; rounds++) {
    const start = process.hrtime.bigint();
    await bcrypt.hash(password, rounds);
    const end = process.hrtime.bigint();
    const ms = Number(end - start) / 1_000_000;
    
    console.log(`rounds=${rounds}: ${ms.toFixed(0)}ms ${ms >= targetTime ? '✓' : '×'}`);
  }
};

// รันก่อน deploy เพื่อหา rounds ที่เหมาะสม
// rounds=10: 100ms
// rounds=11: 200ms ✓
// rounds=12: 400ms ✓ ← เลือกตรงนี้
// rounds=13: 800ms ✓
// rounds=14: 1600ms ✓
```

---

## Password Strength Validation

### กฎขั้นต่ำ (NIST Guidelines)

```javascript
// utils/passwordStrength.js
const zxcvbn = require('zxcvbn');  // npm install zxcvbn

/**
 * ตรวจสอบความแข็งแกร่งของรหัสผ่าน
 * @returns {{ valid: boolean, score: number, feedback: string[] }}
 */
exports.validatePassword = (password, userInputs = []) => {
  const errors = [];
  
  // ขั้นต่ำ NIST SP 800-63B
  if (password.length < 8) {
    errors.push('รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร');
  }
  
  if (password.length > 128) {
    errors.push('รหัสผ่านต้องไม่เกิน 128 ตัวอักษร');
  }
  
  // ตรวจสอบรหัสผ่านที่ถูก breach
  const commonPasswords = [
    'password', '123456', 'password123', 'qwerty', 'abc123',
    'password1', '12345678', 'iloveyou', 'admin', 'welcome',
  ];
  
  if (commonPasswords.includes(password.toLowerCase())) {
    errors.push('รหัสผ่านนี้พบได้ทั่วไป กรุณาใช้รหัสที่ยากกว่านี้');
  }
  
  // วิเคราะห์ด้วย zxcvbn
  const result = zxcvbn(password, userInputs);
  
  const scoreLabels = ['อ่อนมาก', 'อ่อน', 'พอใช้', 'ดี', 'ดีมาก'];
  
  if (result.score < 2) {
    errors.push(`รหัสผ่าน${scoreLabels[result.score]} — ${result.feedback.warning}`);
    result.feedback.suggestions.forEach((s) => errors.push(s));
  }
  
  return {
    valid: errors.length === 0,
    score: result.score,
    scoreLabel: scoreLabels[result.score],
    crackTime: result.crack_times_display.offline_slow_hashing_1e4_per_second,
    errors,
  };
};
```

### Middleware Validation

```javascript
// middleware/validatePassword.js
const { validatePassword } = require('../utils/passwordStrength');

exports.checkPasswordStrength = (req, res, next) => {
  const { password, name, email } = req.body;
  
  // ส่ง user inputs เพื่อป้องกันการใช้ชื่อ/email เป็นรหัสผ่าน
  const userInputs = [name, email].filter(Boolean);
  const result = validatePassword(password, userInputs);
  
  if (!result.valid) {
    return res.status(400).json({
      success: false,
      message: 'รหัสผ่านไม่ปลอดภัยเพียงพอ',
      errors: result.errors,
      score: result.score,
    });
  }
  
  next();
};
```

### Frontend Password Meter

```javascript
// ส่ง score กลับ client สำหรับแสดง password strength meter
exports.checkPassword = async (req, res) => {
  const { password } = req.body;
  
  if (!password) {
    return res.status(400).json({ success: false, message: 'กรุณาระบุรหัสผ่าน' });
  }
  
  const result = validatePassword(password);
  
  res.json({
    success: true,
    score: result.score,
    scoreLabel: result.scoreLabel,
    crackTime: result.crackTime,
    suggestions: result.errors,
  });
};
```

---

## Account Lockout

### Progressive Lockout

```javascript
// models/User.js — เพิ่ม fields
const userSchema = new mongoose.Schema({
  // ... other fields ...
  
  loginAttempts: {
    type: Number,
    default: 0,
  },
  
  lockUntil: Date,
  
  loginHistory: [
    {
      ip: String,
      userAgent: String,
      success: Boolean,
      timestamp: { type: Date, default: Date.now },
    },
  ],
});

// Virtual: บัญชีถูกล็อกอยู่หรือไม่
userSchema.virtual('isLocked').get(function () {
  return !!(this.lockUntil && this.lockUntil > Date.now());
});

// Method: บันทึก login attempt
userSchema.methods.recordLoginAttempt = async function (success, ip, userAgent) {
  const MAX_ATTEMPTS = 5;
  
  // เพิ่มประวัติ
  this.loginHistory.push({ ip, userAgent, success });
  
  // เก็บแค่ 20 รายการล่าสุด
  if (this.loginHistory.length > 20) {
    this.loginHistory = this.loginHistory.slice(-20);
  }
  
  if (success) {
    // Reset เมื่อ login สำเร็จ
    this.loginAttempts = 0;
    this.lockUntil = undefined;
  } else {
    // เพิ่ม attempt count
    this.loginAttempts += 1;
    
    // Progressive lockout
    if (this.loginAttempts >= MAX_ATTEMPTS) {
      const lockMinutes = Math.min(
        Math.pow(2, this.loginAttempts - MAX_ATTEMPTS) * 5,
        1440  // สูงสุด 24 ชั่วโมง
      );
      this.lockUntil = new Date(Date.now() + lockMinutes * 60 * 1000);
    }
  }
  
  await this.save({ validateBeforeSave: false });
};
```

### Login Controller พร้อม Lockout

```javascript
// controllers/authController.js
exports.login = async (req, res) => {
  const { email, password } = req.body;
  const ip = req.ip || req.connection.remoteAddress;
  const userAgent = req.headers['user-agent'];
  
  try {
    const user = await User.findOne({ email }).select('+password');
    
    if (!user) {
      // ใช้เวลาเท่าๆ กันเพื่อป้องกัน timing attack (user enumeration)
      await new Promise((resolve) => setTimeout(resolve, 200));
      return res.status(401).json({
        success: false,
        message: 'Email หรือรหัสผ่านไม่ถูกต้อง',
      });
    }
    
    // ตรวจสอบ lockout
    if (user.isLocked) {
      const remainingMs = user.lockUntil - Date.now();
      const remainingMins = Math.ceil(remainingMs / 60000);
      
      await user.recordLoginAttempt(false, ip, userAgent);
      
      return res.status(423).json({
        success: false,
        message: `บัญชีถูกล็อกชั่วคราว กรุณารอ ${remainingMins} นาที`,
        lockedUntil: user.lockUntil,
      });
    }
    
    // Verify password
    const isMatch = await passwordService.verify(user.password, password);
    
    await user.recordLoginAttempt(isMatch, ip, userAgent);
    
    if (!isMatch) {
      const attemptsLeft = Math.max(0, 5 - user.loginAttempts);
      
      return res.status(401).json({
        success: false,
        message: 'Email หรือรหัสผ่านไม่ถูกต้อง',
        ...(attemptsLeft <= 2 && {
          warning: `เหลืออีก ${attemptsLeft} ครั้งก่อนบัญชีถูกล็อก`,
        }),
      });
    }
    
    // สำเร็จ — ออก tokens
    const { accessToken } = sendTokens(res, user);
    
    res.json({
      success: true,
      accessToken,
      user: { id: user._id, name: user.name, email: user.email, role: user.role },
    });
    
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

---

## Password Reset Flow

### Complete Reset Flow

```javascript
// controllers/authController.js

// 1. Request Reset
exports.forgotPassword = async (req, res) => {
  try {
    const { email } = req.body;
    
    const user = await User.findOne({ email });
    
    // ตอบ success เสมอ ป้องกัน email enumeration
    if (!user) {
      return res.json({
        success: true,
        message: 'ถ้า email ลงทะเบียนไว้ จะได้รับลิงก์รีเซ็ตรหัสผ่าน',
      });
    }
    
    // ตรวจสอบ rate limit
    if (
      user.passwordResetExpires &&
      user.passwordResetExpires > Date.now() - 5 * 60 * 1000
    ) {
      return res.status(429).json({
        success: false,
        message: 'กรุณารอ 5 นาทีก่อนขอ reset ใหม่',
      });
    }
    
    // สร้าง reset token
    const resetToken = user.generatePasswordResetToken();
    await user.save({ validateBeforeSave: false });
    
    // ส่ง email
    const resetURL = `${process.env.FRONTEND_URL}/reset-password/${resetToken}`;
    
    try {
      await emailService.sendPasswordReset(user.email, resetURL);
      
      res.json({
        success: true,
        message: 'ส่งลิงก์รีเซ็ตรหัสผ่านไปยัง email แล้ว',
      });
    } catch (emailError) {
      user.passwordResetToken = undefined;
      user.passwordResetExpires = undefined;
      await user.save({ validateBeforeSave: false });
      
      throw new Error('ส่ง email ไม่สำเร็จ กรุณาลองใหม่');
    }
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// 2. Validate Reset Token
exports.validateResetToken = async (req, res) => {
  try {
    const hashedToken = crypto
      .createHash('sha256')
      .update(req.params.token)
      .digest('hex');
    
    const user = await User.findOne({
      passwordResetToken: hashedToken,
      passwordResetExpires: { $gt: Date.now() },
    });
    
    if (!user) {
      return res.status(400).json({
        success: false,
        message: 'Token ไม่ถูกต้องหรือหมดอายุแล้ว',
      });
    }
    
    res.json({ success: true, message: 'Token ถูกต้อง' });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// 3. Reset Password
exports.resetPassword = async (req, res) => {
  try {
    const hashedToken = crypto
      .createHash('sha256')
      .update(req.params.token)
      .digest('hex');
    
    const user = await User.findOne({
      passwordResetToken: hashedToken,
      passwordResetExpires: { $gt: Date.now() },
    });
    
    if (!user) {
      return res.status(400).json({
        success: false,
        message: 'Token ไม่ถูกต้องหรือหมดอายุแล้ว',
      });
    }
    
    const { password, confirmPassword } = req.body;
    
    if (password !== confirmPassword) {
      return res.status(400).json({
        success: false,
        message: 'รหัสผ่านไม่ตรงกัน',
      });
    }
    
    // ตรวจสอบว่าไม่ใช่รหัสผ่านเดิม
    if (user.password) {
      const isSamePassword = await passwordService.verify(user.password, password);
      if (isSamePassword) {
        return res.status(400).json({
          success: false,
          message: 'ไม่สามารถใช้รหัสผ่านเดิมได้',
        });
      }
    }
    
    // บันทึกรหัสผ่านใหม่
    user.password = await passwordService.hash(password);
    user.passwordResetToken = undefined;
    user.passwordResetExpires = undefined;
    user.passwordChangedAt = new Date();
    user.loginAttempts = 0;  // reset lockout
    user.lockUntil = undefined;
    await user.save({ validateBeforeSave: false });
    
    // Logout จาก devices อื่น (increment token version)
    if (user.tokenVersion !== undefined) {
      await user.incrementTokenVersion();
    }
    
    // ส่ง notification email
    await emailService.sendPasswordChanged(user.email);
    
    res.json({
      success: true,
      message: 'เปลี่ยนรหัสผ่านเรียบร้อยแล้ว',
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

---

## Two-Factor Authentication

### TOTP (Time-based One-Time Password)

TOTP ใช้ algorithm เดียวกับ Google Authenticator, Authy

```bash
npm install speakeasy qrcode
```

```javascript
// services/totpService.js
const speakeasy = require('speakeasy');
const QRCode = require('qrcode');

/**
 * สร้าง TOTP secret
 */
exports.generateSecret = (userEmail, appName = 'MyApp') => {
  const secret = speakeasy.generateSecret({
    name: `${appName} (${userEmail})`,
    length: 32,
  });
  
  return {
    secret: secret.base32,      // เก็บใน database
    otpauth_url: secret.otpauth_url,  // ใช้สร้าง QR code
  };
};

/**
 * สร้าง QR code สำหรับ authenticator app
 */
exports.generateQRCode = async (otpauthUrl) => {
  return QRCode.toDataURL(otpauthUrl);
};

/**
 * ตรวจสอบ TOTP token
 */
exports.verifyToken = (secret, token) => {
  return speakeasy.totp.verify({
    secret,
    encoding: 'base32',
    token,
    window: 2,  // ±2 time steps (60 seconds tolerance)
  });
};

/**
 * Generate backup codes
 */
exports.generateBackupCodes = () => {
  const codes = [];
  for (let i = 0; i < 10; i++) {
    const code = crypto.randomBytes(4).toString('hex').toUpperCase();
    codes.push(`${code.slice(0, 4)}-${code.slice(4)}`);
  }
  return codes;
};
```

### 2FA Controllers

```javascript
// controllers/twoFactorController.js
const totpService = require('../services/totpService');
const bcrypt = require('bcryptjs');

// เริ่มต้นตั้งค่า 2FA
exports.setup2FA = async (req, res) => {
  try {
    const { secret, otpauth_url } = totpService.generateSecret(
      req.user.email,
      'MyApp'
    );
    
    // เก็บ secret ชั่วคราว (ยังไม่ enable จนกว่าจะ verify)
    req.user.twoFactorTempSecret = secret;
    await req.user.save({ validateBeforeSave: false });
    
    const qrCode = await totpService.generateQRCode(otpauth_url);
    
    res.json({
      success: true,
      qrCode,
      secret,  // แสดงให้ backup
      message: 'สแกน QR code ด้วย Authenticator app แล้วยืนยัน',
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// ยืนยันและ Enable 2FA
exports.enable2FA = async (req, res) => {
  try {
    const { token } = req.body;
    
    if (!req.user.twoFactorTempSecret) {
      return res.status(400).json({
        success: false,
        message: 'กรุณาเริ่มต้นตั้งค่า 2FA ก่อน',
      });
    }
    
    const isValid = totpService.verifyToken(req.user.twoFactorTempSecret, token);
    
    if (!isValid) {
      return res.status(400).json({
        success: false,
        message: 'รหัส OTP ไม่ถูกต้อง',
      });
    }
    
    // สร้าง backup codes
    const backupCodes = totpService.generateBackupCodes();
    const hashedBackupCodes = await Promise.all(
      backupCodes.map((code) => bcrypt.hash(code, 10))
    );
    
    req.user.twoFactorSecret = req.user.twoFactorTempSecret;
    req.user.twoFactorTempSecret = undefined;
    req.user.twoFactorEnabled = true;
    req.user.backupCodes = hashedBackupCodes;
    await req.user.save({ validateBeforeSave: false });
    
    res.json({
      success: true,
      message: 'เปิดใช้งาน 2FA แล้ว',
      backupCodes,  // แสดงครั้งเดียว บันทึกเก็บไว้
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// ตรวจสอบ 2FA ระหว่าง login
exports.verify2FA = async (req, res) => {
  try {
    const { userId, token } = req.body;
    
    // ดึง user ที่รอ 2FA verification
    const user = await User.findById(userId);
    
    if (!user || !user.twoFactorEnabled) {
      return res.status(400).json({ success: false, message: 'ไม่ถูกต้อง' });
    }
    
    let isValid = totpService.verifyToken(user.twoFactorSecret, token);
    let usedBackupCode = false;
    
    // ลอง backup codes
    if (!isValid) {
      for (let i = 0; i < user.backupCodes.length; i++) {
        const isBackup = await bcrypt.compare(token, user.backupCodes[i]);
        if (isBackup) {
          isValid = true;
          usedBackupCode = true;
          user.backupCodes.splice(i, 1);  // ลบ backup code ที่ใช้แล้ว
          await user.save({ validateBeforeSave: false });
          break;
        }
      }
    }
    
    if (!isValid) {
      return res.status(401).json({
        success: false,
        message: 'รหัส OTP ไม่ถูกต้อง',
      });
    }
    
    const { accessToken } = sendTokens(res, user);
    
    res.json({
      success: true,
      accessToken,
      ...(usedBackupCode && {
        warning: `ใช้ backup code แล้ว เหลืออีก ${user.backupCodes.length} รหัส`,
      }),
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Disable 2FA
exports.disable2FA = async (req, res) => {
  try {
    const { password } = req.body;
    
    const user = await User.findById(req.user._id).select('+password');
    const isMatch = await passwordService.verify(user.password, password);
    
    if (!isMatch) {
      return res.status(401).json({ success: false, message: 'รหัสผ่านไม่ถูกต้อง' });
    }
    
    user.twoFactorEnabled = false;
    user.twoFactorSecret = undefined;
    user.backupCodes = [];
    await user.save({ validateBeforeSave: false });
    
    res.json({ success: true, message: 'ปิดใช้งาน 2FA แล้ว' });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

### Login Flow พร้อม 2FA

```javascript
exports.login = async (req, res) => {
  // ... verify email/password ...
  
  if (user.twoFactorEnabled) {
    // ส่ง temporary state กลับ ต้อง verify 2FA ต่อ
    const tempToken = jwt.sign(
      { sub: user._id, pending2FA: true },
      process.env.JWT_ACCESS_SECRET,
      { expiresIn: '5m' }  // หมดอายุเร็ว
    );
    
    return res.json({
      success: true,
      requiresTwoFactor: true,
      tempToken,
      message: 'กรุณายืนยัน 2FA',
    });
  }
  
  // ถ้าไม่มี 2FA — login สำเร็จ
  const { accessToken } = sendTokens(res, user);
  res.json({ success: true, accessToken });
};
```

---

## Practical: Secure User Auth

### Security Checklist

```javascript
// middleware/securityHeaders.js
const helmet = require('helmet');

module.exports = helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true,
  },
});
```

```javascript
// Rate limiting
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,
  skipSuccessfulRequests: true,  // นับแค่ failed requests
  keyGenerator: (req) => `${req.ip}-${req.body.email}`,  // per email+IP
  handler: (req, res) => {
    res.status(429).json({
      success: false,
      message: 'พยายาม login มากเกินไป กรุณารอ 15 นาที',
      retryAfter: Math.ceil(req.rateLimit.resetTime / 1000),
    });
  },
});
```

---

## แบบฝึกหัด

### Exercise 1: Password History

ป้องกันการใช้รหัสผ่านเดิม:
1. เก็บ 5 รหัสผ่านล่าสุด (hashed)
2. ตรวจสอบเมื่อ change/reset password
3. แสดง error ถ้าเคยใช้แล้ว

### Exercise 2: TOTP Recovery

1. สร้าง backup codes 10 รหัส
2. แสดงครั้งเดียวหลัง enable 2FA
3. ใช้ได้ครั้งเดียว (one-time)
4. สร้าง codes ใหม่ได้ (regenerate)

### Exercise 3: Security Audit Log

บันทึกเหตุการณ์สำคัญ:
1. Login success/failure
2. Password change
3. 2FA enable/disable
4. Password reset
5. Account lockout

---

## สรุป

- **argon2id** เป็น algorithm ที่แนะนำสำหรับ password hashing
- **Salt** ป้องกัน rainbow table attacks
- **Progressive lockout** ป้องกัน brute force
- **Password reset** ต้องปลอดภัยและหมดอายุ
- **TOTP 2FA** เพิ่มชั้นป้องกัน

> **บทถัดไป:** Part 26 — File Uploads
