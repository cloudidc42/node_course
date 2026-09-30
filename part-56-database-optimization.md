# Part 56: Database Optimization (การเพิ่มประสิทธิภาพฐานข้อมูล)
## ขั้นตอนที่ 56-56 จาก 1000

---

## บทนำ

Database เป็น bottleneck หลักของ web application ส่วนใหญ่ บทนี้จะครอบคลุมเทคนิคการ optimize ฐานข้อมูลตั้งแต่ query optimization, indexing, จนถึง read replicas

---

## 56.1 Query Optimization

### EXPLAIN ANALYZE

```sql
-- ดู query execution plan
EXPLAIN ANALYZE
SELECT u.*, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
GROUP BY u.id
ORDER BY order_count DESC
LIMIT 20;

-- Output ตัวอย่าง:
/*
Limit  (cost=1234.56..1234.58 rows=20 width=64) (actual time=45.123..45.130 rows=20 loops=1)
  ->  Sort  (cost=1234.56..1259.56 rows=10000 width=64) (actual time=45.121..45.125 rows=20 loops=1)
        Sort Key: (count(o.id)) DESC
        Sort Method: top-N heapsort  Memory: 27kB
        ->  HashAggregate  (cost=987.65..1037.65 rows=5000 width=64) (actual time=38.456..42.123 rows=5000 loops=1)
              ->  Hash Left Join  (cost=456.78..875.43 rows=22443 width=56)...
                    Hash Cond: (u.id = o.user_id)
                    ->  Seq Scan on users u  (cost=0..234.56 rows=5000 width=48)
                          Filter: ((status)::text = 'active')
*/
```

### วิเคราะห์ EXPLAIN Output

```javascript
// src/utils/queryAnalyzer.js

/**
 * Helper วิเคราะห์ query performance
 */
const analyzeQuery = async (sequelize, query, replacements = {}) => {
  const [results] = await sequelize.query(
    `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${query}`,
    { replacements }
  );
  
  const plan = results[0]['QUERY PLAN'][0];
  
  return {
    executionTime: plan['Execution Time'],
    planningTime: plan['Planning Time'],
    totalTime: plan['Execution Time'] + plan['Planning Time'],
    plan: plan.Plan,
    warnings: extractWarnings(plan.Plan)
  };
};

const extractWarnings = (plan) => {
  const warnings = [];
  
  // ตรวจหา Sequential Scans บน large tables
  if (plan['Node Type'] === 'Seq Scan' && plan['Actual Rows'] > 10000) {
    warnings.push({
      type: 'SEQ_SCAN',
      table: plan['Relation Name'],
      rows: plan['Actual Rows'],
      suggestion: `Consider adding index on ${plan['Filter'] || 'filter column'}`
    });
  }
  
  // ตรวจหา nested loops ที่มี cost สูง
  if (plan['Node Type'] === 'Nested Loop' && plan['Total Cost'] > 1000) {
    warnings.push({
      type: 'EXPENSIVE_NESTED_LOOP',
      cost: plan['Total Cost'],
      suggestion: 'Consider using Hash Join or adding indexes'
    });
  }
  
  // ตรวจ children
  if (plan.Plans) {
    plan.Plans.forEach(child => {
      warnings.push(...extractWarnings(child));
    });
  }
  
  return warnings;
};

module.exports = { analyzeQuery };
```

### Query Optimization Examples

