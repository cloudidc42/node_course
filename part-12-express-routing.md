# Part 12: Express Routing
## ขั้นตอนที่ 12-2 จาก 1000

---

## สารบัญ
1. Route Methods (GET, POST, PUT, PATCH, DELETE)
2. Route Parameters (:id)
3. Query Strings
4. Route Patterns และ Wildcards
5. Router.route() Chaining
6. Express Router
7. Route Groups
8. Practical: Product Catalog Routes
9. Exercise

---

## 1. Route Methods

HTTP มี methods หลายอย่างที่ Express รองรับ แต่ละ method มีความหมายและการใช้งานที่แตกต่างกัน:

### ความหมายของแต่ละ Method

| Method | ความหมาย | ตัวอย่าง |
|--------|----------|---------|
| GET | ดึงข้อมูล | GET /users |
| POST | สร้างข้อมูลใหม่ | POST /users |
| PUT | อัปเดตทั้ง resource | PUT /users/1 |
| PATCH | อัปเดตบางส่วน | PATCH /users/1 |
| DELETE | ลบข้อมูล | DELETE /users/1 |
| HEAD | ดึงเฉพาะ headers | HEAD /users |
| OPTIONS | ดูว่า server รองรับอะไร | OPTIONS /users |

### ตัวอย่าง Route Methods ทั้งหมด

```javascript
const express = require('express');
const app = express();
app.use(express.json());

// ==============================
// GET - ดึงข้อมูล (Read)
// ==============================
app.get('/users', (req, res) => {
  // ดึงรายชื่อ users ทั้งหมด
  res.json({
    users: [
      { id: 1, name: 'สมชาย', email: 'somchai@example.com' },
      { id: 2, name: 'สมหญิง', email: 'somying@example.com' }
    ]
  });
});

app.get('/users/:id', (req, res) => {
  // ดึงข้อมูล user คนเดียว
  res.json({ id: req.params.id, name: 'สมชาย' });
});

// ==============================
// POST - สร้างข้อมูลใหม่ (Create)
// ==============================
app.post('/users', (req, res) => {
  const { name, email, password } = req.body;
  
  // ตรวจสอบข้อมูล
  if (!name || !email || !password) {
    return res.status(400).json({
      error: 'กรุณากรอกข้อมูลให้ครบ'
    });
  }
  
  // สร้าง user ใหม่
  const newUser = { id: 3, name, email };
  res.status(201).json(newUser);
});

// ==============================
// PUT - อัปเดตทั้ง resource (Replace)
// ==============================
app.put('/users/:id', (req, res) => {
  const { id } = req.params;
  const { name, email } = req.body;
  
  // PUT ต้องส่งข้อมูลทั้งหมด
  if (!name || !email) {
    return res.status(400).json({
      error: 'PUT ต้องส่งข้อมูลทั้งหมด'
    });
  }
  
  res.json({ id, name, email, updatedAt: new Date() });
});

// ==============================
// PATCH - อัปเดตบางส่วน (Partial Update)
// ==============================
app.patch('/users/:id', (req, res) => {
  const { id } = req.params;
  const updates = req.body;
  
  // PATCH ส่งเฉพาะ fields ที่ต้องการเปลี่ยน
  console.log(`Updating user ${id} with:`, updates);
  
  res.json({
    message: 'Updated successfully',
    updatedFields: Object.keys(updates)
  });
});

// ==============================
// DELETE - ลบข้อมูล (Delete)
// ==============================
app.delete('/users/:id', (req, res) => {
  const { id } = req.params;
  
  // ลบ user
  res.json({
    message: `ลบ user ID ${id} สำเร็จ`,
    deletedAt: new Date()
  });
});

// ==============================
// HEAD - ดึงเฉพาะ Headers
// ==============================
app.head('/users', (req, res) => {
  // ส่งเฉพาะ headers ไม่มี body
  res.set('X-Total-Count', '100');
  res.end(); // ไม่ส่ง body
});

// ==============================
// OPTIONS - CORS Preflight
// ==============================
app.options('/users', (req, res) => {
  res.set('Allow', 'GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS');
  res.end();
});

app.listen(3000);
```

---

## 2. Route Parameters (:id)

Route parameters เป็นส่วนที่เปลี่ยนแปลงได้ใน URL path:

