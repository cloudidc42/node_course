# Part 82 | ขั้นตอนที่ 1441-1460 จาก 1000+

## High Availability Architecture สำหรับ Node.js Applications

ในส่วนนี้เราจะเรียนรู้การสร้างระบบที่มี availability สูง (99.99% uptime) รวมถึง failover, health checks, และ zero-downtime deployments

---

## ขั้นตอนที่ 1441: ความเข้าใจเรื่อง High Availability

**High Availability (HA)** หมายถึงระบบที่สามารถทำงานได้อย่างต่อเนื่องโดยมี downtime น้อยที่สุด

| Availability | Downtime ต่อปี | Downtime ต่อเดือน |
|-------------|--------------|----------------|
| 99% (two nines) | 87.6 ชั่วโมง | 7.3 ชั่วโมง |
| 99.9% (three nines) | 8.76 ชั่วโมง | 43.8 นาที |
| 99.99% (four nines) | 52.6 นาที | 4.4 นาที |
| 99.999% (five nines) | 5.26 นาที | 26 วินาที |

```javascript
// ha-config.js
// Configuration สำหรับ HA setup

const haConfig = {
  // Redundancy
  minInstances: 3,        // ต้องมีอย่างน้อย 3 instances
  maxInstances: 10,       // Scale สูงสุด 10 instances
  
  // Health checks
  healthCheckInterval: 10000,  // ตรวจสอบทุก 10 วินาที
  healthCheckTimeout: 5000,    // Timeout 5 วินาที
  unhealthyThreshold: 3,       // ถือว่า unhealthy เมื่อ fail 3 ครั้งติดต่อกัน
  
  // Failover
  failoverDelay: 5000,         // รอ 5 วินาทีก่อน failover
  maxRetries: 3,               // Retry สูงสุด 3 ครั้ง
  
  // Deployment
  rollingUpdateBatchSize: 1,   // Update ทีละ 1 instance
  minHealthyPercent: 66,       // ต้องมี healthy instances อย่างน้อย 66%
};

module.exports = haConfig;
```

---

## ขั้นตอนที่ 1442: Health Check Implementation

