# Part 77 | ขั้นตอนที่ 1341-1360 จาก 1000+

# Data Pipelines & ETL

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. ออกแบบ ETL pipelines ด้วย Node.js
2. implement batch processing
3. ใช้ streaming data pipelines
4. integrate กับ Apache Kafka
5. จัดการ data transformations
6. monitor pipeline health

---

## ขั้นตอนที่ 1341: Data Pipeline Architecture

```
ETL Pipeline Architecture:

Extract → Transform → Load

┌─────────┐    ┌────────────┐    ┌──────────┐
│ Sources │ → │ Transform  │ → │  Targets │
│         │    │            │    │          │
│ - APIs  │    │ - Validate │    │ - DB     │
│ - DBs   │    │ - Clean    │    │ - DW     │
│ - Files │    │ - Enrich   │    │ - S3     │
│ - Kafka │    │ - Aggregate│    │ - Search │
└─────────┘    └────────────┘    └──────────┘

Streaming Pipeline:
┌──────┐   ┌─────────┐   ┌─────────┐   ┌──────┐
│Source│→→→│ Kafka   │→→→│Processor│→→→│Sink  │
│      │   │ Topic   │   │ (Node)  │   │      │
└──────┘   └─────────┘   └─────────┘   └──────┘
```

---

## ขั้นตอนที่ 1342: ETL Pipeline Base

```typescript
// src/pipeline/base-pipeline.ts

import { Logger } from "@nestjs/common";
import * as promClient from "prom-client";

export interface PipelineConfig {
  name: string;
  batchSize: number;
  errorHandling: "skip" | "retry" | "fail";
  retryAttempts: number;
  retryDelayMs: number;
}

export interface PipelineRecord<T = any> {
  id: string;
  source: string;
  data: T;
  timestamp: Date;
  metadata?: Record<string, any>;
}

export interface PipelineResult {
  processed: number;
  succeeded: number;
  failed: number;
  skipped: number;
  durationMs: number;
  errors: Array<{ id: string; error: string }>;
}

export abstract class BasePipeline<TInput, TOutput> {
  protected readonly logger: Logger;
  
  private readonly recordsProcessed: promClient.Counter;
  private readonly processingDuration: promClient.Histogram;

  constructor(protected readonly config: PipelineConfig) {
    this.logger = new Logger(config.name);
    
    this.recordsProcessed = new promClient.Counter({
      name: `pipeline_records_processed_total`,
      help: "Total records processed by pipeline",
      labelNames: ["pipeline", "status"]
    });

    this.processingDuration = new promClient.Histogram({
      name: `pipeline_processing_duration_ms`,
      help: "Pipeline processing duration",
      labelNames: ["pipeline"],
      buckets: [100, 500, 1000, 5000, 10000, 30000, 60000]
    });
  }

  async run(inputs: PipelineRecord<TInput>[]): Promise<PipelineResult> {
    const startTime = Date.now();
    const result: PipelineResult = {
      processed: 0,
      succeeded: 0,
      failed: 0,
      skipped: 0,
      durationMs: 0,
      errors: []
    };

    this.logger.log(`Starting pipeline with ${inputs.length} records`);

    // Process in batches
    for (let i = 0; i < inputs.length; i += this.config.batchSize) {
      const batch = inputs.slice(i, i + this.config.batchSize);
      await this.processBatch(batch, result);
      
      this.logger.debug(
        `Batch ${Math.floor(i / this.config.batchSize) + 1} complete: ` +
        `${result.succeeded} succeeded, ${result.failed} failed`
      );
    }

    result.durationMs = Date.now() - startTime;
    
    this.processingDuration.observe({ pipeline: this.config.name }, result.durationMs);
    this.logger.log(
      `Pipeline complete: ${result.succeeded}/${result.processed} succeeded in ${result.durationMs}ms`
    );

    return result;
  }

  private async processBatch(
    batch: PipelineRecord<TInput>[],
    result: PipelineResult
  ): Promise<void> {
    for (const record of batch) {
      result.processed++;

      try {
        // Extract
        const extracted = await this.extract(record);
        
        // Transform
        const transformed = await this.transform(extracted);
        
        // Validate
        const isValid = await this.validate(transformed);
        if (!isValid) {
          result.skipped++;
          this.recordsProcessed.inc({ pipeline: this.config.name, status: "skipped" });
          continue;
        }
        
        // Load
        await this.load(transformed, record);
        
        result.succeeded++;
        this.recordsProcessed.inc({ pipeline: this.config.name, status: "success" });
        
      } catch (error) {
        const errorMsg = (error as Error).message;
        
        if (this.config.errorHandling === "skip") {
          result.skipped++;
          result.errors.push({ id: record.id, error: errorMsg });
        } else if (this.config.errorHandling === "retry") {
          const success = await this.retryRecord(record, result);
          if (!success) {
            result.failed++;
            result.errors.push({ id: record.id, error: errorMsg });
          } else {
            result.succeeded++;
          }
        } else {
          throw error;  // Fail pipeline
        }
        
        this.recordsProcessed.inc({ pipeline: this.config.name, status: "failed" });
      }
    }
  }

  private async retryRecord(
    record: PipelineRecord<TInput>,
    result: PipelineResult
  ): Promise<boolean> {
    for (let attempt = 1; attempt <= this.config.retryAttempts; attempt++) {
      await new Promise(r => setTimeout(r, this.config.retryDelayMs * attempt));
      
      try {
        const extracted = await this.extract(record);
        const transformed = await this.transform(extracted);
        await this.load(transformed, record);
        return true;
      } catch (error) {
        this.logger.warn(`Retry ${attempt}/${this.config.retryAttempts} failed for ${record.id}`);
      }
    }
    return false;
  }

  protected abstract extract(record: PipelineRecord<TInput>): Promise<TInput>;
  protected abstract transform(data: TInput): Promise<TOutput>;
  protected abstract validate(data: TOutput): Promise<boolean>;
  protected abstract load(data: TOutput, record: PipelineRecord<TInput>): Promise<void>;
}
```

