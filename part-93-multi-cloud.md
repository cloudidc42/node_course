# Part 93 | ขั้นตอนที่ 1661-1680 จาก 1000+

## Multi-Cloud Strategies และ Vendor Lock-in Avoidance

ในส่วนนี้เราจะเรียนรู้วิธีออกแบบ Node.js applications ให้ทำงานบนหลาย cloud providers และหลีกเลี่ยง vendor lock-in

---

## ขั้นตอนที่ 1661: Multi-Cloud Architecture Fundamentals

```javascript
// cloud-abstraction-layer.js
// Abstract cloud provider differences behind a common interface

const AWS = require('aws-sdk');
const { Storage } = require('@google-cloud/storage');
const { BlobServiceClient } = require('@azure/storage-blob');

// Cloud Storage Abstraction
class CloudStorageAdapter {
  static create(provider, config) {
    switch (provider) {
      case 'aws': return new AWSStorageAdapter(config);
      case 'gcp': return new GCPStorageAdapter(config);
      case 'azure': return new AzureStorageAdapter(config);
      default: throw new Error(`Unknown provider: ${provider}`);
    }
  }

  async upload(key, data, options = {}) { throw new Error('Not implemented'); }
  async download(key) { throw new Error('Not implemented'); }
  async delete(key) { throw new Error('Not implemented'); }
  async exists(key) { throw new Error('Not implemented'); }
  async listFiles(prefix) { throw new Error('Not implemented'); }
}

class AWSStorageAdapter extends CloudStorageAdapter {
  constructor(config) {
    super();
    this.s3 = new AWS.S3({
      region: config.region,
      accessKeyId: config.accessKey,
      secretAccessKey: config.secretKey
    });
    this.bucket = config.bucket;
  }

  async upload(key, data, options = {}) {
    await this.s3.putObject({
      Bucket: this.bucket,
      Key: key,
      Body: data,
      ContentType: options.contentType || 'application/octet-stream',
      ...options.metadata && { Metadata: options.metadata }
    }).promise();
    
    return { provider: 'aws', bucket: this.bucket, key };
  }

  async download(key) {
    const result = await this.s3.getObject({
      Bucket: this.bucket,
      Key: key
    }).promise();
    return result.Body;
  }

  async delete(key) {
    await this.s3.deleteObject({ Bucket: this.bucket, Key: key }).promise();
  }

  async exists(key) {
    try {
      await this.s3.headObject({ Bucket: this.bucket, Key: key }).promise();
      return true;
    } catch { return false; }
  }

  async listFiles(prefix) {
    const result = await this.s3.listObjectsV2({
      Bucket: this.bucket,
      Prefix: prefix
    }).promise();
    return result.Contents.map(obj => ({ key: obj.Key, size: obj.Size, lastModified: obj.LastModified }));
  }
}

class GCPStorageAdapter extends CloudStorageAdapter {
  constructor(config) {
    super();
    this.storage = new Storage({ projectId: config.projectId, keyFilename: config.keyFile });
    this.bucket = this.storage.bucket(config.bucketName);
  }

  async upload(key, data, options = {}) {
    const file = this.bucket.file(key);
    await file.save(data, {
      contentType: options.contentType || 'application/octet-stream',
      metadata: options.metadata
    });
    return { provider: 'gcp', bucket: this.bucket.name, key };
  }

  async download(key) {
    const [data] = await this.bucket.file(key).download();
    return data;
  }

  async delete(key) {
    await this.bucket.file(key).delete();
  }

  async exists(key) {
    const [exists] = await this.bucket.file(key).exists();
    return exists;
  }

  async listFiles(prefix) {
    const [files] = await this.bucket.getFiles({ prefix });
    return files.map(f => ({ key: f.name, size: f.metadata.size, lastModified: f.metadata.updated }));
  }
}

class AzureStorageAdapter extends CloudStorageAdapter {
  constructor(config) {
    super();
    this.client = BlobServiceClient.fromConnectionString(config.connectionString);
    this.containerClient = this.client.getContainerClient(config.containerName);
  }

  async upload(key, data, options = {}) {
    const blockBlobClient = this.containerClient.getBlockBlobClient(key);
    await blockBlobClient.upload(data, data.length, {
      blobHTTPHeaders: { blobContentType: options.contentType }
    });
    return { provider: 'azure', container: this.containerClient.containerName, key };
  }

  async download(key) {
    const blockBlobClient = this.containerClient.getBlockBlobClient(key);
    const downloadResponse = await blockBlobClient.download(0);
    return this.streamToBuffer(downloadResponse.readableStreamBody);
  }

  streamToBuffer(stream) {
    return new Promise((resolve, reject) => {
      const chunks = [];
      stream.on('data', chunk => chunks.push(chunk));
      stream.on('end', () => resolve(Buffer.concat(chunks)));
      stream.on('error', reject);
    });
  }

  async delete(key) {
    await this.containerClient.getBlockBlobClient(key).delete();
  }

  async exists(key) {
    return this.containerClient.getBlockBlobClient(key).exists();
  }

  async listFiles(prefix) {
    const files = [];
    for await (const blob of this.containerClient.listBlobsFlat({ prefix })) {
      files.push({ key: blob.name, size: blob.properties.contentLength, lastModified: blob.properties.lastModified });
    }
    return files;
  }
}

module.exports = { CloudStorageAdapter, AWSStorageAdapter, GCPStorageAdapter, AzureStorageAdapter };
```

