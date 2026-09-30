# Part 49 | ขั้นตอนที่ 841-860 จาก 1000

## Caching Strategies - กลยุทธ์การ Cache

การ caching เป็นเทคนิคสำคัญในการเพิ่มประสิทธิภาพของแอปพลิเคชัน โดยการเก็บข้อมูลที่ใช้บ่อยไว้ในที่ที่เข้าถึงได้เร็วกว่า

---

## ขั้นตอนที่ 841: ทำความเข้าใจ Caching

### ประเภทของ Cache:

**1. Client-side Cache** - เบราว์เซอร์เก็บ response
**2. CDN Cache** - Content Delivery Network เก็บ static assets
**3. Reverse Proxy Cache** - Nginx/Varnish เก็บ responses
**4. Application Cache** - Redis/Memcached เก็บข้อมูล
**5. Database Cache** - Query results cache

### Cache Terminology:
- **Cache Hit**: ข้อมูลอยู่ใน cache แล้ว (เร็ว)
- **Cache Miss**: ไม่มีข้อมูลใน cache (ต้องดึงจาก source)
- **Cache Invalidation**: การล้าง cache เมื่อข้อมูลเปลี่ยน
- **Cache Eviction**: การลบข้อมูลเก่าออกเมื่อ cache เต็ม
- **TTL (Time-To-Live)**: อายุของข้อมูลใน cache

---

## ขั้นตอนที่ 842: HTTP Caching Headers

```javascript
// middleware/cacheHeaders.js

// Cache-Control header values
const CACHE_POLICIES = {
  // ไม่ cache เลย (sensitive data)
  noCache: 'no-cache, no-store, must-revalidate',
  
  // Cache ฝั่ง browser เท่านั้น, 1 ชั่วโมง
  private: 'private, max-age=3600',
  
  // Cache ได้ทุกที่, 1 ชั่วโมง
  public: 'public, max-age=3600',
  
  // Cache ระยะยาวสำหรับ static assets
  immutable: 'public, max-age=31536000, immutable',
  
  // Stale-while-revalidate pattern
  staleWhileRevalidate: 'public, max-age=3600, stale-while-revalidate=86400'
};

// Middleware สำหรับตั้ง cache headers
const setCacheHeaders = (policy = 'noCache', options = {}) => {
  return (req, res, next) => {
    const cacheControl = CACHE_POLICIES[policy] || policy;
    
    res.set({
      'Cache-Control': cacheControl,
      'Pragma': policy === 'noCache' ? 'no-cache' : undefined,
      'Expires': policy === 'noCache' ? '0' : undefined
    });
    
    next();
  };
};

// ETag middleware
const etagMiddleware = (req, res, next) => {
  const originalJson = res.json.bind(res);
  
  res.json = function(data) {
    const crypto = require('crypto');
    const body = JSON.stringify(data);
    const etag = `"${crypto.createHash('md5').update(body).digest('hex')}"`;
    
    res.set('ETag', etag);
    
    // ตรวจสอบ If-None-Match header
    const clientEtag = req.headers['if-none-match'];
    if (clientEtag === etag) {
      return res.status(304).end();
    }
    
    return originalJson(data);
  };
  
  next();
};

module.exports = { setCacheHeaders, etagMiddleware, CACHE_POLICIES };
```

---

## ขั้นตอนที่ 843: Last-Modified Headers

```javascript
// middleware/lastModified.js

const setLastModified = (getModifiedDate) => {
  return async (req, res, next) => {
    if (typeof getModifiedDate === 'function') {
      const modifiedDate = await getModifiedDate(req);
      
      if (modifiedDate) {
        const lastModified = modifiedDate.toUTCString();
        res.set('Last-Modified', lastModified);
        
        // ตรวจสอบ If-Modified-Since
        const ifModifiedSince = req.headers['if-modified-since'];
        if (ifModifiedSince) {
          const clientDate = new Date(ifModifiedSince);
          if (modifiedDate <= clientDate) {
            return res.status(304).end();
          }
        }
      }
    }
    
    next();
  };
};

// ตัวอย่างการใช้งาน
const productRouter = express.Router();

productRouter.get('/:id', 
  setLastModified(async (req) => {
    const product = await Product.findById(req.params.id).select('updatedAt');
    return product ? product.updatedAt : null;
  }),
  async (req, res) => {
    const product = await Product.findById(req.params.id);
    res.json(product);
  }
);
```

---

## ขั้นตอนที่ 844: Redis Cache Service

