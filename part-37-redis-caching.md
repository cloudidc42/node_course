# Part 37 | ขั้นตอนที่ 641-660 จาก 1000

## Redis Caching - การใช้ Redis สำหรับ Caching และ Data Storage

---

## สารบัญ

1. [Redis คืออะไร](#redis-คืออะไร)
2. [การติดตั้ง Redis](#การติดตั้ง-redis)
3. [Redis Data Structures](#redis-data-structures)
4. [Caching Strategies](#caching-strategies)
5. [Session Management](#session-management)
6. [Redis Pub/Sub](#redis-pubsub)
7. [Rate Limiting ด้วย Redis](#rate-limiting-ด้วย-redis)
8. [Redis Cluster](#redis-cluster)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Redis คืออะไร

### ขั้นตอนที่ 641: ทำความรู้จัก Redis

Redis (Remote Dictionary Server) เป็น in-memory data structure store ที่ใช้เป็น database, cache, message broker, และ streaming engine

```
Redis ใช้สำหรับ:
  ✅ Caching - เก็บข้อมูลชั่วคราวเพื่อความเร็ว
  ✅ Sessions - เก็บ user sessions
  ✅ Rate Limiting - จำกัดจำนวน requests
  ✅ Pub/Sub - messaging
  ✅ Queues - job queues
  ✅ Leaderboards - ranking ด้วย Sorted Sets
  ✅ Real-time Analytics - counters, statistics
```

### ข้อดีของ Redis

```
1. ความเร็ว: ข้อมูลอยู่ใน memory → อ่าน/เขียน < 1ms
2. Data Structures: strings, hashes, lists, sets, sorted sets
3. Persistence: สามารถ save ลง disk ได้
4. Replication: master-slave replication
5. Clustering: horizontal scaling
6. Atomic Operations: thread-safe
```

---

## การติดตั้ง Redis

### ขั้นตอนที่ 642: ติดตั้ง Redis Server

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install redis-server

# macOS
brew install redis

# Docker (แนะนำสำหรับ development)
docker run -d --name redis -p 6379:6379 redis:alpine

# ตรวจสอบว่า Redis ทำงาน
redis-cli ping
# ตอบ: PONG
```

```bash
# เริ่ม Redis
sudo systemctl start redis

# ให้ Redis เริ่มอัตโนมัติ
sudo systemctl enable redis

# ทดสอบ Redis CLI
redis-cli
127.0.0.1:6379> SET hello "world"
OK
127.0.0.1:6379> GET hello
"world"
127.0.0.1:6379> DEL hello
(integer) 1
127.0.0.1:6379> EXIT
```

### ขั้นตอนที่ 643: ติดตั้ง Node.js Redis Client

```bash
npm install ioredis
# หรือ
npm install redis  # Official client
```

```javascript
// src/config/redis.js
const Redis = require('ioredis');

// Simple connection
const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT) || 6379,
  password: process.env.REDIS_PASSWORD,
  db: parseInt(process.env.REDIS_DB) || 0,

  // Retry strategy
  retryStrategy: (times) => {
    if (times > 10) {
      throw new Error('Redis connection failed after 10 retries');
    }
    return Math.min(times * 100, 3000);
  },

  // Connection options
  connectTimeout: 10000,
  lazyConnect: false,
  keepAlive: 30000,

  // Enable offline queue (เก็บ commands ไว้ระหว่าง reconnect)
  enableOfflineQueue: true,
});

// Events
redis.on('connect', () => console.log('✅ Redis connected'));
redis.on('ready', () => console.log('✅ Redis ready'));
redis.on('error', (err) => console.error('❌ Redis error:', err));
redis.on('close', () => console.log('Redis connection closed'));
redis.on('reconnecting', () => console.log('Redis reconnecting...'));

module.exports = redis;
```

---

## Redis Data Structures

### ขั้นตอนที่ 644: Strings

```javascript
// src/examples/strings.js
const redis = require('../config/redis');

async function stringExamples() {
  // SET และ GET
  await redis.set('name', 'สมชาย');
  const name = await redis.get('name');
  console.log(name); // สมชาย

  // SET พร้อม TTL (Time To Live)
  await redis.set('session', 'abc123', 'EX', 3600); // หมดอายุใน 1 ชั่วโมง
  await redis.setex('cache', 300, JSON.stringify({ data: 'value' })); // 5 นาที

  // ตรวจสอบ TTL
  const ttl = await redis.ttl('session');
  console.log(`Session หมดอายุใน ${ttl} วินาที`);

  // ลบ key
  await redis.del('name');

  // ตรวจสอบว่า key มีอยู่
  const exists = await redis.exists('session');
  console.log(exists); // 1 = มีอยู่, 0 = ไม่มี

  // Increment/Decrement
  await redis.set('counter', 0);
  await redis.incr('counter');     // 1
  await redis.incrby('counter', 5); // 6
  await redis.decr('counter');     // 5

  // Multiple SET/GET
  await redis.mset('key1', 'value1', 'key2', 'value2');
  const values = await redis.mget('key1', 'key2');
  console.log(values); // ['value1', 'value2']

  // SETNX - SET if Not eXists
  const set = await redis.setnx('unique', 'value');
  console.log(set); // 1 = set สำเร็จ, 0 = มีอยู่แล้ว
}
```

### ขั้นตอนที่ 645: Hashes

```javascript
// Hashes - เหมาะสำหรับ objects
async function hashExamples() {
  const userId = 'user:1';

  // HSET - set fields
  await redis.hset(userId, {
    name: 'สมชาย',
    email: 'somchai@example.com',
    age: '25',
    role: 'user',
  });

  // HGET - get one field
  const name = await redis.hget(userId, 'name');
  console.log(name); // สมชาย

  // HMGET - get multiple fields
  const [email, role] = await redis.hmget(userId, 'email', 'role');

  // HGETALL - get all fields
  const user = await redis.hgetall(userId);
  console.log(user); // { name: 'สมชาย', email: '...', ... }

  // HDEL - delete field
  await redis.hdel(userId, 'age');

  // HEXISTS - check field exists
  const exists = await redis.hexists(userId, 'name');

  // HINCRBY - increment numeric field
  await redis.hset(userId, 'loginCount', 0);
  await redis.hincrby(userId, 'loginCount', 1);

  // HKEYS, HVALS, HLEN
  const keys = await redis.hkeys(userId);
  const vals = await redis.hvals(userId);
  const len = await redis.hlen(userId);

  // ตั้ง TTL สำหรับ hash
  await redis.expire(userId, 3600);
}
```

### ขั้นตอนที่ 646: Lists

```javascript
// Lists - เหมาะสำหรับ queues, stacks
async function listExamples() {
  const key = 'mylist';

  // LPUSH/RPUSH - เพิ่มทางซ้าย/ขวา
  await redis.rpush(key, 'item1', 'item2', 'item3');
  await redis.lpush(key, 'item0');
  // list: ['item0', 'item1', 'item2', 'item3']

  // LRANGE - ดึง elements ตาม index
  const all = await redis.lrange(key, 0, -1);
  console.log(all); // ['item0', 'item1', 'item2', 'item3']

  // LLEN - ความยาว list
  const len = await redis.llen(key);

  // LPOP/RPOP - ลบและ return element
  const first = await redis.lpop(key); // item0
  const last = await redis.rpop(key);  // item3

  // LINDEX - ดึง element ตาม index
  const second = await redis.lindex(key, 1);

  // LSET - แก้ไข element ตาม index
  await redis.lset(key, 0, 'new_item');

  // Queue pattern
  // Producer
  await redis.rpush('queue:tasks', JSON.stringify({ id: 1, type: 'email' }));

  // Consumer
  const task = await redis.lpop('queue:tasks');
  if (task) {
    const taskData = JSON.parse(task);
    console.log('Processing task:', taskData);
  }

  // Blocking pop (รอจนกว่าจะมี element)
  const [listName, item] = await redis.blpop('queue:tasks', 0); // 0 = wait forever
}
```

### ขั้นตอนที่ 647: Sets และ Sorted Sets

```javascript
// Sets - unique values
async function setExamples() {
  // SADD - เพิ่ม members
  await redis.sadd('tags', 'nodejs', 'javascript', 'backend');
  await redis.sadd('tags', 'nodejs'); // ซ้ำ ไม่ถูกเพิ่ม

  // SMEMBERS - ดู members ทั้งหมด
  const tags = await redis.smembers('tags');
  console.log(tags); // ['nodejs', 'javascript', 'backend']

  // SISMEMBER - ตรวจสอบ membership
  const isMember = await redis.sismember('tags', 'nodejs');

  // SCARD - จำนวน members
  const count = await redis.scard('tags');

  // SREM - ลบ member
  await redis.srem('tags', 'backend');

  // Set Operations
  await redis.sadd('set1', 'a', 'b', 'c');
  await redis.sadd('set2', 'b', 'c', 'd');

  const union = await redis.sunion('set1', 'set2');        // a,b,c,d
  const inter = await redis.sinter('set1', 'set2');        // b,c
  const diff = await redis.sdiff('set1', 'set2');          // a
}

// Sorted Sets - ranked data
async function sortedSetExamples() {
  const leaderboard = 'game:scores';

  // ZADD - เพิ่ม members พร้อม score
  await redis.zadd(leaderboard, 1500, 'player1');
  await redis.zadd(leaderboard, 2000, 'player2');
  await redis.zadd(leaderboard, 1800, 'player3');
  await redis.zadd(leaderboard, 2500, 'player4');

  // ZRANGE - ดู members ตาม rank (ascending)
  const bottom = await redis.zrange(leaderboard, 0, -1);

  // ZREVRANGE - descending
  const top = await redis.zrevrange(leaderboard, 0, 2, 'WITHSCORES');
  console.log(top); // ['player4', '2500', 'player2', '2000', ...]

  // ZSCORE - ดู score ของ member
  const score = await redis.zscore(leaderboard, 'player1');

  // ZRANK - ดู rank (0-based)
  const rank = await redis.zrevrank(leaderboard, 'player1');
  console.log(`player1 อยู่อันดับ ${rank + 1}`);

  // ZINCRBY - เพิ่ม score
  await redis.zincrby(leaderboard, 500, 'player1');

  // ZRANGEBYSCORE - filter ตาม score range
  const topPlayers = await redis.zrangebyscore(leaderboard, 1800, '+inf', 'WITHSCORES');
}
```

---

## Caching Strategies

### ขั้นตอนที่ 648: Cache-Aside Pattern

```javascript
// src/cache/cacheManager.js
const redis = require('../config/redis');

class CacheManager {
  constructor(defaultTTL = 3600) {
    this.defaultTTL = defaultTTL;
  }

  // Cache key generator
  key(prefix, ...parts) {
    return `${prefix}:${parts.join(':')}`;
  }

  // Get or Set (Cache-Aside)
  async getOrSet(key, fetchFn, ttl = this.defaultTTL) {
    // 1. ลองดึงจาก cache
    const cached = await redis.get(key);

    if (cached !== null) {
      console.log(`✅ Cache hit: ${key}`);
      return JSON.parse(cached);
    }

    // 2. Cache miss - ดึงจาก source
    console.log(`❌ Cache miss: ${key}`);
    const data = await fetchFn();

    // 3. เก็บลง cache
    if (data !== null && data !== undefined) {
      await redis.setex(key, ttl, JSON.stringify(data));
    }

    return data;
  }

  // Invalidate cache
  async invalidate(key) {
    await redis.del(key);
    console.log(`🗑️ Cache invalidated: ${key}`);
  }

  // Invalidate by pattern
  async invalidatePattern(pattern) {
    const keys = await redis.keys(pattern);
    if (keys.length > 0) {
      await redis.del(...keys);
      console.log(`🗑️ Invalidated ${keys.length} cache keys matching: ${pattern}`);
    }
  }

  // Get with stale-while-revalidate
  async getWithSWR(key, fetchFn, ttl, staleTTL) {
    const staleKey = `stale:${key}`;
    const cached = await redis.get(key);

    if (cached !== null) {
      // Fresh data
      return JSON.parse(cached);
    }

    const stale = await redis.get(staleKey);

    if (stale !== null) {
      // Serve stale data while refreshing in background
      fetchFn().then(async (data) => {
        await redis.setex(key, ttl, JSON.stringify(data));
        await redis.setex(staleKey, staleTTL, JSON.stringify(data));
      });
      return JSON.parse(stale);
    }

    // No cache at all - fetch fresh
    const data = await fetchFn();
    await redis.setex(key, ttl, JSON.stringify(data));
    await redis.setex(staleKey, staleTTL, JSON.stringify(data));
    return data;
  }
}

module.exports = new CacheManager();
```

### ขั้นตอนที่ 649: Caching Express Routes

```javascript
// src/middleware/cache.js
const redis = require('../config/redis');

// Cache middleware สำหรับ Express
function cacheMiddleware(ttl = 300) {
  return async (req, res, next) => {
    // ไม่ cache ถ้า method ไม่ใช่ GET
    if (req.method !== 'GET') {
      return next();
    }

    const cacheKey = `cache:${req.originalUrl}`;

    try {
      const cached = await redis.get(cacheKey);

      if (cached) {
        const data = JSON.parse(cached);
        res.setHeader('X-Cache', 'HIT');
        res.setHeader('X-Cache-TTL', await redis.ttl(cacheKey));
        return res.json(data);
      }

      // Override res.json เพื่อ intercept response
      const originalJson = res.json.bind(res);
      res.json = async (data) => {
        if (res.statusCode === 200) {
          await redis.setex(cacheKey, ttl, JSON.stringify(data));
        }
        res.setHeader('X-Cache', 'MISS');
        return originalJson(data);
      };

      next();
    } catch (error) {
      console.error('Cache middleware error:', error);
      next();
    }
  };
}

// User-specific cache
function userCacheMiddleware(ttl = 300) {
  return async (req, res, next) => {
    if (req.method !== 'GET' || !req.user) {
      return next();
    }

    const cacheKey = `cache:user:${req.user.id}:${req.originalUrl}`;

    try {
      const cached = await redis.get(cacheKey);
      if (cached) {
        return res.json(JSON.parse(cached));
      }

      const originalJson = res.json.bind(res);
      res.json = async (data) => {
        if (res.statusCode === 200) {
          await redis.setex(cacheKey, ttl, JSON.stringify(data));
        }
        return originalJson(data);
      };

      next();
    } catch (error) {
      next();
    }
  };
}

module.exports = { cacheMiddleware, userCacheMiddleware };
```

### ขั้นตอนที่ 650: Cache Invalidation

```javascript
// src/services/postService.js
const Post = require('../models/Post');
const cacheManager = require('../cache/cacheManager');

class PostService {
  async getPosts(filters = {}) {
    const cacheKey = cacheManager.key('posts', JSON.stringify(filters));

    return cacheManager.getOrSet(cacheKey, async () => {
      return Post.find(filters).sort({ createdAt: -1 });
    }, 300); // Cache 5 นาที
  }

  async getPost(id) {
    const cacheKey = cacheManager.key('post', id);

    return cacheManager.getOrSet(cacheKey, async () => {
      return Post.findById(id).populate('author', 'username');
    }, 600); // Cache 10 นาที
  }

  async createPost(data) {
    const post = await Post.create(data);

    // Invalidate posts list cache
    await cacheManager.invalidatePattern('cache:posts:*');

    return post;
  }

  async updatePost(id, data) {
    const post = await Post.findByIdAndUpdate(id, data, { new: true });

    // Invalidate specific post cache
    await cacheManager.invalidate(cacheManager.key('post', id));

    // Invalidate posts list cache
    await cacheManager.invalidatePattern('cache:posts:*');

    return post;
  }

  async deletePost(id) {
    await Post.findByIdAndDelete(id);

    // Invalidate caches
    await cacheManager.invalidate(cacheManager.key('post', id));
    await cacheManager.invalidatePattern('cache:posts:*');
  }
}

module.exports = new PostService();
```

---

## Session Management

### ขั้นตอนที่ 651: Redis Sessions

```bash
npm install express-session connect-redis
```

```javascript
// src/config/session.js
const session = require('express-session');
const RedisStore = require('connect-redis')(session);
const redis = require('./redis');

const sessionConfig = {
  store: new RedisStore({
    client: redis,
    prefix: 'sess:',
    ttl: 86400, // 24 ชั่วโมง
    disableTouch: false,
  }),
  secret: process.env.SESSION_SECRET || 'secret',
  resave: false,
  saveUninitialized: false,
  name: 'sessionId',
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 24 * 60 * 60 * 1000, // 24 ชั่วโมง
  },
};

module.exports = sessionConfig;
```

```javascript
// app.js
const express = require('express');
const session = require('express-session');
const sessionConfig = require('./config/session');

const app = express();

app.use(session(sessionConfig));

// Login route
app.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });

  if (!user || !(await user.comparePassword(password))) {
    return res.status(401).json({ message: 'Invalid credentials' });
  }

  // เก็บข้อมูลใน session
  req.session.userId = user._id;
  req.session.role = user.role;

  res.json({ message: 'Login successful', user: { id: user._id, name: user.name } });
});

