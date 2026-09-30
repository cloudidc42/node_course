# Part 76 | ขั้นตอนที่ 1321-1340 จาก 1000+

# Cloud Cost Optimization

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. วิเคราะห์ cloud costs
2. implement auto-scaling อย่างมีประสิทธิภาพ
3. ใช้ Spot/Preemptible instances
4. optimize database costs
5. implement caching strategies
6. ตั้งค่า cost monitoring และ alerting

---

## ขั้นตอนที่ 1321: Cloud Cost Visibility

```typescript
// src/cost/cost-monitor.ts
// Monitor and track cloud costs

export interface CostMetrics {
  service: string;
  region: string;
  costUSD: number;
  period: string;
  breakdown: CostBreakdown[];
}

export interface CostBreakdown {
  resource: string;
  costUSD: number;
  usage: number;
  unit: string;
}

// AWS Cost Explorer integration
import { CostExplorer } from "@aws-sdk/client-cost-explorer";

export class AWSCostMonitor {
  private readonly ce = new CostExplorer({ region: "us-east-1" });

  async getMonthlyCosts(): Promise<CostMetrics[]> {
    const endDate = new Date();
    const startDate = new Date();
    startDate.setDate(1);  // First of current month

    const result = await this.ce.getCostAndUsage({
      TimePeriod: {
        Start: startDate.toISOString().split("T")[0],
        End: endDate.toISOString().split("T")[0]
      },
      Granularity: "MONTHLY",
      GroupBy: [
        { Type: "DIMENSION", Key: "SERVICE" },
        { Type: "DIMENSION", Key: "REGION" }
      ],
      Metrics: ["UnblendedCost"]
    });

    const costs: CostMetrics[] = [];
    
    for (const period of result.ResultsByTime ?? []) {
      for (const group of period.Groups ?? []) {
        costs.push({
          service: group.Keys?.[0] ?? "unknown",
          region: group.Keys?.[1] ?? "unknown",
          costUSD: parseFloat(group.Metrics?.UnblendedCost?.Amount ?? "0"),
          period: "monthly",
          breakdown: []
        });
      }
    }

    return costs.sort((a, b) => b.costUSD - a.costUSD);
  }

  async getCostByTag(tagKey: string, tagValue: string): Promise<number> {
    const endDate = new Date();
    const startDate = new Date();
    startDate.setDate(1);

    const result = await this.ce.getCostAndUsage({
      TimePeriod: {
        Start: startDate.toISOString().split("T")[0],
        End: endDate.toISOString().split("T")[0]
      },
      Granularity: "MONTHLY",
      Filter: {
        Tags: {
          Key: tagKey,
          Values: [tagValue]
        }
      },
      Metrics: ["UnblendedCost"]
    });

    const amount = result.ResultsByTime?.[0]?.Total?.UnblendedCost?.Amount ?? "0";
    return parseFloat(amount);
  }
}
```

---

## ขั้นตอนที่ 1322: Auto-scaling Configuration

```yaml
# k8s/autoscaling/hpa.yaml - Horizontal Pod Autoscaler

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  
  minReplicas: 2      # Minimum 2 for availability
  maxReplicas: 20     # Maximum to control costs
  
  metrics:
    # Scale on CPU (primary)
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # Scale up when CPU > 70%
    
    # Scale on memory
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    
    # Scale on custom metric (requests per second)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"  # 1000 req/s per pod
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30  # Fast scale up
      policies:
        - type: Pods
          value: 4        # Add max 4 pods at once
          periodSeconds: 60
    
    scaleDown:
      stabilizationWindowSeconds: 300  # Slow scale down (5 min)
      policies:
        - type: Pods
          value: 1        # Remove 1 pod at a time
          periodSeconds: 60
```

---

## ขั้นตอนที่ 1323: Vertical Pod Autoscaler

```yaml
# k8s/autoscaling/vpa.yaml

apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: product-service-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  
  updatePolicy:
    updateMode: "Auto"  # Automatically update resources
  
  resourcePolicy:
    containerPolicies:
      - containerName: product-service
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: 2
          memory: 2Gi
        controlledResources: ["cpu", "memory"]
```

---

## ขั้นตอนที่ 1324: Spot Instance Strategy