```javascript
// services/cacheService.js
const redis = require('../config/redis');

class CacheService {
  constructor(defaultTTL = 3600) {
    this.defaultTTL = defaultTTL;
    this.prefix = 'cache:';
  }

  // สร้าง cache key
  makeKey(...parts) {
    return `${this.prefix}${parts.join(':')}`;
  }

  // ดึงข้อมูลจาก cache
  async get(key) {
    try {
      const data = await redis.get(this.makeKey(key));
      if (data) {
        return JSON.parse(data);
      }
      return null;
    } catch (error) {
      console.error('Cache get error:', error);
      return null;
    }
  }

  // บันทึกข้อมูลลง cache
  async set(key, value, ttl = this.defaultTTL) {
    try {
      const serialized = JSON.stringify(value);
      if (ttl > 0) {
        await redis.setex(this.makeKey(key), ttl, serialized);
      } else {
        await redis.set(this.makeKey(key), serialized);
      }
      return true;
    } catch (error) {
      console.error('Cache set error:', error);
      return false;
    }
  }

  // ลบ cache
  async del(key) {
    try {
      await redis.del(this.makeKey(key));
      return true;
    } catch (error) {
      console.error('Cache del error:', error);
      return false;
    }
  }

  // ลบ cache หลาย keys ด้วย pattern
  async delByPattern(pattern) {
    try {
      const keys = await redis.keys(this.makeKey(pattern));
      if (keys.length > 0) {
        await redis.del(...keys);
      }
      return keys.length;
    } catch (error) {
      console.error('Cache delByPattern error:', error);
      return 0;
    }
  }

  // Cache-aside pattern
  async remember(key, ttl, fetchFn) {
    const cached = await this.get(key);
    if (cached !== null) {
      return { data: cached, fromCache: true };
    }

    const data = await fetchFn();
    await this.set(key, data, ttl);
    return { data, fromCache: false };
  }

  // ตรวจสอบว่ามี key อยู่หรือไม่
  async exists(key) {
    return redis.exists(this.makeKey(key));
  }

  // ดู TTL ที่เหลือ
  async ttl(key) {
    return redis.ttl(this.makeKey(key));
  }

  // เพิ่มค่า (สำหรับ counter)
  async increment(key, by = 1) {
    return redis.incrby(this.makeKey(key), by);
  }
}

module.exports = new CacheService();
```

---

## ขั้นตอนที่ 845: Cache Middleware

```javascript
// middleware/cache.js
const cacheService = require('../services/cacheService');

// Middleware สำหรับ cache response
const cacheMiddleware = (options = {}) => {
  const {
    ttl = 3600,
    keyGenerator = (req) => `${req.method}:${req.originalUrl}`,
    condition = () => true,
    invalidateOn = []
  } = options;

  return async (req, res, next) => {
    // ตรวจสอบว่าควร cache หรือไม่
    if (req.method !== 'GET' || !condition(req)) {
      return next();
    }

    const cacheKey = keyGenerator(req);

    try {
      const cached = await cacheService.get(cacheKey);
      
      if (cached) {
        // ส่ง cached response
        res.set('X-Cache', 'HIT');
        res.set('Cache-Control', `max-age=${ttl}`);
        return res.json(cached);
      }

      // ไม่มี cache - intercept response
      res.set('X-Cache', 'MISS');
      
      const originalJson = res.json.bind(res);
      res.json = async function(data) {
        // บันทึก response ลง cache
        if (res.statusCode === 200) {
          await cacheService.set(cacheKey, data, ttl);
        }
        return originalJson(data);
      };

      next();
    } catch (error) {
      console.error('Cache middleware error:', error);
      next();
    }
  };
};

// Cache invalidation middleware
const invalidateCache = (patterns) => {
  return async (req, res, next) => {
    const originalJson = res.json.bind(res);
    
    res.json = async function(data) {
      if (res.statusCode >= 200 && res.statusCode < 300) {
        // ลบ cache เมื่อ request สำเร็จ
        for (const pattern of patterns) {
          const resolvedPattern = typeof pattern === 'function' 
            ? pattern(req, data) 
            : pattern;
          
          await cacheService.delByPattern(resolvedPattern);
        }
      }
      
      return originalJson(data);
    };
    
    next();
  };
};

module.exports = { cacheMiddleware, invalidateCache };
```

---

## ขั้นตอนที่ 846: Cache-Aside Pattern

