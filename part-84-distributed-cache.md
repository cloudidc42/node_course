# Part 84 | ขั้นตอนที่ 1481-1500 จาก 1000+

## Distributed Caching ด้วย Redis Cluster

ในส่วนนี้เราจะเรียนรู้การสร้าง distributed caching system ด้วย Redis Cluster รวมถึง cache coherence และ eviction policies

---

## ขั้นตอนที่ 1481: Redis Cluster Setup

```javascript
// redis-cluster.js
// Redis Cluster configuration และการใช้งาน

const Redis = require('ioredis');

// Redis Cluster connection
const cluster = new Redis.Cluster([
  { host: 'redis-node-1', port: 7000 },
  { host: 'redis-node-2', port: 7001 },
  { host: 'redis-node-3', port: 7002 },
  // Replicas
  { host: 'redis-node-4', port: 7003 },
  { host: 'redis-node-5', port: 7004 },
  { host: 'redis-node-6', port: 7005 }
], {
  // Cluster options
  clusterRetryStrategy: (times) => {
    const delay = Math.min(times * 100, 3000);
    return delay;
  },
  
  // Redis options
  redisOptions: {
    password: process.env.REDIS_PASSWORD,
    tls: process.env.NODE_ENV === 'production' ? {} : null,
    connectTimeout: 10000,
    lazyConnect: false
  },
  
  // Cluster-specific options
  enableOfflineQueue: true,
  maxRedirections: 16,
  retryDelayOnFailover: 100,
  retryDelayOnClusterDown: 300,
  
  // Enable read from replicas
  scaleReads: 'slave'
});

// Event listeners
cluster.on('connect', () => console.log('Redis Cluster connected'));
cluster.on('error', (err) => console.error('Redis Cluster error:', err));
cluster.on('node error', (err, node) => {
  console.error(`Redis node error: ${node.options.host}:${node.options.port}`, err);
});

// Cluster status check
async function getClusterInfo() {
  const info = await cluster.cluster('INFO');
  return info;
}

async function getClusterNodes() {
  const nodes = await cluster.cluster('NODES');
  return nodes.split('\n').filter(Boolean).map(node => {
    const parts = node.split(' ');
    return {
      id: parts[0],
      address: parts[1],
      flags: parts[2],
      role: parts[2].includes('master') ? 'master' : 'slave',
      slots: parts[8]
    };
  });
}

module.exports = { cluster, getClusterInfo, getClusterNodes };
```

---

## ขั้นตอนที่ 1482: Cache Eviction Policies