---

## ขั้นตอนที่ 1343: Order Analytics Pipeline

```typescript
// src/pipelines/order-analytics.pipeline.ts

import { Injectable } from "@nestjs/common";
import { BasePipeline, PipelineRecord } from "./base-pipeline";
import { DataSource } from "typeorm";

interface OrderData {
  id: string;
  userId: string;
  items: Array<{
    productId: string;
    quantity: number;
    price: number;
  }>;
  totalAmount: number;
  status: string;
  createdAt: Date;
  country: string;
}

interface OrderAnalytics {
  orderId: string;
  userId: string;
  totalAmount: number;
  itemCount: number;
  avgItemPrice: number;
  date: string;
  month: string;
  year: number;
  country: string;
  category: string;
  isFirstOrder: boolean;
  processingDate: Date;
}

@Injectable()
export class OrderAnalyticsPipeline extends BasePipeline<OrderData, OrderAnalytics> {
  constructor(private readonly dataSource: DataSource) {
    super({
      name: "OrderAnalyticsPipeline",
      batchSize: 100,
      errorHandling: "skip",
      retryAttempts: 3,
      retryDelayMs: 1000
    });
  }

  protected async extract(record: PipelineRecord<OrderData>): Promise<OrderData> {
    // Data already extracted, just validate structure
    if (!record.data.id || !record.data.items) {
      throw new Error(`Invalid order data for record ${record.id}`);
    }
    return record.data;
  }

  protected async transform(data: OrderData): Promise<OrderAnalytics> {
    const itemCount = data.items.reduce((sum, item) => sum + item.quantity, 0);
    const avgItemPrice = data.totalAmount / (itemCount || 1);
    
    const date = new Date(data.createdAt);
    const dateStr = date.toISOString().split("T")[0];
    const month = `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, "0")}`;
    
    // Determine if first order
    const isFirstOrder = await this.checkFirstOrder(data.userId, data.id);
    
    return {
      orderId: data.id,
      userId: data.userId,
      totalAmount: data.totalAmount,
      itemCount,
      avgItemPrice,
      date: dateStr,
      month,
      year: date.getFullYear(),
      country: data.country,
      category: this.categorizeOrder(data.totalAmount),
      isFirstOrder,
      processingDate: new Date()
    };
  }

  protected async validate(data: OrderAnalytics): Promise<boolean> {
    // Skip orders with negative amounts
    if (data.totalAmount < 0) return false;
    if (!data.orderId || !data.userId) return false;
    return true;
  }

  protected async load(
    data: OrderAnalytics,
    record: PipelineRecord<OrderData>
  ): Promise<void> {
    await this.dataSource.query(`
      INSERT INTO order_analytics (
        order_id, user_id, total_amount, item_count, avg_item_price,
        date, month, year, country, category, is_first_order, processing_date
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12)
      ON CONFLICT (order_id) DO UPDATE SET
        processing_date = $12,
        is_first_order = $11
    `, [
      data.orderId, data.userId, data.totalAmount, data.itemCount,
      data.avgItemPrice, data.date, data.month, data.year, data.country,
      data.category, data.isFirstOrder, data.processingDate
    ]);
  }

  private async checkFirstOrder(userId: string, excludeOrderId: string): Promise<boolean> {
    const result = await this.dataSource.query(
      "SELECT COUNT(*) as count FROM orders WHERE user_id = $1 AND id != $2",
      [userId, excludeOrderId]
    );
    return parseInt(result[0].count) === 0;
  }

  private categorizeOrder(amount: number): string {
    if (amount < 50) return "small";
    if (amount < 200) return "medium";
    if (amount < 1000) return "large";
    return "enterprise";
  }
}
```

---

## ขั้นตอนที่ 1344: Streaming Pipeline with Kafka

```typescript
// src/pipelines/streaming-pipeline.ts

import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from "@nestjs/common";
import { Kafka, Consumer, EachMessagePayload } from "kafkajs";

interface StreamProcessor<T, R> {
  process(data: T): Promise<R | null>;
}

interface StreamConfig {
  topic: string;
  groupId: string;
  fromBeginning?: boolean;
  parallelism?: number;
}

@Injectable()
export class StreamingPipeline implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(StreamingPipeline.name);
  private consumer: Consumer;
  
  private readonly kafka = new Kafka({
    clientId: "streaming-pipeline",
    brokers: (process.env.KAFKA_BROKERS ?? "localhost:9092").split(",")
  });

  async onModuleInit() {
    this.consumer = this.kafka.consumer({
      groupId: "analytics-pipeline",
      sessionTimeout: 30000,
      heartbeatInterval: 3000
    });
    
    await this.consumer.connect();
    await this.startProcessing();
  }

  async onModuleDestroy() {
    await this.consumer.disconnect();
  }

  private async startProcessing() {
    await this.consumer.subscribe({
      topics: ["orders", "page-views", "user-events"],
      fromBeginning: false
    });

    await this.consumer.run({
      partitionsConsumedConcurrently: 3,
      eachBatchAutoResolve: true,
      eachBatch: async ({ batch, resolveOffset, heartbeat }) => {
        const { topic, partition, messages } = batch;
        
        this.logger.debug(
          `Processing batch of ${messages.length} messages from ${topic}[${partition}]`
        );

        for (const message of messages) {
          try {
            const data = JSON.parse(message.value?.toString() ?? "{}");
            
            await this.routeToProcessor(topic, data);
            
            // Commit offset after processing
            resolveOffset(message.offset);
            await heartbeat();
            
          } catch (error) {
            this.logger.error(
              `Error processing message from ${topic}: ${(error as Error).message}`
            );
          }
        }
      }
    });
  }

  private async routeToProcessor(topic: string, data: any): Promise<void> {
    switch (topic) {
      case "orders":
        await this.processOrder(data);
        break;
      case "page-views":
        await this.processPageView(data);
        break;
      case "user-events":
        await this.processUserEvent(data);
        break;
      default:
        this.logger.warn(`No processor for topic: ${topic}`);
    }
  }

  private async processOrder(data: any) {
    // Real-time order analytics
  }

  private async processPageView(data: any) {
    // Track page views
  }

  private async processUserEvent(data: any) {
    // User behavior analytics
  }
}
```

