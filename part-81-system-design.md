# Part 81 | ขั้นตอนที่ 1421-1440 จาก 1000+

## System Design Fundamentals สำหรับ Node.js Applications

ในส่วนนี้เราจะเรียนรู้หลักการออกแบบระบบระดับ world-class ที่ใช้ใน production จริง รวมถึง CAP theorem, scalability patterns, และ load balancing

---

## ขั้นตอนที่ 1421: CAP Theorem และการประยุกต์ใช้

**CAP Theorem** คือทฤษฎีที่กล่าวว่าระบบ distributed ไม่สามารถรับประกันคุณสมบัติทั้งสามพร้อมกันได้:

- **C**onsistency - ข้อมูลทุกโหนดเหมือนกัน
- **A**vailability - ระบบตอบสนองเสมอ
- **P**artition Tolerance - ระบบทำงานได้แม้เครือข่ายขาด

```javascript
// cap-theorem-demo.js
// แสดงการเลือก trade-off ระหว่าง Consistency และ Availability

const express = require('express');
const Redis = require('ioredis');

const app = express();

// CP System - เลือก Consistency + Partition Tolerance
// ใช้ Redis ด้วย Strong Consistency (single master)
class CPSystem {
  constructor() {
    this.redis = new Redis({
      host: 'localhost',
      port: 6379,
      // ตั้ง retry ให้รอนานขึ้น เพื่อรอ master กลับมา
      retryStrategy: (times) => Math.min(times * 100, 3000)
    });
  }

  async write(key, value) {
    // ใช้ WAIT command เพื่อให้แน่ใจว่า replica ได้รับข้อมูล
    await this.redis.set(key, JSON.stringify(value));
    // รอให้อย่างน้อย 1 replica acknowledge
    const replicated = await this.redis.wait(1, 1000);
    if (replicated < 1) {
      throw new Error('Replication failed - cannot guarantee consistency');
    }
    return { success: true, consistent: true };
  }

  async read(key) {
    const value = await this.redis.get(key);
    return value ? JSON.parse(value) : null;
  }
}

// AP System - เลือก Availability + Partition Tolerance
// ใช้ Eventually Consistent approach
class APSystem {
  constructor() {
    this.localCache = new Map();
    this.pendingSync = [];
  }

  async write(key, value) {
    // เขียนใน local cache ก่อน (always available)
    this.localCache.set(key, {
      value,
      timestamp: Date.now(),
      synced: false
    });
    
    // พยายาม sync กับ remote แบบ async
    this.syncAsync(key, value);
    
    return { success: true, consistent: false, note: 'Eventually consistent' };
  }

  syncAsync(key, value) {
    // Async sync - ไม่รอผล
    setImmediate(async () => {
      try {
        // ส่งไปยัง remote storage
        await this.remoteSync(key, value);
        const entry = this.localCache.get(key);
        if (entry) entry.synced = true;
      } catch (err) {
        // เพิ่มใน retry queue
        this.pendingSync.push({ key, value, retries: 0 });
      }
    });
  }

  async remoteSync(key, value) {
    // Simulate remote sync
    console.log(`Syncing ${key} to remote...`);
  }

  async read(key) {
    // อ่านจาก local cache เสมอ (always available)
    const entry = this.localCache.get(key);
    return entry ? { value: entry.value, synced: entry.synced } : null;
  }
}

// API endpoint แสดง CAP trade-off
app.get('/cap/demo', (req, res) => {
  res.json({
    capTheorem: {
      consistency: 'ข้อมูลทุก node เหมือนกัน ณ เวลาใดเวลาหนึ่ง',
      availability: 'ระบบตอบสนองทุก request เสมอ',
      partitionTolerance: 'ระบบทำงานได้แม้ network partition'
    },
    realWorldExamples: {
      CP: ['MongoDB (strong consistency mode)', 'HBase', 'ZooKeeper'],
      AP: ['CouchDB', 'Cassandra', 'DynamoDB (default)'],
      CA: ['Traditional RDBMS (single node)', 'ไม่มีใน distributed system จริงๆ']
    },
    choosingStrategy: {
      useCP: 'เมื่อต้องการ data integrity เช่น financial transactions',
      useAP: 'เมื่อต้องการ user experience เช่น shopping cart, social feeds'
    }
  });
});

module.exports = { CPSystem, APSystem };
```

---

## ขั้นตอนที่ 1422: Scalability Patterns

มี 2 วิธีหลักในการ scale ระบบ:

### Horizontal Scaling (Scale Out)
เพิ่มจำนวน server instances

### Vertical Scaling (Scale Up)
เพิ่ม resources ของ server เดิม

```javascript
// scalability-patterns.js
// แสดง patterns การ scale ระบบ

const cluster = require('cluster');
const os = require('os');
const express = require('express');

// Pattern 1: Horizontal Scaling ด้วย Node.js Cluster
if (cluster.isMaster) {
  const numCPUs = os.cpus().length;
  console.log(`Master process ${process.pid} is running`);
  console.log(`Forking ${numCPUs} workers...`);

  // Fork workers ตามจำนวน CPU cores
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  // ตรวจสอบ worker ที่ตาย และ restart
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died. Restarting...`);
    cluster.fork(); // Auto-restart
  });

  // Graceful shutdown
  process.on('SIGTERM', () => {
    console.log('Shutting down gracefully...');
    for (const id in cluster.workers) {
      cluster.workers[id].send('shutdown');
    }
    setTimeout(() => process.exit(0), 10000);
  });

} else {
  // Worker process
  const app = express();
  
  app.get('/health', (req, res) => {
    res.json({ 
      pid: process.pid, 
      uptime: process.uptime(),
      memory: process.memoryUsage()
    });
  });

  app.listen(3000, () => {
    console.log(`Worker ${process.pid} started`);
  });

  // Handle graceful shutdown message
  process.on('message', (msg) => {
    if (msg === 'shutdown') {
      // รอ requests ที่ค้างอยู่ให้เสร็จก่อน
      server.close(() => {
        process.exit(0);
      });
    }
  });
}
```

```javascript
// Pattern 2: Stateless Design สำหรับ Horizontal Scaling
// state ทั้งหมดต้องอยู่นอก process