```javascript
// eviction-policies.js
// Understanding and implementing cache eviction

/*
Redis Eviction Policies:
- noeviction: Return error when memory limit reached
- allkeys-lru: Remove least recently used keys
- allkeys-lfu: Remove least frequently used keys
- allkeys-random: Remove random keys
- volatile-lru: Remove LRU keys with TTL set
- volatile-lfu: Remove LFU keys with TTL set
- volatile-random: Remove random keys with TTL set
- volatile-ttl: Remove keys with closest expiration
*/

// Choosing the right eviction policy
const evictionGuide = {
  userSessions: 'volatile-lru',      // Sessions มี TTL อยู่แล้ว
  apiCache: 'allkeys-lru',           // Cache ทั้งหมด LRU
  frequentData: 'allkeys-lfu',       // Hot data ต้อง LFU
  temporaryData: 'volatile-ttl',     // ข้อมูลชั่วคราว
};

// Custom LFU (Least Frequently Used) Cache
class LFUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.keyMap = new Map();      // key -> { value, freq }
    this.freqMap = new Map();     // freq -> Set of keys
    this.minFreq = 0;
    this.size = 0;
  }

  get(key) {
    if (!this.keyMap.has(key)) return -1;
    
    this.updateFreq(key);
    return this.keyMap.get(key).value;
  }

  put(key, value) {
    if (this.capacity <= 0) return;
    
    if (this.keyMap.has(key)) {
      this.keyMap.get(key).value = value;
      this.updateFreq(key);
      return;
    }
    
    if (this.size >= this.capacity) {
      this.evict();
    }
    
    this.keyMap.set(key, { value, freq: 1 });
    
    if (!this.freqMap.has(1)) {
      this.freqMap.set(1, new Set());
    }
    this.freqMap.get(1).add(key);
    
    this.minFreq = 1;
    this.size++;
  }

  updateFreq(key) {
    const { freq } = this.keyMap.get(key);
    this.keyMap.get(key).freq = freq + 1;
    
    this.freqMap.get(freq).delete(key);
    if (this.freqMap.get(freq).size === 0) {
      this.freqMap.delete(freq);
      if (this.minFreq === freq) this.minFreq++;
    }
    
    if (!this.freqMap.has(freq + 1)) {
      this.freqMap.set(freq + 1, new Set());
    }
    this.freqMap.get(freq + 1).add(key);
  }

  evict() {
    const keys = this.freqMap.get(this.minFreq);
    const keyToEvict = keys.values().next().value;
    
    keys.delete(keyToEvict);
    if (keys.size === 0) this.freqMap.delete(this.minFreq);
    
    this.keyMap.delete(keyToEvict);
    this.size--;
  }

  getStats() {
    return {
      size: this.size,
      capacity: this.capacity,
      utilization: `${Math.round(this.size / this.capacity * 100)}%`,
      minFreq: this.minFreq
    };
  }
}

// Redis cache with automatic TTL management
class SmartCache {
  constructor(redisClient, options = {}) {
    this.redis = redisClient;
    this.defaultTTL = options.defaultTTL || 3600;
    this.maxMemoryMB = options.maxMemoryMB || 512;
    
    // Configure Redis maxmemory
    this.configureRedis();
  }

  async configureRedis() {
    try {
      await this.redis.config('SET', 'maxmemory', `${this.maxMemoryMB}mb`);
      await this.redis.config('SET', 'maxmemory-policy', 'allkeys-lru');
      console.log(`Redis configured: maxmemory=${this.maxMemoryMB}mb, policy=allkeys-lru`);
    } catch (err) {
      console.warn('Cannot configure Redis:', err.message);
    }
  }

  async set(key, value, options = {}) {
    const ttl = options.ttl || this.defaultTTL;
    const serialized = JSON.stringify({
      value,
      cachedAt: Date.now(),
      ttl
    });
    
    if (ttl > 0) {
      await this.redis.setex(key, ttl, serialized);
    } else {
      await this.redis.set(key, serialized);
    }
  }

  async get(key) {
    const raw = await this.redis.get(key);
    if (!raw) return null;
    
    const { value, cachedAt, ttl } = JSON.parse(raw);
    
    return {
      value,
      cachedAt,
      age: Math.floor((Date.now() - cachedAt) / 1000),
      remainingTTL: ttl - Math.floor((Date.now() - cachedAt) / 1000)
    };
  }

  async getOrFetch(key, fetchFn, options = {}) {
    const cached = await this.get(key);
    
    if (cached && cached.remainingTTL > 0) {
      return { ...cached, fromCache: true };
    }
    
    // Fetch fresh data
    const value = await fetchFn();
    await this.set(key, value, options);
    
    return { value, fromCache: false, cachedAt: Date.now() };
  }

  async invalidate(...keys) {
    if (keys.length === 0) return 0;
    return this.redis.del(...keys);
  }

  async invalidatePattern(pattern) {
    const keys = await this.redis.keys(pattern);
    if (keys.length === 0) return 0;
    return this.redis.del(...keys);
  }

  async getMemoryUsage() {
    const info = await this.redis.info('memory');
    const usedMemory = parseInt(info.match(/used_memory:(\d+)/)?.[1] || '0');
    const maxMemory = parseInt(info.match(/maxmemory:(\d+)/)?.[1] || '0');
    
    return {
      used: `${Math.round(usedMemory / 1024 / 1024)}MB`,
      max: maxMemory > 0 ? `${Math.round(maxMemory / 1024 / 1024)}MB` : 'unlimited',
      usagePercent: maxMemory > 0 ? `${Math.round(usedMemory / maxMemory * 100)}%` : 'N/A'
    };
  }
}

module.exports = { LFUCache, SmartCache };
```

---

## ขั้นตอนที่ 1483: Cache Coherence Strategies

