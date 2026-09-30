# Part 87 | ขั้นตอนที่ 1541-1560 จาก 1000+

## Performance Engineering สำหรับ Node.js

ในส่วนนี้เราจะเรียนรู้การ profiling, flame graphs, memory optimization, และการ optimize performance อย่างเป็นระบบ

---

## ขั้นตอนที่ 1541: Performance Engineering Methodology

```javascript
// performance-baseline.js
// สร้าง performance baseline ก่อน optimize

const { performance, PerformanceObserver } = require('perf_hooks');

class PerformanceBaseline {
  constructor() {
    this.measurements = new Map();
    this.setupObserver();
  }

  setupObserver() {
    const obs = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        this.record(entry.name, entry.duration);
      }
    });
    obs.observe({ entryTypes: ['measure', 'function'] });
  }

  record(name, value) {
    if (!this.measurements.has(name)) {
      this.measurements.set(name, []);
    }
    this.measurements.get(name).push(value);
  }

  async measure(name, fn) {
    const start = performance.now();
    try {
      const result = await fn();
      const duration = performance.now() - start;
      this.record(name, duration);
      return result;
    } catch (err) {
      const duration = performance.now() - start;
      this.record(`${name}_failed`, duration);
      throw err;
    }
  }

  getStats(name) {
    const values = this.measurements.get(name) || [];
    if (values.length === 0) return null;

    const sorted = [...values].sort((a, b) => a - b);
    const sum = values.reduce((a, b) => a + b, 0);

    return {
      count: values.length,
      min: sorted[0].toFixed(2) + 'ms',
      max: sorted[sorted.length - 1].toFixed(2) + 'ms',
      mean: (sum / values.length).toFixed(2) + 'ms',
      p50: sorted[Math.floor(sorted.length * 0.5)].toFixed(2) + 'ms',
      p95: sorted[Math.floor(sorted.length * 0.95)].toFixed(2) + 'ms',
      p99: sorted[Math.floor(sorted.length * 0.99)].toFixed(2) + 'ms'
    };
  }

  generateReport() {
    const report = {};
    for (const [name] of this.measurements) {
      report[name] = this.getStats(name);
    }
    return report;
  }
}

// ใช้งาน
const baseline = new PerformanceBaseline();

async function exampleWithMeasurement() {
  // Measure database query
  const users = await baseline.measure('db.users.findAll', () => 
    db.query('SELECT * FROM users LIMIT 100')
  );
  
  // Measure cache operations
  const cached = await baseline.measure('cache.get', () =>
    redis.get('key:1')
  );
  
  // Generate report
  const report = baseline.generateReport();
  console.table(report);
}

module.exports = { PerformanceBaseline };
```

---

## ขั้นตอนที่ 1542: Node.js Profiling