```typescript
// src/infrastructure/spot-instances.ts
// Handle spot instance interruptions

import { Injectable, OnModuleInit, Logger } from "@nestjs/common";
import axios from "axios";

@Injectable()
export class SpotInstanceService implements OnModuleInit {
  private readonly logger = new Logger(SpotInstanceService.name);
  private isBeingTerminated = false;

  onModuleInit() {
    // Check AWS spot termination notice every 5 seconds
    this.pollTerminationNotice();
  }

  private async pollTerminationNotice() {
    setInterval(async () => {
      try {
        // AWS metadata endpoint for spot termination
        const response = await axios.get(
          "http://169.254.169.254/latest/meta-data/spot/termination-time",
          { timeout: 2000 }
        );
        
        if (response.status === 200) {
          this.logger.warn("SPOT TERMINATION NOTICE RECEIVED!");
          await this.handleTermination(response.data);
        }
      } catch {
        // 404 means no termination notice - continue normally
      }
    }, 5000);
  }

  private async handleTermination(terminationTime: string) {
    if (this.isBeingTerminated) return;
    this.isBeingTerminated = true;

    this.logger.warn(`Spot instance will be terminated at: ${terminationTime}`);
    
    // Step 1: Stop accepting new traffic
    // (Let Kubernetes readiness probe fail)
    process.env.SPOT_TERMINATING = "true";
    
    // Step 2: Drain in-flight requests (allow up to 60 seconds)
    await new Promise(r => setTimeout(r, 60000));
    
    // Step 3: Cleanup
    await this.cleanup();
    
    // Step 4: Graceful shutdown
    process.exit(0);
  }

  private async cleanup() {
    // Flush logs
    // Close database connections
    // Dequeue any locked messages
    this.logger.log("Cleanup complete");
  }

  isTerminating(): boolean {
    return this.isBeingTerminated;
  }
}
```

---

## ขั้นตอนที่ 1325: Database Cost Optimization

```typescript
// src/database/query-optimizer.ts

import { Injectable, Logger } from "@nestjs/common";
import { DataSource } from "typeorm";

@Injectable()
export class QueryOptimizerService {
  private readonly logger = new Logger(QueryOptimizerService.name);

  constructor(private readonly dataSource: DataSource) {}

  // N+1 Query detection
  async analyzeQueryPatterns(): Promise<QueryAnalysis[]> {
    const slowQueries = await this.dataSource.query(`
      SELECT 
        query,
        calls,
        total_time,
        mean_time,
        max_time,
        rows
      FROM pg_stat_statements
      WHERE mean_time > 100  -- queries taking > 100ms avg
      ORDER BY mean_time DESC
      LIMIT 20
    `);

    return slowQueries.map((q: any) => ({
      query: q.query.substring(0, 200),
      calls: q.calls,
      avgTimeMs: parseFloat(q.mean_time),
      maxTimeMs: parseFloat(q.max_time),
      totalTimeMs: parseFloat(q.total_time),
      rowsPerCall: q.rows / q.calls
    }));
  }

  // Check for missing indexes
  async findMissingIndexes(): Promise<string[]> {
    const result = await this.dataSource.query(`
      SELECT
        schemaname,
        tablename,
        attname AS column_name,
        n_distinct,
        correlation
      FROM pg_stats
      WHERE n_distinct > 100  -- High cardinality columns
        AND schemaname = 'public'
      ORDER BY n_distinct DESC
    `);

    const recommendations: string[] = [];
    
    for (const row of result) {
      recommendations.push(
        `Consider index on ${row.tablename}.${row.column_name} (${row.n_distinct} distinct values)`
      );
    }
    
    return recommendations;
  }

  // Connection pool tuning
  async getConnectionStats(): Promise<ConnectionStats> {
    const result = await this.dataSource.query(`
      SELECT 
        count(*) as total,
        count(*) FILTER (WHERE state = 'active') as active,
        count(*) FILTER (WHERE state = 'idle') as idle,
        count(*) FILTER (WHERE wait_event_type = 'Lock') as waiting
      FROM pg_stat_activity
      WHERE datname = current_database()
    `);

    return result[0];
  }
}

interface QueryAnalysis {
  query: string;
  calls: number;
  avgTimeMs: number;
  maxTimeMs: number;
  totalTimeMs: number;
  rowsPerCall: number;
}

interface ConnectionStats {
  total: number;
  active: number;
  idle: number;
  waiting: number;
}
```

---

## ขั้นตอนที่ 1326: Caching for Cost Reduction