```javascript
// cache-coherence.js
// Ensuring cache consistency

const EventEmitter = require('events');

// Cache Invalidation Patterns

// 1. Write-Through: เขียน cache พร้อม database
class WriteThroughCache {
  constructor(db, cache) {
    this.db = db;
    this.cache = cache;
  }

  async set(key, value, ttl = 3600) {
    // เขียนทั้ง DB และ Cache พร้อมกัน
    await Promise.all([
      this.db.write('INSERT INTO cache_data (key, value) VALUES ($1, $2) ON CONFLICT (key) DO UPDATE SET value = $2', [key, JSON.stringify(value)]),
      this.cache.set(key, value, { ttl })
    ]);
  }

  async get(key) {
    const cached = await this.cache.get(key);
    if (cached) return cached.value;
    
    const dbResult = await this.db.read('SELECT value FROM cache_data WHERE key = $1', [key]);
    if (dbResult.rows[0]) {
      const value = JSON.parse(dbResult.rows[0].value);
      await this.cache.set(key, value);
      return value;
    }
    
    return null;
  }
}

// 2. Write-Behind (Write-Back): เขียน cache ก่อน แล้วค่อย sync DB
class WriteBehindCache {
  constructor(db, cache, flushInterval = 5000) {
    this.db = db;
    this.cache = cache;
    this.pendingWrites = new Map();
    this.flushInterval = flushInterval;
    
    this.startFlushing();
  }

  async set(key, value, ttl = 3600) {
    // เขียน cache ทันที
    await this.cache.set(key, value, { ttl });
    
    // Queue for DB write
    this.pendingWrites.set(key, { value, timestamp: Date.now() });
  }

  async get(key) {
    const cached = await this.cache.get(key);
    if (cached) return cached.value;
    
    // Cache miss - อ่านจาก DB
    const dbResult = await this.db.read('SELECT value FROM data WHERE key = $1', [key]);
    if (dbResult.rows[0]) {
      const value = JSON.parse(dbResult.rows[0].value);
      await this.cache.set(key, value);
      return value;
    }
    
    return null;
  }

  startFlushing() {
    this.flushTimer = setInterval(async () => {
      await this.flush();
    }, this.flushInterval);
  }

  async flush() {
    if (this.pendingWrites.size === 0) return;
    
    const batch = new Map(this.pendingWrites);
    this.pendingWrites.clear();
    
    try {
      // Batch write to DB
      const entries = Array.from(batch.entries());
      
      // Use a single batch insert
      const values = entries.map(([key, { value }]) => ({ key, value: JSON.stringify(value) }));
      
      // Bulk upsert
      for (const { key, value } of values) {
        await this.db.write(
          'INSERT INTO data (key, value) VALUES ($1, $2) ON CONFLICT (key) DO UPDATE SET value = $2',
          [key, value]
        );
      }
      
      console.log(`Flushed ${entries.length} pending writes to database`);
    } catch (err) {
      console.error('Flush failed:', err);
      // Re-add failed writes
      batch.forEach((value, key) => {
        this.pendingWrites.set(key, value);
      });
    }
  }

  stop() {
    clearInterval(this.flushTimer);
  }
}

// 3. Cache Invalidation via Events (Pub/Sub)
class EventDrivenCacheInvalidation extends EventEmitter {
  constructor(redisClient) {
    super();
    this.redis = redisClient;
    this.subscriber = redisClient.duplicate();
    
    this.setupSubscriptions();
  }

  setupSubscriptions() {
    // Subscribe to invalidation events
    this.subscriber.subscribe('cache:invalidate', (err) => {
      if (err) console.error('Subscription error:', err);
    });
    
    this.subscriber.on('message', (channel, message) => {
      if (channel === 'cache:invalidate') {
        const { keys, pattern } = JSON.parse(message);
        this.handleInvalidation(keys, pattern);
      }
    });
  }

  async handleInvalidation(keys, pattern) {
    if (keys && keys.length > 0) {
      await this.redis.del(...keys);
      this.emit('invalidated', { keys });
    }
    
    if (pattern) {
      const matchingKeys = await this.redis.keys(pattern);
      if (matchingKeys.length > 0) {
        await this.redis.del(...matchingKeys);
        this.emit('invalidated', { pattern, keys: matchingKeys });
      }
    }
  }

  async publishInvalidation(keys = [], pattern = null) {
    const message = JSON.stringify({ keys, pattern });
    
    // Publish to all cache nodes
    await this.redis.publish('cache:invalidate', message);
  }
}

// 4. Stale-While-Revalidate
class StaleWhileRevalidateCache {
  constructor(cache, options = {}) {
    this.cache = cache;
    this.staleTime = options.staleTime || 60;    // Consider stale after 60s
    this.maxAge = options.maxAge || 300;          // Delete after 300s
    this.revalidating = new Set();
  }

  async get(key, fetchFn) {
    const cached = await this.cache.get(key);
    
    if (!cached) {
      // Cache miss - fetch synchronously
      const value = await fetchFn();
      await this.setWithMetadata(key, value);
      return value;
    }
    
    const { value, cachedAt } = cached;
    const age = (Date.now() - cachedAt) / 1000;
    
    if (age > this.staleTime && !this.revalidating.has(key)) {
      // Stale - serve current value but revalidate in background
      this.revalidating.add(key);
      
      setImmediate(async () => {
        try {
          const freshValue = await fetchFn();
          await this.setWithMetadata(key, freshValue);
        } catch (err) {
          console.error(`Revalidation failed for ${key}:`, err);
        } finally {
          this.revalidating.delete(key);
        }
      });
    }
    
    return value;
  }

  async setWithMetadata(key, value) {
    await this.cache.set(key, value, { ttl: this.maxAge });
  }
}

module.exports = { WriteThroughCache, WriteBehindCache, EventDrivenCacheInvalidation, StaleWhileRevalidateCache };
```

