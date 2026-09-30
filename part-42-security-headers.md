# Part 42: Security Headers ด้วย Helmet.js

> ขั้นตอนที่ 42-42 จาก 1000

---

## สารบัญ

1. [Security Headers คืออะไร](#security-headers-คืออะไร)
2. [Helmet.js](#helmetjs)
3. [Content Security Policy (CSP)](#content-security-policy)
4. [HTTP Strict Transport Security (HSTS)](#hsts)
5. [X-Frame-Options](#x-frame-options)
6. [อื่นๆ Security Headers](#อื่นๆ-security-headers)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Security Headers คืออะไร

Security Headers เป็น HTTP response headers ที่บอก browser ว่าควร behave อย่างไร เพื่อป้องกัน attacks ต่างๆ

```
HTTP/1.1 200 OK
Content-Security-Policy: default-src 'self'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000; includeSubDomains
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

### ตรวจสอบ Security Headers

```bash
# ใช้ curl
curl -I https://example.com

# ใช้ securityheaders.com
# https://securityheaders.com/?q=example.com

# ใช้ Mozilla Observatory
# https://observatory.mozilla.org/
```

---

## Helmet.js

Helmet เป็น Express middleware ที่ตั้งค่า security headers ให้อัตโนมัติ

```bash
npm install helmet
```

### Basic Usage

```javascript
const express = require('express');
const helmet = require('helmet');

const app = express();

// ใช้ helmet ด้วย defaults ที่ดี
app.use(helmet());

// Helmet default headers:
// Content-Security-Policy
// Cross-Origin-Opener-Policy
// Cross-Origin-Resource-Policy
// Origin-Agent-Cluster
// Referrer-Policy
// Strict-Transport-Security
// X-Content-Type-Options
// X-DNS-Prefetch-Control
// X-Download-Options
// X-Frame-Options
// X-Permitted-Cross-Domain-Policies
// X-Powered-By (ลบออก)
// X-XSS-Protection
```

### Custom Configuration

```javascript
app.use(helmet({
  // Content Security Policy
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'", "https://cdn.example.com"],
      styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
      imgSrc: ["'self'", "data:", "https:"],
      fontSrc: ["'self'", "https://fonts.gstatic.com"],
      connectSrc: ["'self'", "https://api.example.com"],
      frameSrc: ["'none'"],
      objectSrc: ["'none'"],
      upgradeInsecureRequests: [],
    },
  },
  
  // HSTS
  strictTransportSecurity: {
    maxAge: 31536000,        // 1 year
    includeSubDomains: true,
    preload: true,
  },
  
  // X-Frame-Options
  frameguard: {
    action: 'deny',
  },
  
  // X-Content-Type-Options: nosniff
  noSniff: true,
  
  // Referrer-Policy
  referrerPolicy: {
    policy: 'strict-origin-when-cross-origin',
  },
  
  // Cross-Origin settings
  crossOriginEmbedderPolicy: false, // disable ถ้า conflict กับ third-party
  crossOriginOpenerPolicy: { policy: 'same-origin' },
  crossOriginResourcePolicy: { policy: 'same-site' },
  
  // Permissions Policy
  permittedCrossDomainPolicies: false,
  
  // ปิด X-Powered-By
  hidePoweredBy: true,
}));
```

---

## Content Security Policy

CSP เป็น security layer ที่ป้องกัน XSS attacks โดยบอก browser ว่า resources อะไรที่โหลดได้

### CSP Directives

```javascript
// contentSecurityPolicy directives
{
  // Default policy สำหรับทุกอย่างที่ไม่ได้ระบุ
  defaultSrc: ["'self'"],
  
  // Scripts
  scriptSrc: [
    "'self'",
    "'unsafe-inline'",      // อนุญาต inline scripts (ไม่แนะนำ)
    "'unsafe-eval'",        // อนุญาต eval() (ไม่แนะนำมาก)
    "https://cdn.jsdelivr.net",
    "'nonce-randomValue'",  // อนุญาต inline script ที่มี nonce
    "'sha256-abc123...'",   // อนุญาต inline script ที่มี hash นี้
  ],
  
  // Styles
  styleSrc: [
    "'self'",
    "'unsafe-inline'",
    "https://fonts.googleapis.com",
  ],
  
  // Images
  imgSrc: [
    "'self'",
    "data:",               // data URIs
    "blob:",               // Blob URLs
    "https:",              // HTTPS images ทั้งหมด
  ],
  
  // Fonts
  fontSrc: [
    "'self'",
    "https://fonts.gstatic.com",
  ],
  
  // API connections, WebSockets
  connectSrc: [
    "'self'",
    "wss://api.example.com",  // WebSocket
    "https://api.example.com",
  ],
  
  // Media (audio/video)
  mediaSrc: ["'self'"],
  
  // iframes
  frameSrc: ["'none'"],
  childSrc: ["'none'"],
  
  // Form actions
  formAction: ["'self'"],
  
  // Base URI
  baseUri: ["'self'"],
  
  // Prevent loading Flash/ActiveX
  objectSrc: ["'none'"],
  
  // Upgrade HTTP to HTTPS
  upgradeInsecureRequests: [],
  
  // Report violations
  reportUri: ["/csp-report"],  // deprecated
  reportTo: ["csp-endpoint"],
}
```

### CSP Nonce

```javascript
// สร้าง nonce สำหรับ inline scripts
const crypto = require('crypto');

app.use((req, res, next) => {
  // สร้าง nonce ใหม่ทุก request
  res.locals.nonce = crypto.randomBytes(16).toString('base64');
  next();
});

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      scriptSrc: [
        "'self'",
        (req, res) => `'nonce-${res.locals.nonce}'`, // dynamic nonce
      ],
    },
  },
}));

