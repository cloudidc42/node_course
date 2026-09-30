# Part 48 | ขั้นตอนที่ 821-840 จาก 1000

## Rate Limiting - การจำกัดอัตราการร้องขอ

Rate Limiting เป็นเทคนิคสำคัญในการป้องกัน API จากการถูกใช้งานเกินขีดจำกัด ช่วยปกป้องระบบจาก DDoS attacks และรักษาคุณภาพการให้บริการ

---

## ขั้นตอนที่ 821: ทำความเข้าใจ Rate Limiting

Rate Limiting คือการควบคุมจำนวนคำขอที่ client สามารถส่งมายัง server ในช่วงเวลาที่กำหนด

### เหตุผลที่ต้องใช้ Rate Limiting:
- **ป้องกัน DDoS**: จำกัดการโจมตีแบบ Distributed Denial of Service
- **ป้องกัน Brute Force**: จำกัดการลองรหัสผ่านซ้ำๆ
- **ควบคุมค่าใช้จ่าย**: ป้องกันการใช้ทรัพยากรเกินกำหนด
- **รักษาคุณภาพบริการ**: ให้บริการเท่าเทียมกันแก่ผู้ใช้ทุกคน
- **ป้องกัน API Abuse**: ป้องกันการใช้งาน API ในทางที่ผิด

### ประเภทของ Rate Limiting:
1. **Fixed Window**: นับจำนวนคำขอในช่วงเวลาคงที่
2. **Sliding Window**: ช่วงเวลาเลื่อนตามเวลาจริง
3. **Token Bucket**: ใช้ระบบ token สะสม
4. **Leaky Bucket**: ควบคุมอัตราการไหลออก

---

## ขั้นตอนที่ 822: ติดตั้ง express-rate-limit

```bash
npm install express-rate-limit
npm install rate-limit-redis ioredis
```

### การตั้งค่าพื้นฐาน:

```javascript
// middleware/rateLimiter.js
const rateLimit = require('express-rate-limit');

// Rate limiter พื้นฐาน
const basicLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 นาที
  max: 100,                  // สูงสุด 100 คำขอต่อ window
  standardHeaders: true,     // ส่ง RateLimit headers ตามมาตรฐาน
  legacyHeaders: false,      // ปิด X-RateLimit-* headers เก่า
  message: {
    status: 429,
    message: 'คำขอมากเกินไป กรุณาลองใหม่ภายหลัง'
  }
});

module.exports = basicLimiter;
```

```javascript
// app.js
const express = require('express');
const basicLimiter = require('./middleware/rateLimiter');

const app = express();

// ใช้กับทุก routes
app.use(basicLimiter);

// หรือใช้กับ route เฉพาะ
app.use('/api/', basicLimiter);

app.get('/', (req, res) => {
  res.json({ message: 'Hello World' });
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

---

## ขั้นตอนที่ 823: Rate Limiter ขั้นสูง

```javascript
// middleware/advancedRateLimiter.js
const rateLimit = require('express-rate-limit');

// สร้าง function สำหรับสร้าง rate limiter
const createRateLimiter = (options = {}) => {
  const defaults = {
    windowMs: 15 * 60 * 1000,
    max: 100,
    standardHeaders: true,
    legacyHeaders: false,
    // กำหนด key จาก IP + User Agent
    keyGenerator: (req) => {
      return `${req.ip}-${req.headers['user-agent']}`;
    },
    // Handler เมื่อถูก rate limit
    handler: (req, res, next, options) => {
      res.status(options.statusCode).json({
        success: false,
        message: options.message,
        retryAfter: Math.ceil(options.windowMs / 1000),
        limit: options.max
      });
    },
    // Skip บาง requests
    skip: (req) => {
      // ไม่ limit requests จาก localhost ใน development
      if (process.env.NODE_ENV === 'development' && req.ip === '::1') {
        return true;
      }
      return false;
    }
  };

  return rateLimit({ ...defaults, ...options });
};

// Rate limiters สำหรับ endpoint ต่างๆ
const apiLimiter = createRateLimiter({
  windowMs: 15 * 60 * 1000,
  max: 100,
  message: 'API rate limit exceeded'
});

const authLimiter = createRateLimiter({
  windowMs: 60 * 60 * 1000, // 1 ชั่วโมง
  max: 5,                    // สูงสุด 5 ครั้งต่อชั่วโมง
  message: 'Too many login attempts'
});

const uploadLimiter = createRateLimiter({
  windowMs: 60 * 60 * 1000,
  max: 10,
  message: 'Upload limit exceeded'
});

module.exports = { apiLimiter, authLimiter, uploadLimiter };
```

---

## ขั้นตอนที่ 824: Redis-based Rate Limiting

Redis-based rate limiting เหมาะสำหรับ distributed systems ที่มีหลาย server instances

```bash
npm install rate-limit-redis ioredis
```

```javascript
// config/redis.js
const Redis = require('ioredis');

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: process.env.REDIS_PORT || 6379,
  password: process.env.REDIS_PASSWORD,
  retryStrategy: (times) => {
    const delay = Math.min(times * 50, 2000);
    return delay;
  }
});

