# Part 15: Template Engines
## ขั้นตอนที่ 15-5 จาก 1000

---

## สารบัญ
1. EJS (Embedded JavaScript)
2. Pug (เดิมชื่อ Jade)
3. Handlebars
4. Server-side Rendering vs Client-side
5. Layouts และ Partials
6. การส่งข้อมูลไปยัง Templates
7. Practical: Blog Website ด้วย EJS
8. Exercise

---

## 1. EJS (Embedded JavaScript)

EJS เป็น template engine ที่ได้รับความนิยมสูงสุด เพราะใช้ JavaScript ธรรมดา และ syntax คล้าย HTML

### ติดตั้ง EJS

```bash
npm install ejs
```

### ตั้งค่า Express ให้ใช้ EJS

```javascript
const express = require('express');
const path = require('path');
const app = express();

// ตั้งค่า template engine
app.set('view engine', 'ejs');
app.set('views', path.join(__dirname, 'views'));
// views คือโฟลเดอร์ที่เก็บ template files

app.get('/', (req, res) => {
  res.render('index', {
    title: 'หน้าแรก',
    message: 'ยินดีต้อนรับ'
  });
});
```

### EJS Syntax

```html
<!-- views/index.ejs -->
<!DOCTYPE html>
<html>
<head>
  <title><%= title %></title>
</head>
<body>

  <!-- 1. <%= %> - แสดงค่า (escaped HTML) -->
  <h1><%= title %></h1>
  <p><%= message %></p>

  <!-- 2. <%- %> - แสดงค่า (unescaped - ระวัง XSS!) -->
  <%- htmlContent %>

  <!-- 3. <% %> - JavaScript code (ไม่แสดงผล) -->
  <% const now = new Date() %>
  <p>วันที่: <%= now.toLocaleDateString('th-TH') %></p>

  <!-- 4. <%# %> - Comment (ไม่แสดงใน source) -->
  <%# นี่คือ comment ใน EJS %>

  <!-- 5. <%- include() %> - Include partial -->
  <%- include('partials/header', { title: title }) %>

  <!-- การใช้ if/else -->
  <% if (user) { %>
    <p>สวัสดี, <%= user.name %>!</p>
  <% } else { %>
    <p><a href="/login">เข้าสู่ระบบ</a></p>
  <% } %>

  <!-- การวน loop -->
  <% if (items && items.length > 0) { %>
    <ul>
      <% items.forEach((item, index) => { %>
        <li>
          <%= index + 1 %>. <%= item.name %> - ฿<%= item.price.toLocaleString() %>
        </li>
      <% }) %>
    </ul>
  <% } else { %>
    <p>ไม่มีรายการ</p>
  <% } %>

  <!-- Ternary operator -->
  <span class="<%= user.active ? 'active' : 'inactive' %>">
    <%= user.active ? 'ใช้งานได้' : 'ปิดการใช้งาน' %>
  </span>

</body>
</html>
```

### EJS Layouts ด้วย express-ejs-layouts

```bash
npm install express-ejs-layouts
```

```javascript
const expressLayouts = require('express-ejs-layouts');
app.use(expressLayouts);
app.set('layout', 'layouts/main'); // default layout
```

```html
<!-- views/layouts/main.ejs -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title><%= title %></title>
  <link rel="stylesheet" href="/css/style.css">
  <%- style %>  <!-- inject styles จาก page -->
</head>
<body>
  <%- include('../partials/navbar') %>
  
  <main class="container">
    <%- body %>  <!-- inject page content -->
  </main>
  
  <%- include('../partials/footer') %>
  
  <script src="/js/main.js"></script>
  <%- script %>  <!-- inject scripts จาก page -->
</body>
</html>
```

---

## 2. Pug (เดิมชื่อ Jade)

Pug ใช้ indentation แทน HTML tags ทำให้โค้ดกระชับมาก

### ติดตั้ง Pug

```bash
npm install pug
```

### ตั้งค่า Express

```javascript
app.set('view engine', 'pug');
app.set('views', './views');
```

### Pug Syntax