---

## ขั้นตอนที่ 1484: Redis Pipeline และ Lua Scripts

```javascript
// redis-advanced.js
// Performance optimization techniques

const Redis = require('ioredis');
const redis = new Redis();

// 1. Pipeline - batch commands ลด round-trips
async function pipelineExample() {
  const pipeline = redis.pipeline();
  
  // Queue multiple commands
  pipeline.set('key1', 'value1');
  pipeline.set('key2', 'value2');
  pipeline.set('key3', 'value3');
  pipeline.get('key1');
  pipeline.get('key2');
  pipeline.incr('counter');
  
  // Execute all at once
  const results = await pipeline.exec();
  
  // results is array of [error, result] pairs
  results.forEach(([err, result], index) => {
    if (err) {
      console.error(`Command ${index} failed:`, err);
    } else {
      console.log(`Command ${index}: ${result}`);
    }
  });
  
  return results;
}

// 2. Lua Scripts - atomic operations
const atomicUpdateScript = `
  -- Atomic increment with limit
  local key = KEYS[1]
  local increment = tonumber(ARGV[1])
  local max_value = tonumber(ARGV[2])
  
  local current = tonumber(redis.call('GET', key)) or 0
  
  if current + increment > max_value then
    return {0, current}  -- Not incremented, return current value
  end
  
  local new_value = redis.call('INCRBY', key, increment)
  return {1, new_value}  -- Incremented, return new value
`;

const atomicUpdate = redis.defineCommand('atomicUpdate', {
  numberOfKeys: 1,
  lua: atomicUpdateScript
});

async function incrementWithLimit(key, amount, max) {
  const [success, value] = await redis.atomicUpdate(key, amount, max);
  return { success: success === 1, value: parseInt(value) };
}

// 3. Distributed Lock with Lua
const acquireLockScript = `
  local key = KEYS[1]
  local value = ARGV[1]
  local ttl = tonumber(ARGV[2])
  
  local existing = redis.call('GET', key)
  
  if existing == false then
    redis.call('SET', key, value, 'EX', ttl)
    return 1
  end
  
  return 0
`;

const releaseLockScript = `
  local key = KEYS[1]
  local value = ARGV[1]
  
  local existing = redis.call('GET', key)
  
  if existing == value then
    redis.call('DEL', key)
    return 1
  end
  
  return 0
