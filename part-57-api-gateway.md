# Part 57: API Gateway (เกตเวย์ API)
## ขั้นตอนที่ 57-57 จาก 1000

---

## บทนำ

API Gateway ทำหน้าที่เป็น single entry point สำหรับ client requests ก่อนส่งต่อไปยัง backend services โดยรวมฟีเจอร์อย่าง authentication, rate limiting, logging ไว้ที่จุดเดียว

---

## 57.1 API Gateway Concepts

### บทบาทของ API Gateway

```
Client Request
      ↓
┌─────────────────────────────────────┐
│            API Gateway              │
│  ✓ Authentication                   │
│  ✓ Authorization                    │
│  ✓ Rate Limiting                    │
│  ✓ Request Logging                  │
│  ✓ SSL Termination                  │
│  ✓ Request Transformation           │
│  ✓ Load Balancing                   │
│  ✓ Circuit Breaking                 │
└─────────────────────────────────────┘
      ↓          ↓          ↓
 User Service  Product   Order
              Service   Service
```

---

## 57.2 Kong API Gateway

### ติดตั้ง Kong ด้วย Docker

```yaml
# docker-compose.kong.yml
version: '3'

services:
  kong-database:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: kong
      POSTGRES_USER: kong
      POSTGRES_PASSWORD: kong
    volumes:
      - kong-db:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "kong"]
      interval: 10s
    networks:
      - kong-net

  kong-migration:
    image: kong:3.4
    command: kong migrations bootstrap
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_DATABASE: kong
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kong
    depends_on:
      kong-database:
        condition: service_healthy
    networks:
      - kong-net

  kong:
    image: kong:3.4
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_DATABASE: kong
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kong
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
    ports:
      - "8000:8000"   # HTTP proxy
      - "8443:8443"   # HTTPS proxy
      - "8001:8001"   # Admin API
      - "8444:8444"   # Admin API HTTPS
    depends_on:
      - kong-migration
    healthcheck:
      test: ["CMD", "kong", "health"]
      interval: 10s
    networks:
      - kong-net

volumes:
  kong-db:

networks:
  kong-net:
    driver: bridge
```

### ตั้งค่า Kong ผ่าน Admin API

```bash
# สร้าง Service
curl -i -s -X POST http://localhost:8001/services \
  --data name=user-service \
  --data url=http://user-service:3001

# สร้าง Route
curl -i -X POST http://localhost:8001/services/user-service/routes \
  --data paths[]=/api/users \
  --data methods[]=GET \
  --data methods[]=POST

# เพิ่ม Rate Limiting Plugin
curl -X POST http://localhost:8001/services/user-service/plugins \
  --data name=rate-limiting \
  --data config.minute=60 \
  --data config.hour=1000 \
  --data config.policy=local

# เพิ่ม JWT Authentication Plugin
curl -X POST http://localhost:8001/plugins \
  --data name=jwt \
  --data config.secret_is_base64=false
```

---

## 57.3 Custom API Gateway ด้วย Express

