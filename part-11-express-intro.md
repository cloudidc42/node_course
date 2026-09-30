# Part 11: Express.js เบื้องต้น
## ขั้นตอนที่ 11-1 จาก 1000

---

## สารบัญ
1. Express.js คืออะไร?
2. ติดตั้งและ Setup
3. Hello World Express
4. Application Structure
5. app.listen, app.get, app.post
6. Request/Response Objects Overview
7. Practical: สร้าง Basic Express Server
8. Exercise

---

## 1. Express.js คืออะไร?

Express.js เป็น **web application framework** สำหรับ Node.js ที่ได้รับความนิยมมากที่สุด ถูกสร้างขึ้นโดย TJ Holowaychuk ในปี 2010 และปัจจุบันดูแลโดย OpenJS Foundation

### ทำไมต้องใช้ Express?

Node.js มี HTTP module ในตัวที่สามารถสร้าง web server ได้ แต่มันมีข้อจำกัดหลายอย่าง:

**Node.js HTTP แบบ native (ยุ่งยาก):**
```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  // ต้องจัดการ routing เอง
  if (req.url === '/' && req.method === 'GET') {
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end('<h1>Hello World</h1>');
  } else if (req.url === '/about' && req.method === 'GET') {
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end('<h1>About Page</h1>');
  } else {
    res.writeHead(404, { 'Content-Type': 'text/html' });
    res.end('<h1>404 Not Found</h1>');
  }
});

server.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

**Express.js (สะอาดกว่ามาก):**
```javascript
const express = require('express');
const app = express();

// กำหนด route ได้ชัดเจน
app.get('/', (req, res) => {
  res.send('<h1>Hello World</h1>');
});