```javascript
// health-check.js
// Comprehensive health check system

const express = require('express');
const mongoose = require('mongoose');
const Redis = require('ioredis');
const { Pool } = require('pg');

const app = express();
const redis = new Redis();
const pgPool = new Pool({ connectionString: process.env.DATABASE_URL });

// Health check registry
class HealthCheckRegistry {
  constructor() {
    this.checks = new Map();
    this.results = new Map();
  }

  register(name, checkFn, options = {}) {
    this.checks.set(name, {
      fn: checkFn,
      timeout: options.timeout || 5000,
      critical: options.critical !== false, // Critical by default
      interval: options.interval || 30000
    });
  }

  async runCheck(name) {
    const check = this.checks.get(name);
    if (!check) throw new Error(`Health check '${name}' not found`);

    const start = Date.now();
    
    try {
      const result = await Promise.race([
        check.fn(),
        new Promise((_, reject) => 
          setTimeout(() => reject(new Error('Health check timeout')), check.timeout)
        )
      ]);
      
      const checkResult = {
        status: 'healthy',
        duration: Date.now() - start,
        result,
        timestamp: new Date().toISOString()
      };
      
      this.results.set(name, checkResult);
      return checkResult;
      
    } catch (error) {
      const checkResult = {
        status: 'unhealthy',
        duration: Date.now() - start,
        error: error.message,
        critical: check.critical,
        timestamp: new Date().toISOString()
      };
      
      this.results.set(name, checkResult);
      return checkResult;
    }
  }

  async runAll() {
    const promises = Array.from(this.checks.keys()).map(name => 
      this.runCheck(name).then(result => ({ name, ...result }))
    );
    
    const results = await Promise.allSettled(promises);
    
    const checks = {};
    let overallStatus = 'healthy';
    
    results.forEach(result => {
      if (result.status === 'fulfilled') {
        const { name, ...checkResult } = result.value;
        checks[name] = checkResult;
        
        if (checkResult.status === 'unhealthy' && checkResult.critical) {
          overallStatus = 'unhealthy';
        } else if (checkResult.status === 'unhealthy') {
          overallStatus = overallStatus === 'unhealthy' ? 'unhealthy' : 'degraded';
        }
      }
    });
    
    return { status: overallStatus, checks };
  }
}

// Create registry and register checks
const registry = new HealthCheckRegistry();

// Database health check
registry.register('database', async () => {
  const client = await pgPool.connect();
  try {
    const result = await client.query('SELECT 1 + 1 as result');
    return { connected: true, result: result.rows[0].result };
  } finally {
    client.release();
  }
}, { critical: true, timeout: 5000 });

// Redis health check
registry.register('redis', async () => {
  await redis.ping();
  const info = await redis.info('server');
  const version = info.match(/redis_version:(.+)/)?.[1]?.trim();
  return { connected: true, version };
}, { critical: true, timeout: 3000 });

// Memory health check
registry.register('memory', async () => {
  const memUsage = process.memoryUsage();
  const heapUsedMB = Math.round(memUsage.heapUsed / 1024 / 1024);
  const heapTotalMB = Math.round(memUsage.heapTotal / 1024 / 1024);
  const usagePercent = Math.round(heapUsedMB / heapTotalMB * 100);
  
  if (usagePercent > 90) {
    throw new Error(`Memory usage critical: ${usagePercent}%`);
  }
  
  return {
    heapUsed: `${heapUsedMB}MB`,
    heapTotal: `${heapTotalMB}MB`,
    usagePercent: `${usagePercent}%`
  };
}, { critical: false });

// Disk space check
registry.register('disk', async () => {
  const { execSync } = require('child_process');
  const output = execSync("df -h / | tail -1 | awk '{print $5}'").toString().trim();
  const usagePercent = parseInt(output);
  
  if (usagePercent > 90) {
    throw new Error(`Disk usage critical: ${usagePercent}%`);
  }
  
  return { usagePercent: `${usagePercent}%` };
}, { critical: false });

// External dependency check
registry.register('payment-service', async () => {
  const response = await fetch('https://api.payment.example.com/health', {
    signal: AbortSignal.timeout(5000)
  });
  return { available: response.ok, statusCode: response.status };
}, { critical: false }); // Not critical - graceful degradation

// Health check endpoints
app.get('/health', async (req, res) => {
  const result = await registry.runAll();
  const statusCode = result.status === 'healthy' ? 200 : 
                     result.status === 'degraded' ? 200 : 503;
  
  res.status(statusCode).json(result);
});

// Liveness probe - ระบบยังทำงานอยู่หรือไม่ (สำหรับ Kubernetes)
app.get('/health/live', (req, res) => {
  // แค่ตรวจสอบว่า process ยังทำงานอยู่
  res.json({ status: 'alive', uptime: process.uptime() });
});

// Readiness probe - ระบบพร้อมรับ traffic หรือยัง (สำหรับ Kubernetes)
app.get('/health/ready', async (req, res) => {
  try {
    // ตรวจสอบ critical dependencies
    await pgPool.query('SELECT 1');
    await redis.ping();
    
    res.json({ status: 'ready' });
  } catch (err) {
    res.status(503).json({ status: 'not ready', error: err.message });
  }
});

// Startup probe - ระบบ initialized แล้วหรือยัง
app.get('/health/startup', async (req, res) => {
  try {
    // ตรวจสอบว่า initialization เสร็จสมบูรณ์
    if (!global.isInitialized) {
      return res.status(503).json({ status: 'initializing' });
    }
    
    res.json({ status: 'started' });
  } catch (err) {
    res.status(503).json({ status: 'failed', error: err.message });
  }
});

module.exports = { app, registry };
```

---

## ขั้นตอนที่ 1443: Graceful Shutdown