```javascript
// profiling.js
// CPU และ Memory profiling

const v8Profiler = require('v8-profiler-next');
const fs = require('fs');
const path = require('path');

class Profiler {
  constructor(outputDir = './profiles') {
    this.outputDir = outputDir;
    fs.mkdirSync(outputDir, { recursive: true });
  }

  // CPU Profiling
  async cpuProfile(name, durationMs = 10000) {
    const profileName = `${name}_${Date.now()}`;
    
    console.log(`Starting CPU profile: ${profileName} (${durationMs}ms)`);
    v8Profiler.startProfiling(profileName, true);
    
    await new Promise(resolve => setTimeout(resolve, durationMs));
    
    const profile = v8Profiler.stopProfiling(profileName);
    
    return new Promise((resolve, reject) => {
      profile.export((error, result) => {
        if (error) {
          reject(error);
          return;
        }
        
        const outputPath = path.join(this.outputDir, `${profileName}.cpuprofile`);
        fs.writeFileSync(outputPath, result);
        
        profile.delete();
        
        console.log(`CPU profile saved: ${outputPath}`);
        resolve({ profileName, outputPath });
      });
    });
  }

  // Heap Snapshot
  takeHeapSnapshot(name = 'heap') {
    const snapshotName = `${name}_${Date.now()}`;
    const outputPath = path.join(this.outputDir, `${snapshotName}.heapsnapshot`);
    
    console.log('Taking heap snapshot...');
    const snapshot = v8Profiler.takeSnapshot(snapshotName);
    
    return new Promise((resolve, reject) => {
      snapshot.export((error, result) => {
        if (error) {
          reject(error);
          return;
        }
        
        fs.writeFileSync(outputPath, result);
        snapshot.delete();
        
        console.log(`Heap snapshot saved: ${outputPath}`);
        resolve({ snapshotName, outputPath });
      });
    });
  }

  // Memory Leak Detection
  async detectMemoryLeak(fn, iterations = 100) {
    const memUsageBefore = process.memoryUsage();
    
    // Run function multiple times
    for (let i = 0; i < iterations; i++) {
      await fn();
      
      // Force GC every 10 iterations
      if (i % 10 === 0 && global.gc) {
        global.gc();
      }
    }
    
    if (global.gc) global.gc();
    
    const memUsageAfter = process.memoryUsage();
    
    const heapGrowth = memUsageAfter.heapUsed - memUsageBefore.heapUsed;
    const heapGrowthMB = heapGrowth / 1024 / 1024;
    
    const analysis = {
      heapUsedBefore: `${Math.round(memUsageBefore.heapUsed / 1024 / 1024)}MB`,
      heapUsedAfter: `${Math.round(memUsageAfter.heapUsed / 1024 / 1024)}MB`,
      heapGrowth: `${heapGrowthMB.toFixed(2)}MB`,
      possibleLeak: heapGrowthMB > 10, // Threshold: 10MB
      perIterationGrowth: `${(heapGrowth / iterations / 1024).toFixed(2)}KB`
    };
    
    if (analysis.possibleLeak) {
      console.warn('⚠️ Possible memory leak detected!', analysis);
    }
    
    return analysis;
  }
}

// Auto-profiling สำหรับ production debugging
class AutoProfiler {
  constructor(options = {}) {
    this.options = {
      cpuThreshold: options.cpuThreshold || 80,     // Alert if CPU > 80%
      memThreshold: options.memThreshold || 85,      // Alert if memory > 85%
      profileDuration: options.profileDuration || 30000,
      ...options
    };
    
    this.profiler = new Profiler(options.outputDir);
    this.profiling = false;
  }

  start() {
    setInterval(async () => {
      await this.checkAndProfile();
    }, 60000); // Check every minute
  }

  async checkAndProfile() {
    const memUsage = process.memoryUsage();
    const heapPercent = (memUsage.heapUsed / memUsage.heapTotal) * 100;
    
    if (heapPercent > this.options.memThreshold && !this.profiling) {
      console.warn(`Memory threshold exceeded: ${heapPercent.toFixed(1)}%`);
      
      this.profiling = true;
      try {
        await this.profiler.takeHeapSnapshot('auto-memory-alert');
      } finally {
        this.profiling = false;
      }
    }
  }
}

module.exports = { Profiler, AutoProfiler };
```

---

## ขั้นตอนที่ 1543: Flame Graphs

```bash
# สร้าง Flame Graph สำหรับ Node.js

# 1. ติดตั้ง tools
npm install -g clinic
npm install -g 0x

# 2. Flame Graph ด้วย 0x
0x -- node server.js

# 3. Clinic.js Suite
clinic doctor -- node server.js
clinic flame -- node server.js
clinic bubbleprof -- node server.js

# 4. รัน load test ระหว่าง profiling
# Terminal 1: Start with profiler
0x --output-dir ./profiles -- node server.js

# Terminal 2: Generate load
npx autocannon -c 100 -d 30 http://localhost:3000/api/users
```

```javascript
// flame-graph-readable.js
// Code ที่ profiler-friendly

// ❌ Bad: Anonymous functions ทำให้ flame graph อ่านยาก
app.get('/api/users', async (req, res) => {
  const data = await db.query('SELECT * FROM users');
  const result = data.rows.map(u => ({...u, fullName: `${u.first_name} ${u.last_name}`}));
  res.json(result);
});

// ✅ Good: Named functions ทำให้ flame graph อ่านง่าย
async function fetchAllUsers(db) {
  return db.query('SELECT * FROM users');
}

function formatUser(user) {
  return {
    ...user,
    fullName: `${user.first_name} ${user.last_name}`
  };
}

async function handleGetUsers(req, res) {
  const data = await fetchAllUsers(req.db);
  const formattedUsers = data.rows.map(formatUser);
  res.json(formattedUsers);
}

app.get('/api/users', handleGetUsers);

// ใช้ --perf_basic_prof สำหรับ Linux perf
// node --perf_basic_prof server.js &
// sleep 30
// perf record -F 99 -p $(cat /tmp/node_perf_pid) -g -- sleep 30
// perf script > /tmp/out.perf
// stackvis < /tmp/out.perf > flame.svg
```