app.get('/about', (req, res) => {
  res.send('<h1>About Page</h1>');
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

### ความสามารถของ Express.js

Express.js ช่วยให้เราทำสิ่งต่อไปนี้ได้ง่ายขึ้น:

1. **Routing** - จัดการ URL และ HTTP methods อย่างเป็นระบบ
2. **Middleware** - เพิ่มความสามารถให้แอปพลิเคชัน
3. **Template Engines** - render HTML ฝั่ง server
4. **Static Files** - serve ไฟล์ CSS, JS, รูปภาพ
5. **Error Handling** - จัดการข้อผิดพลาดอย่างมีประสิทธิภาพ
6. **RESTful APIs** - สร้าง API ได้ง่าย

### Express เปรียบเทียบกับ Framework อื่น

| Feature | Express | Fastify | Koa | NestJS |
|---------|---------|---------|-----|--------|
| ความเร็ว | ดี | ดีมาก | ดี | ปานกลาง |
| ง่ายต่อการเรียนรู้ | ง่ายมาก | ง่าย | ปานกลาง | ยาก |
| Ecosystem | ใหญ่มาก | กำลังเติบโต | ปานกลาง | ใหญ่ |
| TypeScript Support | ต้อง config | ดี | ต้อง config | ดีเยี่ยม |
| Middleware | Connect-style | Fastify-style | async/await | Decorators |

---

## 2. ติดตั้งและ Setup

### ความต้องการของระบบ

ก่อนเริ่มต้น ต้องมี:
- Node.js เวอร์ชัน 14.x หรือสูงกว่า (แนะนำ LTS)
- npm หรือ yarn

ตรวจสอบ version:
```bash
node --version  # ควรได้ v14.x.x หรือสูงกว่า
npm --version   # ควรได้ 6.x.x หรือสูงกว่า
```

### สร้าง Project ใหม่

```bash
# สร้างโฟลเดอร์ใหม่
mkdir my-express-app
cd my-express-app

# เริ่มต้น npm project
npm init -y

# ติดตั้ง Express
npm install express

# ติดตั้ง dev dependencies (optional แต่แนะนำ)
npm install --save-dev nodemon
```

### โครงสร้าง package.json

หลังจากติดตั้ง, `package.json` จะมีลักษณะดังนี้:

```json
{
  "name": "my-express-app",
  "version": "1.0.0",
  "description": "My first Express application",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

### nodemon คืออะไร?

`nodemon` เป็นเครื่องมือที่คอย monitor การเปลี่ยนแปลงของไฟล์ และ restart server อัตโนมัติเมื่อมีการแก้ไขโค้ด ทำให้ไม่ต้อง restart manual ทุกครั้ง

```bash
# รันด้วย node (ต้อง restart manual)
node index.js

# รันด้วย nodemon (restart อัตโนมัติ)
npx nodemon index.js
# หรือถ้าตั้ง script แล้ว
npm run dev
```

---

## 3. Hello World Express

### สร้างไฟล์ index.js

```javascript
// index.js
// นำเข้า Express module
const express = require('express');

// สร้าง Express application instance
const app = express();

// กำหนด PORT
const PORT = 3000;

// สร้าง route สำหรับ GET / (root path)
app.get('/', (req, res) => {
  res.send('Hello World! ยินดีต้อนรับสู่ Express.js');
});

// เริ่มต้น server และ listen บน port ที่กำหนด
app.listen(PORT, () => {
  console.log(`🚀 Server กำลังทำงานที่ http://localhost:${PORT}`);
});
```

### ทดสอบ Server

```bash
# รัน server
node index.js

# เปิด browser ไปที่ http://localhost:3000
# หรือใช้ curl
curl http://localhost:3000
```

### ทำความเข้าใจโค้ด

```javascript
// 1. require('express') - โหลด Express module
const express = require('express');

// 2. express() - สร้าง application instance
//    app มี methods หลายอย่างเช่น get(), post(), listen() เป็นต้น
const app = express();

// 3. app.get(path, callback) - กำหนด route สำหรับ GET request
//    path = '/' คือ URL path
//    callback = function ที่รับ req (request) และ res (response)
app.get('/', (req, res) => {
  // 4. res.send() - ส่ง response กลับไปยัง client
  res.send('Hello World!');
});

// 5. app.listen(port, callback) - เริ่ม server
app.listen(3000, () => {
  console.log('Server is running');
});
```

### การตอบสนองรูปแบบต่างๆ

```javascript
const express = require('express');
const app = express();

// ส่ง plain text
app.get('/text', (req, res) => {
  res.send('นี่คือ plain text');
});

// ส่ง HTML
app.get('/html', (req, res) => {
  res.send('<h1>นี่คือ HTML</h1><p>Express ส่ง HTML ได้ด้วย</p>');
});

// ส่ง JSON
app.get('/json', (req, res) => {
  res.json({
    message: 'นี่คือ JSON response',
    success: true,
    data: {
      name: 'Express.js',
      version: '4.x'
    }
  });
});

// ส่งพร้อม status code
app.get('/status', (req, res) => {
  res.status(201).json({
    message: 'Created successfully'
  });
});

app.listen(3000);
```

---

## 4. Application Structure

### โครงสร้างพื้นฐาน (Simple)

```
my-express-app/
├── node_modules/
├── public/
│   ├── css/
│   ├── js/
│   └── images/
├── views/
│   └── index.ejs
├── routes/
│   ├── index.js
│   └── users.js
├── middleware/
│   └── auth.js
├── index.js         (หรือ app.js / server.js)
└── package.json
```

### โครงสร้าง MVC (สำหรับ project ขนาดกลาง-ใหญ่)

```
my-express-app/
├── node_modules/
├── public/
│   ├── css/
│   │   └── style.css
│   ├── js/
│   │   └── main.js
│   └── images/
├── src/
│   ├── controllers/
│   │   ├── userController.js
│   │   └── productController.js
│   ├── models/
│   │   ├── User.js
│   │   └── Product.js
│   ├── routes/
│   │   ├── userRoutes.js
│   │   └── productRoutes.js
│   ├── middleware/
│   │   ├── auth.js
│   │   └── errorHandler.js
│   ├── utils/
│   │   └── helpers.js
│   └── config/
│       └── database.js
├── views/
│   ├── layouts/
│   │   └── main.ejs
│   └── pages/
│       ├── home.ejs
│       └── about.ejs
├── tests/
├── .env
├── .gitignore
├── app.js
└── package.json
```

### ตัวอย่าง app.js หลัก

```javascript
// app.js - จุดเริ่มต้นของแอปพลิเคชัน
const express = require('express');
const path = require('path');

// นำเข้า routes
const indexRoutes = require('./src/routes/indexRoutes');
const userRoutes = require('./src/routes/userRoutes');
const productRoutes = require('./src/routes/productRoutes');

// นำเข้า middleware
const errorHandler = require('./src/middleware/errorHandler');
const logger = require('./src/middleware/logger');

const app = express();
const PORT = process.env.PORT || 3000;

// ====== Middleware Setup ======
// แปลง JSON body
app.use(express.json());
// แปลง URL-encoded form data
app.use(express.urlencoded({ extended: true }));
// Serve static files
app.use(express.static(path.join(__dirname, 'public')));
// Custom logger
app.use(logger);

// ====== View Engine Setup ======
app.set('view engine', 'ejs');
app.set('views', path.join(__dirname, 'views'));

// ====== Routes Setup ======
app.use('/', indexRoutes);
app.use('/api/users', userRoutes);
app.use('/api/products', productRoutes);

// ====== Error Handler ======
app.use(errorHandler);

// ====== Start Server ======
app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});