```javascript
// graceful-shutdown.js
// ปิด server อย่างถูกต้อง ไม่ทำให้ requests หาย

const express = require('express');
const http = require('http');

class GracefulShutdown {
  constructor(app, options = {}) {
    this.options = {
      timeout: options.timeout || 30000,  // 30 seconds max
      forceTimeout: options.forceTimeout || 5000,
      signals: options.signals || ['SIGTERM', 'SIGINT'],
      ...options
    };
    
    this.server = null;
    this.isShuttingDown = false;
    this.activeConnections = new Set();
    this.app = app;
  }

  setup() {
    // Create HTTP server
    this.server = http.createServer(this.app);
    
    // Track active connections
    this.server.on('connection', (socket) => {
      this.activeConnections.add(socket);
      socket.on('close', () => {
        this.activeConnections.delete(socket);
      });
    });
    
    // ป้องกัน new connections เมื่อกำลัง shutdown
    this.app.use((req, res, next) => {
      if (this.isShuttingDown) {
        res.set('Connection', 'close');
        res.status(503).json({ error: 'Server is shutting down' });
        return;
      }
      next();
    });
    
    // Register signal handlers
    this.options.signals.forEach(signal => {
      process.on(signal, () => this.shutdown(signal));
    });
    
    // Unhandled errors
    process.on('uncaughtException', (err) => {
      console.error('Uncaught Exception:', err);
      this.shutdown('UNCAUGHT_EXCEPTION');
    });
    
    process.on('unhandledRejection', (reason, promise) => {
      console.error('Unhandled Rejection at:', promise, 'reason:', reason);
      this.shutdown('UNHANDLED_REJECTION');
    });
    
    return this.server;
  }

  async shutdown(signal) {
    if (this.isShuttingDown) {
      console.log('Shutdown already in progress...');
      return;
    }
    
    console.log(`\nReceived ${signal}. Starting graceful shutdown...`);
    this.isShuttingDown = true;
    
    const shutdownTimeout = setTimeout(() => {
      console.error('Force shutdown after timeout');
      process.exit(1);
    }, this.options.timeout);
    
    try {
      // Step 1: Stop accepting new connections
      await this.stopServer();
      
      // Step 2: Wait for active requests to finish
      await this.waitForConnections();
      
      // Step 3: Close database connections
      await this.closeConnections();
      
      // Step 4: Flush logs
      await this.flushLogs();
      
      clearTimeout(shutdownTimeout);
      console.log('Graceful shutdown complete');
      process.exit(0);
      
    } catch (err) {
      console.error('Error during shutdown:', err);
      clearTimeout(shutdownTimeout);
      process.exit(1);
    }
  }

  stopServer() {
    return new Promise((resolve, reject) => {
      this.server.close((err) => {
        if (err) {
          reject(err);
        } else {
          console.log('Server stopped accepting connections');
          resolve();
        }
      });
    });
  }

  waitForConnections() {
    return new Promise((resolve) => {
      if (this.activeConnections.size === 0) {
        return resolve();
      }
      
      console.log(`Waiting for ${this.activeConnections.size} connections to close...`);
      
      const interval = setInterval(() => {
        if (this.activeConnections.size === 0) {
          clearInterval(interval);
          resolve();
        } else {
          console.log(`Still waiting for ${this.activeConnections.size} connections...`);
        }
      }, 1000);
      
      // Force close after timeout
      setTimeout(() => {
        if (this.activeConnections.size > 0) {
          console.warn(`Force closing ${this.activeConnections.size} connections`);
          this.activeConnections.forEach(socket => socket.destroy());
          clearInterval(interval);
          resolve();
        }
      }, this.options.forceTimeout);
    });
  }

  async closeConnections() {
    const cleanupTasks = [];
    
    if (global.pgPool) {
      cleanupTasks.push(global.pgPool.end().then(() => console.log('PostgreSQL pool closed')));
    }
    
    if (global.redisClient) {
      cleanupTasks.push(global.redisClient.quit().then(() => console.log('Redis connection closed')));
    }
    
    if (global.mongoConnection) {
      cleanupTasks.push(global.mongoConnection.close().then(() => console.log('MongoDB connection closed')));
    }
    
    await Promise.allSettled(cleanupTasks);
  }

  async flushLogs() {
    // Flush any pending logs
    await new Promise(resolve => setTimeout(resolve, 100));
    console.log('Logs flushed');
  }
}

// ใช้งาน
const app = express();

app.get('/', (req, res) => {
  // Simulate long-running request
  setTimeout(() => {
    res.json({ message: 'Hello World' });
  }, 2000);
});

const gracefulShutdown = new GracefulShutdown(app, {
  timeout: 30000,
  forceTimeout: 10000
});

const server = gracefulShutdown.setup();

server.listen(3000, () => {
  console.log('Server started on port 3000');
});

module.exports = { GracefulShutdown };
```

---

## ขั้นตอนที่ 1444: Zero-Downtime Deployment