```javascript
// src/examples/optimizedQueries.js
const { Op, literal, fn, col, QueryTypes } = require('sequelize');
const { sequelize } = require('../config/database');

/**
 * ❌ ปัญหา: N+1 Query Problem
 */
const getOrdersWithItemsBad = async () => {
  const orders = await Order.findAll({ limit: 100 });
  
  // สำหรับ order แต่ละอัน ทำ query แยก = N+1 queries!
  for (const order of orders) {
    order.items = await OrderItem.findAll({ where: { orderId: order.id } });
  }
  
  return orders;
};

/**
 * ✅ แก้ไข: ใช้ include (JOIN)
 */
const getOrdersWithItemsGood = async () => {
  return await Order.findAll({
    limit: 100,
    include: [{
      model: OrderItem,
      as: 'items',
      include: [{
        model: Product,
        as: 'product',
        attributes: ['id', 'name', 'price']
      }]
    }]
  });
};

/**
 * ❌ ปัญหา: Select ทุก column
 */
const getUsersBad = async () => {
  return await User.findAll();  // SELECT *
};

/**
 * ✅ แก้ไข: Select เฉพาะที่ต้องการ
 */
const getUsersGood = async () => {
  return await User.findAll({
    attributes: ['id', 'name', 'email', 'createdAt'],
    where: { status: 'active' }
  });
};

/**
 * ✅ Batch queries แทน individual queries
 */
const getUsersByIds = async (userIds) => {
  return await User.findAll({
    where: { id: { [Op.in]: userIds } }
  });
};

/**
 * ✅ Raw SQL สำหรับ complex queries
 */
const getTopSellingProducts = async (limit = 10) => {
  return await sequelize.query(`
    SELECT 
      p.id,
      p.name,
      SUM(oi.quantity) as total_sold,
      SUM(oi.quantity * oi.price) as revenue
    FROM products p
    INNER JOIN order_items oi ON p.id = oi.product_id
    INNER JOIN orders o ON oi.order_id = o.id
    WHERE o.status = 'completed'
      AND o.created_at >= NOW() - INTERVAL '30 days'
    GROUP BY p.id, p.name
    ORDER BY total_sold DESC
    LIMIT :limit
  `, {
    replacements: { limit },
    type: QueryTypes.SELECT
  });
};
```

---

## 56.2 Indexing Strategies

### Types of Indexes

```sql
-- 1. B-tree Index (default) - สำหรับ =, <, >, BETWEEN
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_status_created ON users(status, created_at DESC);

-- 2. Hash Index - สำหรับ = เท่านั้น แต่เร็วกว่า B-tree
CREATE INDEX idx_sessions_token ON sessions USING hash(token);

-- 3. GIN Index - สำหรับ full-text search, arrays, JSONB
CREATE INDEX idx_products_search ON products USING GIN(search_vector);
CREATE INDEX idx_products_tags ON products USING GIN(tags);

-- 4. BRIN Index - สำหรับ large tables ที่มี natural ordering
CREATE INDEX idx_logs_created ON logs USING BRIN(created_at);

-- 5. Partial Index - index เฉพาะบาง rows
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';
CREATE INDEX idx_pending_orders ON orders(user_id, created_at) WHERE status = 'pending';

-- 6. Expression Index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- 7. Composite Index (multi-column)
CREATE INDEX idx_orders_user_status ON orders(user_id, status, created_at DESC);
```

### ตรวจสอบ Index Usage

```sql
-- ดู index ที่ใช้และไม่ได้ใช้
SELECT 
  schemaname,
  tablename,
  indexname,
  idx_scan,    -- จำนวนครั้งที่ใช้ index
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- หา index ที่ไม่เคยใช้เลย
SELECT indexname, tablename
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexname NOT LIKE '%pkey';

-- ดู table scan statistics
SELECT 
  tablename,
  seq_scan,
  seq_tup_read,
  idx_scan,
  n_live_tup
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan
ORDER BY seq_tup_read DESC;
```

---

## 56.3 N+1 Problem

### DataLoader Pattern

```javascript
// src/utils/dataLoader.js
const DataLoader = require('dataloader');  // npm install dataloader

/**
 * DataLoader สำหรับ batch load users
 */
const createUserLoader = () => {
  return new DataLoader(async (userIds) => {
    // Load users ทั้งหมดด้วย query เดียว
    const users = await User.findAll({
      where: { id: { [Op.in]: userIds } }
    });
    
    // Map กลับตาม order ของ userIds
    const userMap = new Map(users.map(u => [u.id, u]));
    return userIds.map(id => userMap.get(id) || null);
  }, {
    cache: true,           // Cache ใน memory ระหว่าง request
    maxBatchSize: 100      // Load สูงสุด 100 รายการต่อ batch
  });
};

/**
 * Request-scoped DataLoaders
 */
const createLoaders = () => ({
  userLoader: createUserLoader(),
  productLoader: new DataLoader(async (productIds) => {
    const products = await Product.findAll({
      where: { id: { [Op.in]: productIds } }
    });
    const productMap = new Map(products.map(p => [p.id, p]));
    return productIds.map(id => productMap.get(id) || null);
  })
});

// Middleware ที่ inject loaders เข้า request
const dataLoaderMiddleware = (req, res, next) => {
  req.loaders = createLoaders();
  next();
};

module.exports = { createLoaders, dataLoaderMiddleware };
```

### ใช้งาน DataLoader ใน resolvers

