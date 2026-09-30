# Part 50 | ขั้นตอนที่ 861-880 จาก 1000

## Database Optimization - การเพิ่มประสิทธิภาพฐานข้อมูล

การ optimize database เป็นหนึ่งในงานที่สำคัญที่สุดในการพัฒนาแอปพลิเคชัน เพราะ database มักเป็น bottleneck หลักของระบบ

---

## ขั้นตอนที่ 861: ทำความเข้าใจ Database Performance

### ปัจจัยที่ส่งผลต่อ Database Performance:

1. **Query Design** - การเขียน queries ที่มีประสิทธิภาพ
2. **Indexing** - การสร้าง indexes ที่เหมาะสม
3. **Connection Management** - การจัดการ connections
4. **Schema Design** - การออกแบบโครงสร้างข้อมูล
5. **Hardware** - RAM, Disk I/O, CPU

### เครื่องมือ Profiling:
```javascript
// เปิด Mongoose debug mode
const mongoose = require('mongoose');
mongoose.set('debug', (collectionName, method, query, doc) => {
  console.log(`${collectionName}.${method}`, JSON.stringify(query), doc);
});

// หรือใช้ explain()
const result = await Product.find({ category: 'electronics' }).explain('executionStats');
console.log(JSON.stringify(result.executionStats, null, 2));
```

---

## ขั้นตอนที่ 862: MongoDB Indexing

```javascript
// models/Product.js
const mongoose = require('mongoose');

const productSchema = new mongoose.Schema({
  name: { type: String, required: true },
  slug: { type: String, required: true, unique: true },
  category: { type: mongoose.Schema.Types.ObjectId, ref: 'Category' },
  price: { type: Number, required: true },
  stock: { type: Number, default: 0 },
  tags: [String],
  isActive: { type: Boolean, default: true },
  createdAt: { type: Date, default: Date.now },
  soldCount: { type: Number, default: 0 },
  rating: { type: Number, default: 0 }
}, {
  timestamps: true
});

// Single field indexes
productSchema.index({ slug: 1 });        // Unique index (กำหนดใน field)
productSchema.index({ category: 1 });    // สำหรับ filter โดย category
productSchema.index({ price: 1 });       // สำหรับ sort และ range query
productSchema.index({ isActive: 1 });    // สำหรับ filter

// Compound indexes (สำหรับ queries ที่ใช้หลาย fields)
productSchema.index({ category: 1, price: 1 });
productSchema.index({ isActive: 1, createdAt: -1 });
productSchema.index({ category: 1, isActive: 1, price: 1 });

// Text index สำหรับ full-text search
productSchema.index({
  name: 'text',
  description: 'text',
  tags: 'text'
}, {
  weights: {
    name: 10,        // name มีความสำคัญมากที่สุด
    tags: 5,
    description: 1
  },
  name: 'product_text_index'
});

// TTL index สำหรับข้อมูลที่หมดอายุ
const sessionSchema = new mongoose.Schema({
  token: String,
  userId: mongoose.Schema.Types.ObjectId,
  expiresAt: Date
});

sessionSchema.index({ expiresAt: 1 }, { expireAfterSeconds: 0 });

module.exports = mongoose.model('Product', productSchema);
```

---

## ขั้นตอนที่ 863: Query Optimization

```javascript
// services/productService.js

// BAD: ไม่มี projection - ดึงทุก field
const badQuery = async () => {
  return Product.find({ isActive: true });
};

// GOOD: ใช้ projection เลือกเฉพาะ field ที่ต้องการ
const goodQuery = async () => {
  return Product.find(
    { isActive: true },
    { name: 1, price: 1, category: 1, slug: 1 }
  );
};

// BAD: ดึงข้อมูลทั้งหมดแล้วกรองใน code
const badFilter = async (minPrice) => {
  const products = await Product.find({ isActive: true });
  return products.filter(p => p.price >= minPrice);
};

// GOOD: กรองใน database
const goodFilter = async (minPrice) => {
  return Product.find({
    isActive: true,
    price: { $gte: minPrice }
  });
};

// BAD: ไม่ใช้ index สำหรับ sort
const badSort = async () => {
  return Product.find({ isActive: true })
    .sort({ name: 1 }); // ถ้าไม่มี index บน name จะช้า
};

// GOOD: ใช้ field ที่มี index สำหรับ sort
const goodSort = async () => {
  return Product.find({ isActive: true })
    .sort({ createdAt: -1 }); // มี index บน isActive+createdAt
};

// ใช้ lean() สำหรับ read-only queries
const leanQuery = async () => {
  return Product.find({ isActive: true })
    .select('name price category')
    .lean(); // คืน plain JavaScript objects แทน Mongoose documents
};
```

---

## ขั้นตอนที่ 864: แก้ไข N+1 Problem