---

## ขั้นตอนที่ 1662: Cloud-Agnostic Queue Abstraction

```javascript
// cloud-queue.js
// Abstract message queue behind common interface

const { SQSClient, SendMessageCommand, ReceiveMessageCommand, DeleteMessageCommand } = require('@aws-sdk/client-sqs');
const { PubSub } = require('@google-cloud/pubsub');
const { ServiceBusClient } = require('@azure/service-bus');

class CloudQueueAdapter {
  static create(provider, config) {
    switch (provider) {
      case 'aws': return new SQSAdapter(config);
      case 'gcp': return new PubSubAdapter(config);
      case 'azure': return new ServiceBusAdapter(config);
      default: throw new Error(`Unknown provider: ${provider}`);
    }
  }

  async publish(message, options = {}) { throw new Error('Not implemented'); }
  async subscribe(handler, options = {}) { throw new Error('Not implemented'); }
  async acknowledge(messageId) { throw new Error('Not implemented'); }
}

class SQSAdapter extends CloudQueueAdapter {
  constructor(config) {
    super();
    this.client = new SQSClient({ region: config.region });
    this.queueUrl = config.queueUrl;
    this.isRunning = false;
  }

  async publish(message, options = {}) {
    await this.client.send(new SendMessageCommand({
      QueueUrl: this.queueUrl,
      MessageBody: JSON.stringify(message),
      DelaySeconds: options.delay || 0,
      MessageAttributes: options.attributes ? this.formatAttributes(options.attributes) : undefined
    }));
  }

  async subscribe(handler, options = {}) {
    this.isRunning = true;
    
    while (this.isRunning) {
      const response = await this.client.send(new ReceiveMessageCommand({
        QueueUrl: this.queueUrl,
        MaxNumberOfMessages: options.batchSize || 10,
        WaitTimeSeconds: 20, // Long polling
        VisibilityTimeout: options.visibilityTimeout || 30
      }));
      
      if (response.Messages) {
        await Promise.allSettled(
          response.Messages.map(async msg => {
            try {
              await handler(JSON.parse(msg.Body), msg);
              await this.acknowledge(msg.ReceiptHandle);
            } catch (err) {
              console.error('Message processing failed:', err.message);
            }
          })
        );
      }
    }
  }

  async acknowledge(receiptHandle) {
    await this.client.send(new DeleteMessageCommand({
      QueueUrl: this.queueUrl,
      ReceiptHandle: receiptHandle
    }));
  }

  stop() { this.isRunning = false; }

  formatAttributes(attrs) {
    return Object.fromEntries(
      Object.entries(attrs).map(([k, v]) => [k, { DataType: 'String', StringValue: String(v) }])
    );
  }
}

// Cloud-agnostic worker using the abstraction
class CloudWorker {
  constructor(queue) {
    this.queue = queue;
    this.handlers = new Map();
  }

  on(messageType, handler) {
    this.handlers.set(messageType, handler);
    return this;
  }

  async start() {
    await this.queue.subscribe(async (message, raw) => {
      const { type, payload } = message;
      const handler = this.handlers.get(type);
      
      if (!handler) {
        console.warn(`No handler for message type: ${type}`);
        return;
      }
      
      await handler(payload);
    });
  }
}

// Usage (works with any cloud provider)
const queue = CloudQueueAdapter.create(process.env.CLOUD_PROVIDER || 'aws', {
  region: process.env.AWS_REGION,
  queueUrl: process.env.QUEUE_URL
});

const worker = new CloudWorker(queue);
worker
  .on('user.created', async (payload) => {
    console.log(`Processing user created: ${payload.userId}`);
  })
  .on('order.placed', async (payload) => {
    console.log(`Processing order: ${payload.orderId}`);
  });

module.exports = { CloudQueueAdapter, CloudWorker };
```