```typescript
// src/cache/smart-cache.ts
// Intelligent caching to reduce database load

import { Injectable, Logger } from "@nestjs/common";
import Redis from "ioredis";

interface CacheConfig {
  key: string;
  ttl: number;         // seconds
  staleWhileRevalidate?: number;  // seconds
  tags?: string[];     // for cache invalidation by tag
}

@Injectable()
export class SmartCacheService {
  private readonly logger = new Logger(SmartCacheService.name);
  
  constructor(private readonly redis: Redis) {}

  async getOrSet<T>(
    config: CacheConfig,
    fetchFn: () => Promise<T>
  ): Promise<T> {
    const cacheKey = `cache:${config.key}`;
    const staleKey = `stale:${config.key}`;

    // Try to get from cache
    const cached = await this.redis.get(cacheKey);
    
    if (cached) {
      // Check if stale-while-revalidate should kick in
      const stale = await this.redis.get(staleKey);
      if (!stale && config.staleWhileRevalidate) {
        // Return stale data, refresh in background
        this.refreshInBackground(cacheKey, staleKey, config, fetchFn);
      }
      return JSON.parse(cached);
    }

    // Cache miss - fetch fresh data
    const data = await fetchFn();
    await this.set(config, data);
    return data;
  }

  private async refreshInBackground<T>(
    cacheKey: string,
    staleKey: string,
    config: CacheConfig,
    fetchFn: () => Promise<T>
  ) {
    // Set stale marker to prevent multiple refreshes
    await this.redis.set(staleKey, "1", "EX", config.staleWhileRevalidate ?? 60);
    
    try {
      const data = await fetchFn();
      await this.set(config, data);
      this.logger.debug(`Cache refreshed for ${config.key}`);
    } catch (error) {
      this.logger.warn(`Background refresh failed for ${config.key}`);
    }
  }

  async set<T>(config: CacheConfig, data: T): Promise<void> {
    const pipeline = this.redis.multi();
    const cacheKey = `cache:${config.key}`;
    
    pipeline.set(cacheKey, JSON.stringify(data), "EX", config.ttl);
    
    // Tag tracking for bulk invalidation
    if (config.tags) {
      for (const tag of config.tags) {
        pipeline.sadd(`tag:${tag}`, cacheKey);
        pipeline.expire(`tag:${tag}`, config.ttl + 60);
      }
    }
    
    await pipeline.exec();
  }

  async invalidateByTag(tag: string): Promise<void> {
    const keys = await this.redis.smembers(`tag:${tag}`);
    if (keys.length === 0) return;
    
    const pipeline = this.redis.multi();
    keys.forEach(key => pipeline.del(key));
    pipeline.del(`tag:${tag}`);
    await pipeline.exec();
    
    this.logger.log(`Invalidated ${keys.length} cache entries for tag: ${tag}`);
  }

  async getCacheHitRate(): Promise<number> {
    const info = await this.redis.info("stats");
    const lines = info.split("\n");
    
    const hits = parseInt(lines.find(l => l.startsWith("keyspace_hits"))?.split(":")[1] ?? "0");
    const misses = parseInt(lines.find(l => l.startsWith("keyspace_misses"))?.split(":")[1] ?? "0");
    
    const total = hits + misses;
    return total === 0 ? 0 : hits / total;
  }
}
```

---

## ขั้นตอนที่ 1327: Resource Right-Sizing

```typescript
// src/tools/resource-analyzer.ts
// Analyze resource usage to right-size pods

export interface ResourceUsage {
  service: string;
  period: string;
  cpu: {
    requested: string;
    limit: string;
    p50: string;
    p95: string;
    p99: string;
    max: string;
  };
  memory: {
    requested: string;
    limit: string;
    p50: string;
    p95: string;
    p99: string;
    max: string;
  };
  recommendation: ResourceRecommendation;
}

interface ResourceRecommendation {
  cpu: { request: string; limit: string };
  memory: { request: string; limit: string };
  estimatedMonthlySavings: number;
}

export class ResourceAnalyzer {
  
  analyzeUsage(prometheusData: any): ResourceRecommendation {
    const cpuP99 = prometheusData.cpu_p99;  // 0.8 = 800m CPU
    const memP99 = prometheusData.mem_p99;  // bytes

    // Recommended: 110% of p99 usage for request
    // Limit: 150% of p99 for headroom
    const cpuRequest = Math.ceil(cpuP99 * 1.1 * 1000) + "m";
    const cpuLimit = Math.ceil(cpuP99 * 1.5 * 1000) + "m";
    
    const memRequestMi = Math.ceil(memP99 * 1.1 / 1024 / 1024);
    const memLimitMi = Math.ceil(memP99 * 1.5 / 1024 / 1024);
    
    // Calculate savings
    const currentCpuRequest = parseFloat(prometheusData.cpu_requested);
    const suggestedCpuRequest = cpuP99 * 1.1;
    const cpuDiff = currentCpuRequest - suggestedCpuRequest;
    
    // $0.048 per vCPU-hour on GKE
    const monthlySavings = cpuDiff * 0.048 * 24 * 30;

    return {
      cpu: { request: cpuRequest, limit: cpuLimit },
      memory: { request: `${memRequestMi}Mi`, limit: `${memLimitMi}Mi` },
      estimatedMonthlySavings: Math.max(0, monthlySavings)
    };
  }

  generateReport(usageData: ResourceUsage[]): string {
    let totalSavings = 0;
    const lines = ["# Resource Optimization Report\n"];

    for (const service of usageData) {
      const rec = service.recommendation;
      totalSavings += rec.estimatedMonthlySavings;

      lines.push(`## ${service.service}`);
      lines.push(`- CPU: Request ${rec.cpu.request}, Limit ${rec.cpu.limit}`);
      lines.push(`- Memory: Request ${rec.memory.request}, Limit ${rec.memory.limit}`);
      lines.push(`- Estimated savings: $${rec.estimatedMonthlySavings.toFixed(2)}/month\n`);
    }

    lines.push(`## Total Estimated Savings: $${totalSavings.toFixed(2)}/month`);
    return lines.join("\n");
  }
}
```

---

## ขั้นตอนที่ 1328: Image Optimization

```dockerfile
# Multi-stage build to minimize image size
# Before: 1.2GB, After: ~120MB