// ใน template
app.get('/', (req, res) => {
  res.send(`
    <html>
      <head>
        <!-- Script ที่มี nonce เท่านั้นที่จะทำงาน -->
        <script nonce="${res.locals.nonce}">
          // This works!
          console.log('Trusted script');
        </script>
        
        <!-- Script นี้จะถูก block -->
        <script>
          console.log('Untrusted script - BLOCKED!');
        </script>
      </head>
    </html>
  `);
});
```

### CSP Report-Only Mode

```javascript
// ทดสอบ CSP โดยไม่บล็อก (แค่ report)
app.use(helmet({
  contentSecurityPolicy: false, // ปิด CSP normal
}));

// เพิ่ม Content-Security-Policy-Report-Only header
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy-Report-Only',
    "default-src 'self'; script-src 'self'; report-uri /csp-report"
  );
  next();
});

// Endpoint รับ CSP violations
app.post('/csp-report', express.json({ type: 'application/csp-report' }), (req, res) => {
  const report = req.body['csp-report'];
  console.error('CSP Violation:', {
    violatedDirective: report['violated-directive'],
    blockedUri: report['blocked-uri'],
    documentUri: report['document-uri'],
  });
  
  res.status(204).end();
});
```

---

## HSTS

HTTP Strict Transport Security บังคับให้ browser ใช้ HTTPS เสมอ

```javascript
app.use(helmet.strictTransportSecurity({
  // เก็บ HSTS ไว้ใน browser นานแค่ไหน (seconds)
  maxAge: 31536000,        // 1 year (แนะนำ)
  
  // รวม subdomains ด้วย
  includeSubDomains: true,
  
  // ขอให้ browser preload list รวม domain นี้ด้วย
  // ต้องสมัครที่ hstspreload.org ก่อน
  preload: true,
}));

// Header ที่ได้:
// Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

### HSTS Preload

```bash
# สมัคร HSTS Preload ที่
# https://hstspreload.org/

# Requirements:
# 1. ต้องมี valid SSL certificate
# 2. Redirect HTTP → HTTPS
# 3. max-age >= 1 year
# 4. includeSubDomains
# 5. preload directive
```

---

## X-Frame-Options

ป้องกัน Clickjacking attacks โดยบอก browser ว่าหน้าเว็บนี้ฝังใน iframe ได้หรือไม่

```javascript
// ไม่อนุญาตใน iframe เลย
app.use(helmet.frameguard({ action: 'deny' }));

// อนุญาตเฉพาะ same origin
app.use(helmet.frameguard({ action: 'sameorigin' }));

// อนุญาต specific origin (เก่า - ใช้ CSP frame-ancestors แทน)
app.use(helmet.frameguard({
  action: 'allow-from',
  domain: 'https://trusted-site.com',
}));

// ด้วย CSP (แนะนำกว่า)
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      frameAncestors: ["'none'"],                    // deny
      frameAncestors: ["'self'"],                    // sameorigin
      frameAncestors: ["https://trusted-site.com"], // specific
    },
  },
  frameguard: false, // ปิด X-Frame-Options ถ้าใช้ CSP แทน
}));
```

---

## อื่นๆ Security Headers

### X-Content-Type-Options

```javascript
// ป้องกัน MIME-type sniffing
app.use(helmet.noSniff());
// Adds: X-Content-Type-Options: nosniff
```