---

## ขั้นตอนที่ 1663: Multi-Cloud Database Strategy

```javascript
// multi-cloud-db.js
// Cloud-agnostic database configuration

const { Pool } = require('pg');
const mysql = require('mysql2/promise');
const { MongoClient } = require('mongodb');

class DatabaseFactory {
  static create(config) {
    switch (config.type) {
      case 'postgres': return new PostgresAdapter(config);
      case 'mysql': return new MySQLAdapter(config);
      case 'mongodb': return new MongoDBAdapter(config);
      default: throw new Error(`Unknown database type: ${config.type}`);
    }
  }
}

class PostgresAdapter {
  constructor(config) {
    this.pool = new Pool({
      host: config.host,
      port: config.port || 5432,
      database: config.database,
      user: config.user,
      password: config.password,
      ssl: config.ssl ? { rejectUnauthorized: false } : undefined,
      max: config.poolSize || 10
    });
  }

  async query(sql, params = []) {
    const client = await this.pool.connect();
    try {
      const result = await client.query(sql, params);
      return result.rows;
    } finally {
      client.release();
    }
  }

  async transaction(callback) {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');
      const result = await callback({
        query: (sql, params) => client.query(sql, params)
      });
      await client.query('COMMIT');
      return result;
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }

  async close() { await this.pool.end(); }
}

// Cloud database config by provider
function getDatabaseConfig() {
  const provider = process.env.CLOUD_PROVIDER;
  
  const configs = {
    aws: {
      type: 'postgres',
      host: process.env.RDS_HOST,
      port: 5432,
      database: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      ssl: true
    },
    gcp: {
      type: 'postgres',
      host: `/cloudsql/${process.env.CLOUD_SQL_CONNECTION_NAME}`,
      database: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD
    },
    azure: {
      type: 'postgres',
      host: process.env.POSTGRESQL_HOST,
      port: 5432,
      database: process.env.DB_NAME,
      user: `${process.env.DB_USER}@${process.env.POSTGRESQL_HOST}`,
      password: process.env.DB_PASSWORD,
      ssl: true
    }
  };
  
  return configs[provider] || configs.aws;
}

module.exports = { DatabaseFactory, getDatabaseConfig };
```

---

## ขั้นตอนที่ 1664-1680: Multi-Cloud Resilience Patterns