redis.on('connect', () => {
  console.log('Redis connected');
});

redis.on('error', (err) => {
  console.error('Redis error:', err);
});

module.exports = redis;
```

```javascript
// middleware/redisRateLimiter.js
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const redis = require('../config/redis');

const createRedisRateLimiter = (options = {}) => {
  const store = new RedisStore({
    sendCommand: (...args) => redis.call(...args),
    prefix: options.prefix || 'rl:',
  });

  return rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 100,
    standardHeaders: true,
    legacyHeaders: false,
    store,
    ...options
  });
};

// Rate limiters ที่ใช้ Redis
const globalLimiter = createRedisRateLimiter({
  prefix: 'rl:global:',
  windowMs: 15 * 60 * 1000,
  max: 1000
});

const apiLimiter = createRedisRateLimiter({
  prefix: 'rl:api:',
  windowMs: 15 * 60 * 1000,
  max: 100
});

const authLimiter = createRedisRateLimiter({
  prefix: 'rl:auth:',
  windowMs: 60 * 60 * 1000,
  max: 5
});

module.exports = { globalLimiter, apiLimiter, authLimiter };
```

---

## ขั้นตอนที่ 825: Sliding Window Rate Limiting

Sliding Window ให้ความแม่นยำกว่า Fixed Window

```javascript
// middleware/slidingWindowLimiter.js
const redis = require('../config/redis');

class SlidingWindowLimiter {
  constructor(options = {}) {
    this.windowMs = options.windowMs || 60 * 1000; // 1 นาที
    this.max = options.max || 60;
    this.keyPrefix = options.keyPrefix || 'sw:';
  }

  async isAllowed(identifier) {
    const key = `${this.keyPrefix}${identifier}`;
    const now = Date.now();
    const windowStart = now - this.windowMs;

    // ใช้ Redis sorted set สำหรับ sliding window
    const pipeline = redis.pipeline();
    
    // ลบ requests ที่หมดอายุออก
    pipeline.zremrangebyscore(key, '-inf', windowStart);
    
    // นับจำนวน requests ปัจจุบัน
    pipeline.zcard(key);
    
    // เพิ่ม request ปัจจุบัน
    pipeline.zadd(key, now, `${now}-${Math.random()}`);
    
    // ตั้งเวลาหมดอายุ
    pipeline.pexpire(key, this.windowMs);

    const results = await pipeline.exec();
    const currentCount = results[1][1];

    return {
      allowed: currentCount < this.max,
      current: currentCount,
      remaining: Math.max(0, this.max - currentCount - 1),
      resetAt: new Date(now + this.windowMs)
    };
  }
}

// Middleware
const createSlidingWindowMiddleware = (options = {}) => {
  const limiter = new SlidingWindowLimiter(options);

  return async (req, res, next) => {
    try {
      const identifier = req.ip;
      const result = await limiter.isAllowed(identifier);

      // ตั้ง headers
      res.set({
        'X-RateLimit-Limit': options.max || 60,
        'X-RateLimit-Remaining': result.remaining,
        'X-RateLimit-Reset': result.resetAt.toISOString()
      });

      if (!result.allowed) {
        return res.status(429).json({
          success: false,
          message: 'Rate limit exceeded',
          retryAfter: Math.ceil(options.windowMs / 1000)
        });
      }

      next();
    } catch (error) {
      console.error('Rate limiter error:', error);
      next(); // ถ้า error ให้ผ่านไปได้ (fail open)
    }
  };
};

module.exports = { SlidingWindowLimiter, createSlidingWindowMiddleware };
```

---

## ขั้นตอนที่ 826: Token Bucket Algorithm

Token Bucket อนุญาตให้มี burst traffic ได้ในระดับที่กำหนด

```javascript
// middleware/tokenBucket.js
const redis = require('../config/redis');

class TokenBucketLimiter {
  constructor(options = {}) {
    this.capacity = options.capacity || 100;     // ความจุ bucket
    this.refillRate = options.refillRate || 10;  // token ที่เพิ่มต่อวินาที
    this.keyPrefix = options.keyPrefix || 'tb:';
  }

