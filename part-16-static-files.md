# Part 16: Static Files
## ขั้นตอนที่ 16-6 จาก 1000

---

## สารบัญ
1. express.static()
2. Virtual Paths
3. Multiple Static Directories
4. Caching Strategies
5. CDN Integration
6. Practical: Static Website with Express
7. Exercise

---

## 1. express.static()

`express.static()` เป็น built-in middleware สำหรับ serve static files เช่น CSS, JavaScript, รูปภาพ, fonts

### การใช้งานพื้นฐาน

```javascript
const express = require('express');
const path = require('path');
const app = express();

// Serve ไฟล์จากโฟลเดอร์ 'public'
app.use(express.static('public'));

// แนะนำให้ใช้ path.join เพื่อ cross-platform compatibility
app.use(express.static(path.join(__dirname, 'public')));
```

### โครงสร้างไฟล์

```
project/
├── public/
│   ├── css/
│   │   ├── style.css
│   │   └── bootstrap.min.css
│   ├── js/
│   │   ├── main.js
│   │   └── jquery.min.js
│   ├── images/
│   │   ├── logo.png
│   │   └── banner.jpg
│   ├── fonts/
│   │   └── custom-font.woff2
│   └── index.html
└── app.js
```

เมื่อตั้งค่า `express.static('public')` แล้ว:
- `/public/css/style.css` → เข้าถึงที่ `/css/style.css`
- `/public/js/main.js` → เข้าถึงที่ `/js/main.js`
- `/public/index.html` → เข้าถึงที่ `/index.html` หรือ `/`

### Options ของ express.static()

```javascript
app.use(express.static('public', {
  // ===== Caching =====
  maxAge: '1d',           // cache นาน 1 วัน (string)
  // หรือ
  maxAge: 86400000,       // cache นาน 86400000 ms (1 วัน)
  
  // ===== ETag =====
  etag: true,             // เปิด ETag (default: true)
  
  // ===== Last-Modified =====
  lastModified: true,     // ส่ง Last-Modified header (default: true)
  
  // ===== Default File =====
  index: 'index.html',    // ไฟล์ default เมื่อเข้า directory (default: index.html)
  // index: false,        // ปิด directory index
  
  // ===== Dot Files =====
  dotfiles: 'ignore',     // ignore, allow, หรือ deny
  
  // ===== Redirect =====
  redirect: true,         // redirect trailing slash (default: true)
  
  // ===== Headers =====
  setHeaders: (res, path, stat) => {
    // เพิ่ม headers เอง
    if (path.endsWith('.jpg') || path.endsWith('.png')) {
      res.set('Cache-Control', 'public, max-age=31536000'); // 1 ปี
    }
  }
}));
```

---

## 2. Virtual Paths

Virtual path คือการกำหนด URL prefix ที่แตกต่างจากโครงสร้างไฟล์จริง:

```javascript
// ไฟล์อยู่ที่ /public/images/logo.png
// แต่เข้าถึงที่ /static/images/logo.png
app.use('/static', express.static('public'));

// ตัวอย่าง:
// /static/css/style.css → public/css/style.css
// /static/js/main.js   → public/js/main.js
// /static/logo.png      → public/logo.png
```

### ประโยชน์ของ Virtual Paths

```javascript
// 1. ป้องกัน path ชน
app.use('/vendor', express.static('node_modules'));
// /vendor/bootstrap/dist/css/bootstrap.min.css
// → node_modules/bootstrap/dist/css/bootstrap.min.css

// 2. Version-based caching
app.use('/v1.0.0', express.static('public'));
// เมื่อ deploy เวอร์ชันใหม่ เปลี่ยนเป็น /v1.1.0
// Browser จะ download ไฟล์ใหม่เพราะ URL เปลี่ยน

// 3. Separate public assets
app.use('/assets', express.static('assets'));
app.use('/uploads', express.static('uploads'));
```

---

## 3. Multiple Static Directories

Express รองรับการตั้งค่า static directories หลายแห่ง:

```javascript
const express = require('express');
const path = require('path');
const app = express();

// ลำดับสำคัญ! Express หาไฟล์จาก directory แรกที่กำหนดก่อน

// 1. Public assets
app.use(express.static(path.join(__dirname, 'public')));

// 2. Vendor libraries
app.use(express.static(path.join(__dirname, 'vendor')));

// 3. Uploads
app.use('/uploads', express.static(path.join(__dirname, 'uploads')));

// 4. Node modules (ถ้าต้องการ serve บาง packages)
app.use('/vendor', express.static(path.join(__dirname, 'node_modules')));
```

### ตัวอย่างจริง: E-commerce Static Assets

```javascript
const express = require('express');
const path = require('path');
const app = express();

// Public assets (CSS, JS, fonts)
app.use(express.static(path.join(__dirname, 'public'), {
  maxAge: '7d',
  etag: true
}));

// Product images (เปลี่ยนบ่อย)
app.use('/product-images', express.static(
  path.join(__dirname, 'uploads/products'),
  { maxAge: '1d' }
));

// User avatars
app.use('/avatars', express.static(
  path.join(__dirname, 'uploads/avatars'),
  { maxAge: '1d' }
));

// Documents / PDFs
app.use('/docs', express.static(
  path.join(__dirname, 'uploads/documents'),
  {
    maxAge: '0', // ไม่ cache เพราะเปลี่ยนบ่อย
    setHeaders: (res, filePath) => {
      if (filePath.endsWith('.pdf')) {
        // บังคับ download แทน preview
        // res.set('Content-Disposition', 'attachment');
        // หรือ allow preview
        res.set('Content-Type', 'application/pdf');
      }
    }
  }
));
```

---

## 4. Caching Strategies

### HTTP Caching Headers

```javascript
const express = require('express');
const path = require('path');
const app = express();

// ==============================
// Strategy 1: Long-term Caching (Static Assets)
// เหมาะกับ: CSS, JS ที่มี content hash ในชื่อ
// เช่น: main.a3f4b2c1.css
// ==============================
app.use('/static', express.static('public/dist', {
  maxAge: '1y', // cache 1 ปี
  immutable: true // บอก browser ว่าไฟล์ไม่เปลี่ยน
}));

// ==============================
// Strategy 2: Medium Caching (Images)
// ==============================
app.use('/images', express.static('public/images', {
  maxAge: '30d',
  setHeaders: (res, filePath) => {
    // Image ขนาดใหญ่ cache นานกว่า
    const stat = require('fs').statSync(filePath);
    if (stat.size > 100 * 1024) { // > 100KB
      res.set('Cache-Control', 'public, max-age=2592000'); // 30 วัน
    }
  }
}));

// ==============================
// Strategy 3: No Cache (Dynamic)
// เหมาะกับ: HTML, API responses
// ==============================
app.use('/uploads', express.static('uploads', {
  setHeaders: (res) => {
    res.set('Cache-Control', 'no-cache, no-store, must-revalidate');
    res.set('Pragma', 'no-cache');
    res.set('Expires', '0');
  }
}));

// ==============================
// Strategy 4: Conditional (Revalidation)
// ==============================
app.use('/assets', express.static('assets', {
  etag: true,           // ETag-based validation
  lastModified: true,   // Last-Modified-based validation
  maxAge: '0'          // ต้อง revalidate ทุกครั้ง แต่ใช้ cache ได้ถ้า not modified
}));
```

### Custom Cache Headers

```javascript
// Middleware จัดการ cache ตาม file type
app.use(express.static('public', {
  setHeaders: (res, filePath) => {
    const ext = path.extname(filePath).toLowerCase();
    
    const cacheRules = {
      '.html': 'no-cache',
      '.css': 'public, max-age=86400',      // 1 วัน
      '.js': 'public, max-age=86400',       // 1 วัน
      '.jpg': 'public, max-age=2592000',    // 30 วัน
      '.jpeg': 'public, max-age=2592000',
      '.png': 'public, max-age=2592000',
      '.gif': 'public, max-age=2592000',
      '.webp': 'public, max-age=2592000',
      '.svg': 'public, max-age=604800',     // 7 วัน
      '.woff': 'public, max-age=31536000',  // 1 ปี
      '.woff2': 'public, max-age=31536000',
      '.ico': 'public, max-age=604800'      // 7 วัน
    };
    
    const cacheControl = cacheRules[ext] || 'public, max-age=3600';
    res.set('Cache-Control', cacheControl);
  }
}));
```

