# Part 83 | ขั้นตอนที่ 1461-1480 จาก 1000+

## Database Sharding Strategies สำหรับ Node.js

ในส่วนนี้เราจะเรียนรู้การ shard database เพื่อรองรับข้อมูลและ traffic ปริมาณมากที่เกินความสามารถของ single server

---

## ขั้นตอนที่ 1461: Database Sharding Fundamentals

**Database Sharding** คือการแบ่งข้อมูลออกเป็นส่วนย่อยๆ (shards) และกระจายไปยัง database servers หลายตัว

### ประเภทของ Sharding

1. **Horizontal Sharding** - แบ่ง rows ไปยัง shards ต่างๆ
2. **Vertical Sharding** - แบ่ง columns หรือ tables ไปยัง databases ต่างๆ
3. **Functional Sharding** - แบ่งตาม business function

```javascript
// sharding-basics.js
// Basic sharding concepts

// Sharding Keys
const shardingStrategies = {
  // Strategy 1: Range-based sharding
  // ข้อดี: Query range ได้ง่าย
  // ข้อเสีย: Hot spots ถ้า access pattern ไม่สม่ำเสมอ
  rangeBasedShard(userId, numShards = 4) {
    const ranges = [
      { min: 0, max: 250000, shard: 0 },
      { min: 250001, max: 500000, shard: 1 },
      { min: 500001, max: 750000, shard: 2 },
      { min: 750001, max: Infinity, shard: 3 }
    ];
    
    const numId = parseInt(userId);
    const range = ranges.find(r => numId >= r.min && numId <= r.max);
    return range ? range.shard : 0;
  },

  // Strategy 2: Hash-based sharding
  // ข้อดี: กระจายข้อมูลได้สม่ำเสมอ
  // ข้อเสีย: Query range ยาก, rebalancing ยาก
  hashBasedShard(key, numShards = 4) {
    let hash = 0;
    const str = String(key);
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash;
    }
    return Math.abs(hash) % numShards;
  },

  // Strategy 3: Directory-based sharding
  // ข้อดี: Flexible, easy to rebalance
  // ข้อเสีย: Single point of failure สำหรับ directory
  directoryBasedShard: null, // ต้องมี lookup table
};

// ตัวอย่าง
console.log('Range sharding:');
[100, 300000, 600000, 900000].forEach(id => {
  console.log(`  User ${id} -> Shard ${shardingStrategies.rangeBasedShard(id)}`);
});

console.log('\nHash sharding:');
['user_001', 'user_002', 'user_003', 'user_004'].forEach(id => {
  console.log(`  ${id} -> Shard ${shardingStrategies.hashBasedShard(id)}`);
});
```

---

## ขั้นตอนที่ 1462: Consistent Hashing

**Consistent Hashing** แก้ปัญหาการ rebalance เมื่อเพิ่มหรือลด nodes

