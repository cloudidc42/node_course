# Part 51: Microservices Architecture (สถาปัตยกรรม Microservices)
## ขั้นตอนที่ 51-51 จาก 1000

---

## บทนำ

Microservices เป็น architectural pattern ที่แบ่งแอปพลิเคชันออกเป็น services ขนาดเล็ก ๆ ที่ทำงานอิสระ สื่อสารกันผ่าน network

---

## 51.1 Microservices vs Monolith

### Monolith Architecture

```
┌─────────────────────────────────────┐
│           Monolith App              │
│  ┌─────────┐ ┌──────────┐          │
│  │  Users  │ │ Products │          │
│  └─────────┘ └──────────┘          │
│  ┌─────────┐ ┌──────────┐          │
│  │ Orders  │ │Payments  │          │
│  └─────────┘ └──────────┘          │
│  ┌─────────────────────────┐       │
│  │     Single Database     │       │
│  └─────────────────────────┘       │
└─────────────────────────────────────┘
```

**ข้อดี Monolith:**
- ง่ายในการพัฒนาและ test
- Deploy ง่าย
- Performance ดี (in-process calls)
- ไม่มีปัญหา network

**ข้อเสีย Monolith:**
- Scale ทั้งแอปพร้อมกัน (ไม่สามารถ scale บางส่วนได้)
- Deploy ทั้งแอปทุกครั้ง
- Technology lock-in
- Team ใหญ่ทำงานใน codebase เดียวกัน → conflicts

### Microservices Architecture

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  Users   │  │ Products │  │ Orders   │  │Payments  │
│ Service  │  │ Service  │  │ Service  │  │ Service  │
│          │  │          │  │          │  │          │
│  DB/SQL  │  │  DB/SQL  │  │  DB/SQL  │  │  DB/SQL  │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
     │              │              │              │
     └──────────────┴──────────────┴──────────────┘
                         │
                   ┌─────┴─────┐
                   │ API Gateway│
                   └─────┬─────┘
                         │
                      Clients
```

---

## 51.2 Service Decomposition (การแบ่ง Services)

### Domain-Driven Design (DDD)

```javascript
// การแบ่ง e-commerce เป็น bounded contexts

/*
User Context:
- User registration/login
- Profile management
- Authentication/Authorization

Product Context:
- Product catalog
- Inventory
- Categories

Order Context:
- Order management
- Order status
- Order history

Payment Context:
- Payment processing
- Refunds
- Payment history

Notification Context:
- Email notifications
- SMS
- Push notifications
*/
```

### โครงสร้างแต่ละ Service

```
services/
├── user-service/
│   ├── src/
│   │   ├── controllers/
│   │   ├── models/
│   │   ├── routes/
│   │   └── app.js
│   ├── Dockerfile
│   └── package.json
│
├── product-service/
│   ├── src/
│   ├── Dockerfile
│   └── package.json
│
├── order-service/
│   ├── src/
│   ├── Dockerfile
│   └── package.json
│
├── api-gateway/
│   ├── src/
│   ├── Dockerfile
│   └── package.json
│
└── docker-compose.yml
```

---

## 51.3 Communication Patterns

### Synchronous Communication (HTTP/REST)

```javascript
// order-service/src/services/userServiceClient.js
const axios = require('axios');

const USER_SERVICE_URL = process.env.USER_SERVICE_URL || 'http://user-service:3001';

/**
 * เรียก User Service เพื่อดึงข้อมูล user
 */
const getUserById = async (userId, authToken) => {
  try {
    const response = await axios.get(`${USER_SERVICE_URL}/api/users/${userId}`, {
      headers: {
        'Authorization': authToken,
        'X-Internal-Service': 'order-service'
      },
      timeout: 5000
    });
    
    return response.data;
  } catch (error) {
    if (error.response?.status === 404) {
      throw new Error(`User ${userId} not found`);
    }
    throw new Error(`User service unavailable: ${error.message}`);
  }
};

/**
 * Circuit Breaker Pattern
 */
const CircuitBreaker = require('opossum');  // npm install opossum