### Cache Busting

```javascript
// วิธีที่ 1: Timestamp query string (ง่ายแต่ไม่ดีนัก)
app.locals.assetVersion = Date.now();

// ใน EJS template:
// <link rel="stylesheet" href="/css/style.css?v=<%= assetVersion %>">

// วิธีที่ 2: Content Hash (แนะนำ)
// ใช้ webpack, vite หรือ gulp สร้าง hash ในชื่อไฟล์
// main.a3f4b2c1.js

// วิธีที่ 3: Version prefix
const APP_VERSION = require('./package.json').version;
app.use(`/v${APP_VERSION}`, express.static('public/dist', {
  maxAge: '1y'
}));
```

---

## 5. CDN Integration

### การใช้ CDN สำหรับ Library

```html
<!-- แทนที่จะ serve จาก server เอง -->
<!-- ใช้ CDN แทน -->

<!-- Bootstrap -->
<link rel="stylesheet" 
      href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css"
      integrity="sha384-..."
      crossorigin="anonymous">

<!-- Font Awesome -->
<link rel="stylesheet" 
      href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">

<!-- jQuery -->
<script src="https://code.jquery.com/jquery-3.7.0.min.js"></script>

<!-- Chart.js -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
```

### CDN สำหรับ Assets ของตัวเอง

```javascript
// config/cdn.js
const CDN_URL = process.env.CDN_URL || '';

const cdnUrl = (path) => {
  if (CDN_URL) {
    return `${CDN_URL}${path}`;
  }
  return path;
};

module.exports = { cdnUrl };
```

```javascript
// app.js
const { cdnUrl } = require('./config/cdn');

// ส่ง cdnUrl helper ไปยัง templates
app.use((req, res, next) => {
  res.locals.cdnUrl = cdnUrl;
  next();
});
```

```html
<!-- ใน EJS template -->
<link rel="stylesheet" href="<%= cdnUrl('/css/style.css') %>">
<img src="<%= cdnUrl('/images/logo.png') %>" alt="Logo">
```

### Nginx Reverse Proxy สำหรับ Static Files

```nginx
# nginx.conf
server {
  listen 80;
  server_name example.com;

  # Static files - nginx serve แทน Node.js
  location /static/ {
    root /var/www/myapp/public;
    expires 30d;
    add_header Cache-Control "public, max-age=2592000";
    
    # Gzip compression
    gzip on;
    gzip_types text/css application/javascript image/svg+xml;
  }

  # API routes - forward ไป Node.js
  location /api/ {
    proxy_pass http://localhost:3000;
  }

  # Everything else - forward ไป Node.js
  location / {
    proxy_pass http://localhost:3000;
  }
}
```

---

## 6. Practical: Static Website with Express

### สร้าง Static Website สมบูรณ์

```
static-website/
├── public/
│   ├── css/
│   │   ├── style.css
│   │   └── responsive.css
│   ├── js/
│   │   ├── main.js
│   │   └── animations.js
│   ├── images/
│   │   ├── logo.svg
│   │   ├── hero.jpg
│   │   └── about.jpg
│   ├── fonts/
│   │   └── custom.woff2
│   └── favicon.ico
├── views/
│   ├── partials/
│   │   ├── head.ejs
│   │   ├── header.ejs
│   │   └── footer.ejs
│   ├── index.ejs
│   ├── about.ejs
│   ├── services.ejs
│   └── contact.ejs
└── app.js
```

### public/css/style.css