```javascript
// routes/products.js
const express = require('express');
const router = express.Router();
const Product = require('../models/Product');
const cacheService = require('../services/cacheService');

// Cache-Aside: application จัดการ cache เอง
router.get('/', async (req, res) => {
  try {
    const { page = 1, limit = 10, category } = req.query;
    const cacheKey = `products:list:${page}:${limit}:${category || 'all'}`;
    
    // ลอง cache ก่อน
    const { data, fromCache } = await cacheService.remember(
      cacheKey,
      300, // 5 นาที
      async () => {
        // ถ้าไม่มีใน cache ดึงจาก DB
        const query = {};
        if (category) query.category = category;
        
        const [products, total] = await Promise.all([
          Product.find(query)
            .skip((page - 1) * limit)
            .limit(parseInt(limit))
            .lean(),
          Product.countDocuments(query)
        ]);
        
        return {
          products,
          pagination: {
            page: parseInt(page),
            limit: parseInt(limit),
            total,
            pages: Math.ceil(total / limit)
          }
        };
      }
    );

    res.set('X-Cache', fromCache ? 'HIT' : 'MISS');
    res.json({ success: true, ...data });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

router.get('/:id', async (req, res) => {
  try {
    const cacheKey = `products:${req.params.id}`;
    
    const { data, fromCache } = await cacheService.remember(
      cacheKey,
      600, // 10 นาที
      async () => {
        const product = await Product.findById(req.params.id).lean();
        if (!product) throw new Error('Product not found');
        return product;
      }
    );

    res.set('X-Cache', fromCache ? 'HIT' : 'MISS');
    res.json({ success: true, data });
  } catch (error) {
    if (error.message === 'Product not found') {
      return res.status(404).json({ success: false, error: 'Product not found' });
    }
    res.status(500).json({ success: false, error: error.message });
  }
});

// อัพเดทพร้อม invalidate cache
router.put('/:id', async (req, res) => {
  try {
    const product = await Product.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    );
    
    if (!product) {
      return res.status(404).json({ success: false, error: 'Product not found' });
    }
    
    // Invalidate cache
    await Promise.all([
      cacheService.del(`products:${req.params.id}`),
      cacheService.delByPattern('products:list:*')
    ]);
    
    res.json({ success: true, data: product });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

---

## ขั้นตอนที่ 847: Write-Through Cache Pattern

```javascript
// services/writeThroughCache.js

class WriteThroughCacheService {
  constructor(model, cacheService) {
    this.model = model;
    this.cache = cacheService;
    this.ttl = 3600;
  }

  // อ่านข้อมูล (ลอง cache ก่อน)
  async findById(id) {
    const cacheKey = `${this.model.modelName.toLowerCase()}:${id}`;
    const cached = await this.cache.get(cacheKey);
    
    if (cached) {
      return cached;
    }
    
    const data = await this.model.findById(id).lean();
    if (data) {
      await this.cache.set(cacheKey, data, this.ttl);
    }
    
    return data;
  }

  // สร้าง - บันทึกทั้ง DB และ cache พร้อมกัน
  async create(data) {
    const doc = await this.model.create(data);
    const cacheKey = `${this.model.modelName.toLowerCase()}:${doc._id}`;
    await this.cache.set(cacheKey, doc.toObject(), this.ttl);
    return doc;
  }

  // อัพเดท - อัพเดททั้ง DB และ cache พร้อมกัน
  async update(id, data) {
    const doc = await this.model.findByIdAndUpdate(id, data, { new: true }).lean();
    
    if (doc) {
      const cacheKey = `${this.model.modelName.toLowerCase()}:${id}`;
      await this.cache.set(cacheKey, doc, this.ttl);
    }
    
    return doc;
  }

  // ลบ - ลบทั้ง DB และ cache พร้อมกัน
  async delete(id) {
    const [doc] = await Promise.all([
      this.model.findByIdAndDelete(id),
      this.cache.del(`${this.model.modelName.toLowerCase()}:${id}`)
    ]);
    
    return doc;
  }
}

module.exports = WriteThroughCacheService;
```

---

## ขั้นตอนที่ 848: Cache Invalidation Strategies

```javascript
// services/cacheInvalidation.js
const cacheService = require('./cacheService');
const EventEmitter = require('events');

class CacheInvalidationService extends EventEmitter {
  constructor() {
    super();
    this.patterns = new Map();
    this.setupListeners();
  }

  // ลงทะเบียน invalidation patterns
  register(event, patterns) {
    this.patterns.set(event, patterns);
  }

  setupListeners() {
    this.on('invalidate', async (event, data) => {
      const patterns = this.patterns.get(event) || [];
      
      for (const pattern of patterns) {
        let resolvedPattern;
        
        if (typeof pattern === 'function') {
          resolvedPattern = pattern(data);
        } else if (typeof pattern === 'string') {
          // แทนที่ placeholders ด้วย data
          resolvedPattern = pattern.replace(/:(\w+)/g, (match, key) => {
            return data[key] || '*';
          });
        } else {
          resolvedPattern = pattern;
        }
        
        const deleted = await cacheService.delByPattern(resolvedPattern);
        console.log(`Invalidated ${deleted} cache keys for pattern: ${resolvedPattern}`);
      }
    });
  }

  // เรียกใช้เมื่อข้อมูลเปลี่ยน
  invalidate(event, data = {}) {
    this.emit('invalidate', event, data);
  }
}

const invalidator = new CacheInvalidationService();

// ลงทะเบียน patterns
invalidator.register('product:updated', [
  'products::id:*',          // product caches
  'products:list:*',        // list caches
  'categories:*:products:*' // category product caches
]);