---

## ขั้นตอนที่ 1345: Data Validation Pipeline

```typescript
// src/pipelines/validation/schema-validator.ts

import { z } from "zod";

// Define expected data schemas
const OrderSchema = z.object({
  id: z.string().uuid(),
  userId: z.string().uuid(),
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity: z.number().positive().int(),
    price: z.number().positive()
  })).min(1),
  totalAmount: z.number().positive(),
  status: z.enum(["pending", "completed", "cancelled"]),
  createdAt: z.string().datetime()
});

const UserEventSchema = z.object({
  userId: z.string().uuid(),
  event: z.string().min(1),
  properties: z.record(z.unknown()),
  timestamp: z.string().datetime()
});

export class DataValidator {
  
  validateOrder(data: unknown): { valid: boolean; errors: string[]; data?: any } {
    const result = OrderSchema.safeParse(data);
    
    if (result.success) {
      return { valid: true, errors: [], data: result.data };
    }
    
    return {
      valid: false,
      errors: result.error.errors.map(e => `${e.path.join(".")}: ${e.message}`)
    };
  }

  validateUserEvent(data: unknown): { valid: boolean; errors: string[] } {
    const result = UserEventSchema.safeParse(data);
    
    if (result.success) {
      return { valid: true, errors: [] };
    }
    
    return {
      valid: false,
      errors: result.error.errors.map(e => `${e.path.join(".")}: ${e.message}`)
    };
  }

  // Business rule validation
  validateBusinessRules(order: z.infer<typeof OrderSchema>): string[] {
    const errors: string[] = [];
    
    // Validate total matches items
    const calculatedTotal = order.items.reduce(
      (sum, item) => sum + (item.price * item.quantity), 0
    );
    
    if (Math.abs(calculatedTotal - order.totalAmount) > 0.01) {
      errors.push(
        `Total mismatch: calculated ${calculatedTotal.toFixed(2)}, ` +
        `got ${order.totalAmount.toFixed(2)}`
      );
    }
    
    // Validate no duplicate products
    const productIds = order.items.map(i => i.productId);
    const unique = new Set(productIds);
    if (unique.size !== productIds.length) {
      errors.push("Duplicate products in order");
    }
    
    return errors;
  }
}
```

---

## ขั้นตอนที่ 1346: Data Enrichment

```typescript
// src/pipelines/enrichment/data-enricher.ts

import { Injectable } from "@nestjs/common";
import { DataSource } from "typeorm";
import Redis from "ioredis";

export interface EnrichedOrder {
  orderId: string;
  userId: string;
  user: {
    name: string;
    email: string;
    country: string;
    tier: "basic" | "premium" | "enterprise";
    joinDate: Date;
  };
  products: Array<{
    id: string;
    name: string;
    category: string;
    brand: string;
  }>;
  totalAmount: number;
  currency: string;
  exchangeRate: number;
  amountUSD: number;
}

@Injectable()
export class DataEnricher {
  constructor(
    private readonly dataSource: DataSource,
    private readonly redis: Redis
  ) {}

  async enrichOrder(orderId: string): Promise<EnrichedOrder> {
    const [order, user, products, currency] = await Promise.all([
      this.getOrder(orderId),
      this.getUserData(orderId),
      this.getProductData(orderId),
      this.getCurrencyRate()
    ]);

    return {
      orderId: order.id,
      userId: order.userId,
      user,
      products,
      totalAmount: order.totalAmount,
      currency: order.currency,
      exchangeRate: currency.rate,
      amountUSD: order.totalAmount * currency.rate
    };
  }

  private async getOrder(orderId: string) {
    const cached = await this.redis.get(`order:${orderId}`);
    if (cached) return JSON.parse(cached);

    const result = await this.dataSource.query(
      "SELECT * FROM orders WHERE id = $1",
      [orderId]
    );
    
    const order = result[0];
    await this.redis.set(`order:${orderId}`, JSON.stringify(order), "EX", 300);
    return order;
  }

  private async getUserData(orderId: string) {
    const result = await this.dataSource.query(`
      SELECT u.id, u.name, u.email, u.country, u.tier, u.created_at
      FROM users u
      JOIN orders o ON o.user_id = u.id
      WHERE o.id = $1
    `, [orderId]);
    
    return result[0] ?? null;
  }

  private async getProductData(orderId: string) {
    return this.dataSource.query(`
      SELECT p.id, p.name, p.category, p.brand
      FROM products p
      JOIN order_items oi ON oi.product_id = p.id
      WHERE oi.order_id = $1
    `, [orderId]);
  }

  private async getCurrencyRate(): Promise<{ rate: number }> {
    const cached = await this.redis.get("currency:USD:rate");
    if (cached) return { rate: parseFloat(cached) };

    // In production, fetch from forex API
    const rate = 1.0;
    await this.redis.set("currency:USD:rate", rate.toString(), "EX", 3600);
    return { rate };
  }
}
```

---

## ขั้นตอนที่ 1347: Pipeline Scheduler