### พื้นฐาน Route Parameters

```javascript
const express = require('express');
const app = express();

// Parameter เดียว
app.get('/users/:id', (req, res) => {
  console.log(req.params);    // { id: '123' }
  console.log(req.params.id); // '123'  (เป็น string เสมอ!)
  
  const id = parseInt(req.params.id); // แปลงเป็น number
  res.json({ userId: id });
});

// หลาย parameters
app.get('/users/:userId/posts/:postId', (req, res) => {
  const { userId, postId } = req.params;
  console.log(req.params); // { userId: '1', postId: '5' }
  
  res.json({
    message: `Post ${postId} ของ User ${userId}`
  });
});

// Parameter ที่มีชื่อชัดเจน
app.get('/categories/:categorySlug/products/:productSlug', (req, res) => {
  const { categorySlug, productSlug } = req.params;
  res.json({
    category: categorySlug, // 'electronics'
    product: productSlug    // 'iphone-15'
  });
});
```

### Optional Parameters

```javascript
// Express ไม่รองรับ optional parameter โดยตรง
// ต้องสร้างสอง routes แทน

app.get('/posts', (req, res) => {
  // แสดงทุก posts
  res.json({ posts: 'ทุก posts' });
});

app.get('/posts/:id', (req, res) => {
  // แสดง post เดียว
  res.json({ post: `Post ID ${req.params.id}` });
});
```

### Parameter Callback (app.param)

```javascript
// app.param ใช้สำหรับ validate parameter ที่ใช้บ่อย
app.param('userId', (req, res, next, id) => {
  console.log(`Processing userId: ${id}`);
  
  // แปลงเป็น number และตรวจสอบ
  const userId = parseInt(id);
  if (isNaN(userId)) {
    return res.status(400).json({ error: 'userId ต้องเป็นตัวเลข' });
  }
  
  // เพิ่มข้อมูลใน req สำหรับใช้ใน route handler
  req.userId = userId;
  next();
});

// ทุก route ที่มี :userId จะผ่าน callback นี้ก่อน
app.get('/users/:userId', (req, res) => {
  res.json({ userId: req.userId }); // ใช้ req.userId ที่ set ไว้
});

app.get('/users/:userId/profile', (req, res) => {
  res.json({ userId: req.userId, type: 'profile' });
});
```

### ตัวอย่างจริง: User + Posts API

```javascript
const express = require('express');
const app = express();
app.use(express.json());

// ข้อมูลจำลอง
const users = [
  { id: 1, name: 'สมชาย', email: 'somchai@example.com' },
  { id: 2, name: 'สมหญิง', email: 'somying@example.com' }
];

const posts = [
  { id: 1, userId: 1, title: 'บทความแรก', content: 'เนื้อหา...' },
  { id: 2, userId: 1, title: 'บทความสอง', content: 'เนื้อหา...' },
  { id: 3, userId: 2, title: 'บทความของสมหญิง', content: 'เนื้อหา...' }
];

// Validate userId parameter
app.param('userId', (req, res, next, id) => {
  const userId = parseInt(id);
  if (isNaN(userId)) {
    return res.status(400).json({ error: 'userId ต้องเป็นตัวเลข' });
  }
  
  const user = users.find(u => u.id === userId);
  if (!user) {
    return res.status(404).json({ error: `ไม่พบ user ID: ${userId}` });
  }
  
  req.user = user; // เก็บ user ไว้ใน req
  next();
});

// GET /users/:userId - ข้อมูล user
app.get('/users/:userId', (req, res) => {
  res.json({ user: req.user });
});

// GET /users/:userId/posts - posts ของ user
app.get('/users/:userId/posts', (req, res) => {
  const userPosts = posts.filter(p => p.userId === req.user.id);
  res.json({
    user: req.user.name,
    posts: userPosts,
    count: userPosts.length
  });
});

// GET /users/:userId/posts/:postId - post เดียวของ user
app.get('/users/:userId/posts/:postId', (req, res) => {
  const postId = parseInt(req.params.postId);
  const post = posts.find(p => p.id === postId && p.userId === req.user.id);
  
  if (!post) {
    return res.status(404).json({ error: 'ไม่พบ post นี้' });
  }
  
  res.json({ user: req.user.name, post });
});

app.listen(3000);
```