# Stage 1: Dependencies
FROM node:20-alpine AS deps
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install only production dependencies
RUN npm ci --only=production

# Stage 2: Build
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Stage 3: Production image
FROM node:20-alpine AS runner
WORKDIR /app

# Security: Non-root user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nodeuser

# Copy only what's needed
COPY --from=deps --chown=nodeuser:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodeuser:nodejs /app/dist ./dist

USER nodeuser

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"

CMD ["node", "dist/main.js"]
```

---

## ขั้นตอนที่ 1329: Database Connection Pooling

```typescript
// src/database/connection-pool.ts
// Optimize database connections

import { TypeOrmModuleOptions } from "@nestjs/typeorm";

export function getDatabaseConfig(): TypeOrmModuleOptions {
  return {
    type: "postgres",
    host: process.env.DB_HOST,
    port: parseInt(process.env.DB_PORT ?? "5432"),
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    
    // Connection Pool Settings
    // Rule of thumb: (2 * CPU_CORES) + effective_spindle_count
    // For 4 CPU: (2*4) + 1 = 9 connections
    extra: {
      // PgBouncer transaction pooling settings
      poolSize: parseInt(process.env.DB_POOL_SIZE ?? "10"),
      connectionTimeoutMillis: 3000,
      idleTimeoutMillis: 60000,
      maxUses: 7500,  // Recycle connections after 7500 uses
      
      // Keep-alive to prevent connection drops
      keepAlive: true,
      keepAliveInitialDelayMillis: 60000,
    },
    
    // Logging
    logging: process.env.NODE_ENV === "development" ? "all" : ["error"],
    
    synchronize: false,  // Never in production
    migrations: ["dist/migrations/*.js"],
    migrationsRun: true
  };
}
```

---

## ขั้นตอนที่ 1330: Cost Allocation Tags

```typescript
// src/infrastructure/cost-tags.ts
// Apply consistent tags for cost attribution

export interface ResourceTags {
  Environment: "production" | "staging" | "development";
  Service: string;
  Team: string;
  Project: string;
  CostCenter: string;
  ManagedBy: "terraform" | "helm" | "manual";
}

export function getRequiredTags(
  service: string,
  team: string
): ResourceTags {
  return {
    Environment: (process.env.NODE_ENV ?? "development") as any,
    Service: service,
    Team: team,
    Project: process.env.PROJECT_NAME ?? "unknown",
    CostCenter: process.env.COST_CENTER ?? "engineering",
    ManagedBy: "terraform"
  };
}

// Kubernetes labels for cost tracking
export const kubernetesLabels = {
  "app.kubernetes.io/name": "product-service",
  "app.kubernetes.io/instance": "product-service-prod",
  "app.kubernetes.io/component": "backend",
  "app.kubernetes.io/part-of": "ecommerce-platform",
  "app.kubernetes.io/managed-by": "helm",
  "team": "platform",
  "cost-center": "engineering"
};
```

---

## ขั้นตอนที่ 1331: Cost Budget Alerts

```typescript
// src/cost/budget-alerts.ts