```typescript
// src/pipelines/pipeline-scheduler.ts

import { Injectable, Logger } from "@nestjs/common";
import { Cron, SchedulerRegistry } from "@nestjs/schedule";
import { DataSource } from "typeorm";
import { OrderAnalyticsPipeline } from "./order-analytics.pipeline";

@Injectable()
export class PipelineScheduler {
  private readonly logger = new Logger(PipelineScheduler.name);

  constructor(
    private readonly dataSource: DataSource,
    private readonly orderPipeline: OrderAnalyticsPipeline
  ) {}

  // Daily ETL job at 2am
  @Cron("0 2 * * *")
  async runDailyETL() {
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1);
    const dateStr = yesterday.toISOString().split("T")[0];

    this.logger.log(`Starting daily ETL for ${dateStr}`);

    try {
      // Extract orders from previous day
      const orders = await this.dataSource.query(`
        SELECT * FROM orders
        WHERE DATE(created_at) = $1
        ORDER BY created_at ASC
      `, [dateStr]);

      this.logger.log(`Found ${orders.length} orders to process`);

      const records = orders.map((order: any) => ({
        id: order.id,
        source: "orders",
        data: order,
        timestamp: new Date()
      }));

      // Run pipeline
      const result = await this.orderPipeline.run(records);

      this.logger.log(
        `ETL complete: ${result.succeeded}/${result.processed} in ${result.durationMs}ms`
      );

      // Log to audit table
      await this.logPipelineRun("daily_order_etl", dateStr, result);

    } catch (error) {
      this.logger.error(`Daily ETL failed: ${(error as Error).message}`);
    }
  }

  // Hourly aggregation
  @Cron("5 * * * *")  // 5 minutes past every hour
  async runHourlyAggregation() {
    const lastHour = new Date();
    lastHour.setHours(lastHour.getHours() - 1);
    const hourStr = lastHour.toISOString().substring(0, 13);  // YYYY-MM-DDTHH

    try {
      await this.dataSource.query(`
        INSERT INTO hourly_stats (hour, total_orders, total_revenue, unique_users)
        SELECT
          DATE_TRUNC('hour', created_at) as hour,
          COUNT(*) as total_orders,
          SUM(total_amount) as total_revenue,
          COUNT(DISTINCT user_id) as unique_users
        FROM orders
        WHERE DATE_TRUNC('hour', created_at) = $1
        GROUP BY 1
        ON CONFLICT (hour) DO UPDATE SET
          total_orders = EXCLUDED.total_orders,
          total_revenue = EXCLUDED.total_revenue,
          unique_users = EXCLUDED.unique_users
      `, [hourStr + ":00:00"]);

      this.logger.debug(`Hourly aggregation complete for ${hourStr}`);
    } catch (error) {
      this.logger.error(`Hourly aggregation failed: ${(error as Error).message}`);
    }
  }

  private async logPipelineRun(
    pipelineName: string,
    period: string,
    result: any
  ): Promise<void> {
    await this.dataSource.query(`
      INSERT INTO pipeline_runs (pipeline_name, period, processed, succeeded, failed, duration_ms, ran_at)
      VALUES ($1, $2, $3, $4, $5, $6, NOW())
    `, [
      pipelineName, period, result.processed, result.succeeded,
      result.failed, result.durationMs
    ]);
  }
}
```

---

## ขั้นตอนที่ 1348: Change Data Capture (CDC)

```typescript
// src/cdc/postgres-cdc.ts
// Capture database changes in real-time

import { Injectable, OnModuleInit, Logger } from "@nestjs/common";
import { Client } from "pg";

interface CDCEvent {
  operation: "INSERT" | "UPDATE" | "DELETE";
  table: string;
  old?: Record<string, any>;
  new?: Record<string, any>;
  timestamp: Date;
  lsn: string;
}

@Injectable()
export class PostgresCDCService implements OnModuleInit {
  private readonly logger = new Logger(PostgresCDCService.name);
  private client: Client;
  private handlers: Map<string, (event: CDCEvent) => Promise<void>> = new Map();

  async onModuleInit() {
    this.client = new Client({
      connectionString: process.env.DATABASE_URL,
      replication: "database"
    });

    await this.client.connect();
    await this.setupReplication();
  }

  private async setupReplication() {
    // Create replication slot if not exists
    try {
      await this.client.query(`
        SELECT pg_create_logical_replication_slot(
          'node_cdc_slot', 
          'pgoutput'
        )
      `);
      this.logger.log("Replication slot created");
    } catch {
      this.logger.debug("Replication slot already exists");
    }

    // Create publication for tables we want to track
    await this.client.query(`
      CREATE PUBLICATION IF NOT EXISTS node_cdc_pub
      FOR TABLE orders, products, users
    `);

    // Start consuming changes
    this.startConsuming();
  }

  private async startConsuming() {
    // This is simplified - in production use pg-logical-replication
    this.logger.log("CDC replication started");
  }

  // Register handler for table changes
  on(table: string, handler: (event: CDCEvent) => Promise<void>): void {
    this.handlers.set(table, handler);
    this.logger.log(`CDC handler registered for table: ${table}`);
  }

  private async dispatch(event: CDCEvent): Promise<void> {
    const handler = this.handlers.get(event.table);
    if (handler) {
      await handler(event);
    }
  }
}

// Usage in module
@Injectable()
export class OrderSyncService implements OnModuleInit {
  constructor(private readonly cdc: PostgresCDCService) {}

  onModuleInit() {
    // Sync order changes to Elasticsearch
    this.cdc.on("orders", async (event) => {
      if (event.operation === "INSERT" || event.operation === "UPDATE") {
        // Index/update in Elasticsearch
        console.log(`Syncing order ${event.new?.id} to search index`);
      } else if (event.operation === "DELETE") {
        // Remove from Elasticsearch
        console.log(`Removing order ${event.old?.id} from search index`);
      }
    });
  }
}
```

---

## ขั้นตอนที่ 1349: Data Quality Monitoring