```pug
//- views/index.pug
doctype html
html(lang="th")
  head
    title= title
    link(rel='stylesheet', href='/css/style.css')
  body
    //- นี่คือ comment ใน Pug
    
    h1= title
    p= message
    
    //- Variables
    - const now = new Date()
    p วันที่: #{now.toLocaleDateString('th-TH')}
    
    //- Interpolation
    p ยินดีต้อนรับ, #{user.name}!
    
    //- Conditional
    if user
      p.text-green ผู้ใช้: #{user.name}
    else
      a(href='/login') เข้าสู่ระบบ
    
    //- Loop
    ul
      each item in items
        li #{item.name} - ฿#{item.price}
    
    //- Attributes
    a(href='/about', class='btn btn-primary') เกี่ยวกับ
    
    //- Multiline
    p.
      นี่คือข้อความหลายบรรทัด
      ที่ยังคงเป็น paragraph เดียว
    
    //- Include
    include partials/header
    
    //- Extends layout
```

```pug
//- views/layout.pug (Layout template)
doctype html
html(lang="th")
  head
    title= title
    block styles
      link(rel='stylesheet', href='/css/style.css')
  body
    include partials/navbar
    
    main.container
      block content
    
    include partials/footer
    
    block scripts
      script(src='/js/main.js')
```

```pug
//- views/page.pug (ใช้ layout)
extends layout

block styles
  link(rel='stylesheet', href='/css/page.css')

block content
  h1= title
  p= content

block scripts
  script(src='/js/page.js')
```

---

## 3. Handlebars

Handlebars เน้น logic-less templates ทำให้ปลอดภัยกว่า

### ติดตั้ง

```bash
npm install express-handlebars
```

### ตั้งค่า Express

```javascript
const { engine } = require('express-handlebars');

app.engine('handlebars', engine({
  defaultLayout: 'main',
  layoutsDir: './views/layouts/',
  partialsDir: './views/partials/'
}));
app.set('view engine', 'handlebars');
app.set('views', './views');
```

### Handlebars Syntax

```handlebars
{{!-- views/index.handlebars --}}
<!DOCTYPE html>
<html>
<head>
  <title>{{title}}</title>
</head>
<body>
  {{!-- Variable --}}
  <h1>{{title}}</h1>
  
  {{!-- Unescaped HTML --}}
  {{{htmlContent}}}
  
  {{!-- Conditional --}}
  {{#if user}}
    <p>สวัสดี, {{user.name}}!</p>
  {{else}}
    <p>กรุณา login</p>
  {{/if}}
  
  {{!-- Unless (ตรงข้าม if) --}}
  {{#unless isLoggedIn}}
    <a href="/login">เข้าสู่ระบบ</a>
  {{/unless}}
  
  {{!-- Loop --}}
  {{#each items}}
    <div>
      {{@index}}. {{this.name}} - ฿{{this.price}}
      {{#if this.inStock}}
        <span class="badge">มีสินค้า</span>
      {{/if}}
    </div>
  {{/each}}
  
  {{!-- Partial --}}
  {{> header}}
  {{> nav title=title}}
  
  {{!-- Custom Helper --}}
  <p>วันที่: {{formatDate date}}</p>
  <p>ราคา: {{formatPrice price}}</p>
</body>
</html>
```

### Custom Helpers

```javascript
const { engine } = require('express-handlebars');

app.engine('handlebars', engine({
  defaultLayout: 'main',
  helpers: {
    // Helper สำหรับ format วันที่
    formatDate: (date) => {
      return new Date(date).toLocaleDateString('th-TH', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      });
    },
    
    // Helper สำหรับ format ราคา
    formatPrice: (price) => {
      return `฿${price.toLocaleString('th-TH')}`;
    },
    
    // Helper ตรวจสอบ equality
    eq: (a, b) => a === b,
    
    // Helper คำนวณ
    add: (a, b) => a + b,
    
    // Helper uppercase
    upper: (str) => str.toUpperCase(),
    
    // Block helper
    limit: (array, num) => array.slice(0, num)
  }
}));
```

---

## 4. Server-side Rendering vs Client-side

### Server-side Rendering (SSR)

```javascript
// Express render HTML บน server
app.get('/products', async (req, res) => {
  // ดึงข้อมูลจาก database บน server
  const products = await Product.findAll();
  
  // render HTML แล้วส่งกลับ
  res.render('products', { products });
});
```

