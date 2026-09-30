# Part 39: Caching ด้วย Redis

> ขั้นตอนที่ 39-39 จาก 1000

---

## สารบัญ

1. [Cache Concepts](#cache-concepts)
2. [Redis Setup](#redis-setup)
3. [ioredis](#ioredis)
4. [Cache Strategies](#cache-strategies)
5. [Cache Invalidation](#cache-invalidation)
6. [Session Storage](#session-storage)
7. [Rate Limiting ด้วย Redis](#rate-limiting-ดวย-redis)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Cache Concepts

Cache คือการเก็บข้อมูลชั่วคราวเพื่อให้ access ได้เร็วขึ้น

```
Without Cache:
Client → API → Database → API → Client
  (ทุก request ต้องคุยกับ DB - ช้า)

With Cache:
Client → API → Cache (HIT) → API → Client
  (อ่านจาก cache - เร็วมาก)

Client → API → Cache (MISS) → Database → Cache → API → Client
  (cache miss ครั้งแรก แต่ครั้งต่อไป HIT)
```

### เมื่อไหรควรใช้ Cache

```
✅ ข้อมูลที่:
- อ่านบ่อยมาก แต่เปลี่ยนไม่บ่อย
- Compute ช้า (complex queries, expensive calculations)
- External API responses
- Static data (categories, configs)

❌ ไม่ควร cache:
- ข้อมูล sensitive (passwords, payment info)
- ข้อมูลที่ต้องการ real-time accuracy
- ข้อมูลที่เปลี่ยนบ่อยมาก
- User-specific data (unless ทำ properly)
```

### Cache Metrics

```javascript
// Cache hit rate = hits / (hits + misses) * 100
// ถ้า hit rate < 80% → strategy อาจต้องปรับ

// Cache miss penalty = time to get data from source
// ยิ่ง miss penalty สูง → caching ยิ่งคุ้มค่า

// Memory usage = total cached data size
// ต้องสมดุลระหว่าง hit rate และ memory
```

---

## Redis Setup

### ติดตั้ง Redis

```bash
# Mac
brew install redis
brew services start redis

# Ubuntu/Debian
sudo apt update
sudo apt install redis-server
sudo systemctl start redis

# Docker (ง่ายที่สุด)
docker run -d \
  --name redis \
  -p 6379:6379 \
  -v redis-data:/data \
  redis:7-alpine \
  redis-server --appendonly yes --requirepass yourpassword

# ทดสอบ
redis-cli ping   # PONG
redis-cli -a yourpassword ping
```

### redis.conf ที่แนะนำ

```conf
# /etc/redis/redis.conf

# การ bind
bind 127.0.0.1
protected-mode yes

# Port
port 6379

# Password
requirepass your-strong-password

# Memory
maxmemory 256mb
maxmemory-policy allkeys-lru    # LRU eviction policy

# Persistence
appendonly yes
appendfilename "appendonly.aof"
save 900 1       # บันทึกทุก 15 นาที ถ้ามีการเปลี่ยน 1 key
save 300 10      # บันทึกทุก 5 นาที ถ้ามีการเปลี่ยน 10 keys
save 60 10000    # บันทึกทุก 1 นาที ถ้ามีการเปลี่ยน 10000 keys

# Logging
loglevel notice
logfile /var/log/redis/redis-server.log
```

---

## ioredis

ioredis เป็น Redis client สำหรับ Node.js ที่ดีที่สุด

### ติดตั้ง

```bash
npm install ioredis
```

### Connection Setup

```javascript
// config/redis.js
const Redis = require('ioredis');
const config = require('./index');

let redis;

function createRedisClient() {
  if (redis) return redis;
  
  redis = new Redis({
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT) || 6379,
    password: process.env.REDIS_PASSWORD,
    db: parseInt(process.env.REDIS_DB) || 0,
    
    // Connection settings
    maxRetriesPerRequest: 3,
    retryStrategy(times) {
      const delay = Math.min(times * 50, 2000);
      return delay;
    },
    
    // Reconnect on error
    reconnectOnError(err) {
      const targetError = 'READONLY';
      if (err.message.includes(targetError)) {
        return true; // ลอง reconnect ใหม่
      }
      return false;
    },
    
    // Lazy connect
    lazyConnect: false,
    
    // TLS สำหรับ production
    ...(process.env.REDIS_TLS === 'true' && { tls: {} }),
  });
  
  redis.on('connect', () => {
    console.log('✅ Redis connected');
  });
  
  redis.on('error', (error) => {
    console.error('❌ Redis error:', error.message);
  });
  
  redis.on('close', () => {
    console.log('Redis connection closed');
  });
  
  redis.on('reconnecting', () => {
    console.log('Redis reconnecting...');
  });
  
  return redis;
}

// Redis Cluster (สำหรับ production scale)
function createRedisCluster() {
  return new Redis.Cluster([
    { host: 'redis-node-1', port: 6379 },
    { host: 'redis-node-2', port: 6379 },
    { host: 'redis-node-3', port: 6379 },
  ], {
    redisOptions: {
      password: process.env.REDIS_PASSWORD,
    },
    scaleReads: 'slave', // อ่านจาก slave
  });
}

module.exports = { createRedisClient, redis: createRedisClient() };
```

### Basic Operations

```javascript
const { redis } = require('./config/redis');

// String operations
await redis.set('key', 'value');
await redis.set('key', 'value', 'EX', 3600);  // expire 1 hour
await redis.set('key', 'value', 'PX', 60000); // expire 60 seconds
await redis.set('key', 'value', 'NX');         // set only if not exists

const value = await redis.get('key');  // 'value' หรือ null

await redis.del('key');
await redis.del('key1', 'key2', 'key3');

// TTL operations
const ttl = await redis.ttl('key');   // seconds, -1 = no expiry, -2 = not exists
await redis.expire('key', 3600);
await redis.expireat('key', Math.floor(Date.now() / 1000) + 3600);
await redis.persist('key');           // ลบ expiration

// Numeric operations
await redis.set('counter', 0);
await redis.incr('counter');          // atomically increment by 1
await redis.incrby('counter', 5);
await redis.decr('counter');
await redis.decrby('counter', 3);

// Hash operations
await redis.hset('user:123', 'name', 'John', 'age', '30');
await redis.hset('user:123', { name: 'John', age: 30 }); // object form
const name = await redis.hget('user:123', 'name');
const user = await redis.hgetall('user:123'); // { name: 'John', age: '30' }
await redis.hdel('user:123', 'age');
const exists = await redis.hexists('user:123', 'name');

// List operations
await redis.rpush('list', 'item1', 'item2', 'item3'); // append right
await redis.lpush('list', 'item0');                    // prepend left
const items = await redis.lrange('list', 0, -1);       // ดูทั้งหมด
await redis.llen('list');
const item = await redis.lpop('list');  // ดึงจาก left
await redis.rpop('list');               // ดึงจาก right

// Set operations
await redis.sadd('tags', 'nodejs', 'express', 'javascript');
const tags = await redis.smembers('tags');
const isMember = await redis.sismember('tags', 'nodejs'); // 1 = yes
await redis.srem('tags', 'javascript');
await redis.scard('tags'); // จำนวน members

// Sorted Set
await redis.zadd('leaderboard', 100, 'player1', 200, 'player2');
await redis.zincrby('leaderboard', 50, 'player1'); // เพิ่ม score
const topPlayers = await redis.zrevrange('leaderboard', 0, 9, 'WITHSCORES');
const rank = await redis.zrevrank('leaderboard', 'player1');

// Pipeline (batch commands - เร็วกว่า)
const pipeline = redis.pipeline();
pipeline.set('key1', 'value1');
pipeline.set('key2', 'value2');
pipeline.get('key1');
const results = await pipeline.exec();
// results = [[null, 'OK'], [null, 'OK'], [null, 'value1']]
```

---

## Cache Strategies

### 1. Cache-Aside (Lazy Loading)

```javascript
// services/cacheService.js
const { redis } = require('../config/redis');

class CacheService {
  constructor(prefix = '', defaultTTL = 3600) {
    this.prefix = prefix;
    this.defaultTTL = defaultTTL;
  }
  
  key(k) {
    return `${this.prefix}:${k}`;
  }
  
  async get(key) {
    const data = await redis.get(this.key(key));
    if (!data) return null;
    
    try {
      return JSON.parse(data);
    } catch {
      return data;
    }
  }
  
  async set(key, value, ttl = this.defaultTTL) {
    const serialized = typeof value === 'string' ? value : JSON.stringify(value);
    
    if (ttl) {
      await redis.set(this.key(key), serialized, 'EX', ttl);
    } else {
      await redis.set(this.key(key), serialized);
    }
  }
  
  async del(key) {
    await redis.del(this.key(key));
  }
  
  async delPattern(pattern) {
    const keys = await redis.keys(`${this.prefix}:${pattern}`);
    if (keys.length) await redis.del(...keys);
  }
  
  // Cache-Aside pattern
  async getOrSet(key, fetchFn, ttl = this.defaultTTL) {
    const cached = await this.get(key);
    
    if (cached !== null) {
      return cached; // Cache HIT
    }
    
    // Cache MISS - ดึงจาก source
    const data = await fetchFn();
    
    if (data !== null && data !== undefined) {
      await this.set(key, data, ttl);
    }
    
    return data;
  }
}

module.exports = CacheService;
```

```javascript
// ใช้งาน Cache-Aside
const CacheService = require('../services/cacheService');
const userCache = new CacheService('users', 3600);

class UserService {
  async getUserById(id) {
    return userCache.getOrSet(
      id,
      async () => {
        const user = await User.findById(id).lean();
        return user;
      },
      1800 // 30 minutes
    );
  }
  
  async updateUser(id, data) {
    const user = await User.findByIdAndUpdate(id, data, { new: true });
    
    // Invalidate cache
    await userCache.del(id);
    
    return user;
  }
}
```

### 2. Write-Through Cache

```javascript
class WriteThoughCache {
  async updateUser(id, data) {
    // Update ทั้ง DB และ cache พร้อมกัน
    const [user] = await Promise.all([
      User.findByIdAndUpdate(id, data, { new: true }),
      userCache.set(id, { ...currentUser, ...data }),
    ]);
    
    return user;
  }
  
  async createUser(data) {
    const user = await User.create(data);
    
    // เขียน cache ทันที
    await userCache.set(user._id.toString(), user.toObject());
    
    return user;
  }
}
```

### 3. Read-Through Cache

```javascript
// Cache layer ที่อยู่หน้า DB
class ReadThroughCache {
  constructor(model, cache) {
    this.model = model;
    this.cache = cache;
  }
  
  async findById(id) {
    // ลองอ่านจาก cache
    let data = await this.cache.get(id);
    
    if (!data) {
      // ถ้า miss, อ่านจาก DB แล้วเขียน cache
      data = await this.model.findById(id).lean();
      
      if (data) {
        await this.cache.set(id, data);
      }
    }
    
    return data;
  }
}
```

### 4. Cache Decorator

```javascript
// Decorator สำหรับ cache function results
function cacheable(keyFn, ttl = 3600) {
  return function(target, propertyName, descriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(...args) {
      const cacheKey = keyFn(...args);
      
      const cached = await redis.get(cacheKey);
      if (cached) return JSON.parse(cached);
      
      const result = await originalMethod.apply(this, args);
      
      if (result !== null) {
        await redis.set(cacheKey, JSON.stringify(result), 'EX', ttl);
      }
      
      return result;
    };
    
    return descriptor;
  };
}

// ใช้งาน (TypeScript decorators)
class ProductService {
  @cacheable((id) => `product:${id}`, 3600)
  async getProduct(id) {
    return Product.findById(id);
  }
  
  @cacheable((category) => `products:category:${category}`, 1800)
  async getProductsByCategory(category) {
    return Product.find({ category });
  }
}
```

---

## Cache Invalidation

Cache invalidation เป็นปัญหาที่ยากที่สุดใน computer science

### Strategies

```javascript
// 1. TTL-based (ง่ายที่สุด)
await redis.set('data', value, 'EX', 3600); // expire หลัง 1 ชั่วโมง

// 2. Event-based (แม่นยำกว่า)
async function updateProduct(id, data) {
  await Product.findByIdAndUpdate(id, data);
  
  // Invalidate related caches
  await redis.del(`product:${id}`);
  await redis.del(`products:featured`);
  
  // หรือใช้ pattern
  const keys = await redis.keys(`product:*:${id}`);
  if (keys.length) await redis.del(...keys);
}

// 3. Cache Tags
class TaggedCache {
  async set(key, value, tags = [], ttl = 3600) {
    // เก็บ key ไว้ใน tag sets
    const pipeline = redis.pipeline();
    
    pipeline.set(key, JSON.stringify(value), 'EX', ttl);
    tags.forEach(tag => {
      pipeline.sadd(`tag:${tag}`, key);
      pipeline.expire(`tag:${tag}`, ttl);
    });
    
    await pipeline.exec();
  }
  
  async invalidateByTag(tag) {
    const keys = await redis.smembers(`tag:${tag}`);
    
    if (keys.length) {
      const pipeline = redis.pipeline();
      keys.forEach(key => pipeline.del(key));
      pipeline.del(`tag:${tag}`);
      await pipeline.exec();
    }
  }
}

// ใช้งาน
const cache = new TaggedCache();

await cache.set(
  'posts:page:1',
  posts,
  ['posts', 'page'],  // tags
  3600
);

// เมื่อ post เปลี่ยน, invalidate ทุก cache ที่มี tag 'posts'
await cache.invalidateByTag('posts');
```

---

## Session Storage

```javascript
// ใช้ connect-redis สำหรับ Express sessions
npm install express-session connect-redis
```

```javascript
// config/session.js
const session = require('express-session');
const { createClient } = require('redis');
const RedisStore = require('connect-redis').default;

const redisClient = createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379',
  password: process.env.REDIS_PASSWORD,
});

redisClient.connect().catch(console.error);

const sessionStore = new RedisStore({
  client: redisClient,
  prefix: 'sess:',          // prefix สำหรับ session keys
  ttl: 86400,               // 24 hours
});

const sessionConfig = {
  store: sessionStore,
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
    sameSite: 'strict',
  },
  name: 'sessionId', // ชื่อ cookie
};

module.exports = sessionConfig;
```

```javascript
// app.js
const session = require('express-session');
const sessionConfig = require('./config/session');

app.use(session(sessionConfig));

// ใช้งาน session
app.post('/login', async (req, res) => {
  const user = await authenticateUser(req.body);
  
  req.session.userId = user.id;
  req.session.role = user.role;
  req.session.save(); // force save
  
  res.json({ success: true });
});

app.get('/profile', (req, res) => {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Not logged in' });
  }
  
  res.json({ userId: req.session.userId });
});

app.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) return res.status(500).json({ error: 'Logout failed' });
    res.clearCookie('sessionId');
    res.json({ success: true });
  });
});
```

---

## Rate Limiting ด้วย Redis

```javascript
// middleware/rateLimiter.js
const { redis } = require('../config/redis');

class RateLimiter {
  constructor({ max, window, keyFn }) {
    this.max = max;
    this.window = window; // seconds
    this.keyFn = keyFn || ((req) => req.ip);
  }
  
  async isAllowed(req) {
    const key = `rate:${this.keyFn(req)}`;
    const now = Date.now();
    const windowStart = now - (this.window * 1000);
    
    // Sliding window algorithm ด้วย Sorted Set
    const pipeline = redis.pipeline();
    
    // ลบ entries ที่ expired
    pipeline.zremrangebyscore(key, '-inf', windowStart);
    
    // นับ current requests
    pipeline.zcard(key);
    
    // เพิ่ม current request
    pipeline.zadd(key, now, `${now}`);
    
    // ตั้ง expiration
    pipeline.expire(key, this.window);
    
    const results = await pipeline.exec();
    const count = results[1][1]; // จำนวน requests ก่อนเพิ่มอันใหม่
    
    return {
      allowed: count < this.max,
      remaining: Math.max(0, this.max - count - 1),
      total: this.max,
      resetAt: now + (this.window * 1000),
    };
  }
  
  middleware() {
    return async (req, res, next) => {
      const { allowed, remaining, total, resetAt } = await this.isAllowed(req);
      
      // เพิ่ม headers
      res.set({
        'X-RateLimit-Limit': total,
        'X-RateLimit-Remaining': remaining,
        'X-RateLimit-Reset': Math.ceil(resetAt / 1000),
      });
      
      if (!allowed) {
        return res.status(429).json({
          status: 'error',
          code: 'RATE_LIMIT_EXCEEDED',
          message: 'Too many requests',
          retryAfter: Math.ceil((resetAt - Date.now()) / 1000),
        });
      }
      
      next();
    };
  }
}

// ใช้งาน
const apiLimiter = new RateLimiter({
  max: 100,
  window: 60, // 100 requests per minute
});

const loginLimiter = new RateLimiter({
  max: 5,
  window: 300, // 5 login attempts per 5 minutes
  keyFn: (req) => `${req.ip}:${req.body.email}`, // per IP + email
});

app.use('/api/', apiLimiter.middleware());
app.use('/auth/login', loginLimiter.middleware());
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Cache Service

สร้าง CacheService ที่ครบสมบูรณ์:
1. get/set/del operations
2. getOrSet (cache-aside pattern)
3. TTL support
4. Pattern-based deletion
5. Cache statistics (hits, misses, hit rate)

### แบบฝึกหัดที่ 2: API Response Caching

Cache API responses:
1. Cache GET /api/products (TTL: 5 minutes)
2. Cache GET /api/products/:id (TTL: 1 hour)
3. Invalidate cache เมื่อ product ถูก update/delete
4. สร้าง cache middleware ที่ใช้ซ้ำได้

### แบบฝึกหัดที่ 3: Leaderboard

สร้าง game leaderboard ด้วย Redis Sorted Set:
1. เพิ่ม/อัพเดต score
2. ดู top 10 players
3. ดู rank ของ player
4. ดู players ใกล้เคียง rank ตัวเอง

---

## สรุป

Redis เป็น tool สำคัญสำหรับ high-performance applications

| Feature | Redis Command |
|---------|--------------|
| Simple cache | SET/GET with EX |
| Object cache | HASH or SET+JSON |
| Session | SET with EX |
| Rate limiting | ZADD/ZCARD |
| Queue | RPUSH/LPOP |
| Pub/Sub | PUBLISH/SUBSCRIBE |
| Leaderboard | ZADD/ZRANGE |

**ถัดไป**: [Part 40: Rate Limiting →](./part-40-rate-limiting.md)