```javascript
// N+1 Problem - BAD
const badOrderService = async () => {
  // Query 1: ดึง orders
  const orders = await Order.find({ status: 'completed' }).limit(10);
  
  // N queries: ดึง user สำหรับแต่ละ order (N+1 problem!)
  const ordersWithUsers = await Promise.all(
    orders.map(async order => {
      const user = await User.findById(order.userId); // N queries
      return { ...order.toObject(), user };
    })
  );
  
  return ordersWithUsers;
};

// GOOD: ใช้ populate
const goodOrderService = async () => {
  return Order.find({ status: 'completed' })
    .limit(10)
    .populate('userId', 'name email')
    .lean();
};

// GOOD: ใช้ aggregation pipeline
const aggregationService = async () => {
  return Order.aggregate([
    { $match: { status: 'completed' } },
    { $limit: 10 },
    {
      $lookup: {
        from: 'users',
        localField: 'userId',
        foreignField: '_id',
        as: 'user',
        pipeline: [
          { $project: { name: 1, email: 1 } }
        ]
      }
    },
    { $unwind: '$user' }
  ]);
};

// GOOD: Manual batching สำหรับกรณีที่ซับซ้อน
const batchLoadService = async () => {
  const orders = await Order.find({ status: 'completed' })
    .select('userId total')
    .limit(10)
    .lean();
  
  // ดึง unique user IDs
  const userIds = [...new Set(orders.map(o => o.userId.toString()))];
  
  // ดึง users ทั้งหมดใน 1 query
  const users = await User.find({ _id: { $in: userIds } })
    .select('name email')
    .lean();
  
  // สร้าง lookup map
  const userMap = users.reduce((acc, user) => {
    acc[user._id.toString()] = user;
    return acc;
  }, {});
  
  // Merge ข้อมูล
  return orders.map(order => ({
    ...order,
    user: userMap[order.userId.toString()]
  }));
};
```

---

## ขั้นตอนที่ 865: Connection Pooling

```javascript
// config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  const options = {
    // Connection Pool
    maxPoolSize: 10,        // สูงสุด 10 connections ใน pool
    minPoolSize: 2,         // ขั้นต่ำ 2 connections
    
    // Timeouts
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 45000,
    connectTimeoutMS: 10000,
    
    // Retry
    retryWrites: true,
    
    // Compression
    compressors: ['zlib']
  };

  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, options);
    console.log(`MongoDB connected: ${conn.connection.host}`);
    
    // Monitor connection events
    mongoose.connection.on('connected', () => console.log('MongoDB connected'));
    mongoose.connection.on('error', (err) => console.error('MongoDB error:', err));
    mongoose.connection.on('disconnected', () => console.log('MongoDB disconnected'));
    
    // Graceful shutdown
    process.on('SIGINT', async () => {
      await mongoose.connection.close();
      process.exit(0);
    });
  } catch (error) {
    console.error('Database connection failed:', error);
    process.exit(1);
  }
};

// ตรวจสอบ pool stats
const getPoolStats = () => {
  const pool = mongoose.connection;
  return {
    poolSize: pool.pool ? pool.pool.totalConnectionCount : 0,
    available: pool.pool ? pool.pool.availableConnectionCount : 0,
    pending: pool.pool ? pool.pool.pendingConnectionCount : 0
  };
};

module.exports = { connectDB, getPoolStats };
```

---

## ขั้นตอนที่ 866: Aggregation Pipeline Optimization

```javascript
// services/analyticsService.js
const Order = require('../models/Order');
const Product = require('../models/Product');

class AnalyticsService {
  // ยอดขายรายเดือน
  async getMonthlySales(year) {
    return Order.aggregate([
      {
        $match: {
          status: 'completed',
          createdAt: {
            $gte: new Date(`${year}-01-01`),
            $lt: new Date(`${year + 1}-01-01`)
          }
        }
      },
      {
        $group: {
          _id: { $month: '$createdAt' },
          totalSales: { $sum: '$total' },
          orderCount: { $sum: 1 },
          avgOrderValue: { $avg: '$total' }
        }
      },
      {
        $sort: { '_id': 1 }
      },
      {
        $project: {
          month: '$_id',
          totalSales: 1,
          orderCount: 1,
          avgOrderValue: { $round: ['$avgOrderValue', 2] },
          _id: 0
        }
      }
    ]);
  }

  // สินค้าขายดี
  async getTopProducts(limit = 10) {
    return Order.aggregate([
      { $match: { status: 'completed' } },
      { $unwind: '$items' },
      {
        $group: {
          _id: '$items.productId',
          totalSold: { $sum: '$items.quantity' },
          totalRevenue: { $sum: { $multiply: ['$items.price', '$items.quantity'] } }
        }
      },
      { $sort: { totalSold: -1 } },
      { $limit: limit },
      {
        $lookup: {
          from: 'products',
          localField: '_id',
          foreignField: '_id',
          as: 'product',
          pipeline: [{ $project: { name: 1, price: 1, category: 1 } }]
        }
      },
      { $unwind: '$product' }
    ]);
  }

  // Customer Lifetime Value
  async getCustomerLTV() {
    return Order.aggregate([
      { $match: { status: 'completed' } },
      {
        $group: {
          _id: '$userId',
          totalSpent: { $sum: '$total' },
          orderCount: { $sum: 1 },
          firstOrder: { $min: '$createdAt' },
          lastOrder: { $max: '$createdAt' }
        }
      },
      {
        $addFields: {
          avgOrderValue: { $divide: ['$totalSpent', '$orderCount'] },
          daysSinceFirstOrder: {
            $divide: [
              { $subtract: [new Date(), '$firstOrder'] },
              1000 * 60 * 60 * 24
            ]
          }
        }
      },
      {
        $lookup: {
          from: 'users',
          localField: '_id',
          foreignField: '_id',
          as: 'user',
          pipeline: [{ $project: { name: 1, email: 1 } }]
        }
      },
      { $unwind: '$user' },
      { $sort: { totalSpent: -1 } },
      { $limit: 100 }
    ]);
  }
}

module.exports = new AnalyticsService();
```

