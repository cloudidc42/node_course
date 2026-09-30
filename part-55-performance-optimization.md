# Part 55: Performance Optimization (การเพิ่มประสิทธิภาพ)
## ขั้นตอนที่ 55-55 จาก 1000

---

## บทนำ

Performance optimization เป็นกระบวนการที่ต้อง measure ก่อน optimize เสมอ ห้าม optimize โดยไม่มีข้อมูล

---

## 55.1 Performance Profiling

### Node.js Built-in Profiler

```bash
# รัน app พร้อม profiling
node --prof src/app.js

# ทดสอบด้วย load test
npm install -g artillery
artillery quick --count 100 -n 10 http://localhost:3000/api/users

# แปลง profiling output
node --prof-process isolate-*.log > processed.txt
```

### clinic.js - Advanced Profiling

```bash
npm install -g clinic

# Doctor - วิเคราะห์ bottlenecks
clinic doctor -- node src/app.js

# Flame - Flame graph
clinic flame -- node src/app.js

# Bubbleprof - Async profiling
clinic bubbleprof -- node src/app.js
```

### Custom Performance Monitoring

```javascript
// src/middleware/performanceMonitor.js
const { performance } = require('perf_hooks');

/**
 * Middleware วัด request duration
 */
const performanceMonitor = (req, res, next) => {
  const startTime = performance.now();
  const startMemory = process.memoryUsage();
  
  // Override res.end เพื่อ capture response time
  const originalEnd = res.end.bind(res);
  res.end = (...args) => {
    const duration = performance.now() - startTime;
    const endMemory = process.memoryUsage();
    
    const metrics = {
      method: req.method,
      path: req.route?.path || req.path,
      statusCode: res.statusCode,
      duration: Math.round(duration),
      memoryDelta: {
        heapUsed: endMemory.heapUsed - startMemory.heapUsed
      }
    };
    
    // Log slow requests
    if (duration > 1000) {
      console.warn('🐌 Slow request:', metrics);
    }
    
    // ส่ง metrics ไปยัง monitoring service
    recordMetrics(metrics);
    
    return originalEnd(...args);
  };
  
  next();
};

const recordMetrics = (metrics) => {
  // ส่งไปยัง Prometheus, DataDog, ฯลฯ
};

module.exports = performanceMonitor;
```

---

## 55.2 Clustering

Node.js single-threaded แต่เราสามารถใช้ CPU หลาย core ได้ด้วย Clustering

### Node.js Cluster Module

```javascript
// src/cluster.js
const cluster = require('cluster');
const os = require('os');
const path = require('path');

const NUM_CPUS = os.cpus().length;

if (cluster.isPrimary) {
  console.log(`🚀 Master process ${process.pid} started`);
  console.log(`💻 Creating ${NUM_CPUS} workers...`);
  
  // Fork workers
  for (let i = 0; i < NUM_CPUS; i++) {
    cluster.fork();
  }
  
  // Handle worker events
  cluster.on('exit', (worker, code, signal) => {
    console.log(`⚠️ Worker ${worker.process.pid} died (${signal || code}). Restarting...`);
    cluster.fork();  // Auto-restart dead workers
  });
  
  cluster.on('online', (worker) => {
    console.log(`✅ Worker ${worker.process.pid} online`);
  });
  
  // Graceful shutdown
  process.on('SIGTERM', () => {
    console.log('Graceful shutdown initiated...');
    
    for (const id in cluster.workers) {
      cluster.workers[id].send('shutdown');
    }
    
    setTimeout(() => {
      process.exit(0);
    }, 10000);
  });
  
} else {
  // Worker process
  const app = require('./app');
  
  const PORT = process.env.PORT || 3000;
  const server = app.listen(PORT, () => {
    console.log(`Worker ${process.pid} listening on port ${PORT}`);
  });
  
  // Handle graceful shutdown message from master
  process.on('message', (msg) => {
    if (msg === 'shutdown') {
      server.close(() => {
        console.log(`Worker ${process.pid} shut down`);
        process.exit(0);
      });
    }
  });
}
```

---

## 55.3 PM2

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'myapp',
    script: 'src/app.js',
    
    // Clustering
    instances: 'max',       // ใช้ CPU ทั้งหมด
    exec_mode: 'cluster',
    
    // Environment
    env_production: {
      NODE_ENV: 'production',
      PORT: 3000
    },
    
    // Restart policies
    watch: false,
    max_memory_restart: '500M',
    min_uptime: 5000,
    max_restarts: 10,
    
    // Logs
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    error_file: './logs/pm2-error.log',
    out_file: './logs/pm2-out.log',
    merge_logs: true,
    
    // Graceful shutdown
    kill_timeout: 5000,
    listen_timeout: 10000,
    
    // Auto-scaling (PM2 Plus)
    instance_var: 'INSTANCE_ID'
  }]
};
```

```bash
# PM2 Commands
pm2 start ecosystem.config.js --env production
pm2 reload myapp         # Zero-downtime reload
pm2 restart myapp        # Restart all instances
pm2 scale myapp 4        # Scale to 4 instances
pm2 monit                # Real-time monitoring
pm2 logs myapp           # View logs
pm2 status               # Show status
```

---

## 55.4 Load Testing

### Artillery

```bash
npm install -g artillery
```

```yaml
# load-test.yml
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 120
      arrivalRate: 50
      name: "Ramp up"
    - duration: 300
      arrivalRate: 100
      name: "Sustained load"
  
  variables:
    authToken: "Bearer test-token"