---

## 3. Query Strings

Query strings คือข้อมูลที่ส่งผ่าน URL หลังเครื่องหมาย `?`:

```
URL: /products?category=electronics&minPrice=1000&maxPrice=50000&sort=price&order=asc&page=1&limit=10
```

### การเข้าถึง Query String

```javascript
app.get('/products', (req, res) => {
  // req.query คือ object ของ query parameters
  console.log(req.query);
  // {
  //   category: 'electronics',
  //   minPrice: '1000',     <- ค่าเป็น string!
  //   maxPrice: '50000',
  //   sort: 'price',
  //   order: 'asc',
  //   page: '1',
  //   limit: '10'
  // }
  
  // แปลงค่า
  const {
    category,
    minPrice,
    maxPrice,
    sort = 'id',
    order = 'asc',
    page = '1',
    limit = '10'
  } = req.query;
  
  const filter = {
    category,
    minPrice: minPrice ? parseFloat(minPrice) : undefined,
    maxPrice: maxPrice ? parseFloat(maxPrice) : undefined,
  };
  
  const pagination = {
    page: parseInt(page),
    limit: parseInt(limit),
    offset: (parseInt(page) - 1) * parseInt(limit)
  };
  
  res.json({ filter, sort, order, pagination });
});
```

### Query String แบบ Array

```javascript
// URL: /products?tags=sale&tags=new&tags=featured
app.get('/products', (req, res) => {
  const { tags } = req.query;
  
  // tags อาจเป็น string หรือ array
  const tagArray = Array.isArray(tags) ? tags : tags ? [tags] : [];
  
  console.log(tagArray); // ['sale', 'new', 'featured']
  res.json({ tags: tagArray });
});
```

### ตัวอย่าง Search และ Filter API

```javascript
const express = require('express');
const app = express();

// ข้อมูลสินค้า
const products = [
  { id: 1, name: 'iPhone 15', category: 'electronics', price: 40000, brand: 'Apple', inStock: true },
  { id: 2, name: 'Samsung Galaxy S24', category: 'electronics', price: 35000, brand: 'Samsung', inStock: true },
  { id: 3, name: 'MacBook Pro', category: 'computers', price: 90000, brand: 'Apple', inStock: false },
  { id: 4, name: 'เสื้อยืด', category: 'clothing', price: 299, brand: 'Generic', inStock: true },
  { id: 5, name: 'กางเกงยีนส์', category: 'clothing', price: 890, brand: 'Levi\'s', inStock: true }
];

app.get('/api/products', (req, res) => {
  const {
    search,       // ค้นหาจากชื่อ
    category,     // กรองตาม category
    brand,        // กรองตาม brand
    minPrice,     // ราคาต่ำสุด
    maxPrice,     // ราคาสูงสุด
    inStock,      // มีสินค้าหรือไม่
    sort = 'id',  // เรียงตาม field
    order = 'asc', // asc หรือ desc
    page = '1',   // หน้าที่
    limit = '10'  // จำนวนต่อหน้า
  } = req.query;
  
  let result = [...products];
  
  // ========== Filtering ==========
  
  // ค้นหาจากชื่อ (case-insensitive)
  if (search) {
    const searchLower = search.toLowerCase();
    result = result.filter(p => 
      p.name.toLowerCase().includes(searchLower)
    );
  }
  
  // กรองตาม category
  if (category) {
    result = result.filter(p => p.category === category);
  }
  
  // กรองตาม brand
  if (brand) {
    result = result.filter(p => p.brand === brand);
  }
  
  // กรองตามราคา
  if (minPrice) {
    result = result.filter(p => p.price >= parseFloat(minPrice));
  }
  if (maxPrice) {
    result = result.filter(p => p.price <= parseFloat(maxPrice));
  }
  
  // กรองตามสินค้าคงเหลือ
  if (inStock !== undefined) {
    const stockFilter = inStock === 'true';
    result = result.filter(p => p.inStock === stockFilter);
  }
  
  // ========== Sorting ==========
  const validSortFields = ['id', 'name', 'price'];
  const sortField = validSortFields.includes(sort) ? sort : 'id';
  const sortOrder = order === 'desc' ? -1 : 1;
  
  result.sort((a, b) => {
    if (a[sortField] < b[sortField]) return -1 * sortOrder;
    if (a[sortField] > b[sortField]) return 1 * sortOrder;
    return 0;
  });
  
  // ========== Pagination ==========
  const pageNum = Math.max(1, parseInt(page));
  const limitNum = Math.min(100, Math.max(1, parseInt(limit)));
  const offset = (pageNum - 1) * limitNum;
  const total = result.length;
  const totalPages = Math.ceil(total / limitNum);
  
  const paginatedResult = result.slice(offset, offset + limitNum);
  
  res.json({
    success: true,
    data: paginatedResult,
    pagination: {
      total,
      totalPages,
      currentPage: pageNum,
      limit: limitNum,
      hasNextPage: pageNum < totalPages,
      hasPrevPage: pageNum > 1
    }
  });
});

app.listen(3000);
```