---

## ขั้นตอนที่ 1544: Memory Optimization

```javascript
// memory-optimization.js
// เทคนิคการลด memory usage

// 1. Object Pool Pattern - reuse objects แทน create/garbage collect
class ObjectPool {
  constructor(factory, options = {}) {
    this.factory = factory;
    this.maxSize = options.maxSize || 100;
    this.pool = [];
    this.inUse = new Set();
    
    // Pre-allocate some objects
    const preAllocate = options.preAllocate || Math.floor(this.maxSize * 0.2);
    for (let i = 0; i < preAllocate; i++) {
      this.pool.push(this.factory());
    }
  }

  acquire() {
    let obj;
    
    if (this.pool.length > 0) {
      obj = this.pool.pop();
    } else if (this.inUse.size < this.maxSize) {
      obj = this.factory();
    } else {
      throw new Error('Object pool exhausted');
    }
    
    this.inUse.add(obj);
    return obj;
  }

  release(obj) {
    if (!this.inUse.has(obj)) {
      throw new Error('Object not from this pool');
    }
    
    this.inUse.delete(obj);
    
    // Reset object state before returning to pool
    if (obj.reset) obj.reset();
    
    this.pool.push(obj);
  }

  getStats() {
    return {
      poolSize: this.pool.length,
      inUse: this.inUse.size,
      total: this.pool.length + this.inUse.size,
      maxSize: this.maxSize
    };
  }
}

// Example: Buffer Pool
class BufferPool {
  constructor(bufferSize = 65536, poolSize = 50) {
    this.pool = new ObjectPool(
      () => Buffer.allocUnsafe(bufferSize),
      { maxSize: poolSize, preAllocate: 10 }
    );
  }

  acquire() {
    const buf = this.pool.acquire();
    buf.fill(0);  // Clear before use
    return buf;
  }

  release(buf) {
    this.pool.release(buf);
  }
}

// 2. WeakRef สำหรับ cache ที่ GC-friendly
class WeakCache {
  constructor() {
    this.cache = new Map();
    this.registry = new FinalizationRegistry((key) => {
      // Cleanup when object is garbage collected
      this.cache.delete(key);
      console.log(`Cache entry ${key} was garbage collected`);
    });
  }

  set(key, value) {
    const ref = new WeakRef(value);
    this.cache.set(key, ref);
    this.registry.register(value, key);
  }

  get(key) {
    const ref = this.cache.get(key);
    if (!ref) return undefined;
    
    const value = ref.deref();
    if (value === undefined) {
      this.cache.delete(key);
      return undefined;
    }
    
    return value;
  }

  size() {
    // Actual size may be less due to GC
    return this.cache.size;
  }
}

// 3. Streaming ใหญ่ๆ แทน loading ทั้งหมด
const { Transform, pipeline } = require('stream');
const { promisify } = require('util');
const pipelineAsync = promisify(pipeline);

async function processLargeFile(inputPath, outputPath) {
  const fs = require('fs');
  
  const readStream = fs.createReadStream(inputPath);
  const writeStream = fs.createWriteStream(outputPath);
  
  const transform = new Transform({
    objectMode: false,
    transform(chunk, encoding, callback) {
      // Process chunk by chunk - memory efficient!
      const processed = processChunk(chunk);
      callback(null, processed);
    }
  });
  
  // Pipeline handles backpressure automatically
  await pipelineAsync(readStream, transform, writeStream);
  
  console.log('File processed successfully');
}

function processChunk(chunk) {
  return chunk.toString().toUpperCase();
}

// 4. Avoid Memory Leaks

// ❌ Bad: Event listener leaks
function badPattern(emitter) {
  emitter.on('data', (data) => {
    processData(data);
  });
  // Listener never removed! Memory leak if emitter lives long
}

// ✅ Good: Remove listeners when done
function goodPattern(emitter) {
  const handler = (data) => {
    processData(data);
  };
  
  emitter.on('data', handler);
  
  // Return cleanup function
  return () => emitter.off('data', handler);
}

// ❌ Bad: Storing too many references in closures
function badClosure() {
  const HUGE_ARRAY = new Array(1000000).fill('data'); // 1M items
  
  return function smallFunction() {
    return HUGE_ARRAY[0]; // Closure keeps entire array alive!
  };
}

// ✅ Good: Extract only what you need
function goodClosure() {
  const HUGE_ARRAY = new Array(1000000).fill('data');
  const firstItem = HUGE_ARRAY[0]; // Extract only what's needed
  // HUGE_ARRAY can now be GC'd
  
  return function smallFunction() {
    return firstItem; // Only reference to firstItem, not entire array
  };
}

// 5. Monitor memory in production
function setupMemoryMonitoring() {
  const used = [];
  
  setInterval(() => {
    const memUsage = process.memoryUsage();
    used.push(memUsage.heapUsed);
    
    // Keep only last 60 readings
    if (used.length > 60) used.shift();
    
    // Detect trend
    if (used.length === 60) {
      const firstHalf = used.slice(0, 30).reduce((a, b) => a + b, 0) / 30;
      const secondHalf = used.slice(30).reduce((a, b) => a + b, 0) / 30;
      
      const growthRate = (secondHalf - firstHalf) / firstHalf * 100;
      
      if (growthRate > 20) {
        console.warn(`Memory growth trend detected: ${growthRate.toFixed(1)}% increase over last 60 samples`);
        
        // Alert operations team
        alertOps({
          type: 'memory_growth',
          growthRate: growthRate.toFixed(1),
          currentUsage: `${Math.round(memUsage.heapUsed / 1024 / 1024)}MB`
        });
      }
    }
  }, 60000); // Every minute
}

function alertOps(data) {
  console.error('ALERT:', data);
  // Send to monitoring system
}

function processData(data) {}

module.exports = { ObjectPool, BufferPool, WeakCache };
```