```javascript
// consistent-hashing.js
// Consistent hashing implementation

const crypto = require('crypto');

class ConsistentHashRing {
  constructor(options = {}) {
    this.virtualNodes = options.virtualNodes || 150; // Virtual replicas per node
    this.ring = new Map();
    this.sortedKeys = [];
    this.nodes = new Set();
  }

  // Hash function
  hash(key) {
    return parseInt(
      crypto.createHash('md5').update(String(key)).digest('hex').substring(0, 8),
      16
    );
  }

  // Add a node to the ring
  addNode(node) {
    this.nodes.add(node);
    
    for (let i = 0; i < this.virtualNodes; i++) {
      const virtualKey = this.hash(`${node}:${i}`);
      this.ring.set(virtualKey, node);
      this.sortedKeys.push(virtualKey);
    }
    
    // Keep sorted
    this.sortedKeys.sort((a, b) => a - b);
    
    console.log(`Added node: ${node} (${this.virtualNodes} virtual nodes)`);
    return this;
  }

  // Remove a node from the ring
  removeNode(node) {
    this.nodes.delete(node);
    
    for (let i = 0; i < this.virtualNodes; i++) {
      const virtualKey = this.hash(`${node}:${i}`);
      this.ring.delete(virtualKey);
      const index = this.sortedKeys.indexOf(virtualKey);
      if (index !== -1) {
        this.sortedKeys.splice(index, 1);
      }
    }
    
    console.log(`Removed node: ${node}`);
    return this;
  }

  // Get the node responsible for a key
  getNode(key) {
    if (this.sortedKeys.length === 0) {
      throw new Error('No nodes in the ring');
    }
    
    const keyHash = this.hash(key);
    
    // Binary search for next position
    let lo = 0;
    let hi = this.sortedKeys.length;
    
    while (lo < hi) {
      const mid = Math.floor((lo + hi) / 2);
      if (this.sortedKeys[mid] < keyHash) {
        lo = mid + 1;
      } else {
        hi = mid;
      }
    }
    
    // Wrap around to first node if past last
    const position = lo % this.sortedKeys.length;
    const ringKey = this.sortedKeys[position];
    
    return this.ring.get(ringKey);
  }

  // Get N nodes for replication
  getNodes(key, count = 3) {
    if (this.nodes.size < count) {
      return Array.from(this.nodes);
    }
    
    const nodes = new Set();
    const keyHash = this.hash(key);
    
    // Binary search starting position
    let lo = 0;
    let hi = this.sortedKeys.length;
    
    while (lo < hi) {
      const mid = Math.floor((lo + hi) / 2);
      if (this.sortedKeys[mid] < keyHash) {
        lo = mid + 1;
      } else {
        hi = mid;
      }
    }
    
    let pos = lo % this.sortedKeys.length;
    
    while (nodes.size < count) {
      const node = this.ring.get(this.sortedKeys[pos % this.sortedKeys.length]);
      nodes.add(node);
      pos++;
    }
    
    return Array.from(nodes);
  }

  getDistribution() {
    const counts = {};
    this.nodes.forEach(node => { counts[node] = 0; });
    
    this.ring.forEach((node) => {
      counts[node] = (counts[node] || 0) + 1;
    });
    
    return counts;
  }
}

// ใช้งาน
const ring = new ConsistentHashRing({ virtualNodes: 150 });

ring.addNode('shard-1:5432');
ring.addNode('shard-2:5432');
ring.addNode('shard-3:5432');

console.log('\nKey distribution (before adding node):');
const keys = ['user:1', 'user:2', 'user:3', 'user:4', 'user:5', 'order:100', 'product:abc'];
keys.forEach(key => {
  console.log(`  ${key} -> ${ring.getNode(key)}`);
});

console.log('\nAdding new node...');
ring.addNode('shard-4:5432');

console.log('\nKey distribution (after adding node):');
keys.forEach(key => {
  console.log(`  ${key} -> ${ring.getNode(key)}`);
});

// สังเกตว่าเพียงบาง keys เท่านั้นที่เปลี่ยน shard

module.exports = { ConsistentHashRing };
```

---

## ขั้นตอนที่ 1463: Sharding Router Implementation