import { CostExplorer } from "@aws-sdk/client-cost-explorer";
import { Budgets } from "@aws-sdk/client-budgets";

export class CostBudgetService {
  private readonly budgets = new Budgets({ region: "us-east-1" });

  async createMonthlyBudget(
    accountId: string,
    budgetName: string,
    amountUSD: number,
    emails: string[]
  ): Promise<void> {
    await this.budgets.createBudget({
      AccountId: accountId,
      Budget: {
        BudgetName: budgetName,
        BudgetType: "COST",
        TimeUnit: "MONTHLY",
        BudgetLimit: {
          Amount: amountUSD.toString(),
          Unit: "USD"
        },
        CostFilters: {
          TagKeyValue: ["user:Environment$production"]
        }
      },
      NotificationsWithSubscribers: [
        {
          Notification: {
            NotificationType: "ACTUAL",
            ComparisonOperator: "GREATER_THAN",
            Threshold: 80,  // Alert at 80% of budget
            ThresholdType: "PERCENTAGE"
          },
          Subscribers: emails.map(email => ({
            SubscriptionType: "EMAIL",
            Address: email
          }))
        },
        {
          Notification: {
            NotificationType: "FORECASTED",
            ComparisonOperator: "GREATER_THAN",
            Threshold: 100,  // Alert when forecast exceeds 100%
            ThresholdType: "PERCENTAGE"
          },
          Subscribers: emails.map(email => ({
            SubscriptionType: "EMAIL",
            Address: email
          }))
        }
      ]
    });
  }
}
```

---

## ขั้นตอนที่ 1332: Lambda Cost Optimization

```typescript
// src/lambda/optimized-handler.ts
// Optimize AWS Lambda for cost

// Use connection reuse across invocations
import { DataSource } from "typeorm";
import Redis from "ioredis";

let dataSource: DataSource | null = null;
let redis: Redis | null = null;

// Reuse connections between Lambda invocations
async function getDatabase(): Promise<DataSource> {
  if (!dataSource || !dataSource.isInitialized) {
    dataSource = new DataSource({
      type: "postgres",
      url: process.env.DATABASE_URL,
      // Connection pool: Lambda only runs 1 request at a time
      // so pool of 1 is fine
      extra: { max: 1, min: 0 }
    });
    await dataSource.initialize();
  }
  return dataSource;
}

async function getRedis(): Promise<Redis> {
  if (!redis) {
    redis = new Redis(process.env.REDIS_URL ?? "");
    // Lambda: use lazy connect
    redis.on("error", (e) => console.error("Redis error:", e));
  }
  return redis;
}

// Optimized Lambda handler
export const handler = async (event: any) => {
  // Connection reuse reduces cold start time and costs
  const db = await getDatabase();
  
  // Process event...
  const result = await processEvent(event, db);
  
  return {
    statusCode: 200,
    body: JSON.stringify(result)
  };
};

async function processEvent(event: any, db: DataSource) {
  return {};
}
```

---

## ขั้นตอนที่ 1333: Request Deduplication

```typescript
// src/cache/request-deduplication.ts
// Prevent duplicate expensive operations

import Redis from "ioredis";

export class RequestDeduplicator {
  constructor(private readonly redis: Redis) {}

  async deduplicateRequest<T>(
    key: string,
    ttlSeconds: number,
    fn: () => Promise<T>
  ): Promise<T> {
    const lockKey = `lock:${key}`;
    const resultKey = `result:${key}`;

    // Check if result already cached
    const cached = await this.redis.get(resultKey);
    if (cached) {
      return JSON.parse(cached);
    }

    // Try to acquire lock (only one execution at a time)
    const locked = await this.redis.set(lockKey, "1", "EX", 30, "NX");
    
    if (!locked) {
      // Wait for the other process to complete
      return this.waitForResult(resultKey, 30000);
    }

    try {
      // We have the lock - execute the function
      const result = await fn();
      
      // Cache the result
      await this.redis.set(
        resultKey, 
        JSON.stringify(result), 
        "EX", 
        ttlSeconds
      );
      
      return result;
    } finally {
      await this.redis.del(lockKey);
    }
  }

  private async waitForResult<T>(
    resultKey: string,
    timeoutMs: number
  ): Promise<T> {
    const startTime = Date.now();
    
    while (Date.now() - startTime < timeoutMs) {
      await new Promise(r => setTimeout(r, 100));
      
      const result = await this.redis.get(resultKey);
      if (result) return JSON.parse(result);
    }
    
    throw new Error("Timeout waiting for deduplicated result");
  }
}
```

---

## ขั้นตอนที่ 1334: Storage Optimization

```typescript
// src/storage/s3-optimizer.ts
// Optimize S3 storage costs