---

## ขั้นตอนที่ 867: Database Query Batching

```javascript
// services/dataLoader.js
// DataLoader pattern สำหรับ batch loading

class DataLoader {
  constructor(batchFn, options = {}) {
    this.batchFn = batchFn;
    this.maxBatchSize = options.maxBatchSize || 100;
    this.batchDelay = options.batchDelay || 10; // ms
    this.cache = options.cache !== false;
    
    this.queue = [];
    this.cacheMap = new Map();
    this.batchTimer = null;
  }

  async load(key) {
    if (this.cache && this.cacheMap.has(key)) {
      return this.cacheMap.get(key);
    }

    return new Promise((resolve, reject) => {
      this.queue.push({ key, resolve, reject });
      
      if (!this.batchTimer) {
        this.batchTimer = setTimeout(() => this.executeBatch(), this.batchDelay);
      }
    });
  }

  async loadMany(keys) {
    return Promise.all(keys.map(key => this.load(key)));
  }

  async executeBatch() {
    this.batchTimer = null;
    
    const batch = this.queue.splice(0, this.maxBatchSize);
    if (batch.length === 0) return;

    const keys = batch.map(item => item.key);
    
    try {
      const results = await this.batchFn(keys);
      
      batch.forEach(({ key, resolve, reject }) => {
        const result = results.find(r => 
          r._id.toString() === key.toString()
        );
        
        if (result) {
          if (this.cache) {
            this.cacheMap.set(key, result);
          }
          resolve(result);
        } else {
          reject(new Error(`Not found: ${key}`));
        }
      });
    } catch (error) {
      batch.forEach(({ reject }) => reject(error));
    }

    // ถ้ายังมี queue ให้ทำต่อ
    if (this.queue.length > 0) {
      this.batchTimer = setTimeout(() => this.executeBatch(), this.batchDelay);
    }
  }

  clear(key) {
    if (key) {
      this.cacheMap.delete(key);
    } else {
      this.cacheMap.clear();
    }
  }
}

// สร้าง loaders สำหรับ models ต่างๆ
const User = require('../models/User');
const Product = require('../models/Product');

const userLoader = new DataLoader(async (ids) => {
  return User.find({ _id: { $in: ids } }).lean();
});

const productLoader = new DataLoader(async (ids) => {
  return Product.find({ _id: { $in: ids } }).lean();
});

module.exports = { DataLoader, userLoader, productLoader };
```

---

## ขั้นตอนที่ 868: Pagination Optimization

```javascript
// utils/pagination.js

// Cursor-based pagination (เร็วกว่า offset-based สำหรับ large datasets)
const cursorPagination = async (Model, options = {}) => {
  const {
    cursor,        // cursor จาก previous page
    limit = 20,
    query = {},
    sort = { _id: 1 },
    select = ''
  } = options;

  const queryObj = { ...query };
  
  if (cursor) {
    const sortField = Object.keys(sort)[0];
    const sortDir = sort[sortField];
    
    if (sortDir === 1) {
      queryObj[sortField] = { $gt: cursor };
    } else {
      queryObj[sortField] = { $lt: cursor };
    }
  }

  const results = await Model.find(queryObj)
    .select(select)
    .sort(sort)
    .limit(limit + 1) // ดึงมากกว่า 1 เพื่อตรวจสอบว่ามีหน้าถัดไปไหม
    .lean();

  const hasNextPage = results.length > limit;
  const items = hasNextPage ? results.slice(0, limit) : results;
  
  const sortField = Object.keys(sort)[0];
  const nextCursor = hasNextPage ? items[items.length - 1][sortField] : null;

  return {
    items,
    pagination: {
      cursor: nextCursor,
      hasNextPage,
      limit
    }
  };
};

// Offset-based pagination ปกติ (สำหรับ small datasets)
const offsetPagination = async (Model, options = {}) => {
  const {
    page = 1,
    limit = 20,
    query = {},
    sort = { createdAt: -1 },
    select = '',
    populate = null
  } = options;

  const skip = (page - 1) * limit;

  let queryBuilder = Model.find(query)
    .select(select)
    .sort(sort)
    .skip(skip)
    .limit(limit)
    .lean();

  if (populate) {
    queryBuilder = queryBuilder.populate(populate);
  }

  const [items, total] = await Promise.all([
    queryBuilder,
    Model.countDocuments(query)
  ]);

  return {
    items,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit),
      hasNextPage: page < Math.ceil(total / limit),
      hasPrevPage: page > 1
    }
  };
};

module.exports = { cursorPagination, offsetPagination };
```