```css
/* style.css */
:root {
  --primary: #2563eb;
  --secondary: #64748b;
  --dark: #1e293b;
  --light: #f8fafc;
  --white: #ffffff;
}

*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: 'Sarabun', 'Segoe UI', sans-serif;
  color: var(--dark);
  line-height: 1.6;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 1rem;
}

/* Navbar */
.navbar {
  background: var(--white);
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  position: sticky;
  top: 0;
  z-index: 100;
}

.navbar-inner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem;
}

.navbar-brand {
  font-size: 1.5rem;
  font-weight: 700;
  color: var(--primary);
  text-decoration: none;
}

.navbar-links {
  display: flex;
  gap: 2rem;
  list-style: none;
}

.navbar-links a {
  color: var(--secondary);
  text-decoration: none;
  font-weight: 500;
  transition: color 0.2s;
}

.navbar-links a:hover,
.navbar-links a.active {
  color: var(--primary);
}

/* Hero */
.hero {
  background: linear-gradient(135deg, var(--primary), #7c3aed);
  color: var(--white);
  padding: 6rem 1rem;
  text-align: center;
}

.hero h1 {
  font-size: 3rem;
  margin-bottom: 1rem;
}

.hero p {
  font-size: 1.25rem;
  opacity: 0.9;
  max-width: 600px;
  margin: 0 auto 2rem;
}

/* Buttons */
.btn {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.75rem 1.5rem;
  border-radius: 0.5rem;
  font-weight: 600;
  text-decoration: none;
  cursor: pointer;
  transition: all 0.2s;
  border: none;
}

.btn-primary {
  background: var(--primary);
  color: var(--white);
}

.btn-primary:hover {
  background: #1d4ed8;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(37, 99, 235, 0.4);
}

.btn-outline {
  border: 2px solid var(--primary);
  color: var(--primary);
  background: transparent;
}

.btn-outline:hover {
  background: var(--primary);
  color: var(--white);
}

/* Cards */
.card {
  background: var(--white);
  border-radius: 1rem;
  box-shadow: 0 4px 6px rgba(0,0,0,0.07);
  overflow: hidden;
  transition: transform 0.2s, box-shadow 0.2s;
}

.card:hover {
  transform: translateY(-4px);
  box-shadow: 0 10px 20px rgba(0,0,0,0.1);
}

/* Grid */
.grid-3 {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 2rem;
}

/* Section */
.section {
  padding: 5rem 1rem;
}

.section-title {
  font-size: 2rem;
  text-align: center;
  margin-bottom: 3rem;
  color: var(--dark);
}
```

### app.js - Static Website Server

