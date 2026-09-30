# Part 39 | ขั้นตอนที่ 681-700 จาก 1000

## Microservices Architecture - สถาปัตยกรรม Microservices

---

## สารบัญ

1. [Microservices คืออะไร](#microservices-คืออะไร)
2. [Service Design Patterns](#service-design-patterns)
3. [API Gateway](#api-gateway)
4. [Service Discovery](#service-discovery)
5. [Inter-Service Communication](#inter-service-communication)
6. [Database per Service](#database-per-service)
7. [Distributed Tracing](#distributed-tracing)
8. [Resilience Patterns](#resilience-patterns)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Microservices คืออะไร

### ขั้นตอนที่ 681: Monolith vs Microservices

```
Monolith:
  ┌─────────────────────────────────┐
  │         Single Application      │
  │  ┌─────┐ ┌─────┐ ┌──────────┐  │
  │  │Users│ │Posts│ │Payments  │  │
  │  └─────┘ └─────┘ └──────────┘  │
  │  ┌──────────────────────────┐   │
  │  │      Single Database     │   │
  │  └──────────────────────────┘   │
  └─────────────────────────────────┘

Microservices:
  ┌──────────┐  ┌──────────┐  ┌──────────────┐
  │  User    │  │  Post    │  │   Payment    │
  │ Service  │  │ Service  │  │   Service    │
  │ ┌──────┐ │  │ ┌──────┐ │  │  ┌────────┐ │
  │ │  DB  │ │  │ │  DB  │ │  │  │   DB   │ │
  │ └──────┘ │  │ └──────┘ │  │  └────────┘ │
  └──────────┘  └──────────┘  └──────────────┘
        ↑              ↑              ↑
        └──────────────┴──────────────┘
                   API Gateway
                       ↑
                    Client
```

### ขั้นตอนที่ 682: ข้อดีและข้อเสีย

```
ข้อดี:
  ✅ Scale แยกกันได้ตาม service
  ✅ Deploy แยกกันได้
  ✅ Technology stack ต่างกันได้
  ✅ Fault isolation - ถ้า service ล้ม ไม่กระทบทั้งระบบ
  ✅ Team autonomy - แต่ละทีมรับผิดชอบ service ของตัวเอง

ข้อเสีย:
  ❌ ซับซ้อนกว่า monolith
  ❌ Network latency ระหว่าง services
  ❌ Distributed transactions ยาก
  ❌ ต้องการ DevOps expertise
  ❌ Debugging ยากกว่า
```

---

## Service Design Patterns

### ขั้นตอนที่ 683: โครงสร้าง Microservices Project

```
microservices/
├── api-gateway/           ← Entry point
│   ├── src/
│   │   ├── routes/
│   │   ├── middleware/
│   │   └── index.js
│   └── package.json
├── user-service/          ← User management
│   ├── src/
│   │   ├── models/
│   │   ├── controllers/
│   │   └── index.js
│   └── package.json
├── post-service/          ← Content management
├── notification-service/  ← Notifications
├── payment-service/       ← Payments
├── docker-compose.yml
└── README.md
```

### ขั้นตอนที่ 684: User Service

```javascript
// user-service/src/index.js
const express = require('express');
const mongoose = require('mongoose');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');

const app = express();
app.use(express.json());

// Service info
const SERVICE = {
  name: 'user-service',
  version: '1.0.0',
  port: process.env.PORT || 3001,
};

// Health check endpoint (สำคัญมากสำหรับ microservices)
app.get('/health', (req, res) => {
  res.json({
    service: SERVICE.name,
    status: 'healthy',
    version: SERVICE.version,
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
  });
});

// Routes
app.post('/users/register', async (req, res) => {
  try {
    const { username, email, password } = req.body;

    const existingUser = await User.findOne({ $or: [{ email }, { username }] });
    if (existingUser) {
      return res.status(409).json({ error: 'User already exists' });
    }

    const user = await User.create({ username, email, password });

    // Publish event (ส่งให้ services อื่นรู้)
    await publishEvent('user.registered', {
      userId: user._id,
      email: user.email,
      username: user.username,
    });

    res.status(201).json({
      id: user._id,
      username: user.username,
      email: user.email,
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.post('/users/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });

    if (!user || !(await bcrypt.compare(password, user.password))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    const token = jwt.sign(
      { id: user._id, email: user.email, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );

    res.json({ token, user: { id: user._id, username: user.username } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Internal endpoint (สำหรับ services อื่น)
app.get('/internal/users/:id', validateInternalRequest, async (req, res) => {
  const user = await User.findById(req.params.id).select('-password');
  if (!user) return res.status(404).json({ error: 'User not found' });
  res.json(user);
});

app.get('/internal/users', validateInternalRequest, async (req, res) => {
  const { ids } = req.query;
  const userIds = ids.split(',');
  const users = await User.find({ _id: { $in: userIds } }).select('-password');
  res.json(users);
});

// Middleware สำหรับ internal requests
function validateInternalRequest(req, res, next) {
  const secret = req.headers['x-internal-secret'];
  if (secret !== process.env.INTERNAL_SECRET) {
    return res.status(403).json({ error: 'Forbidden' });
  }
  next();
}

mongoose.connect(process.env.MONGODB_URI).then(() => {
  app.listen(SERVICE.port, () => {
    console.log(`🚀 ${SERVICE.name} running on port ${SERVICE.port}`);
  });
});
```

### ขั้นตอนที่ 685: Post Service

```javascript
// post-service/src/index.js
const express = require('express');
const axios = require('axios');

const app = express();
app.use(express.json());

const USER_SERVICE_URL = process.env.USER_SERVICE_URL || 'http://user-service:3001';
const INTERNAL_SECRET = process.env.INTERNAL_SECRET;

// สร้าง axios instance สำหรับ internal calls
const internalClient = axios.create({
  headers: { 'x-internal-secret': INTERNAL_SECRET },
  timeout: 5000,
});

app.get('/health', (req, res) => {
  res.json({ service: 'post-service', status: 'healthy' });
});

// Get posts พร้อม author info
app.get('/posts', async (req, res) => {
  try {
    const posts = await Post.find({ status: 'PUBLISHED' })
      .sort({ createdAt: -1 })
      .limit(20);

    // ดึง unique author IDs
    const authorIds = [...new Set(posts.map(p => p.authorId.toString()))];

    // Fetch users from user-service
    const usersResponse = await internalClient.get(
      `${USER_SERVICE_URL}/internal/users?ids=${authorIds.join(',')}`
    );

    const usersMap = {};
    usersResponse.data.forEach(u => (usersMap[u._id] = u));

    // Combine posts with author data
    const postsWithAuthors = posts.map(post => ({
      ...post.toObject(),
      author: usersMap[post.authorId] || null,
    }));

    res.json(postsWithAuthors);
  } catch (error) {
    if (error.code === 'ECONNREFUSED') {
      // User service ไม่พร้อมใช้งาน - ส่งข้อมูลบางส่วน
      const posts = await Post.find({ status: 'PUBLISHED' });
      return res.json(posts); // ไม่มี author info
    }
    res.status(500).json({ error: error.message });
  }
});
```

---

## API Gateway

### ขั้นตอนที่ 686: Express API Gateway

```javascript
// api-gateway/src/index.js
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');
const jwt = require('jsonwebtoken');
const rateLimit = require('express-rate-limit');
const helmet = require('helmet');
const cors = require('cors');

const app = express();

// Security middleware
app.use(helmet());
app.use(cors());

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  message: 'Too many requests',
});
app.use(limiter);

// Service registry (ใน production ใช้ Consul/etcd)
const services = {
  'user-service': process.env.USER_SERVICE_URL || 'http://user-service:3001',
  'post-service': process.env.POST_SERVICE_URL || 'http://post-service:3002',
  'notification-service': process.env.NOTIFICATION_SERVICE_URL || 'http://notification-service:3003',
  'payment-service': process.env.PAYMENT_SERVICE_URL || 'http://payment-service:3004',
};

// Authentication middleware
function authenticate(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');

  if (!token) {
    req.user = null;
    return next();
  }

  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
}

// Authorization middleware
function requireAuth(req, res, next) {
  if (!req.user) {
    return res.status(401).json({ error: 'Authentication required' });
  }
  next();
}

function requireRole(role) {
  return (req, res, next) => {
    if (!req.user || req.user.role !== role) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}

// Request logging
app.use((req, res, next) => {
  const start = Date.now();
  console.log(`→ ${req.method} ${req.path}`);

  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`← ${req.method} ${req.path} ${res.statusCode} ${duration}ms`);
  });

  next();
});

app.use(authenticate);

// Proxy configurations
const proxyOptions = (target, pathRewrite = {}) => ({
  target,
  changeOrigin: true,
  pathRewrite,
  on: {
    proxyReq: (proxyReq, req) => {
      // Forward user info ไปยัง services
      if (req.user) {
        proxyReq.setHeader('x-user-id', req.user.id);
        proxyReq.setHeader('x-user-role', req.user.role);
        proxyReq.setHeader('x-user-email', req.user.email);
      }
      proxyReq.setHeader('x-request-id', generateRequestId());
      proxyReq.setHeader('x-internal-secret', process.env.INTERNAL_SECRET);
    },
    error: (err, req, res) => {
      console.error('Proxy error:', err.message);
      res.status(503).json({ error: 'Service temporarily unavailable' });
    },
  },
});

// Route to services
// User routes - public
app.use('/api/auth', createProxyMiddleware(
  proxyOptions(services['user-service'], { '^/api/auth': '/users' })
));

// User routes - protected
app.use('/api/users', requireAuth, createProxyMiddleware(
  proxyOptions(services['user-service'], { '^/api/users': '/users' })
));

// Post routes
app.use('/api/posts', createProxyMiddleware(
  proxyOptions(services['post-service'], { '^/api/posts': '/posts' })
));

// Notification routes - protected
app.use('/api/notifications', requireAuth, createProxyMiddleware(
  proxyOptions(services['notification-service'], { '^/api/notifications': '/notifications' })
));

// Payment routes - protected
app.use('/api/payments', requireAuth, createProxyMiddleware(
  proxyOptions(services['payment-service'], { '^/api/payments': '/payments' })
));

// Admin routes
app.use('/api/admin', requireAuth, requireRole('ADMIN'), createProxyMiddleware(
  proxyOptions(services['user-service'])
));

// Health check
app.get('/health', async (req, res) => {
  const serviceChecks = await Promise.allSettled(
    Object.entries(services).map(async ([name, url]) => {
      const response = await fetch(`${url}/health`, { timeout: 3000 });
      const data = await response.json();
      return { name, status: data.status, url };
    })
  );

  const results = serviceChecks.map((result, i) => ({
    service: Object.keys(services)[i],
    ...(result.status === 'fulfilled' ? result.value : { status: 'unhealthy' }),
  }));

  const allHealthy = results.every(r => r.status === 'healthy');
  res.status(allHealthy ? 200 : 503).json({
    gateway: 'healthy',
    services: results,
  });
});

function generateRequestId() {
  return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
}

app.listen(3000, () => {
  console.log('🚀 API Gateway running on port 3000');
});
```

---

## Service Discovery

### ขั้นตอนที่ 687: Service Registry

```javascript
// src/registry/ServiceRegistry.js
class ServiceRegistry {
  constructor() {
    this.services = new Map();
    this.healthCheckInterval = 30000;
    this.startHealthChecks();
  }

  // Register service
  register(name, instance) {
    if (!this.services.has(name)) {
      this.services.set(name, []);
    }

    const serviceList = this.services.get(name);
    const existing = serviceList.find(s => s.url === instance.url);

    if (!existing) {
      serviceList.push({
        ...instance,
        status: 'healthy',
        lastHeartbeat: Date.now(),
        failCount: 0,
      });
      console.log(`✅ Registered: ${name} at ${instance.url}`);
    }
  }

  // Deregister service
  deregister(name, url) {
    const serviceList = this.services.get(name);
    if (serviceList) {
      const index = serviceList.findIndex(s => s.url === url);
      if (index !== -1) {
        serviceList.splice(index, 1);
        console.log(`❌ Deregistered: ${name} at ${url}`);
      }
    }
  }

  // Get healthy instances (Round Robin load balancing)
  getInstance(name) {
    const instances = this.services.get(name);
    if (!instances || instances.length === 0) {
      throw new Error(`No instances found for service: ${name}`);
    }

    const healthyInstances = instances.filter(i => i.status === 'healthy');
    if (healthyInstances.length === 0) {
      throw new Error(`No healthy instances for service: ${name}`);
    }

    // Round Robin
    const instance = healthyInstances[
      (this._roundRobinIndex(name)) % healthyInstances.length
    ];

    return instance.url;
  }

  _roundRobinIndex(name) {
    if (!this._counters) this._counters = {};
    if (!this._counters[name]) this._counters[name] = 0;
    return this._counters[name]++;
  }

  // Health checks
  startHealthChecks() {
    setInterval(async () => {
      for (const [name, instances] of this.services) {
        for (const instance of instances) {
          try {
            const response = await fetch(`${instance.url}/health`, {
              signal: AbortSignal.timeout(3000),
            });

            if (response.ok) {
              instance.status = 'healthy';
              instance.lastHeartbeat = Date.now();
              instance.failCount = 0;
            } else {
              instance.failCount++;
            }
          } catch (error) {
            instance.failCount++;
            if (instance.failCount > 3) {
              instance.status = 'unhealthy';
              console.warn(`⚠️ Service ${name} at ${instance.url} is unhealthy`);
            }
          }
        }
      }
    }, this.healthCheckInterval);
  }

  getAllServices() {
    const result = {};
    for (const [name, instances] of this.services) {
      result[name] = instances.map(({ url, status, lastHeartbeat }) => ({
        url, status, lastHeartbeat,
      }));
    }
    return result;
  }
}

module.exports = new ServiceRegistry();
```

---

## Inter-Service Communication

### ขั้นตอนที่ 688: HTTP Client สำหรับ Services

```javascript
// src/clients/ServiceClient.js
const axios = require('axios');
const registry = require('../registry/ServiceRegistry');

class ServiceClient {
  constructor(serviceName) {
    this.serviceName = serviceName;
    this.client = axios.create({
      timeout: 10000,
      headers: {
        'x-internal-secret': process.env.INTERNAL_SECRET,
        'Content-Type': 'application/json',
      },
    });

    // Interceptors
    this.client.interceptors.request.use(
      (config) => {
        const serviceUrl = registry.getInstance(this.serviceName);
        config.baseURL = serviceUrl;
        config.headers['x-request-id'] = generateRequestId();
        return config;
      },
      (error) => Promise.reject(error)
    );

    this.client.interceptors.response.use(
      (response) => response.data,
      async (error) => {
        if (error.response?.status === 503) {
          // Service unavailable - retry once
          const config = error.config;
          if (!config._retry) {
            config._retry = true;
            config.baseURL = registry.getInstance(this.serviceName);
            return this.client(config);
          }
        }
        throw error;
      }
    );
  }

  async get(path, params = {}) {
    return this.client.get(path, { params });
  }

  async post(path, data) {
    return this.client.post(path, data);
  }

  async put(path, data) {
    return this.client.put(path, data);
  }

  async delete(path) {
    return this.client.delete(path);
  }
}

// สร้าง clients
const userClient = new ServiceClient('user-service');
const postClient = new ServiceClient('post-service');

module.exports = { userClient, postClient };
```

### ขั้นตอนที่ 689: Event-Driven Communication

```javascript
// src/events/eventBus.js
const { createClient } = require('redis');

class EventBus {
  constructor() {
    this.publisher = createClient({ url: process.env.REDIS_URL });
    this.subscriber = createClient({ url: process.env.REDIS_URL });
    this.handlers = new Map();
  }

  async connect() {
    await this.publisher.connect();
    await this.subscriber.connect();
    console.log('✅ Event bus connected');
  }

  async publish(event, data) {
    const message = {
      event,
      data,
      service: process.env.SERVICE_NAME,
      timestamp: new Date().toISOString(),
      id: generateEventId(),
    };

    await this.publisher.publish(
      `events:${event}`,
      JSON.stringify(message)
    );

    console.log(`📤 Published event: ${event}`);
  }

  async subscribe(event, handler) {
    const channel = `events:${event}`;

    await this.subscriber.subscribe(channel, (message) => {
      try {
        const data = JSON.parse(message);
        handler(data);
      } catch (error) {
        console.error(`Failed to handle event ${event}:`, error);
      }
    });

    console.log(`📥 Subscribed to event: ${event}`);
  }

  async subscribePattern(pattern, handler) {
    await this.subscriber.pSubscribe(`events:${pattern}`, (message, channel) => {
      try {
        const data = JSON.parse(message);
        const event = channel.replace('events:', '');
        handler(event, data);
      } catch (error) {
        console.error('Failed to handle event:', error);
      }
    });
  }
}

const eventBus = new EventBus();
module.exports = eventBus;
```

### ขั้นตอนที่ 690: Saga Pattern สำหรับ Distributed Transactions

```javascript
// src/sagas/orderSaga.js
const eventBus = require('../events/eventBus');

// Saga เป็น pattern สำหรับ manage distributed transactions
// โดยใช้ compensating transactions เมื่อเกิดข้อผิดพลาด

class OrderSaga {
  constructor(orderId) {
    this.orderId = orderId;
    this.steps = [];
    this.completedSteps = [];
  }

  async execute() {
    try {
      // Step 1: Reserve inventory
      await this.step(
        'RESERVE_INVENTORY',
        () => this.reserveInventory(),
        () => this.releaseInventory()  // Compensating action
      );

      // Step 2: Process payment
      await this.step(
        'PROCESS_PAYMENT',
        () => this.processPayment(),
        () => this.refundPayment()
      );

      // Step 3: Confirm order
      await this.step(
        'CONFIRM_ORDER',
        () => this.confirmOrder(),
        () => this.cancelOrder()
      );

      await eventBus.publish('order.completed', { orderId: this.orderId });
      console.log(`✅ Order ${this.orderId} completed`);

    } catch (error) {
      console.error(`❌ Order ${this.orderId} failed:`, error.message);
      await this.compensate();
      await eventBus.publish('order.failed', {
        orderId: this.orderId,
        reason: error.message,
      });
    }
  }

  async step(name, action, compensate) {
    try {
      console.log(`Executing step: ${name}`);
      await action();
      this.completedSteps.push({ name, compensate });
    } catch (error) {
      throw new Error(`Step ${name} failed: ${error.message}`);
    }
  }

  async compensate() {
    console.log('Starting compensation...');
    // ทำ compensating actions ในลำดับย้อนกลับ
    for (const step of this.completedSteps.reverse()) {
      try {
        console.log(`Compensating step: ${step.name}`);
        await step.compensate();
      } catch (error) {
        console.error(`Compensation failed for ${step.name}:`, error);
      }
    }
  }

  async reserveInventory() {
    // Call inventory service
  }

  async releaseInventory() {
    // Release reserved inventory
  }

  async processPayment() {
    // Call payment service
  }

  async refundPayment() {
    // Refund payment
  }

  async confirmOrder() {
    // Confirm order in DB
  }

  async cancelOrder() {
    // Cancel order
  }
}
```

---

## Database per Service

### ขั้นตอนที่ 691: การจัดการ Data Consistency

```javascript
// ปัญหา: แต่ละ service มี database ของตัวเอง
// วิธีแก้: Event Sourcing + CQRS

// Event Store
class EventStore {
  async appendEvent(aggregateId, eventType, data) {
    const event = {
      id: generateId(),
      aggregateId,
      eventType,
      data,
      timestamp: new Date(),
      version: await this.getNextVersion(aggregateId),
    };

    await Event.create(event);
    await eventBus.publish(eventType, event);

    return event;
  }

  async getEvents(aggregateId) {
    return Event.find({ aggregateId }).sort({ version: 1 });
  }

  async getNextVersion(aggregateId) {
    const latest = await Event.findOne({ aggregateId }).sort({ version: -1 });
    return (latest?.version || 0) + 1;
  }
}

// Command handlers
async function handleCreateOrder(command) {
  const { userId, items, totalAmount } = command;

  const orderId = generateId();

  // Append events
  await eventStore.appendEvent(orderId, 'ORDER_CREATED', {
    userId,
    items,
    totalAmount,
  });

  return orderId;
}

// Read models (Projections)
async function buildOrderReadModel(orderId) {
  const events = await eventStore.getEvents(orderId);
  let order = null;

  for (const event of events) {
    order = applyEvent(order, event);
  }

  return order;
}

function applyEvent(state, event) {
  switch (event.eventType) {
    case 'ORDER_CREATED':
      return { id: event.aggregateId, ...event.data, status: 'PENDING' };
    case 'PAYMENT_COMPLETED':
      return { ...state, status: 'PAID' };
    case 'ORDER_SHIPPED':
      return { ...state, status: 'SHIPPED', trackingNumber: event.data.trackingNumber };
    default:
      return state;
  }
}
```

---

## Distributed Tracing

### ขั้นตอนที่ 692: OpenTelemetry

```bash
npm install @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node
npm install @opentelemetry/exporter-jaeger
```

```javascript
// src/tracing/setup.js
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: process.env.SERVICE_NAME,
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.0.0',
  }),
  traceExporter: new JaegerExporter({
    endpoint: process.env.JAEGER_ENDPOINT || 'http://jaeger:14268/api/traces',
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
console.log('✅ Tracing initialized');

// Graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown().then(() => console.log('Tracing terminated'));
});
```

```javascript
// ใช้ tracing ใน code
const { trace, context, propagation } = require('@opentelemetry/api');

const tracer = trace.getTracer('my-service');

async function processOrder(orderId) {
  return tracer.startActiveSpan('processOrder', async (span) => {
    try {
      span.setAttribute('orderId', orderId);

      const order = await tracer.startActiveSpan('fetchOrder', async (childSpan) => {
        const result = await Order.findById(orderId);
        childSpan.setAttribute('found', !!result);
        childSpan.end();
        return result;
      });

      span.setStatus({ code: SpanStatusCode.OK });
      return order;
    } catch (error) {
      span.recordException(error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

---

## Resilience Patterns

### ขั้นตอนที่ 693: Circuit Breaker

```javascript
// src/resilience/CircuitBreaker.js
const EventEmitter = require('events');

class CircuitBreaker extends EventEmitter {
  constructor(options = {}) {
    super();
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 60000;
    this.halfOpenRequests = options.halfOpenRequests || 3;
    this.monitoringPeriod = options.monitoringPeriod || 10000;

    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
    this.nextAttemptTime = null;
    this.halfOpenCount = 0;
  }

  async execute(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttemptTime) {
        throw new Error(`Circuit breaker is OPEN until ${new Date(this.nextAttemptTime).toISOString()}`);
      }
      this.transitionToHalfOpen();
    }

    if (this.state === 'HALF_OPEN') {
      if (this.halfOpenCount >= this.halfOpenRequests) {
        throw new Error('Circuit breaker is HALF_OPEN - limit reached');
      }
      this.halfOpenCount++;
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failureCount = 0;

    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.halfOpenRequests) {
        this.transitionToClosed();
      }
    }
  }

  onFailure() {
    this.failureCount++;

    if (this.state === 'HALF_OPEN') {
      this.transitionToOpen();
      return;
    }

    if (this.failureCount >= this.failureThreshold) {
      this.transitionToOpen();
    }
  }

  transitionToOpen() {
    this.state = 'OPEN';
    this.nextAttemptTime = Date.now() + this.resetTimeout;
    this.emit('open', { failures: this.failureCount });
    console.warn(`⚡ Circuit breaker OPENED (${this.failureCount} failures)`);
  }

  transitionToHalfOpen() {
    this.state = 'HALF_OPEN';
    this.halfOpenCount = 0;
    this.successCount = 0;
    this.emit('half-open');
    console.log('🔄 Circuit breaker HALF_OPEN');
  }

  transitionToClosed() {
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.emit('close');
    console.log('✅ Circuit breaker CLOSED');
  }

  getState() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      nextAttemptTime: this.nextAttemptTime,
    };
  }
}

module.exports = CircuitBreaker;
```

### ขั้นตอนที่ 694: Retry with Exponential Backoff

```javascript
// src/resilience/retry.js
async function withRetry(fn, options = {}) {
  const {
    attempts = 3,
    initialDelay = 1000,
    maxDelay = 30000,
    factor = 2,
    jitter = true,
    retryCondition = (err) => true,
  } = options;

  let lastError;

  for (let attempt = 1; attempt <= attempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      if (attempt === attempts || !retryCondition(error)) {
        throw error;
      }

      // คำนวณ delay
      let delay = Math.min(initialDelay * Math.pow(factor, attempt - 1), maxDelay);

      // เพิ่ม jitter เพื่อป้องกัน thundering herd
      if (jitter) {
        delay = delay * (0.5 + Math.random() * 0.5);
      }

      console.log(`Retry attempt ${attempt}/${attempts} after ${Math.round(delay)}ms`);
      await sleep(delay);
    }
  }

  throw lastError;
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// ตัวอย่างใช้งาน
const result = await withRetry(
  () => userClient.get('/internal/users/123'),
  {
    attempts: 3,
    retryCondition: (error) => {
      // ไม่ retry สำหรับ client errors (4xx)
      return !error.response || error.response.status >= 500;
    },
  }
);
```

### ขั้นตอนที่ 695: Bulkhead Pattern

```javascript
// จำกัด concurrent requests ไปยัง service
class Bulkhead {
  constructor(maxConcurrent = 10, maxQueue = 100) {
    this.maxConcurrent = maxConcurrent;
    this.maxQueue = maxQueue;
    this.active = 0;
    this.queue = [];
  }

  async execute(fn) {
    if (this.active >= this.maxConcurrent) {
      if (this.queue.length >= this.maxQueue) {
        throw new Error('Bulkhead queue full - rejecting request');
      }

      // รอใน queue
      await new Promise((resolve, reject) => {
        this.queue.push({ resolve, reject });
      });
    }

    this.active++;
    try {
      return await fn();
    } finally {
      this.active--;
      if (this.queue.length > 0) {
        const next = this.queue.shift();
        next.resolve();
      }
    }
  }

  getStats() {
    return { active: this.active, queued: this.queue.length };
  }
}

const userServiceBulkhead = new Bulkhead(20, 50);

async function getUserById(id) {
  return userServiceBulkhead.execute(async () => {
    return userClient.get(`/internal/users/${id}`);
  });
}
```

---

## docker-compose สำหรับ Microservices

### ขั้นตอนที่ 696: Docker Compose Configuration

```yaml
# docker-compose.yml
version: '3.8'

networks:
  microservices:
    driver: bridge

services:
  # API Gateway
  api-gateway:
    build: ./api-gateway
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - JWT_SECRET=${JWT_SECRET}
      - USER_SERVICE_URL=http://user-service:3001
      - POST_SERVICE_URL=http://post-service:3002
      - INTERNAL_SECRET=${INTERNAL_SECRET}
    depends_on:
      - user-service
      - post-service
    networks:
      - microservices
    restart: unless-stopped

  # User Service
  user-service:
    build: ./user-service
    environment:
      - NODE_ENV=production
      - PORT=3001
      - MONGODB_URI=mongodb://mongodb-users:27017/users
      - JWT_SECRET=${JWT_SECRET}
      - REDIS_URL=redis://redis:6379
      - INTERNAL_SECRET=${INTERNAL_SECRET}
    depends_on:
      - mongodb-users
      - redis
    networks:
      - microservices
    restart: unless-stopped

  # Post Service
  post-service:
    build: ./post-service
    environment:
      - NODE_ENV=production
      - PORT=3002
      - MONGODB_URI=mongodb://mongodb-posts:27017/posts
      - USER_SERVICE_URL=http://user-service:3001
      - INTERNAL_SECRET=${INTERNAL_SECRET}
    depends_on:
      - mongodb-posts
    networks:
      - microservices
    restart: unless-stopped

  # Databases
  mongodb-users:
    image: mongo:6
    volumes:
      - mongodb-users-data:/data/db
    networks:
      - microservices
    restart: unless-stopped

  mongodb-posts:
    image: mongo:6
    volumes:
      - mongodb-posts-data:/data/db
    networks:
      - microservices
    restart: unless-stopped

  # Redis
  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    networks:
      - microservices
    restart: unless-stopped

  # Jaeger (Distributed Tracing)
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "14268:14268"  # Collector
    networks:
      - microservices

volumes:
  mongodb-users-data:
  mongodb-posts-data:
  redis-data:
```

---

## Service Mesh

### ขั้นตอนที่ 697: Consul Service Discovery

```javascript
// src/consul/index.js
const Consul = require('consul');

const consul = new Consul({
  host: process.env.CONSUL_HOST || 'consul',
  port: parseInt(process.env.CONSUL_PORT) || 8500,
});

async function registerService() {
  const serviceName = process.env.SERVICE_NAME;
  const servicePort = parseInt(process.env.PORT);
  const serviceHost = process.env.SERVICE_HOST || 'localhost';

  await consul.agent.service.register({
    id: `${serviceName}-${serviceHost}-${servicePort}`,
    name: serviceName,
    address: serviceHost,
    port: servicePort,
    check: {
      http: `http://${serviceHost}:${servicePort}/health`,
      interval: '15s',
      timeout: '5s',
      deregistercriticalserviceafter: '30s',
    },
    tags: ['nodejs', 'microservice'],
  });

  console.log(`✅ Registered with Consul: ${serviceName}`);
}

async function discoverService(name) {
  const result = await consul.health.service({
    service: name,
    passing: true,  // เฉพาะ healthy instances
  });

  if (result.length === 0) {
    throw new Error(`No healthy instances for: ${name}`);
  }

  // Random selection
  const instance = result[Math.floor(Math.random() * result.length)];
  return `http://${instance.Service.Address}:${instance.Service.Port}`;
}

module.exports = { registerService, discoverService };
```

---

## Logging

### ขั้นตอนที่ 698: Centralized Logging

```javascript
// src/logging/logger.js
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  defaultMeta: {
    service: process.env.SERVICE_NAME,
    version: process.env.SERVICE_VERSION,
    env: process.env.NODE_ENV,
  },
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      ),
    }),
    // ส่งไปยัง centralized logging (ELK Stack)
    new winston.transports.Http({
      host: process.env.LOGSTASH_HOST,
      port: parseInt(process.env.LOGSTASH_PORT) || 5000,
      path: '/',
    }),
  ],
});