// Logout route
app.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      return res.status(500).json({ message: 'Logout failed' });
    }
    res.clearCookie('sessionId');
    res.json({ message: 'Logged out successfully' });
  });
});

// Protected route
app.get('/profile', requireAuth, async (req, res) => {
  const user = await User.findById(req.session.userId);
  res.json(user);
});

function requireAuth(req, res, next) {
  if (!req.session.userId) {
    return res.status(401).json({ message: 'Unauthorized' });
  }
  next();
}
```

---

## Redis Pub/Sub

### ขั้นตอนที่ 652: Publish/Subscribe Pattern

```javascript
// src/pubsub/index.js
const Redis = require('ioredis');

// ต้องใช้ Redis clients แยกกันสำหรับ pub และ sub
const publisher = new Redis(process.env.REDIS_URL);
const subscriber = new Redis(process.env.REDIS_URL);

// Subscribe ช่อง
async function subscribe(channel, handler) {
  await subscriber.subscribe(channel);

  subscriber.on('message', (ch, message) => {
    if (ch === channel) {
      try {
        const data = JSON.parse(message);
        handler(data);
      } catch (error) {
        console.error('Failed to parse message:', error);
      }
    }
  });

  console.log(`✅ Subscribed to channel: ${channel}`);
}

// Subscribe หลาย channels
async function subscribePattern(pattern, handler) {
  await subscriber.psubscribe(pattern);

  subscriber.on('pmessage', (pat, channel, message) => {
    if (pat === pattern) {
      try {
        const data = JSON.parse(message);
        handler(channel, data);
      } catch (error) {
        console.error('Failed to parse message:', error);
      }
    }
  });
}