const express = require('express');
const Redis = require('ioredis');
const jwt = require('jsonwebtoken');

const app = express();
const redis = new Redis();

// ❌ Bad: State ใน memory (ไม่ scale)
const badSessions = {}; // นี่คือปัญหา - แต่ละ instance มี session แยกกัน

// ✅ Good: State ใน Redis (scale ได้)
class StatelessSessionManager {
  constructor(redisClient) {
    this.redis = redisClient;
    this.ttl = 3600; // 1 hour
  }

  async createSession(userId, data) {
    const sessionId = require('crypto').randomUUID();
    const sessionData = {
      userId,
      ...data,
      createdAt: Date.now()
    };
    
    await this.redis.setex(
      `session:${sessionId}`,
      this.ttl,
      JSON.stringify(sessionData)
    );
    
    return sessionId;
  }

  async getSession(sessionId) {
    const data = await this.redis.get(`session:${sessionId}`);
    return data ? JSON.parse(data) : null;
  }

  async destroySession(sessionId) {
    await this.redis.del(`session:${sessionId}`);
  }

  async extendSession(sessionId) {
    await this.redis.expire(`session:${sessionId}`, this.ttl);
  }
}

const sessionManager = new StatelessSessionManager(redis);

// Pattern 3: Shard by User ID
function getShardId(userId, numShards = 8) {
  // Consistent hash ของ userId
  let hash = 0;
  for (let i = 0; i < userId.length; i++) {
    const char = userId.charCodeAt(i);
    hash = ((hash << 5) - hash) + char;
    hash = hash & hash; // Convert to 32bit integer
  }
  return Math.abs(hash) % numShards;
}

// Route requests ไปยัง shard ที่ถูกต้อง
app.get('/user/:userId/data', async (req, res) => {
  const { userId } = req.params;
  const shardId = getShardId(userId);
  
  res.json({
    userId,
    shardId,
    message: `Data would be fetched from shard ${shardId}`
  });
});

module.exports = { StatelessSessionManager, getShardId };
```

---

## ขั้นตอนที่ 1423: Load Balancing Strategies

```javascript
// load-balancer.js
// Custom load balancer implementation

const http = require('http');
const httpProxy = require('http-proxy-middleware');

// Load Balancing Algorithms
class LoadBalancer {
  constructor(servers) {
    this.servers = servers.map(s => ({ ...s, connections: 0, healthy: true }));
    this.currentIndex = 0;
  }

  // Algorithm 1: Round Robin
  roundRobin() {
    const healthyServers = this.servers.filter(s => s.healthy);
    if (healthyServers.length === 0) throw new Error('No healthy servers');
    
    const server = healthyServers[this.currentIndex % healthyServers.length];
    this.currentIndex = (this.currentIndex + 1) % healthyServers.length;
    return server;
  }

  // Algorithm 2: Weighted Round Robin
  weightedRoundRobin() {
    const healthyServers = this.servers.filter(s => s.healthy);
    const totalWeight = healthyServers.reduce((sum, s) => sum + (s.weight || 1), 0);
    let random = Math.random() * totalWeight;
    
    for (const server of healthyServers) {
      random -= (server.weight || 1);
      if (random <= 0) return server;
    }
    
    return healthyServers[healthyServers.length - 1];
  }

  // Algorithm 3: Least Connections
  leastConnections() {
    const healthyServers = this.servers.filter(s => s.healthy);
    return healthyServers.reduce((least, server) => 
      server.connections < least.connections ? server : least
    );
  }

  // Algorithm 4: IP Hash (Sticky Sessions)
  ipHash(clientIP) {
    const healthyServers = this.servers.filter(s => s.healthy);
    let hash = 0;
    for (let i = 0; i < clientIP.length; i++) {
      hash = (hash * 31 + clientIP.charCodeAt(i)) % healthyServers.length;
    }
    return healthyServers[Math.abs(hash) % healthyServers.length];
  }

  // Health Check
  async healthCheck() {
    const checks = this.servers.map(async (server) => {
      try {
        const response = await fetch(`http://${server.host}:${server.port}/health`, {
          signal: AbortSignal.timeout(5000)
        });
        server.healthy = response.ok;
        server.lastCheck = Date.now();
      } catch {
        server.healthy = false;
        server.lastCheck = Date.now();
      }
    });
    
    await Promise.allSettled(checks);
    
    const healthyCount = this.servers.filter(s => s.healthy).length;
    console.log(`Health check: ${healthyCount}/${this.servers.length} servers healthy`);
  }

  startHealthChecks(intervalMs = 30000) {
    this.healthCheckInterval = setInterval(() => {
      this.healthCheck();
    }, intervalMs);
    
    // Run immediately
    this.healthCheck();
    
    return () => clearInterval(this.healthCheckInterval);
  }

  getStats() {
    return this.servers.map(s => ({
      host: s.host,
      port: s.port,
      healthy: s.healthy,
      connections: s.connections,
      weight: s.weight || 1
    }));
  }
}

// Express middleware สำหรับ load balancing metrics
const express = require('express');
const app = express();

const lb = new LoadBalancer([
  { host: 'app1', port: 3001, weight: 3 }, // Server แรงกว่า รับ traffic มากกว่า
  { host: 'app2', port: 3002, weight: 2 },
  { host: 'app3', port: 3003, weight: 1 }
]);

lb.startHealthChecks(10000);

