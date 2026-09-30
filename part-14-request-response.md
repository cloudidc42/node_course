# Part 14: Request และ Response Objects
## ขั้นตอนที่ 14-4 จาก 1000

---

## สารบัญ
1. Request Object ทุก Property
2. Response Object ทุก Method
3. req.body, req.params, req.query, req.headers
4. res.json(), res.send(), res.redirect()
5. File Uploads
6. Cookies
7. Practical: Full Request/Response Handling
8. Exercise

---

## 1. Request Object ทุก Property

`req` (Request object) คือ object ที่มีข้อมูลทั้งหมดเกี่ยวกับ HTTP request:

### Properties หลักของ req

```javascript
app.use((req, res, next) => {
  // ==============================
  // URL และ Path
  // ==============================
  console.log(req.url);         // '/users/123?sort=asc'
  console.log(req.path);        // '/users/123' (ไม่รวม query string)
  console.log(req.originalUrl); // URL เดิมก่อน middleware เปลี่ยน
  console.log(req.baseUrl);     // base path ที่ router mount อยู่
  
  // ==============================
  // HTTP Method
  // ==============================
  console.log(req.method);      // 'GET', 'POST', 'PUT', etc.
  
  // ==============================
  // Route Parameters
  // ==============================
  console.log(req.params);      // { id: '123' }
  
  // ==============================
  // Query String
  // ==============================
  console.log(req.query);       // { sort: 'asc', page: '1' }
  
  // ==============================
  // Request Body (ต้องมี body-parser)
  // ==============================
  console.log(req.body);        // { name: 'John', email: 'john@example.com' }
  
  // ==============================
  // Headers
  // ==============================
  console.log(req.headers);     // ทุก headers
  console.log(req.headers['content-type']); // 'application/json'
  console.log(req.get('Content-Type'));      // ใช้ method นี้ดีกว่า
  console.log(req.header('Accept'));         // alias ของ req.get()
  
  // ==============================
  // Network Info
  // ==============================
  console.log(req.ip);          // '127.0.0.1'
  console.log(req.ips);         // ['127.0.0.1'] เมื่อผ่าน proxy
  console.log(req.hostname);    // 'localhost'
  console.log(req.protocol);    // 'http' หรือ 'https'
  console.log(req.secure);      // true ถ้าเป็น HTTPS
  console.log(req.subdomains);  // ['api'] สำหรับ api.example.com
  
  // ==============================
  // Content Type
  // ==============================
  console.log(req.is('json'));        // true/false
  console.log(req.is('text/html'));   // true/false
  console.log(req.is(['json', 'html'])); // 'json' หรือ 'html' หรือ false
  
  // ==============================
  // Accepts (Content Negotiation)
  // ==============================
  console.log(req.accepts('html'));          // 'html' หรือ false
  console.log(req.accepts(['html', 'json'])); // 'html' หรือ 'json'
  console.log(req.acceptsCharsets('utf-8'));
  console.log(req.acceptsEncodings('gzip'));
  console.log(req.acceptsLanguages('th', 'en'));
  
  // ==============================
  // Cookies (ต้องใช้ cookie-parser)
  // ==============================
  console.log(req.cookies);         // { sessionId: 'abc123' }
  console.log(req.signedCookies);   // signed cookies
  
  // ==============================
  // Authentication (custom properties)
  // ==============================
  console.log(req.user);      // เพิ่มโดย auth middleware
  console.log(req.isAuthenticated()); // เพิ่มโดย passport.js
  
  // ==============================
  // Fresh / Stale
  // ==============================
  console.log(req.fresh);     // true ถ้า response ยัง fresh (cache)
  console.log(req.stale);     // ตรงข้ามกับ fresh
  
  // ==============================
  // XHR
  // ==============================
  console.log(req.xhr);       // true ถ้าเป็น AJAX request
  
  next();
});
```

### req.params - Route Parameters