```javascript
// api-gateway/src/app.js
const express = require('express');
const httpProxy = require('http-proxy-middleware');
const rateLimit = require('express-rate-limit');
const jwt = require('jsonwebtoken');
const morgan = require('morgan');
const redis = require('ioredis');

const app = express();
const redisClient = new redis(process.env.REDIS_URL);

// ============================================
// Services Registry
// ============================================
const services = {
  '/api/users': {
    url: process.env.USER_SERVICE_URL || 'http://user-service:3001',
    requiresAuth: false,  // Public endpoint
    rateLimit: { windowMs: 15 * 60 * 1000, max: 100 }
  },
  '/api/products': {
    url: process.env.PRODUCT_SERVICE_URL || 'http://product-service:3002',
    requiresAuth: false,
    rateLimit: { windowMs: 15 * 60 * 1000, max: 200 }
  },
  '/api/orders': {
    url: process.env.ORDER_SERVICE_URL || 'http://order-service:3003',
    requiresAuth: true,
    rateLimit: { windowMs: 15 * 60 * 1000, max: 50 }
  },
  '/api/payments': {
    url: process.env.PAYMENT_SERVICE_URL || 'http://payment-service:3004',
    requiresAuth: true,
    roles: ['user', 'admin'],
    rateLimit: { windowMs: 60 * 1000, max: 10 }
  }
};

// ============================================
// Middleware
// ============================================

// Logging
app.use(morgan('combined'));

// Request ID
app.use((req, res, next) => {
  req.requestId = require('crypto').randomUUID();
  res.setHeader('X-Request-Id', req.requestId);
  next();
});

// ============================================
// Authentication
// ============================================
const authenticate = async (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({
      error: 'Unauthorized',
      message: 'Authentication token required'
    });
  }
  
  // Check token blacklist (logout)
  const isBlacklisted = await redisClient.get(`blacklist:${token}`);
  if (isBlacklisted) {
    return res.status(401).json({ error: 'Token revoked' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    
    // ส่ง user info ไปยัง downstream service
    req.headers['X-User-Id'] = decoded.id;
    req.headers['X-User-Role'] = decoded.role;
    req.headers['X-User-Email'] = decoded.email;
    
    next();
  } catch (error) {
    return res.status(401).json({
      error: 'Unauthorized',
      message: 'Invalid or expired token'
    });
  }
};

// ============================================
// Request Routing
// ============================================

// Setup routes dynamically
Object.entries(services).forEach(([path, config]) => {
  const middlewares = [];
  
  // Rate limiting
  if (config.rateLimit) {
    middlewares.push(rateLimit({
      ...config.rateLimit,
      handler: (req, res) => {
        res.status(429).json({
          error: 'Too Many Requests',
          retryAfter: Math.ceil(config.rateLimit.windowMs / 1000)
        });
      }
    }));
  }
  
  // Authentication
  if (config.requiresAuth) {
    middlewares.push(authenticate);
  }
  
  // Role-based access
  if (config.roles) {
    middlewares.push((req, res, next) => {
      if (!config.roles.includes(req.user?.role)) {
        return res.status(403).json({
          error: 'Forbidden',
          message: 'Insufficient permissions'
        });
      }
      next();
    });
  }
  
  // Proxy to upstream service
  middlewares.push(httpProxy.createProxyMiddleware({
    target: config.url,
    changeOrigin: true,
    timeout: 30000,
    
    on: {
      proxyReq: (proxyReq, req) => {
        proxyReq.setHeader('X-Request-Id', req.requestId);
        proxyReq.setHeader('X-Gateway-Version', '1.0');
      },
      
      error: (err, req, res) => {
        console.error(`Proxy error for ${path}:`, err.message);
        
        if (!res.headersSent) {
          res.status(503).json({
            error: 'Service Unavailable',
            message: `Upstream service error: ${err.message}`
          });
        }
      }
    }
  }));
  
  app.use(path, ...middlewares);
});

// ============================================
// Health Check
// ============================================
app.get('/health', async (req, res) => {
  const serviceChecks = await Promise.allSettled(
    Object.entries(services).map(async ([path, config]) => {
      const controller = new AbortController();
      const timeout = setTimeout(() => controller.abort(), 3000);
      
      try {
        const response = await fetch(`${config.url}/health`, {
          signal: controller.signal
        });
        clearTimeout(timeout);
        return { path, status: response.ok ? 'up' : 'degraded' };
      } catch {
        clearTimeout(timeout);
        return { path, status: 'down' };
      }
    })
  );
  
  const results = serviceChecks.reduce((acc, result) => {
    if (result.status === 'fulfilled') {
      acc[result.value.path] = result.value.status;
    }
    return acc;
  }, {});
  
  const overall = Object.values(results).every(s => s === 'up')
    ? 'healthy'
    : Object.values(results).some(s => s === 'up')
      ? 'degraded'
      : 'unhealthy';
  
  res.status(overall === 'unhealthy' ? 503 : 200).json({
    status: overall,
    services: results,
    timestamp: new Date().toISOString()
  });
});

// ============================================
// Error Handler
// ============================================
app.use((err, req, res, next) => {
  console.error('Gateway error:', err);
  res.status(500).json({ error: 'Internal Gateway Error' });
});

module.exports = app;
```

---

## 57.4 Nginx API Gateway

