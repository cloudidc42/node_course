# Part 43 | ขั้นตอนที่ 761-780 จาก 1000

## Monitoring และ Performance - การตรวจสอบและเพิ่มประสิทธิภาพ

---

## สารบัญ

1. [PM2 Process Management](#pm2-process-management)
2. [Application Logging](#application-logging)
3. [Prometheus Metrics](#prometheus-metrics)
4. [Grafana Dashboard](#grafana-dashboard)
5. [Performance Profiling](#performance-profiling)
6. [Memory Leak Detection](#memory-leak-detection)
7. [Database Performance](#database-performance)
8. [APM Tools](#apm-tools)
9. [Alerting](#alerting)
10. [แบบฝึกหัด](#แบบฝึกหัด)

---

## PM2 Process Management

### ขั้นตอนที่ 761: PM2 Advanced Configuration

```javascript
// ecosystem.config.js - Production configuration
module.exports = {
  apps: [
    {
      name: 'myapp-api',
      script: 'src/index.js',

      // Cluster mode - ใช้ทุก CPU cores
      instances: 'max',
      exec_mode: 'cluster',

      // Environment
      env: {
        NODE_ENV: 'development',
        PORT: 3000,
      },
      env_production: {
        NODE_ENV: 'production',
        PORT: 3000,
        NODE_OPTIONS: '--max-old-space-size=512',
      },

      // Logs
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
      error_file: '/var/log/myapp/error.log',
      out_file: '/var/log/myapp/out.log',
      merge_logs: true,
      log_type: 'json',

      // Monitoring
      max_memory_restart: '512M',
      min_uptime: '10s',
      max_restarts: 10,
      restart_delay: 4000,

      // Watch (development only)
      watch: false,
      ignore_watch: ['node_modules', 'logs', 'coverage'],

      // Signal handling
      kill_timeout: 3000,
      wait_ready: true,
      listen_timeout: 10000,
    },

    // Background worker
    {
      name: 'worker',
      script: 'src/workers/index.js',
      instances: 2,
      exec_mode: 'cluster',
      env_production: {
        NODE_ENV: 'production',
        WORKER_CONCURRENCY: 5,
      },
    },

    // Cron scheduler
    {
      name: 'scheduler',
      script: 'src/scheduler.js',
      instances: 1,  // Scheduler ควรรัน instance เดียว
      exec_mode: 'fork',
    },
  ],
};
```

### ขั้นตอนที่ 762: PM2 Commands และ Monitoring

```bash
# Start application
pm2 start ecosystem.config.js --env production

# Status
pm2 status
pm2 ls

# ดู logs realtime
pm2 logs
pm2 logs myapp-api --lines 100

# Monitor realtime (CPU, Memory, etc.)
pm2 monit

# Restart
pm2 restart myapp-api
pm2 reload myapp-api   # Zero-downtime reload

# Scale
pm2 scale myapp-api 4  # เพิ่มเป็น 4 instances

# Stop/Delete
pm2 stop all
pm2 delete all

# Save configuration
pm2 save

# Setup startup script
pm2 startup
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u $USER --hp $HOME

# PM2 Plus (Dashboard)
pm2 link <secret_key> <public_key>
```

### ขั้นตอนที่ 763: Graceful Reload

```javascript
// src/index.js - PM2 ready signal
const app = express();

async function start() {
  await connectDatabase();
  await connectRedis();

  const server = app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);

    // แจ้ง PM2 ว่า app พร้อมแล้ว
    if (process.send) {
      process.send('ready');
    }
  });

  // Graceful shutdown สำหรับ PM2
  process.on('SIGINT', async () => {
    console.log('SIGINT received, shutting down gracefully...');

    server.close(async () => {
      await mongoose.disconnect();
      await redis.quit();
      process.exit(0);
    });

    setTimeout(() => {
      console.error('Forced exit after timeout');
      process.exit(1);
    }, 10000);
  });
}

start().catch(console.error);
```

---

## Application Logging

### ขั้นตอนที่ 764: Winston Logging Setup

```javascript
// src/config/logger.js
const winston = require('winston');
const DailyRotateFile = require('winston-daily-rotate-file');

// Log format
const logFormat = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
  winston.format.errors({ stack: true }),
  winston.format.splat(),
  winston.format.json()
);

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || (process.env.NODE_ENV === 'production' ? 'warn' : 'debug'),
  format: logFormat,

  defaultMeta: {
    service: process.env.SERVICE_NAME || 'myapp',
    version: process.env.APP_VERSION || '1.0.0',
    environment: process.env.NODE_ENV,
    pid: process.pid,
  },

  transports: [
    // Console (development)
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.printf(({ level, message, timestamp, ...meta }) => {
          const metaStr = Object.keys(meta).length
            ? `\n${JSON.stringify(meta, null, 2)}`
            : '';
          return `${timestamp} [${level}] ${message}${metaStr}`;
        })
      ),
    }),

    // File - all logs
    new DailyRotateFile({
      filename: 'logs/application-%DATE%.log',
      datePattern: 'YYYY-MM-DD',
      zippedArchive: true,
      maxSize: '20m',
      maxFiles: '14d',
    }),

    // File - error logs only
    new DailyRotateFile({
      level: 'error',
      filename: 'logs/error-%DATE%.log',
      datePattern: 'YYYY-MM-DD',
      zippedArchive: true,
      maxSize: '20m',
      maxFiles: '30d',
    }),
  ],
});

// Stream สำหรับ Morgan HTTP logger
logger.stream = {
  write: (message) => logger.info(message.trim()),
};

module.exports = logger;
```

### ขั้นตอนที่ 765: Structured Logging

```javascript
// src/middleware/requestLogger.js
const logger = require('../config/logger');

function requestLogger(req, res, next) {
  const requestId = req.headers['x-request-id'] || generateId();
  req.requestId = requestId;
  req.startTime = Date.now();

  // Log request
  logger.info('Incoming request', {
    requestId,
    method: req.method,
    url: req.originalUrl,
    ip: req.ip,
    userAgent: req.headers['user-agent'],
    userId: req.user?.id,
  });

  // Override res.json เพื่อ log response
  const originalJson = res.json;
  res.json = function(body) {
    const duration = Date.now() - req.startTime;

    logger.info('Request completed', {
      requestId,
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      responseSize: JSON.stringify(body).length,
    });

    // Log slow requests
    if (duration > 1000) {
      logger.warn('Slow request detected', {
        requestId,
        url: req.originalUrl,
        duration: `${duration}ms`,
      });
    }

    return originalJson.call(this, body);
  };

  res.setHeader('X-Request-ID', requestId);
  next();
}

function generateId() {
  return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
}

module.exports = requestLogger;
```

---

## Prometheus Metrics

### ขั้นตอนที่ 766: ติดตั้ง Prometheus

```bash
npm install prom-client
```

```javascript
// src/config/metrics.js
const client = require('prom-client');

// เปิด default metrics (CPU, Memory, GC, etc.)
const collectDefaultMetrics = client.collectDefaultMetrics;
collectDefaultMetrics({
  prefix: 'myapp_',
  gcDurationBuckets: [0.001, 0.01, 0.1, 1, 2, 5],
  register: client.register,
});

// Custom metrics

// HTTP Request counter
const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
});

// HTTP Request duration
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route'],
  buckets: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
});

// Active connections
const activeConnections = new client.Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
});

// Business metrics
const userRegistrations = new client.Counter({
  name: 'user_registrations_total',
  help: 'Total number of user registrations',
  labelNames: ['plan'],
});

const activeUsers = new client.Gauge({
  name: 'active_users',
  help: 'Number of currently active users',
});

// Database query duration
const dbQueryDuration = new client.Histogram({
  name: 'db_query_duration_seconds',
  help: 'Database query duration in seconds',
  labelNames: ['operation', 'collection'],
  buckets: [0.001, 0.01, 0.1, 0.5, 1, 5],
});

// Queue size
const queueSize = new client.Gauge({
  name: 'queue_size',
  help: 'Current queue size',
  labelNames: ['queue_name'],
});

module.exports = {
  client,
  httpRequestsTotal,
  httpRequestDuration,
  activeConnections,
  userRegistrations,
  activeUsers,
  dbQueryDuration,
  queueSize,
};
```

### ขั้นตอนที่ 767: Metrics Middleware

```javascript
// src/middleware/metrics.js
const { httpRequestsTotal, httpRequestDuration } = require('../config/metrics');

function metricsMiddleware(req, res, next) {
  const start = Date.now();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const route = req.route?.path || req.path || 'unknown';
    const method = req.method;
    const statusCode = res.statusCode;

    // Increment counter
    httpRequestsTotal.inc({
      method,
      route: normalizeRoute(route),
      status_code: statusCode,
    });

    // Record duration
    httpRequestDuration.observe({
      method,
      route: normalizeRoute(route),
    }, duration);
  });

  next();
}

// Normalize dynamic routes
function normalizeRoute(route) {
  return route
    .replace(/\/[0-9a-fA-F]{24}/g, '/:id')  // MongoDB ObjectId
    .replace(/\/\d+/g, '/:id')               // Numeric IDs
    .replace(/\/[0-9a-f-]{36}/g, '/:uuid')   // UUIDs
    .substring(0, 100);  // จำกัดความยาว
}

module.exports = metricsMiddleware;
```

```javascript
// src/routes/metrics.js - Metrics endpoint
const express = require('express');
const { client } = require('../config/metrics');

const router = express.Router();

// เฉพาะ internal access
router.get('/metrics', async (req, res) => {
  try {
    res.set('Content-Type', client.register.contentType);
    res.end(await client.register.metrics());
  } catch (error) {
    res.status(500).send(error.message);
  }
});

module.exports = router;
```

---

## Grafana Dashboard

### ขั้นตอนที่ 768: Docker Compose สำหรับ Monitoring Stack

```yaml
# monitoring/docker-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/config.yml:/etc/alertmanager/config.yml
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:
```

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    environment: production

rule_files:
  - 'alerts.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: 'nodejs-app'
    static_configs:
      - targets: ['app:3000']
    metrics_path: '/metrics'

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'mongodb'
    static_configs:
      - targets: ['mongodb-exporter:9216']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
```

---

## Performance Profiling

### ขั้นตอนที่ 769: CPU Profiling

```javascript
// src/profiling/cpu.js
const v8Profiler = require('v8-profiler-next');
const fs = require('fs');
const path = require('path');

// Start CPU profiling
function startCPUProfile(duration = 10000) {
  const profileId = `profile_${Date.now()}`;
  v8Profiler.startProfiling(profileId, true);
  console.log(`CPU profiling started: ${profileId}`);

  return new Promise((resolve) => {
    setTimeout(() => {
      const profile = v8Profiler.stopProfiling(profileId);
      profile.export((error, result) => {
        if (error) {
          console.error('Profiling error:', error);
          resolve(null);
          return;
        }

        const filePath = path.join('/tmp', `${profileId}.cpuprofile`);
        fs.writeFileSync(filePath, result);
        profile.delete();

        console.log(`CPU profile saved: ${filePath}`);
        resolve(filePath);
      });
    }, duration);
  });
}

// Route สำหรับ trigger profiling (dev only)
if (process.env.NODE_ENV !== 'production') {
  app.post('/debug/profile', async (req, res) => {
    const { duration = 10000 } = req.body;
    const filePath = await startCPUProfile(duration);
    if (filePath) {
      res.download(filePath);
    } else {
      res.status(500).json({ error: 'Profiling failed' });
    }
  });
}
```

### ขั้นตอนที่ 770: Heap Snapshots

```javascript
// src/profiling/heap.js
const v8 = require('v8');
const fs = require('fs');
const path = require('path');

function takeHeapSnapshot() {
  const snapshotStream = v8.writeHeapSnapshot();
  console.log(`Heap snapshot saved: ${snapshotStream}`);
  return snapshotStream;
}

// Memory statistics
function getMemoryStats() {
  const usage = process.memoryUsage();
  const heapStats = v8.getHeapStatistics();

  return {
    rss: formatBytes(usage.rss),
    heapTotal: formatBytes(usage.heapTotal),
    heapUsed: formatBytes(usage.heapUsed),
    external: formatBytes(usage.external),
    heapUsedPercent: ((usage.heapUsed / usage.heapTotal) * 100).toFixed(2) + '%',
    // Heap statistics
    totalHeapSize: formatBytes(heapStats.total_heap_size),
    usedHeapSize: formatBytes(heapStats.used_heap_size),
    heapSizeLimit: formatBytes(heapStats.heap_size_limit),
    mallocedMemory: formatBytes(heapStats.malloced_memory),
  };
}

function formatBytes(bytes) {
  return `${(bytes / 1024 / 1024).toFixed(2)} MB`;
}

// Monitor memory over time
function startMemoryMonitor(interval = 60000) {
  setInterval(() => {
    const stats = getMemoryStats();

    if (parseFloat(stats.heapUsedPercent) > 80) {
      console.warn('⚠️ High memory usage:', stats);
      // ส่ง alert
    }

    console.log('Memory stats:', stats);
  }, interval);
}

module.exports = { takeHeapSnapshot, getMemoryStats, startMemoryMonitor };
```

### ขั้นตอนที่ 771: Node.js Performance Hooks

```javascript
// src/monitoring/performance.js
const { performance, PerformanceObserver } = require('perf_hooks');
const { dbQueryDuration } = require('../config/metrics');

// Measure async operations
async function measureAsync(name, fn) {
  const start = performance.now();
  try {
    return await fn();
  } finally {
    const duration = performance.now() - start;
    console.log(`${name}: ${duration.toFixed(2)}ms`);
  }
}

// Wrap Mongoose queries สำหรับ monitoring
function instrumentMongoose(mongoose) {
  mongoose.plugin((schema) => {
    schema.pre(/^find/, function() {
      this._startTime = Date.now();
    });

    schema.post(/^find/, function(result) {
      if (this._startTime) {
        const duration = (Date.now() - this._startTime) / 1000;
        const collection = this.model.collection.name;

        dbQueryDuration.observe(
          { operation: 'find', collection },
          duration
        );

        if (duration > 1) {
          console.warn(`Slow query on ${collection}: ${duration.toFixed(3)}s`);
        }
      }
    });

    schema.pre('save', function() {
      this._saveStartTime = Date.now();
    });

    schema.post('save', function() {
      if (this._saveStartTime) {
        const duration = (Date.now() - this._saveStartTime) / 1000;
        dbQueryDuration.observe(
          { operation: 'save', collection: this.constructor.collection.name },
          duration
        );
      }
    });
  });
}

// PerformanceObserver สำหรับ async hooks
const obs = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  entries.forEach((entry) => {
    if (entry.duration > 100) {
      console.warn(`Slow operation: ${entry.name} took ${entry.duration.toFixed(2)}ms`);
    }
  });
});

obs.observe({ type: 'measure', buffered: true });

module.exports = { measureAsync, instrumentMongoose };
```

---

## Memory Leak Detection

### ขั้นตอนที่ 772: Memory Leak Patterns

```javascript
// ❌ Memory leaks ที่พบบ่อย

// 1. Event listeners ไม่ถูก remove
function badCode() {
  const emitter = new EventEmitter();
  // ❌ ทุก request เพิ่ม listener แต่ไม่ remove
  emitter.on('data', handleData);
}

function goodCode() {
  const emitter = new EventEmitter();
  // ✅ เก็บ reference และ remove เมื่อเสร็จ
  const handler = (data) => handleData(data);
  emitter.on('data', handler);

  // Cleanup
  return () => emitter.off('data', handler);
}

// 2. Closure ที่เก็บ reference ใหญ่
// ❌
function createHandler(largeData) {
  return function() {
    console.log(largeData.length); // เก็บ largeData ใน closure
  };
}

// ✅
function createHandler(dataLength) {
  const len = dataLength; // เก็บเฉพาะที่ต้องการ
  return function() {
    console.log(len);
  };
}

// 3. Timer ไม่ถูก clear
// ❌
function badTimer() {
  setInterval(() => {
    // ทำงานตลอดไป
  }, 1000);
}

// ✅
function goodTimer() {
  const timer = setInterval(() => {
    // ทำงาน
  }, 1000);

  // Clear เมื่อไม่ต้องการ
  process.on('SIGTERM', () => clearInterval(timer));
}

// 4. Cache ไม่มี size limit
// ❌
const cache = {};
app.use((req, res, next) => {
  cache[req.url] = response; // เพิ่มตลอด ไม่มี eviction
});

// ✅
const LRU = require('lru-cache');
const cache = new LRU({ max: 500, ttl: 1000 * 60 * 5 });
```

### ขั้นตอนที่ 773: Memory Monitoring

```javascript
// src/monitoring/memoryMonitor.js
const { Gauge } = require('prom-client');

const heapUsed = new Gauge({
  name: 'nodejs_heap_used_bytes',
  help: 'Process heap used size',
});

const heapTotal = new Gauge({
  name: 'nodejs_heap_total_bytes',
  help: 'Process heap total size',
});

const externalMemory = new Gauge({
  name: 'nodejs_external_memory_bytes',
  help: 'Process external memory',
});

function startMemoryTracking() {
  setInterval(() => {
    const usage = process.memoryUsage();

    heapUsed.set(usage.heapUsed);
    heapTotal.set(usage.heapTotal);
    externalMemory.set(usage.external);

    // Alert on high memory
    const heapPercent = (usage.heapUsed / usage.heapTotal) * 100;
    if (heapPercent > 90) {
      console.error(`🚨 Critical memory usage: ${heapPercent.toFixed(1)}%`);
      // ทำ heap snapshot
      require('./heap').takeHeapSnapshot();
    } else if (heapPercent > 75) {
      console.warn(`⚠️ High memory usage: ${heapPercent.toFixed(1)}%`);
    }
  }, 30000); // ทุก 30 วินาที
}

module.exports = { startMemoryTracking };
```

---

## Database Performance

### ขั้นตอนที่ 774: MongoDB Performance Optimization

```javascript
// MongoDB query optimization

// 1. ใช้ Indexes อย่างถูกต้อง
const userSchema = new mongoose.Schema({
  email: { type: String, index: true, unique: true },
  username: { type: String, index: true },
  role: { type: String, index: true },
  createdAt: { type: Date, index: true },
});

// Compound index
userSchema.index({ role: 1, createdAt: -1 });
userSchema.index({ username: 'text', email: 'text' }); // Text search

// 2. Explain queries
const explain = await User.find({ role: 'ADMIN' }).explain('executionStats');
console.log('Index used:', explain.executionStats.executionStages.inputStage?.indexName);
console.log('Documents examined:', explain.executionStats.totalDocsExamined);
console.log('Documents returned:', explain.executionStats.nReturned);

// 3. Projection - ดึงเฉพาะ fields ที่ต้องการ
// ❌ ดึงทุก fields
const users = await User.find({ role: 'ADMIN' });

// ✅ ดึงเฉพาะที่ต้องการ
const users = await User.find({ role: 'ADMIN' }).select('username email role');

// 4. Pagination อย่างถูกต้อง
// ❌ skip() กับ large datasets ช้ามาก
const page1 = await Post.find().skip(10000).limit(10);

// ✅ Cursor-based pagination
const page2 = await Post.find({ _id: { $gt: lastId } }).limit(10);

// 5. Lean queries (Plain JS objects แทน Mongoose documents)
const users = await User.find().lean(); // เร็วกว่า 10x สำหรับ read-only

// 6. Aggregation pipeline
const stats = await Order.aggregate([
  { $match: { status: 'COMPLETED' } },
  { $group: {
    _id: '$userId',
    totalOrders: { $sum: 1 },
    totalAmount: { $sum: '$amount' },
  }},
  { $sort: { totalAmount: -1 } },
  { $limit: 10 },
]);
```

### ขั้นตอนที่ 775: Connection Pool Tuning

```javascript
// MongoDB connection pool
mongoose.connect(process.env.MONGODB_URI, {
  maxPoolSize: 10,         // Maximum connections ใน pool
  minPoolSize: 2,          // Minimum connections
  maxIdleTimeMS: 30000,    // Close connections idle > 30s
  connectionTimeoutMS: 10000,
  socketTimeoutMS: 45000,
});

// Monitor connection pool
mongoose.connection.on('connected', () => console.log('MongoDB connected'));
mongoose.connection.on('disconnected', () => console.log('MongoDB disconnected'));
mongoose.connection.on('reconnected', () => console.log('MongoDB reconnected'));

// ดู pool stats
setInterval(() => {
  const { poolSize, waitingConnections, activeConnections } = mongoose.connection.client.topology;
  console.log('MongoDB Pool:', { poolSize, activeConnections });
}, 60000);
```

---

## APM Tools

### ขั้นตอนที่ 776: New Relic Integration

```javascript
// ติดตั้ง: npm install newrelic
// ต้องอยู่ก่อน imports อื่นๆ ทั้งหมด!

// newrelic.js (config)
exports.config = {
  app_name: ['My Node App'],
  license_key: process.env.NEW_RELIC_LICENSE_KEY,
  logging: {
    level: 'info',
  },
  allow_all_headers: true,
  attributes: {
    exclude: ['request.headers.cookie', 'request.headers.authorization'],
  },
  transaction_tracer: {
    enabled: true,
    transaction_threshold: 100, // ms
  },
  error_collector: {
    enabled: true,
  },
  browser_monitoring: {
    enable: true,
  },
};
```

### ขั้นตอนที่ 777: Datadog APM

```javascript
// src/tracing/datadog.js
const tracer = require('dd-trace').init({
  service: 'myapp',
  env: process.env.NODE_ENV,
  version: process.env.APP_VERSION,
  logInjection: true,
  analytics: true,
  runtimeMetrics: true,
  profiling: true,
});

// Custom spans
async function processUser(userId) {
  const span = tracer.startSpan('processUser');
  span.setTag('userId', userId);

  try {
    const user = await User.findById(userId);
    span.setTag('found', !!user);
    return user;
  } catch (error) {
    span.setTag('error', error.message);
    throw error;
  } finally {
    span.finish();
  }
}

module.exports = tracer;
```

---

## Alerting

### ขั้นตอนที่ 778: Prometheus Alerts

```yaml
# prometheus/alerts.yml
groups:
  - name: nodejs-app
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) /
          sum(rate(http_requests_total[5m])) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      # High response time
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High API latency"
          description: "95th percentile latency is {{ $value }}s"

      # High memory usage
      - alert: HighMemoryUsage
        expr: |
          (nodejs_heap_used_bytes / nodejs_heap_total_bytes) > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Heap usage is {{ $value | humanizePercentage }}"

      # Instance down
      - alert: InstanceDown
        expr: up{job="nodejs-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Node.js app is down"
          description: "Instance {{ $labels.instance }} is down"
```

### ขั้นตอนที่ 779: Alertmanager Configuration

```yaml
# alertmanager/config.yml
global:
  resolve_timeout: 5m
  slack_api_url: '${SLACK_WEBHOOK}'

route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    - match:
        severity: critical
      receiver: 'critical-alerts'
      repeat_interval: 1h

    - match:
        severity: warning
      receiver: 'warning-alerts'

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'critical-alerts'
    slack_configs:
      - channel: '#critical-alerts'
        color: 'danger'
        title: '🚨 CRITICAL: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    pagerduty_configs:
      - routing_key: '${PAGERDUTY_KEY}'

  - name: 'warning-alerts'
    slack_configs:
      - channel: '#alerts'
        color: 'warning'
        title: '⚠️ WARNING: {{ .GroupLabels.alertname }}'
```

---

## Load Testing

### ขั้นตอนที่ 780: k6 Load Testing

```javascript
// tests/load/api-test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Counter, Trend, Rate } from 'k6/metrics';

// Custom metrics
const successRate = new Rate('success_rate');
const responseTime = new Trend('response_time');
const errors = new Counter('errors');

export const options = {
  scenarios: {
    // Smoke test - ตรวจสอบว่าทำงานได้
    smoke: {
      executor: 'constant-vus',
      vus: 1,
      duration: '1m',
      tags: { test_type: 'smoke' },
    },

    // Load test - normal load
    load: {
      executor: 'ramping-vus',
      stages: [
        { duration: '2m', target: 50 },    // Ramp up
        { duration: '5m', target: 50 },    // Stay
        { duration: '2m', target: 0 },     // Ramp down
      ],
      tags: { test_type: 'load' },
    },

    // Stress test - find breaking point
    stress: {
      executor: 'ramping-vus',
      stages: [
        { duration: '2m', target: 100 },
        { duration: '5m', target: 200 },
        { duration: '2m', target: 300 },
        { duration: '1m', target: 0 },
      ],
      tags: { test_type: 'stress' },
    },
  },

  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    success_rate: ['rate>0.95'],
    errors: ['count<100'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

export function setup() {
  // Login และเก็บ token
  const loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    email: 'test@example.com',
    password: 'password123',
  }), { headers: { 'Content-Type': 'application/json' } });

  return { token: loginRes.json().token };
}

export default function(data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };

  group('API Tests', () => {
    // GET Posts
    const getPostsRes = http.get(`${BASE_URL}/api/posts`, { headers });

    check(getPostsRes, {
      'GET posts: status 200': (r) => r.status === 200,
      'GET posts: response time < 500ms': (r) => r.timings.duration < 500,
    });

    successRate.add(getPostsRes.status === 200);
    responseTime.add(getPostsRes.timings.duration);

    if (getPostsRes.status !== 200) {
      errors.add(1);
    }

    sleep(1);

    // POST Create Post
    const createPostRes = http.post(
      `${BASE_URL}/api/posts`,
      JSON.stringify({
        title: `Load Test Post ${Date.now()}`,
        content: 'Test content',
      }),
      { headers }
    );

    check(createPostRes, {
      'Create post: status 201': (r) => r.status === 201,
    });

    sleep(2);
  });
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Custom Prometheus Dashboard

สร้าง Grafana dashboard ที่แสดง:
- Request rate per endpoint
- Error rate
- Response time percentiles (p50, p95, p99)
- Active connections
- Memory usage over time

### แบบฝึกหัดที่ 2: Memory Leak Simulation

```javascript
// TODO: เขียนโค้ดที่จงใจให้เกิด memory leak
// แล้วใช้ heap snapshots เพื่อหาต้นเหตุ
```

### แบบฝึกหัดที่ 3: Performance Benchmark

```javascript
// TODO: เปรียบเทียบ performance ของ:
// 1. MongoDB with/without indexes
// 2. Redis cache vs direct DB query
// 3. JSON.parse vs JSON5.parse
// ใช้ benchmark.js หรือ jest performance testing
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **PM2** - process management, clustering, graceful reload
2. **Logging** - structured logging, log rotation
3. **Prometheus** - custom metrics, exporters
4. **Grafana** - dashboards, visualization
5. **Performance Profiling** - CPU profiling, heap snapshots
6. **Memory Leaks** - detection, prevention
7. **Database Performance** - indexes, query optimization
8. **APM** - New Relic, Datadog integration
9. **Alerting** - Prometheus alerts, Alertmanager
10. **Load Testing** - k6 scenarios

ในบทถัดไปเราจะเรียนรู้ **Testing Strategies** - Jest, integration tests, E2E, TDD/BDD