```html
<!-- HTML ที่ส่งกลับมาพร้อม data แล้ว -->
<!DOCTYPE html>
<html>
<body>
  <ul>
    <li>iPhone 15 - ฿44,900</li>
    <li>Samsung S24 - ฿35,900</li>
  </ul>
</body>
</html>
```

### Client-side Rendering (CSR)

```javascript
// Express ส่งเฉพาะ JSON data
app.get('/api/products', async (req, res) => {
  const products = await Product.findAll();
  res.json({ products });
});
```

```html
<!-- HTML ว่าง, JavaScript fetch data มา render -->
<!DOCTYPE html>
<html>
<body>
  <ul id="products-list"></ul>
  
  <script>
    fetch('/api/products')
      .then(r => r.json())
      .then(({ products }) => {
        const list = document.getElementById('products-list');
        products.forEach(p => {
          const li = document.createElement('li');
          li.textContent = `${p.name} - ฿${p.price}`;
          list.appendChild(li);
        });
      });
  </script>
</body>
</html>
```

### เปรียบเทียบ SSR vs CSR

| | SSR | CSR |
|--|-----|-----|
| SEO | ดีเยี่ยม | ต้องใช้ prerender |
| First Load | เร็วกว่า | ช้ากว่า |
| Subsequent Load | ช้ากว่า | เร็วกว่า |
| Server Load | สูง | ต่ำ |
| Complexity | ง่าย | ซับซ้อน |
| Interactivity | จำกัด | สูง |

---

## 5. Layouts และ Partials

### EJS Partials

```javascript
// ไม่ต้องติดตั้ง package เพิ่ม
// ใช้ <%- include() %>
```

```html
<!-- views/partials/navbar.ejs -->
<nav class="navbar">
  <a class="navbar-brand" href="/">
    <%= siteName || 'My Blog' %>
  </a>
  
  <ul class="navbar-nav">
    <li><a href="/" class="<%= currentPage === 'home' ? 'active' : '' %>">หน้าแรก</a></li>
    <li><a href="/blog" class="<%= currentPage === 'blog' ? 'active' : '' %>">บทความ</a></li>
    <li><a href="/about" class="<%= currentPage === 'about' ? 'active' : '' %>">เกี่ยวกับ</a></li>
  </ul>
  
  <% if (user) { %>
    <div class="user-info">
      <span>สวัสดี, <%= user.name %></span>
      <a href="/logout">ออกจากระบบ</a>
    </div>
  <% } else { %>
    <a href="/login" class="btn">เข้าสู่ระบบ</a>
  <% } %>
</nav>
```

```html
<!-- views/partials/footer.ejs -->
<footer class="footer">
  <div class="container">
    <p>&copy; <%= new Date().getFullYear() %> <%= siteName %>. All rights reserved.</p>
    <ul class="footer-links">
      <li><a href="/privacy">Privacy Policy</a></li>
      <li><a href="/terms">Terms of Service</a></li>
      <li><a href="/contact">ติดต่อเรา</a></li>
    </ul>
  </div>
</footer>
```

```html
<!-- views/partials/head.ejs -->
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><%= title %> | <%= siteName %></title>
  <meta name="description" content="<%= description || 'My awesome website' %>">
  
  <!-- Open Graph -->
  <meta property="og:title" content="<%= title %>">
  <meta property="og:description" content="<%= description || '' %>">
  
  <!-- CSS -->
  <link rel="stylesheet" href="/css/style.css">
  <% if (typeof extraStyles !== 'undefined') { %>
    <%- extraStyles %>
  <% } %>
</head>
```

### Layout ด้วย EJS

```html
<!-- views/layouts/main.ejs (แบบ manual) -->
<!DOCTYPE html>
<html lang="th">

<%- include('../partials/head', { title, siteName, description }) %>

<body>
  <%- include('../partials/navbar', { user, currentPage }) %>
  
  <main>
    <%- body %>
  </main>
  
  <%- include('../partials/footer', { siteName }) %>
  
  <script src="/js/main.js"></script>
  <% if (typeof extraScripts !== 'undefined') { %>
    <%- extraScripts %>
  <% } %>
</body>
</html>
```

---

## 6. การส่งข้อมูลไปยัง Templates

### ส่งข้อมูลผ่าน res.render()