module.exports = app; // สำหรับ testing
```

---

## 5. app.listen, app.get, app.post

### app.listen()

`app.listen()` เริ่มต้น HTTP server:

```javascript
// รูปแบบพื้นฐาน
app.listen(port);

// พร้อม callback
app.listen(port, callback);

// พร้อม hostname
app.listen(port, hostname, callback);

// ตัวอย่าง
app.listen(3000, () => {
  console.log('Server is running on port 3000');
});

// ใช้ PORT จาก environment variable
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

// รับค่า server instance
const server = app.listen(3000, () => {
  const { address, port } = server.address();
  console.log(`Listening at http://${address}:${port}`);
});

// ปิด server (ใช้ใน testing)
server.close(() => {
  console.log('Server closed');
});
```

### app.get()

`app.get()` จัดการ HTTP GET requests:

```javascript
// รูปแบบ
app.get(path, [middlewares], handler);

// ตัวอย่างพื้นฐาน
app.get('/', (req, res) => {
  res.send('Home Page');
});

// พร้อม path parameter
app.get('/users/:id', (req, res) => {
  const { id } = req.params;
  res.send(`User ID: ${id}`);
});

// พร้อม query string
app.get('/search', (req, res) => {
  const { q, page } = req.query;
  res.json({ query: q, page: page || 1 });
});

// หลาย middleware
app.get('/dashboard', 
  checkAuth,          // middleware ตรวจสอบ authentication
  checkPermission,    // middleware ตรวจสอบ permission
  (req, res) => {     // route handler
    res.render('dashboard');
  }
);

// app.get() สำหรับดึงค่า application setting
const viewEngine = app.get('view engine'); // ดึงค่า setting
```

### app.post()

`app.post()` จัดการ HTTP POST requests:

```javascript
// ตัวอย่างพื้นฐาน
app.post('/users', (req, res) => {
  const { name, email } = req.body;
  // สร้าง user ใหม่
  res.status(201).json({
    message: 'User created',
    user: { name, email }
  });
});

// สร้าง login endpoint
app.post('/login', (req, res) => {
  const { username, password } = req.body;
  
  // ตรวจสอบ credentials (ตัวอย่าง)
  if (username === 'admin' && password === 'password') {
    res.json({ success: true, message: 'Login successful' });
  } else {
    res.status(401).json({ success: false, message: 'Invalid credentials' });
  }
});
```

### HTTP Methods อื่นๆ

```javascript
const express = require('express');
const app = express();
app.use(express.json());

// GET - ดึงข้อมูล
app.get('/products', (req, res) => {
  res.json([{ id: 1, name: 'Product 1' }]);
});

// POST - สร้างข้อมูลใหม่
app.post('/products', (req, res) => {
  const product = req.body;
  // บันทึกลงฐานข้อมูล...
  res.status(201).json({ message: 'Created', product });
});

// PUT - อัปเดตข้อมูลทั้งหมด
app.put('/products/:id', (req, res) => {
  const { id } = req.params;
  const product = req.body;
  // อัปเดตทั้ง record...
  res.json({ message: `Product ${id} updated`, product });
});

// PATCH - อัปเดตข้อมูลบางส่วน
app.patch('/products/:id', (req, res) => {
  const { id } = req.params;
  const updates = req.body;
  // อัปเดตเฉพาะ fields ที่ส่งมา...
  res.json({ message: `Product ${id} partially updated` });
});

// DELETE - ลบข้อมูล
app.delete('/products/:id', (req, res) => {
  const { id } = req.params;
  // ลบจากฐานข้อมูล...
  res.json({ message: `Product ${id} deleted` });
});

// ALL - รับทุก HTTP method
app.all('/test', (req, res) => {
  res.send(`Received ${req.method} request`);
});