// Publish message
async function publish(channel, data) {
  const message = JSON.stringify(data);
  const receivers = await publisher.publish(channel, message);
  return receivers;
}

module.exports = { subscribe, subscribePattern, publish, publisher, subscriber };
```

### ขั้นตอนที่ 653: Event-Driven Architecture

```javascript
// src/events/userEvents.js
const { publish, subscribe } = require('../pubsub');

// Events constants
const USER_EVENTS = {
  REGISTERED: 'user:registered',
  LOGGED_IN: 'user:logged_in',
  PROFILE_UPDATED: 'user:profile_updated',
  PASSWORD_CHANGED: 'user:password_changed',
};

// Publishers
async function publishUserRegistered(user) {
  await publish(USER_EVENTS.REGISTERED, {
    userId: user.id,
    email: user.email,
    username: user.username,
    timestamp: new Date().toISOString(),
  });
}

async function publishUserLoggedIn(user) {
  await publish(USER_EVENTS.LOGGED_IN, {
    userId: user.id,
    ip: user.ip,
    timestamp: new Date().toISOString(),
  });
}

// Subscribers
async function setupUserEventHandlers() {
  // ส่ง welcome email เมื่อ user สมัคร
  await subscribe(USER_EVENTS.REGISTERED, async (data) => {
    console.log('User registered:', data.email);
    await sendWelcomeEmail(data.email);
    await createDefaultProfile(data.userId);
  });

  // Log login activity
  await subscribe(USER_EVENTS.LOGGED_IN, async (data) => {
    await LoginLog.create({
      userId: data.userId,
      ip: data.ip,
      timestamp: data.timestamp,
    });
  });
}