invalidator.register('user:updated', [
  (data) => `users:${data.userId}:*`,
  'users:list:*'
]);

invalidator.register('order:created', [
  (data) => `users:${data.userId}:orders:*`,
  'orders:stats:*'
]);

module.exports = invalidator;
```

---

## ขั้นตอนที่ 849: CDN Integration

```javascript
// services/cdnService.js
const AWS = require('aws-sdk');
const path = require('path');
const crypto = require('crypto');

class CDNService {
  constructor() {
    this.s3 = new AWS.S3({
      accessKeyId: process.env.AWS_ACCESS_KEY_ID,
      secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
      region: process.env.AWS_REGION
    });
    
    this.cloudfront = new AWS.CloudFront();
    this.bucket = process.env.S3_BUCKET;
    this.cdnDomain = process.env.CDN_DOMAIN;
    this.distributionId = process.env.CLOUDFRONT_DISTRIBUTION_ID;
  }

  // อัพโหลดไฟล์ไปยัง S3/CDN
  async upload(file, options = {}) {
    const {
      folder = 'uploads',
      cacheControl = 'max-age=31536000',
      contentType
    } = options;
    
    const ext = path.extname(file.originalname);
    const hash = crypto.createHash('md5').update(file.buffer).digest('hex');
    const key = `${folder}/${hash}${ext}`;
    
    const params = {
      Bucket: this.bucket,
      Key: key,
      Body: file.buffer,
      ContentType: contentType || file.mimetype,
      CacheControl: cacheControl,
      ACL: 'public-read'
    };
    
    await this.s3.upload(params).promise();
    
    return {
      key,
      url: `https://${this.cdnDomain}/${key}`
    };
  }

  // Invalidate CDN cache
  async invalidate(paths) {
    const params = {
      DistributionId: this.distributionId,
      InvalidationBatch: {
        CallerReference: Date.now().toString(),
        Paths: {
          Quantity: paths.length,
          Items: paths.map(p => p.startsWith('/') ? p : `/${p}`)
        }
      }
    };
    
    return this.cloudfront.createInvalidation(params).promise();
  }

  // สร้าง signed URL สำหรับ private content
  async getSignedUrl(key, expiresIn = 3600) {
    const params = {
      Bucket: this.bucket,
      Key: key,
      Expires: expiresIn
    };
    
    return this.s3.getSignedUrlPromise('getObject', params);
  }
}

module.exports = new CDNService();
```

---

## ขั้นตอนที่ 850: Server-side Caching ด้วย Node-Cache

```javascript
// สำหรับ caching ใน memory (เหมาะกับ single instance)
const NodeCache = require('node-cache');

class MemoryCache {
  constructor(options = {}) {
    this.cache = new NodeCache({
      stdTTL: options.ttl || 600,
      checkperiod: options.checkperiod || 120,
      useClones: options.useClones !== false,
      maxKeys: options.maxKeys || 1000
    });
    
    // ติดตาม stats
    this.stats = {
      hits: 0,
      misses: 0,
      sets: 0,
      deletes: 0
    };
    
    this.cache.on('expired', (key, value) => {
      console.debug(`Cache expired: ${key}`);
    });
  }

  get(key) {
    const value = this.cache.get(key);
    if (value !== undefined) {
      this.stats.hits++;
      return value;
    }
    this.stats.misses++;
    return null;
  }

  set(key, value, ttl) {
    this.stats.sets++;
    return this.cache.set(key, value, ttl);
  }

  del(key) {
    this.stats.deletes++;
    return this.cache.del(key);
  }

  flush() {
    return this.cache.flushAll();
  }

  getStats() {
    const cacheStats = this.cache.getStats();
    return {
      ...this.stats,
      keys: cacheStats.keys,
      ksize: cacheStats.ksize,
      vsize: cacheStats.vsize,
      hitRate: this.stats.hits / (this.stats.hits + this.stats.misses) || 0
    };
  }
  
  // Memoize function
  memoize(fn, keyFn, ttl) {
    return async (...args) => {
      const key = keyFn ? keyFn(...args) : args.join(':');
      const cached = this.get(key);
      
      if (cached !== null) {
        return cached;
      }
      
      const result = await fn(...args);
      this.set(key, result, ttl);
      return result;
    };
  }
}

module.exports = new MemoryCache();
```

---

## ขั้นตอนที่ 851: Multi-Level Cache

```javascript
// services/multiLevelCache.js
const memoryCache = require('./memoryCache');
const redisCache = require('./cacheService');

class MultiLevelCache {
  constructor() {
    this.l1 = memoryCache;  // Level 1: In-memory (เร็วที่สุด)
    this.l2 = redisCache;   // Level 2: Redis (shared)
    
    this.l1TTL = 60;         // 1 นาที สำหรับ L1
    this.l2TTL = 3600;       // 1 ชั่วโมง สำหรับ L2
  }