app.get('/lb/stats', (req, res) => {
  res.json({
    algorithm: 'weighted-round-robin',
    servers: lb.getStats()
  });
});

// Simulate load balancing
app.get('/lb/next', (req, res) => {
  const clientIP = req.ip || '127.0.0.1';
  
  const roundRobin = lb.roundRobin();
  const leastConn = lb.leastConnections();
  const ipBased = lb.ipHash(clientIP);
  
  res.json({
    roundRobin: { host: roundRobin.host, port: roundRobin.port },
    leastConnections: { host: leastConn.host, port: leastConn.port },
    ipHash: { host: ipBased.host, port: ipBased.port, clientIP }
  });
});

module.exports = { LoadBalancer };
```

---

## ขั้นตอนที่ 1424: Database Connection Pooling และ Optimization

```javascript
// db-pooling.js
// Connection pool สำหรับ database ที่ scale ได้

const { Pool } = require('pg');
const mysql = require('mysql2/promise');

// PostgreSQL Connection Pool
class PostgreSQLPool {
  constructor(config) {
    this.pool = new Pool({
      host: config.host,
      port: config.port || 5432,
      database: config.database,
      user: config.user,
      password: config.password,
      
      // Pool configuration
      min: 5,           // Minimum connections
      max: 20,          // Maximum connections
      idleTimeoutMillis: 30000,   // Remove idle connections after 30s
      connectionTimeoutMillis: 2000, // Error if can't connect in 2s
      
      // Statement timeout
      statement_timeout: 10000,  // 10 second query timeout
    });

    // Monitor pool events
    this.pool.on('connect', (client) => {
      console.log('New database connection established');
    });

    this.pool.on('error', (err, client) => {
      console.error('Unexpected error on idle client', err);
    });
  }

  async query(text, params) {
    const start = Date.now();
    const client = await this.pool.connect();
    
    try {
      const result = await client.query(text, params);
      const duration = Date.now() - start;
      
      // Log slow queries
      if (duration > 1000) {
        console.warn(`Slow query detected (${duration}ms):`, text);
      }
      
      return result;
    } finally {
      client.release();
    }
  }

  // Transaction helper
  async transaction(callback) {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      const result = await callback(client);
      await client.query('COMMIT');
      return result;
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }

  getPoolStats() {
    return {
      totalCount: this.pool.totalCount,
      idleCount: this.pool.idleCount,
      waitingCount: this.pool.waitingCount
    };
  }

  async close() {
    await this.pool.end();
  }
}

// Read/Write Split Pattern
class ReadWriteSplitDB {
  constructor(masterConfig, replicaConfigs) {
    this.master = new PostgreSQLPool(masterConfig);
    this.replicas = replicaConfigs.map(config => new PostgreSQLPool(config));
    this.replicaIndex = 0;
  }

  // Write operations -> Master
  async write(text, params) {
    return this.master.query(text, params);
  }

  // Read operations -> Replicas (Round Robin)
  async read(text, params) {
    if (this.replicas.length === 0) {
      return this.master.query(text, params);
    }
    
    const replica = this.replicas[this.replicaIndex % this.replicas.length];
    this.replicaIndex++;
    
    return replica.query(text, params);
  }

  // Transaction ต้องใช้ Master เสมอ
  async transaction(callback) {
    return this.master.transaction(callback);
  }
}

// Query Builder with automatic Read/Write routing
class SmartQueryBuilder {
  constructor(db) {
    this.db = db;
  }

  isReadQuery(sql) {
    const normalized = sql.trim().toUpperCase();
    return normalized.startsWith('SELECT') || 
           normalized.startsWith('WITH') ||
           normalized.startsWith('EXPLAIN');
  }

  async execute(sql, params = []) {
    if (this.isReadQuery(sql)) {
      return this.db.read(sql, params);
    } else {
      return this.db.write(sql, params);
    }
  }

  // Helper methods
  async findById(table, id) {
    return this.execute(`SELECT * FROM ${table} WHERE id = $1`, [id]);
  }

  async findMany(table, conditions = {}) {
    const keys = Object.keys(conditions);
    if (keys.length === 0) {
      return this.execute(`SELECT * FROM ${table}`);
    }
    
    const whereClause = keys.map((k, i) => `${k} = $${i + 1}`).join(' AND ');
    const values = Object.values(conditions);
    
    return this.execute(`SELECT * FROM ${table} WHERE ${whereClause}`, values);
  }

  async insert(table, data) {
    const keys = Object.keys(data);
    const values = Object.values(data);
    const placeholders = keys.map((_, i) => `$${i + 1}`).join(', ');
    
    return this.execute(
      `INSERT INTO ${table} (${keys.join(', ')}) VALUES (${placeholders}) RETURNING *`,
      values
    );
  }
}

module.exports = { PostgreSQLPool, ReadWriteSplitDB, SmartQueryBuilder };
```

---

## ขั้นตอนที่ 1425: Message Queue Pattern

```javascript
// message-queue.js
// Async message processing ด้วย Bull Queue

const Bull = require('bull');
const Redis = require('ioredis');

// สร้าง queues ต่างๆ
const emailQueue = new Bull('email', {
  redis: { host: 'localhost', port: 6379 },
  defaultJobOptions: {
    attempts: 3,              // Retry 3 ครั้ง
    backoff: {
      type: 'exponential',
      delay: 2000             // เริ่มจาก 2s, แล้วเพิ่มเป็น 2x
    },
    removeOnComplete: 100,    // เก็บ completed jobs ล่าสุด 100 รายการ
    removeOnFail: 50          // เก็บ failed jobs 50 รายการ
  }
});

const reportQueue = new Bull('report-generation', {
  redis: { host: 'localhost', port: 6379 }
});