```javascript
// ส่งข้อมูลหลายรูปแบบ
app.get('/page', (req, res) => {
  res.render('page', {
    // String
    title: 'My Page',
    
    // Number
    count: 42,
    
    // Boolean
    isLoggedIn: true,
    
    // Object
    user: {
      id: 1,
      name: 'สมชาย',
      email: 'somchai@example.com'
    },
    
    // Array
    items: ['item1', 'item2', 'item3'],
    
    // Array of objects
    products: [
      { id: 1, name: 'Product A', price: 100 },
      { id: 2, name: 'Product B', price: 200 }
    ],
    
    // Function (ระวัง! บาง engine อาจไม่รองรับ)
    formatDate: (date) => new Date(date).toLocaleDateString('th-TH'),
    
    // HTML string (ต้องใช้ <%- %> ใน EJS)
    htmlContent: '<strong>Bold text</strong>',
    
    // Nested objects
    settings: {
      theme: 'dark',
      language: 'th',
      currency: 'THB'
    }
  });
});
```

### Global Data ด้วย res.locals

```javascript
// Middleware ตั้งค่า locals ที่ใช้ทุกหน้า
app.use((req, res, next) => {
  res.locals.siteName = 'My Blog';
  res.locals.currentYear = new Date().getFullYear();
  res.locals.user = req.user || null;
  res.locals.currentPage = req.path.split('/')[1] || 'home';
  res.locals.flashMessages = req.flash ? req.flash() : {};
  next();
});

// ใน route ไม่ต้องส่ง siteName อีกต่อไป
app.get('/about', (req, res) => {
  res.render('about', {
    title: 'เกี่ยวกับเรา',
    description: 'เรื่องราวของเรา'
    // siteName มาจาก res.locals อัตโนมัติ
  });
});
```

### Flash Messages

```bash
npm install connect-flash express-session
```

```javascript
const session = require('express-session');
const flash = require('connect-flash');

app.use(session({ secret: 'secret', resave: false, saveUninitialized: false }));
app.use(flash());

// Set flash ใน middleware
app.use((req, res, next) => {
  res.locals.success_msg = req.flash('success');
  res.locals.error_msg = req.flash('error');
  next();
});

// ใช้ flash ใน routes
app.post('/login', (req, res) => {
  // login logic...
  req.flash('success', 'Login สำเร็จ!');
  res.redirect('/dashboard');
});

app.post('/register', (req, res) => {
  // validation failed
  req.flash('error', 'กรุณากรอกข้อมูลให้ครบถ้วน');
  res.redirect('/register');
});
```

```html
<!-- views/partials/flash.ejs -->
<% if (success_msg && success_msg.length > 0) { %>
  <div class="alert alert-success">
    <% success_msg.forEach(msg => { %>
      <p><%= msg %></p>
    <% }) %>
  </div>
<% } %>

<% if (error_msg && error_msg.length > 0) { %>
  <div class="alert alert-error">
    <% error_msg.forEach(msg => { %>
      <p><%= msg %></p>
    <% }) %>
  </div>
<% } %>
```

---

## 7. Practical: Blog Website ด้วย EJS

### โครงสร้างไฟล์

```
blog/
├── public/
│   ├── css/
│   │   └── style.css
│   └── js/
│       └── main.js
├── views/
│   ├── layouts/
│   │   └── main.ejs
│   ├── partials/
│   │   ├── head.ejs
│   │   ├── navbar.ejs
│   │   ├── footer.ejs
│   │   └── post-card.ejs
│   ├── pages/
│   │   ├── home.ejs
│   │   ├── blog.ejs
│   │   ├── post.ejs
│   │   ├── about.ejs
│   │   └── not-found.ejs
│   └── admin/
│       ├── dashboard.ejs
│       ├── new-post.ejs
│       └── edit-post.ejs
├── routes/
│   ├── index.js
│   ├── blog.js
│   └── admin.js
├── data/
│   └── posts.js
└── app.js
```

### data/posts.js