---

## 4. Route Patterns และ Wildcards

Express รองรับ patterns พิเศษในการกำหนด routes:

```javascript
const express = require('express');
const app = express();

// ==============================
// String Patterns
// ==============================

// ? - ตัวอักษรก่อนหน้าเป็น optional
// match: /abt หรือ /abct
app.get('/ab?ct', (req, res) => {
  res.send('Matched: /abt หรือ /abct');
});

// + - ตัวอักษรก่อนหน้า 1 ตัวหรือมากกว่า
// match: /abct, /abbct, /abbbct
app.get('/ab+ct', (req, res) => {
  res.send('Matched: /ab+ct pattern');
});

// * - wildcard (ตัวอักษรอะไรก็ได้)
// match: /abct, /abXct, /abXYZct
app.get('/ab*ct', (req, res) => {
  res.send('Matched: /ab*ct pattern');
});

// () - grouping
// match: /abcd หรือ /abXcd
app.get('/ab(cd)?e', (req, res) => {
  res.send('Matched: /abe หรือ /abcde');
});

// ==============================
// Regular Expressions
// ==============================

// ใช้ regex สำหรับ path matching
app.get(/.*fly$/, (req, res) => {
  // match: /butterfly, /dragonfly, /anything-fly
  res.send(`Matched URL ที่ลงท้าย fly: ${req.url}`);
});

// match เฉพาะ UUID
app.get(/^\/users\/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/, (req, res) => {
  res.send('Matched UUID');
});

// ==============================
// Wildcard สำหรับ catch-all
// ==============================

// Express v5 ใช้ :path(*) สำหรับ catch-all
// Express v4 ใช้ *
app.get('/files/*', (req, res) => {
  const filePath = req.params[0]; // ส่วนหลัง /files/
  res.json({
    requestedFile: filePath,
    fullPath: req.url
  });
});

app.listen(3000);
```

---

## 5. Router.route() Chaining

`router.route()` ช่วยให้เขียน routes สำหรับ path เดียวกันได้ชัดเจนขึ้น:

```javascript
const express = require('express');
const app = express();
app.use(express.json());

// แบบไม่ใช้ route chaining (ซ้ำซ้อน)
app.get('/products/:id', getProduct);
app.put('/products/:id', updateProduct);
app.delete('/products/:id', deleteProduct);

// แบบใช้ route chaining (สะอาดกว่า)
app.route('/products/:id')
  .get(getProduct)
  .put(updateProduct)
  .delete(deleteProduct);

// ฟังก์ชันจัดการ
function getProduct(req, res) {
  res.json({ action: 'GET', id: req.params.id });
}

function updateProduct(req, res) {
  res.json({ action: 'PUT', id: req.params.id, data: req.body });
}

function deleteProduct(req, res) {
  res.json({ action: 'DELETE', id: req.params.id });
}

// ตัวอย่างเต็มๆ กับ products API
let products = [
  { id: 1, name: 'Product A', price: 100 },
  { id: 2, name: 'Product B', price: 200 }
];

// Routes สำหรับ /products (collection)
app.route('/products')
  .get((req, res) => {
    res.json({ products, count: products.length });
  })
  .post((req, res) => {
    const newProduct = {
      id: products.length + 1,
      ...req.body,
      createdAt: new Date()
    };
    products.push(newProduct);
    res.status(201).json(newProduct);
  });

// Routes สำหรับ /products/:id (single item)
app.route('/products/:id')
  .get((req, res) => {
    const product = products.find(p => p.id === parseInt(req.params.id));
    if (!product) return res.status(404).json({ error: 'ไม่พบสินค้า' });
    res.json(product);
  })
  .put((req, res) => {
    const idx = products.findIndex(p => p.id === parseInt(req.params.id));
    if (idx === -1) return res.status(404).json({ error: 'ไม่พบสินค้า' });
    products[idx] = { ...products[idx], ...req.body };
    res.json(products[idx]);
  })
  .patch((req, res) => {
    const idx = products.findIndex(p => p.id === parseInt(req.params.id));
    if (idx === -1) return res.status(404).json({ error: 'ไม่พบสินค้า' });
    Object.assign(products[idx], req.body);
    res.json(products[idx]);
  })
  .delete((req, res) => {
    const idx = products.findIndex(p => p.id === parseInt(req.params.id));
    if (idx === -1) return res.status(404).json({ error: 'ไม่พบสินค้า' });
    const deleted = products.splice(idx, 1)[0];
    res.json({ message: 'ลบสำเร็จ', product: deleted });
  });

app.listen(3000);
```