// Email Queue Processor
emailQueue.process('welcome-email', 5, async (job) => {
  // Process สูงสุด 5 concurrent
  const { to, name, verificationToken } = job.data;
  
  console.log(`Processing welcome email for ${to} (Job ${job.id})`);
  
  // Update progress
  await job.progress(25);
  
  // ส่ง email (simulate)
  await sendEmail({
    to,
    subject: `Welcome, ${name}!`,
    template: 'welcome',
    variables: { name, verificationToken }
  });
  
  await job.progress(100);
  
  return { sent: true, to, timestamp: Date.now() };
});

// Report Queue Processor
reportQueue.process('generate-pdf', 2, async (job) => {
  const { reportId, userId, params } = job.data;
  
  await job.progress(10);
  
  // Fetch data
  const data = await fetchReportData(params);
  await job.progress(50);
  
  // Generate PDF
  const pdfBuffer = await generatePDF(data);
  await job.progress(80);
  
  // Upload to S3
  const url = await uploadToS3(pdfBuffer, `reports/${reportId}.pdf`);
  await job.progress(100);
  
  return { reportId, url, userId };
});

// Queue Event Listeners
emailQueue.on('completed', (job, result) => {
  console.log(`Email job ${job.id} completed:`, result);
});

emailQueue.on('failed', (job, err) => {
  console.error(`Email job ${job.id} failed:`, err.message);
  // ส่ง alert ไปยัง monitoring system
});

emailQueue.on('stalled', (job) => {
  console.warn(`Job ${job.id} stalled - will be retried`);
});

// API สำหรับ add jobs
const express = require('express');
const app = express();
app.use(express.json());

app.post('/api/send-welcome-email', async (req, res) => {
  const { to, name, verificationToken } = req.body;
  
  const job = await emailQueue.add('welcome-email', {
    to, name, verificationToken
  }, {
    priority: 1,      // สูงกว่า = ทำก่อน
    delay: 0          // ส่งทันที
  });
  
  res.json({ 
    success: true,
    jobId: job.id,
    message: 'Email queued successfully'
  });
});

app.get('/api/job-status/:jobId', async (req, res) => {
  const job = await emailQueue.getJob(req.params.jobId);
  
  if (!job) {
    return res.status(404).json({ error: 'Job not found' });
  }
  
  const state = await job.getState();
  const progress = job.progress();
  
  res.json({
    jobId: job.id,
    state,
    progress,
    data: job.data,
    result: job.returnvalue,
    failedReason: job.failedReason
  });
});

// Queue Dashboard
app.get('/api/queue-stats', async (req, res) => {
  const [waiting, active, completed, failed, delayed] = await Promise.all([
    emailQueue.getWaitingCount(),
    emailQueue.getActiveCount(),
    emailQueue.getCompletedCount(),
    emailQueue.getFailedCount(),
    emailQueue.getDelayedCount()
  ]);
  
  res.json({
    email: { waiting, active, completed, failed, delayed }
  });
});

// Simulate helper functions
async function sendEmail(options) {
  await new Promise(resolve => setTimeout(resolve, 100));
  console.log(`Email sent to ${options.to}`);
}

async function fetchReportData(params) {
  await new Promise(resolve => setTimeout(resolve, 500));
  return { data: 'report data' };
}

async function generatePDF(data) {
  await new Promise(resolve => setTimeout(resolve, 1000));
  return Buffer.from('PDF content');
}

async function uploadToS3(buffer, key) {
  await new Promise(resolve => setTimeout(resolve, 300));
  return `https://s3.amazonaws.com/bucket/${key}`;
}

module.exports = { emailQueue, reportQueue };
```

---

## ขั้นตอนที่ 1426: Circuit Breaker Pattern

```javascript
// circuit-breaker.js
// ป้องกัน cascading failures

class CircuitBreaker {
  constructor(options = {}) {
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.failureCount = 0;
    this.successCount = 0;
    this.lastFailureTime = null;
    
    this.options = {
      failureThreshold: options.failureThreshold || 5,
      successThreshold: options.successThreshold || 2,
      timeout: options.timeout || 60000, // 1 minute
      monitoringPeriod: options.monitoringPeriod || 10000
    };
    
    this.stats = {
      totalCalls: 0,
      successfulCalls: 0,
      failedCalls: 0,
      rejectedCalls: 0
    };
  }

  async call(fn, ...args) {
    this.stats.totalCalls++;
    
    if (this.state === 'OPEN') {
      // ตรวจสอบว่าถึงเวลา retry หรือยัง
      if (Date.now() - this.lastFailureTime >= this.options.timeout) {
        this.state = 'HALF_OPEN';
        console.log('Circuit breaker: OPEN -> HALF_OPEN');
      } else {
        this.stats.rejectedCalls++;
        throw new Error('Circuit breaker is OPEN - request rejected');
      }
    }
    
    try {
      const result = await fn(...args);
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  onSuccess() {
    this.stats.successfulCalls++;
    
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.options.successThreshold) {
        this.state = 'CLOSED';
        this.failureCount = 0;
        this.successCount = 0;
        console.log('Circuit breaker: HALF_OPEN -> CLOSED');
      }
    } else {
      this.failureCount = 0;
    }
  }

  onFailure() {
    this.stats.failedCalls++;
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.state === 'HALF_OPEN') {
      this.state = 'OPEN';
      console.log('Circuit breaker: HALF_OPEN -> OPEN');
    } else if (this.failureCount >= this.options.failureThreshold) {
      this.state = 'OPEN';
      console.log(`Circuit breaker: CLOSED -> OPEN (${this.failureCount} failures)`);
    }
  }

  getState() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      successCount: this.successCount,
      lastFailureTime: this.lastFailureTime,
      stats: this.stats,
      options: this.options
    };
  }
}