  async get(key) {
    // ลอง L1 ก่อน
    const l1Value = this.l1.get(key);
    if (l1Value !== null) {
      return { value: l1Value, level: 'L1' };
    }
    
    // ลอง L2
    const l2Value = await this.l2.get(key);
    if (l2Value !== null) {
      // โปรโมทไปยัง L1
      this.l1.set(key, l2Value, this.l1TTL);
      return { value: l2Value, level: 'L2' };
    }
    
    return { value: null, level: 'MISS' };
  }

  async set(key, value, options = {}) {
    const {
      l1TTL = this.l1TTL,
      l2TTL = this.l2TTL,
      skipL1 = false
    } = options;
    
    const promises = [];
    
    if (!skipL1) {
      this.l1.set(key, value, l1TTL);
    }
    
    promises.push(this.l2.set(key, value, l2TTL));
    
    await Promise.all(promises);
  }

  async del(key) {
    this.l1.del(key);
    await this.l2.del(key);
  }

  async remember(key, fetchFn, options = {}) {
    const { value, level } = await this.get(key);
    
    if (value !== null) {
      return { data: value, level };
    }
    
    const data = await fetchFn();
    await this.set(key, data, options);
    
    return { data, level: 'FETCH' };
  }
}

module.exports = new MultiLevelCache();
```

---

## ขั้นตอนที่ 852: Stale-While-Revalidate Pattern

```javascript
// services/staleWhileRevalidate.js
const redis = require('../config/redis');

class StaleWhileRevalidateCache {
  constructor(options = {}) {
    this.freshTTL = options.freshTTL || 60;       // 1 นาที: ยังใหม่
    this.staleTTL = options.staleTTL || 3600;      // 1 ชั่วโมง: stale แต่ยังใช้ได้
    this.prefix = options.prefix || 'swr:';
    this.revalidating = new Set();
  }

  async get(key, fetchFn) {
    const dataKey = `${this.prefix}data:${key}`;
    const metaKey = `${this.prefix}meta:${key}`;
    
    const [data, meta] = await Promise.all([
      redis.get(dataKey),
      redis.get(metaKey)
    ]);
    
    if (data) {
      const parsedData = JSON.parse(data);
      const parsedMeta = meta ? JSON.parse(meta) : null;
      
      const isFresh = parsedMeta && 
        (Date.now() - parsedMeta.cachedAt) < this.freshTTL * 1000;
      
      if (!isFresh && !this.revalidating.has(key)) {
        // Revalidate ใน background
        this.revalidateInBackground(key, fetchFn);
      }
      
      return {
        data: parsedData,
        fresh: isFresh,
        cachedAt: parsedMeta ? new Date(parsedMeta.cachedAt) : null
      };
    }
    
    // ไม่มี cache - ดึงข้อมูลและบันทึก
    const freshData = await fetchFn();
    await this.set(key, freshData);
    
    return { data: freshData, fresh: true, cachedAt: new Date() };
  }

  async set(key, data) {
    const dataKey = `${this.prefix}data:${key}`;
    const metaKey = `${this.prefix}meta:${key}`;
    
    const meta = { cachedAt: Date.now() };
    
    await Promise.all([
      redis.setex(dataKey, this.staleTTL, JSON.stringify(data)),
      redis.setex(metaKey, this.staleTTL, JSON.stringify(meta))
    ]);
  }

  async revalidateInBackground(key, fetchFn) {
    this.revalidating.add(key);
    
    try {
      const freshData = await fetchFn();
      await this.set(key, freshData);
    } catch (error) {
      console.error(`Background revalidation failed for ${key}:`, error);
    } finally {
      this.revalidating.delete(key);
    }
  }
}

module.exports = StaleWhileRevalidateCache;
```

---

## ขั้นตอนที่ 853: Cache Warming

```javascript
// services/cacheWarmer.js
const cacheService = require('./cacheService');
const Product = require('../models/Product');
const Category = require('../models/Category');

class CacheWarmer {
  constructor() {
    this.warmingFunctions = [];
  }

  register(fn) {
    this.warmingFunctions.push(fn);
  }

  async warm() {
    console.log('Starting cache warming...');
    const startTime = Date.now();
    
    const results = await Promise.allSettled(
      this.warmingFunctions.map(fn => fn())
    );
    
    const succeeded = results.filter(r => r.status === 'fulfilled').length;
    const failed = results.filter(r => r.status === 'rejected').length;
    
    console.log(`Cache warming complete: ${succeeded} succeeded, ${failed} failed`);
    console.log(`Total time: ${Date.now() - startTime}ms`);
    
    return { succeeded, failed };
  }
}

const warmer = new CacheWarmer();