---

## ขั้นตอนที่ 869: Read Replicas

```javascript
// config/database.js - Read Replica setup
const mongoose = require('mongoose');

// Primary connection (Read/Write)
const primaryConnection = mongoose.createConnection(
  process.env.MONGODB_PRIMARY_URI,
  {
    maxPoolSize: 10,
    serverSelectionTimeoutMS: 5000
  }
);

// Secondary connections (Read only)
const replicaConnections = (process.env.MONGODB_REPLICA_URIS || '')
  .split(',')
  .filter(Boolean)
  .map(uri => mongoose.createConnection(uri, {
    maxPoolSize: 5,
    readPreference: 'secondary'
  }));

let replicaIndex = 0;

// Round-robin load balancing สำหรับ replicas
const getReadConnection = () => {
  if (replicaConnections.length === 0) {
    return primaryConnection; // fallback ถ้าไม่มี replica
  }
  
  const conn = replicaConnections[replicaIndex];
  replicaIndex = (replicaIndex + 1) % replicaConnections.length;
  return conn;
};

// สร้าง models สำหรับ read/write
const createModels = (schema, modelName) => {
  const WriteModel = primaryConnection.model(modelName, schema);
  
  const ReadModel = new Proxy({}, {
    get: (target, prop) => {
      const conn = getReadConnection();
      const model = conn.model(modelName, schema);
      
      if (typeof model[prop] === 'function') {
        return model[prop].bind(model);
      }
      return model[prop];
    }
  });
  
  return { WriteModel, ReadModel };
};

module.exports = {
  primaryConnection,
  getReadConnection,
  createModels
};
```

---

## ขั้นตอนที่ 870: Database Query Profiling

```javascript
// middleware/queryProfiler.js
const mongoose = require('mongoose');

class QueryProfiler {
  constructor() {
    this.slowQueryThreshold = 100; // ms
    this.queries = [];
    this.maxQueries = 1000;
    
    this.setupHooks();
  }

  setupHooks() {
    mongoose.plugin((schema) => {
      ['find', 'findOne', 'findOneAndUpdate', 'updateOne', 'updateMany',
       'deleteOne', 'deleteMany', 'count', 'countDocuments'].forEach(method => {
        
        schema.pre(method, function() {
          this._startTime = Date.now();
          this._method = method;
        });

        schema.post(method, function(result) {
          const duration = Date.now() - this._startTime;
          const query = this.getFilter ? this.getFilter() : {};
          
          const profileData = {
            method: this._method,
            collection: this.model.collection.name,
            query: JSON.stringify(query),
            duration,
            timestamp: new Date(),
            slow: duration > this.slowQueryThreshold
          };
          
          if (profileData.slow) {
            console.warn('Slow query detected:', profileData);
          }
        });
      });
    });
  }

  // Analyze slow queries
  getSlowQueries(threshold = this.slowQueryThreshold) {
    return this.queries
      .filter(q => q.duration >= threshold)
      .sort((a, b) => b.duration - a.duration);
  }

  // สรุปสถิติ
  getSummary() {
    if (this.queries.length === 0) return null;
    
    const durations = this.queries.map(q => q.duration);
    const sum = durations.reduce((a, b) => a + b, 0);
    
    return {
      total: this.queries.length,
      avgDuration: sum / this.queries.length,
      maxDuration: Math.max(...durations),
      minDuration: Math.min(...durations),
      slowQueries: this.queries.filter(q => q.slow).length
    };
  }
}

module.exports = new QueryProfiler();
```

---

## ขั้นตอนที่ 871: PostgreSQL Optimization ด้วย Sequelize