// ใช้ Circuit Breaker กับ external service calls
class ExternalServiceClient {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
    this.breaker = new CircuitBreaker({
      failureThreshold: 5,
      successThreshold: 2,
      timeout: 30000
    });
  }

  async fetchUser(userId) {
    return this.breaker.call(async () => {
      const response = await fetch(`${this.baseUrl}/users/${userId}`, {
        signal: AbortSignal.timeout(5000) // 5s timeout
      });
      
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }
      
      return response.json();
    });
  }

  async createOrder(orderData) {
    return this.breaker.call(async () => {
      const response = await fetch(`${this.baseUrl}/orders`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(orderData),
        signal: AbortSignal.timeout(10000)
      });
      
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }
      
      return response.json();
    });
  }

  getCircuitBreakerState() {
    return this.breaker.getState();
  }
}

// Express endpoint สำหรับ monitor circuit breaker
const express = require('express');
const app = express();

const paymentService = new ExternalServiceClient('https://payment.example.com');
const inventoryService = new ExternalServiceClient('https://inventory.example.com');

app.get('/circuit-breakers', (req, res) => {
  res.json({
    paymentService: paymentService.getCircuitBreakerState(),
    inventoryService: inventoryService.getCircuitBreakerState()
  });
});

module.exports = { CircuitBreaker, ExternalServiceClient };
```

---

## ขั้นตอนที่ 1427: Rate Limiting และ Throttling

```javascript
// rate-limiter.js
// Advanced rate limiting strategies

const Redis = require('ioredis');
const express = require('express');

const redis = new Redis();
const app = express();

// Algorithm 1: Fixed Window Counter
async function fixedWindowRateLimit(key, limit, windowMs) {
  const windowKey = `ratelimit:fixed:${key}:${Math.floor(Date.now() / windowMs)}`;
  
  const count = await redis.incr(windowKey);
  
  if (count === 1) {
    await redis.pexpire(windowKey, windowMs);
  }
  
  return {
    allowed: count <= limit,
    count,
    limit,
    resetTime: Math.floor(Date.now() / windowMs) * windowMs + windowMs
  };
}

// Algorithm 2: Sliding Window Log
async function slidingWindowRateLimit(key, limit, windowMs) {
  const now = Date.now();
  const windowStart = now - windowMs;
  const windowKey = `ratelimit:sliding:${key}`;
  
  const pipeline = redis.pipeline();
  pipeline.zremrangebyscore(windowKey, '-inf', windowStart); // Remove old entries
  pipeline.zadd(windowKey, now, `${now}`);                   // Add current request
  pipeline.zcard(windowKey);                                  // Count requests
  pipeline.pexpire(windowKey, windowMs);                      // Set expiry
  
  const results = await pipeline.exec();
  const count = results[2][1]; // zcard result
  
  return {
    allowed: count <= limit,
    count,
    limit,
    windowMs
  };
}

// Algorithm 3: Token Bucket
class TokenBucket {
  constructor(capacity, refillRate, redisClient) {
    this.capacity = capacity;
    this.refillRate = refillRate; // tokens per second
    this.redis = redisClient;
  }

  async consume(key, tokens = 1) {
    const bucketKey = `tokenbucket:${key}`;
    const now = Date.now() / 1000; // Convert to seconds
    
    const luaScript = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local refill_rate = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])
      local tokens_requested = tonumber(ARGV[4])
      
      local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
      local current_tokens = tonumber(bucket[1]) or capacity
      local last_refill = tonumber(bucket[2]) or now
      
      -- Calculate tokens to add
      local elapsed = now - last_refill
      local new_tokens = math.min(capacity, current_tokens + (elapsed * refill_rate))
      
      if new_tokens < tokens_requested then
        -- Not enough tokens
        redis.call('HMSET', key, 'tokens', new_tokens, 'last_refill', now)
        redis.call('EXPIRE', key, 3600)
        return {0, new_tokens}
      end
      
      -- Consume tokens
      new_tokens = new_tokens - tokens_requested
      redis.call('HMSET', key, 'tokens', new_tokens, 'last_refill', now)
      redis.call('EXPIRE', key, 3600)
      return {1, new_tokens}
    `;
    
    const result = await this.redis.eval(
      luaScript, 1, bucketKey,
      this.capacity, this.refillRate, now, tokens
    );
    
    return {
      allowed: result[0] === 1,
      remainingTokens: result[1],
      capacity: this.capacity
    };
  }
}

// Middleware Factory
function createRateLimiter(options = {}) {
  const {
    algorithm = 'sliding-window',
    limit = 100,
    windowMs = 60000,
    keyGenerator = (req) => req.ip,
    message = 'Too many requests',
    statusCode = 429,
    skipSuccessfulRequests = false
  } = options;

  const bucket = algorithm === 'token-bucket' ? 
    new TokenBucket(options.capacity || limit, options.refillRate || 10, redis) : null;

  return async (req, res, next) => {
    const key = keyGenerator(req);
    let result;
    
    try {
      switch (algorithm) {
        case 'fixed-window':
          result = await fixedWindowRateLimit(key, limit, windowMs);
          break;
        case 'sliding-window':
          result = await slidingWindowRateLimit(key, limit, windowMs);
          break;
        case 'token-bucket':
          result = await bucket.consume(key);
          break;
        default:
          result = await slidingWindowRateLimit(key, limit, windowMs);
      }
      
      // Set rate limit headers
      res.set({
        'X-RateLimit-Limit': limit,
        'X-RateLimit-Remaining': Math.max(0, limit - (result.count || 0)),
        'X-RateLimit-Reset': result.resetTime || Date.now() + windowMs
      });
      
      if (!result.allowed) {
        res.set('Retry-After', Math.ceil(windowMs / 1000));
        return res.status(statusCode).json({
          error: message,
          retryAfter: Math.ceil(windowMs / 1000)
        });
      }
      
      next();
    } catch (err) {
      // Fail open - ถ้า Redis down ให้ผ่านได้
      console.error('Rate limiter error:', err);
      next();
    }
  };
}

// Apply different rate limits to different routes
const globalLimiter = createRateLimiter({
  algorithm: 'sliding-window',
  limit: 1000,
  windowMs: 60000 // 1000 requests per minute
});

const authLimiter = createRateLimiter({
  algorithm: 'fixed-window',
  limit: 5,
  windowMs: 60000, // 5 login attempts per minute
  keyGenerator: (req) => `auth:${req.ip}:${req.body?.email || 'unknown'}`
});

const apiLimiter = createRateLimiter({
  algorithm: 'token-bucket',
  capacity: 100,
  refillRate: 10, // 10 tokens per second
  keyGenerator: (req) => `api:${req.headers['x-api-key'] || req.ip}`
});

app.use(globalLimiter);
app.post('/auth/login', authLimiter, (req, res) => {
  res.json({ message: 'Login endpoint' });
});
app.use('/api', apiLimiter);

module.exports = { createRateLimiter, CircuitBreaker, TokenBucket };
```