```javascript
// Route: /users/:userId/posts/:postId
app.get('/users/:userId/posts/:postId', (req, res) => {
  console.log(req.params);
  // { userId: '5', postId: '10' }
  
  const userId = Number(req.params.userId);   // แปลงเป็น number
  const postId = Number(req.params.postId);
  
  res.json({ userId, postId });
});

// Optional segment (ต้องสร้าง 2 routes)
app.get('/posts', (req, res) => {
  res.json({ allPosts: true });
});

app.get('/posts/:id', (req, res) => {
  res.json({ postId: req.params.id });
});
```

### req.query - Query Parameters

```javascript
// URL: /search?q=express&lang=th&page=2&limit=20&tags[]=node&tags[]=web
app.get('/search', (req, res) => {
  console.log(req.query);
  // {
  //   q: 'express',
  //   lang: 'th',
  //   page: '2',
  //   limit: '20',
  //   tags: ['node', 'web']  // Express แปลง array ให้อัตโนมัติ
  // }
  
  // แปลงค่าเสมอ เพราะทุกค่าเป็น string
  const page = parseInt(req.query.page) || 1;
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);
  
  res.json({ q: req.query.q, page, limit });
});
```

---

## 2. Response Object ทุก Method

`res` (Response object) ใช้สำหรับส่งข้อมูลกลับไปยัง client:

### res.send()

```javascript
// ส่งข้อมูลหลายรูปแบบ
app.get('/demo', (req, res) => {
  // String → Content-Type: text/html
  res.send('<h1>Hello HTML</h1>');
  
  // Buffer → Content-Type: application/octet-stream
  // res.send(Buffer.from('Hello'));
  
  // Object/Array → Content-Type: application/json
  // res.send({ key: 'value' });
  
  // Number → HTTP Status Code (deprecated, ใช้ res.sendStatus แทน)
  // res.send(404);
});
```

### res.json()

```javascript
app.get('/json', (req, res) => {
  // ส่ง JSON response
  res.json({
    success: true,
    data: { id: 1, name: 'Test' },
    message: 'OK'
  });
  
  // JSON null
  // res.json(null);
  
  // JSON array
  // res.json([1, 2, 3]);
  
  // JSON string
  // res.json('string');
  
  // JSON number
  // res.json(42);
});

// ตั้งค่า JSON formatting
app.set('json spaces', 2); // indent 2 spaces สำหรับ development
```

### res.status()

```javascript
app.get('/status-demo', (req, res) => {
  // กำหนด status code และ chaining
  res.status(201).json({ message: 'Created' });
  res.status(400).json({ error: 'Bad Request' });
  res.status(401).json({ error: 'Unauthorized' });
  res.status(403).json({ error: 'Forbidden' });
  res.status(404).json({ error: 'Not Found' });
  res.status(422).json({ error: 'Unprocessable Entity' });
  res.status(429).json({ error: 'Too Many Requests' });
  res.status(500).json({ error: 'Internal Server Error' });
});

// res.sendStatus() - ส่ง status code พร้อม default message
app.get('/send-status', (req, res) => {
  res.sendStatus(200); // "OK"
  res.sendStatus(404); // "Not Found"
  res.sendStatus(500); // "Internal Server Error"
});
```

### res.set() / res.header()

```javascript
app.get('/headers', (req, res) => {
  // ตั้งค่า header เดียว
  res.set('Content-Type', 'application/json');
  res.set('X-Custom-Header', 'MyValue');
  
  // ตั้งค่าหลาย headers พร้อมกัน
  res.set({
    'Cache-Control': 'no-cache, no-store',
    'X-Frame-Options': 'DENY',
    'X-Content-Type-Options': 'nosniff',
    'X-Request-ID': req.id || 'unknown'
  });
  
  // alias
  res.header('X-Another', 'value');
  
  // ดู header ที่ตั้งไว้
  console.log(res.get('Content-Type'));
  
  res.json({ message: 'Headers set' });
});
```