---

## 6. Express Router

`express.Router` เป็น mini-application ที่ช่วยแยก routes ออกเป็นไฟล์ต่างๆ:

### สร้าง Router แยกไฟล์

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();

// ข้อมูลจำลอง
let users = [
  { id: 1, name: 'สมชาย', email: 'somchai@example.com', role: 'user' },
  { id: 2, name: 'สมหญิง', email: 'somying@example.com', role: 'admin' }
];

// GET /users
router.get('/', (req, res) => {
  res.json({ users });
});

// GET /users/:id
router.get('/:id', (req, res) => {
  const user = users.find(u => u.id === parseInt(req.params.id));
  if (!user) return res.status(404).json({ error: 'ไม่พบผู้ใช้' });
  res.json({ user });
});

// POST /users
router.post('/', (req, res) => {
  const { name, email } = req.body;
  if (!name || !email) {
    return res.status(400).json({ error: 'กรุณาระบุ name และ email' });
  }
  const newUser = { id: users.length + 1, name, email, role: 'user' };
  users.push(newUser);
  res.status(201).json({ user: newUser });
});

// PUT /users/:id
router.put('/:id', (req, res) => {
  const idx = users.findIndex(u => u.id === parseInt(req.params.id));
  if (idx === -1) return res.status(404).json({ error: 'ไม่พบผู้ใช้' });
  users[idx] = { ...users[idx], ...req.body, id: users[idx].id };
  res.json({ user: users[idx] });
});

// DELETE /users/:id
router.delete('/:id', (req, res) => {
  const idx = users.findIndex(u => u.id === parseInt(req.params.id));
  if (idx === -1) return res.status(404).json({ error: 'ไม่พบผู้ใช้' });
  const deleted = users.splice(idx, 1)[0];
  res.json({ message: 'ลบสำเร็จ', user: deleted });
});

module.exports = router;
```

```javascript
// routes/products.js
const express = require('express');
const router = express.Router();

let products = [
  { id: 1, name: 'Laptop', price: 35000, category: 'electronics' },
  { id: 2, name: 'Mouse', price: 590, category: 'electronics' },
  { id: 3, name: 'Desk', price: 4500, category: 'furniture' }
];

router.get('/', (req, res) => {
  const { category } = req.query;
  let result = products;
  if (category) result = products.filter(p => p.category === category);
  res.json({ products: result });
});

router.get('/:id', (req, res) => {
  const product = products.find(p => p.id === parseInt(req.params.id));
  if (!product) return res.status(404).json({ error: 'ไม่พบสินค้า' });
  res.json({ product });
});

router.post('/', (req, res) => {
  const product = { id: products.length + 1, ...req.body };
  products.push(product);
  res.status(201).json({ product });
});

router.put('/:id', (req, res) => {
  const idx = products.findIndex(p => p.id === parseInt(req.params.id));
  if (idx === -1) return res.status(404).json({ error: 'ไม่พบสินค้า' });
  products[idx] = { id: products[idx].id, ...req.body };
  res.json({ product: products[idx] });
});