```javascript
// models/sequelize/Product.js
const { DataTypes } = require('sequelize');
const sequelize = require('../config/sequelize');

const Product = sequelize.define('Product', {
  id: {
    type: DataTypes.UUID,
    defaultValue: DataTypes.UUIDV4,
    primaryKey: true
  },
  name: {
    type: DataTypes.STRING(255),
    allowNull: false
  },
  slug: {
    type: DataTypes.STRING(255),
    allowNull: false,
    unique: true
  },
  price: {
    type: DataTypes.DECIMAL(10, 2),
    allowNull: false
  },
  categoryId: {
    type: DataTypes.UUID,
    allowNull: false
  },
  isActive: {
    type: DataTypes.BOOLEAN,
    defaultValue: true
  }
}, {
  tableName: 'products',
  indexes: [
    { fields: ['slug'], unique: true },
    { fields: ['categoryId'] },
    { fields: ['price'] },
    { fields: ['isActive', 'createdAt'] },
    { fields: ['categoryId', 'isActive', 'price'] },
    {
      // Full-text search index (PostgreSQL)
      name: 'products_fts_index',
      fields: [
        sequelize.literal("to_tsvector('english', name || ' ' || COALESCE(description, ''))")
      ],
      using: 'GIN'
    }
  ]
});

// Eager loading ที่มีประสิทธิภาพ
const getProductsWithCategory = async () => {
  return Product.findAll({
    include: [{
      model: Category,
      as: 'category',
      attributes: ['id', 'name', 'slug']
    }],
    where: { isActive: true },
    attributes: ['id', 'name', 'price', 'slug'],
    order: [['createdAt', 'DESC']],
    limit: 20
  });
};

// Raw queries สำหรับ complex queries
const getComplexReport = async () => {
  const [results] = await sequelize.query(`
    SELECT 
      p.id,
      p.name,
      p.price,
      COUNT(oi.id) as order_count,
      SUM(oi.quantity) as total_sold
    FROM products p
    LEFT JOIN order_items oi ON oi.product_id = p.id
    LEFT JOIN orders o ON o.id = oi.order_id AND o.status = 'completed'
    WHERE p.is_active = true
    GROUP BY p.id, p.name, p.price
    ORDER BY total_sold DESC NULLS LAST
    LIMIT 10
  `, {
    type: sequelize.QueryTypes.SELECT
  });
  
  return results;
};

module.exports = Product;
```

---

## ขั้นตอนที่ 872: Database Migration ที่มีประสิทธิภาพ

```javascript
// migrations/add-product-indexes.js
'use strict';

module.exports = {
  async up(queryInterface, Sequelize) {
    // เพิ่ม indexes แบบ concurrent (ไม่ lock table)
    // PostgreSQL specific
    await queryInterface.sequelize.query(
      'CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_products_category_active ON products(category_id, is_active)'
    );
    
    await queryInterface.sequelize.query(
      'CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_products_price ON products(price)'
    );
    
    // Partial index - index เฉพาะ active products
    await queryInterface.sequelize.query(
      'CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_active_products ON products(created_at DESC) WHERE is_active = true'
    );
  },

  async down(queryInterface, Sequelize) {
    await queryInterface.sequelize.query('DROP INDEX IF EXISTS idx_products_category_active');
    await queryInterface.sequelize.query('DROP INDEX IF EXISTS idx_products_price');
    await queryInterface.sequelize.query('DROP INDEX IF EXISTS idx_active_products');
  }
};
```

---

## ขั้นตอนที่ 873: Query Caching Layer

```javascript
// services/queryCacheService.js
const cacheService = require('./cacheService');
const crypto = require('crypto');

class QueryCacheService {
  constructor(model, options = {}) {
    this.model = model;
    this.ttl = options.ttl || 300;
    this.prefix = options.prefix || model.modelName.toLowerCase();
  }

  // สร้าง cache key จาก query parameters
  createKey(method, query, options = {}) {
    const data = JSON.stringify({ method, query, options });
    const hash = crypto.createHash('md5').update(data).digest('hex');
    return `${this.prefix}:query:${hash}`;
  }

  // Cached find
  async find(query = {}, options = {}) {
    const key = this.createKey('find', query, options);
    
    return cacheService.remember(key, this.ttl, async () => {
      let builder = this.model.find(query);
      
      if (options.select) builder = builder.select(options.select);
      if (options.sort) builder = builder.sort(options.sort);
      if (options.skip) builder = builder.skip(options.skip);
      if (options.limit) builder = builder.limit(options.limit);
      if (options.populate) builder = builder.populate(options.populate);
      if (options.lean !== false) builder = builder.lean();
      
      return builder;
    });
  }

  // Cached findOne
  async findOne(query = {}, options = {}) {
    const key = this.createKey('findOne', query, options);
    
    return cacheService.remember(key, this.ttl, async () => {
      let builder = this.model.findOne(query);
      
      if (options.select) builder = builder.select(options.select);
      if (options.populate) builder = builder.populate(options.populate);
      if (options.lean !== false) builder = builder.lean();
      
      return builder;
    });
  }

  // Cached findById
  async findById(id, options = {}) {
    const key = `${this.prefix}:${id}`;
    
    return cacheService.remember(key, this.ttl, async () => {
      let builder = this.model.findById(id);
      
      if (options.select) builder = builder.select(options.select);
      if (options.populate) builder = builder.populate(options.populate);
      if (options.lean !== false) builder = builder.lean();
      
      return builder;
    });
  }

  // Cached count
  async count(query = {}) {
    const key = this.createKey('count', query);
    
    return cacheService.remember(key, this.ttl, () => {
      return this.model.countDocuments(query);
    });
  }

  // Invalidate all caches for this model
  async invalidateAll() {
    return cacheService.delByPattern(`${this.prefix}:*`);
  }

  // Invalidate specific document
  async invalidateById(id) {
    return cacheService.del(`${this.prefix}:${id}`);
  }
}

module.exports = QueryCacheService;
```