// ลงทะเบียน warming functions
warmer.register(async () => {
  // อุ่น product categories
  const categories = await Category.find({ active: true }).lean();
  await cacheService.set('categories:all', categories, 3600);
  console.log(`Warmed ${categories.length} categories`);
});

warmer.register(async () => {
  // อุ่น popular products
  const products = await Product.find({ featured: true })
    .sort({ soldCount: -1 })
    .limit(100)
    .lean();
  
  // Cache แต่ละ product
  await Promise.all(products.map(p => 
    cacheService.set(`products:${p._id}`, p, 3600)
  ));
  
  console.log(`Warmed ${products.length} featured products`);
});

warmer.register(async () => {
  // อุ่น first page ของ products
  const products = await Product.find()
    .sort({ createdAt: -1 })
    .limit(20)
    .lean();
  
  await cacheService.set('products:list:1:20:all', {
    products,
    pagination: { page: 1, limit: 20 }
  }, 300);
  
  console.log('Warmed products first page');
});

module.exports = warmer;
```

---

## ขั้นตอนที่ 854: Cache Analytics

```javascript
// services/cacheAnalytics.js
const redis = require('../config/redis');

class CacheAnalytics {
  constructor() {
    this.prefix = 'analytics:cache:';
  }

  // บันทึก cache hit/miss
  async record(key, hit) {
    const hour = new Date();
    hour.setMinutes(0, 0, 0);
    const hourKey = hour.toISOString();
    
    const pipeline = redis.pipeline();
    
    if (hit) {
      pipeline.incr(`${this.prefix}hits:${hourKey}`);
    } else {
      pipeline.incr(`${this.prefix}misses:${hourKey}`);
    }
    
    pipeline.expire(`${this.prefix}hits:${hourKey}`, 86400 * 7);
    pipeline.expire(`${this.prefix}misses:${hourKey}`, 86400 * 7);
    
    await pipeline.exec();
  }

  // คำนวณ hit rate
  async getHitRate(hours = 24) {
    const stats = [];
    const now = new Date();
    
    for (let i = 0; i < hours; i++) {
      const hour = new Date(now - i * 3600 * 1000);
      hour.setMinutes(0, 0, 0);
      const hourKey = hour.toISOString();
      
      const [hits, misses] = await Promise.all([
        redis.get(`${this.prefix}hits:${hourKey}`),
        redis.get(`${this.prefix}misses:${hourKey}`)
      ]);
      
      const h = parseInt(hits) || 0;
      const m = parseInt(misses) || 0;
      const total = h + m;
      
      stats.push({
        hour: hourKey,
        hits: h,
        misses: m,
        total,
        hitRate: total > 0 ? (h / total * 100).toFixed(2) : 0
      });
    }
    
    return stats.reverse();
  }

  // ดู key usage
  async getTopKeys(limit = 10) {
    const keys = await redis.keys(`${this.prefix}key:*`);
    const scores = await Promise.all(
      keys.map(async (k) => ({
        key: k.replace(`${this.prefix}key:`, ''),
        count: parseInt(await redis.get(k)) || 0
      }))
    );
    
    return scores.sort((a, b) => b.count - a.count).slice(0, limit);
  }
}

module.exports = new CacheAnalytics();
```

---

## ขั้นตอนที่ 855: Cache ใน Production

```javascript
// config/cache.js
const NodeCache = require('node-cache');
const redis = require('./redis');

// ตั้งค่า cache ตาม environment
const getCacheConfig = () => {
  const env = process.env.NODE_ENV;
  
  if (env === 'production') {
    return {
      driver: 'redis',
      ttl: {
        short: 60,       // 1 นาที
        medium: 3600,    // 1 ชั่วโมง
        long: 86400,     // 1 วัน
        permanent: 0     // ไม่หมดอายุ
      }
    };
  }
  
  if (env === 'test') {
    return {
      driver: 'memory',
      ttl: {
        short: 5,
        medium: 10,
        long: 30,
        permanent: 60
      }
    };
  }
  
  // Development
  return {
    driver: 'memory',
    ttl: {
      short: 30,
      medium: 300,
      long: 1800,
      permanent: 0
    }
  };
};

module.exports = getCacheConfig();
```

---

## ขั้นตอนที่ 856: ตัวอย่างการใช้งานจริง - Product Catalog

```javascript
// services/productCatalogService.js
const Product = require('../models/Product');
const multiLevelCache = require('./multiLevelCache');
const invalidator = require('./cacheInvalidation');

class ProductCatalogService {
  async getProducts(filters = {}) {
    const cacheKey = `catalog:products:${JSON.stringify(filters)}`;
    
    const { data } = await multiLevelCache.remember(
      cacheKey,
      () => this.fetchProducts(filters),
      { l1TTL: 60, l2TTL: 300 }
    );
    
    return data;
  }