```javascript
// shard-router.js
// Route queries to correct shards

const { Pool } = require('pg');

class ShardRouter {
  constructor(shardConfigs) {
    this.shards = new Map();
    this.ring = new ConsistentHashRing({ virtualNodes: 150 });
    
    // Initialize connection pools for each shard
    shardConfigs.forEach(config => {
      const pool = new Pool({
        host: config.host,
        port: config.port,
        database: config.database,
        user: config.user,
        password: config.password,
        max: 10,
        min: 2
      });
      
      this.shards.set(config.id, {
        id: config.id,
        pool,
        config,
        healthy: true,
        queryCount: 0
      });
      
      this.ring.addNode(config.id);
    });
  }

  getShardForKey(shardKey) {
    const shardId = this.ring.getNode(String(shardKey));
    const shard = this.shards.get(shardId);
    
    if (!shard || !shard.healthy) {
      // Try to find another healthy shard
      const healthyShards = Array.from(this.shards.values()).filter(s => s.healthy);
      if (healthyShards.length === 0) throw new Error('No healthy shards available');
      return healthyShards[0];
    }
    
    return shard;
  }

  async query(shardKey, sql, params = []) {
    const shard = this.getShardForKey(shardKey);
    shard.queryCount++;
    
    try {
      const result = await shard.pool.query(sql, params);
      return result;
    } catch (err) {
      if (this.isConnectionError(err)) {
        shard.healthy = false;
        // Retry on different shard
        const healthyShard = Array.from(this.shards.values()).find(s => s.healthy && s.id !== shard.id);
        if (healthyShard) {
          return healthyShard.pool.query(sql, params);
        }
      }
      throw err;
    }
  }

  // Cross-shard query (เช่น aggregate)
  async queryAllShards(sql, params = []) {
    const promises = Array.from(this.shards.values())
      .filter(shard => shard.healthy)
      .map(shard => 
        shard.pool.query(sql, params)
          .then(result => ({ shardId: shard.id, rows: result.rows }))
          .catch(err => ({ shardId: shard.id, error: err.message, rows: [] }))
      );
    
    const results = await Promise.all(promises);
    
    // Merge results
    return {
      rows: results.flatMap(r => r.rows),
      shards: results.map(r => ({ id: r.shardId, error: r.error, count: r.rows.length }))
    };
  }

  // Scatter-gather for specific shards
  async queryShards(shardIds, sql, params = []) {
    const promises = shardIds
      .map(id => this.shards.get(id))
      .filter(shard => shard && shard.healthy)
      .map(shard => shard.pool.query(sql, params).then(r => r.rows));
    
    const results = await Promise.all(promises);
    return results.flat();
  }

  isConnectionError(err) {
    return ['ECONNREFUSED', 'ETIMEDOUT', 'ENOTFOUND'].includes(err.code);
  }

  getStats() {
    const stats = {};
    this.shards.forEach((shard, id) => {
      stats[id] = {
        healthy: shard.healthy,
        queryCount: shard.queryCount,
        poolStats: {
          total: shard.pool.totalCount,
          idle: shard.pool.idleCount,
          waiting: shard.pool.waitingCount
        }
      };
    });
    return stats;
  }

  async close() {
    const closePromises = Array.from(this.shards.values()).map(s => s.pool.end());
    await Promise.all(closePromises);
  }
}

// Repository ที่ใช้ ShardRouter
class UserRepository {
  constructor(shardRouter) {
    this.router = shardRouter;
  }

  async findById(userId) {
    const result = await this.router.query(
      userId, // Shard key = userId
      'SELECT * FROM users WHERE id = $1',
      [userId]
    );
    return result.rows[0] || null;
  }

  async create(user) {
    const result = await this.router.query(
      user.id, // Shard key = userId
      'INSERT INTO users (id, email, name, created_at) VALUES ($1, $2, $3, NOW()) RETURNING *',
      [user.id, user.email, user.name]
    );
    return result.rows[0];
  }

  async update(userId, updates) {
    const setClauses = Object.keys(updates).map((k, i) => `${k} = $${i + 2}`).join(', ');
    const values = [userId, ...Object.values(updates)];
    
    const result = await this.router.query(
      userId,
      `UPDATE users SET ${setClauses} WHERE id = $1 RETURNING *`,
      values
    );
    return result.rows[0];
  }

  async delete(userId) {
    await this.router.query(
      userId,
      'DELETE FROM users WHERE id = $1',
      [userId]
    );
  }

  // ค้นหาแบบ cross-shard (expensive operation!)
  async findByEmail(email) {
    const result = await this.router.queryAllShards(
      'SELECT * FROM users WHERE email = $1',
      [email]
    );
    return result.rows[0] || null;
  }

  // Get total count across all shards
  async getTotalCount() {
    const result = await this.router.queryAllShards(
      'SELECT COUNT(*) as count FROM users'
    );
    
    const total = result.rows.reduce((sum, row) => sum + parseInt(row.count), 0);
    return total;
  }
}

module.exports = { ShardRouter, UserRepository };
```