```javascript
// src/controllers/orderController.js

const getOrderWithUserDetails = async (req, res) => {
  const { id } = req.params;
  const { loaders } = req;
  
  const order = await Order.findByPk(id, {
    include: [{ model: OrderItem }]
  });
  
  if (!order) return res.status(404).json({ error: 'Order not found' });
  
  // ใช้ DataLoader - batch requests อัตโนมัติ
  const user = await loaders.userLoader.load(order.userId);
  
  // Load products ทุก item พร้อมกัน (ไม่ใช่ทีละ item!)
  const products = await Promise.all(
    order.items.map(item => loaders.productLoader.load(item.productId))
  );
  
  res.json({
    ...order.toJSON(),
    user,
    items: order.items.map((item, index) => ({
      ...item.toJSON(),
      product: products[index]
    }))
  });
};
```

---

## 56.4 Connection Pooling

```javascript
// src/config/database.js
const { Sequelize } = require('sequelize');
const pg = require('pg');

// ตั้งค่า connection pool
const sequelize = new Sequelize(process.env.DATABASE_URL, {
  dialect: 'postgres',
  dialectModule: pg,
  
  pool: {
    max: parseInt(process.env.DB_POOL_MAX) || 20,
    min: parseInt(process.env.DB_POOL_MIN) || 5,
    acquire: 30000,   // timeout รอ connection (ms)
    idle: 10000,      // ปิด connection ที่ idle นาน (ms)
    evict: 60000      // ตรวจสอบ idle connections ทุก (ms)
  },
  
  // SSL สำหรับ production
  dialectOptions: process.env.NODE_ENV === 'production' ? {
    ssl: { require: true, rejectUnauthorized: false }
  } : {},
  
  // Query timeout
  dialectOptions: {
    statement_timeout: 30000,  // 30 seconds
    query_timeout: 30000
  }
});

// Monitor pool stats
const getPoolStats = () => {
  const pool = sequelize.connectionManager.pool;
  return {
    size: pool.size,
    available: pool.available,
    using: pool.using,
    waiting: pool.waiting
  };
};

module.exports = { sequelize, getPoolStats };
```

---

## 56.5 Read Replicas

```javascript
// src/config/databaseWithReplicas.js
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize(
  process.env.DB_NAME,
  process.env.DB_USER,
  process.env.DB_PASSWORD,
  {
    dialect: 'postgres',
    
    // Replication config
    replication: {
      read: [
        // Read replicas
        {
          host: process.env.DB_READ_HOST_1,
          port: process.env.DB_READ_PORT_1 || 5432
        },
        {
          host: process.env.DB_READ_HOST_2,
          port: process.env.DB_READ_PORT_2 || 5432
        }
      ],
      write: {
        host: process.env.DB_WRITE_HOST,
        port: process.env.DB_WRITE_PORT || 5432
      }
    },
    
    pool: {
      max: 10,
      min: 0,
      idle: 10000
    }
  }
);

// Sequelize จะ route automatically:
// - SELECT → read replicas (round-robin)
// - INSERT/UPDATE/DELETE → write host

module.exports = sequelize;
```

### Manual Read/Write Splitting

```javascript
// src/services/dbService.js
const writeDb = require('../config/writeDatabase');
const readDb = require('../config/readDatabase');

/**
 * ใช้ write DB สำหรับ mutations
 */
const createUser = async (userData) => {
  return await User.create(userData, { sequelize: writeDb });
};

/**
 * ใช้ read replica สำหรับ queries
 */
const getUsers = async (filters) => {
  return await User.findAll({ where: filters, sequelize: readDb });
};
```

---

## แบบฝึกหัดที่ 56

### แบบฝึกหัดพื้นฐาน

**1. Query Analysis**

วิเคราะห์ queries ใน application:
- ใช้ EXPLAIN ANALYZE
- หา slow queries
- เพิ่ม indexes ที่เหมาะสม

**2. Fix N+1 Problem**

แก้ไข N+1 query:
- ใช้ Sequelize `include`
- หรือใช้ DataLoader
- เปรียบเทียบ query count ก่อน/หลัง

### แบบฝึกหัดขั้นสูง

**3. Connection Pool Optimization**

ทดสอบ connection pool:
- Load test ด้วย Artillery
- Monitor pool stats
- หา optimal pool size

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Query Optimization** - EXPLAIN ANALYZE, avoid N+1
2. **Indexing** - B-tree, GIN, partial indexes
3. **N+1 Problem** - DataLoader, eager loading
4. **Connection Pooling** - pool config, monitoring
5. **Read Replicas** - read/write splitting

**ถัดไป:** Part 57 - API Gateway