---

## ขั้นตอนที่ 1428: Caching Strategies

```javascript
// caching-strategies.js
// Multi-layer caching implementation

const Redis = require('ioredis');
const NodeCache = require('node-cache');

// Multi-Layer Cache
class MultiLayerCache {
  constructor() {
    // Layer 1: In-memory cache (fastest, smallest)
    this.l1Cache = new NodeCache({
      stdTTL: 60,       // 1 minute
      maxKeys: 1000,
      useClones: false
    });
    
    // Layer 2: Redis cache (fast, larger)
    this.l2Cache = new Redis({
      host: 'localhost',
      port: 6379,
      keyPrefix: 'cache:'
    });
    
    this.stats = {
      l1Hits: 0,
      l2Hits: 0,
      misses: 0,
      sets: 0
    };
  }

  async get(key) {
    // Check L1 (in-memory)
    const l1Value = this.l1Cache.get(key);
    if (l1Value !== undefined) {
      this.stats.l1Hits++;
      return { value: l1Value, layer: 'L1' };
    }
    
    // Check L2 (Redis)
    const l2Value = await this.l2Cache.get(key);
    if (l2Value !== null) {
      const parsed = JSON.parse(l2Value);
      
      // Populate L1 cache
      this.l1Cache.set(key, parsed, 60);
      
      this.stats.l2Hits++;
      return { value: parsed, layer: 'L2' };
    }
    
    this.stats.misses++;
    return null;
  }

  async set(key, value, ttlSeconds = 3600) {
    const serialized = JSON.stringify(value);
    
    // Set in both layers
    this.l1Cache.set(key, value, Math.min(ttlSeconds, 300)); // Max 5 min in L1
    await this.l2Cache.setex(key, ttlSeconds, serialized);
    
    this.stats.sets++;
  }

  async delete(key) {
    this.l1Cache.del(key);
    await this.l2Cache.del(key);
  }

  async invalidatePattern(pattern) {
    // Invalidate matching keys in Redis
    const keys = await this.l2Cache.keys(pattern);
    if (keys.length > 0) {
      await this.l2Cache.del(...keys);
    }
    
    // Clear all L1 (simpler approach)
    this.l1Cache.flushAll();
    
    return keys.length;
  }

  getStats() {
    const total = this.stats.l1Hits + this.stats.l2Hits + this.stats.misses;
    return {
      ...this.stats,
      hitRate: total > 0 ? ((this.stats.l1Hits + this.stats.l2Hits) / total * 100).toFixed(2) + '%' : '0%',
      l1HitRate: total > 0 ? (this.stats.l1Hits / total * 100).toFixed(2) + '%' : '0%'
    };
  }
}

// Cache-Aside Pattern
class CacheAsideRepository {
  constructor(db, cache) {
    this.db = db;
    this.cache = cache;
  }

  async findById(id) {
    const cacheKey = `user:${id}`;
    
    // Try cache first
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      return cached.value;
    }
    
    // Fetch from DB
    const user = await this.db.query('SELECT * FROM users WHERE id = $1', [id]);
    
    if (user.rows[0]) {
      // Store in cache
      await this.cache.set(cacheKey, user.rows[0], 3600);
    }
    
    return user.rows[0] || null;
  }

  async update(id, data) {
    // Update DB
    const result = await this.db.query(
      'UPDATE users SET data = $1 WHERE id = $2 RETURNING *',
      [data, id]
    );
    
    // Invalidate cache
    await this.cache.delete(`user:${id}`);
    
    return result.rows[0];
  }
}

// Write-Through Pattern
class WriteThroughCache {
  constructor(db, cache) {
    this.db = db;
    this.cache = cache;
  }

  async set(key, value) {
    // Write to both simultaneously
    await Promise.all([
      this.db.query('INSERT INTO cache_store (key, value) VALUES ($1, $2) ON CONFLICT (key) DO UPDATE SET value = $2', [key, JSON.stringify(value)]),
      this.cache.set(key, value)
    ]);
  }
}

module.exports = { MultiLayerCache, CacheAsideRepository };
```

---

## ขั้นตอนที่ 1429: API Gateway Pattern