---

## ขั้นตอนที่ 1464: Read Replicas Pattern

```javascript
// read-replicas.js
// Setup และ use read replicas

const { Pool } = require('pg');

class DatabaseCluster {
  constructor(config) {
    // Primary (write operations)
    this.primary = new Pool({
      host: config.primary.host,
      port: config.primary.port || 5432,
      database: config.database,
      user: config.user,
      password: config.password,
      max: config.primary.maxConnections || 20
    });

    // Replicas (read operations)
    this.replicas = config.replicas.map((replicaConfig, index) => ({
      id: `replica-${index}`,
      pool: new Pool({
        host: replicaConfig.host,
        port: replicaConfig.port || 5432,
        database: config.database,
        user: config.user,
        password: config.password,
        max: replicaConfig.maxConnections || 10
      }),
      weight: replicaConfig.weight || 1,
      region: replicaConfig.region || 'default',
      healthy: true,
      lagSeconds: 0
    }));

    this.replicaIndex = 0;
    this.startLagMonitoring();
  }

  // Write to primary
  async write(sql, params = []) {
    try {
      return await this.primary.query(sql, params);
    } catch (err) {
      throw new Error(`Primary write failed: ${err.message}`);
    }
  }

  // Read from replica (with automatic fallback)
  async read(sql, params = [], options = {}) {
    const replica = this.selectReplica(options);
    
    if (!replica) {
      // Fallback to primary
      console.warn('No healthy replicas, reading from primary');
      return this.primary.query(sql, params);
    }
    
    try {
      return await replica.pool.query(sql, params);
    } catch (err) {
      replica.healthy = false;
      console.warn(`Replica ${replica.id} failed, falling back to primary`);
      return this.primary.query(sql, params);
    }
  }

  selectReplica(options = {}) {
    let availableReplicas = this.replicas.filter(r => r.healthy);
    
    // Filter by region if specified
    if (options.region) {
      const regionReplicas = availableReplicas.filter(r => r.region === options.region);
      if (regionReplicas.length > 0) {
        availableReplicas = regionReplicas;
      }
    }
    
    // Filter by max lag if specified
    if (options.maxLagSeconds !== undefined) {
      availableReplicas = availableReplicas.filter(r => r.lagSeconds <= options.maxLagSeconds);
    }
    
    if (availableReplicas.length === 0) return null;
    
    // Weighted round-robin selection
    const totalWeight = availableReplicas.reduce((sum, r) => sum + r.weight, 0);
    let random = Math.random() * totalWeight;
    
    for (const replica of availableReplicas) {
      random -= replica.weight;
      if (random <= 0) return replica;
    }
    
    return availableReplicas[availableReplicas.length - 1];
  }

  // Transaction - must use primary
  async transaction(callback) {
    const client = await this.primary.connect();
    
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

  // Monitor replication lag
  startLagMonitoring() {
    setInterval(async () => {
      for (const replica of this.replicas) {
        try {
          const result = await replica.pool.query(`
            SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))::int AS lag_seconds
          `);
          
          replica.lagSeconds = result.rows[0]?.lag_seconds || 0;
          replica.healthy = true;
          
          if (replica.lagSeconds > 30) {
            console.warn(`Replica ${replica.id} lag: ${replica.lagSeconds}s`);
          }
        } catch (err) {
          replica.healthy = false;
          console.error(`Replica ${replica.id} health check failed:`, err.message);
        }
      }
    }, 10000);
  }

  getClusterStats() {
    return {
      primary: {
        total: this.primary.totalCount,
        idle: this.primary.idleCount,
        waiting: this.primary.waitingCount
      },
      replicas: this.replicas.map(r => ({
        id: r.id,
        region: r.region,
        healthy: r.healthy,
        lagSeconds: r.lagSeconds,
        weight: r.weight,
        pool: {
          total: r.pool.totalCount,
          idle: r.pool.idleCount,
          waiting: r.pool.waitingCount
        }
      }))
    };
  }

  async close() {
    await this.primary.end();
    await Promise.all(this.replicas.map(r => r.pool.end()));
  }
}

// ตัวอย่างการใช้งาน
const db = new DatabaseCluster({
  database: 'myapp',
  user: 'postgres',
  password: 'secret',
  primary: {
    host: 'primary.db.example.com',
    maxConnections: 20
  },
  replicas: [
    {
      host: 'replica-1.db.example.com',
      region: 'us-east-1',
      weight: 3,
      maxConnections: 10
    },
    {
      host: 'replica-2.db.example.com',
      region: 'us-west-2',
      weight: 2,
      maxConnections: 10
    }
  ]
});

// Express API
const express = require('express');
const app = express();
app.use(express.json());

// Write operation - ใช้ primary
app.post('/users', async (req, res) => {
  try {
    const result = await db.write(
      'INSERT INTO users (id, email, name) VALUES ($1, $2, $3) RETURNING *',
      [req.body.id, req.body.email, req.body.name]
    );
    res.json(result.rows[0]);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Read operation - ใช้ replica
app.get('/users/:id', async (req, res) => {
  try {
    const result = await db.read(
      'SELECT * FROM users WHERE id = $1',
      [req.params.id],
      { maxLagSeconds: 10 } // ยอมรับ lag ไม่เกิน 10 วินาที
    );
    
    if (!result.rows[0]) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json(result.rows[0]);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Cluster stats
app.get('/db/stats', (req, res) => {
  res.json(db.getClusterStats());
});

module.exports = { DatabaseCluster };
```