```javascript
// app.js
const express = require('express');
const expressLayouts = require('express-ejs-layouts');
const path = require('path');
const compression = require('compression');
const helmet = require('helmet');

const app = express();
const PORT = process.env.PORT || 3000;

// ==============================
// Security & Compression
// ==============================
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
      fontSrc: ["'self'", "https://fonts.gstatic.com"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:"]
    }
  }
}));

// Gzip compression
app.use(compression());

// ==============================
// View Engine
// ==============================
app.set('view engine', 'ejs');
app.set('views', path.join(__dirname, 'views'));
app.use(expressLayouts);
app.set('layout', 'layouts/main');

// ==============================
// Static Files
// ==============================

// Main public folder
app.use(express.static(path.join(__dirname, 'public'), {
  maxAge: process.env.NODE_ENV === 'production' ? '7d' : 0,
  etag: true,
  lastModified: true,
  setHeaders: (res, filePath) => {
    // Long cache สำหรับ fonts
    if (filePath.includes('/fonts/')) {
      res.set('Cache-Control', 'public, max-age=31536000');
    }
    
    // Security headers สำหรับ downloads
    if (filePath.endsWith('.pdf')) {
      res.set('Content-Disposition', 'inline');
    }
  }
}));

// ==============================
// Global Variables
// ==============================
app.use((req, res, next) => {
  res.locals.siteName = 'My Company';
  res.locals.currentYear = new Date().getFullYear();
  res.locals.currentPath = req.path;
  next();
});

// ==============================
// Routes
// ==============================
app.get('/', (req, res) => {
  res.render('index', {
    title: 'หน้าแรก',
    description: 'ยินดีต้อนรับสู่เว็บไซต์ของเรา',
    features: [
      { icon: 'rocket', title: 'รวดเร็ว', desc: 'โหลดเร็ว ใช้งานง่าย' },
      { icon: 'shield', title: 'ปลอดภัย', desc: 'ระบบความปลอดภัยสูง' },
      { icon: 'heart', title: 'น่าเชื่อถือ', desc: 'บริการที่ไว้วางใจได้' }
    ]
  });
});

app.get('/about', (req, res) => {
  res.render('about', {
    title: 'เกี่ยวกับเรา',
    description: 'รู้จักทีมงานของเรา',
    team: [
      { name: 'สมชาย ใจดี', role: 'CEO', image: '/images/team1.jpg' },
      { name: 'สมหญิง ใจงาม', role: 'CTO', image: '/images/team2.jpg' },
      { name: 'วิชาย เก่งมาก', role: 'Designer', image: '/images/team3.jpg' }
    ]
  });
});

app.get('/services', (req, res) => {
  res.render('services', {
    title: 'บริการ',
    services: [
      {
        icon: 'code',
        title: 'Web Development',
        description: 'พัฒนาเว็บไซต์ด้วย technology ใหม่ล่าสุด',
        price: 'เริ่มต้น ฿50,000'
      },
      {
        icon: 'mobile',
        title: 'Mobile App',
        description: 'สร้าง app ทั้ง iOS และ Android',
        price: 'เริ่มต้น ฿150,000'
      }
    ]
  });
});

app.get('/contact', (req, res) => {
  res.render('contact', {
    title: 'ติดต่อเรา',
    contactInfo: {
      email: 'contact@company.com',
      phone: '02-xxx-xxxx',
      address: 'กรุงเทพมหานคร 10xxx'
    }
  });
});

// ==============================
// 404 Handler
// ==============================
app.use((req, res) => {
  res.status(404).render('404', {
    title: '404 - ไม่พบหน้าที่ต้องการ'
  });
});

app.listen(PORT, () => {
  console.log(`Static Website: http://localhost:${PORT}`);
});
```

### Serve Static Files อย่างมีประสิทธิภาพ

```javascript
// Middleware สำหรับ monitoring static file requests
app.use('/static', (req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    if (duration > 100) { // Log ถ้าช้าเกิน 100ms
      console.warn(`Slow static file: ${req.url} (${duration}ms)`);
    }
  });
  
  next();
}, express.static('public'));
```

---

## 7. Exercise

### Exercise 1: Asset Versioning

สร้างระบบ asset versioning อัตโนมัติ:
- อ่าน hash ของ CSS/JS files จากไฟล์
- สร้าง URL ที่มี hash เช่น `/css/style.abc123.css`
- เมื่อไฟล์เปลี่ยน hash เปลี่ยน → browser download ใหม่

### Exercise 2: Image Optimization Middleware

สร้าง middleware ที่:
- ตรวจสอบว่า browser รองรับ WebP
- ถ้ารองรับ → serve `.webp` แทน `.jpg`
- ถ้าไม่รองรับ → serve `.jpg` ปกติ

### Exercise 3: Download Manager

สร้าง endpoint สำหรับจัดการ file downloads:
- บันทึกจำนวน downloads
- ตรวจสอบสิทธิ์ก่อน download
- ส่ง proper Content-Disposition header
- รองรับ resume download (Range requests)

---

## สรุป Part 16

ในบทนี้เราได้เรียนรู้:

1. **express.static()** - serve static files ด้วย built-in middleware
2. **Virtual Paths** - กำหนด URL prefix ที่ต่างจากโครงสร้างไฟล์
3. **Multiple Directories** - ตั้งค่าหลาย static directories
4. **Caching** - strategies ต่างๆ สำหรับ performance
5. **CDN** - การใช้ CDN สำหรับ assets
6. **Practical** - สร้าง static website ที่ complete

---

*Part 16 | Node.js/Express.js Course | ขั้นตอนที่ 16 จาก 1000*