app.listen(3000);
```

---

## 6. Request/Response Objects Overview

### Request Object (req)

`req` object มี properties และ methods ที่มีประโยชน์:

```javascript
app.get('/example/:id', (req, res) => {
  // URL Parameters (/example/123)
  console.log(req.params);       // { id: '123' }
  console.log(req.params.id);    // '123'
  
  // Query String (/example/123?sort=asc&page=2)
  console.log(req.query);        // { sort: 'asc', page: '2' }
  console.log(req.query.sort);   // 'asc'
  
  // Request Body (ต้องใช้ middleware แปลง)
  console.log(req.body);         // { name: 'John', email: 'john@example.com' }
  
  // Headers
  console.log(req.headers);      // ทุก header
  console.log(req.headers['content-type']); // 'application/json'
  console.log(req.get('Content-Type'));      // วิธีที่ดีกว่า
  
  // HTTP Method
  console.log(req.method);       // 'GET'
  
  // URL และ Path
  console.log(req.url);          // '/example/123?sort=asc'
  console.log(req.path);         // '/example/123'
  console.log(req.originalUrl);  // URL เดิมก่อน middleware เปลี่ยน
  
  // Protocol และ Host
  console.log(req.protocol);     // 'http' หรือ 'https'
  console.log(req.hostname);     // 'localhost'
  console.log(req.ip);           // IP ของ client
  
  // ตรวจสอบ content type
  console.log(req.is('json'));    // true ถ้า Content-Type เป็น JSON
  
  // ตรวจสอบ accepts
  console.log(req.accepts('html')); // 'html' ถ้า client ยอมรับ HTML
  
  res.send('OK');
});
```

### Response Object (res)

```javascript
app.get('/response-demo', (req, res) => {
  // ส่ง plain text หรือ HTML
  // res.send('Hello World');
  
  // ส่ง JSON
  // res.json({ key: 'value' });
  
  // กำหนด status code
  // res.status(404).send('Not Found');
  
  // กำหนด header
  res.set('X-Custom-Header', 'MyValue');
  res.set({
    'Content-Type': 'application/json',
    'X-Another-Header': 'Value'
  });
  
  // Redirect
  // res.redirect('/new-path');
  // res.redirect(301, '/permanent-redirect');
  
  // Render template
  // res.render('index', { title: 'My Page' });
  
  // Download file
  // res.download('/path/to/file.pdf');
  
  // Send file
  // res.sendFile(path.join(__dirname, 'public', 'index.html'));
  
  // End response
  // res.end(); // ไม่ส่งข้อมูล
  
  res.json({ status: 'ok' });
});
```

---

## 7. Practical: สร้าง Basic Express Server

มาสร้าง Express server ที่ทำงานได้จริง สำหรับ "Simple Task Manager":

### โครงสร้างไฟล์

```
task-manager/
├── public/
│   └── style.css
├── app.js
└── package.json
```

### app.js - Complete Basic Server

```javascript
// app.js
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

// ==============================
// Middleware
// ==============================
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Middleware บันทึก logs
app.use((req, res, next) => {
  const timestamp = new Date().toISOString();
  console.log(`[${timestamp}] ${req.method} ${req.url}`);
  next(); // ไปยัง middleware ถัดไป
});

// ==============================
// "Database" (in-memory)
// ==============================
let tasks = [
  { id: 1, title: 'เรียน Express.js', completed: false, priority: 'high' },
  { id: 2, title: 'สร้าง REST API', completed: false, priority: 'medium' },
  { id: 3, title: 'ทดสอบ middleware', completed: true, priority: 'low' }
];
let nextId = 4;

// ==============================
// Routes
// ==============================

// Home page
app.get('/', (req, res) => {
  res.send(`
    <!DOCTYPE html>
    <html lang="th">
    <head>
      <meta charset="UTF-8">
      <title>Task Manager API</title>
      <style>
        body { font-family: Arial, sans-serif; max-width: 800px; margin: 50px auto; padding: 20px; }
        h1 { color: #333; }
        .endpoint { background: #f4f4f4; padding: 10px; margin: 10px 0; border-radius: 4px; }
        .method { font-weight: bold; color: #007bff; }
      </style>
    </head>
    <body>
      <h1>🚀 Task Manager API</h1>
      <p>ยินดีต้อนรับสู่ Task Manager API</p>
      
      <h2>Available Endpoints:</h2>
      
      <div class="endpoint">
        <span class="method">GET</span> /api/tasks - ดู tasks ทั้งหมด
      </div>
      <div class="endpoint">
        <span class="method">GET</span> /api/tasks/:id - ดู task เดียว
      </div>
      <div class="endpoint">
        <span class="method">POST</span> /api/tasks - สร้าง task ใหม่
      </div>
      <div class="endpoint">
        <span class="method">PUT</span> /api/tasks/:id - อัปเดต task
      </div>
      <div class="endpoint">
        <span class="method">DELETE</span> /api/tasks/:id - ลบ task
      </div>
    </body>
    </html>
  `);
});