```typescript
// src/quality/data-quality.service.ts

import { Injectable, Logger } from "@nestjs/common";
import { DataSource } from "typeorm";
import * as promClient from "prom-client";

interface DataQualityRule {
  name: string;
  table: string;
  query: string;
  threshold: number;  // Minimum percentage
  severity: "critical" | "warning";
}

@Injectable()
export class DataQualityService {
  private readonly logger = new Logger(DataQualityService.name);
  
  private readonly qualityScore = new promClient.Gauge({
    name: "data_quality_score",
    help: "Data quality score (0-100)",
    labelNames: ["rule", "table"]
  });

  private readonly rules: DataQualityRule[] = [
    {
      name: "orders_have_users",
      table: "orders",
      query: `
        SELECT (COUNT(CASE WHEN u.id IS NOT NULL THEN 1 END)::float / COUNT(*) * 100) as score
        FROM orders o
        LEFT JOIN users u ON o.user_id = u.id
        WHERE o.created_at > NOW() - INTERVAL '1 hour'
      `,
      threshold: 99.9,
      severity: "critical"
    },
    {
      name: "orders_total_matches_items",
      table: "orders",
      query: `
        SELECT (COUNT(CASE WHEN ABS(o.total_amount - calculated.total) < 0.01 THEN 1 END)::float / COUNT(*) * 100) as score
        FROM orders o
        JOIN (
          SELECT order_id, SUM(price * quantity) as total
          FROM order_items
          GROUP BY order_id
        ) calculated ON calculated.order_id = o.id
        WHERE o.created_at > NOW() - INTERVAL '1 hour'
      `,
      threshold: 100,
      severity: "critical"
    },
    {
      name: "emails_are_valid",
      table: "users",
      query: `
        SELECT (COUNT(CASE WHEN email ~ '^[^@]+@[^@]+\.[^@]+$' THEN 1 END)::float / COUNT(*) * 100) as score
        FROM users
        WHERE created_at > NOW() - INTERVAL '1 day'
      `,
      threshold: 99,
      severity: "warning"
    }
  ];

  async runQualityChecks(): Promise<QualityReport> {
    const results: QualityCheckResult[] = [];

    for (const rule of this.rules) {
      const result = await this.runRule(rule);
      results.push(result);
      
      this.qualityScore.set(
        { rule: rule.name, table: rule.table },
        result.score
      );
      
      if (!result.passed) {
        this.logger.warn(
          `Data quality check FAILED: ${rule.name} (${result.score.toFixed(1)}% < ${rule.threshold}%)`
        );
      }
    }

    const overallScore = results.reduce((sum, r) => sum + r.score, 0) / results.length;

    return {
      timestamp: new Date(),
      overallScore,
      checks: results,
      allPassed: results.every(r => r.passed)
    };
  }

  private async runRule(rule: DataQualityRule): Promise<QualityCheckResult> {
    try {
      const result = await this.dataSource?.query(rule.query);
      const score = parseFloat(result?.[0]?.score ?? "0");
      
      return {
        rule: rule.name,
        table: rule.table,
        score,
        threshold: rule.threshold,
        passed: score >= rule.threshold,
        severity: rule.severity
      };
    } catch (error) {
      return {
        rule: rule.name,
        table: rule.table,
        score: 0,
        threshold: rule.threshold,
        passed: false,
        severity: rule.severity,
        error: (error as Error).message
      };
    }
  }

  private dataSource?: DataSource;
}

interface QualityCheckResult {
  rule: string;
  table: string;
  score: number;
  threshold: number;
  passed: boolean;
  severity: "critical" | "warning";
  error?: string;
}

interface QualityReport {
  timestamp: Date;
  overallScore: number;
  checks: QualityCheckResult[];
  allPassed: boolean;
}
```

---

## ขั้นตอนที่ 1350: Pipeline Error Recovery

```typescript
// src/pipelines/dead-letter-queue.ts

import { Injectable, Logger } from "@nestjs/common";
import { DataSource } from "typeorm";

interface FailedRecord {
  id: string;
  pipeline: string;
  data: any;
  error: string;
  attempts: number;
  lastAttempt: Date;
  createdAt: Date;
}

@Injectable()
export class PipelineDeadLetterQueue {
  private readonly logger = new Logger(PipelineDeadLetterQueue.name);

  constructor(private readonly dataSource: DataSource) {}

  async addToQueue(
    pipeline: string,
    recordId: string,
    data: any,
    error: string
  ): Promise<void> {
    await this.dataSource.query(`
      INSERT INTO pipeline_dlq (pipeline, record_id, data, error, attempts, created_at)
      VALUES ($1, $2, $3, $4, 1, NOW())
      ON CONFLICT (pipeline, record_id) DO UPDATE SET
        error = $4,
        attempts = pipeline_dlq.attempts + 1,
        last_attempt = NOW()
    `, [pipeline, recordId, JSON.stringify(data), error]);
  }

  async getRetryableRecords(
    pipeline: string,
    maxAttempts: number = 3
  ): Promise<FailedRecord[]> {
    return this.dataSource.query(`
      SELECT *
      FROM pipeline_dlq
      WHERE pipeline = $1
        AND attempts < $2
        AND last_attempt < NOW() - (INTERVAL '1 minute' * POWER(2, attempts))
      ORDER BY created_at ASC
      LIMIT 100
    `, [pipeline, maxAttempts]);
  }

  async markResolved(pipeline: string, recordId: string): Promise<void> {
    await this.dataSource.query(`
      DELETE FROM pipeline_dlq
      WHERE pipeline = $1 AND record_id = $2
    `, [pipeline, recordId]);
  }

  async getDLQStats(): Promise<any> {
    return this.dataSource.query(`
      SELECT 
        pipeline,
        COUNT(*) as total,
        AVG(attempts) as avg_attempts,
        MAX(attempts) as max_attempts,
        MIN(created_at) as oldest_record
      FROM pipeline_dlq
      GROUP BY pipeline
    `);
  }
}
```

---

## ขั้นตอนที่ 1351: Backpressure Handling