### res.redirect()

```javascript
app.get('/old-path', (req, res) => {
  // 302 Temporary Redirect (default)
  res.redirect('/new-path');
  
  // 301 Permanent Redirect
  res.redirect(301, '/permanent-new-path');
  
  // Redirect ไป URL ภายนอก
  res.redirect('https://www.google.com');
  
  // Redirect กลับ (Referer)
  res.redirect('back');
});

// ตัวอย่างจริง: URL shortener
const urlMap = {
  'abc123': 'https://www.example.com/very-long-url',
  'xyz789': 'https://www.another-site.com/page'
};

app.get('/s/:code', (req, res) => {
  const longUrl = urlMap[req.params.code];
  
  if (!longUrl) {
    return res.status(404).send('Short URL not found');
  }
  
  // Redirect พร้อมบันทึก clicks
  console.log(`Redirect: ${req.params.code} → ${longUrl}`);
  res.redirect(302, longUrl);
});
```

### res.render()

```javascript
app.set('view engine', 'ejs');
app.set('views', './views');

app.get('/page', (req, res) => {
  // Render template พร้อมส่ง data
  res.render('index', {
    title: 'My Page',
    user: { name: 'สมชาย' },
    items: ['item1', 'item2', 'item3'],
    isLoggedIn: true
  });
});
```

### res.sendFile()

```javascript
const path = require('path');

app.get('/download', (req, res) => {
  const filePath = path.join(__dirname, 'files', 'document.pdf');
  
  res.sendFile(filePath, (err) => {
    if (err) {
      console.error('Error sending file:', err);
      res.status(500).send('Error downloading file');
    }
  });
});
```

### res.download()

```javascript
app.get('/download-report', (req, res) => {
  const filePath = path.join(__dirname, 'reports', 'monthly.pdf');
  
  // ส่งไฟล์ให้ client ดาวน์โหลด
  res.download(filePath, 'monthly-report.pdf', (err) => {
    if (err) {
      res.status(500).send('Error downloading file');
    }
  });
});
```

### res.end()

```javascript
app.get('/end', (req, res) => {
  // จบ response โดยไม่ส่งข้อมูล
  res.end();
  
  // ส่ง data แล้วจบ
  res.end('Some data');
  
  // ใช้สำหรับ HEAD requests
  res.status(200).end();
});
```

### res.locals

```javascript
// res.locals ใช้ pass data ไปยัง templates
app.use((req, res, next) => {
  // ตั้งค่า locals ที่ใช้ทุกหน้า
  res.locals.currentUser = req.user;
  res.locals.siteName = 'My Website';
  res.locals.year = new Date().getFullYear();
  next();
});

app.get('/about', (req, res) => {
  res.render('about');
  // ใน template สามารถใช้ currentUser, siteName, year ได้เลย
});
```

---

## 3. req.body, req.params, req.query, req.headers

### เปรียบเทียบ req.body vs req.params vs req.query

```javascript
// URL: POST /users/123/posts?notify=true
// Body: { "title": "My Post", "content": "Hello" }

app.post('/users/:userId/posts', (req, res) => {
  // req.params - ค่าใน URL path
  console.log(req.params.userId);   // '123'
  
  // req.query - query string ใน URL
  console.log(req.query.notify);    // 'true'
  
  // req.body - ข้อมูลใน request body (ต้อง middleware)
  console.log(req.body.title);      // 'My Post'
  console.log(req.body.content);    // 'Hello'
  
  res.json({
    params: req.params,
    query: req.query,
    body: req.body
  });
});
```

### req.headers ทุก header ที่ควรรู้