// GET ทุก tasks
app.get('/api/tasks', (req, res) => {
  // รองรับ query string สำหรับ filter
  const { completed, priority } = req.query;
  
  let filteredTasks = [...tasks];
  
  // กรองตาม completed status
  if (completed !== undefined) {
    const isCompleted = completed === 'true';
    filteredTasks = filteredTasks.filter(t => t.completed === isCompleted);
  }
  
  // กรองตาม priority
  if (priority) {
    filteredTasks = filteredTasks.filter(t => t.priority === priority);
  }
  
  res.json({
    success: true,
    count: filteredTasks.length,
    data: filteredTasks
  });
});

// GET task เดียวตาม ID
app.get('/api/tasks/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const task = tasks.find(t => t.id === id);
  
  if (!task) {
    return res.status(404).json({
      success: false,
      message: `ไม่พบ Task ID: ${id}`
    });
  }
  
  res.json({
    success: true,
    data: task
  });
});

// POST สร้าง task ใหม่
app.post('/api/tasks', (req, res) => {
  const { title, priority = 'medium' } = req.body;
  
  // Validation
  if (!title || title.trim() === '') {
    return res.status(400).json({
      success: false,
      message: 'กรุณาระบุ title ของ task'
    });
  }
  
  const validPriorities = ['low', 'medium', 'high'];
  if (!validPriorities.includes(priority)) {
    return res.status(400).json({
      success: false,
      message: `priority ต้องเป็น: ${validPriorities.join(', ')}`
    });
  }
  
  const newTask = {
    id: nextId++,
    title: title.trim(),
    completed: false,
    priority,
    createdAt: new Date().toISOString()
  };
  
  tasks.push(newTask);
  
  res.status(201).json({
    success: true,
    message: 'สร้าง task สำเร็จ',
    data: newTask
  });
});

// PUT อัปเดต task
app.put('/api/tasks/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const taskIndex = tasks.findIndex(t => t.id === id);
  
  if (taskIndex === -1) {
    return res.status(404).json({
      success: false,
      message: `ไม่พบ Task ID: ${id}`
    });
  }
  
  const { title, completed, priority } = req.body;
  
  // อัปเดตเฉพาะ fields ที่ส่งมา
  if (title !== undefined) tasks[taskIndex].title = title;
  if (completed !== undefined) tasks[taskIndex].completed = completed;
  if (priority !== undefined) tasks[taskIndex].priority = priority;
  tasks[taskIndex].updatedAt = new Date().toISOString();
  
  res.json({
    success: true,
    message: 'อัปเดต task สำเร็จ',
    data: tasks[taskIndex]
  });
});

// DELETE ลบ task
app.delete('/api/tasks/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const taskIndex = tasks.findIndex(t => t.id === id);
  
  if (taskIndex === -1) {
    return res.status(404).json({
      success: false,
      message: `ไม่พบ Task ID: ${id}`
    });
  }
  
  const deletedTask = tasks[taskIndex];
  tasks.splice(taskIndex, 1);
  
  res.json({
    success: true,
    message: 'ลบ task สำเร็จ',
    data: deletedTask
  });
});

// ==============================
// 404 Handler (ต้องอยู่ท้ายสุด)
// ==============================
app.use((req, res) => {
  res.status(404).json({
    success: false,
    message: `ไม่พบ endpoint: ${req.method} ${req.url}`
  });
});

// ==============================
// Error Handler
// ==============================
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({
    success: false,
    message: 'เกิดข้อผิดพลาดภายใน Server',
    error: process.env.NODE_ENV === 'development' ? err.message : undefined
  });
});

// ==============================
// Start Server
// ==============================
app.listen(PORT, () => {
  console.log(`✅ Task Manager API กำลังทำงานที่ http://localhost:${PORT}`);
  console.log(`📖 API Documentation: http://localhost:${PORT}`);
});
```

### ทดสอบด้วย curl

```bash
# ดู tasks ทั้งหมด
curl http://localhost:3000/api/tasks