`;

class DistributedLock {
  constructor(redisClient) {
    this.redis = redisClient;
    this.redis.defineCommand('acquireLock', {
      numberOfKeys: 1,
      lua: acquireLockScript
    });
    this.redis.defineCommand('releaseLock', {
      numberOfKeys: 1,
      lua: releaseLockScript
    });
  }

  async acquire(resource, ttlSeconds = 30) {
    const lockKey = `lock:${resource}`;
    const lockValue = `${process.pid}:${Date.now()}:${Math.random()}`;
    
    const acquired = await this.redis.acquireLock(lockKey, lockValue, ttlSeconds);
    
    if (acquired === 1) {
      return {
        acquired: true,
        release: () => this.release(resource, lockValue)
      };
    }
    
    return { acquired: false };
  }

  async acquireWithRetry(resource, options = {}) {
    const {
      ttlSeconds = 30,
      retries = 10,
      retryDelay = 200,
      jitter = 100
    } = options;
    
    for (let attempt = 0; attempt < retries; attempt++) {
      const result = await this.acquire(resource, ttlSeconds);
      if (result.acquired) return result;
      
      // Wait with jitter
      const delay = retryDelay + Math.random() * jitter;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
    
    throw new Error(`Failed to acquire lock for ${resource} after ${retries} attempts`);
  }

  async release(resource, lockValue) {
    const lockKey = `lock:${resource}`;
    return this.redis.releaseLock(lockKey, lockValue);
  }

  // Run function with lock
  async withLock(resource, fn, options = {}) {
    const lock = await this.acquireWithRetry(resource, options);
    
    try {
      return await fn();
    } finally {
      await lock.release();
    }
  }
}

// 4. Pub/Sub Pattern
class RedisPubSub {
  constructor() {
    this.publisher = new Redis();
    this.subscriber = new Redis();
    this.handlers = new Map();
    
    this.subscriber.on('message', (channel, message) => {
      const handlers = this.handlers.get(channel) || [];
      handlers.forEach(handler => {
        try {
          handler(JSON.parse(message));
        } catch (err) {
          console.error(`Handler error for channel ${channel}:`, err);
        }
      });
    });
  }

  async subscribe(channel, handler) {
    if (!this.handlers.has(channel)) {
      this.handlers.set(channel, []);
      await this.subscriber.subscribe(channel);
    }
    
    this.handlers.get(channel).push(handler);
    
    // Return unsubscribe function
    return () => {
      const handlers = this.handlers.get(channel);
      const index = handlers.indexOf(handler);
      if (index !== -1) handlers.splice(index, 1);
    };
  }

  async publish(channel, message) {
    return this.publisher.publish(channel, JSON.stringify(message));
  }

  async close() {
    await this.publisher.quit();
    await this.subscriber.quit();
  }
}

// 5. Redis Streams สำหรับ Event Log
class RedisEventStream {
  constructor(redisClient, streamName) {
    this.redis = redisClient;
    this.stream = streamName;
  }

  async publish(eventType, data) {
    const id = await this.redis.xadd(
      this.stream,
      '*',  // Auto-generate ID
      'type', eventType,
      'data', JSON.stringify(data),
      'timestamp', Date.now().toString()
    );
    
    return id;
  }

  async consume(groupName, consumerName, count = 10) {
    try {
      // Create consumer group if not exists
      await this.redis.xgroup('CREATE', this.stream, groupName, '0', 'MKSTREAM').catch(() => {});
    } catch (err) {
      // Group already exists
    }
    
    const messages = await this.redis.xreadgroup(
      'GROUP', groupName, consumerName,
      'COUNT', count,
      'BLOCK', 5000,
      'STREAMS', this.stream, '>'
    );
    
    if (!messages) return [];
    
    return messages[0][1].map(([id, fields]) => {
      const event = {};
      for (let i = 0; i < fields.length; i += 2) {
        event[fields[i]] = fields[i + 1];
      }
      return { id, ...event, data: JSON.parse(event.data) };
    });
  }

  async acknowledge(groupName, ...ids) {
    return this.redis.xack(this.stream, groupName, ...ids);
  }

  async getLength() {
    return this.redis.xlen(this.stream);
  }

  async trim(maxLength) {
    return this.redis.xtrim(this.stream, 'MAXLEN', '~', maxLength);
  }
}

// Express integration
const express = require('express');
const app = express();
app.use(express.json());

const distributedLock = new DistributedLock(redis);
const pubsub = new RedisPubSub();
const eventStream = new RedisEventStream(redis, 'app:events');

// Endpoint ที่ต้องการ distributed lock
app.post('/inventory/reserve', async (req, res) => {
  const { productId, quantity } = req.body;
  
  try {
    const result = await distributedLock.withLock(`inventory:${productId}`, async () => {
      // ตรวจสอบ inventory
      const current = await redis.get(`inventory:${productId}`);
      const available = parseInt(current || '0');
      
      if (available < quantity) {
        throw new Error(`Insufficient inventory: ${available} available, ${quantity} requested`);
      }
      
      // Reserve
      await redis.decrby(`inventory:${productId}`, quantity);
      
      // Log event
      await eventStream.publish('inventory_reserved', {
        productId,
        quantity,
        remaining: available - quantity
      });
      
      return { reserved: true, remaining: available - quantity };
    });
    
    res.json(result);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

module.exports = { DistributedLock, RedisPubSub, RedisEventStream };
```