import { S3Client, PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

export class S3OptimizerService {
  private readonly s3 = new S3Client({ region: process.env.AWS_REGION ?? "us-east-1" });
  private readonly bucket = process.env.S3_BUCKET ?? "";

  // Use appropriate storage class based on access patterns
  async uploadWithSmartTiering(
    key: string,
    data: Buffer,
    contentType: string,
    accessFrequency: "frequent" | "infrequent" | "archive"
  ) {
    const storageClass = {
      frequent: "STANDARD",
      infrequent: "STANDARD_IA",  // ~40% cheaper, min 30 days
      archive: "GLACIER_IR"       // ~68% cheaper, min 90 days
    }[accessFrequency];

    await this.s3.send(new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: data,
      ContentType: contentType,
      StorageClass: storageClass as any
    }));
  }

  // Presigned URLs avoid proxying through app
  async getPresignedDownloadUrl(key: string, expiresInSeconds: number = 3600): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: key
    });

    return getSignedUrl(this.s3, command, { expiresIn: expiresInSeconds });
  }

  // Image compression before upload
  async uploadCompressedImage(
    key: string,
    imageBuffer: Buffer,
    quality: number = 80
  ): Promise<{ key: string; originalSize: number; compressedSize: number; savings: number }> {
    const sharp = await import("sharp");
    
    const compressed = await sharp.default(imageBuffer)
      .jpeg({ quality })
      .toBuffer();

    await this.uploadWithSmartTiering(key, compressed, "image/jpeg", "frequent");

    const savings = ((imageBuffer.length - compressed.length) / imageBuffer.length) * 100;

    return {
      key,
      originalSize: imageBuffer.length,
      compressedSize: compressed.length,
      savings
    };
  }
}
```

---

## ขั้นตอนที่ 1335: Cost Optimization Patterns

```typescript
// src/patterns/batch-processor.ts
// Batch operations to reduce API calls and costs

import { Injectable, Logger } from "@nestjs/common";

interface BatchConfig<T, R> {
  maxBatchSize: number;
  maxWaitMs: number;
  processBatch: (items: T[]) => Promise<R[]>;
}

export class BatchProcessor<T, R> {
  private batch: Array<{ item: T; resolve: (result: R) => void; reject: (error: Error) => void }> = [];
  private timer: NodeJS.Timeout | null = null;
  private readonly logger = new Logger(BatchProcessor.name);

  constructor(private readonly config: BatchConfig<T, R>) {}

  async add(item: T): Promise<R> {
    return new Promise<R>((resolve, reject) => {
      this.batch.push({ item, resolve, reject });

      // Process immediately if batch is full
      if (this.batch.length >= this.config.maxBatchSize) {
        this.flush();
      } else if (!this.timer) {
        // Otherwise wait for more items
        this.timer = setTimeout(() => this.flush(), this.config.maxWaitMs);
      }
    });
  }

  private async flush() {
    if (this.timer) {
      clearTimeout(this.timer);
      this.timer = null;
    }

    const currentBatch = this.batch.splice(0, this.config.maxBatchSize);
    if (currentBatch.length === 0) return;

    try {
      const items = currentBatch.map(b => b.item);
      const results = await this.config.processBatch(items);
      
      currentBatch.forEach((b, i) => b.resolve(results[i]));
    } catch (error) {
      currentBatch.forEach(b => b.reject(error as Error));
    }
  }
}

// Email notification batching (reduce SES API calls)
const emailBatcher = new BatchProcessor<
  { to: string; subject: string; body: string },
  { messageId: string }
>({
  maxBatchSize: 50,
  maxWaitMs: 1000,
  processBatch: async (emails) => {
    // Send all emails in one SES batch call
    // Instead of individual calls
    return emails.map(() => ({ messageId: `msg-${Date.now()}` }));
  }
});
```

---

## ขั้นตอนที่ 1336: Cost Dashboard

```typescript
// src/cost/cost-dashboard.service.ts

import { Injectable, Logger } from "@nestjs/common";

export interface CostSummary {
  currentMonthTotal: number;
  forecastedMonthTotal: number;
  lastMonthTotal: number;
  percentageChange: number;
  topServices: ServiceCost[];
  alerts: CostAlert[];
  recommendations: Recommendation[];
}

interface ServiceCost {
  service: string;
  cost: number;
  percentageOfTotal: number;
  trend: "up" | "down" | "stable";
}