```javascript
app.use((req, res, next) => {
  // Standard Headers
  console.log(req.headers['content-type']);    // 'application/json'
  console.log(req.headers['accept']);           // 'text/html,application/xhtml+xml'
  console.log(req.headers['accept-language']); // 'th,en-US;q=0.9'
  console.log(req.headers['accept-encoding']); // 'gzip, deflate'
  console.log(req.headers['user-agent']);      // 'Mozilla/5.0...'
  console.log(req.headers['host']);            // 'localhost:3000'
  console.log(req.headers['referer']);         // หน้าที่มาจาก
  console.log(req.headers['origin']);          // origin ของ request
  console.log(req.headers['connection']);      // 'keep-alive'
  console.log(req.headers['cache-control']);   // 'no-cache'
  
  // Authentication
  console.log(req.headers['authorization']);   // 'Bearer eyJhbG...'
  console.log(req.headers['x-api-key']);       // 'my-api-key'
  
  // Custom Headers
  console.log(req.headers['x-request-id']);   // 'uuid'
  console.log(req.headers['x-forwarded-for']); // IP ผ่าน proxy
  
  // ใช้ req.get() method แทนได้ (case-insensitive)
  console.log(req.get('Content-Type'));        // แนะนำวิธีนี้
  console.log(req.get('Authorization'));
  
  next();
});
```

---

## 4. res.json(), res.send(), res.redirect()

### เปรียบเทียบ res.send() vs res.json()

```javascript
// res.send() - versatile
app.get('/send', (req, res) => {
  // ส่ง string
  res.send('Hello World');
  // Content-Type: text/html; charset=utf-8
  
  // ส่ง object (auto convert to JSON)
  // res.send({ hello: 'world' });
  // Content-Type: application/json; charset=utf-8
  
  // ส่ง Buffer
  // res.send(Buffer.from('nope'));
  // Content-Type: application/octet-stream
});

// res.json() - specific for JSON
app.get('/json', (req, res) => {
  // ส่ง JSON เสมอ
  res.json({ hello: 'world' });
  // Content-Type: application/json; charset=utf-8
  
  // แปลง special values ที่ res.send() ทำไม่ได้
  res.json(null);
  res.json(undefined);  // ส่ง null
  res.json(NaN);        // ส่ง null
  res.json(Infinity);   // ส่ง null
});
```

### Redirect Patterns

```javascript
const express = require('express');
const app = express();

// ==============================
// Redirect ทั่วไป
// ==============================

// Trailing slash redirect
app.get('/about/', (req, res) => {
  res.redirect(301, '/about'); // ถาวร
});

// HTTP → HTTPS redirect middleware
app.use((req, res, next) => {
  if (!req.secure && process.env.NODE_ENV === 'production') {
    return res.redirect(301, `https://${req.hostname}${req.url}`);
  }
  next();
});

// ==============================
// Login redirect
// ==============================
app.get('/dashboard', (req, res) => {
  if (!req.user) {
    // บันทึก URL ที่ต้องการไป แล้ว redirect ไป login
    return res.redirect(`/login?returnUrl=${encodeURIComponent(req.url)}`);
  }
  res.render('dashboard');
});

app.post('/login', (req, res) => {
  // หลัง login สำเร็จ
  const returnUrl = req.query.returnUrl || '/dashboard';
  
  // ตรวจสอบ returnUrl ว่าเป็น path ในเว็บไซต์ (ป้องกัน open redirect)
  if (!returnUrl.startsWith('/')) {
    return res.redirect('/dashboard');
  }
  
  res.redirect(returnUrl);
});
```

---

## 5. File Uploads

### ติดตั้ง Multer

```bash
npm install multer
```

### การใช้งาน Multer

```javascript
const express = require('express');
const multer = require('multer');
const path = require('path');
const app = express();

// ==============================
// Memory Storage (เก็บใน RAM)
// ==============================
const memoryUpload = multer({ storage: multer.memoryStorage() });