```javascript
// zero-downtime-deploy.js
// Strategies สำหรับ deploy โดยไม่มี downtime

// Strategy 1: Rolling Update Script
const { exec } = require('child_process');
const util = require('promisify');
const execAsync = util(exec);

class RollingDeployment {
  constructor(config) {
    this.instances = config.instances; // List of server instances
    this.newVersion = config.newVersion;
    this.healthCheckUrl = config.healthCheckUrl;
    this.batchSize = config.batchSize || 1;
    this.healthCheckRetries = config.healthCheckRetries || 5;
    this.healthCheckInterval = config.healthCheckInterval || 10000;
  }

  async deploy() {
    console.log(`Starting rolling deployment to version ${this.newVersion}`);
    
    // Process instances in batches
    for (let i = 0; i < this.instances.length; i += this.batchSize) {
      const batch = this.instances.slice(i, i + this.batchSize);
      console.log(`\nDeploying batch ${Math.floor(i/this.batchSize) + 1}: ${batch.join(', ')}`);
      
      // Deploy batch
      await this.deployBatch(batch);
      
      // Verify health
      const healthy = await this.verifyBatchHealth(batch);
      
      if (!healthy) {
        console.error('Batch health check failed! Rolling back...');
        await this.rollback(this.instances.slice(0, i + this.batchSize));
        throw new Error('Deployment failed - rolled back');
      }
      
      console.log(`Batch ${Math.floor(i/this.batchSize) + 1} deployed successfully`);
    }
    
    console.log('\nRolling deployment complete!');
  }

  async deployBatch(instances) {
    const deployPromises = instances.map(instance => 
      this.deployInstance(instance)
    );
    await Promise.all(deployPromises);
  }

  async deployInstance(instance) {
    console.log(`  Deploying ${instance}...`);
    
    // Remove from load balancer
    await this.removeFromLoadBalancer(instance);
    
    // Wait for connections to drain
    await new Promise(resolve => setTimeout(resolve, 5000));
    
    // Deploy new version
    await execAsync(`ssh ${instance} "cd /app && git pull && npm install --production && pm2 reload all"`);
    
    console.log(`  ${instance} deployed`);
  }

  async verifyBatchHealth(instances) {
    for (const instance of instances) {
      const healthy = await this.waitForHealth(instance);
      if (!healthy) return false;
      
      // Re-add to load balancer
      await this.addToLoadBalancer(instance);
    }
    return true;
  }

  async waitForHealth(instance) {
    for (let attempt = 1; attempt <= this.healthCheckRetries; attempt++) {
      try {
        const response = await fetch(`http://${instance}${this.healthCheckUrl}`, {
          signal: AbortSignal.timeout(5000)
        });
        
        if (response.ok) {
          console.log(`  ${instance} is healthy`);
          return true;
        }
      } catch (err) {
        console.log(`  ${instance} health check attempt ${attempt}/${this.healthCheckRetries} failed`);
      }
      
      if (attempt < this.healthCheckRetries) {
        await new Promise(resolve => setTimeout(resolve, this.healthCheckInterval));
      }
    }
    
    return false;
  }

  async removeFromLoadBalancer(instance) {
    console.log(`  Removing ${instance} from load balancer`);
    // Implementation depends on your load balancer (nginx, HAProxy, etc.)
  }

  async addToLoadBalancer(instance) {
    console.log(`  Adding ${instance} to load balancer`);
    // Implementation depends on your load balancer
  }

  async rollback(deployedInstances) {
    console.log('Rolling back deployed instances...');
    for (const instance of deployedInstances) {
      await execAsync(`ssh ${instance} "cd /app && git checkout HEAD~1 && npm install --production && pm2 reload all"`);
      await this.addToLoadBalancer(instance);
    }
  }
}

// Strategy 2: Blue-Green Deployment
class BlueGreenDeployment {
  constructor(config) {
    this.loadBalancer = config.loadBalancer;
    this.blueEnvironment = config.blueEnvironment;
    this.greenEnvironment = config.greenEnvironment;
    this.currentActive = config.currentActive || 'blue';
  }

  get activeEnv() {
    return this.currentActive === 'blue' ? this.blueEnvironment : this.greenEnvironment;
  }

  get inactiveEnv() {
    return this.currentActive === 'blue' ? this.greenEnvironment : this.blueEnvironment;
  }

  async deploy(newVersion) {
    console.log(`Deploying ${newVersion} to ${this.inactiveEnv.name} environment`);
    
    // Deploy to inactive environment
    await this.deployToEnvironment(this.inactiveEnv, newVersion);
    
    // Run smoke tests
    const testsPassed = await this.runSmokeTests(this.inactiveEnv);
    
    if (!testsPassed) {
      throw new Error('Smoke tests failed - aborting deployment');
    }
    
    console.log('Smoke tests passed. Switching traffic...');
    
    // Switch traffic
    await this.switchTraffic();
    
    console.log(`Traffic switched to ${this.currentActive} environment`);
    
    // Keep old environment for quick rollback
    console.log(`Old environment (${this.inactiveEnv.name}) kept as rollback target`);
  }

  async deployToEnvironment(env, version) {
    // Deploy new version to inactive environment
    console.log(`Deploying version ${version} to ${env.name}...`);
    // Implementation: deploy to specific environment
  }