```typescript
// src/pipelines/backpressure.ts
// Handle backpressure in streaming pipelines

import { Transform, TransformCallback, TransformOptions } from "stream";

interface BackpressureOptions extends TransformOptions {
  maxQueueSize: number;
  processItem: (item: any) => Promise<any>;
  onHighWatermark?: () => void;
  onLowWatermark?: () => void;
}

export class BackpressureTransform extends Transform {
  private queue: any[] = [];
  private processing = false;
  private isHighWatermark = false;

  constructor(private readonly options: BackpressureOptions) {
    super({ objectMode: true });
  }

  _transform(chunk: any, encoding: string, callback: TransformCallback) {
    this.queue.push({ chunk, callback });

    // Check high watermark
    if (this.queue.length >= this.options.maxQueueSize && !this.isHighWatermark) {
      this.isHighWatermark = true;
      this.options.onHighWatermark?.();
    }

    if (!this.processing) {
      this.processQueue();
    }
  }

  private async processQueue() {
    if (this.processing || this.queue.length === 0) return;
    this.processing = true;

    while (this.queue.length > 0) {
      const { chunk, callback } = this.queue.shift()!;
      
      try {
        const result = await this.options.processItem(chunk);
        if (result !== null && result !== undefined) {
          this.push(result);
        }
        callback();
      } catch (error) {
        callback(error as Error);
        break;
      }

      // Check low watermark
      if (this.queue.length < this.options.maxQueueSize / 2 && this.isHighWatermark) {
        this.isHighWatermark = false;
        this.options.onLowWatermark?.();
      }
    }

    this.processing = false;
  }

  _flush(callback: TransformCallback) {
    if (this.queue.length === 0) {
      callback();
    } else {
      const check = setInterval(() => {
        if (this.queue.length === 0) {
          clearInterval(check);
          callback();
        }
      }, 100);
    }
  }
}
```

---

## ขั้นตอนที่ 1352: Data Pipeline Testing

```typescript
// src/pipelines/__tests__/order-analytics.pipeline.spec.ts

import { Test, TestingModule } from "@nestjs/testing";
import { OrderAnalyticsPipeline } from "../order-analytics.pipeline";
import { PipelineRecord } from "../base-pipeline";

describe("OrderAnalyticsPipeline", () => {
  let pipeline: OrderAnalyticsPipeline;
  let mockDataSource: any;

  beforeEach(async () => {
    mockDataSource = {
      query: jest.fn().mockResolvedValue([{ count: "0" }])
    };

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        OrderAnalyticsPipeline,
        { provide: "DataSource", useValue: mockDataSource }
      ]
    }).compile();

    pipeline = module.get<OrderAnalyticsPipeline>(OrderAnalyticsPipeline);
  });

  it("should process valid orders", async () => {
    const records: PipelineRecord[] = [
      {
        id: "r1",
        source: "orders",
        data: {
          id: "order-1",
          userId: "user-1",
          items: [{ productId: "p1", quantity: 2, price: 50 }],
          totalAmount: 100,
          status: "completed",
          createdAt: new Date().toISOString(),
          country: "TH"
        },
        timestamp: new Date()
      }
    ];

    const result = await pipeline.run(records);
    
    expect(result.processed).toBe(1);
    expect(result.succeeded).toBe(1);
    expect(result.failed).toBe(0);
    expect(mockDataSource.query).toHaveBeenCalled();
  });

  it("should skip invalid records", async () => {
    const records: PipelineRecord[] = [
      {
        id: "r1",
        source: "orders",
        data: {
          id: "order-1",
          userId: "user-1",
          items: [],
          totalAmount: -100,  // Invalid negative amount
          status: "completed",
          createdAt: new Date().toISOString(),
          country: "TH"
        },
        timestamp: new Date()
      }
    ];

    const result = await pipeline.run(records);
    
    expect(result.processed).toBe(1);
    expect(result.skipped).toBe(1);
    expect(result.succeeded).toBe(0);
  });
});
```

---

## ขั้นตอนที่ 1353: Monitoring Pipeline Health

```typescript
// src/pipelines/pipeline-monitor.service.ts

import { Injectable, Logger } from "@nestjs/common";
import * as promClient from "prom-client";

@Injectable()
export class PipelineMonitorService {
  private readonly logger = new Logger(PipelineMonitorService.name);
  
  private readonly pipelineLag = new promClient.Gauge({
    name: "pipeline_lag_seconds",
    help: "Pipeline processing lag in seconds",
    labelNames: ["pipeline"]
  });

  private readonly pipelineThroughput = new promClient.Gauge({
    name: "pipeline_throughput_records_per_second",
    help: "Pipeline processing throughput",
    labelNames: ["pipeline"]
  });

  private readonly lastSuccessfulRun = new promClient.Gauge({
    name: "pipeline_last_successful_run_timestamp",
    help: "Timestamp of last successful pipeline run",
    labelNames: ["pipeline"]
  });

  recordSuccessfulRun(
    pipelineName: string,
    recordsProcessed: number,
    durationMs: number
  ) {
    const throughput = recordsProcessed / (durationMs / 1000);
    
    this.pipelineThroughput.set({ pipeline: pipelineName }, throughput);
    this.lastSuccessfulRun.set({ pipeline: pipelineName }, Date.now() / 1000);
    
    this.logger.log(
      `Pipeline ${pipelineName}: ${throughput.toFixed(1)} records/sec`
    );
  }

  updateLag(pipelineName: string, oldestUnprocessedTimestamp: Date) {
    const lagSeconds = (Date.now() - oldestUnprocessedTimestamp.getTime()) / 1000;
    this.pipelineLag.set({ pipeline: pipelineName }, lagSeconds);

    if (lagSeconds > 3600) {
      this.logger.warn(
        `Pipeline ${pipelineName} is ${(lagSeconds / 3600).toFixed(1)} hours behind!`
      );
    }
  }
}
```

---