---

## ขั้นตอนที่ 1465: Cross-Shard Transactions

```javascript
// cross-shard-transactions.js
// Handling transactions across multiple shards

// Saga Pattern สำหรับ distributed transactions
class SagaOrchestrator {
  constructor(stepDefs) {
    this.steps = stepDefs;
    this.executionLog = [];
  }

  async execute(context) {
    const executed = [];
    
    for (const step of this.steps) {
      try {
        console.log(`Executing step: ${step.name}`);
        const result = await step.execute(context);
        context[step.outputKey] = result;
        executed.push(step);
        
        this.executionLog.push({
          step: step.name,
          status: 'success',
          timestamp: new Date().toISOString()
        });
        
      } catch (err) {
        console.error(`Step ${step.name} failed:`, err.message);
        
        this.executionLog.push({
          step: step.name,
          status: 'failed',
          error: err.message,
          timestamp: new Date().toISOString()
        });
        
        // Compensate in reverse order
        console.log('Starting compensation...');
        await this.compensate(executed, context);
        
        throw new Error(`Saga failed at step ${step.name}: ${err.message}`);
      }
    }
    
    return context;
  }

  async compensate(executedSteps, context) {
    const reversedSteps = [...executedSteps].reverse();
    
    for (const step of reversedSteps) {
      if (step.compensate) {
        try {
          console.log(`Compensating: ${step.name}`);
          await step.compensate(context);
          
          this.executionLog.push({
            step: `compensate:${step.name}`,
            status: 'success',
            timestamp: new Date().toISOString()
          });
        } catch (compensateErr) {
          console.error(`Compensation failed for ${step.name}:`, compensateErr.message);
          // Log but continue compensating other steps
        }
      }
    }
  }
}

// Order Fulfillment Saga
class OrderFulfillmentSaga {
  constructor({ orderShard, inventoryShard, paymentShard, notificationService }) {
    this.orchestrator = new SagaOrchestrator([
      {
        name: 'ReserveInventory',
        outputKey: 'reservation',
        execute: async (ctx) => {
          const reservation = await inventoryShard.query(ctx.orderData.userId,
            'INSERT INTO reservations (order_id, items) VALUES ($1, $2) RETURNING *',
            [ctx.orderId, JSON.stringify(ctx.orderData.items)]
          );
          return reservation.rows[0];
        },
        compensate: async (ctx) => {
          await inventoryShard.query(ctx.orderData.userId,
            'DELETE FROM reservations WHERE order_id = $1',
            [ctx.orderId]
          );
        }
      },
      {
        name: 'ProcessPayment',
        outputKey: 'payment',
        execute: async (ctx) => {
          const payment = await paymentShard.query(ctx.orderData.userId,
            'INSERT INTO payments (order_id, amount, status) VALUES ($1, $2, $3) RETURNING *',
            [ctx.orderId, ctx.orderData.total, 'captured']
          );
          return payment.rows[0];
        },
        compensate: async (ctx) => {
          if (ctx.payment) {
            await paymentShard.query(ctx.orderData.userId,
              'UPDATE payments SET status = $1 WHERE id = $2',
              ['refunded', ctx.payment.id]
            );
          }
        }
      },
      {
        name: 'CreateOrder',
        outputKey: 'order',
        execute: async (ctx) => {
          const order = await orderShard.query(ctx.orderData.userId,
            'INSERT INTO orders (id, user_id, items, total, status) VALUES ($1, $2, $3, $4, $5) RETURNING *',
            [ctx.orderId, ctx.orderData.userId, JSON.stringify(ctx.orderData.items), ctx.orderData.total, 'confirmed']
          );
          return order.rows[0];
        },
        compensate: async (ctx) => {
          await orderShard.query(ctx.orderData.userId,
            'UPDATE orders SET status = $1 WHERE id = $2',
            ['cancelled', ctx.orderId]
          );
        }
      },
      {
        name: 'SendNotification',
        outputKey: 'notification',
        execute: async (ctx) => {
          await notificationService.send({
            userId: ctx.orderData.userId,
            type: 'order_confirmed',
            orderId: ctx.orderId
          });
          return { sent: true };
        },
        compensate: async (ctx) => {
          // Not critical - just log
          console.log(`Notification compensation for order ${ctx.orderId} not needed`);
        }
      }
    ]);
  }

  async execute(orderData) {
    const context = {
      orderId: require('crypto').randomUUID(),
      orderData
    };
    
    return this.orchestrator.execute(context);
  }
}

// Two-Phase Commit (2PC) - สำหรับ cases ที่ต้องการ strict consistency
class TwoPhaseCommit {
  constructor(participants) {
    this.participants = participants;
    this.coordinatorLog = [];
  }

  async commit(operations) {
    const transactionId = require('crypto').randomUUID();
    
    try {
      // Phase 1: PREPARE
      console.log(`2PC: Phase 1 - Prepare (txn: ${transactionId})`);
      const prepareResults = await Promise.all(
        operations.map(({ participant, operation }) => 
          this.participants[participant].prepare(transactionId, operation)
        )
      );
      
      const allPrepared = prepareResults.every(r => r.ready);
      
      if (!allPrepared) {
        // ถ้ามีใครไม่พร้อม ให้ abort ทุกคน
        await this.abort(transactionId, operations);
        throw new Error('2PC: Prepare phase failed');
      }
      
      // Phase 2: COMMIT
      console.log(`2PC: Phase 2 - Commit (txn: ${transactionId})`);
      await Promise.all(
        operations.map(({ participant }) => 
          this.participants[participant].commit(transactionId)
        )
      );
      
      this.coordinatorLog.push({ transactionId, status: 'committed', timestamp: new Date() });
      return { success: true, transactionId };
      
    } catch (err) {
      this.coordinatorLog.push({ transactionId, status: 'failed', error: err.message, timestamp: new Date() });
      throw err;
    }
  }

  async abort(transactionId, operations) {
    await Promise.allSettled(
      operations.map(({ participant }) => 
        this.participants[participant].abort(transactionId)
      )
    );
  }
}

module.exports = { SagaOrchestrator, OrderFulfillmentSaga, TwoPhaseCommit };
```