// Middleware
function requestLogger(req, res, next) {
  const requestId = req.headers['x-request-id'] || generateId();
  req.requestId = requestId;

  const start = Date.now();

  res.on('finish', () => {
    logger.info('HTTP Request', {
      requestId,
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration: Date.now() - start,
      ip: req.ip,
      userAgent: req.headers['user-agent'],
      userId: req.headers['x-user-id'],
    });
  });

  next();
}

module.exports = { logger, requestLogger };
```

---

## Testing Microservices

### ขั้นตอนที่ 699: Contract Testing

```javascript
// Contract testing ด้วย Pact
const { Pact } = require('@pact-foundation/pact');
const { like, eachLike } = require('@pact-foundation/pact').Matchers;

describe('Post Service - User Service contract', () => {
  const provider = new Pact({
    consumer: 'PostService',
    provider: 'UserService',
    port: 4000,
    log: path.resolve(process.cwd(), 'logs', 'pact.log'),
    dir: path.resolve(process.cwd(), 'pacts'),
  });

  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());
  afterEach(() => provider.verify());

  describe('Get user by ID', () => {
    beforeEach(() => {
      return provider.addInteraction({
        state: 'User 1 exists',
        uponReceiving: 'a request for user 1',
        withRequest: {
          method: 'GET',
          path: '/internal/users/1',
          headers: { 'x-internal-secret': 'test-secret' },
        },
        willRespondWith: {
          status: 200,
          body: like({
            id: '1',
            username: like('testuser'),
            email: like('test@example.com'),
          }),
        },
      });
    });

    it('should return user data', async () => {
      const response = await userClient.get('/internal/users/1');
      expect(response.id).toBeDefined();
      expect(response.username).toBeDefined();
    });
  });
});
```

### ขั้นตอนที่ 700: Integration Testing

```javascript
// integration.test.js
const request = require('supertest');
const app = require('../src/app');