module.exports = { publishUserRegistered, publishUserLoggedIn, setupUserEventHandlers, USER_EVENTS };
```

---

## Rate Limiting ด้วย Redis

### ขั้นตอนที่ 654: Rate Limiter

```javascript
// src/middleware/rateLimiter.js
const redis = require('../config/redis');

// Fixed Window Rate Limiter
function fixedWindowRateLimiter({ windowMs = 60000, max = 100, keyGenerator }) {
  return async (req, res, next) => {
    const key = keyGenerator
      ? keyGenerator(req)
      : `ratelimit:${req.ip}:${req.path}`;

    const window = Math.floor(Date.now() / windowMs);
    const redisKey = `${key}:${window}`;

    try {
      const pipeline = redis.pipeline();
      pipeline.incr(redisKey);
      pipeline.pexpire(redisKey, windowMs);
      const results = await pipeline.exec();

      const count = results[0][1];

      res.setHeader('X-RateLimit-Limit', max);
      res.setHeader('X-RateLimit-Remaining', Math.max(0, max - count));
      res.setHeader('X-RateLimit-Reset', (window + 1) * windowMs);

      if (count > max) {
        return res.status(429).json({
          error: 'Too Many Requests',
          message: 'โปรดรอสักครู่แล้วลองอีกครั้ง',
          retryAfter: Math.ceil(windowMs / 1000),
        });
      }

      next();
    } catch (error) {
      console.error('Rate limit error:', error);
      next(); // Fail open - อนุญาตถ้า Redis มีปัญหา
    }
  };
}