# ดู tasks ที่ยังไม่เสร็จ
curl "http://localhost:3000/api/tasks?completed=false"

# ดู tasks ที่มี priority สูง
curl "http://localhost:3000/api/tasks?priority=high"

# ดู task เดียว
curl http://localhost:3000/api/tasks/1

# สร้าง task ใหม่
curl -X POST http://localhost:3000/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "เรียน Middleware", "priority": "high"}'

# อัปเดต task
curl -X PUT http://localhost:3000/api/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"completed": true}'

# ลบ task
curl -X DELETE http://localhost:3000/api/tasks/2
```

### ทดสอบด้วย Thunder Client หรือ Postman

```
GET    http://localhost:3000/api/tasks
GET    http://localhost:3000/api/tasks/1
POST   http://localhost:3000/api/tasks
PUT    http://localhost:3000/api/tasks/1
DELETE http://localhost:3000/api/tasks/1
```

---

## 8. Exercise

### Exercise 1: สร้าง Book Library API

สร้าง Express server ที่จัดการข้อมูลหนังสือ (Books) โดยมีโครงสร้างข้อมูลดังนี้:

```javascript
// โครงสร้างข้อมูลหนังสือ
{
  id: 1,
  title: "ชื่อหนังสือ",
  author: "ชื่อผู้แต่ง",
  genre: "ประเภท (fiction/non-fiction/sci-fi/etc.)",
  year: 2023,
  available: true
}
```

**Requirements:**
- `GET /api/books` - ดูหนังสือทั้งหมด (รองรับ query: `?genre=fiction&available=true`)
- `GET /api/books/:id` - ดูหนังสือเล่มเดียว
- `POST /api/books` - เพิ่มหนังสือใหม่
- `PUT /api/books/:id` - อัปเดตข้อมูลหนังสือ
- `DELETE /api/books/:id` - ลบหนังสือ
- `PATCH /api/books/:id/borrow` - ยืมหนังสือ (เปลี่ยน available เป็น false)
- `PATCH /api/books/:id/return` - คืนหนังสือ (เปลี่ยน available เป็น true)

### Exercise 2: Server Information Endpoint

สร้าง endpoint ที่แสดงข้อมูลของ server:

```
GET /api/server-info
```

ต้อง return:
```json
{
  "serverTime": "2024-01-01T00:00:00.000Z",
  "nodeVersion": "v20.x.x",
  "uptime": "X seconds",
  "platform": "linux",
  "memoryUsage": {
    "used": "X MB",
    "total": "X MB"
  }
}
```

**Hint:** ใช้ `process.version`, `process.uptime()`, `process.platform`, `process.memoryUsage()`

### Exercise 3: Calculator API

สร้าง Calculator API:
- `GET /api/calc/add?a=5&b=3` → `{ result: 8 }`
- `GET /api/calc/subtract?a=10&b=4` → `{ result: 6 }`
- `GET /api/calc/multiply?a=3&b=4` → `{ result: 12 }`
- `GET /api/calc/divide?a=10&b=2` → `{ result: 5 }`

ต้องมี error handling:
- ถ้า a หรือ b ไม่ใช่ตัวเลข → 400 Bad Request
- ถ้าหาร 0 → 400 Bad Request พร้อม message "ไม่สามารถหารด้วย 0 ได้"

---

## สรุป Part 11

ในบทนี้เราได้เรียนรู้:

1. **Express.js คืออะไร** - Framework สำหรับสร้าง web server บน Node.js
2. **การติดตั้ง** - `npm install express` และ setup โปรเจกต์
3. **Hello World** - สร้าง server แรกด้วย Express
4. **Application Structure** - โครงสร้างไฟล์และโฟลเดอร์ที่ดี
5. **HTTP Methods** - get, post, put, patch, delete
6. **Request/Response** - overview ของ objects สำคัญ
7. **Practical** - สร้าง Task Manager API ที่ใช้งานได้จริง

---

## อ่านต่อ

ใน Part 12 เราจะเรียนรู้เรื่อง **Express Routing** อย่างละเอียด:
- Route parameters (:id)
- Query strings
- Router patterns
- Express Router module
- Route grouping

---

*Part 11 | Node.js/Express.js Course | ขั้นตอนที่ 11 จาก 1000*