---

## ขั้นตอนที่ 1485: Cache Warming Strategies

```javascript
// cache-warming.js
// Populate cache before traffic hits

class CacheWarmer {
  constructor(cache, dataSource) {
    this.cache = cache;
    this.dataSource = dataSource;
    this.isWarming = false;
  }

  async warmUp(strategy = 'eager') {
    if (this.isWarming) {
      console.log('Cache warming already in progress');
      return;
    }
    
    this.isWarming = true;
    console.log(`Starting cache warm-up (strategy: ${strategy})...`);
    
    try {
      switch (strategy) {
        case 'eager':
          await this.eagerWarmUp();
          break;
        case 'lazy':
          await this.lazyWarmUp();
          break;
        case 'predictive':
          await this.predictiveWarmUp();
          break;
      }
    } finally {
      this.isWarming = false;
      console.log('Cache warm-up complete');
    }
  }

  // Load all popular data immediately
  async eagerWarmUp() {
    const popularItems = await this.dataSource.getPopularItems(1000);
    
    const batches = this.chunk(popularItems, 100);
    
    for (const batch of batches) {
      await Promise.all(
        batch.map(item => 
          this.cache.set(`item:${item.id}`, item, { ttl: 3600 })
        )
      );
      
      // Small delay to avoid overwhelming Redis
      await new Promise(resolve => setTimeout(resolve, 10));
    }
    
    console.log(`Eagerly cached ${popularItems.length} items`);
  }

  // Load data on first access
  async lazyWarmUp() {
    // Pre-populate only critical data
    const criticalData = await this.dataSource.getCriticalData();
    
    await Promise.all(
      criticalData.map(data =>
        this.cache.set(`critical:${data.id}`, data, { ttl: 86400 })
      )
    );
    
    console.log(`Lazily cached ${criticalData.length} critical items`);
  }

  // Load based on predicted access patterns
  async predictiveWarmUp() {
    const hour = new Date().getHours();
    const dayOfWeek = new Date().getDay();
    
    // Load data based on time of day
    let dataToLoad;
    
    if (hour >= 9 && hour <= 18) {
      // Business hours - load business data
      dataToLoad = await this.dataSource.getBusinessHoursData();
    } else if (hour >= 18 && hour <= 22) {
      // Evening - load consumer data
      dataToLoad = await this.dataSource.getEveningData();
    } else {
      // Off hours - load minimal data
      dataToLoad = await this.dataSource.getMinimalData();
    }
    
    await Promise.all(
      dataToLoad.map(data =>
        this.cache.set(`data:${data.id}`, data, { ttl: 7200 })
      )
    );
    
    console.log(`Predictively cached ${dataToLoad.length} items for hour ${hour}`);
  }

  chunk(array, size) {
    const chunks = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }
}

// Cache Stampede Prevention (Dog-Pile Effect)
class StampedeProtectedCache {
  constructor(cache) {
    this.cache = cache;
    this.inFlight = new Map(); // Key -> Promise
  }

  async getOrFetch(key, fetchFn, ttl = 3600) {
    // Check cache first
    const cached = await this.cache.get(key);
    if (cached) return cached.value;
    
    // Check if there's already an in-flight request for this key
    if (this.inFlight.has(key)) {
      console.log(`Coalescing request for ${key}`);
      return this.inFlight.get(key);
    }
    
    // Start fetch and register as in-flight
    const fetchPromise = fetchFn()
      .then(async value => {
        await this.cache.set(key, value, { ttl });
        this.inFlight.delete(key);
        return value;
      })
      .catch(err => {
        this.inFlight.delete(key);
        throw err;
      });
    
    this.inFlight.set(key, fetchPromise);
    
    return fetchPromise;
  }
}

module.exports = { CacheWarmer, StampedeProtectedCache };
```