// Sliding Window Rate Limiter (แม่นยำกว่า)
function slidingWindowRateLimiter({ windowMs = 60000, max = 100 }) {
  return async (req, res, next) => {
    const key = `sliding:${req.ip}`;
    const now = Date.now();
    const windowStart = now - windowMs;

    try {
      const pipeline = redis.pipeline();
      // ลบ timestamps เก่า
      pipeline.zremrangebyscore(key, '-inf', windowStart);
      // เพิ่ม timestamp ปัจจุบัน
      pipeline.zadd(key, now, `${now}-${Math.random()}`);
      // นับ requests ใน window
      pipeline.zcard(key);
      // ตั้ง expiry
      pipeline.pexpire(key, windowMs);

      const results = await pipeline.exec();
      const count = results[2][1];

      if (count > max) {
        const oldest = await redis.zrange(key, 0, 0, 'WITHSCORES');
        const resetTime = oldest.length > 0
          ? parseInt(oldest[1]) + windowMs
          : Date.now() + windowMs;

        return res.status(429).json({
          error: 'Too Many Requests',
          retryAfter: Math.ceil((resetTime - Date.now()) / 1000),
        });
      }

      next();
    } catch (error) {
      next();
    }
  };
}

// Token Bucket Rate Limiter
function tokenBucketRateLimiter({ capacity = 10, refillRate = 1, refillInterval = 1000 }) {
  return async (req, res, next) => {
    const key = `bucket:${req.ip}`;
    const now = Date.now();

    try {
      const bucketData = await redis.hgetall(key);

      let tokens = capacity;
      let lastRefill = now;

      if (bucketData && bucketData.tokens) {
        tokens = parseFloat(bucketData.tokens);
        lastRefill = parseInt(bucketData.lastRefill);

        // คำนวณ tokens ที่ refill
        const elapsed = now - lastRefill;
        const refillAmount = (elapsed / refillInterval) * refillRate;
        tokens = Math.min(capacity, tokens + refillAmount);
        lastRefill = now;
      }

      if (tokens < 1) {
        return res.status(429).json({ error: 'Too Many Requests' });
      }

      tokens -= 1;

      await redis.hset(key, { tokens: tokens.toString(), lastRefill: lastRefill.toString() });
      await redis.pexpire(key, capacity * refillInterval);

      res.setHeader('X-RateLimit-Remaining', Math.floor(tokens));
      next();
    } catch (error) {
      next();
    }
  };
}