router.delete('/:id', (req, res) => {
  const idx = products.findIndex(p => p.id === parseInt(req.params.id));
  if (idx === -1) return res.status(404).json({ error: 'ไม่พบสินค้า' });
  const deleted = products.splice(idx, 1)[0];
  res.json({ message: 'ลบสำเร็จ', product: deleted });
});

module.exports = router;
```

```javascript
// app.js - ใช้ Router modules
const express = require('express');
const app = express();

// นำเข้า routers
const userRoutes = require('./routes/users');
const productRoutes = require('./routes/products');

app.use(express.json());

// Mount routers ที่ path prefix
app.use('/api/users', userRoutes);
app.use('/api/products', productRoutes);

// ผลลัพธ์ที่ได้:
// GET    /api/users           -> userRoutes GET /
// GET    /api/users/:id       -> userRoutes GET /:id
// POST   /api/users           -> userRoutes POST /
// GET    /api/products        -> productRoutes GET /
// GET    /api/products/:id    -> productRoutes GET /:id

app.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

---

## 7. Route Groups

Route groups ช่วยจัดระเบียบ routes ที่มี prefix หรือ middleware เหมือนกัน:

```javascript
// routes/api/v1/index.js
const express = require('express');
const router = express.Router();

const userRoutes = require('./users');
const productRoutes = require('./products');
const orderRoutes = require('./orders');

// Middleware สำหรับทุก API routes
router.use((req, res, next) => {
  console.log('API v1 request:', req.method, req.path);
  next();
});

// Mount sub-routers
router.use('/users', userRoutes);
router.use('/products', productRoutes);
router.use('/orders', orderRoutes);

module.exports = router;
```

```javascript
// app.js
const express = require('express');
const app = express();
app.use(express.json());

const apiV1Routes = require('./routes/api/v1');
const apiV2Routes = require('./routes/api/v2'); // future

// Version-based routing
app.use('/api/v1', apiV1Routes);
app.use('/api/v2', apiV2Routes);

// ผลลัพธ์:
// /api/v1/users
// /api/v1/products
// /api/v2/users (รุ่นใหม่กว่า)

app.listen(3000);
```

### Nested Routers

```javascript
// routes/admin.js
const express = require('express');
const router = express.Router();

// Middleware เฉพาะ admin routes
const requireAdmin = (req, res, next) => {
  // ตรวจสอบ admin role
  if (req.headers['x-role'] !== 'admin') {
    return res.status(403).json({ error: 'ต้องเป็น admin' });
  }
  next();
};

router.use(requireAdmin); // ใช้กับทุก routes ใน router นี้

router.get('/dashboard', (req, res) => {
  res.json({ page: 'Admin Dashboard' });
});

router.get('/users', (req, res) => {
  res.json({ page: 'Manage Users' });
});

router.get('/reports', (req, res) => {
  res.json({ page: 'Reports' });
});

module.exports = router;
```

---

## 8. Practical: Product Catalog Routes

สร้าง Product Catalog API ที่สมบูรณ์:

### โครงสร้างไฟล์

```
product-catalog/
├── routes/
│   ├── products.js
│   ├── categories.js
│   └── index.js
├── data/
│   └── products.js
├── middleware/
│   └── validate.js
└── app.js
```

### data/products.js