interface CostAlert {
  severity: "critical" | "warning" | "info";
  message: string;
}

interface Recommendation {
  title: string;
  description: string;
  estimatedMonthlySavings: number;
  effort: "low" | "medium" | "high";
}

@Injectable()
export class CostDashboardService {
  private readonly logger = new Logger(CostDashboardService.name);

  async getSummary(): Promise<CostSummary> {
    const [current, lastMonth, forecast] = await Promise.all([
      this.getCurrentMonthCost(),
      this.getLastMonthCost(),
      this.getForecast()
    ]);

    const percentageChange = lastMonth > 0
      ? ((current - lastMonth) / lastMonth) * 100
      : 0;

    const alerts: CostAlert[] = [];
    
    if (current > forecast * 0.9) {
      alerts.push({
        severity: "warning",
        message: `Spending is at ${((current / forecast) * 100).toFixed(0)}% of forecasted amount`
      });
    }

    if (percentageChange > 20) {
      alerts.push({
        severity: "critical",
        message: `Spending increased ${percentageChange.toFixed(1)}% vs last month`
      });
    }

    return {
      currentMonthTotal: current,
      forecastedMonthTotal: forecast,
      lastMonthTotal: lastMonth,
      percentageChange,
      topServices: await this.getTopServices(),
      alerts,
      recommendations: this.getRecommendations()
    };
  }

  private async getCurrentMonthCost(): Promise<number> { return 1250; }
  private async getLastMonthCost(): Promise<number> { return 1100; }
  private async getForecast(): Promise<number> { return 1400; }
  private async getTopServices(): Promise<ServiceCost[]> { return []; }
  
  private getRecommendations(): Recommendation[] {
    return [
      {
        title: "Right-size underutilized EC2 instances",
        description: "3 instances are running at < 20% CPU utilization",
        estimatedMonthlySavings: 180,
        effort: "low"
      },
      {
        title: "Use Reserved Instances for stable workloads",
        description: "Convert 5 On-Demand instances to 1-year Reserved Instances",
        estimatedMonthlySavings: 320,
        effort: "medium"
      },
      {
        title: "Enable S3 Intelligent-Tiering",
        description: "Auto-tiering for 2TB of infrequently accessed data",
        estimatedMonthlySavings: 45,
        effort: "low"
      }
    ];
  }
}
```

---

## ขั้นตอนที่ 1337: CDN Configuration

```typescript
// src/cdn/cloudfront-config.ts
// Use CDN to reduce origin server costs

export const cloudfrontBehaviors = {
  // Static assets - cache forever (content-addressed)
  staticAssets: {
    pathPattern: "/static/*",
    cachePolicyId: "CachingOptimized",  // AWS managed policy
    compress: true,
    viewerProtocolPolicy: "redirect-to-https",
    cacheSettings: {
      defaultTTL: 86400 * 365,  // 1 year (use versioned URLs)
      maxTTL: 86400 * 365,
      minTTL: 86400 * 365
    }
  },
  
  // API responses - short cache with revalidation
  apiResponses: {
    pathPattern: "/api/products*",
    cachePolicyId: "CachingOptimized",
    compress: true,
    cacheSettings: {
      defaultTTL: 60,    // 1 minute
      maxTTL: 300,       // 5 minutes
      minTTL: 0
    },
    forwardedValues: {
      headers: ["Authorization"],     // Vary by auth header
      queryStringCacheKeys: ["page", "limit", "sort"]
    }
  },
  
  // Dynamic content - no cache
  dynamicContent: {
    pathPattern: "/api/orders*",
    cachePolicyId: "CachingDisabled",
    viewerProtocolPolicy: "redirect-to-https"
  }
};
```

---

## ขั้นตอนที่ 1338: Cleanup Automation

```typescript
// src/cleanup/resource-cleanup.ts
// Automate cleanup of unused resources

import { Cron } from "@nestjs/schedule";
import { Injectable, Logger } from "@nestjs/common";
import { S3Client, DeleteObjectCommand, ListObjectsV2Command } from "@aws-sdk/client-s3";

@Injectable()
export class ResourceCleanupService {
  private readonly logger = new Logger(ResourceCleanupService.name);
  private readonly s3 = new S3Client({ region: process.env.AWS_REGION ?? "us-east-1" });

  // Clean up old temp files daily at 3am
  @Cron("0 3 * * *")
  async cleanupTempFiles() {
    const bucket = process.env.S3_BUCKET ?? "";
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - 7);  // 7 days old

    let deleted = 0;
    let continuationToken: string | undefined;