module.exports = { fixedWindowRateLimiter, slidingWindowRateLimiter, tokenBucketRateLimiter };
```

### ขั้นตอนที่ 655: ใช้งาน Rate Limiter

```javascript
// app.js
const { fixedWindowRateLimiter, slidingWindowRateLimiter } = require('./middleware/rateLimiter');

// Global rate limit
app.use(fixedWindowRateLimiter({ windowMs: 15 * 60 * 1000, max: 100 }));

// เข้มงวดกับ auth endpoints
app.use('/api/auth', slidingWindowRateLimiter({ windowMs: 60 * 1000, max: 5 }));

// API endpoints
app.use('/api', fixedWindowRateLimiter({ windowMs: 60 * 1000, max: 60 }));
```

---

## Redis Cluster

### ขั้นตอนที่ 656: Redis Sentinel

```javascript
// High Availability ด้วย Redis Sentinel
const Redis = require('ioredis');

const redis = new Redis({
  sentinels: [
    { host: 'sentinel-1', port: 26379 },
    { host: 'sentinel-2', port: 26379 },
    { host: 'sentinel-3', port: 26379 },
  ],
  name: 'mymaster',
  password: process.env.REDIS_PASSWORD,
  sentinelPassword: process.env.SENTINEL_PASSWORD,
});
```

### ขั้นตอนที่ 657: Redis Cluster Mode

```javascript
// Horizontal scaling ด้วย Redis Cluster
const Redis = require('ioredis');