app.post('/upload-memory', memoryUpload.single('file'), (req, res) => {
  if (!req.file) {
    return res.status(400).json({ error: 'ไม่พบไฟล์' });
  }
  
  console.log('File info:', req.file);
  // {
  //   fieldname: 'file',
  //   originalname: 'test.jpg',
  //   encoding: '7bit',
  //   mimetype: 'image/jpeg',
  //   buffer: <Buffer ...>,
  //   size: 12345
  // }
  
  res.json({
    message: 'อัปโหลดสำเร็จ',
    filename: req.file.originalname,
    size: req.file.size
  });
});

// ==============================
// Disk Storage (เก็บลงไฟล์)
// ==============================
const diskStorage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, 'uploads/'); // โฟลเดอร์ปลายทาง
  },
  filename: (req, file, cb) => {
    // สร้างชื่อไฟล์ใหม่ เพื่อป้องกันชื่อซ้ำ
    const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1e9);
    const ext = path.extname(file.originalname);
    cb(null, file.fieldname + '-' + uniqueSuffix + ext);
  }
});

// ==============================
// File Filter
// ==============================
const imageFilter = (req, file, cb) => {
  const allowedTypes = ['image/jpeg', 'image/jpg', 'image/png', 'image/gif', 'image/webp'];
  
  if (allowedTypes.includes(file.mimetype)) {
    cb(null, true);  // ยอมรับไฟล์
  } else {
    cb(new Error('อนุญาตเฉพาะไฟล์รูปภาพ (jpg, png, gif, webp)'), false);
  }
};

// ==============================
// Upload Configuration
// ==============================
const upload = multer({
  storage: diskStorage,
  fileFilter: imageFilter,
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB
    files: 5                    // สูงสุด 5 ไฟล์
  }
});

// Single file
app.post('/upload-single', upload.single('avatar'), (req, res) => {
  if (!req.file) {
    return res.status(400).json({ error: 'ไม่พบไฟล์' });
  }
  
  res.json({
    message: 'อัปโหลดสำเร็จ',
    file: {
      name: req.file.filename,
      originalName: req.file.originalname,
      size: req.file.size,
      path: `/uploads/${req.file.filename}`,
      mimetype: req.file.mimetype
    }
  });
});

// Multiple files (ชื่อ field เดียวกัน)
app.post('/upload-multiple', upload.array('photos', 5), (req, res) => {
  if (!req.files || req.files.length === 0) {
    return res.status(400).json({ error: 'ไม่พบไฟล์' });
  }
  
  const filesInfo = req.files.map(f => ({
    name: f.filename,
    originalName: f.originalname,
    size: f.size,
    url: `/uploads/${f.filename}`
  }));
  
  res.json({
    message: `อัปโหลดสำเร็จ ${req.files.length} ไฟล์`,
    files: filesInfo
  });
});

// Multiple fields
app.post('/upload-fields', 
  upload.fields([
    { name: 'avatar', maxCount: 1 },
    { name: 'gallery', maxCount: 10 }
  ]),
  (req, res) => {
    res.json({
      avatar: req.files['avatar']?.[0]?.filename,
      gallery: req.files['gallery']?.map(f => f.filename)
    });
  }
);

// Error handling สำหรับ multer
app.use((err, req, res, next) => {
  if (err instanceof multer.MulterError) {
    if (err.code === 'LIMIT_FILE_SIZE') {
      return res.status(400).json({ error: 'ไฟล์ใหญ่เกินไป (สูงสุด 5MB)' });
    }
    if (err.code === 'LIMIT_FILE_COUNT') {
      return res.status(400).json({ error: 'จำนวนไฟล์มากเกินไป' });
    }
  }
  next(err);
});

// Serve uploaded files
app.use('/uploads', express.static('uploads'));

app.listen(3000);
```

---

## 6. Cookies

### cookie-parser

```bash
npm install cookie-parser
```

### การใช้งาน Cookies

```javascript
const express = require('express');
const cookieParser = require('cookie-parser');
const app = express();

// ใช้ secret สำหรับ signed cookies
app.use(cookieParser('my-super-secret-key'));