const getUserCircuitBreaker = new CircuitBreaker(getUserById, {
  timeout: 3000,      // timeout หลัง 3 วินาที
  errorThresholdPercentage: 50,  // เปิด circuit ถ้า 50% fail
  resetTimeout: 30000  // ลองใหม่หลัง 30 วินาที
});

getUserCircuitBreaker.fallback((userId) => ({
  id: userId,
  name: 'Unknown User',
  email: 'unknown@example.com'
}));

module.exports = { getUserById: (id, token) => getUserCircuitBreaker.fire(id, token) };
```

### Asynchronous Communication (Message Queue)

```javascript
// order-service/src/services/orderEventPublisher.js
const amqp = require('amqplib');

let channel;

const connectToRabbitMQ = async () => {
  const connection = await amqp.connect(process.env.RABBITMQ_URL);
  channel = await connection.createChannel();
  
  // สร้าง exchange
  await channel.assertExchange('order-events', 'topic', { durable: true });
  
  console.log('✅ Connected to RabbitMQ');
};

/**
 * Publish event เมื่อ order ถูกสร้าง
 */
const publishOrderCreated = async (order) => {
  const event = {
    eventType: 'order.created',
    data: {
      orderId: order.id,
      userId: order.userId,
      items: order.items,
      total: order.total
    },
    timestamp: new Date().toISOString(),
    source: 'order-service'
  };
  
  channel.publish(
    'order-events',
    'order.created',
    Buffer.from(JSON.stringify(event)),
    { persistent: true }
  );
  
  console.log(`📤 Published order.created event for order ${order.id}`);
};

module.exports = { connectToRabbitMQ, publishOrderCreated };
```

```javascript
// notification-service/src/consumers/orderEventConsumer.js
const amqp = require('amqplib');

const consumeOrderEvents = async () => {
  const connection = await amqp.connect(process.env.RABBITMQ_URL);
  const channel = await connection.createChannel();
  
  await channel.assertExchange('order-events', 'topic', { durable: true });
  
  const { queue } = await channel.assertQueue('notification-service.orders', {
    durable: true
  });
  
  // Subscribe to order events
  await channel.bindQueue(queue, 'order-events', 'order.*');
  
  channel.prefetch(10);
  
  channel.consume(queue, async (msg) => {
    if (!msg) return;
    
    try {
      const event = JSON.parse(msg.content.toString());
      console.log(`📨 Received event: ${event.eventType}`);
      
      await handleOrderEvent(event);
      channel.ack(msg);
    } catch (error) {
      console.error('Error processing event:', error);
      channel.nack(msg, false, true);  // requeue
    }
  });
};

const handleOrderEvent = async (event) => {
  switch (event.eventType) {
    case 'order.created':
      await sendOrderConfirmationEmail(event.data);
      break;
    case 'order.cancelled':
      await sendCancellationEmail(event.data);
      break;
  }
};

module.exports = { consumeOrderEvents };
```

---

## 51.4 API Gateway

```javascript
// api-gateway/src/app.js
const express = require('express');
const httpProxy = require('http-proxy-middleware');
const rateLimit = require('express-rate-limit');
const jwt = require('jsonwebtoken');

const app = express();

// Services configuration
const services = {
  users: process.env.USER_SERVICE_URL || 'http://user-service:3001',
  products: process.env.PRODUCT_SERVICE_URL || 'http://product-service:3002',
  orders: process.env.ORDER_SERVICE_URL || 'http://order-service:3003',
  payments: process.env.PAYMENT_SERVICE_URL || 'http://payment-service:3004'
};

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 นาที
  max: 100
});
app.use(limiter);