const cluster = new Redis.Cluster([
  { host: 'redis-node-1', port: 7000 },
  { host: 'redis-node-2', port: 7001 },
  { host: 'redis-node-3', port: 7002 },
], {
  redisOptions: {
    password: process.env.REDIS_PASSWORD,
  },
  enableReadyCheck: true,
  maxRedirections: 16,
});

// Cluster มี 16,384 hash slots
// Keys ถูก distribute ตาม hash ของ key
```

---

## Advanced Caching Patterns

### ขั้นตอนที่ 658: Write-Through Cache

```javascript
// Write-through: เขียนทั้ง cache และ DB พร้อมกัน
class WriteThroughCache {
  constructor(ttl = 3600) {
    this.ttl = ttl;
  }

  async write(key, data, saveFn) {
    // 1. บันทึกลง DB ก่อน
    const saved = await saveFn(data);

    // 2. อัปเดต cache ทันที
    await redis.setex(key, this.ttl, JSON.stringify(saved));

    return saved;
  }

  async read(key, fetchFn) {
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);

    const data = await fetchFn();
    if (data) {
      await redis.setex(key, this.ttl, JSON.stringify(data));
    }
    return data;
  }
}
```

### ขั้นตอนที่ 659: Cache Warming

```javascript
// Pre-populate cache เมื่อเริ่ม server
async function warmupCache() {
  console.log('🔥 Warming up cache...');

  try {
    // Load popular data ล่วงหน้า
    const [topPosts, featuredUsers] = await Promise.all([
      Post.find({ status: 'PUBLISHED' }).sort({ viewCount: -1 }).limit(10),
      User.find({ featured: true }).limit(5),
    ]);

    const pipeline = redis.pipeline();

    topPosts.forEach(post => {
      pipeline.setex(
        `post:${post._id}`,
        3600,
        JSON.stringify(post)
      );
    });

    featuredUsers.forEach(user => {
      pipeline.setex(
        `user:${user._id}`,
        3600,
        JSON.stringify(user)
      );
    });

    // Cache top posts list
    pipeline.setex('posts:top', 1800, JSON.stringify(topPosts));

    await pipeline.exec();
    console.log('✅ Cache warmed up successfully');
  } catch (error) {
    console.error('Cache warmup failed:', error);
  }
}
```

### ขั้นตอนที่ 660: Cache Health Check

```javascript
// src/health/redis.js
async function checkRedisHealth() {
  try {
    const start = Date.now();
    await redis.ping();
    const latency = Date.now() - start;

    const info = await redis.info('server');
    const memory = await redis.info('memory');

    return {
      status: 'healthy',
      latency: `${latency}ms`,
      info: parseRedisInfo(info),
      memory: parseRedisInfo(memory),
    };
  } catch (error) {
    return {
      status: 'unhealthy',
      error: error.message,
    };
  }
}