  async fetchProducts(filters) {
    const { page = 1, limit = 20, category, search, sort = 'createdAt' } = filters;
    
    const query = { active: true };
    if (category) query.category = category;
    if (search) query.$text = { $search: search };
    
    const [products, total] = await Promise.all([
      Product.find(query)
        .sort({ [sort]: -1 })
        .skip((page - 1) * limit)
        .limit(limit)
        .populate('category', 'name slug')
        .lean(),
      Product.countDocuments(query)
    ]);
    
    return { products, total, page, limit };
  }

  async getProduct(id) {
    const cacheKey = `catalog:product:${id}`;
    
    const { data } = await multiLevelCache.remember(
      cacheKey,
      () => Product.findById(id)
        .populate('category')
        .populate('reviews')
        .lean()
    );
    
    return data;
  }

  async updateProduct(id, updates) {
    const product = await Product.findByIdAndUpdate(id, updates, { new: true });
    
    // Invalidate related caches
    invalidator.invalidate('product:updated', { id, category: product.category });
    
    return product;
  }
}

module.exports = new ProductCatalogService();
```

---

## ขั้นตอนที่ 857: Cache Testing

```javascript
// tests/cache.test.js
const cacheService = require('../services/cacheService');
const redis = require('../config/redis');

describe('Cache Service Tests', () => {
  afterEach(async () => {
    await redis.flushdb();
  });

  afterAll(async () => {
    await redis.quit();
  });

  describe('Basic Operations', () => {
    it('should set and get value', async () => {
      await cacheService.set('test:key', { data: 'test' }, 60);
      const result = await cacheService.get('test:key');
      expect(result).toEqual({ data: 'test' });
    });

    it('should return null for non-existent key', async () => {
      const result = await cacheService.get('test:nonexistent');
      expect(result).toBeNull();
    });

    it('should delete key', async () => {
      await cacheService.set('test:key', 'value', 60);
      await cacheService.del('test:key');
      const result = await cacheService.get('test:key');
      expect(result).toBeNull();
    });

    it('should expire after TTL', async () => {
      await cacheService.set('test:expire', 'value', 1);
      
      await new Promise(resolve => setTimeout(resolve, 1100));
      
      const result = await cacheService.get('test:expire');
      expect(result).toBeNull();
    });
  });

  describe('Remember Pattern', () => {
    it('should fetch and cache on miss', async () => {
      let fetchCount = 0;
      const fetchFn = async () => {
        fetchCount++;
        return { data: 'fetched' };
      };
      
      const result1 = await cacheService.remember('test:remember', 60, fetchFn);
      const result2 = await cacheService.remember('test:remember', 60, fetchFn);
      
      expect(result1.data).toEqual({ data: 'fetched' });
      expect(result2.data).toEqual({ data: 'fetched' });
      expect(fetchCount).toBe(1); // ดึงข้อมูลแค่ครั้งเดียว
    });
  });
});
```

---

## ขั้นตอนที่ 858: Cache Monitoring Dashboard

```javascript
// routes/admin/cache.js
const express = require('express');
const router = express.Router();
const redis = require('../../config/redis');
const cacheAnalytics = require('../../services/cacheAnalytics');
const memoryCache = require('../../services/memoryCache');