```javascript
// data/posts.js
const posts = [
  {
    id: 1,
    slug: 'getting-started-with-nodejs',
    title: 'เริ่มต้นกับ Node.js',
    excerpt: 'Node.js คือ runtime ที่ช่วยให้เราใช้ JavaScript บน server...',
    content: `
      <h2>Node.js คืออะไร?</h2>
      <p>Node.js เป็น JavaScript runtime ที่สร้างบน V8 engine ของ Chrome...</p>
      
      <h2>ทำไมต้องใช้ Node.js?</h2>
      <p>Node.js เหมาะสำหรับการสร้าง real-time applications เนื่องจาก...</p>
      
      <pre><code>const http = require('http');
const server = http.createServer((req, res) => {
  res.end('Hello Node.js!');
});
server.listen(3000);</code></pre>
    `,
    author: { id: 1, name: 'สมชาย ใจดี', avatar: '/img/avatar1.jpg' },
    category: 'Node.js',
    tags: ['nodejs', 'javascript', 'backend'],
    coverImage: '/img/nodejs.jpg',
    publishedAt: new Date('2024-01-15'),
    updatedAt: new Date('2024-01-20'),
    published: true,
    views: 1250,
    likes: 89,
    readTime: 5
  },
  {
    id: 2,
    slug: 'express-middleware-guide',
    title: 'คู่มือ Express Middleware',
    excerpt: 'Middleware เป็นหัวใจของ Express.js ที่ช่วยให้เราจัดการ request...',
    content: `
      <h2>Middleware คืออะไร?</h2>
      <p>Middleware คือ function ที่ทำงานระหว่าง request และ response...</p>
    `,
    author: { id: 1, name: 'สมชาย ใจดี', avatar: '/img/avatar1.jpg' },
    category: 'Express.js',
    tags: ['express', 'middleware', 'nodejs'],
    coverImage: '/img/express.jpg',
    publishedAt: new Date('2024-01-20'),
    updatedAt: new Date('2024-01-22'),
    published: true,
    views: 890,
    likes: 65,
    readTime: 8
  }
];

module.exports = { posts };
```

### views/layouts/main.ejs

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><%= typeof title !== 'undefined' ? title + ' | ' : '' %><%= siteName %></title>
  <meta name="description" content="<%= typeof description !== 'undefined' ? description : 'Blog เกี่ยวกับ Node.js และ Web Development' %>">
  
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
  <link rel="stylesheet" href="/css/style.css">
</head>
<body class="<%= typeof bodyClass !== 'undefined' ? bodyClass : '' %>">
  
  <%- include('../partials/navbar') %>
  
  <main>
    <%- body %>
  </main>
  
  <%- include('../partials/footer') %>
  
  <script src="/js/main.js"></script>
</body>
</html>
```

### views/partials/navbar.ejs

```html
<nav class="navbar">
  <div class="navbar-container">
    <a href="/" class="navbar-brand">
      <i class="fas fa-code"></i> <%= siteName %>
    </a>
    
    <button class="navbar-toggle" id="navToggle">
      <i class="fas fa-bars"></i>
    </button>
    
    <ul class="navbar-menu" id="navMenu">
      <li><a href="/" class="<%= currentPage === '' ? 'active' : '' %>">หน้าแรก</a></li>
      <li><a href="/blog" class="<%= currentPage === 'blog' ? 'active' : '' %>">บทความ</a></li>
      <li><a href="/about" class="<%= currentPage === 'about' ? 'active' : '' %>">เกี่ยวกับ</a></li>
      
      <% if (user && user.role === 'admin') { %>
        <li><a href="/admin" class="<%= currentPage === 'admin' ? 'active' : '' %>">จัดการ</a></li>
      <% } %>
    </ul>
    
    <div class="navbar-search">
      <form action="/blog/search" method="GET">
        <input type="text" name="q" placeholder="ค้นหา..." value="<%= typeof searchQuery !== 'undefined' ? searchQuery : '' %>">
        <button type="submit"><i class="fas fa-search"></i></button>
      </form>
    </div>
  </div>
</nav>
```

### views/pages/home.ejs

```html
<!-- Hero Section -->
<section class="hero">
  <div class="hero-content">
    <h1>ยินดีต้อนรับสู่ <%= siteName %></h1>
    <p>บทความเกี่ยวกับ Node.js, Express.js และ Web Development</p>
    <a href="/blog" class="btn btn-primary">อ่านบทความ</a>
  </div>
</section>