```javascript
// api-gateway.js
// API Gateway implementation ด้วย Express

const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');
const jwt = require('jsonwebtoken');
const rateLimit = require('express-rate-limit');

const app = express();

// Service Registry
const services = {
  users: {
    url: process.env.USER_SERVICE_URL || 'http://user-service:3001',
    routes: ['/users', '/auth']
  },
  orders: {
    url: process.env.ORDER_SERVICE_URL || 'http://order-service:3002',
    routes: ['/orders']
  },
  products: {
    url: process.env.PRODUCT_SERVICE_URL || 'http://product-service:3003',
    routes: ['/products', '/categories']
  },
  notifications: {
    url: process.env.NOTIFICATION_SERVICE_URL || 'http://notification-service:3004',
    routes: ['/notifications']
  }
};

// Middleware: Request ID
app.use((req, res, next) => {
  req.requestId = require('crypto').randomUUID();
  res.set('X-Request-ID', req.requestId);
  next();
});

// Middleware: Authentication
const authenticate = async (req, res, next) => {
  const publicRoutes = ['/auth/login', '/auth/register', '/health'];
  
  if (publicRoutes.some(route => req.path.startsWith(route))) {
    return next();
  }
  
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// Middleware: Logging
const requestLogger = (req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(JSON.stringify({
      requestId: req.requestId,
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      userId: req.user?.id,
      userAgent: req.headers['user-agent']
    }));
  });
  
  next();
};

app.use(requestLogger);
app.use(authenticate);

// Rate Limiting per service
const rateLimiters = {
  users: rateLimit({ windowMs: 60000, max: 100 }),
  orders: rateLimit({ windowMs: 60000, max: 50 }),
  products: rateLimit({ windowMs: 60000, max: 200 })
};

// Dynamic route registration
Object.entries(services).forEach(([serviceName, service]) => {
  service.routes.forEach(route => {
    const limiter = rateLimiters[serviceName];
    
    const middlewares = [
      limiter,
      createProxyMiddleware({
        target: service.url,
        changeOrigin: true,
        on: {
          proxyReq: (proxyReq, req) => {
            // Forward user info to services
            if (req.user) {
              proxyReq.setHeader('X-User-ID', req.user.id);
              proxyReq.setHeader('X-User-Role', req.user.role);
            }
            proxyReq.setHeader('X-Request-ID', req.requestId);
          },
          error: (err, req, res) => {
            console.error(`Proxy error for ${serviceName}:`, err.message);
            res.status(503).json({ error: 'Service temporarily unavailable' });
          }
        }
      })
    ].filter(Boolean);
    
    app.use(route, ...middlewares);
  });
});

// Health Check
app.get('/health', async (req, res) => {
  const healthChecks = await Promise.allSettled(
    Object.entries(services).map(async ([name, service]) => {
      const response = await fetch(`${service.url}/health`, {
        signal: AbortSignal.timeout(5000)
      });
      return { name, healthy: response.ok };
    })
  );
  
  const serviceHealth = healthChecks.map((result, i) => ({
    service: Object.keys(services)[i],
    healthy: result.status === 'fulfilled' && result.value.healthy
  }));
  
  const allHealthy = serviceHealth.every(s => s.healthy);
  
  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'healthy' : 'degraded',
    services: serviceHealth,
    timestamp: new Date().toISOString()
  });
});

app.listen(3000, () => {
  console.log('API Gateway running on port 3000');
});

module.exports = app;
```

---

## ขั้นตอนที่ 1430: Monitoring และ Observability

```javascript
// observability.js
// Three pillars: Metrics, Logging, Tracing

const express = require('express');
const promClient = require('prom-client');
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { SimpleSpanProcessor } = require('@opentelemetry/sdk-trace-base');

const app = express();

// ========================================
// METRICS - Prometheus
// ========================================

// สร้าง custom metrics
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5]
});

const httpRequestTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

const activeConnections = new promClient.Gauge({
  name: 'active_connections',
  help: 'Number of active connections'
});

const dbQueryDuration = new promClient.Histogram({
  name: 'db_query_duration_seconds',
  help: 'Duration of database queries',
  labelNames: ['operation', 'table'],
  buckets: [0.001, 0.01, 0.1, 1, 5]
});

// HTTP metrics middleware
app.use((req, res, next) => {
  const start = Date.now();
  activeConnections.inc();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const route = req.route?.path || req.path;
    
    httpRequestDuration
      .labels(req.method, route, res.statusCode.toString())
      .observe(duration);
    
    httpRequestTotal
      .labels(req.method, route, res.statusCode.toString())
      .inc();
    
    activeConnections.dec();
  });
  
  next();
});

// Metrics endpoint สำหรับ Prometheus scrape
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});

// ========================================
// STRUCTURED LOGGING
// ========================================

const winston = require('winston');
const { format, transports } = winston;

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: format.combine(
    format.timestamp(),
    format.errors({ stack: true }),
    format.json()
  ),
  defaultMeta: {
    service: 'my-api',
    version: process.env.npm_package_version || '1.0.0',
    environment: process.env.NODE_ENV || 'development'
  },
  transports: [
    new transports.Console({
      format: process.env.NODE_ENV === 'development'
        ? format.combine(format.colorize(), format.simple())
        : format.json()
    }),
    new transports.File({ filename: 'logs/error.log', level: 'error' }),
    new transports.File({ filename: 'logs/combined.log' })
  ]
});

// Request logging middleware
app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    logger.info('HTTP Request', {
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration: Date.now() - start,
      requestId: req.headers['x-request-id'],
      userId: req.user?.id,
      ip: req.ip
    });
  });
  
  next();
});

// ========================================
// DISTRIBUTED TRACING
// ========================================

const { context, trace } = require('@opentelemetry/api');

function createTracer(serviceName) {
  const provider = new NodeTracerProvider();
  provider.register();
  return trace.getTracer(serviceName);
}

const tracer = createTracer('my-api');

// Trace middleware
app.use((req, res, next) => {
  const span = tracer.startSpan(`${req.method} ${req.path}`);
  
  span.setAttributes({
    'http.method': req.method,
    'http.url': req.url,
    'http.user_agent': req.headers['user-agent'] || ''
  });
  
  req.span = span;
  
  res.on('finish', () => {
    span.setAttribute('http.status_code', res.statusCode);
    span.end();
  });
  
  next();
});

// Dashboard endpoint
app.get('/dashboard', (req, res) => {
  res.json({
    metrics: 'Available at /metrics',
    logging: 'Structured JSON logs',
    tracing: 'OpenTelemetry traces',
    health: '/health'
  });
});

module.exports = { app, logger, tracer };
```