### Referrer-Policy

```javascript
app.use(helmet.referrerPolicy({
  policy: 'strict-origin-when-cross-origin',
  // Options:
  // no-referrer             - ไม่ส่ง Referer header เลย
  // no-referrer-when-downgrade - ส่งเมื่อ HTTPS→HTTPS, ไม่ส่ง HTTPS→HTTP
  // same-origin             - ส่งเฉพาะ same origin
  // strict-origin           - ส่ง origin เท่านั้น (ไม่มี path)
  // strict-origin-when-cross-origin - (แนะนำ)
  // unsafe-url              - ส่งทุกอย่าง (ไม่แนะนำ)
}));
```

### Permissions-Policy

```javascript
// ควบคุม browser features ที่ page ใช้ได้
app.use((req, res, next) => {
  res.setHeader('Permissions-Policy', [
    'camera=()',              // ปิด camera
    'microphone=()',          // ปิด microphone
    'geolocation=()',         // ปิด geolocation
    'payment=(self)',         // payment API เฉพาะ same-origin
    'accelerometer=()',
    'gyroscope=()',
    'magnetometer=()',
    'usb=()',
    'fullscreen=(self)',      // fullscreen เฉพาะ self
  ].join(', '));
  next();
});
```

### Complete Security Headers Setup

```javascript
// config/security.js
const helmet = require('helmet');

function applySecurityHeaders(app) {
  const isDev = process.env.NODE_ENV === 'development';
  const isProd = process.env.NODE_ENV === 'production';
  
  app.use(helmet({
    // CSP - ปรับตามความต้องการ
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: [
          "'self'",
          ...(isDev ? ["'unsafe-inline'", "'unsafe-eval'"] : []),
          "https://cdn.jsdelivr.net",
        ],
        styleSrc: [
          "'self'",
          "'unsafe-inline'",
          "https://fonts.googleapis.com",
        ],
        imgSrc: ["'self'", "data:", "https:"],
        fontSrc: ["'self'", "https://fonts.gstatic.com"],
        connectSrc: [
          "'self'",
          ...(isDev ? ["ws://localhost:*", "http://localhost:*"] : []),
        ],
        frameSrc: ["'none'"],
        objectSrc: ["'none'"],
        baseUri: ["'self'"],
        formAction: ["'self'"],
        upgradeInsecureRequests: isProd ? [] : undefined,
      },
      reportOnly: isDev, // ใน dev แค่ report ไม่ block
    },
    
    // HSTS - เฉพาะ production
    strictTransportSecurity: isProd ? {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    } : false,
    
    // Frame protection
    frameguard: { action: 'deny' },
    
    // เอา X-Powered-By ออก
    hidePoweredBy: true,
    
    // ป้องกัน MIME sniffing
    noSniff: true,
    
    // Referrer
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
    
    // Cross-Origin
    crossOriginEmbedderPolicy: false, // อาจต้อง disable
    crossOriginOpenerPolicy: { policy: 'same-origin' },
    crossOriginResourcePolicy: { policy: 'same-site' },
  }));
  
  // Permissions-Policy
  app.use((req, res, next) => {
    res.setHeader('Permissions-Policy',
      'camera=(), microphone=(), geolocation=(), payment=(self)'
    );
    next();
  });
}

module.exports = { applySecurityHeaders };
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Helmet Setup

ติดตั้ง helmet ด้วย configuration ที่เหมาะสมสำหรับ:
- Blog API (public, read-only)
- Admin Dashboard (authenticated)
- E-commerce checkout (payment)

### แบบฝึกหัดที่ 2: CSP Testing

1. ตั้งค่า CSP Report-Only
2. สร้างหน้าเว็บที่มี inline script
3. ดู violations ใน console
4. แก้ไขโดยใช้ nonce หรือ hash

### แบบฝึกหัดที่ 3: Security Scan

1. Deploy app ไป production
2. สแกนด้วย securityheaders.com
3. แก้ไข headers ให้ได้ rating A+

---

## สรุป

Security Headers เป็น defense layer ที่สำคัญ

| Header | ป้องกัน |
|--------|---------|
| CSP | XSS, data injection |
| HSTS | Downgrade attacks, MITM |
| X-Frame-Options | Clickjacking |
| X-Content-Type-Options | MIME sniffing |
| Referrer-Policy | Data leakage |
| Permissions-Policy | Browser feature abuse |

**ถัดไป**: [Part 43: SQL Injection →](./part-43-sql-injection.md)