  async runSmokeTests(env) {
    const tests = [
      () => fetch(`${env.url}/health`).then(r => r.ok),
      () => fetch(`${env.url}/api/products`).then(r => r.ok),
      // เพิ่ม tests ตามต้องการ
    ];
    
    const results = await Promise.all(tests.map(test => test().catch(() => false)));
    return results.every(r => r === true);
  }

  async switchTraffic() {
    // Switch load balancer to point to inactive environment
    const newActive = this.currentActive === 'blue' ? 'green' : 'blue';
    
    // Update load balancer config
    await this.loadBalancer.updateTarget(this.inactiveEnv.url);
    
    // Update current active
    this.currentActive = newActive;
  }

  async rollback() {
    console.log('Rolling back to previous version...');
    await this.loadBalancer.updateTarget(this.inactiveEnv.url);
    this.currentActive = this.currentActive === 'blue' ? 'green' : 'blue';
    console.log(`Rolled back to ${this.currentActive} environment`);
  }
}

// Strategy 3: Canary Deployment
class CanaryDeployment {
  constructor(config) {
    this.loadBalancer = config.loadBalancer;
    this.stableInstances = config.stableInstances;
    this.canaryInstances = config.canaryInstances;
    this.canaryPercent = config.canaryPercent || 5;
    this.successMetricThreshold = config.successMetricThreshold || 99; // 99% success rate
  }

  async deploy(newVersion) {
    // Phase 1: Deploy to canary instances (5% of traffic)
    await this.deployCanary(newVersion);
    await this.setTrafficSplit(this.canaryPercent);
    
    console.log(`Canary deployed with ${this.canaryPercent}% traffic`);
    
    // Phase 2: Monitor metrics
    const metricsOk = await this.monitorMetrics(300000); // 5 minutes
    
    if (!metricsOk) {
      console.error('Canary metrics failed! Rolling back...');
      await this.rollbackCanary();
      throw new Error('Canary deployment failed');
    }
    
    // Phase 3: Gradually increase traffic
    for (const percent of [25, 50, 75, 100]) {
      await this.setTrafficSplit(percent);
      console.log(`Traffic at ${percent}%`);
      
      const stillOk = await this.monitorMetrics(120000); // 2 minutes each
      if (!stillOk) {
        await this.rollbackCanary();
        throw new Error(`Deployment failed at ${percent}% traffic`);
      }
    }
    
    console.log('Canary deployment successful - promoting to stable');
    await this.promoteCanary();
  }

  async deployCanary(version) {
    for (const instance of this.canaryInstances) {
      await this.deployToInstance(instance, version);
    }
  }

  async setTrafficSplit(canaryPercent) {
    // Update load balancer traffic weights
    const stablePercent = 100 - canaryPercent;
    await this.loadBalancer.setWeights({
      stable: stablePercent,
      canary: canaryPercent
    });
  }

  async monitorMetrics(durationMs) {
    const endTime = Date.now() + durationMs;
    const samples = [];
    
    while (Date.now() < endTime) {
      const metrics = await this.collectMetrics();
      samples.push(metrics);
      
      // Check if metrics are bad
      if (metrics.errorRate > (100 - this.successMetricThreshold)) {
        console.error(`Error rate ${metrics.errorRate}% exceeds threshold`);
        return false;
      }
      
      await new Promise(resolve => setTimeout(resolve, 10000)); // Check every 10s
    }
    
    // Analyze collected samples
    const avgErrorRate = samples.reduce((sum, s) => sum + s.errorRate, 0) / samples.length;
    console.log(`Average error rate: ${avgErrorRate.toFixed(2)}%`);
    
    return avgErrorRate <= (100 - this.successMetricThreshold);
  }

  async collectMetrics() {
    // Collect from Prometheus or your monitoring system
    return {
      errorRate: Math.random() * 2, // Simulate: usually < 1%
      latencyP99: Math.random() * 100 + 50
    };
  }

  async deployToInstance(instance, version) {
    console.log(`Deploying ${version} to ${instance}`);
  }

  async rollbackCanary() {
    await this.setTrafficSplit(0);
    console.log('Rolled back canary deployment');
  }

  async promoteCanary() {
    // Deploy canary version to stable instances
    console.log('Promoting canary to stable...');
  }
}

module.exports = { RollingDeployment, BlueGreenDeployment, CanaryDeployment };
```

---

## ขั้นตอนที่ 1445: Failover และ Redundancy

```javascript
// failover.js
// Automatic failover implementation