---

## ขั้นตอนที่ 1545: JavaScript V8 Optimizations

```javascript
// v8-optimizations.js
// เขียน code ที่ V8 optimize ได้ดี

// 1. Monomorphic functions (same types)
// ❌ Bad: Polymorphic - V8 ต้อง deoptimize
function addBad(a, b) {
  return a + b;
}
addBad(1, 2);          // int + int
addBad('a', 'b');      // string + string
addBad(1.5, 2.5);      // float + float

// ✅ Good: Monomorphic - V8 can optimize
function addNumbers(a, b) {
  return a + b;  // Always numbers
}
function addStrings(a, b) {
  return a + b;  // Always strings
}

// 2. Object Shape (Hidden Classes)
// ❌ Bad: Different property orders = different shapes
function createUserBad(name, age) {
  const user = {};
  if (age) user.age = age;  // Conditional property
  user.name = name;
  return user;
}

// ✅ Good: Consistent object shape
function createUserGood(name, age) {
  return { name, age: age || null };  // Always same shape
}

// 3. Array optimization
// ❌ Bad: Mixed types prevent Typed Arrays optimization
const badArray = [1, 'two', 3, true]; // SMI_ELEMENTS -> ELEMENTS

// ✅ Good: Typed arrays for numeric data
const intBuffer = new Int32Array(1000);  // Compact, fast
const floatBuffer = new Float64Array(1000);

// 4. Avoid Hidden Class transitions
// ❌ Bad: Adding properties after creation
class BadClass {
  constructor(name) {
    this.name = name;
    // Properties added later
  }
  
  setAge(age) {
    this.age = age;  // Creates new hidden class!
  }
}

// ✅ Good: Initialize all properties in constructor
class GoodClass {
  constructor(name, age = null) {
    this.name = name;
    this.age = age;  // Always initialized
  }
}

// 5. String optimization
// ❌ Bad: String concatenation in loops
function buildStringBad(items) {
  let result = '';
  for (const item of items) {
    result += item + ', ';  // Creates new string each iteration
  }
  return result;
}

// ✅ Good: Array.join
function buildStringGood(items) {
  return items.join(', ');  // Single allocation
}

// ✅ Also good: Template literals for small counts
function formatUser(user) {
  return `${user.name} (${user.age})`;
}

// 6. Async optimizations
// ❌ Bad: Sequential async calls
async function fetchDataBad(ids) {
  const results = [];
  for (const id of ids) {
    const data = await fetchItem(id);  // Waits for each one
    results.push(data);
  }
  return results;
}

// ✅ Good: Parallel with concurrency limit
async function fetchDataGood(ids, concurrency = 10) {
  const results = new Array(ids.length);
  
  for (let i = 0; i < ids.length; i += concurrency) {
    const batch = ids.slice(i, i + concurrency);
    const batchResults = await Promise.all(batch.map(fetchItem));
    batchResults.forEach((result, j) => {
      results[i + j] = result;
    });
  }
  
  return results;
}

async function fetchItem(id) {
  return { id };
}

// 7. Regex optimization
// ❌ Bad: Regex compiled in function
function validateEmailBad(email) {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);  // Compiled each call!
}

// ✅ Good: Pre-compiled regex
const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
function validateEmailGood(email) {
  return EMAIL_REGEX.test(email);
}

module.exports = { buildStringGood, fetchDataGood };
```