---

## ขั้นตอนที่ 874: Database Health Monitoring

```javascript
// services/dbHealthService.js
const mongoose = require('mongoose');

class DatabaseHealthService {
  async check() {
    const checks = await Promise.allSettled([
      this.checkConnection(),
      this.checkResponseTime(),
      this.checkPoolStatus()
    ]);

    const results = {
      connection: checks[0].status === 'fulfilled' ? checks[0].value : { status: 'error', error: checks[0].reason?.message },
      responseTime: checks[1].status === 'fulfilled' ? checks[1].value : { status: 'error', error: checks[1].reason?.message },
      pool: checks[2].status === 'fulfilled' ? checks[2].value : { status: 'error', error: checks[2].reason?.message }
    };

    const overallHealth = Object.values(results).every(r => r.status === 'ok') 
      ? 'healthy' 
      : 'degraded';

    return { status: overallHealth, checks: results };
  }

  async checkConnection() {
    const state = mongoose.connection.readyState;
    const states = {
      0: 'disconnected',
      1: 'connected',
      2: 'connecting',
      3: 'disconnecting'
    };
    
    return {
      status: state === 1 ? 'ok' : 'error',
      state: states[state]
    };
  }

  async checkResponseTime() {
    const start = Date.now();
    await mongoose.connection.db.admin().ping();
    const duration = Date.now() - start;
    
    return {
      status: duration < 100 ? 'ok' : 'slow',
      duration,
      threshold: 100
    };
  }

  async checkPoolStatus() {
    const pool = mongoose.connection;
    
    return {
      status: 'ok',
      details: {
        readyState: pool.readyState
      }
    };
  }

  // ดึง database stats
  async getStats() {
    const db = mongoose.connection.db;
    const stats = await db.stats();
    
    return {
      collections: stats.collections,
      objects: stats.objects,
      dataSize: `${(stats.dataSize / 1024 / 1024).toFixed(2)} MB`,
      storageSize: `${(stats.storageSize / 1024 / 1024).toFixed(2)} MB`,
      indexes: stats.indexes,
      indexSize: `${(stats.indexSize / 1024 / 1024).toFixed(2)} MB`
    };
  }
}

module.exports = new DatabaseHealthService();
```

---

## ขั้นตอนที่ 875: Bulk Operations

```javascript
// services/bulkOperations.js
const Product = require('../models/Product');
const mongoose = require('mongoose');

class BulkOperationsService {
  // Bulk insert
  async bulkInsert(documents, options = {}) {
    const { batchSize = 1000, ordered = false } = options;
    const results = [];
    
    for (let i = 0; i < documents.length; i += batchSize) {
      const batch = documents.slice(i, i + batchSize);
      
      try {
        const result = await Product.insertMany(batch, {
          ordered, // ordered=false ข้ามเอกสารที่ error ต่อไปได้
          lean: true
        });
        results.push(...result);
      } catch (error) {
        console.error(`Batch ${i / batchSize} failed:`, error.message);
      }
    }
    
    return results;
  }

  // Bulk update
  async bulkUpdate(updates) {
    const bulkOps = updates.map(({ filter, update, options = {} }) => ({
      updateOne: { filter, update, ...options }
    }));
    
    return Product.bulkWrite(bulkOps, { ordered: false });
  }

  // Bulk upsert
  async bulkUpsert(documents, matchField = 'slug') {
    const bulkOps = documents.map(doc => ({
      replaceOne: {
        filter: { [matchField]: doc[matchField] },
        replacement: doc,
        upsert: true
      }
    }));
    
    return Product.bulkWrite(bulkOps);
  }

  // Bulk delete
  async bulkDelete(ids) {
    return Product.deleteMany({ _id: { $in: ids } });
  }

  // เพิ่ม price ทุกสินค้าใน category
  async updatePricesByCategory(categoryId, percentageChange) {
    const multiplier = 1 + (percentageChange / 100);
    
    return Product.updateMany(
      { category: categoryId },
      [{
        $set: {
          price: { $multiply: ['$price', multiplier] },
          updatedAt: new Date()
        }
      }]
    );
  }
}

module.exports = new BulkOperationsService();
```

---

## ขั้นตอนที่ 876: Transactions