```javascript
// multi-cloud-failover.js
// Automatic failover between cloud providers

class MultiCloudRouter {
  constructor(providers) {
    this.providers = providers;
    this.healthStatus = new Map(providers.map(p => [p.name, true]));
    this.startHealthChecks();
  }

  startHealthChecks() {
    setInterval(async () => {
      for (const provider of this.providers) {
        try {
          await provider.healthCheck();
          this.healthStatus.set(provider.name, true);
        } catch {
          this.healthStatus.set(provider.name, false);
          console.warn(`Provider ${provider.name} is unhealthy`);
        }
      }
    }, 30000);
  }

  getHealthyProviders() {
    return this.providers.filter(p => this.healthStatus.get(p.name) === true);
  }

  getPrimaryProvider() {
    const healthy = this.getHealthyProviders();
    if (healthy.length === 0) throw new Error('All providers are unavailable');
    return healthy[0]; // First healthy provider
  }

  // Execute with automatic failover
  async execute(operation) {
    const providers = this.getHealthyProviders();
    
    for (const provider of providers) {
      try {
        return await operation(provider);
      } catch (err) {
        console.warn(`Provider ${provider.name} failed: ${err.message}, trying next...`);
        this.healthStatus.set(provider.name, false);
      }
    }
    
    throw new Error('All providers failed');
  }

  // Execute on all providers (for replication)
  async executeAll(operation) {
    const results = await Promise.allSettled(
      this.providers.map(provider => operation(provider))
    );
    
    const failures = results.filter(r => r.status === 'rejected');
    if (failures.length > 0) {
      console.warn(`${failures.length} providers failed:`, failures.map(f => f.reason.message));
    }
    
    const successes = results.filter(r => r.status === 'fulfilled');
    return successes.map(r => r.value);
  }
}

// Cost optimization across clouds
class MultiCloudCostOptimizer {
  constructor(providers) {
    this.providers = providers;
    this.priceCache = new Map();
  }

  async getCheapestProvider(resource) {
    const prices = await Promise.all(
      this.providers.map(async provider => ({
        provider,
        price: await this.getPrice(provider, resource)
      }))
    );
    
    return prices.sort((a, b) => a.price - b.price)[0].provider;
  }

  async getPrice(provider, resource) {
    const cacheKey = `${provider.name}:${resource}`;
    if (this.priceCache.has(cacheKey)) return this.priceCache.get(cacheKey);
    
    const price = await provider.getPrice(resource);
    this.priceCache.set(cacheKey, price);
    
    setTimeout(() => this.priceCache.delete(cacheKey), 3600000); // 1 hour TTL
    return price;
  }
}

// Terraform-based multi-cloud provisioning
const terraformConfig = `
# main.tf - Multi-cloud Node.js deployment

terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
    google = { source = "hashicorp/google", version = "~> 4.0" }
    azurerm = { source = "hashicorp/azurerm", version = "~> 3.0" }
  }
}

# AWS ECS deployment
module "aws_ecs" {
  source       = "./modules/aws-ecs"
  image        = var.docker_image
  cpu          = var.cpu
  memory       = var.memory
  environment  = var.environment_vars
}

# GCP Cloud Run deployment
module "gcp_cloud_run" {
  source      = "./modules/gcp-cloud-run"
  image       = var.docker_image
  region      = var.gcp_region
  environment = var.environment_vars
}

# Azure Container Apps deployment
module "azure_aca" {
  source      = "./modules/azure-container-apps"
  image       = var.docker_image
  location    = var.azure_location
  environment = var.environment_vars
}

# Global load balancer routing across all clouds
module "global_lb" {
  source = "./modules/global-lb"
  endpoints = [
    module.aws_ecs.endpoint,
    module.gcp_cloud_run.endpoint,
    module.azure_aca.endpoint
  ]
  health_check_path = "/health"
}
`;

module.exports = { MultiCloudRouter, MultiCloudCostOptimizer };
```

---

## แบบฝึกหัด

### Exercise 1: Storage Abstraction
เพิ่ม DigitalOcean Spaces adapter เข้าไปใน CloudStorageAdapter (ใช้ S3-compatible API)

### Exercise 2: Multi-Cloud Failover Test
เขียน integration test ที่จำลองการล้มเหลวของ cloud provider และตรวจสอบ failover

### Exercise 3: Cost Monitoring
สร้าง cost monitoring dashboard ที่แสดง resource costs จากแต่ละ cloud provider

### คำถามทบทวน
1. Vendor lock-in เกิดขึ้นอย่างไร และวิธีหลีกเลี่ยง?
2. Trade-offs ของ multi-cloud vs single-cloud strategy คืออะไร?
3. Multi-cloud routing มีกลยุทธ์อะไรบ้าง?

---

*ต่อไป: Part 94 - Edge Computing*