```nginx
# nginx/api-gateway.conf

upstream user_service {
    least_conn;
    server user-service:3001 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

upstream product_service {
    least_conn;
    server product-service:3002 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

upstream order_service {
    server order-service:3003;
}

# Rate limiting zones
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;
limit_req_zone $binary_remote_addr zone=auth_limit:10m rate=10r/m;

server {
    listen 443 ssl http2;
    server_name api.example.com;
    
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    
    # Request ID
    add_header X-Request-Id $request_id;
    
    # Users API
    location /api/users {
        limit_req zone=api_limit burst=20 nodelay;
        
        proxy_pass http://user_service;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Request-Id $request_id;
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }
    
    # Products API (with caching)
    location /api/products {
        limit_req zone=api_limit burst=50 nodelay;
        
        # Cache GET requests
        proxy_cache product_cache;
        proxy_cache_valid 200 5m;
        proxy_cache_key $uri$is_args$args;
        add_header X-Cache-Status $upstream_cache_status;
        
        proxy_pass http://product_service;
        proxy_set_header Host $host;
    }
    
    # Orders API (requires auth)
    location /api/orders {
        limit_req zone=api_limit burst=10 nodelay;
        
        # JWT validation ผ่าน sub-request
        auth_request /auth/validate;
        auth_request_set $auth_user_id $upstream_http_x_user_id;
        
        proxy_set_header X-User-Id $auth_user_id;
        proxy_pass http://order_service;
    }
    
    # Auth validation endpoint
    location = /auth/validate {
        internal;
        proxy_pass http://user_service/api/auth/validate;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URI $request_uri;
    }
}

# Cache configuration
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=product_cache:10m max_size=100m inactive=60m;
```

---

## 57.5 Rate Limiting ที่ Gateway

```javascript
// api-gateway/src/middleware/rateLimiter.js
const redis = require('ioredis');
const client = new redis(process.env.REDIS_URL);

/**
 * Sliding window rate limiter ด้วย Redis
 */
const slidingWindowRateLimiter = (options = {}) => {
  const {
    windowMs = 60000,   // 1 นาที
    maxRequests = 100,
    keyFn = (req) => req.ip,
    message = 'Too many requests'
  } = options;
  
  return async (req, res, next) => {
    const key = `ratelimit:${keyFn(req)}`;
    const now = Date.now();
    const windowStart = now - windowMs;
    
    try {
      const pipeline = client.pipeline();
      
      // ลบ requests เก่า
      pipeline.zremrangebyscore(key, '-inf', windowStart);
      
      // นับ requests ในช่วงเวลา
      pipeline.zcard(key);
      
      // เพิ่ม request ปัจจุบัน
      pipeline.zadd(key, now, `${now}-${Math.random()}`);
      
      // ตั้ง expiry
      pipeline.pexpire(key, windowMs);
      
      const results = await pipeline.exec();
      const requestCount = results[1][1];
      
      res.setHeader('X-RateLimit-Limit', maxRequests);
      res.setHeader('X-RateLimit-Remaining', Math.max(0, maxRequests - requestCount));
      res.setHeader('X-RateLimit-Reset', Math.ceil((now + windowMs) / 1000));
      
      if (requestCount >= maxRequests) {
        return res.status(429).json({
          error: 'Too Many Requests',
          message,
          retryAfter: Math.ceil(windowMs / 1000)
        });
      }
      
      next();
    } catch (error) {
      // ถ้า Redis ล้มเหลว ให้ผ่านไปก่อน
      console.error('Rate limiter error:', error);
      next();
    }
  };
};

module.exports = { slidingWindowRateLimiter };
```

---

## แบบฝึกหัดที่ 57

### แบบฝึกหัดพื้นฐาน

**1. Custom Express Gateway**

สร้าง API Gateway ด้วย Express:
- Route ไปยัง 3 microservices
- JWT authentication
- Rate limiting per endpoint

**2. Kong Setup**

ตั้งค่า Kong:
- Register services และ routes
- เพิ่ม rate limiting plugin
- เพิ่ม JWT auth plugin

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **API Gateway** - role, features
2. **Kong** - production-grade gateway
3. **Custom Gateway** - Express implementation
4. **Nginx** - as reverse proxy/gateway
5. **Rate Limiting** - sliding window algorithm

**ถัดไป:** Part 58 - Message Queue