scenarios:
  - name: "Browse products"
    weight: 70
    flow:
      - get:
          url: "/api/products"
          qs:
            page: 1
            limit: 20
          expect:
            - statusCode: 200
  
  - name: "Search products"
    weight: 20
    flow:
      - get:
          url: "/api/products/search"
          qs:
            q: "laptop"
          expect:
            - statusCode: 200
  
  - name: "Create order"
    weight: 10
    flow:
      - post:
          url: "/api/orders"
          headers:
            Authorization: "{{ authToken }}"
          json:
            items: [{ productId: 1, quantity: 2 }]
          expect:
            - statusCode: 201
```

```bash
# รัน load test
artillery run load-test.yml

# รัน และ save report
artillery run load-test.yml --output report.json
artillery report report.json
```

### k6 Load Testing

```javascript
// k6-test.js
import http from 'k6/http';
import { sleep, check } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '1m', target: 50 },   // Ramp up
    { duration: '3m', target: 50 },   // Steady state
    { duration: '1m', target: 0 }     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% ต้องเร็วกว่า 500ms
    errors: ['rate<0.01'],             // Error rate น้อยกว่า 1%
    http_req_failed: ['rate<0.05']
  }
};

const BASE_URL = 'http://localhost:3000/api';

export default function() {
  const params = {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer test-token'
    }
  };
  
  // GET /products
  const productsRes = http.get(`${BASE_URL}/products?page=1&limit=20`, params);
  
  check(productsRes, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'has data': (r) => {
      const body = JSON.parse(r.body);
      return body.data && body.data.length > 0;
    }
  }) || errorRate.add(1);
  
  sleep(1);
}
```

```bash
# ติดตั้ง k6
# macOS: brew install k6
# Ubuntu: apt-get install k6

# รัน test
k6 run k6-test.js

# รัน ด้วย output ไป Grafana
k6 run --out influxdb=http://localhost:8086/k6 k6-test.js
```

---

## 55.5 Memory Optimization

```javascript
// src/utils/memoryUtils.js

/**
 * Monitor memory usage
 */
const monitorMemory = (intervalMs = 30000) => {
  const interval = setInterval(() => {
    const usage = process.memoryUsage();
    const formatted = {
      rss: formatBytes(usage.rss),
      heapTotal: formatBytes(usage.heapTotal),
      heapUsed: formatBytes(usage.heapUsed),
      external: formatBytes(usage.external)
    };
    
    console.log('📊 Memory:', formatted);
    
    // Alert ถ้า heap usage สูงเกิน 80%
    const heapUsagePercent = (usage.heapUsed / usage.heapTotal) * 100;
    if (heapUsagePercent > 80) {
      console.warn(`⚠️ High memory usage: ${heapUsagePercent.toFixed(1)}%`);
    }
  }, intervalMs);
  
  return interval;
};

const formatBytes = (bytes) => {
  return `${(bytes / 1024 / 1024).toFixed(2)} MB`;
};

/**
 * Memory leak detection (simple)
 */
class MemoryTracker {
  constructor() {
    this.snapshots = [];
  }
  
  takeSnapshot(label) {
    const usage = process.memoryUsage();
    this.snapshots.push({
      label,
      time: Date.now(),
      heapUsed: usage.heapUsed
    });
    
    if (this.snapshots.length > 10) {
      this.snapshots.shift();
    }
  }
  
  detectLeak() {
    if (this.snapshots.length < 5) return false;
    
    // ตรวจสอบว่า memory เพิ่มขึ้นเรื่อย ๆ ไหม
    const recentSnapshots = this.snapshots.slice(-5);
    let increasingCount = 0;
    
    for (let i = 1; i < recentSnapshots.length; i++) {
      if (recentSnapshots[i].heapUsed > recentSnapshots[i-1].heapUsed) {
        increasingCount++;
      }
    }
    
    return increasingCount >= 4;  // เพิ่มขึ้น 4 จาก 4 ครั้งล่าสุด
  }
}

module.exports = { monitorMemory, MemoryTracker };
```

### Caching Strategies

```javascript
// src/utils/cache.js
const redis = require('ioredis');
const client = new redis(process.env.REDIS_URL);

/**
 * Cache decorator
 */
const withCache = (fn, keyFn, ttl = 3600) => {
  return async (...args) => {
    const key = keyFn(...args);
    
    // ลองหาจาก cache
    const cached = await client.get(key);
    if (cached) {
      return JSON.parse(cached);
    }
    
    // ถ้าไม่มี execute function
    const result = await fn(...args);
    
    // Cache ผลลัพธ์
    await client.setex(key, ttl, JSON.stringify(result));
    
    return result;
  };
};

// ตัวอย่างใช้งาน
const getProductById = withCache(
  async (id) => Product.findByPk(id),
  (id) => `product:${id}`,
  3600  // cache 1 ชั่วโมง
);
```

---

## แบบฝึกหัดที่ 55

### แบบฝึกหัดพื้นฐาน

**1. Performance Profiling**

Profile Node.js application:
- ใช้ `clinic doctor` หา bottlenecks
- ดู flame graph
- แก้ไข hot spots

**2. Load Testing**

ทดสอบ performance ด้วย Artillery:
- สร้าง load test script
- Test scenarios ต่าง ๆ
- วิเคราะห์ results

### แบบฝึกหัดขั้นสูง

**3. Clustering + PM2**

Setup production-ready Node.js:
- Cluster mode ด้วย PM2
- Zero-downtime deployment
- Memory monitoring และ auto-restart

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Profiling** - clinic.js, flame graphs
2. **Clustering** - ใช้ CPU หลาย cores
3. **PM2** - process manager, cluster mode
4. **Load Testing** - Artillery, k6
5. **Memory** - monitoring, caching

**ถัดไป:** Part 56 - Database Optimization
