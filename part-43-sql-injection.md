# Part 43: SQL Injection Prevention

> ขั้นตอนที่ 43-43 จาก 1000

---

## สารบัญ

1. [SQL Injection คืออะไร](#sql-injection-คืออะไร)
2. [ตัวอย่าง Attack](#ตัวอยาง-attack)
3. [Parameterized Queries](#parameterized-queries)
4. [ORM Protection](#orm-protection)
5. [Input Sanitization](#input-sanitization)
6. [Additional Defenses](#additional-defenses)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## SQL Injection คืออะไร

SQL Injection คือ attack ที่ attacker ใส่ SQL code ลงใน input fields เพื่อ manipulate database queries

### ประเภทของ SQL Injection

```
1. Classic SQL Injection - ดึงข้อมูล
2. Blind SQL Injection   - ไม่เห็น error แต่ยังดึงข้อมูลได้
3. Time-based Blind     - ใช้ time delay
4. Out-of-band          - ส่งข้อมูลผ่าน channel อื่น
```

---

## ตัวอย่าง Attack

### Login Bypass

```javascript
// ❌ Vulnerable code
async function login(email, password) {
  const query = `
    SELECT * FROM users 
    WHERE email = '${email}' 
    AND password = '${password}'
  `;
  
  const result = await db.query(query);
  return result.rows[0];
}

// Attack payload:
// email: admin@example.com' --
// password: anything

// Query ที่เกิดขึ้น:
// SELECT * FROM users WHERE email = 'admin@example.com' --' AND password = 'anything'
// -- หมายถึง comment ใน SQL ทำให้ password check ถูก skip
// ผล: login ได้โดยไม่ต้องรู้ password!
```

### Data Extraction

```javascript
// ❌ Vulnerable search
async function searchUsers(name) {
  const query = `SELECT id, name, email FROM users WHERE name LIKE '%${name}%'`;
  return db.query(query);
}

// Attack payload:
// name: ' UNION SELECT username, password, credit_card FROM admin_users --
// 
// Query ที่เกิดขึ้น:
// SELECT id, name, email FROM users WHERE name LIKE '%' 
// UNION SELECT username, password, credit_card FROM admin_users --'%'
// ผล: ได้ข้อมูล admin users ทั้งหมด!
```

### Data Deletion

```javascript
// ❌ Extremely dangerous
async function getUserById(id) {
  const query = `SELECT * FROM users WHERE id = ${id}`;
  return db.query(query);
}

// Attack payload:
// id: 1; DROP TABLE users; --
// 
// Query ที่เกิดขึ้น:
// SELECT * FROM users WHERE id = 1; DROP TABLE users; --
// ผล: ลบ table users ทั้งหมด!
```

---

## Parameterized Queries

Parameterized Queries (Prepared Statements) เป็นวิธีหลักในการป้องกัน SQL Injection

### PostgreSQL ด้วย pg

```javascript
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// ✅ Safe - Parameterized query
async function login(email, password) {
  const query = 'SELECT * FROM users WHERE email = $1 AND password_hash = $2';
  const values = [email, hashPassword(password)];
  
  const result = await pool.query(query, values);
  return result.rows[0];
}

// ✅ Safe - Search
async function searchUsers(name) {
  const query = 'SELECT id, name, email FROM users WHERE name ILIKE $1';
  const values = [`%${name}%`]; // ใส่ wildcards ใน value ไม่ใช่ใน query
  
  const result = await pool.query(query, values);
  return result.rows;
}

// ✅ Safe - Multiple params
async function createUser(name, email, passwordHash) {
  const query = `
    INSERT INTO users (name, email, password_hash, created_at)
    VALUES ($1, $2, $3, NOW())
    RETURNING id, name, email, created_at
  `;
  const values = [name, email, passwordHash];
  
  const result = await pool.query(query, values);
  return result.rows[0];
}

// ✅ Safe - Dynamic column selection (whitelist approach)
async function getUserFields(userId, fields) {
  const ALLOWED_FIELDS = ['id', 'name', 'email', 'created_at'];
  
  // Whitelist validation
  const safeFields = fields
    .filter(f => ALLOWED_FIELDS.includes(f))
    .join(', ');
  
  if (!safeFields) {
    throw new Error('No valid fields specified');
  }
  
  // Column names ต้องใช้ whitelist ไม่ใช่ parameterization
  const query = `SELECT ${safeFields} FROM users WHERE id = $1`;
  const result = await pool.query(query, [userId]);
  return result.rows[0];
}
```

### MySQL ด้วย mysql2

```javascript
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
});

// ✅ Safe - ใช้ ? สำหรับ placeholders
async function getUser(id) {
  const [rows] = await pool.execute(
    'SELECT * FROM users WHERE id = ?',
    [id]
  );
  return rows[0];
}

// ✅ Safe - Multiple params
async function updateUser(id, name, email) {
  const [result] = await pool.execute(
    'UPDATE users SET name = ?, email = ?, updated_at = NOW() WHERE id = ?',
    [name, email, id]
  );
  return result;
}

// ✅ Safe - IN clause
async function getUsersByIds(ids) {
  if (!ids.length) return [];
  
  const placeholders = ids.map(() => '?').join(',');
  const [rows] = await pool.execute(
    `SELECT * FROM users WHERE id IN (${placeholders})`,
    ids
  );
  return rows;
}
```

### Prepared Statements สำหรับ Performance

```javascript
// Prepared statement - เร็วกว่าเมื่อรันซ้ำหลายครั้ง
const { Pool } = require('pg');
const pool = new Pool();

// เตรียม statement ล่วงหน้า
async function prepareStatements() {
  await pool.query({
    name: 'get-user',
    text: 'SELECT * FROM users WHERE id = $1',
  });
  
  await pool.query({
    name: 'get-user-by-email',
    text: 'SELECT * FROM users WHERE email = $1',
  });
}

// ใช้งาน prepared statement
async function getUserById(id) {
  const result = await pool.query({
    name: 'get-user',  // ใช้ cached plan
    values: [id],
  });
  return result.rows[0];
}
```

---

## ORM Protection

ORMs เช่น Sequelize, Prisma, TypeORM สร้าง parameterized queries ให้อัตโนมัติ

### Sequelize

```javascript
const { Sequelize, Op } = require('sequelize');
const sequelize = new Sequelize(process.env.DATABASE_URL);

// ✅ Safe - Sequelize ใช้ parameterized queries เสมอ
async function loginUser(email, password) {
  const user = await User.findOne({
    where: { email, password: hashPassword(password) },
  });
  return user;
}

// ✅ Safe - Complex queries
async function searchUsers(searchTerm, page, limit) {
  const offset = (page - 1) * limit;
  
  const { rows: users, count } = await User.findAndCountAll({
    where: {
      [Op.or]: [
        { name: { [Op.iLike]: `%${searchTerm}%` } },
        { email: { [Op.iLike]: `%${searchTerm}%` } },
      ],
      isActive: true,
    },
    limit,
    offset,
    order: [['createdAt', 'DESC']],
  });
  
  return { users, count };
}

// ⚠️ ระวัง - Raw queries ต้องใช้ replacements
async function rawQuery(userId) {
  // ❌ Dangerous
  const result = await sequelize.query(
    `SELECT * FROM users WHERE id = ${userId}`
  );
  
  // ✅ Safe
  const result = await sequelize.query(
    'SELECT * FROM users WHERE id = :id',
    {
      replacements: { id: userId },
      type: Sequelize.QueryTypes.SELECT,
    }
  );
  
  return result;
}
```

### Prisma

```javascript
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

// ✅ Safe - Prisma ใช้ parameterized queries
async function getUser(email, password) {
  return prisma.user.findFirst({
    where: {
      email,
      passwordHash: hashPassword(password),
    },
  });
}

// ✅ Safe - Complex filters
async function searchPosts(query, page, limit) {
  return prisma.post.findMany({
    where: {
      OR: [
        { title: { contains: query, mode: 'insensitive' } },
        { content: { contains: query, mode: 'insensitive' } },
      ],
      status: 'published',
    },
    include: {
      author: { select: { name: true, avatar: true } },
    },
    skip: (page - 1) * limit,
    take: limit,
    orderBy: { publishedAt: 'desc' },
  });
}

// ⚠️ Raw queries ต้องระวัง
async function rawExample(userId) {
  // ❌ Dangerous
  await prisma.$queryRawUnsafe(`SELECT * FROM users WHERE id = ${userId}`);
  
  // ✅ Safe
  await prisma.$queryRaw`SELECT * FROM users WHERE id = ${userId}`;
  // Template literal ที่ Prisma สร้าง parameterized query ให้
}
```

---

## Input Sanitization

```javascript
// ⚠️ Sanitization เป็น defense-in-depth ไม่ใช่ primary defense
// Primary defense คือ Parameterized Queries!

// 1. Type coercion
function parseId(value) {
  const id = parseInt(value, 10);
  if (isNaN(id) || id <= 0) {
    throw new Error('Invalid ID');
  }
  return id;
}

// 2. Whitelist validation
function validateOrderBy(field, direction) {
  const ALLOWED_FIELDS = ['name', 'email', 'createdAt', 'price'];
  const ALLOWED_DIRECTIONS = ['ASC', 'DESC'];
  
  if (!ALLOWED_FIELDS.includes(field)) {
    throw new Error(`Invalid sort field: ${field}`);
  }
  
  if (!ALLOWED_DIRECTIONS.includes(direction.toUpperCase())) {
    throw new Error(`Invalid sort direction: ${direction}`);
  }
  
  return `${field} ${direction.toUpperCase()}`;
}

// 3. Escape special characters (สำหรับ LIKE queries)
function escapeLikePattern(value) {
  return value.replace(/[%_\\]/g, '\\$&');
}

async function searchWithEscape(term) {
  const escaped = escapeLikePattern(term);
  return db.query('SELECT * FROM products WHERE name LIKE $1', [`%${escaped}%`]);
}
```

### Input Validation ด้วย Joi

```javascript
const Joi = require('joi');

// Define validation schemas
const loginSchema = Joi.object({
  email: Joi.string().email().required().max(255),
  password: Joi.string().min(8).max(128).required(),
});

const searchSchema = Joi.object({
  q: Joi.string().trim().max(100).pattern(/^[\w\s-]+$/).optional(),
  page: Joi.number().integer().min(1).default(1),
  limit: Joi.number().integer().min(1).max(100).default(10),
  sortBy: Joi.string().valid('name', 'email', 'createdAt').default('createdAt'),
  sortOrder: Joi.string().valid('asc', 'desc').default('desc'),
});

// Validation middleware
function validate(schema) {
  return (req, res, next) => {
    const { error, value } = schema.validate(req.body || req.query);
    
    if (error) {
      return res.status(400).json({
        error: 'Validation Error',
        details: error.details.map(d => d.message),
      });
    }
    
    req.validated = value;
    next();
  };
}

// ใช้งาน
app.post('/auth/login', validate(loginSchema), loginController);
app.get('/users', validate(searchSchema), searchController);
```

---

## Additional Defenses

### Principle of Least Privilege

```sql
-- สร้าง database user ที่มี permissions แค่ที่จำเป็น
CREATE USER app_user WITH PASSWORD 'strong_password';

-- Grant เฉพาะที่ต้องการ
GRANT SELECT, INSERT, UPDATE ON users TO app_user;
GRANT SELECT ON products TO app_user;

-- ไม่ควร grant:
-- GRANT ALL PRIVILEGES ON ALL TABLES TO app_user;
-- ไม่มี DROP, TRUNCATE, ALTER permissions
```

### Error Handling

```javascript
// ❌ ไม่ดี - แสดง database error ให้ user เห็น
app.get('/users/:id', async (req, res) => {
  try {
    const user = await db.query(`SELECT * FROM users WHERE id = ${req.params.id}`);
    res.json(user);
  } catch (error) {
    // แสดง stack trace และ SQL error
    res.status(500).json({ error: error.message }); 
  }
});

// ✅ ดี - ซ่อน internal errors
app.get('/users/:id', async (req, res) => {
  try {
    const id = parseInt(req.params.id);
    if (isNaN(id)) return res.status(400).json({ error: 'Invalid ID' });
    
    const result = await db.query('SELECT * FROM users WHERE id = $1', [id]);
    if (!result.rows[0]) return res.status(404).json({ error: 'Not found' });
    
    res.json(result.rows[0]);
  } catch (error) {
    // Log internally
    console.error('Database error:', error);
    
    // ไม่แสดง details ให้ user
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

### WAF (Web Application Firewall)

```javascript
// ใช้ middleware ตรวจจับ SQL injection patterns
function sqlInjectionDetector(req, res, next) {
  const SQL_PATTERNS = [
    /(\b(SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|EXEC|UNION)\b)/i,
    /('|--|;|\/\*|\*\/)/,
    /(OR\s+\d+\s*=\s*\d+)/i,
  ];
  
  const checkValue = (value) => {
    if (typeof value !== 'string') return false;
    return SQL_PATTERNS.some(pattern => pattern.test(value));
  };
  
  const suspicious = (
    Object.values(req.query).some(checkValue) ||
    Object.values(req.body || {}).some(checkValue)
  );
  
  if (suspicious) {
    console.warn('Potential SQL injection detected:', {
      ip: req.ip,
      path: req.path,
      query: req.query,
    });
    
    return res.status(400).json({ error: 'Invalid input detected' });
  }
  
  next();
}

// ใช้เฉพาะบาง routes
app.use('/api/', sqlInjectionDetector);
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Fix Vulnerable Code

แก้ไข vulnerable code ต่อไปนี้:

```javascript
// ❌ Vulnerable - แก้ไขให้ปลอดภัย
async function searchProducts(query) {
  return db.query(`SELECT * FROM products WHERE name LIKE '%${query}%'`);
}

async function getUserByEmail(email) {
  return db.query(`SELECT * FROM users WHERE email = '${email}'`);
}

async function deleteUser(id) {
  return db.query(`DELETE FROM users WHERE id = ${id}`);
}

async function getOrdersByStatus(userId, status) {
  return db.query(
    `SELECT * FROM orders WHERE user_id = ${userId} AND status = '${status}'`
  );
}
```

### แบบฝึกหัดที่ 2: Security Audit

ตรวจสอบโค้ดในโปรเจกต์:
1. หาทุก database query
2. ตรวจสอบว่าใช้ parameterized queries
3. ตรวจสอบ raw SQL queries
4. เขียน test สำหรับ SQL injection

### แบบฝึกหัดที่ 3: ORM Migration

เปลี่ยนจาก raw queries เป็น Prisma:
1. สร้าง schema
2. เปลี่ยน queries ทั้งหมด
3. ทดสอบว่า SQL injection ไม่ได้ผลอีกต่อไป

---

## สรุป

SQL Injection ป้องกันได้ง่ายด้วย Parameterized Queries

| Layer | Defense |
|-------|---------|
| Primary | Parameterized queries เสมอ |
| Secondary | Input validation และ sanitization |
| Database | Least privilege permissions |
| Application | Error handling ที่ไม่ leak info |
| Network | WAF, monitoring |

**ถัดไป**: [Part 44: XSS Prevention →](./part-44-xss-prevention.md)