// ดู cache stats
router.get('/stats', async (req, res) => {
  try {
    const [redisInfo, hitRate, memStats] = await Promise.all([
      redis.info('memory'),
      cacheAnalytics.getHitRate(24),
      Promise.resolve(memoryCache.getStats())
    ]);

    const redisMemory = parseRedisInfo(redisInfo);

    res.json({
      success: true,
      data: {
        redis: {
          memory: redisMemory,
          connected: true
        },
        memory: memStats,
        hitRate: hitRate,
        summary: {
          avgHitRate: calculateAvgHitRate(hitRate),
          totalHits: hitRate.reduce((sum, h) => sum + h.hits, 0),
          totalMisses: hitRate.reduce((sum, h) => sum + h.misses, 0)
        }
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Flush cache
router.delete('/flush', async (req, res) => {
  const { pattern } = req.body;
  
  if (pattern) {
    const keys = await redis.keys(`cache:${pattern}`);
    if (keys.length > 0) {
      await redis.del(...keys);
    }
    res.json({ success: true, deleted: keys.length });
  } else {
    await redis.flushdb();
    memoryCache.flush();
    res.json({ success: true, message: 'All cache flushed' });
  }
});

function parseRedisInfo(info) {
  const lines = info.split('\n');
  const result = {};
  lines.forEach(line => {
    const [key, value] = line.split(':');
    if (key && value) {
      result[key.trim()] = value.trim();
    }
  });
  return result;
}

function calculateAvgHitRate(stats) {
  const validStats = stats.filter(s => s.total > 0);
  if (validStats.length === 0) return 0;
  const sum = validStats.reduce((acc, s) => acc + parseFloat(s.hitRate), 0);
  return (sum / validStats.length).toFixed(2);
}

module.exports = router;
```

---

## ขั้นตอนที่ 859: Cache Best Practices

```javascript
// utils/cacheUtils.js

// สร้าง cache key ที่สอดคล้องกัน
const createCacheKey = (...parts) => {
  return parts
    .map(p => {
      if (typeof p === 'object') return JSON.stringify(p);
      return String(p);
    })
    .join(':')
    .toLowerCase()
    .replace(/[^a-z0-9:_-]/g, '_');
};

// ตรวจสอบ TTL ที่เหมาะสม
const TTL = {
  REAL_TIME: 5,         // 5 วินาที: ข้อมูล real-time
  VERY_SHORT: 30,       // 30 วินาที: ข้อมูลที่เปลี่ยนบ่อย
  SHORT: 300,           // 5 นาที: API responses
  MEDIUM: 3600,         // 1 ชั่วโมง: Product info
  LONG: 86400,          // 1 วัน: Static data
  VERY_LONG: 604800,    // 1 อาทิตย์: Configuration
};

// Helper สำหรับ pagination cache
const getPaginationCacheKey = (base, { page, limit, sort, filter }) => {
  const filterStr = filter ? JSON.stringify(filter) : 'all';
  return `${base}:p${page}:l${limit}:s${sort}:${filterStr}`;
};

// Decorator สำหรับ cache
const cacheable = (keyFn, ttl = TTL.SHORT) => {
  return (target, propertyKey, descriptor) => {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(...args) {
      const cacheService = require('../services/cacheService');
      const key = keyFn(...args);
      
      const cached = await cacheService.get(key);
      if (cached !== null) {
        return cached;
      }
      
      const result = await originalMethod.apply(this, args);
      await cacheService.set(key, result, ttl);
      
      return result;
    };
    
    return descriptor;
  };
};

module.exports = { createCacheKey, TTL, getPaginationCacheKey, cacheable };
```

---

## ขั้นตอนที่ 860: Integration Example

```javascript
// ตัวอย่าง Full Integration ใน Express App
const express = require('express');
const app = express();
const { cacheMiddleware, invalidateCache } = require('./middleware/cache');
const { setCacheHeaders } = require('./middleware/cacheHeaders');
const cacheWarmer = require('./services/cacheWarmer');

// Warm cache เมื่อ server เริ่มต้น
app.on('ready', async () => {
  await cacheWarmer.warm();
});

// Static assets - cache นาน
app.use('/static', 
  setCacheHeaders('immutable'),
  express.static('public')
);

// API - cache ระยะสั้น
app.get('/api/products',
  setCacheHeaders('public'),
  cacheMiddleware({
    ttl: 300,
    keyGenerator: (req) => `products:${req.query.page}:${req.query.category}`
  }),
  async (req, res) => {
    const products = await Product.find().lean();
    res.json({ data: products });
  }
);

// Write operations - invalidate cache
app.post('/api/products',
  invalidateCache([
    'products:*',
    (req, data) => `categories:${data.category}:*`
  ]),
  async (req, res) => {
    const product = await Product.create(req.body);
    res.status(201).json({ data: product });
  }
);

// No cache for private data
app.get('/api/profile',
  setCacheHeaders('noCache'),
  async (req, res) => {
    const user = await User.findById(req.user.id);
    res.json({ data: user });
  }
);

module.exports = app;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Implement Cache Manager
สร้าง CacheManager class ที่รองรับทั้ง Memory, Redis, และ Multi-level cache พร้อม interface ที่สม่ำเสมอ

### แบบฝึกหัดที่ 2: Cache Invalidation Strategy
สร้างระบบ event-driven cache invalidation ที่ทำงานผ่าน Redis pub/sub เพื่อ invalidate cache ข้าม multiple server instances

### แบบฝึกหัดที่ 3: Cache Performance Monitor
สร้าง middleware ที่วัด cache performance รวมถึง hit rate, latency, และ memory usage แล้วส่ง metrics ไปยัง monitoring system

### แบบฝึกหัดที่ 4: HTTP Cache Headers Test
เขียน tests ที่ตรวจสอบว่า HTTP cache headers ถูกต้องสำหรับแต่ละประเภทของ resource

### แบบฝึกหัดที่ 5: CDN Cache Invalidation
Implement ระบบที่ invalidate CDN cache โดยอัตโนมัติเมื่อ content เปลี่ยนแปลง

---

## สรุป

การ caching ที่ดีต้องพิจารณาหลายระดับ ตั้งแต่ HTTP headers, CDN, reverse proxy, application cache จนถึง database query cache การเลือกกลยุทธ์ที่เหมาะสมและการจัดการ cache invalidation อย่างถูกต้องเป็นกุญแจสำคัญของระบบที่มีประสิทธิภาพ
