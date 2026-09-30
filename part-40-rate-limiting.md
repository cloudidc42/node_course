# Part 40: Rate Limiting

> ขั้นตอนที่ 40-40 จาก 1000

---

## สารบัญ

1. [Rate Limiting คืออะไร](#rate-limiting-คืออะไร)
2. [express-rate-limit](#express-rate-limit)
3. [Redis-based Rate Limiting](#redis-based-rate-limiting)
4. [IP-based vs User-based](#ip-based-vs-user-based)
5. [Sliding Window Algorithm](#sliding-window-algorithm)
6. [Advanced Strategies](#advanced-strategies)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Rate Limiting คืออะไร

Rate Limiting คือการจำกัดจำนวน requests ที่ client สามารถส่งมาได้ในช่วงเวลาหนึ่ง

### ทำไมต้องมี Rate Limiting

```
ป้องกัน:
- DDoS attacks (ส่ง requests จำนวนมาก)
- Brute force attacks (เดา password)
- API abuse (ใช้เกิน quota)
- Web scraping (ดึงข้อมูลมากเกินไป)
- Expensive operations (email sending)

ประโยชน์:
- ปกป้อง server resources
- ความเป็นธรรมสำหรับทุก user
- ประหยัด cost (external APIs)
- ป้องกัน data leak
```

### Rate Limit Algorithms

```
1. Fixed Window Counter
   [0-60s]: |||||| (6 requests)
   [60-120s]: || (2 requests)
   
   ง่าย แต่มี edge case ที่ burst ได้ที่ window boundary

2. Sliding Window Counter
   [0-60s]: นับ requests ในช่วง 60s ที่ผ่านมาเสมอ
   
   แม่นยำกว่า แต่ใช้ memory มากกว่า

3. Token Bucket
   Bucket มี tokens N ตัว เติมใหม่ทุก T seconds
   แต่ละ request ใช้ 1 token
   
   รองรับ burst traffic ได้ดี

4. Leaky Bucket
   Requests เข้า queue แบบ FIFO, ประมวลผลด้วย rate คงที่
   
   Output rate สม่ำเสมอ แต่อาจ drop requests
```

---

## express-rate-limit

```bash
npm install express-rate-limit
```

### Basic Usage

```javascript
const rateLimit = require('express-rate-limit');

// Global rate limiter
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,                  // 100 requests per window
  message: {
    status: 'error',
    code: 'RATE_LIMIT_EXCEEDED',
    message: 'Too many requests, please try again later.',
  },
  standardHeaders: true,     // Return rate limit info in headers (RateLimit-*)
  legacyHeaders: false,      // Disable X-RateLimit-* headers
});

app.use('/api/', globalLimiter);
```

### Multiple Rate Limiters

```javascript
const rateLimit = require('express-rate-limit');

// 1. General API limit
const apiLimiter = rateLimit({
  windowMs: 60 * 1000,  // 1 minute
  max: 60,              // 60 requests/minute
  message: { error: 'Too many API requests' },
});

// 2. Auth endpoints (stricter)
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 10,                    // 10 attempts
  message: { error: 'Too many auth attempts, try again in 15 minutes' },
  skipSuccessfulRequests: true, // ไม่นับ successful logins
});

// 3. Email sending
const emailLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,  // 1 hour
  max: 5,                     // 5 emails/hour
  message: { error: 'Email limit exceeded' },
});

// 4. File upload
const uploadLimiter = rateLimit({
  windowMs: 60 * 1000,  // 1 minute
  max: 3,               // 3 uploads/minute
  message: { error: 'Upload limit exceeded' },
});

// ใช้งาน
app.use('/api/', apiLimiter);
app.use('/api/auth/', authLimiter);
app.post('/api/users/send-email', emailLimiter);
app.post('/api/uploads', uploadLimiter);
```

### Advanced Configuration

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  
  // Custom key generator (default: req.ip)
  keyGenerator: (req) => {
    // ใช้ userId ถ้า login แล้ว, ไม่งั้นใช้ IP
    return req.user?.id || req.ip;
  },
  
  // Skip certain requests
  skip: (req) => {
    // ไม่ rate limit internal services
    if (req.ip === '127.0.0.1') return true;
    // ไม่ rate limit admins
    if (req.user?.role === 'admin') return true;
    return false;
  },
  
  // Custom handler when limit exceeded
  handler: (req, res) => {
    console.warn(`Rate limit exceeded for ${req.ip}`);
    
    res.status(429).json({
      status: 'error',
      code: 'RATE_LIMIT_EXCEEDED',
      message: 'Too many requests',
      retryAfter: Math.ceil(req.rateLimit.resetTime / 1000),
    });
  },
  
  // Callback on limit
  onLimitReached: (req, res, options) => {
    // Log หรือ alert เมื่อถึง limit
    alertTeam(`IP ${req.ip} hit rate limit`);
  },
  
  // Standard headers
  standardHeaders: true,
  
  // Draft RFC 6585
  statusCode: 429,
});
```

---

## Redis-based Rate Limiting

express-rate-limit ใช้ memory store โดย default ซึ่งไม่ sync กันระหว่าง servers ต้องใช้ Redis store

```bash
npm install rate-limit-redis ioredis
```

```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const { createClient } = require('ioredis');

const redisClient = createClient({
  host: process.env.REDIS_HOST || 'localhost',
  port: 6379,
  password: process.env.REDIS_PASSWORD,
});

// Global rate limiter ด้วย Redis
const apiLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 100,
  standardHeaders: true,
  
  // Redis store - ทำงานได้กับ multiple servers
  store: new RedisStore({
    client: redisClient,
    prefix: 'rl:',            // Redis key prefix
    sendCommand: (...args) => redisClient.call(...args),
  }),
});

app.use('/api/', apiLimiter);
```

### Custom Redis Rate Limiter

```javascript
// utils/redisRateLimiter.js
const redis = require('../config/redis');

class RedisRateLimiter {
  /**
   * Fixed Window Rate Limiter
   */
  static async fixedWindow(key, max, windowSec) {
    const current = await redis.incr(key);
    
    if (current === 1) {
      await redis.expire(key, windowSec);
    }
    
    const ttl = await redis.ttl(key);
    
    return {
      allowed: current <= max,
      count: current,
      remaining: Math.max(0, max - current),
      resetAt: Date.now() + (ttl * 1000),
    };
  }
  
  /**
   * Sliding Window Rate Limiter (ใช้ Sorted Set)
   */
  static async slidingWindow(key, max, windowSec) {
    const now = Date.now();
    const windowStart = now - (windowSec * 1000);
    
    const pipeline = redis.pipeline();
    
    // ลบ old entries
    pipeline.zremrangebyscore(key, '-inf', windowStart);
    // นับ current entries
    pipeline.zcard(key);
    // เพิ่ม current request
    pipeline.zadd(key, now, `${now}-${Math.random()}`);
    // ตั้ง TTL
    pipeline.expire(key, windowSec);
    
    const results = await pipeline.exec();
    const count = results[1][1] + 1; // +1 สำหรับ request ปัจจุบัน
    
    return {
      allowed: count <= max,
      count,
      remaining: Math.max(0, max - count),
      resetAt: now + (windowSec * 1000),
    };
  }
  
  /**
   * Token Bucket Rate Limiter
   */
  static async tokenBucket(key, capacity, refillRate, refillPeriodSec) {
    const now = Date.now();
    
    // Lua script สำหรับ atomic operation
    const script = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local refillRate = tonumber(ARGV[2])
      local refillPeriod = tonumber(ARGV[3])
      local now = tonumber(ARGV[4])
      
      local bucket = redis.call('HMGET', key, 'tokens', 'lastRefill')
      local tokens = tonumber(bucket[1]) or capacity
      local lastRefill = tonumber(bucket[2]) or now
      
      -- Calculate tokens to add
      local elapsed = (now - lastRefill) / 1000
      local newTokens = math.min(capacity, tokens + (elapsed * refillRate / refillPeriod))
      
      if newTokens >= 1 then
        redis.call('HMSET', key, 'tokens', newTokens - 1, 'lastRefill', now)
        redis.call('EXPIRE', key, refillPeriod * 2)
        return {1, math.floor(newTokens - 1)}
      else
        redis.call('HMSET', key, 'tokens', newTokens, 'lastRefill', now)
        redis.call('EXPIRE', key, refillPeriod * 2)
        return {0, 0}
      end
    `;
    
    const result = await redis.eval(
      script, 1, key,
      capacity, refillRate, refillPeriodSec, now
    );
    
    return {
      allowed: result[0] === 1,
      remaining: result[1],
    };
  }
}

module.exports = RedisRateLimiter;
```

---

## IP-based vs User-based

### IP-based Rate Limiting

```javascript
// IP-based - ใช้กับ unauthenticated endpoints
const ipLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  keyGenerator: (req) => {
    // รองรับ proxy (X-Forwarded-For)
    return req.headers['x-forwarded-for']?.split(',')[0].trim()
      || req.headers['x-real-ip']
      || req.connection.remoteAddress;
  },
});
```

### User-based Rate Limiting

```javascript
// User-based - ใช้กับ authenticated endpoints
const userLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 60,
  keyGenerator: (req) => {
    // ใช้ userId ถ้า login แล้ว
    if (req.user) {
      return `user:${req.user.id}`;
    }
    // fallback ไป IP
    return `ip:${req.ip}`;
  },
  skip: (req) => {
    // Premium users ไม่ถูก rate limit
    return req.user?.tier === 'premium';
  },
});
```

### Role-based Rate Limiting

```javascript
// สร้าง rate limiter ตาม user tier
function createTieredLimiter() {
  const tiers = {
    free: { max: 60, window: 60 * 1000 },
    pro: { max: 300, window: 60 * 1000 },
    premium: { max: 1000, window: 60 * 1000 },
    admin: { max: 999999, window: 60 * 1000 },
  };
  
  return async (req, res, next) => {
    const tier = req.user?.tier || 'free';
    const { max, window } = tiers[tier] || tiers.free;
    
    const key = `rl:${req.user?.id || req.ip}`;
    const result = await RedisRateLimiter.slidingWindow(key, max, window / 1000);
    
    res.set({
      'X-RateLimit-Limit': max,
      'X-RateLimit-Remaining': result.remaining,
      'X-RateLimit-Reset': Math.ceil(result.resetAt / 1000),
      'X-RateLimit-Policy': `${max};w=${window / 1000}`,
    });
    
    if (!result.allowed) {
      return res.status(429).json({
        error: 'Rate limit exceeded',
        tier,
        retryAfter: Math.ceil((result.resetAt - Date.now()) / 1000),
      });
    }
    
    next();
  };
}
```

---

## Sliding Window Algorithm

```javascript
// ตัวอย่าง Sliding Window ที่สมบูรณ์
class SlidingWindowRateLimiter {
  constructor(redis, options = {}) {
    this.redis = redis;
    this.prefix = options.prefix || 'sw:';
    this.max = options.max || 100;
    this.window = options.window || 60; // seconds
  }
  
  getKey(identifier) {
    return `${this.prefix}${identifier}`;
  }
  
  async consume(identifier) {
    const key = this.getKey(identifier);
    const now = Date.now();
    const windowStart = now - (this.window * 1000);
    
    const script = `
      local key = KEYS[1]
      local now = tonumber(ARGV[1])
      local windowStart = tonumber(ARGV[2])
      local max = tonumber(ARGV[3])
      local window = tonumber(ARGV[4])
      
      -- ลบ old entries
      redis.call('ZREMRANGEBYSCORE', key, '-inf', windowStart)
      
      -- นับ current entries
      local count = redis.call('ZCARD', key)
      
      -- เพิ่ม current request
      redis.call('ZADD', key, now, now .. '-' .. math.random())
      redis.call('EXPIRE', key, window)
      
      return {count, count < max}
    `;
    
    const [count, allowed] = await this.redis.eval(
      script, 1, key,
      now, windowStart, this.max, this.window
    );
    
    return {
      allowed: allowed === 1,
      count: count + 1,
      remaining: Math.max(0, this.max - count - 1),
      limit: this.max,
      resetAt: now + (this.window * 1000),
    };
  }
  
  async reset(identifier) {
    await this.redis.del(this.getKey(identifier));
  }
  
  async getStatus(identifier) {
    const key = this.getKey(identifier);
    const now = Date.now();
    const windowStart = now - (this.window * 1000);
    
    await this.redis.zremrangebyscore(key, '-inf', windowStart);
    const count = await this.redis.zcard(key);
    
    return {
      count,
      remaining: Math.max(0, this.max - count),
      limit: this.max,
    };
  }
  
  middleware(keyFn) {
    return async (req, res, next) => {
      const identifier = keyFn ? keyFn(req) : req.ip;
      
      const result = await this.consume(identifier);
      
      res.set({
        'RateLimit-Limit': result.limit,
        'RateLimit-Remaining': result.remaining,
        'RateLimit-Reset': Math.ceil(result.resetAt / 1000),
      });
      
      if (!result.allowed) {
        return res.status(429).json({
          error: 'Too many requests',
          retryAfter: Math.ceil((result.resetAt - Date.now()) / 1000),
        });
      }
      
      next();
    };
  }
}

// ใช้งาน
const { redis } = require('./config/redis');

const apiLimiter = new SlidingWindowRateLimiter(redis, {
  max: 100,
  window: 60,
  prefix: 'api:',
});

app.use('/api/', apiLimiter.middleware());

// Per-user rate limiting
app.use('/api/', apiLimiter.middleware(
  (req) => req.user?.id || req.ip
));
```

---

## Advanced Strategies

### Distributed Rate Limiting

```javascript
// สำหรับ microservices ที่ต้องการ shared rate limit
class DistributedRateLimiter {
  constructor(redis, serviceName) {
    this.redis = redis;
    this.serviceName = serviceName;
  }
  
  async checkLimit(userId, action, limit, windowSec) {
    const key = `drl:${this.serviceName}:${action}:${userId}`;
    
    return SlidingWindowRateLimiter.prototype.consume.call(
      { redis: this.redis, prefix: '', max: limit, window: windowSec },
      key
    );
  }
}
```

### Rate Limiting Dashboard

```javascript
// ดู rate limit stats ผ่าน API (admin only)
app.get('/admin/rate-limit/stats', authenticate, authorize('admin'), async (req, res) => {
  const pattern = 'rl:*';
  const keys = await redis.keys(pattern);
  
  const stats = await Promise.all(
    keys.map(async (key) => {
      const type = await redis.type(key);
      let count = 0;
      
      if (type === 'zset') {
        count = await redis.zcard(key);
      } else if (type === 'string') {
        count = parseInt(await redis.get(key)) || 0;
      }
      
      return { key: key.replace('rl:', ''), count };
    })
  );
  
  res.json({
    total: stats.length,
    clients: stats
      .filter(s => s.count > 0)
      .sort((a, b) => b.count - a.count)
      .slice(0, 100), // top 100
  });
});
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Multi-tier Rate Limiting

สร้าง rate limiting system ที่:
- Free users: 60 requests/minute
- Pro users: 300 requests/minute
- Premium users: 1000 requests/minute
- ใช้ Redis สำหรับ distributed tracking

### แบบฝึกหัดที่ 2: Login Protection

```javascript
// ป้องกัน brute force login
// - 5 failed attempts per email ใน 15 นาที
// - 20 failed attempts per IP ใน 15 นาที
// - ล็อค account หลัง 10 failed attempts

// Test:
// 1. รัน loop POST /auth/login ด้วย email เดิมแต่ password ผิด
// 2. ตรวจสอบว่า rate limit ทำงานหลัง 5 attempts
// 3. ตรวจสอบว่า account ถูกล็อคหลัง 10 attempts
```

### แบบฝึกหัดที่ 3: API Quota System

สร้าง API quota system:
- API key-based (ไม่ใช่ IP)
- Monthly quota (1000 calls/month สำหรับ free)
- Real-time usage tracking
- Webhook แจ้งเตือนเมื่อใกล้ถึง limit (80%, 100%)

---

## สรุป

Rate Limiting ป้องกัน API จาก abuse และ attacks

| Algorithm | ข้อดี | ข้อเสีย |
|-----------|-------|---------|
| Fixed Window | ง่าย | Burst ที่ boundary |
| Sliding Window | แม่นยำ | Memory มากกว่า |
| Token Bucket | รองรับ burst | ซับซ้อนกว่า |
| Leaky Bucket | Output สม่ำเสมอ | อาจ drop requests |

**ถัดไป**: [Part 41: CORS →](./part-41-cors.md)