```javascript
// data/products.js
const products = [
  {
    id: 1,
    name: 'iPhone 15 Pro',
    slug: 'iphone-15-pro',
    categoryId: 1,
    price: 44900,
    salePrice: null,
    brand: 'Apple',
    description: 'สมาร์ทโฟนรุ่นล่าสุดจาก Apple',
    images: ['iphone15pro-1.jpg', 'iphone15pro-2.jpg'],
    specs: { ram: '8GB', storage: '256GB', color: 'Natural Titanium' },
    inStock: true,
    quantity: 50,
    rating: 4.8,
    reviewCount: 234,
    tags: ['smartphone', 'apple', 'flagship'],
    createdAt: '2024-01-01T00:00:00Z'
  },
  {
    id: 2,
    name: 'Samsung Galaxy S24',
    slug: 'samsung-galaxy-s24',
    categoryId: 1,
    price: 35900,
    salePrice: 32900,
    brand: 'Samsung',
    description: 'สมาร์ทโฟน Android รุ่นใหม่',
    images: ['s24-1.jpg'],
    specs: { ram: '8GB', storage: '128GB', color: 'Onyx Black' },
    inStock: true,
    quantity: 30,
    rating: 4.6,
    reviewCount: 178,
    tags: ['smartphone', 'samsung', 'android'],
    createdAt: '2024-01-02T00:00:00Z'
  },
  {
    id: 3,
    name: 'MacBook Pro 14"',
    slug: 'macbook-pro-14',
    categoryId: 2,
    price: 89900,
    salePrice: null,
    brand: 'Apple',
    description: 'Laptop สำหรับ professional',
    images: ['macbook-1.jpg'],
    specs: { ram: '18GB', storage: '512GB', chip: 'M3 Pro' },
    inStock: false,
    quantity: 0,
    rating: 4.9,
    reviewCount: 89,
    tags: ['laptop', 'apple', 'professional'],
    createdAt: '2024-01-03T00:00:00Z'
  }
];

const categories = [
  { id: 1, name: 'สมาร์ทโฟน', slug: 'smartphones', parentId: null },
  { id: 2, name: 'แล็ปท็อป', slug: 'laptops', parentId: null },
  { id: 3, name: 'อุปกรณ์เสริม', slug: 'accessories', parentId: null }
];

module.exports = { products, categories };
```

### routes/products.js

```javascript
// routes/products.js
const express = require('express');
const router = express.Router();
const { products } = require('../data/products');

// GET /api/products - ดูสินค้าทั้งหมด พร้อม filter และ pagination
router.get('/', (req, res) => {
  const {
    search, category, brand, minPrice, maxPrice,
    inStock, onSale, tags,
    sort = 'id', order = 'asc',
    page = 1, limit = 12
  } = req.query;
  
  let result = [...products];
  
  // Search
  if (search) {
    const q = search.toLowerCase();
    result = result.filter(p =>
      p.name.toLowerCase().includes(q) ||
      p.description.toLowerCase().includes(q) ||
      p.brand.toLowerCase().includes(q)
    );
  }
  
  // Filter by category
  if (category) {
    result = result.filter(p => p.categoryId === parseInt(category));
  }
  
  // Filter by brand
  if (brand) {
    result = result.filter(p =>
      p.brand.toLowerCase() === brand.toLowerCase()
    );
  }
  
  // Price range
  if (minPrice) result = result.filter(p => p.price >= parseFloat(minPrice));
  if (maxPrice) result = result.filter(p => p.price <= parseFloat(maxPrice));
  
  // In stock
  if (inStock === 'true') result = result.filter(p => p.inStock);
  
  // On sale
  if (onSale === 'true') result = result.filter(p => p.salePrice !== null);
  
  // Filter by tags
  if (tags) {
    const tagList = Array.isArray(tags) ? tags : [tags];
    result = result.filter(p =>
      tagList.some(tag => p.tags.includes(tag))
    );
  }
  
  // Sort
  const sortOrder = order === 'desc' ? -1 : 1;
  result.sort((a, b) => {
    if (sort === 'price') return (a.price - b.price) * sortOrder;
    if (sort === 'rating') return (a.rating - b.rating) * sortOrder;
    if (sort === 'newest') return new Date(b.createdAt) - new Date(a.createdAt);
    return (a.id - b.id) * sortOrder;
  });
  
  // Pagination
  const pageNum = parseInt(page);
  const limitNum = parseInt(limit);
  const total = result.length;
  const paginatedData = result.slice((pageNum - 1) * limitNum, pageNum * limitNum);
  
  res.json({
    success: true,
    data: paginatedData,
    meta: {
      total,
      page: pageNum,
      limit: limitNum,
      totalPages: Math.ceil(total / limitNum)
    }
  });
});

// GET /api/products/featured - สินค้า featured
router.get('/featured', (req, res) => {
  const featured = products
    .filter(p => p.rating >= 4.7 && p.inStock)
    .slice(0, 4);
  res.json({ success: true, data: featured });
});

// GET /api/products/on-sale - สินค้าลดราคา
router.get('/on-sale', (req, res) => {
  const onSale = products
    .filter(p => p.salePrice !== null)
    .map(p => ({
      ...p,
      discount: Math.round((1 - p.salePrice / p.price) * 100)
    }));
  res.json({ success: true, data: onSale });
});

// GET /api/products/:id - ดูสินค้าเดียว
router.get('/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const product = products.find(p => p.id === id);
  
  if (!product) {
    return res.status(404).json({
      success: false,
      message: `ไม่พบสินค้า ID: ${id}`
    });
  }
  
  // ดึงสินค้าที่เกี่ยวข้อง (category เดียวกัน)
  const related = products
    .filter(p => p.categoryId === product.categoryId && p.id !== product.id)
    .slice(0, 4);
  
  res.json({
    success: true,
    data: product,
    related
  });
});

// GET /api/products/slug/:slug - ดูสินค้าโดย slug
router.get('/slug/:slug', (req, res) => {
  const product = products.find(p => p.slug === req.params.slug);
  
  if (!product) {
    return res.status(404).json({
      success: false,
      message: `ไม่พบสินค้า: ${req.params.slug}`
    });
  }
  
  res.json({ success: true, data: product });
});

// POST /api/products - เพิ่มสินค้าใหม่
router.post('/', (req, res) => {
  const required = ['name', 'price', 'categoryId', 'brand'];
  const missing = required.filter(f => !req.body[f]);
  
  if (missing.length > 0) {
    return res.status(400).json({
      success: false,
      message: `ขาด fields: ${missing.join(', ')}`
    });
  }
  
  const newProduct = {
    id: Math.max(...products.map(p => p.id)) + 1,
    slug: req.body.name.toLowerCase().replace(/\s+/g, '-'),
    inStock: true,
    quantity: 0,
    rating: 0,
    reviewCount: 0,
    tags: [],
    images: [],
    salePrice: null,
    ...req.body,
    createdAt: new Date().toISOString()
  };
  
  products.push(newProduct);
  res.status(201).json({ success: true, data: newProduct });
});

module.exports = router;
```