// Authentication middleware
const authenticate = (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

// Proxy routes
app.use('/api/users', 
  httpProxy.createProxyMiddleware({
    target: services.users,
    changeOrigin: true,
    pathRewrite: { '^/api/users': '/api/users' }
  })
);

app.use('/api/products',
  httpProxy.createProxyMiddleware({
    target: services.products,
    changeOrigin: true
  })
);

app.use('/api/orders',
  authenticate,
  httpProxy.createProxyMiddleware({
    target: services.orders,
    changeOrigin: true,
    on: {
      proxyReq: (proxyReq, req) => {
        // ส่ง user info ไปยัง downstream service
        proxyReq.setHeader('X-User-Id', req.user.id);
        proxyReq.setHeader('X-User-Role', req.user.role);
      }
    }
  })
);

// Health check
app.get('/health', async (req, res) => {
  const checks = await Promise.allSettled(
    Object.entries(services).map(async ([name, url]) => {
      const response = await fetch(`${url}/health`);
      return { name, status: response.ok ? 'up' : 'down' };
    })
  );
  
  const results = checks.reduce((acc, result) => {
    if (result.status === 'fulfilled') {
      acc[result.value.name] = result.value.status;
    }
    return acc;
  }, {});
  
  const allUp = Object.values(results).every(s => s === 'up');
  
  res.status(allUp ? 200 : 503).json({
    status: allUp ? 'healthy' : 'degraded',
    services: results
  });
});

module.exports = app;
```

---

## 51.5 Service Discovery

```javascript
// src/serviceRegistry.js

/**
 * Simple Service Registry
 */
class ServiceRegistry {
  constructor() {
    this.services = new Map();
    this.healthCheckInterval = null;
  }
  
  register(name, url, metadata = {}) {
    if (!this.services.has(name)) {
      this.services.set(name, []);
    }
    
    const instances = this.services.get(name);
    const existing = instances.find(i => i.url === url);
    
    if (!existing) {
      instances.push({
        url,
        metadata,
        registeredAt: new Date(),
        lastHeartbeat: new Date(),
        healthy: true
      });
    } else {
      existing.lastHeartbeat = new Date();
    }
    
    console.log(`📝 Service registered: ${name} at ${url}`);
  }
  
  deregister(name, url) {
    if (this.services.has(name)) {
      const instances = this.services.get(name);
      const index = instances.findIndex(i => i.url === url);
      if (index !== -1) {
        instances.splice(index, 1);
        console.log(`❌ Service deregistered: ${name} at ${url}`);
      }
    }
  }
  
  discover(name) {
    const instances = this.services.get(name) || [];
    const healthy = instances.filter(i => i.healthy);
    
    if (healthy.length === 0) {
      throw new Error(`No healthy instances of ${name}`);
    }
    
    // Round-robin load balancing
    const instance = healthy[Math.floor(Math.random() * healthy.length)];
    return instance.url;
  }
  
  startHealthChecks(intervalMs = 30000) {
    this.healthCheckInterval = setInterval(async () => {
      for (const [name, instances] of this.services.entries()) {
        for (const instance of instances) {
          try {
            const response = await fetch(`${instance.url}/health`, {
              signal: AbortSignal.timeout(5000)
            });
            instance.healthy = response.ok;
          } catch {
            instance.healthy = false;
          }
        }
      }
    }, intervalMs);
  }
}

const registry = new ServiceRegistry();
module.exports = registry;
```

---

## แบบฝึกหัดที่ 51

### แบบฝึกหัดพื้นฐาน

**1. สร้าง 2 Microservices**

สร้าง User Service และ Product Service:
- แต่ละ service มี database แยกกัน
- Communication ผ่าน HTTP
- Health check endpoint

**2. API Gateway**

สร้าง API Gateway ที่:
- Route requests ไปยัง services
- Authentication centralized
- Rate limiting

### แบบฝึกหัดขั้นสูง

**3. Event-Driven Architecture**

สร้าง 3 services ที่สื่อสารผ่าน RabbitMQ:
- Order Service publishes events
- Inventory Service subscribes
- Notification Service subscribes

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Monolith vs Microservices** - trade-offs
2. **Service Decomposition** - DDD, bounded contexts
3. **Communication** - sync (HTTP), async (MQ)
4. **API Gateway** - routing, auth, rate limiting
5. **Service Discovery** - registry, health checks

**ถัดไป:** Part 52 - Docker