// ==============================
// ตั้งค่า Cookie
// ==============================
app.get('/set-cookie', (req, res) => {
  // Cookie พื้นฐาน
  res.cookie('username', 'somchai');
  
  // Cookie พร้อม options
  res.cookie('sessionId', 'abc123xyz', {
    maxAge: 24 * 60 * 60 * 1000,  // หมดอายุใน 1 วัน (milliseconds)
    httpOnly: true,                 // JavaScript เข้าไม่ถึง
    secure: process.env.NODE_ENV === 'production', // HTTPS only
    sameSite: 'strict',             // CSRF protection
    domain: 'example.com',          // domain ที่ใช้ได้
    path: '/'                       // path ที่ใช้ได้
  });
  
  // Signed cookie (ป้องกันการแก้ไข)
  res.cookie('userId', '12345', { signed: true });
  
  res.json({ message: 'Cookie ถูกตั้งค่าแล้ว' });
});

// ==============================
// อ่าน Cookie
// ==============================
app.get('/get-cookie', (req, res) => {
  // อ่าน cookies ทั้งหมด
  console.log(req.cookies);        // { username: 'somchai', sessionId: 'abc123xyz' }
  
  // อ่าน cookie เดียว
  const username = req.cookies.username;
  const sessionId = req.cookies.sessionId;
  
  // อ่าน signed cookies
  const userId = req.signedCookies.userId;
  
  res.json({ username, sessionId, userId });
});

// ==============================
// ลบ Cookie
// ==============================
app.get('/clear-cookie', (req, res) => {
  res.clearCookie('username');
  res.clearCookie('sessionId');
  res.clearCookie('userId');
  
  res.json({ message: 'Cookie ถูกลบแล้ว' });
});

app.listen(3000);
```

---

## 7. Practical: Full Request/Response Handling

### สร้าง API ที่จัดการ Request/Response อย่างครบถ้วน

```javascript
// app.js - Full Request/Response Demo
const express = require('express');
const cookieParser = require('cookie-parser');
const multer = require('multer');
const path = require('path');
const fs = require('fs');
const app = express();

// Ensure uploads directory exists
if (!fs.existsSync('uploads')) {
  fs.mkdirSync('uploads');
}

// Middleware
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser('secret-key-for-signing'));
app.use('/uploads', express.static('uploads'));

// ==============================
// Request Inspector
// ==============================
app.all('/inspect', (req, res) => {
  res.json({
    method: req.method,
    url: req.url,
    path: req.path,
    params: req.params,
    query: req.query,
    body: req.body,
    headers: req.headers,
    cookies: req.cookies,
    ip: req.ip,
    protocol: req.protocol,
    secure: req.secure,
    hostname: req.hostname,
    xhr: req.xhr,
    accepts: {
      html: req.accepts('html'),
      json: req.accepts('json'),
      xml: req.accepts('xml')
    }
  });
});

// ==============================
// Content Negotiation
// ==============================
const users = [
  { id: 1, name: 'สมชาย', email: 'somchai@example.com' }
];

app.get('/users-cn', (req, res) => {
  // ตอบสนองตามที่ client ต้องการ
  if (req.accepts('json')) {
    return res.json(users);
  }
  
  if (req.accepts('html')) {
    const html = users
      .map(u => `<div><strong>${u.name}</strong> - ${u.email}</div>`)
      .join('');
    return res.send(`<html><body>${html}</body></html>`);
  }
  
  if (req.accepts('text')) {
    const text = users.map(u => `${u.name}: ${u.email}`).join('\n');
    return res.type('text/plain').send(text);
  }
  
  res.status(406).send('ไม่รองรับ format ที่ต้องการ');
});