describe('User Service Integration Tests', () => {
  let authToken;

  beforeAll(async () => {
    // ตั้งค่า test database
    await mongoose.connect(process.env.MONGODB_TEST_URI);
  });

  afterAll(async () => {
    await mongoose.disconnect();
  });

  test('POST /users/register should create user', async () => {
    const response = await request(app)
      .post('/users/register')
      .send({
        username: 'testuser',
        email: 'test@example.com',
        password: 'password123',
      });

    expect(response.status).toBe(201);
    expect(response.body.username).toBe('testuser');
    expect(response.body.email).toBe('test@example.com');
    expect(response.body.password).toBeUndefined();
  });

  test('POST /users/login should return token', async () => {
    const response = await request(app)
      .post('/users/login')
      .send({
        email: 'test@example.com',
        password: 'password123',
      });

    expect(response.status).toBe(200);
    expect(response.body.token).toBeDefined();
    authToken = response.body.token;
  });

  test('GET /internal/users/:id should require internal secret', async () => {
    const response = await request(app)
      .get('/internal/users/123')
      .expect(403);
  });
});
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Notification Service

```javascript
// TODO: สร้าง notification-service ที่:
// - Subscribe to events จาก services อื่น
// - ส่ง email, SMS, push notifications
// - Store notification history
// - Mark as read
```

### แบบฝึกหัดที่ 2: Implement Service Mesh

```javascript
// TODO: เพิ่ม service mesh capabilities:
// - mTLS ระหว่าง services
// - Traffic management (canary deployments)
// - Observability (metrics, traces, logs)
```

### แบบฝึกหัดที่ 3: CQRS Pattern

```javascript
// TODO: Implement CQRS ใน post-service:
// - Command side: create, update, delete posts
// - Query side: optimized read models
// - Event sourcing สำหรับ audit trail
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Microservices Concepts** - ข้อดี/ข้อเสีย และเมื่อไหรควรใช้
2. **API Gateway** - single entry point, routing, authentication
3. **Service Discovery** - registry และ health checks
4. **Inter-Service Communication** - HTTP และ event-driven
5. **Database per Service** - data consistency, event sourcing
6. **Distributed Tracing** - OpenTelemetry, Jaeger
7. **Resilience** - circuit breaker, retry, bulkhead
8. **Testing** - contract testing, integration testing

ในบทถัดไปเราจะเรียนรู้ **Docker และ Node.js** - containerization และ deployment