---

## ขั้นตอนที่ 1546: Database Query Optimization

```javascript
// query-optimization.js
// Optimize database queries สำหรับ performance

const { Pool } = require('pg');

class QueryOptimizer {
  constructor(pool) {
    this.pool = pool;
    this.queryLog = [];
  }

  // EXPLAIN ANALYZE wrapper
  async analyzeQuery(sql, params = []) {
    const result = await this.pool.query(
      `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`,
      params
    );
    
    const plan = result.rows[0]['QUERY PLAN'][0];
    
    return {
      planningTime: plan['Planning Time'],
      executionTime: plan['Execution Time'],
      plan: plan['Plan'],
      analysis: this.analyzePlan(plan['Plan'])
    };
  }

  analyzePlan(node, depth = 0) {
    const issues = [];
    
    // Sequential scans on large tables
    if (node['Node Type'] === 'Seq Scan' && node['Actual Rows'] > 10000) {
      issues.push({
        type: 'SEQUENTIAL_SCAN',
        table: node['Relation Name'],
        rows: node['Actual Rows'],
        recommendation: `Add index on ${node['Relation Name']}`
      });
    }
    
    // Nested loop with many iterations
    if (node['Node Type'] === 'Nested Loop' && node['Actual Rows'] > 1000) {
      issues.push({
        type: 'NESTED_LOOP',
        rows: node['Actual Rows'],
        recommendation: 'Consider using Hash Join instead'
      });
    }
    
    // Recurse into child plans
    if (node['Plans']) {
      for (const child of node['Plans']) {
        issues.push(...this.analyzePlan(child, depth + 1));
      }
    }
    
    return issues;
  }

  // N+1 Query Detection
  detectN1(queryLog) {
    const grouped = {};
    
    for (const { sql, duration } of queryLog) {
      const normalized = sql.replace(/\$\d+/g, '?').trim();
      if (!grouped[normalized]) grouped[normalized] = { count: 0, totalDuration: 0 };
      grouped[normalized].count++;
      grouped[normalized].totalDuration += duration;
    }
    
    return Object.entries(grouped)
      .filter(([, stats]) => stats.count > 10)
      .map(([sql, stats]) => ({
        query: sql,
        count: stats.count,
        totalDuration: `${stats.totalDuration.toFixed(0)}ms`,
        avgDuration: `${(stats.totalDuration / stats.count).toFixed(2)}ms`,
        recommendation: 'N+1 query detected - consider using JOIN or eager loading'
      }));
  }
}

// Common optimization patterns
const QueryPatterns = {
  // ❌ N+1 Problem
  async getUsersWithOrdersBad(db) {
    const users = await db.query('SELECT * FROM users');
    
    for (const user of users.rows) {
      const orders = await db.query(
        'SELECT * FROM orders WHERE user_id = $1',
        [user.id]
      );
      user.orders = orders.rows;
    }
    
    return users.rows;
  },

  // ✅ Eager Loading with JOIN
  async getUsersWithOrdersGood(db) {
    const result = await db.query(`
      SELECT 
        u.id,
        u.name,
        u.email,
        json_agg(
          CASE WHEN o.id IS NOT NULL 
          THEN json_build_object('id', o.id, 'total', o.total, 'status', o.status)
          END
        ) FILTER (WHERE o.id IS NOT NULL) as orders
      FROM users u
      LEFT JOIN orders o ON u.id = o.user_id
      GROUP BY u.id, u.name, u.email
    `);
    
    return result.rows;
  },

  // ❌ Fetching all columns
  async getUsersBad(db, userId) {
    return db.query('SELECT * FROM users WHERE id = $1', [userId]);
  },

  // ✅ Select only needed columns
  async getUsersGood(db, userId) {
    return db.query('SELECT id, name, email, created_at FROM users WHERE id = $1', [userId]);
  },

  // Pagination with cursor (efficient for large datasets)
  async getUsersWithCursor(db, cursor = null, limit = 20) {
    if (cursor) {
      return db.query(
        'SELECT id, name, email FROM users WHERE id > $1 ORDER BY id ASC LIMIT $2',
        [cursor, limit]
      );
    }
    
    return db.query(
      'SELECT id, name, email FROM users ORDER BY id ASC LIMIT $1',
      [limit]
    );
  },

  // Batch operations
  async insertUsersBatch(db, users) {
    if (users.length === 0) return;
    
    const values = users.flatMap((user, i) => [
      user.name,
      user.email,
      user.createdAt || new Date()
    ]);
    
    const placeholders = users.map((_, i) => 
      `($${i*3+1}, $${i*3+2}, $${i*3+3})`
    ).join(', ');
    
    return db.query(
      `INSERT INTO users (name, email, created_at) VALUES ${placeholders}`,
      values
    );
  }
};

// Index recommendations
const IndexRecommendations = {
  async analyze(db, tableName) {
    // Find columns used in WHERE clauses without indexes
    const result = await db.query(`
      SELECT 
        schemaname,
        tablename,
        attname as column_name,
        n_distinct,
        correlation
      FROM pg_stats
      WHERE tablename = $1
      ORDER BY n_distinct DESC
    `, [tableName]);
    
    return result.rows;
  },

  async findMissingIndexes(db) {
    const result = await db.query(`
      SELECT 
        schemaname || '.' || tablename AS table,
        indexname,
        idx_scan as index_scans,
        idx_tup_read as tuples_read,
        idx_tup_fetch as tuples_fetched
      FROM pg_stat_user_indexes
      WHERE idx_scan = 0
      ORDER BY pg_relation_size(indexrelid) DESC
      LIMIT 20
    `);
    
    return result.rows;
  },

  async findSlowQueries(db, minDurationMs = 1000) {
    const result = await db.query(`
      SELECT 
        query,
        calls,
        total_exec_time,
        mean_exec_time,
        stddev_exec_time,
        rows
      FROM pg_stat_statements
      WHERE mean_exec_time > $1
      ORDER BY mean_exec_time DESC
      LIMIT 20
    `, [minDurationMs]);
    
    return result.rows;
  }
};

module.exports = { QueryOptimizer, QueryPatterns, IndexRecommendations };
```