class HADatabasePool {
  constructor(primary, replicas) {
    this.primary = primary;
    this.replicas = replicas;
    this.currentPrimary = primary;
    this.failedNodes = new Set();
    this.isFailingOver = false;
    
    this.startHeartbeat();
  }

  startHeartbeat() {
    setInterval(async () => {
      await this.checkPrimary();
    }, 5000);
  }

  async checkPrimary() {
    try {
      await this.currentPrimary.query('SELECT 1', [], { timeout: 2000 });
    } catch (err) {
      console.error('Primary health check failed:', err.message);
      
      if (!this.isFailingOver) {
        await this.initiateFailover();
      }
    }
  }

  async initiateFailover() {
    this.isFailingOver = true;
    console.log('Initiating failover...');
    
    const healthyReplica = await this.findHealthyReplica();
    
    if (!healthyReplica) {
      throw new Error('No healthy replicas available for failover');
    }
    
    // Promote replica to primary
    await this.promoteReplica(healthyReplica);
    
    this.failedNodes.add(this.currentPrimary);
    this.currentPrimary = healthyReplica;
    
    console.log(`Failover complete: New primary is ${healthyReplica.host}`);
    
    // Notify monitoring
    this.emit('failover', {
      oldPrimary: this.primary,
      newPrimary: healthyReplica,
      timestamp: new Date().toISOString()
    });
    
    this.isFailingOver = false;
    
    // Try to recover original primary
    this.scheduleRecovery();
  }

  async findHealthyReplica() {
    for (const replica of this.replicas) {
      if (this.failedNodes.has(replica)) continue;
      
      try {
        await replica.query('SELECT 1', [], { timeout: 2000 });
        return replica;
      } catch (err) {
        this.failedNodes.add(replica);
      }
    }
    return null;
  }

  async promoteReplica(replica) {
    // PostgreSQL: promote replica to primary
    await replica.query('SELECT pg_promote()');
    
    // Wait for promotion
    let attempts = 0;
    while (attempts < 30) {
      try {
        const result = await replica.query('SELECT pg_is_in_recovery()');
        if (!result.rows[0].pg_is_in_recovery) {
          console.log('Replica promoted to primary');
          return;
        }
      } catch (err) {}
      
      await new Promise(resolve => setTimeout(resolve, 1000));
      attempts++;
    }
    
    throw new Error('Replica promotion timeout');
  }

  scheduleRecovery() {
    setTimeout(async () => {
      try {
        await this.primary.query('SELECT 1', [], { timeout: 5000 });
        console.log('Original primary recovered');
        this.failedNodes.delete(this.primary);
        
        // Re-add as replica (don't automatically make it primary again)
        this.replicas.unshift(this.primary);
      } catch (err) {
        console.log('Original primary still unavailable, retrying...');
        this.scheduleRecovery();
      }
    }, 60000); // Check after 1 minute
  }

  async write(query, params) {
    return this.currentPrimary.query(query, params);
  }

  async read(query, params) {
    // Load balance across replicas
    const availableReplicas = this.replicas.filter(r => !this.failedNodes.has(r));
    
    if (availableReplicas.length > 0) {
      const replica = availableReplicas[Math.floor(Math.random() * availableReplicas.length)];
      try {
        return await replica.query(query, params);
      } catch (err) {
        // Fallback to primary
        console.warn('Replica read failed, falling back to primary');
      }
    }
    
    return this.currentPrimary.query(query, params);
  }

  emit(event, data) {
    // Send to monitoring system
    console.log(`EVENT: ${event}`, data);
  }
}

// Multi-Region Failover
class MultiRegionConfig {
  constructor() {
    this.regions = [
      {
        name: 'us-east-1',
        priority: 1,
        endpoints: {
          api: 'https://api.us-east-1.example.com',
          db: 'postgres://us-east-1-db.example.com:5432/mydb'
        }
      },
      {
        name: 'eu-west-1',
        priority: 2,
        endpoints: {
          api: 'https://api.eu-west-1.example.com',
          db: 'postgres://eu-west-1-db.example.com:5432/mydb'
        }
      },
      {
        name: 'ap-southeast-1',
        priority: 3,
        endpoints: {
          api: 'https://api.ap-southeast-1.example.com',
          db: 'postgres://ap-southeast-1-db.example.com:5432/mydb'
        }
      }
    ];
    
    this.activeRegion = this.regions[0];
  }