```javascript
// services/orderService.js
const mongoose = require('mongoose');
const Order = require('../models/Order');
const Product = require('../models/Product');
const User = require('../models/User');

class OrderService {
  async createOrder(userId, items) {
    // เริ่ม MongoDB transaction
    const session = await mongoose.startSession();
    session.startTransaction();

    try {
      // 1. ตรวจสอบ stock
      for (const item of items) {
        const product = await Product.findById(item.productId).session(session);
        
        if (!product) {
          throw new Error(`Product not found: ${item.productId}`);
        }
        
        if (product.stock < item.quantity) {
          throw new Error(`Insufficient stock for: ${product.name}`);
        }
      }

      // 2. คำนวณราคา
      const productIds = items.map(i => i.productId);
      const products = await Product.find({ _id: { $in: productIds } })
        .session(session);
      
      const productMap = products.reduce((acc, p) => {
        acc[p._id.toString()] = p;
        return acc;
      }, {});

      const orderItems = items.map(item => {
        const product = productMap[item.productId.toString()];
        return {
          productId: item.productId,
          quantity: item.quantity,
          price: product.price,
          subtotal: product.price * item.quantity
        };
      });

      const total = orderItems.reduce((sum, item) => sum + item.subtotal, 0);

      // 3. สร้าง Order
      const [order] = await Order.create([{
        userId,
        items: orderItems,
        total,
        status: 'pending'
      }], { session });

      // 4. ลด stock
      for (const item of items) {
        await Product.findByIdAndUpdate(
          item.productId,
          { 
            $inc: { stock: -item.quantity, soldCount: item.quantity }
          },
          { session }
        );
      }

      // 5. อัพเดท user stats
      await User.findByIdAndUpdate(
        userId,
        {
          $inc: { totalOrders: 1, totalSpent: total }
        },
        { session }
      );

      // Commit transaction
      await session.commitTransaction();
      
      return order;
    } catch (error) {
      // Rollback ถ้าเกิด error
      await session.abortTransaction();
      throw error;
    } finally {
      session.endSession();
    }
  }
}

module.exports = new OrderService();
```

---

## ขั้นตอนที่ 877: Sharding Strategy

```javascript
// config/sharding.js
// การกระจายข้อมูลใน MongoDB

/*
  Sharding Key Selection Criteria:
  1. High Cardinality - ค่าหลากหลาย
  2. Low Frequency - ไม่มี hotspot
  3. Non-monotonic - ไม่เพิ่มขึ้นเรื่อยๆ
*/

// ตัวอย่าง shard key สำหรับ User collection
const userShardKey = { userId: 'hashed' }; // Hash sharding

// ตัวอย่าง shard key สำหรับ Order collection
const orderShardKey = { userId: 1, orderId: 1 }; // Compound key

// Zone sharding - กระจายตาม geography
const zoneConfig = {
  'zone-us': {
    minKey: { region: 'US-EAST' },
    maxKey: { region: 'US-WEST-ZZZ' }
  },
  'zone-eu': {
    minKey: { region: 'EU-NORTH' },
    maxKey: { region: 'EU-SOUTH-ZZZ' }
  },
  'zone-asia': {
    minKey: { region: 'ASIA-EAST' },
    maxKey: { region: 'ASIA-WEST-ZZZ' }
  }
};

// Application-level sharding
class ApplicationSharding {
  constructor(shards) {
    this.shards = shards; // Array ของ database connections
  }

  // เลือก shard จาก user ID
  getShard(userId) {
    const hash = this.hashUserId(userId);
    const shardIndex = hash % this.shards.length;
    return this.shards[shardIndex];
  }

  hashUserId(userId) {
    const str = userId.toString();
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash;
    }
    return Math.abs(hash);
  }

  async findUser(userId) {
    const shard = this.getShard(userId);
    return shard.model('User').findById(userId);
  }
}

module.exports = ApplicationSharding;
```

---

## ขั้นตอนที่ 878: Database Seeding และ Testing

```javascript
// scripts/seed.js
const mongoose = require('mongoose');
const { faker } = require('@faker-js/faker');
const Product = require('../models/Product');
const Category = require('../models/Category');

const seed = async () => {
  console.log('Seeding database...');

  // สร้าง categories
  const categories = await Category.insertMany([
    { name: 'Electronics', slug: 'electronics' },
    { name: 'Clothing', slug: 'clothing' },
    { name: 'Books', slug: 'books' }
  ]);

  console.log(`Created ${categories.length} categories`);

  // สร้าง products แบบ batch
  const batchSize = 100;
  const totalProducts = 1000;
  let created = 0;

  for (let i = 0; i < totalProducts; i += batchSize) {
    const batch = Array.from({ length: Math.min(batchSize, totalProducts - i) }, () => {
      const category = categories[Math.floor(Math.random() * categories.length)];
      const name = faker.commerce.productName();
      
      return {
        name,
        slug: `${name.toLowerCase().replace(/\s+/g, '-')}-${faker.string.alphanumeric(6)}`,
        description: faker.commerce.productDescription(),
        price: parseFloat(faker.commerce.price()),
        category: category._id,
        stock: faker.number.int({ min: 0, max: 100 }),
        isActive: faker.datatype.boolean(0.9),
        tags: faker.helpers.arrayElements(['new', 'sale', 'featured', 'hot'], { min: 0, max: 3 })
      };
    });

    await Product.insertMany(batch, { ordered: false });
    created += batch.length;
    console.log(`Created ${created}/${totalProducts} products`);
  }

  console.log('Seeding complete!');
};

// รัน seed
mongoose.connect(process.env.MONGODB_URI)
  .then(() => seed())
  .then(() => process.exit(0))
  .catch(err => {
    console.error(err);
    process.exit(1);
  });
```