---

## ขั้นตอนที่ 1466: Shard Rebalancing

```javascript
// shard-rebalancing.js
// Migrate data between shards

class ShardRebalancer {
  constructor(sourceShards, targetShards) {
    this.sourceShards = sourceShards;
    this.targetShards = targetShards;
    this.migrationLog = [];
  }

  async rebalance(table, shardKeyColumn) {
    console.log(`Starting rebalance of ${table}...`);
    
    const newRouter = new ConsistentHashRing({ virtualNodes: 150 });
    this.targetShards.forEach(shard => newRouter.addNode(shard.id));
    
    // Process in batches to avoid downtime
    const batchSize = 1000;
    let offset = 0;
    let totalMigrated = 0;
    
    while (true) {
      // Get batch of records from ALL source shards
      const allRows = await this.getAllRows(table, batchSize, offset);
      
      if (allRows.length === 0) break;
      
      // Determine new shard for each row
      const migrations = allRows.map(row => ({
        row,
        sourceShard: this.getCurrentShard(row[shardKeyColumn]),
        targetShard: newRouter.getNode(row[shardKeyColumn])
      }));
      
      // Only migrate rows that need to move
      const toMigrate = migrations.filter(m => m.sourceShard !== m.targetShard);
      
      if (toMigrate.length > 0) {
        await this.migrateBatch(table, toMigrate);
        totalMigrated += toMigrate.length;
      }
      
      offset += batchSize;
      
      console.log(`Progress: ${offset} records processed, ${totalMigrated} migrated`);
      
      // Small delay to avoid overwhelming the system
      await new Promise(resolve => setTimeout(resolve, 100));
    }
    
    console.log(`Rebalance complete. Total migrated: ${totalMigrated}`);
    return { totalMigrated };
  }

  async getAllRows(table, limit, offset) {
    // Query all source shards
    const results = await Promise.all(
      this.sourceShards.map(shard => 
        shard.pool.query(
          `SELECT * FROM ${table} LIMIT $1 OFFSET $2`,
          [limit, offset]
        ).then(r => r.rows)
      )
    );
    
    return results.flat();
  }

  async migrateBatch(table, migrations) {
    // Group by target shard
    const byTargetShard = {};
    migrations.forEach(m => {
      if (!byTargetShard[m.targetShard]) byTargetShard[m.targetShard] = [];
      byTargetShard[m.targetShard].push(m);
    });
    
    // Insert into target shards
    await Promise.all(
      Object.entries(byTargetShard).map(([shardId, items]) => 
        this.insertBatch(shardId, table, items.map(i => i.row))
      )
    );
    
    // Delete from source shards
    const bySourceShard = {};
    migrations.forEach(m => {
      if (!bySourceShard[m.sourceShard]) bySourceShard[m.sourceShard] = [];
      bySourceShard[m.sourceShard].push(m.row.id);
    });
    
    await Promise.all(
      Object.entries(bySourceShard).map(([shardId, ids]) => 
        this.deleteBatch(shardId, table, ids)
      )
    );
  }

  async insertBatch(shardId, table, rows) {
    const shard = this.targetShards.find(s => s.id === shardId);
    if (!shard) throw new Error(`Target shard ${shardId} not found`);
    
    if (rows.length === 0) return;
    
    const columns = Object.keys(rows[0]);
    const values = rows.map(row => Object.values(row));
    
    const placeholders = values.map((_, rowIndex) => 
      `(${columns.map((_, colIndex) => `$${rowIndex * columns.length + colIndex + 1}`).join(', ')})`
    ).join(', ');
    
    await shard.pool.query(
      `INSERT INTO ${table} (${columns.join(', ')}) VALUES ${placeholders} ON CONFLICT DO NOTHING`,
      values.flat()
    );
  }

  async deleteBatch(shardId, table, ids) {
    const shard = this.sourceShards.find(s => s.id === shardId);
    if (!shard) return;
    
    await shard.pool.query(
      `DELETE FROM ${table} WHERE id = ANY($1)`,
      [ids]
    );
  }

  getCurrentShard(key) {
    // Determine current shard based on old routing
    const oldRouter = new ConsistentHashRing({ virtualNodes: 150 });
    this.sourceShards.forEach(shard => oldRouter.addNode(shard.id));
    return oldRouter.getNode(String(key));
  }
}

module.exports = { ShardRebalancer };
```