  async consume(identifier, tokens = 1) {
    const key = `${this.keyPrefix}${identifier}`;
    const now = Date.now();
    
    const script = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local refill_rate = tonumber(ARGV[2])
      local tokens_requested = tonumber(ARGV[3])
      local now = tonumber(ARGV[4])
      
      -- ดึงข้อมูล bucket ปัจจุบัน
      local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
      local current_tokens = tonumber(bucket[1]) or capacity
      local last_refill = tonumber(bucket[2]) or now
      
      -- คำนวณ tokens ที่เพิ่มขึ้น
      local elapsed = (now - last_refill) / 1000
      local new_tokens = math.min(capacity, current_tokens + elapsed * refill_rate)
      
      -- ตรวจสอบว่ามี token พอหรือไม่
      if new_tokens >= tokens_requested then
        new_tokens = new_tokens - tokens_requested
        redis.call('HMSET', key, 'tokens', new_tokens, 'last_refill', now)
        redis.call('PEXPIRE', key, 60000)
        return {1, math.floor(new_tokens)}
      else
        redis.call('HMSET', key, 'tokens', new_tokens, 'last_refill', now)
        redis.call('PEXPIRE', key, 60000)
        return {0, math.floor(new_tokens)}
      end
    `;

    const result = await redis.eval(
      script, 1, key,
      this.capacity, this.refillRate, tokens, now
    );

    return {
      allowed: result[0] === 1,
      remaining: result[1]
    };
  }
}

// Middleware factory
const createTokenBucketMiddleware = (options = {}) => {
  const limiter = new TokenBucketLimiter(options);

  return async (req, res, next) => {
    try {
      const result = await limiter.consume(req.ip);
      
      res.set('X-RateLimit-Remaining', result.remaining);

      if (!result.allowed) {
        return res.status(429).json({
          success: false,
          message: 'Too many requests',
          remaining: result.remaining
        });
      }

      next();
    } catch (error) {
      console.error('Token bucket error:', error);
      next();
    }
  };
};

module.exports = { TokenBucketLimiter, createTokenBucketMiddleware };
```

---

## ขั้นตอนที่ 827: Rate Limiting ตาม User/API Key

```javascript
// middleware/userRateLimiter.js
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const redis = require('../config/redis');

// Rate limits ตาม tier ของผู้ใช้
const TIER_LIMITS = {
  free: { windowMs: 60 * 60 * 1000, max: 100 },
  basic: { windowMs: 60 * 60 * 1000, max: 1000 },
  premium: { windowMs: 60 * 60 * 1000, max: 10000 },
  enterprise: { windowMs: 60 * 60 * 1000, max: 100000 }
};

const userBasedLimiter = async (req, res, next) => {
  try {
    // ดึง user tier จาก token
    const tier = req.user ? req.user.tier : 'free';
    const limits = TIER_LIMITS[tier] || TIER_LIMITS.free;
    
    const key = req.user 
      ? `user:${req.user.id}` 
      : `ip:${req.ip}`;
    
    const windowStart = Date.now() - limits.windowMs;
    
    // ใช้ Redis sorted set
    const pipeline = redis.pipeline();
    pipeline.zremrangebyscore(`rl:${key}`, '-inf', windowStart);
    pipeline.zcard(`rl:${key}`);
    pipeline.zadd(`rl:${key}`, Date.now(), Date.now().toString());
    pipeline.pexpire(`rl:${key}`, limits.windowMs);
    
    const results = await pipeline.exec();
    const count = results[1][1];
    
    // ตั้ง headers
    res.set({
      'X-RateLimit-Limit': limits.max,
      'X-RateLimit-Remaining': Math.max(0, limits.max - count),
      'X-RateLimit-Tier': tier
    });
    
    if (count >= limits.max) {
      return res.status(429).json({
        success: false,
        message: 'Rate limit exceeded for your tier',
        tier,
        limit: limits.max,
        upgrade: tier !== 'enterprise' ? 'Upgrade for higher limits' : null
      });
    }
    
    next();
  } catch (error) {
    console.error('User rate limiter error:', error);
    next();
  }
};

module.exports = userBasedLimiter;
```

---

## ขั้นตอนที่ 828: DDoS Protection

```javascript
// middleware/ddosProtection.js
const rateLimit = require('express-rate-limit');
const redis = require('../config/redis');

// รายการ IP ที่ถูก block
const BLOCKED_IPS_KEY = 'blocked:ips';

// Middleware ตรวจสอบ IP ที่ถูก block
const checkBlockedIP = async (req, res, next) => {
  const ip = req.ip;
  
  const isBlocked = await redis.sismember(BLOCKED_IPS_KEY, ip);
  
  if (isBlocked) {
    return res.status(403).json({
      success: false,
      message: 'Access denied'
    });
  }
  
  next();
};

// Auto-block IP ที่ส่ง request มากเกินไป
const autoBlockLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 นาที
  max: 1000,
  handler: async (req, res, next, options) => {
    const ip = req.ip;
    
    // Block IP เป็นเวลา 1 ชั่วโมง
    await redis.sadd(BLOCKED_IPS_KEY, ip);
    await redis.expire(BLOCKED_IPS_KEY, 3600);
    
    // บันทึก log
    console.warn(`Auto-blocked IP: ${ip} at ${new Date().toISOString()}`);
    
    res.status(429).json({
      success: false,
      message: 'Too many requests. IP has been temporarily blocked.'
    });
  }
});

// Slow down requests ก่อน block
const speedLimiter = require('express-slow-down');

const slowDownMiddleware = speedLimiter({
  windowMs: 15 * 60 * 1000,
  delayAfter: 100,    // เริ่มช้าลงหลังจาก 100 requests
  delayMs: () => 500, // เพิ่ม 500ms ต่อ request
  maxDelayMs: 20000   // ล่าช้าสูงสุด 20 วินาที
});

// ตรวจสอบ User Agent ที่น่าสงสัย
const suspiciousUserAgentCheck = (req, res, next) => {
  const userAgent = req.headers['user-agent'] || '';
  
  const suspiciousPatterns = [
    /bot/i,
    /crawler/i,
    /spider/i,
    /scraper/i,
    /curl/i,
    /wget/i,
    /python-requests/i
  ];
  
  // Whitelist บาง bots
  const allowedBots = [
    /Googlebot/i,
    /Bingbot/i,
    /facebookexternalhit/i
  ];
  
  const isSuspicious = suspiciousPatterns.some(pattern => pattern.test(userAgent));
  const isAllowedBot = allowedBots.some(pattern => pattern.test(userAgent));
  
  if (isSuspicious && !isAllowedBot) {
    // ใช้ rate limit เข้มงวดขึ้นสำหรับ bots
    req.isSuspiciousBot = true;
  }
  
  next();
};

module.exports = {
  checkBlockedIP,
  autoBlockLimiter,
  slowDownMiddleware,
  suspiciousUserAgentCheck
};
```

---

## ขั้นตอนที่ 829: Rate Limiting สำหรับ Authentication

```javascript
// middleware/authRateLimiter.js
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const redis = require('../config/redis');

// Progressive delay สำหรับ login attempts
class ProgressiveAuthLimiter {
  constructor() {
    this.maxAttempts = 5;
    this.lockoutDuration = 15 * 60; // 15 นาที (วินาที)
    this.keyPrefix = 'auth:attempts:';
  }

  async recordAttempt(identifier, success) {
    const key = `${this.keyPrefix}${identifier}`;
    
    if (success) {
      // ถ้า login สำเร็จ ลบ attempts
      await redis.del(key);
      return { locked: false, attempts: 0 };
    }
    
    // เพิ่ม attempt count
    const attempts = await redis.incr(key);
    
    if (attempts === 1) {
      // ตั้ง TTL เมื่อ attempt แรก
      await redis.expire(key, this.lockoutDuration);
    }
    
    if (attempts >= this.maxAttempts) {
      const ttl = await redis.ttl(key);
      return {
        locked: true,
        attempts,
        unlockAt: new Date(Date.now() + ttl * 1000)
      };
    }
    
    return {
      locked: false,
      attempts,
      remaining: this.maxAttempts - attempts
    };
  }

  async isLocked(identifier) {
    const key = `${this.keyPrefix}${identifier}`;
    const attempts = await redis.get(key);
    
    if (!attempts) return { locked: false };
    
    const count = parseInt(attempts);
    if (count >= this.maxAttempts) {
      const ttl = await redis.ttl(key);
      return {
        locked: true,
        attempts: count,
        unlockAt: new Date(Date.now() + ttl * 1000)
      };
    }
    
    return { locked: false, attempts: count };
  }
}

const authLimiter = new ProgressiveAuthLimiter();

// Middleware สำหรับ login
const loginRateLimiter = async (req, res, next) => {
  const identifier = req.body.email || req.ip;
  
  const status = await authLimiter.isLocked(identifier);
  
  if (status.locked) {
    return res.status(429).json({
      success: false,
      message: 'Account temporarily locked due to too many failed attempts',
      unlockAt: status.unlockAt,
      attempts: status.attempts
    });
  }
  
  // เก็บ limiter ใน req เพื่อใช้ใน route handler
  req.authLimiter = authLimiter;
  req.authIdentifier = identifier;
  
  next();
};

module.exports = { ProgressiveAuthLimiter, loginRateLimiter };
```

---

## ขั้นตอนที่ 830: Rate Limiting Headers

```javascript
// middleware/rateLimitHeaders.js

// เพิ่ม custom headers สำหรับ rate limit information
const addRateLimitInfo = (req, res, next) => {
  const originalJson = res.json.bind(res);
  
  res.json = function(data) {
    // เพิ่ม rate limit info ใน response body ถ้าต้องการ
    if (res.get('X-RateLimit-Limit')) {
      data._rateLimit = {
        limit: parseInt(res.get('X-RateLimit-Limit')),
        remaining: parseInt(res.get('X-RateLimit-Remaining')),
        reset: res.get('X-RateLimit-Reset')
      };
    }
    
    return originalJson(data);
  };
  
  next();
};

// Middleware สำหรับ log rate limit violations
const logRateLimitViolations = (req, res, next) => {
  const originalStatus = res.status.bind(res);
  
  res.status = function(code) {
    if (code === 429) {
      console.warn({
        timestamp: new Date().toISOString(),
        type: 'RATE_LIMIT_EXCEEDED',
        ip: req.ip,
        userAgent: req.headers['user-agent'],
        path: req.path,
        method: req.method,
        userId: req.user ? req.user.id : 'anonymous'
      });
    }
    
    return originalStatus(code);
  };
  
  next();
};

module.exports = { addRateLimitInfo, logRateLimitViolations };
```

---

## ขั้นตอนที่ 831: Whitelist และ Blacklist

```javascript
// middleware/ipFilter.js
const redis = require('../config/redis');

const WHITELIST_KEY = 'ip:whitelist';
const BLACKLIST_KEY = 'ip:blacklist';

// ตรวจสอบ IP whitelist/blacklist
const ipFilterMiddleware = async (req, res, next) => {
  const ip = req.ip;
  
  // ตรวจสอบ blacklist ก่อน
  const isBlacklisted = await redis.sismember(BLACKLIST_KEY, ip);
  if (isBlacklisted) {
    return res.status(403).json({
      success: false,
      message: 'Access denied'
    });
  }
  
  // ตรวจสอบ whitelist
  const isWhitelisted = await redis.sismember(WHITELIST_KEY, ip);
  req.isWhitelisted = isWhitelisted;
  
  next();
};

// Admin API สำหรับจัดการ whitelist/blacklist
class IPFilterService {
  async addToWhitelist(ip) {
    await redis.sadd(WHITELIST_KEY, ip);
    console.log(`Added ${ip} to whitelist`);
  }

  async removeFromWhitelist(ip) {
    await redis.srem(WHITELIST_KEY, ip);
  }

  async addToBlacklist(ip, ttl = null) {
    await redis.sadd(BLACKLIST_KEY, ip);
    if (ttl) {
      // Temporary blacklist
      await redis.expire(BLACKLIST_KEY, ttl);
    }
    console.log(`Added ${ip} to blacklist`);
  }

  async removeFromBlacklist(ip) {
    await redis.srem(BLACKLIST_KEY, ip);
  }

  async getWhitelist() {
    return redis.smembers(WHITELIST_KEY);
  }

  async getBlacklist() {
    return redis.smembers(BLACKLIST_KEY);
  }
}

module.exports = {
  ipFilterMiddleware,
  IPFilterService: new IPFilterService()
};
```

---

## ขั้นตอนที่ 832: Rate Limiting ด้วย Custom Store

```javascript
// stores/mongoRateLimitStore.js
const mongoose = require('mongoose');

// Schema สำหรับเก็บ rate limit data ใน MongoDB
const rateLimitSchema = new mongoose.Schema({
  key: { type: String, required: true, unique: true, index: true },
  count: { type: Number, default: 0 },
  resetTime: { type: Date, required: true }
}, {
  timestamps: true
});

// Auto-expire documents
rateLimitSchema.index({ resetTime: 1 }, { expireAfterSeconds: 0 });

const RateLimit = mongoose.model('RateLimit', rateLimitSchema);

// Custom Store สำหรับ express-rate-limit
class MongoDBStore {
  constructor(options = {}) {
    this.windowMs = options.windowMs || 60 * 1000;
    this.prefix = options.prefix || 'rl:';
  }

  async increment(key) {
    const now = new Date();
    const resetTime = new Date(now.getTime() + this.windowMs);
    
    const result = await RateLimit.findOneAndUpdate(
      { key: `${this.prefix}${key}`, resetTime: { $gt: now } },
      { $inc: { count: 1 }, $setOnInsert: { resetTime } },
      { upsert: true, new: true, setDefaultsOnInsert: true }
    );
    
    return {
      totalHits: result.count,
      resetTime: result.resetTime
    };
  }

  async decrement(key) {
    await RateLimit.findOneAndUpdate(
      { key: `${this.prefix}${key}` },
      { $inc: { count: -1 } }
    );
  }

  async resetKey(key) {
    await RateLimit.deleteOne({ key: `${this.prefix}${key}` });
  }
}

module.exports = MongoDBStore;
```

---

## ขั้นตอนที่ 833: Rate Limiting Analytics

```javascript
// services/rateLimitAnalytics.js
const redis = require('../config/redis');

class RateLimitAnalytics {
  constructor() {
    this.analyticsPrefix = 'analytics:rl:';
  }

  // บันทึกข้อมูลการ rate limit
  async record(event) {
    const {
      ip,
      path,
      method,
      userId,
      blocked,
      timestamp = new Date()
    } = event;

    const hour = new Date(timestamp);
    hour.setMinutes(0, 0, 0);
    const hourKey = hour.toISOString();

    const pipeline = redis.pipeline();

    // นับจำนวน requests ทั้งหมดต่อชั่วโมง
    pipeline.incr(`${this.analyticsPrefix}total:${hourKey}`);
    pipeline.expire(`${this.analyticsPrefix}total:${hourKey}`, 86400 * 7);

    if (blocked) {
      // นับจำนวน blocked requests
      pipeline.incr(`${this.analyticsPrefix}blocked:${hourKey}`);
      pipeline.expire(`${this.analyticsPrefix}blocked:${hourKey}`, 86400 * 7);

      // เก็บ top blocked IPs
      pipeline.zincrby(`${this.analyticsPrefix}blocked_ips`, 1, ip);
      pipeline.expire(`${this.analyticsPrefix}blocked_ips`, 86400);

      // เก็บ top blocked paths
      pipeline.zincrby(`${this.analyticsPrefix}blocked_paths`, 1, `${method}:${path}`);
      pipeline.expire(`${this.analyticsPrefix}blocked_paths`, 86400);
    }

    await pipeline.exec();
  }

  // ดึงสถิติ
  async getStats(hours = 24) {
    const stats = [];
    const now = new Date();

    for (let i = 0; i < hours; i++) {
      const hour = new Date(now - i * 3600 * 1000);
      hour.setMinutes(0, 0, 0);
      const hourKey = hour.toISOString();

      const [total, blocked] = await Promise.all([
        redis.get(`${this.analyticsPrefix}total:${hourKey}`),
        redis.get(`${this.analyticsPrefix}blocked:${hourKey}`)
      ]);

      stats.push({
        hour: hourKey,
        total: parseInt(total) || 0,
        blocked: parseInt(blocked) || 0
      });
    }

    const [topBlockedIPs, topBlockedPaths] = await Promise.all([
      redis.zrevrange(`${this.analyticsPrefix}blocked_ips`, 0, 9, 'WITHSCORES'),
      redis.zrevrange(`${this.analyticsPrefix}blocked_paths`, 0, 9, 'WITHSCORES')
    ]);

    return {
      hourly: stats.reverse(),
      topBlockedIPs: this.parseZrevrange(topBlockedIPs),
      topBlockedPaths: this.parseZrevrange(topBlockedPaths)
    };
  }

  parseZrevrange(data) {
    const result = [];
    for (let i = 0; i < data.length; i += 2) {
      result.push({ value: data[i], score: parseInt(data[i + 1]) });
    }
    return result;
  }
}

module.exports = new RateLimitAnalytics();
```

---

## ขั้นตอนที่ 834: การตั้งค่า Rate Limiting แบบ Dynamic

```javascript
// services/dynamicRateLimiter.js
const redis = require('../config/redis');

class DynamicRateLimiter {
  constructor() {
    this.configKey = 'config:rate_limits';
    this.defaultConfig = {
      global: { windowMs: 15 * 60 * 1000, max: 1000 },
      api: { windowMs: 15 * 60 * 1000, max: 100 },
      auth: { windowMs: 60 * 60 * 1000, max: 5 }
    };
  }

  // ดึง config ปัจจุบัน
  async getConfig() {
    const config = await redis.get(this.configKey);
    if (config) {
      return JSON.parse(config);
    }
    return this.defaultConfig;
  }

  // อัพเดท config
  async updateConfig(key, newConfig) {
    const currentConfig = await this.getConfig();
    currentConfig[key] = { ...currentConfig[key], ...newConfig };
    await redis.set(this.configKey, JSON.stringify(currentConfig));
    return currentConfig[key];
  }

  // สร้าง dynamic middleware
  createMiddleware(configKey) {
    return async (req, res, next) => {
      try {
        const config = await this.getConfig();
        const limits = config[configKey] || config.global;

        const identifier = req.user ? `user:${req.user.id}` : `ip:${req.ip}`;
        const key = `dynamic:${configKey}:${identifier}`;
        const windowStart = Date.now() - limits.windowMs;

        const pipeline = redis.pipeline();
        pipeline.zremrangebyscore(key, '-inf', windowStart);
        pipeline.zcard(key);
        pipeline.zadd(key, Date.now(), Date.now().toString());
        pipeline.pexpire(key, limits.windowMs);

        const results = await pipeline.exec();
        const count = results[1][1];

        res.set({
          'X-RateLimit-Limit': limits.max,
          'X-RateLimit-Remaining': Math.max(0, limits.max - count)
        });

        if (count >= limits.max) {
          return res.status(429).json({
            success: false,
            message: 'Rate limit exceeded',
            retryAfter: Math.ceil(limits.windowMs / 1000)
          });
        }

        next();
      } catch (error) {
        console.error('Dynamic rate limiter error:', error);
        next();
      }
    };
  }
}

module.exports = new DynamicRateLimiter();
```

---

## ขั้นตอนที่ 835: การทดสอบ Rate Limiting

```javascript
// tests/rateLimiter.test.js
const request = require('supertest');
const app = require('../app');
const redis = require('../config/redis');

describe('Rate Limiting Tests', () => {
  beforeEach(async () => {
    // ล้าง rate limit keys ก่อนทดสอบ
    const keys = await redis.keys('rl:*');
    if (keys.length > 0) {
      await redis.del(...keys);
    }
  });

  afterAll(async () => {
    await redis.quit();
  });

  describe('Basic Rate Limiting', () => {
    it('should allow requests within limit', async () => {
      const requests = Array(5).fill(null).map(() =>
        request(app).get('/api/test')
      );
      
      const responses = await Promise.all(requests);
      
      responses.forEach(res => {
        expect(res.status).not.toBe(429);
      });
    });

    it('should block requests exceeding limit', async () => {
      // ส่ง requests มากกว่า limit
      const limit = 5;
      const requests = Array(limit + 1).fill(null).map(() =>
        request(app).get('/api/test-limited')
      );
      
      const responses = await Promise.all(requests);
      
      // อย่างน้อยหนึ่ง request ควรถูก block
      const blockedResponses = responses.filter(res => res.status === 429);
      expect(blockedResponses.length).toBeGreaterThan(0);
    });

    it('should return correct rate limit headers', async () => {
      const res = await request(app).get('/api/test');
      
      expect(res.headers['x-ratelimit-limit']).toBeDefined();
      expect(res.headers['x-ratelimit-remaining']).toBeDefined();
      expect(parseInt(res.headers['x-ratelimit-remaining']))
        .toBeLessThanOrEqual(parseInt(res.headers['x-ratelimit-limit']));
    });
  });

  describe('Auth Rate Limiting', () => {
    it('should block after max failed login attempts', async () => {
      const attempts = Array(6).fill(null).map(() =>
        request(app)
          .post('/api/auth/login')
          .send({ email: 'test@example.com', password: 'wrongpassword' })
      );
      
      const responses = await Promise.all(attempts);
      const lastResponse = responses[responses.length - 1];
      
      expect(lastResponse.status).toBe(429);
    });
  });
});
```

---

## ขั้นตอนที่ 836: การใช้ Rate Limiting ใน Production

```javascript
// config/rateLimits.js
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const redis = require('./redis');

const createLimiter = (config) => {
  const store = new RedisStore({
    sendCommand: (...args) => redis.call(...args),
    prefix: `rl:${config.name}:`
  });

  return rateLimit({
    ...config,
    store,
    standardHeaders: true,
    legacyHeaders: false,
    skip: (req) => {
      // Skip สำหรับ health checks
      if (req.path === '/health') return true;
      // Skip สำหรับ whitelisted IPs
      if (req.isWhitelisted) return true;
      return false;
    }
  });
};

module.exports = {
  // Rate limiters ต่างๆ สำหรับ production
  global: createLimiter({
    name: 'global',
    windowMs: 15 * 60 * 1000,
    max: 1000
  }),
  
  api: createLimiter({
    name: 'api',
    windowMs: 15 * 60 * 1000,
    max: 100
  }),
  
  auth: createLimiter({
    name: 'auth',
    windowMs: 60 * 60 * 1000,
    max: 10
  }),
  
  upload: createLimiter({
    name: 'upload',
    windowMs: 60 * 60 * 1000,
    max: 20
  }),
  
  search: createLimiter({
    name: 'search',
    windowMs: 60 * 1000,
    max: 30
  })
};
```

---

## ขั้นตอนที่ 837: Monitoring Rate Limits

```javascript
// routes/admin/rateLimits.js
const express = require('express');
const router = express.Router();
const redis = require('../../config/redis');
const analytics = require('../../services/rateLimitAnalytics');
const { IPFilterService } = require('../../middleware/ipFilter');
const { requireAdmin } = require('../../middleware/auth');

router.use(requireAdmin);

// ดู stats ของ rate limiting
router.get('/stats', async (req, res) => {
  try {
    const hours = parseInt(req.query.hours) || 24;
    const stats = await analytics.getStats(hours);
    
    res.json({
      success: true,
      data: stats
    });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// จัดการ IP whitelist
router.post('/whitelist', async (req, res) => {
  const { ip } = req.body;
  await IPFilterService.addToWhitelist(ip);
  res.json({ success: true, message: `${ip} added to whitelist` });
});

router.delete('/whitelist/:ip', async (req, res) => {
  await IPFilterService.removeFromWhitelist(req.params.ip);
  res.json({ success: true, message: `${req.params.ip} removed from whitelist` });
});

// จัดการ IP blacklist
router.post('/blacklist', async (req, res) => {
  const { ip, duration } = req.body;
  await IPFilterService.addToBlacklist(ip, duration);
  res.json({ success: true, message: `${ip} added to blacklist` });
});

router.delete('/blacklist/:ip', async (req, res) => {
  await IPFilterService.removeFromBlacklist(req.params.ip);
  res.json({ success: true, message: `${req.params.ip} removed from blacklist` });
});

// Reset rate limit สำหรับ user/IP
router.post('/reset', async (req, res) => {
  const { key } = req.body;
  const keys = await redis.keys(`rl:*${key}*`);
  
  if (keys.length > 0) {
    await redis.del(...keys);
  }
  
  res.json({ 
    success: true, 
    message: `Reset rate limits for ${key}`,
    keysReset: keys.length
  });
});

module.exports = router;
```

---

## ขั้นตอนที่ 838: Express App รวมทุกอย่าง

```javascript
// app.js - Complete rate limiting setup
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const { checkBlockedIP, suspiciousUserAgentCheck } = require('./middleware/ddosProtection');
const { ipFilterMiddleware } = require('./middleware/ipFilter');
const rateLimits = require('./config/rateLimits');
const { logRateLimitViolations } = require('./middleware/rateLimitHeaders');

const app = express();

// Security middleware
app.use(helmet());
app.use(cors());
app.use(express.json());

// IP filtering
app.use(ipFilterMiddleware);
app.use(checkBlockedIP);
app.use(suspiciousUserAgentCheck);

// Logging
app.use(logRateLimitViolations);

// Global rate limiting
app.use(rateLimits.global);

// API routes
app.use('/api', rateLimits.api);
app.use('/api/auth', rateLimits.auth);
app.use('/api/upload', rateLimits.upload);
app.use('/api/search', rateLimits.search);

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/products', require('./routes/products'));
app.use('/admin/rate-limits', require('./routes/admin/rateLimits'));

// Health check (ไม่มี rate limiting)
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Error handling
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ 
    success: false, 
    message: 'Internal server error' 
  });
});

module.exports = app;
```

---

## ขั้นตอนที่ 839: Best Practices สำหรับ Rate Limiting

### หลักการสำคัญ:

**1. กำหนด Rate Limits ตาม Business Logic**
```javascript
// ตัวอย่างการกำหนด limits ที่เหมาะสม
const RATE_LIMITS = {
  // Public API endpoints
  'GET /api/products': { windowMs: '15m', max: 100 },
  'POST /api/search': { windowMs: '1m', max: 10 },
  
  // Auth endpoints (เข้มงวดมาก)
  'POST /api/auth/login': { windowMs: '1h', max: 5 },
  'POST /api/auth/register': { windowMs: '1d', max: 3 },
  
  // User actions
  'POST /api/orders': { windowMs: '1h', max: 10 },
  'POST /api/reviews': { windowMs: '1d', max: 5 },
  
  // Upload endpoints
  'POST /api/upload': { windowMs: '1h', max: 20 }
};
```

**2. Return ข้อมูลที่เป็นประโยชน์เมื่อ Rate Limited**
```javascript
// ข้อมูลที่ควรส่งกลับเมื่อ rate limited
const rateLimitResponse = {
  success: false,
  error: {
    code: 'RATE_LIMIT_EXCEEDED',
    message: 'Too many requests',
    details: {
      limit: 100,
      remaining: 0,
      reset: '2024-01-01T12:00:00Z',
      retryAfter: 900 // วินาที
    }
  }
};
```

**3. ใช้ Fail-Open Strategy สำหรับ Rate Limiter ที่สำคัญ**
```javascript
const safeRateLimiter = async (req, res, next) => {
  try {
    // ทำ rate limiting ปกติ
    await checkRateLimit(req);
    next();
  } catch (error) {
    // ถ้า rate limiter เสีย ให้ผ่านไปได้ (availability > security)
    console.error('Rate limiter failed:', error);
    next();
  }
};
```

---

## ขั้นตอนที่ 840: การ Deploy Rate Limiting ใน Cluster

```javascript
// เมื่อใช้หลาย Node.js processes (cluster mode)
// ต้องใช้ shared store เช่น Redis

// ecosystem.config.js (PM2)
module.exports = {
  apps: [{
    name: 'api-server',
    script: 'app.js',
    instances: 'max',      // ใช้ CPU ทุก core
    exec_mode: 'cluster',  // Cluster mode
    env: {
      NODE_ENV: 'production',
      REDIS_HOST: 'redis-server',
      REDIS_PORT: 6379
    }
  }]
};

// ตรวจสอบว่าใช้ Redis store เสมอใน production
const isProduction = process.env.NODE_ENV === 'production';

const createStore = () => {
  if (isProduction) {
    // ใช้ Redis ใน production (shared across all instances)
    return new RedisStore({
      sendCommand: (...args) => redis.call(...args)
    });
  }
  // ใช้ Memory store ใน development (ไม่ shared)
  return undefined; // default: in-memory
};

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  store: createStore()
});
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: สร้าง Rate Limiter แบบ Tiered
สร้าง rate limiter ที่รองรับ 4 ระดับ: free (100/hour), basic (500/hour), premium (2000/hour), enterprise (unlimited)

### แบบฝึกหัดที่ 2: Implement Token Bucket ด้วย Redis
สร้าง Token Bucket algorithm โดยใช้ Redis Lua script เพื่อความ atomic

### แบบฝึกหัดที่ 3: Rate Limit Dashboard
สร้าง real-time dashboard แสดงสถิติ rate limiting พร้อม charts

### แบบฝึกหัดที่ 4: Geo-based Rate Limiting
สร้าง rate limiter ที่แตกต่างกันตามประเทศหรือภูมิภาคของ IP

### แบบฝึกหัดที่ 5: Integration Test
เขียน integration tests ครอบคลุมสถานการณ์ rate limiting ทั้งหมด รวมถึง:
- Normal flow
- Limit exceeded
- Recovery after window reset
- Whitelist bypass
- Blacklist blocking

---

## สรุป

Rate Limiting เป็นส่วนสำคัญของการป้องกัน API โดยเฉพาะใน production environment ควรใช้ Redis-based store เมื่อมีหลาย instances และกำหนด limits ให้เหมาะสมกับ business requirements ของแต่ละ endpoint