<!-- Latest Posts -->
<section class="section">
  <div class="container">
    <h2 class="section-title">บทความล่าสุด</h2>
    
    <div class="posts-grid">
      <% latestPosts.forEach(post => { %>
        <%- include('../partials/post-card', { post }) %>
      <% }) %>
    </div>
    
    <div class="text-center">
      <a href="/blog" class="btn btn-outline">ดูทั้งหมด</a>
    </div>
  </div>
</section>

<!-- Stats -->
<section class="stats-section">
  <div class="container">
    <div class="stats-grid">
      <div class="stat-item">
        <span class="stat-number"><%= stats.totalPosts %></span>
        <span class="stat-label">บทความ</span>
      </div>
      <div class="stat-item">
        <span class="stat-number"><%= stats.totalViews.toLocaleString() %></span>
        <span class="stat-label">ผู้อ่าน</span>
      </div>
      <div class="stat-item">
        <span class="stat-number"><%= stats.totalCategories %></span>
        <span class="stat-label">หมวดหมู่</span>
      </div>
    </div>
  </div>
</section>
```

### views/partials/post-card.ejs

```html
<article class="post-card">
  <% if (post.coverImage) { %>
    <a href="/blog/<%= post.slug %>">
      <img src="<%= post.coverImage %>" alt="<%= post.title %>" class="post-card-image" loading="lazy">
    </a>
  <% } %>
  
  <div class="post-card-body">
    <div class="post-card-meta">
      <span class="post-category"><%= post.category %></span>
      <span class="post-read-time"><i class="far fa-clock"></i> <%= post.readTime %> นาที</span>
    </div>
    
    <h3 class="post-card-title">
      <a href="/blog/<%= post.slug %>"><%= post.title %></a>
    </h3>
    
    <p class="post-card-excerpt"><%= post.excerpt %></p>
    
    <div class="post-card-footer">
      <div class="post-author">
        <img src="<%= post.author.avatar %>" alt="<%= post.author.name %>" class="author-avatar">
        <span><%= post.author.name %></span>
      </div>
      
      <div class="post-stats">
        <span><i class="far fa-eye"></i> <%= post.views %></span>
        <span><i class="far fa-heart"></i> <%= post.likes %></span>
      </div>
      
      <time datetime="<%= post.publishedAt.toISOString() %>">
        <%= post.publishedAt.toLocaleDateString('th-TH', { year: 'numeric', month: 'long', day: 'numeric' }) %>
      </time>
    </div>
    
    <div class="post-tags">
      <% post.tags.forEach(tag => { %>
        <a href="/blog/tag/<%= tag %>" class="tag">#<%= tag %></a>
      <% }) %>
    </div>
  </div>
</article>
```

### routes/blog.js

```javascript
// routes/blog.js
const express = require('express');
const router = express.Router();
const { posts } = require('../data/posts');

// GET /blog - รายการบทความ
router.get('/', (req, res) => {
  const { category, tag, page = 1 } = req.query;
  const limit = 6;
  
  let filteredPosts = posts.filter(p => p.published);
  
  if (category) {
    filteredPosts = filteredPosts.filter(p =>
      p.category.toLowerCase() === category.toLowerCase()
    );
  }
  
  if (tag) {
    filteredPosts = filteredPosts.filter(p => p.tags.includes(tag));
  }
  
  const total = filteredPosts.length;
  const offset = (page - 1) * limit;
  const currentPosts = filteredPosts.slice(offset, offset + limit);
  
  const categories = [...new Set(posts.map(p => p.category))];
  
  res.render('pages/blog', {
    title: 'บทความทั้งหมด',
    description: 'บทความเกี่ยวกับ Node.js และ Web Development',
    posts: currentPosts,
    categories,
    pagination: {
      total,
      currentPage: parseInt(page),
      totalPages: Math.ceil(total / limit),
      limit
    },
    currentCategory: category,
    currentTag: tag
  });
});

// GET /blog/search - ค้นหา
router.get('/search', (req, res) => {
  const { q } = req.query;
  
  if (!q) return res.redirect('/blog');
  
  const results = posts.filter(p =>
    p.published && (
      p.title.toLowerCase().includes(q.toLowerCase()) ||
      p.excerpt.toLowerCase().includes(q.toLowerCase()) ||
      p.tags.some(t => t.includes(q.toLowerCase()))
    )
  );
  
  res.render('pages/blog', {
    title: `ผลการค้นหา: "${q}"`,
    posts: results,
    searchQuery: q,
    pagination: null
  });
});