---

## ขั้นตอนที่ 879: Explain Query Analysis

```javascript
// utils/queryAnalyzer.js
const mongoose = require('mongoose');

const analyzeQuery = async (model, query, options = {}) => {
  const explain = await model
    .find(query, null, options)
    .explain('executionStats');
  
  const stats = explain.executionStats;
  const stage = explain.queryPlanner.winningPlan.stage;
  
  const analysis = {
    executionTime: stats.executionTimeMillis,
    documentsExamined: stats.totalDocsExamined,
    documentsReturned: stats.nReturned,
    indexUsed: stage !== 'COLLSCAN',
    scanType: stage,
    efficiency: stats.nReturned / (stats.totalDocsExamined || 1),
    recommendation: null
  };
  
  // ให้คำแนะนำ
  if (stage === 'COLLSCAN') {
    analysis.recommendation = 'COLLSCAN detected! Add an index to improve performance';
  } else if (analysis.efficiency < 0.1) {
    analysis.recommendation = 'Low efficiency. Consider a more specific index';
  } else if (stats.executionTimeMillis > 100) {
    analysis.recommendation = 'Slow query. Review index coverage and query structure';
  } else {
    analysis.recommendation = 'Query is performing well';
  }
  
  return analysis;
};

// ใช้งาน
const runAnalysis = async () => {
  const Product = require('../models/Product');
  
  const analyses = await Promise.all([
    analyzeQuery(Product, { isActive: true, category: 'some-id' }),
    analyzeQuery(Product, { price: { $gte: 100, $lte: 500 } }),
    analyzeQuery(Product, { $text: { $search: 'laptop' } })
  ]);
  
  analyses.forEach((analysis, index) => {
    console.log(`\nQuery ${index + 1}:`);
    console.log(JSON.stringify(analysis, null, 2));
  });
};

module.exports = { analyzeQuery };
```

---

## ขั้นตอนที่ 880: Production Database Configuration

```javascript
// config/productionDB.js

const getMongooseOptions = () => ({
  // Connection Pool
  maxPoolSize: parseInt(process.env.DB_MAX_POOL_SIZE) || 10,
  minPoolSize: parseInt(process.env.DB_MIN_POOL_SIZE) || 2,
  
  // Timeouts
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
  connectTimeoutMS: 10000,
  heartbeatFrequencyMS: 10000,
  
  // Retry
  retryWrites: true,
  retryReads: true,
  
  // Write Concern
  w: 'majority',
  journal: true,
  
  // Read Preference
  readPreference: 'primaryPreferred',
  
  // Compression
  compressors: ['zlib'],
  
  // Auto index (ปิดใน production เพื่อ performance)
  autoIndex: process.env.NODE_ENV !== 'production',
  autoCreate: process.env.NODE_ENV !== 'production'
});

// Connection string สำหรับ MongoDB Atlas
const getConnectionString = () => {
  const {
    MONGODB_USER,
    MONGODB_PASSWORD,
    MONGODB_CLUSTER,
    MONGODB_DATABASE
  } = process.env;
  
  return `mongodb+srv://${MONGODB_USER}:${MONGODB_PASSWORD}@${MONGODB_CLUSTER}/${MONGODB_DATABASE}?retryWrites=true&w=majority`;
};

module.exports = { getMongooseOptions, getConnectionString };
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Index Analysis
ใช้ `explain()` วิเคราะห์ queries ที่มีอยู่ใน project และสร้าง indexes เพื่อเพิ่มประสิทธิภาพ

### แบบฝึกหัดที่ 2: Fix N+1 Problem
ระบุและแก้ไข N+1 queries ในระบบ โดยใช้ populate, aggregation หรือ dataloader

### แบบฝึกหัดที่ 3: Implement Cursor Pagination
แทนที่ offset pagination ด้วย cursor-based pagination สำหรับ large datasets

### แบบฝึกหัดที่ 4: Connection Pool Monitoring
สร้าง dashboard แสดงสถานะ connection pool และ slow queries

### แบบฝึกหัดที่ 5: Bulk Import Performance Test
เปรียบเทียบ performance ระหว่างการ insert แบบ one-by-one กับ bulk insert สำหรับข้อมูล 10,000 รายการ

---

## สรุป

Database optimization เป็นกระบวนการต่อเนื่องที่ต้องวิเคราะห์ performance อยู่เสมอ การสร้าง indexes ที่เหมาะสม การแก้ไข N+1 problems การใช้ connection pooling และการ optimize queries เป็นพื้นฐานที่สำคัญ