## ขั้นตอนที่ 1354: Data Lineage Tracking

```typescript
// src/lineage/data-lineage.service.ts
// Track where data comes from and where it goes

export interface LineageNode {
  id: string;
  type: "source" | "transformation" | "destination";
  name: string;
  description: string;
  schema?: Record<string, string>;
}

export interface LineageEdge {
  from: string;
  to: string;
  transformation?: string;
  fields?: Record<string, string>;  // Source field → Target field
}

export class DataLineageTracker {
  private nodes: Map<string, LineageNode> = new Map();
  private edges: LineageEdge[] = [];

  addNode(node: LineageNode): void {
    this.nodes.set(node.id, node);
  }

  addEdge(edge: LineageEdge): void {
    this.edges.push(edge);
  }

  getLineageForField(destinationNodeId: string, field: string): string[] {
    const lineage: string[] = [];
    const visited = new Set<string>();
    
    const trace = (nodeId: string, fieldName: string) => {
      if (visited.has(nodeId)) return;
      visited.add(nodeId);
      
      const node = this.nodes.get(nodeId);
      if (!node) return;
      
      lineage.unshift(`${node.name}.${fieldName}`);
      
      const sourceEdges = this.edges.filter(e => e.to === nodeId);
      for (const edge of sourceEdges) {
        const sourceField = edge.fields?.[fieldName];
        if (sourceField) {
          trace(edge.from, sourceField);
        }
      }
    };
    
    trace(destinationNodeId, field);
    return lineage;
  }
}
```

---

## ขั้นตอนที่ 1355: Pipeline Configuration

```typescript
// src/pipelines/pipeline-config.ts

import { z } from "zod";

const PipelineConfigSchema = z.object({
  name: z.string(),
  enabled: z.boolean().default(true),
  schedule: z.string().optional(),
  source: z.object({
    type: z.enum(["postgres", "kafka", "s3", "api"]),
    config: z.record(z.unknown())
  }),
  transformations: z.array(z.object({
    name: z.string(),
    type: z.enum(["filter", "map", "enrich", "aggregate", "validate"]),
    config: z.record(z.unknown()).optional()
  })),
  destination: z.object({
    type: z.enum(["postgres", "kafka", "s3", "elasticsearch"]),
    config: z.record(z.unknown())
  }),
  errorHandling: z.object({
    strategy: z.enum(["skip", "retry", "dlq", "fail"]).default("dlq"),
    maxRetries: z.number().default(3),
    retryDelayMs: z.number().default(1000)
  }).default({}),
  monitoring: z.object({
    alertOnLagMinutes: z.number().default(60),
    alertOnErrorRate: z.number().default(0.01)
  }).default({})
});

export type PipelineConfig = z.infer<typeof PipelineConfigSchema>;

// Load from environment or file
export function loadPipelineConfig(name: string): PipelineConfig {
  const raw = {
    name,
    source: {
      type: "postgres",
      config: {
        query: `SELECT * FROM orders WHERE DATE(created_at) = CURRENT_DATE - 1`
      }
    },
    transformations: [
      { name: "validate", type: "validate" },
      { name: "enrich", type: "enrich" },
      { name: "aggregate", type: "aggregate" }
    ],
    destination: {
      type: "postgres",
      config: {
        table: "order_analytics"
      }
    }
  };

  return PipelineConfigSchema.parse(raw);
}
```

---

## ขั้นตอนที่ 1356: Data Transformation Functions

```typescript
// src/pipelines/transformations/index.ts

export type TransformFn<T, R> = (data: T) => R | Promise<R>;

export const transformations = {
  // Flatten nested objects
  flatten(data: Record<string, any>, prefix = ""): Record<string, any> {
    return Object.entries(data).reduce((result, [key, value]) => {
      const fullKey = prefix ? `${prefix}_${key}` : key;
      
      if (typeof value === "object" && value !== null && !Array.isArray(value) && !(value instanceof Date)) {
        Object.assign(result, this.flatten(value, fullKey));
      } else {
        result[fullKey] = value;
      }
      
      return result;
    }, {} as Record<string, any>);
  },

  // Parse dates in multiple formats
  parseDate(value: string | Date): Date {
    if (value instanceof Date) return value;
    const date = new Date(value);
    if (isNaN(date.getTime())) throw new Error(`Invalid date: ${value}`);
    return date;
  },

  // Convert currency
  convertCurrency(amount: number, fromCurrency: string, toCurrency: string, rates: Record<string, number>): number {
    const fromRate = rates[fromCurrency] ?? 1;
    const toRate = rates[toCurrency] ?? 1;
    return amount * (toRate / fromRate);
  },

  // Anonymize PII
  anonymize(data: Record<string, any>, piiFields: string[]): Record<string, any> {
    return Object.entries(data).reduce((result, [key, value]) => {
      if (piiFields.includes(key)) {
        // Hash the value
        const crypto = require("crypto");
        result[key] = crypto.createHash("sha256").update(String(value)).digest("hex").substring(0, 8);
      } else {
        result[key] = value;
      }
      return result;
    }, {} as Record<string, any>);
  },

  // Normalize text
  normalizeText(value: string): string {
    return value.trim().toLowerCase().replace(/\s+/g, " ");
  }
};
```

---

## ขั้นตอนที่ 1357: Export to Data Warehouse