    do {
      const listing = await this.s3.send(new ListObjectsV2Command({
        Bucket: bucket,
        Prefix: "temp/",
        ContinuationToken: continuationToken
      }));

      for (const obj of listing.Contents ?? []) {
        if (obj.LastModified && obj.LastModified < cutoffDate) {
          await this.s3.send(new DeleteObjectCommand({
            Bucket: bucket,
            Key: obj.Key!
          }));
          deleted++;
        }
      }

      continuationToken = listing.NextContinuationToken;
    } while (continuationToken);

    this.logger.log(`Cleaned up ${deleted} temp files`);
  }

  // Clean up expired user sessions weekly
  @Cron("0 4 * * 0")  // Sunday 4am
  async cleanupExpiredSessions() {
    // Delete sessions older than 30 days
    // Implementation depends on session storage
    this.logger.log("Cleaned up expired sessions");
  }
}
```

---

## ขั้นตอนที่ 1339: FinOps Best Practices

```typescript
// Cost optimization checklist and automation

export const costOptimizationChecklist = {
  compute: [
    { item: "Enable HPA for all stateless services", priority: "high", savings: "high" },
    { item: "Use Spot instances for batch/non-critical workloads", priority: "high", savings: "high" },
    { item: "Right-size instance types based on p95 usage", priority: "high", savings: "medium" },
    { item: "Use ARM instances (Graviton) where possible", priority: "medium", savings: "20%" },
    { item: "Schedule dev/staging scale-down after hours", priority: "medium", savings: "medium" }
  ],
  storage: [
    { item: "Enable S3 Intelligent-Tiering", priority: "high", savings: "medium" },
    { item: "Set up S3 Lifecycle policies", priority: "high", savings: "medium" },
    { item: "Compress logs before archiving", priority: "medium", savings: "low" },
    { item: "Clean up unattached EBS volumes", priority: "high", savings: "immediate" }
  ],
  database: [
    { item: "Use connection pooling (PgBouncer)", priority: "high", savings: "medium" },
    { item: "Right-size RDS instances", priority: "high", savings: "high" },
    { item: "Use Reserved Instances for production DB", priority: "high", savings: "40%" },
    { item: "Archive old data to S3 + Athena", priority: "medium", savings: "medium" }
  ],
  network: [
    { item: "Use CDN for static assets", priority: "high", savings: "high" },
    { item: "Use VPC endpoints to avoid NAT costs", priority: "medium", savings: "medium" },
    { item: "Optimize API payload sizes", priority: "low", savings: "low" }
  ]
};
```

---

## ขั้นตอนที่ 1340: Cost Optimization Monitoring

```yaml
# prometheus-cost-alerts.yaml

groups:
  - name: cost_optimization
    rules:
      # Alert when a service has been idle for too long
      - alert: ServiceIdle
        expr: |
          rate(http_requests_total[1h]) == 0
        for: 2h
        labels:
          severity: info
        annotations:
          summary: "Service {{ $labels.service }} has no traffic for 2h"
          description: "Consider scaling down or checking if this service is needed"
      
      # Alert on high cache miss rate (indicates inefficient caching)
      - alert: HighCacheMissRate
        expr: |
          sum(rate(cache_misses_total[5m])) /
          (sum(rate(cache_hits_total[5m])) + sum(rate(cache_misses_total[5m])))
          > 0.5
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "High cache miss rate (>50%)"
          description: "Cache is not effective - review cache TTL and keys"
      
      # Alert when p99 memory > 80% of limit (may need right-sizing)
      - alert: HighMemoryUsage
        expr: |
          container_memory_working_set_bytes /
          kube_pod_container_resource_limits{resource="memory"} > 0.8
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.container }} using > 80% memory limit"
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: Auto-scaling
1. ตั้งค่า HPA สำหรับ service
2. ทดสอบ scale-up เมื่อ load สูง
3. ตรวจสอบ scale-down behavior

### แบบฝึกหัดที่ 2: Caching
1. implement SmartCacheService
2. วัด cache hit rate
3. เพิ่ม cache invalidation by tag

### แบบฝึกหัดที่ 3: Cost Analysis
1. สร้าง cost breakdown report
2. ระบุ top 3 cost optimization opportunities
3. implement cleanup automation

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- Cloud cost visibility
- Auto-scaling strategies
- Spot instance handling
- Database connection optimization
- Smart caching
- Resource right-sizing
- Image optimization
- Storage tiering
- FinOps best practices
- Cost monitoring and alerting

**Part ถัดไป**: Data Pipelines