// GET /blog/:slug - บทความเดียว
router.get('/:slug', (req, res) => {
  const post = posts.find(p => p.slug === req.params.slug && p.published);
  
  if (!post) {
    return res.status(404).render('pages/not-found', {
      title: '404 - ไม่พบหน้าที่ต้องการ'
    });
  }
  
  post.views++;
  
  const relatedPosts = posts
    .filter(p =>
      p.id !== post.id &&
      p.published &&
      (p.category === post.category || p.tags.some(t => post.tags.includes(t)))
    )
    .slice(0, 3);
  
  res.render('pages/post', {
    title: post.title,
    description: post.excerpt,
    post,
    relatedPosts
  });
});

module.exports = router;
```

### app.js - Blog Application

```javascript
// app.js
const express = require('express');
const expressLayouts = require('express-ejs-layouts');
const path = require('path');
const { posts } = require('./data/posts');

const app = express();
const PORT = 3000;

// View Engine
app.set('view engine', 'ejs');
app.set('views', path.join(__dirname, 'views'));
app.use(expressLayouts);
app.set('layout', 'layouts/main');

// Static files
app.use(express.static(path.join(__dirname, 'public')));

// Body parsing
app.use(express.urlencoded({ extended: true }));

// Global template variables
app.use((req, res, next) => {
  res.locals.siteName = 'DevBlog TH';
  res.locals.currentYear = new Date().getFullYear();
  res.locals.user = req.user || null;
  res.locals.currentPage = req.path.split('/')[1];
  next();
});

// Routes
app.get('/', (req, res) => {
  const latestPosts = posts
    .filter(p => p.published)
    .sort((a, b) => b.publishedAt - a.publishedAt)
    .slice(0, 3);
  
  const stats = {
    totalPosts: posts.filter(p => p.published).length,
    totalViews: posts.reduce((sum, p) => sum + p.views, 0),
    totalCategories: new Set(posts.map(p => p.category)).size
  };
  
  res.render('pages/home', {
    title: 'หน้าแรก',
    latestPosts,
    stats
  });
});

app.use('/blog', require('./routes/blog'));

app.get('/about', (req, res) => {
  res.render('pages/about', {
    title: 'เกี่ยวกับเรา'
  });
});

// 404
app.use((req, res) => {
  res.status(404).render('pages/not-found', {
    title: '404 - ไม่พบหน้า'
  });
});

app.listen(PORT, () => {
  console.log(`Blog running at http://localhost:${PORT}`);
});
```

---

## 8. Exercise

### Exercise 1: Portfolio Website

สร้าง Portfolio Website ด้วย EJS ที่มีหน้าต่างๆ:
- หน้าแรก (About Me)
- หน้า Projects (แสดง projects ทั้งหมด)
- หน้า Project Detail (รายละเอียดแต่ละ project)
- หน้า Skills
- หน้า Contact

### Exercise 2: News Website

สร้าง News Website ด้วย Pug หรือ Handlebars:
- หน้า Breaking News
- หมวดหมู่ข่าว (Politics, Business, Tech, Sports)
- หน้า Article
- Search functionality
- Related articles

### Exercise 3: Admin Dashboard

สร้าง Admin Dashboard ที่มี:
- Overview stats (จำนวน users, posts, orders)
- Recent activities
- Charts (ใช้ Chart.js หรือ CSS สร้าง bar chart)
- Quick actions

---

## สรุป Part 15

ในบทนี้เราได้เรียนรู้:

1. **EJS** - Template engine ที่ใช้ JavaScript ธรรมดา syntax คล้าย HTML
2. **Pug** - Template engine ที่ใช้ indentation แทน tags กระชับมาก
3. **Handlebars** - Logic-less templates ที่ปลอดภัย
4. **SSR vs CSR** - ความแตกต่างและเมื่อไหรควรใช้อะไร
5. **Layouts และ Partials** - โครงสร้าง template ที่ดี
6. **การส่งข้อมูล** - res.render(), res.locals, flash messages

---

*Part 15 | Node.js/Express.js Course | ขั้นตอนที่ 15 จาก 1000*