```typescript
// src/warehousing/data-warehouse.service.ts
// Export data to BigQuery or Redshift

import { Injectable, Logger } from "@nestjs/common";

@Injectable()
export class DataWarehouseService {
  private readonly logger = new Logger(DataWarehouseService.name);

  async exportToS3(
    data: any[],
    s3Bucket: string,
    s3Key: string
  ): Promise<string> {
    // Convert to CSV
    if (data.length === 0) return "";
    
    const headers = Object.keys(data[0]);
    const csvLines = [
      headers.join(","),
      ...data.map(row =>
        headers.map(h => {
          const val = row[h];
          if (val === null || val === undefined) return "";
          if (typeof val === "string") return `"${val.replace(/"/g, '""')}"`;
          return String(val);
        }).join(",")
      )
    ];

    const csv = csvLines.join("\n");
    
    // Upload to S3
    const { S3Client, PutObjectCommand } = await import("@aws-sdk/client-s3");
    const s3 = new S3Client({ region: process.env.AWS_REGION ?? "us-east-1" });
    
    await s3.send(new PutObjectCommand({
      Bucket: s3Bucket,
      Key: s3Key,
      Body: csv,
      ContentType: "text/csv"
    }));

    this.logger.log(`Exported ${data.length} records to s3://${s3Bucket}/${s3Key}`);
    return `s3://${s3Bucket}/${s3Key}`;
  }
}
```

---

## ขั้นตอนที่ 1358: Real-time Aggregation

```typescript
// src/aggregation/realtime-aggregator.ts
// Real-time sliding window aggregations

export class SlidingWindowAggregator {
  private windows: Map<string, number[]> = new Map();
  private readonly windowSizeMs: number;

  constructor(windowSizeMs: number) {
    this.windowSizeMs = windowSizeMs;
  }

  addValue(key: string, value: number, timestamp: Date = new Date()) {
    if (!this.windows.has(key)) {
      this.windows.set(key, []);
    }
    
    const window = this.windows.get(key)!;
    const entry = timestamp.getTime() * 1000 + value;  // Pack time+value
    window.push(entry);
    
    // Evict old entries
    const cutoff = Date.now() - this.windowSizeMs;
    const start = window.findIndex(e => e / 1000 > cutoff);
    if (start > 0) window.splice(0, start);
  }

  getStats(key: string): WindowStats {
    const window = this.windows.get(key) ?? [];
    const values = window.map(e => e % 1000);  // Extract value
    
    if (values.length === 0) {
      return { count: 0, sum: 0, min: 0, max: 0, avg: 0 };
    }

    return {
      count: values.length,
      sum: values.reduce((a, b) => a + b, 0),
      min: Math.min(...values),
      max: Math.max(...values),
      avg: values.reduce((a, b) => a + b, 0) / values.length
    };
  }
}

interface WindowStats {
  count: number;
  sum: number;
  min: number;
  max: number;
  avg: number;
}
```

---

## ขั้นตอนที่ 1359: Pipeline API

```typescript
// src/pipelines/pipeline.controller.ts

import { Controller, Get, Post, Body, Param, Query } from "@nestjs/common";
import { PipelineScheduler } from "./pipeline-scheduler";
import { PipelineMonitorService } from "./pipeline-monitor.service";

@Controller("admin/pipelines")
export class PipelineController {
  constructor(
    private readonly scheduler: PipelineScheduler,
    private readonly monitor: PipelineMonitorService
  ) {}

  @Get()
  listPipelines() {
    return {
      pipelines: [
        { name: "daily_order_etl", schedule: "0 2 * * *", status: "active" },
        { name: "hourly_aggregation", schedule: "5 * * * *", status: "active" },
        { name: "user_analytics", schedule: "0 3 * * *", status: "active" }
      ]
    };
  }

  @Post(":name/trigger")
  async triggerPipeline(@Param("name") name: string) {
    if (name === "daily_order_etl") {
      await this.scheduler.runDailyETL();
      return { message: "Pipeline triggered successfully" };
    }
    return { error: `Pipeline not found: ${name}` };
  }

  @Get(":name/history")
  async getPipelineHistory(
    @Param("name") name: string,
    @Query("limit") limit = "10"
  ) {
    // Return recent pipeline run history
    return { runs: [] };
  }

  @Get(":name/stats")
  async getPipelineStats(@Param("name") name: string) {
    return {
      pipeline: name,
      lastRun: new Date(),
      successRate: 0.998,
      avgDurationMs: 45000
    };
  }
}
```

---

## ขั้นตอนที่ 1360: Complete Pipeline Module

```typescript
// src/pipelines/pipeline.module.ts

import { Module } from "@nestjs/common";
import { ScheduleModule } from "@nestjs/schedule";
import { TypeOrmModule } from "@nestjs/typeorm";
import { OrderAnalyticsPipeline } from "./order-analytics.pipeline";
import { PipelineScheduler } from "./pipeline-scheduler";
import { PipelineMonitorService } from "./pipeline-monitor.service";
import { PipelineDeadLetterQueue } from "./dead-letter-queue";
import { DataQualityService } from "../quality/data-quality.service";
import { PipelineController } from "./pipeline.controller";
import { StreamingPipeline } from "./streaming-pipeline";

@Module({
  imports: [
    ScheduleModule.forRoot(),
    TypeOrmModule.forFeature([])
  ],
  providers: [
    OrderAnalyticsPipeline,
    PipelineScheduler,
    PipelineMonitorService,
    PipelineDeadLetterQueue,
    DataQualityService,
    StreamingPipeline
  ],
  controllers: [PipelineController],
  exports: [
    PipelineMonitorService
  ]
})
export class PipelineModule {}
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: ETL Pipeline
1. สร้าง BasePipeline สำหรับ user analytics
2. implement extract, transform, load steps
3. ทดสอบ error handling

### แบบฝึกหัดที่ 2: Streaming
1. ตั้งค่า Kafka consumer
2. implement real-time order processing
3. Monitor consumer lag

### แบบฝึกหัดที่ 3: Data Quality
1. สร้าง DataQualityService
2. กำหนด quality rules
3. สร้าง dashboard แสดง quality metrics

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- ETL pipeline architecture
- BasePipeline pattern
- Streaming data pipelines
- Data validation
- Change Data Capture (CDC)
- Pipeline error recovery
- Dead letter queue
- Data quality monitoring
- Data lineage tracking
- Real-time aggregations

**Part ถัดไป**: Machine Learning Integration