  async getActiveRegion() {
    // Try regions in priority order
    for (const region of this.regions.sort((a, b) => a.priority - b.priority)) {
      const healthy = await this.checkRegionHealth(region);
      if (healthy) return region;
    }
    
    throw new Error('All regions unhealthy');
  }

  async checkRegionHealth(region) {
    try {
      const response = await fetch(`${region.endpoints.api}/health`, {
        signal: AbortSignal.timeout(5000)
      });
      return response.ok;
    } catch {
      return false;
    }
  }
}

module.exports = { HADatabasePool, MultiRegionConfig };
```

---

## ขั้นตอนที่ 1446: Kubernetes Health Probes Configuration

```yaml
# k8s-deployment.yaml
# Kubernetes deployment สำหรับ HA Node.js application

apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: node-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # เพิ่มได้ 1 pod ระหว่าง update
      maxUnavailable: 0    # ไม่ลด pod ระหว่าง update (zero-downtime)
  template:
    metadata:
      labels:
        app: node-api
    spec:
      containers:
      - name: node-api
        image: myapp:latest
        ports:
        - containerPort: 3000
        
        # Resource limits
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        
        # Liveness probe - restart container if fails
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        
        # Readiness probe - remove from service if fails
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        
        # Startup probe - give time for slow startup
        startupProbe:
          httpGet:
            path: /health/startup
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 30  # 30 * 5s = 150s max startup time
        
        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
        
        terminationGracePeriodSeconds: 30
        
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
```

```javascript
// kubernetes-ha.js
// Node.js configuration สำหรับ Kubernetes HA setup

const express = require('express');
const app = express();

// State tracking สำหรับ probes
const appState = {
  initialized: false,
  ready: false,
  shutting_down: false
};

// Initialization
async function initialize() {
  try {
    // Wait for database
    await waitForDatabase();
    
    // Load configuration
    await loadConfiguration();
    
    // Warm up caches
    await warmUpCaches();
    
    appState.initialized = true;
    appState.ready = true;
    
    console.log('Application initialized and ready');
  } catch (err) {
    console.error('Initialization failed:', err);
    process.exit(1);
  }
}

async function waitForDatabase() {
  const maxRetries = 30;
  for (let i = 0; i < maxRetries; i++) {
    try {
      await global.db.query('SELECT 1');
      console.log('Database connected');
      return;
    } catch (err) {
      console.log(`Waiting for database... attempt ${i + 1}/${maxRetries}`);
      await new Promise(resolve => setTimeout(resolve, 2000));
    }
  }
  throw new Error('Database connection timeout');
}

async function loadConfiguration() {
  // Load config from external source
  await new Promise(resolve => setTimeout(resolve, 500));
  console.log('Configuration loaded');
}

async function warmUpCaches() {
  // Pre-populate caches
  await new Promise(resolve => setTimeout(resolve, 1000));
  console.log('Caches warmed up');
}

// Kubernetes probes
app.get('/health/live', (req, res) => {
  if (appState.shutting_down) {
    return res.status(503).json({ status: 'shutting_down' });
  }
  res.json({ status: 'alive', pid: process.pid });
});

app.get('/health/ready', (req, res) => {
  if (!appState.ready || appState.shutting_down) {
    return res.status(503).json({ status: 'not_ready' });
  }
  res.json({ status: 'ready' });
});

app.get('/health/startup', (req, res) => {
  if (!appState.initialized) {
    return res.status(503).json({ status: 'initializing' });
  }
  res.json({ status: 'started' });
});

// Graceful shutdown handler
const shutdown = async (signal) => {
  console.log(`Received ${signal}`);
  appState.ready = false;       // Stop serving traffic
  appState.shutting_down = true;
  
  // Wait for Kubernetes to notice we're not ready (5-10s)
  await new Promise(resolve => setTimeout(resolve, 10000));
  
  // Finish serving current requests
  server.close(async () => {
    // Cleanup resources
    if (global.db) await global.db.end();
    if (global.redis) await global.redis.quit();
    
    console.log('Graceful shutdown complete');
    process.exit(0);
  });
};

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));

const server = app.listen(3000, async () => {
  console.log('Server listening on port 3000');
  await initialize();
});

module.exports = app;
```

---

## ขั้นตอนที่ 1447-1460: Advanced HA Patterns

### Bulkhead Pattern

```javascript
// bulkhead.js
// แบ่ง resource pools เพื่อป้องกัน cascading failures

class BulkheadPool {
  constructor(options) {
    this.maxConcurrent = options.maxConcurrent || 10;
    this.maxQueued = options.maxQueued || 100;
    this.timeout = options.timeout || 30000;
    
    this.running = 0;
    this.queue = [];
  }