// ==============================
// Streaming Response
// ==============================
app.get('/stream', (req, res) => {
  res.setHeader('Content-Type', 'text/plain');
  res.setHeader('Transfer-Encoding', 'chunked');
  
  let count = 0;
  const interval = setInterval(() => {
    count++;
    res.write(`ข้อมูลส่วนที่ ${count}\n`);
    
    if (count >= 5) {
      clearInterval(interval);
      res.end('Stream สิ้นสุดแล้ว\n');
    }
  }, 1000);
  
  // จัดการเมื่อ client ตัดการเชื่อมต่อ
  req.on('close', () => {
    clearInterval(interval);
    console.log('Client disconnected');
  });
});

// ==============================
// File Upload with Response
// ==============================
const upload = multer({
  storage: multer.diskStorage({
    destination: 'uploads/',
    filename: (req, file, cb) => {
      cb(null, `${Date.now()}-${file.originalname}`);
    }
  }),
  limits: { fileSize: 5 * 1024 * 1024 }
});

app.post('/upload', upload.single('file'), (req, res) => {
  if (!req.file) {
    return res.status(400).json({ error: 'กรุณาเลือกไฟล์' });
  }
  
  // ตอบกลับพร้อมข้อมูลไฟล์
  res.status(201)
    .set('X-File-ID', req.file.filename)
    .json({
      success: true,
      file: {
        id: req.file.filename,
        originalName: req.file.originalname,
        size: req.file.size,
        mimetype: req.file.mimetype,
        url: `${req.protocol}://${req.hostname}:3000/uploads/${req.file.filename}`
      }
    });
});

// ==============================
// Cookie Demo
// ==============================
app.get('/login-demo', (req, res) => {
  // จำลองการ login
  const user = { id: 1, name: 'สมชาย', role: 'user' };
  
  // ตั้งค่า session cookie
  res.cookie('userId', user.id.toString(), {
    signed: true,
    httpOnly: true,
    maxAge: 3600000 // 1 ชั่วโมง
  });
  
  res.json({
    success: true,
    message: 'Login สำเร็จ',
    user: { id: user.id, name: user.name }
  });
});

app.get('/me-demo', (req, res) => {
  const userId = req.signedCookies.userId;
  
  if (!userId) {
    return res.status(401).json({ error: 'กรุณา login ก่อน' });
  }
  
  res.json({ userId, message: 'ข้อมูลผู้ใช้' });
});

// Start server
app.listen(3000, () => {
  console.log('Full Request/Response Demo: http://localhost:3000');
});
```

---

## 8. Exercise

### Exercise 1: Request Analyzer

สร้าง middleware ที่วิเคราะห์ request และเพิ่มข้อมูลใน `req`:
- `req.isAjax` - ตรวจสอบว่าเป็น AJAX request
- `req.isMobile` - ตรวจสอบจาก User-Agent
- `req.language` - ดึงภาษาหลักจาก Accept-Language
- `req.requestedAt` - เวลาที่ request เข้ามา

### Exercise 2: File Upload API

สร้าง API สำหรับ upload และจัดการไฟล์:
```
POST /api/files/upload    - อัปโหลดไฟล์
GET  /api/files           - ดูรายการไฟล์ทั้งหมด
GET  /api/files/:id       - ดูข้อมูลไฟล์
DELETE /api/files/:id     - ลบไฟล์
```

### Exercise 3: Cookie-based Cart

สร้าง Shopping Cart ที่เก็บข้อมูลใน cookie:
- เพิ่มสินค้าใน cart
- ดูสินค้าใน cart
- ลบสินค้าออกจาก cart
- ล้าง cart ทั้งหมด

---

## สรุป Part 14

ในบทนี้เราได้เรียนรู้:

1. **Request Object** - ทุก property ที่มีประโยชน์
2. **Response Object** - ทุก method สำหรับส่ง response
3. **req ต่างๆ** - body, params, query, headers
4. **res ต่างๆ** - json(), send(), redirect()
5. **File Uploads** - multer สำหรับจัดการไฟล์
6. **Cookies** - cookie-parser และ signed cookies

---

*Part 14 | Node.js/Express.js Course | ขั้นตอนที่ 14 จาก 1000*