---

## ขั้นตอนที่ 1486-1500: Advanced Cache Patterns

### Multi-Region Cache Synchronization

```javascript
// multi-region-cache.js

class MultiRegionCache {
  constructor(regions) {
    this.regions = regions; // Map of region -> Redis client
    this.localRegion = process.env.REGION || 'us-east-1';
    this.localCache = this.regions.get(this.localRegion);
    
    // Subscribe to invalidation events from other regions
    this.setupCrossRegionSync();
  }

  setupCrossRegionSync() {
    this.regions.forEach((client, region) => {
      if (region === this.localRegion) return;
      
      const subscriber = client.duplicate();
      subscriber.subscribe('cache:invalidate');
      
      subscriber.on('message', (channel, message) => {
        const { key, sourceRegion } = JSON.parse(message);
        
        if (sourceRegion !== this.localRegion) {
          // Invalidate in local cache
          this.localCache.del(key).catch(err => 
            console.error(`Failed to invalidate ${key}:`, err)
          );
        }
      });
    });
  }

  async get(key) {
    // Read from local region cache first
    const value = await this.localCache.get(key);
    return value ? JSON.parse(value) : null;
  }

  async set(key, value, ttl = 3600) {
    const serialized = JSON.stringify(value);
    
    // Write to local region
    await this.localCache.setex(key, ttl, serialized);
    
    // Async replicate to other regions
    this.replicateAsync(key, serialized, ttl);
  }

  replicateAsync(key, value, ttl) {
    this.regions.forEach(async (client, region) => {
      if (region === this.localRegion) return;
      
      try {
        await client.setex(key, ttl, value);
      } catch (err) {
        console.error(`Failed to replicate to ${region}:`, err.message);
      }
    });
  }

  async invalidate(key) {
    // Publish invalidation event to all regions
    const message = JSON.stringify({
      key,
      sourceRegion: this.localRegion,
      timestamp: Date.now()
    });
    
    const publishPromises = Array.from(this.regions.values()).map(client =>
      client.publish('cache:invalidate', message).catch(() => {})
    );
    
    await Promise.all(publishPromises);
    
    // Also delete locally
    await this.localCache.del(key);
  }
}

// Express middleware สำหรับ cache headers
const cacheMiddleware = (options = {}) => {
  const {
    ttl = 300,           // 5 minutes default
    vary = ['Accept-Encoding'],
    staleWhileRevalidate = 60,
    staleIfError = 300
  } = options;

  return (req, res, next) => {
    // Set cache headers
    res.set({
      'Cache-Control': `public, max-age=${ttl}, stale-while-revalidate=${staleWhileRevalidate}, stale-if-error=${staleIfError}`,
      'Vary': vary.join(', ')
    });
    
    // Add ETag support
    const originalSend = res.send.bind(res);
    res.send = function(body) {
      if (typeof body === 'string' || Buffer.isBuffer(body)) {
        const crypto = require('crypto');
        const etag = crypto.createHash('md5').update(body).digest('hex');
        res.set('ETag', `"${etag}"`);
        
        // Check If-None-Match
        if (req.headers['if-none-match'] === `"${etag}"`) {
          return res.status(304).end();
        }
      }
      return originalSend(body);
    };
    
    next();
  };
};

module.exports = { MultiRegionCache, cacheMiddleware };
```

---

## แบบฝึกหัด

### Exercise 1: Implement Token Bucket Cache Rate Limiter
ใช้ Redis Lua script สร้าง token bucket rate limiter ที่ accurate แม้มีหลาย instances

### Exercise 2: Cache Analytics Dashboard
สร้าง dashboard แสดง cache hit rate, memory usage, และ top keys

### Exercise 3: Multi-Region Cache Replication
Setup Redis cluster ใน 2 regions พร้อม automatic replication และ failover

### คำถามทบทวน
1. ความแตกต่างระหว่าง Write-Through และ Write-Behind cache คืออะไร?
2. Cache Stampede (Dog-pile effect) คืออะไร และป้องกันได้อย่างไร?
3. เมื่อใดควรใช้ LRU แทน LFU eviction policy?
4. Distributed lock ต้องการคุณสมบัติอะไรบ้างเพื่อให้ safe?

---

*ต่อไป: Part 85 - Event-Driven Architecture*