function parseRedisInfo(info) {
  const lines = info.split('\r\n');
  const result = {};
  lines.forEach(line => {
    if (line.includes(':')) {
      const [key, value] = line.split(':');
      result[key] = value;
    }
  });
  return result;
}

// Health endpoint
app.get('/health/redis', async (req, res) => {
  const health = await checkRedisHealth();
  const statusCode = health.status === 'healthy' ? 200 : 503;
  res.status(statusCode).json(health);
});
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Implement Leaderboard

```javascript
// TODO: สร้าง leaderboard ด้วย Redis Sorted Set
// - เพิ่มคะแนน
// - ดู top 10
// - ดูอันดับของ user เฉพาะ
// - อัปเดตคะแนน
class Leaderboard {
  async addScore(userId, score) { /* TODO */ }
  async getTopN(n = 10) { /* TODO */ }
  async getUserRank(userId) { /* TODO */ }
  async updateScore(userId, delta) { /* TODO */ }
}
```

### แบบฝึกหัดที่ 2: Distributed Lock

```javascript
// TODO: สร้าง distributed lock ด้วย Redis
// ป้องกัน race conditions ใน distributed environment
class RedisLock {
  async acquire(resource, ttl = 10000) { /* TODO */ }
  async release(resource) { /* TODO */ }
  async withLock(resource, fn) { /* TODO */ }
}
```

### แบบฝึกหัดที่ 3: Implement Shopping Cart

```javascript
// TODO: สร้าง shopping cart ด้วย Redis Hash
// - เพิ่มสินค้า
// - ลบสินค้า
// - อัปเดตจำนวน
// - ดูรายการทั้งหมด
// - ล้าง cart
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Redis Basics** - data structures และ commands ต่างๆ
2. **Caching** - cache-aside, write-through patterns
3. **Sessions** - เก็บ sessions ใน Redis
4. **Pub/Sub** - event-driven communication
5. **Rate Limiting** - fixed window, sliding window, token bucket
6. **Clustering** - Redis Sentinel และ Cluster mode

ในบทถัดไปเราจะเรียนรู้ **Message Queues** - RabbitMQ, Bull และ job processing