---

## ขั้นตอนที่ 1467-1480: Advanced Sharding Topics

### Global Tables vs Sharded Tables

```javascript
// global-vs-sharded.js
// ตัดสินใจว่าข้อมูลไหนควร global ข้อมูลไหนควร shard

/*
Global Tables (เก็บใน ทุก shard):
- Countries/Currencies/Timezones
- Configuration/Settings
- Static reference data
- เปลี่ยนแปลงน้อยมาก

Sharded Tables:
- Users, Orders, Sessions
- Transaction records
- Frequently changing data
*/

class SmartShardRouter {
  constructor(shards, globalDb) {
    this.shards = shards;
    this.globalDb = globalDb; // Database สำหรับ global tables
    
    this.globalTables = new Set([
      'countries', 'currencies', 'categories', 'configurations'
    ]);
  }

  isGlobalTable(tableName) {
    return this.globalTables.has(tableName.toLowerCase());
  }

  async query(tableName, shardKey, sql, params = []) {
    if (this.isGlobalTable(tableName)) {
      // Query global table
      return this.globalDb.query(sql, params);
    } else {
      // Route to appropriate shard
      const shard = this.getShardForKey(shardKey);
      return shard.pool.query(sql, params);
    }
  }

  getShardForKey(key) {
    const hash = this.hash(String(key));
    const index = hash % this.shards.length;
    return this.shards[index];
  }

  hash(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }
}

// Cross-Shard Join (ควรหลีกเลี่ยงถ้าทำได้)
class CrossShardJoin {
  constructor(router) {
    this.router = router;
  }

  // Strategy: Fetch and join in application layer
  async joinUserOrders(userId) {
    // Fetch user from shard
    const userResult = await this.router.query('users', userId,
      'SELECT * FROM users WHERE id = $1', [userId]);
    
    if (userResult.rows.length === 0) return null;
    
    const user = userResult.rows[0];
    
    // Fetch orders from shard (same shard key = userId)
    const ordersResult = await this.router.query('orders', userId,
      'SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC', [userId]);
    
    // Fetch country from global table
    const countryResult = await this.router.query('countries', null,
      'SELECT * FROM countries WHERE code = $1', [user.country_code]);
    
    return {
      ...user,
      country: countryResult.rows[0] || null,
      orders: ordersResult.rows
    };
  }
}

module.exports = { SmartShardRouter, CrossShardJoin };
```

---

## แบบฝึกหัด

### Exercise 1: Implement Hash Sharding
สร้าง hash-based sharding สำหรับ `messages` table โดยใช้ `conversation_id` เป็น shard key

### Exercise 2: Migration Tool
เขียน script ที่ migrate ข้อมูลจาก single database ไปยัง 4 shards โดยไม่มี downtime

### Exercise 3: Cross-Shard Analytics
สร้าง system ที่ aggregate ข้อมูลจาก shards ทั้งหมดเพื่อทำ analytics รายงาน

### คำถามทบทวน
1. Consistent hashing แก้ปัญหา rehashing ได้อย่างไร?
2. เมื่อใดควรใช้ Saga pattern แทน Two-Phase Commit?
3. Global tables vs Sharded tables ควรใช้เมื่อใด?
4. ปัญหาหลักของ cross-shard joins คืออะไร และแก้ได้อย่างไร?

---

*ต่อไป: Part 84 - Distributed Cache*