  async execute(fn) {
    return new Promise((resolve, reject) => {
      const task = { fn, resolve, reject, startTime: Date.now() };
      
      if (this.running < this.maxConcurrent) {
        this.run(task);
      } else if (this.queue.length < this.maxQueued) {
        this.queue.push(task);
      } else {
        reject(new Error('Bulkhead queue full - request rejected'));
      }
    });
  }

  async run(task) {
    this.running++;
    
    const timeout = setTimeout(() => {
      task.reject(new Error('Bulkhead task timeout'));
      this.running--;
      this.processQueue();
    }, this.timeout);

    try {
      const result = await task.fn();
      clearTimeout(timeout);
      task.resolve(result);
    } catch (err) {
      clearTimeout(timeout);
      task.reject(err);
    } finally {
      this.running--;
      this.processQueue();
    }
  }

  processQueue() {
    if (this.queue.length > 0 && this.running < this.maxConcurrent) {
      const task = this.queue.shift();
      
      // Check if task has expired
      if (Date.now() - task.startTime > this.timeout) {
        task.reject(new Error('Task expired in queue'));
        this.processQueue();
      } else {
        this.run(task);
      }
    }
  }

  getStats() {
    return {
      running: this.running,
      queued: this.queue.length,
      maxConcurrent: this.maxConcurrent,
      maxQueued: this.maxQueued
    };
  }
}

// สร้าง separate bulkheads สำหรับ different services
const bulkheads = {
  payment: new BulkheadPool({ maxConcurrent: 5, maxQueued: 20 }),
  inventory: new BulkheadPool({ maxConcurrent: 10, maxQueued: 50 }),
  notification: new BulkheadPool({ maxConcurrent: 20, maxQueued: 200 })
};

// Express middleware
const express = require('express');
const app = express();

app.post('/checkout', async (req, res) => {
  try {
    // ทำงานใน bulkhead สำหรับ payment
    const paymentResult = await bulkheads.payment.execute(async () => {
      return processPayment(req.body);
    });
    
    // ทำงานใน bulkhead สำหรับ inventory
    const inventoryResult = await bulkheads.inventory.execute(async () => {
      return updateInventory(req.body.items);
    });
    
    // notification เป็น non-critical ส่งแบบ fire-and-forget
    bulkheads.notification.execute(async () => {
      return sendOrderConfirmation(req.body);
    }).catch(err => console.warn('Notification failed:', err.message));
    
    res.json({ success: true, payment: paymentResult, inventory: inventoryResult });
    
  } catch (err) {
    if (err.message.includes('Bulkhead')) {
      res.status(503).json({ error: 'Service temporarily overloaded' });
    } else {
      res.status(500).json({ error: err.message });
    }
  }
});

app.get('/bulkhead-stats', (req, res) => {
  res.json({
    payment: bulkheads.payment.getStats(),
    inventory: bulkheads.inventory.getStats(),
    notification: bulkheads.notification.getStats()
  });
});

async function processPayment(data) {
  await new Promise(resolve => setTimeout(resolve, 200));
  return { transactionId: 'txn_' + Date.now() };
}

async function updateInventory(items) {
  await new Promise(resolve => setTimeout(resolve, 100));
  return { updated: true };
}

async function sendOrderConfirmation(data) {
  await new Promise(resolve => setTimeout(resolve, 500));
  return { sent: true };
}

module.exports = { BulkheadPool, bulkheads };
```

---

## แบบฝึกหัด

### Exercise 1: Implement Health Check Dashboard
สร้าง dashboard แสดง health status ของ services ทั้งหมดแบบ real-time โดยใช้ Server-Sent Events

### Exercise 2: Blue-Green Deployment Script
เขียน script ที่ทำ blue-green deployment โดยอัตโนมัติ พร้อม automated smoke tests

### Exercise 3: Multi-Region Setup
ออกแบบ architecture สำหรับ multi-region deployment ที่รองรับ automatic failover ระหว่าง regions

### คำถามทบทวน
1. ความแตกต่างระหว่าง liveness probe, readiness probe, และ startup probe คืออะไร?
2. Blue-Green กับ Canary deployment ต่างกันอย่างไร และควรใช้เมื่อใด?
3. Bulkhead pattern ช่วยป้องกัน cascading failures ได้อย่างไร?
4. Zero-downtime deployment ต้องการ prerequisites อะไรบ้าง?

---

*ต่อไป: Part 83 - Database Sharding*