### app.js

```javascript
// app.js
const express = require('express');
const app = express();
app.use(express.json());

const productRoutes = require('./routes/products');
const categoryRoutes = require('./routes/categories');

app.use('/api/products', productRoutes);
app.use('/api/categories', categoryRoutes);

app.use((req, res) => {
  res.status(404).json({ error: 'Not Found' });
});

app.listen(3000, () => {
  console.log('Product Catalog API: http://localhost:3000');
});
```

---

## 9. Exercise

### Exercise 1: Blog API Routes

สร้าง Blog API ที่มี routes ดังนี้:

```
GET    /api/posts                    - ดูทุก posts (พร้อม filter และ pagination)
GET    /api/posts/search?q=          - ค้นหา posts
GET    /api/posts/:id                - ดู post เดียว
GET    /api/posts/slug/:slug         - ดู post โดย slug
POST   /api/posts                    - สร้าง post ใหม่
PUT    /api/posts/:id                - อัปเดต post
DELETE /api/posts/:id                - ลบ post
GET    /api/posts/:id/comments       - ดู comments ของ post
POST   /api/posts/:id/comments       - เพิ่ม comment
DELETE /api/posts/:id/comments/:cid  - ลบ comment
```

### Exercise 2: Express Router แยกไฟล์

แยก routes ออกเป็นไฟล์ต่างๆ:
- `routes/auth.js` - login, register, logout
- `routes/users.js` - user CRUD
- `routes/posts.js` - post CRUD
- `app.js` - mount ทุก routers

### Exercise 3: API Versioning

สร้าง API ที่มี versioning:
- `/api/v1/users` - version เดิม (return เฉพาะ id, name, email)
- `/api/v2/users` - version ใหม่ (return ข้อมูลเพิ่มเติม เช่น avatar, role)

---

## สรุป Part 12

ในบทนี้เราได้เรียนรู้:

1. **Route Methods** - GET, POST, PUT, PATCH, DELETE และความหมาย
2. **Route Parameters** - `:id`, หลาย params, `app.param()`
3. **Query Strings** - `req.query`, filtering, pagination
4. **Route Patterns** - wildcards, regex
5. **Route Chaining** - `router.route().get().post()`
6. **Express Router** - แยก routes เป็นโมดูล
7. **Route Groups** - จัดระเบียบ routes ด้วย prefix

---

*Part 12 | Node.js/Express.js Course | ขั้นตอนที่ 12 จาก 1000*