---

## ขั้นตอนที่ 1547-1560: Advanced Performance Topics

### HTTP Response Optimization

```javascript
// response-optimization.js
// Optimize HTTP responses

const express = require('express');
const compression = require('compression');
const { createBrotliCompress, createGzip } = require('zlib');

const app = express();

// 1. Compression
app.use(compression({
  level: 6,           // Balanced speed vs compression
  threshold: 1024,    // Only compress > 1KB
  filter: (req, res) => {
    if (req.headers['x-no-compression']) return false;
    return compression.filter(req, res);
  }
}));

// 2. HTTP/2 Push (เมื่อใช้ HTTP/2)
const http2 = require('http2');
const fs = require('fs');

const server = http2.createSecureServer({
  key: fs.readFileSync('./server.key'),
  cert: fs.readFileSync('./server.crt')
});

server.on('stream', (stream, headers) => {
  const path = headers[':path'];
  
  if (path === '/') {
    // Push critical resources
    stream.pushStream({ ':path': '/static/main.css' }, (err, pushStream) => {
      if (!err) {
        pushStream.respondWithFile('./public/static/main.css');
      }
    });
    
    stream.respondWithFile('./public/index.html');
  }
});

// 3. Response Caching Middleware
function cacheResponse(ttl = 300, options = {}) {
  const Redis = require('ioredis');
  const redis = new Redis();
  
  return async (req, res, next) => {
    if (req.method !== 'GET') return next();
    
    const cacheKey = `response:${req.originalUrl}`;
    
    try {
      const cached = await redis.get(cacheKey);
      
      if (cached) {
        const { body, headers, statusCode } = JSON.parse(cached);
        
        res.set(headers);
        res.set('X-Cache', 'HIT');
        res.status(statusCode).send(body);
        return;
      }
    } catch (err) {
      // Cache miss or error - continue
    }
    
    // Capture response
    const originalSend = res.send.bind(res);
    
    res.send = async function(body) {
      if (res.statusCode === 200 && !options.noStore) {
        const cacheData = {
          body,
          headers: res.getHeaders(),
          statusCode: res.statusCode
        };
        
        await redis.setex(cacheKey, ttl, JSON.stringify(cacheData)).catch(() => {});
      }
      
      res.set('X-Cache', 'MISS');
      originalSend(body);
    };
    
    next();
  };
}

// 4. API Response Shaping
class ResponseShaper {
  // Sparse fieldsets - return only requested fields
  static shapeResponse(data, fields) {
    if (!fields || fields.length === 0) return data;
    
    if (Array.isArray(data)) {
      return data.map(item => this.pickFields(item, fields));
    }
    
    return this.pickFields(data, fields);
  }

  static pickFields(obj, fields) {
    return fields.reduce((result, field) => {
      if (field.includes('.')) {
        // Nested field support: "user.name"
        const [parent, ...rest] = field.split('.');
        if (obj[parent]) {
          result[parent] = result[parent] || {};
          result[parent][rest.join('.')] = obj[parent][rest.join('.')];
        }
      } else if (obj[field] !== undefined) {
        result[field] = obj[field];
      }
      return result;
    }, {});
  }
}

// Middleware สำหรับ field selection
app.use((req, res, next) => {
  if (req.query.fields) {
    const fields = req.query.fields.split(',').map(f => f.trim());
    
    const originalJson = res.json.bind(res);
    res.json = (data) => {
      const shaped = ResponseShaper.shapeResponse(data, fields);
      originalJson(shaped);
    };
  }
  next();
});

// 5. Concurrent Request Optimization
async function optimizeParallelRequests(req, res) {
  const userId = req.params.userId;
  
  // ❌ Sequential
  // const user = await getUser(userId);
  // const orders = await getOrders(userId);
  // const preferences = await getPreferences(userId);
  
  // ✅ Parallel
  const [user, orders, preferences] = await Promise.all([
    getUser(userId),
    getOrders(userId),
    getPreferences(userId)
  ]);
  
  res.json({ user, orders, preferences });
}

async function getUser(id) { return { id, name: 'User' }; }
async function getOrders(userId) { return []; }
async function getPreferences(userId) { return {}; }

module.exports = { cacheResponse, ResponseShaper };
```

---

## แบบฝึกหัด

### Exercise 1: Performance Profiling
Profile Node.js application ด้วย clinic.js และสร้าง flame graph หา bottleneck

### Exercise 2: Memory Leak Detection
เขียน script ที่ detect memory leaks โดยการ monitor heap growth trend

### Exercise 3: Query Optimization
ใช้ EXPLAIN ANALYZE เพื่อ optimize slow queries ใน PostgreSQL

### คำถามทบทวน
1. Flame graph อ่านอย่างไร และบอกอะไรเกี่ยวกับ performance?
2. V8 hidden classes คืออะไร และส่งผลต่อ performance อย่างไร?
3. Object Pool pattern ช่วย performance ได้อย่างไร?
4. N+1 query problem คืออะไร และแก้ได้อย่างไร?

---

*ต่อไป: Part 88 - Load Testing*