---

## ขั้นตอนที่ 1431-1440: Advanced System Design Concepts

### Event Sourcing Pattern

```javascript
// event-sourcing.js
// เก็บ history ของทุก state change

class EventStore {
  constructor(db) {
    this.db = db;
  }

  async append(streamId, events, expectedVersion = null) {
    return this.db.transaction(async (client) => {
      // ตรวจสอบ optimistic concurrency
      if (expectedVersion !== null) {
        const current = await client.query(
          'SELECT MAX(version) as version FROM events WHERE stream_id = $1',
          [streamId]
        );
        
        const currentVersion = current.rows[0]?.version || 0;
        if (currentVersion !== expectedVersion) {
          throw new Error(`Concurrency conflict: expected ${expectedVersion}, got ${currentVersion}`);
        }
      }

      // Append events
      const results = [];
      for (let i = 0; i < events.length; i++) {
        const event = events[i];
        const result = await client.query(
          `INSERT INTO events (stream_id, type, data, metadata, version, created_at)
           VALUES ($1, $2, $3, $4, 
                  (SELECT COALESCE(MAX(version), 0) + 1 FROM events WHERE stream_id = $1),
                  NOW()) RETURNING *`,
          [streamId, event.type, JSON.stringify(event.data), JSON.stringify(event.metadata || {})]
        );
        results.push(result.rows[0]);
      }

      return results;
    });
  }

  async read(streamId, fromVersion = 0) {
    const result = await this.db.query(
      'SELECT * FROM events WHERE stream_id = $1 AND version > $2 ORDER BY version',
      [streamId, fromVersion]
    );
    
    return result.rows.map(row => ({
      id: row.id,
      streamId: row.stream_id,
      type: row.type,
      data: row.data,
      metadata: row.metadata,
      version: row.version,
      createdAt: row.created_at
    }));
  }
}

// Order Aggregate ด้วย Event Sourcing
class Order {
  constructor(id) {
    this.id = id;
    this.status = null;
    this.items = [];
    this.total = 0;
    this.version = 0;
    this.pendingEvents = [];
  }

  // Apply events to rebuild state
  apply(event) {
    switch (event.type) {
      case 'OrderCreated':
        this.status = 'pending';
        this.items = event.data.items;
        this.total = event.data.total;
        break;
      
      case 'OrderConfirmed':
        this.status = 'confirmed';
        break;
      
      case 'OrderShipped':
        this.status = 'shipped';
        this.trackingNumber = event.data.trackingNumber;
        break;
      
      case 'OrderCancelled':
        this.status = 'cancelled';
        this.cancelReason = event.data.reason;
        break;
    }
    
    this.version = event.version || this.version + 1;
  }

  // Commands that produce events
  create(items, total) {
    if (this.status !== null) throw new Error('Order already created');
    
    const event = {
      type: 'OrderCreated',
      data: { items, total, orderId: this.id }
    };
    
    this.pendingEvents.push(event);
    this.apply(event);
    return this;
  }

  confirm() {
    if (this.status !== 'pending') throw new Error('Can only confirm pending orders');
    
    const event = { type: 'OrderConfirmed', data: { orderId: this.id } };
    this.pendingEvents.push(event);
    this.apply(event);
    return this;
  }

  ship(trackingNumber) {
    if (this.status !== 'confirmed') throw new Error('Can only ship confirmed orders');
    
    const event = {
      type: 'OrderShipped',
      data: { orderId: this.id, trackingNumber }
    };
    
    this.pendingEvents.push(event);
    this.apply(event);
    return this;
  }

  // Rebuild from event history
  static reconstitute(id, events) {
    const order = new Order(id);
    events.forEach(event => order.apply(event));
    order.pendingEvents = [];
    return order;
  }
}

// Order Repository
class OrderRepository {
  constructor(eventStore) {
    this.eventStore = eventStore;
  }

  async save(order) {
    if (order.pendingEvents.length === 0) return;
    
    await this.eventStore.append(
      `order-${order.id}`,
      order.pendingEvents,
      order.version - order.pendingEvents.length
    );
    
    order.pendingEvents = [];
  }

  async findById(orderId) {
    const events = await this.eventStore.read(`order-${orderId}`);
    
    if (events.length === 0) return null;
    
    return Order.reconstitute(orderId, events);
  }
}

module.exports = { EventStore, Order, OrderRepository };
```

---

## แบบฝึกหัด

### Exercise 1: Implement Bulkhead Pattern
สร้าง middleware ที่แบ่ง connection pools ออกตาม service type เพื่อป้องกัน resource exhaustion

### Exercise 2: Design a URL Shortener System
ออกแบบ URL shortener ที่รองรับ 1 billion URLs และ 100M requests/day โดยพิจารณา:
- Hash function
- Database choice
- Caching strategy
- Rate limiting

### Exercise 3: Build a Real-time Leaderboard
สร้าง leaderboard ที่อัพเดทแบบ real-time โดยใช้:
- Redis Sorted Sets
- WebSocket for push updates
- Pagination

### คำถามทบทวน
1. เมื่อใดควรเลือก AP system แทน CP system?
2. ความแตกต่างระหว่าง horizontal และ vertical scaling คืออะไร?
3. Circuit breaker มีกี่ state และ transition เกิดขึ้นเมื่อใด?
4. อธิบาย CAP theorem และยกตัวอย่าง database ที่เลือก CP และ AP

---

*ต่อไป: Part 82 - High Availability Architecture*
