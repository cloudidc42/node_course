# Part 75 | ขั้นตอนที่ 1301-1320 จาก 1000+

# Observability

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. เข้าใจ 3 pillars of observability
2. implement structured logging
3. ตั้งค่า metrics collection
4. กำหนด SLO/SLA/SLI
5. สร้าง alerting strategy
6. ใช้ dashboards อย่างมีประสิทธิภาพ

---

## ขั้นตอนที่ 1301: Observability คืออะไร?

```
Three Pillars of Observability:

┌─────────────────────────────────────────────────────────┐
│                    OBSERVABILITY                        │
│                                                         │
│    Logs          Metrics          Traces                │
│  ┌───────┐    ┌──────────┐    ┌──────────┐             │
│  │ What  │    │  How     │    │   Why    │             │
│  │happened│   │much/many │    │ it's slow│             │
│  │       │    │          │    │          │             │
│  │ Text  │    │ Numbers  │    │ Spans    │             │
│  │Events │    │ Counts   │    │ Context  │             │
│  │Errors │    │ Rates    │    │ Flow     │             │
│  └───────┘    └──────────┘    └──────────┘             │
│                                                         │
│  Why it matters:                                        │
│  - Understand system behavior                           │
│  - Detect anomalies quickly                             │
│  - Debug production issues                              │
│  - Capacity planning                                    │
└─────────────────────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 1302: Structured Logging

```typescript
// src/logger/logger.service.ts

import { Injectable, LoggerService } from "@nestjs/common";
import pino from "pino";

export interface LogContext {
  requestId?: string;
  userId?: string;
  service?: string;
  [key: string]: any;
}

@Injectable()
export class StructuredLoggerService implements LoggerService {
  private readonly logger: pino.Logger;

  constructor() {
    this.logger = pino({
      level: process.env.LOG_LEVEL ?? "info",
      base: {
        service: process.env.SERVICE_NAME ?? "unknown",
        environment: process.env.NODE_ENV ?? "development",
        version: process.env.APP_VERSION ?? "0.0.0"
      },
      serializers: {
        err: pino.stdSerializers.err,
        req: pino.stdSerializers.req,
        res: pino.stdSerializers.res
      },
      formatters: {
        level: (label) => ({ level: label })
      },
      timestamp: () => `,"timestamp":"${new Date().toISOString()}"`
    });
  }

  log(message: string, context?: string | LogContext) {
    this.logger.info(this.formatContext(context), message);
  }

  error(message: string, trace?: string, context?: string | LogContext) {
    this.logger.error({ ...this.formatContext(context), trace }, message);
  }

  warn(message: string, context?: string | LogContext) {
    this.logger.warn(this.formatContext(context), message);
  }

  debug(message: string, context?: string | LogContext) {
    this.logger.debug(this.formatContext(context), message);
  }

  verbose(message: string, context?: string | LogContext) {
    this.logger.trace(this.formatContext(context), message);
  }

  // Business event logging
  event(eventName: string, data: Record<string, any>, context?: LogContext) {
    this.logger.info({
      event_name: eventName,
      event_data: data,
      ...this.formatContext(context)
    }, `Event: ${eventName}`);
  }

  // Performance logging
  performance(operation: string, durationMs: number, context?: LogContext) {
    this.logger.info({
      operation,
      duration_ms: durationMs,
      ...this.formatContext(context)
    }, `Performance: ${operation}`);
  }

  private formatContext(context?: string | LogContext): object {
    if (!context) return {};
    if (typeof context === "string") return { context };
    return context;
  }
}
```

---

## ขั้นตอนที่ 1303: Request Logging Middleware

```typescript
// src/middleware/request-logger.middleware.ts

import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";
import { StructuredLoggerService } from "../logger/logger.service";
import { v4 as uuidv4 } from "uuid";

@Injectable()
export class RequestLoggerMiddleware implements NestMiddleware {
  constructor(private readonly logger: StructuredLoggerService) {}

  use(req: Request, res: Response, next: NextFunction) {
    const startTime = Date.now();
    const requestId = req.headers["x-request-id"] as string ?? uuidv4();
    
    // Attach request ID to request object
    (req as any).requestId = requestId;
    res.setHeader("x-request-id", requestId);

    // Log incoming request
    this.logger.log("Incoming request", {
      requestId,
      method: req.method,
      path: req.path,
      query: req.query,
      userAgent: req.headers["user-agent"],
      ip: req.ip,
      userId: (req as any).user?.id
    });

    // Log response when finished
    res.on("finish", () => {
      const duration = Date.now() - startTime;
      const logLevel = res.statusCode >= 500 ? "error" : 
                       res.statusCode >= 400 ? "warn" : "log";

      this.logger[logLevel]("Request completed", {
        requestId,
        method: req.method,
        path: req.path,
        statusCode: res.statusCode,
        duration_ms: duration,
        contentLength: res.getHeader("content-length"),
        userId: (req as any).user?.id
      });

      // Alert on slow requests
      if (duration > 2000) {
        this.logger.warn("Slow request detected", {
          requestId,
          path: req.path,
          duration_ms: duration
        });
      }
    });

    next();
  }
}
```

---

## ขั้นตอนที่ 1304: Metrics with Prometheus

```typescript
// src/metrics/metrics.service.ts

import { Injectable } from "@nestjs/common";
import * as promClient from "prom-client";

@Injectable()
export class MetricsService {
  // HTTP metrics
  private readonly httpRequestsTotal: promClient.Counter;
  private readonly httpRequestDuration: promClient.Histogram;
  private readonly httpRequestsInFlight: promClient.Gauge;

  // Business metrics
  private readonly ordersTotal: promClient.Counter;
  private readonly orderRevenue: promClient.Gauge;
  private readonly activeUsers: promClient.Gauge;

  // Database metrics
  private readonly dbQueryDuration: promClient.Histogram;
  private readonly dbConnectionsActive: promClient.Gauge;
  private readonly dbQueryErrors: promClient.Counter;

  // Cache metrics
  private readonly cacheHits: promClient.Counter;
  private readonly cacheMisses: promClient.Counter;

  constructor() {
    // HTTP metrics
    this.httpRequestsTotal = new promClient.Counter({
      name: "http_requests_total",
      help: "Total HTTP requests",
      labelNames: ["method", "path", "status_code"]
    });

    this.httpRequestDuration = new promClient.Histogram({
      name: "http_request_duration_ms",
      help: "HTTP request duration in milliseconds",
      labelNames: ["method", "path", "status_code"],
      buckets: [1, 5, 15, 50, 100, 200, 500, 1000, 2000, 5000]
    });

    this.httpRequestsInFlight = new promClient.Gauge({
      name: "http_requests_in_flight",
      help: "HTTP requests currently being processed"
    });

    // Business metrics
    this.ordersTotal = new promClient.Counter({
      name: "orders_total",
      help: "Total orders processed",
      labelNames: ["status", "payment_method"]
    });

    this.orderRevenue = new promClient.Gauge({
      name: "order_revenue_total",
      help: "Total order revenue",
      labelNames: ["currency"]
    });

    this.activeUsers = new promClient.Gauge({
      name: "active_users",
      help: "Number of active users in last 5 minutes"
    });

    // Database metrics
    this.dbQueryDuration = new promClient.Histogram({
      name: "db_query_duration_ms",
      help: "Database query duration",
      labelNames: ["operation", "table"],
      buckets: [1, 5, 10, 50, 100, 500, 1000, 5000]
    });

    this.dbConnectionsActive = new promClient.Gauge({
      name: "db_connections_active",
      help: "Active database connections"
    });

    this.dbQueryErrors = new promClient.Counter({
      name: "db_query_errors_total",
      help: "Database query errors",
      labelNames: ["operation", "error_code"]
    });

    // Cache metrics
    this.cacheHits = new promClient.Counter({
      name: "cache_hits_total",
      help: "Cache hits",
      labelNames: ["cache_name"]
    });

    this.cacheMisses = new promClient.Counter({
      name: "cache_misses_total",
      help: "Cache misses",
      labelNames: ["cache_name"]
    });

    // Enable default metrics (CPU, memory, GC, etc.)
    promClient.collectDefaultMetrics({
      labels: {
        service: process.env.SERVICE_NAME ?? "unknown"
      }
    });
  }

  // HTTP methods
  recordHttpRequest(method: string, path: string, statusCode: number, durationMs: number) {
    this.httpRequestsTotal.inc({ method, path, status_code: statusCode.toString() });
    this.httpRequestDuration.observe({ method, path, status_code: statusCode.toString() }, durationMs);
  }

  incrementInFlight() { this.httpRequestsInFlight.inc(); }
  decrementInFlight() { this.httpRequestsInFlight.dec(); }

  // Business methods
  recordOrder(status: string, paymentMethod: string, revenue: number, currency: string) {
    this.ordersTotal.inc({ status, payment_method: paymentMethod });
    if (status === "completed") {
      this.orderRevenue.inc({ currency }, revenue);
    }
  }

  setActiveUsers(count: number) {
    this.activeUsers.set(count);
  }

  // Database methods
  recordDbQuery(operation: string, table: string, durationMs: number) {
    this.dbQueryDuration.observe({ operation, table }, durationMs);
  }

  recordDbError(operation: string, errorCode: string) {
    this.dbQueryErrors.inc({ operation, error_code: errorCode });
  }

  setDbConnections(count: number) {
    this.dbConnectionsActive.set(count);
  }

  // Cache methods
  recordCacheHit(cacheName: string) { this.cacheHits.inc({ cache_name: cacheName }); }
  recordCacheMiss(cacheName: string) { this.cacheMisses.inc({ cache_name: cacheName }); }

  // Get all metrics
  async getMetrics(): Promise<string> {
    return promClient.register.metrics();
  }
}
```

---

## ขั้นตอนที่ 1305: SLO/SLA/SLI Definitions

```typescript
// src/slo/slo-definitions.ts
// Service Level Objectives

export interface SLI {
  name: string;
  description: string;
  query: string;  // Prometheus query
  unit: string;
}

export interface SLO {
  name: string;
  description: string;
  sli: SLI;
  target: number;       // e.g., 0.999 = 99.9%
  window: string;       // e.g., "30d"
  alerting: SLOAlerting;
}

export interface SLOAlerting {
  burnRateAlert: BurnRateAlert[];
}

export interface BurnRateAlert {
  name: string;
  shortWindow: string;
  longWindow: string;
  burnRate: number;
  severity: "critical" | "warning";
  description: string;
}

export const sloDefinitions: SLO[] = [
  {
    name: "api_availability",
    description: "API endpoint availability",
    sli: {
      name: "availability",
      description: "Percentage of successful requests (non-5xx)",
      query: `
        sum(rate(http_requests_total{status_code!~"5.."}[5m])) 
        / 
        sum(rate(http_requests_total[5m]))
      `,
      unit: "ratio"
    },
    target: 0.999,  // 99.9% (43.2 minutes downtime/month)
    window: "30d",
    alerting: {
      burnRateAlert: [
        {
          name: "HighBurnRate1h",
          shortWindow: "5m",
          longWindow: "1h",
          burnRate: 14.4,  // Consuming 1h error budget in 5m
          severity: "critical",
          description: "Burning through error budget too fast"
        },
        {
          name: "MediumBurnRate6h",
          shortWindow: "30m",
          longWindow: "6h",
          burnRate: 6,
          severity: "warning",
          description: "Error budget consumption elevated"
        }
      ]
    }
  },
  
  {
    name: "api_latency",
    description: "API response time",
    sli: {
      name: "latency",
      description: "Percentage of requests under 200ms",
      query: `
        histogram_quantile(0.95, 
          sum(rate(http_request_duration_ms_bucket[5m])) by (le)
        )
      `,
      unit: "milliseconds"
    },
    target: 0.95,   // 95% of requests under 200ms
    window: "30d",
    alerting: {
      burnRateAlert: [
        {
          name: "LatencyHighBurnRate",
          shortWindow: "5m",
          longWindow: "1h",
          burnRate: 14.4,
          severity: "critical",
          description: "p95 latency SLO being violated rapidly"
        }
      ]
    }
  },

  {
    name: "order_success_rate",
    description: "Order completion success rate",
    sli: {
      name: "order_success",
      description: "Percentage of orders completed successfully",
      query: `
        sum(rate(orders_total{status="completed"}[5m]))
        /
        sum(rate(orders_total[5m]))
      `,
      unit: "ratio"
    },
    target: 0.999,  // 99.9%
    window: "30d",
    alerting: {
      burnRateAlert: [
        {
          name: "OrderSuccessRateCritical",
          shortWindow: "5m",
          longWindow: "1h",
          burnRate: 14.4,
          severity: "critical",
          description: "Order failure rate too high"
        }
      ]
    }
  }
];
```

---

## ขั้นตอนที่ 1306: Error Budget Calculation

```typescript
// src/slo/error-budget.ts

export class ErrorBudgetCalculator {
  
  calculateErrorBudget(sloTarget: number, windowDays: number): ErrorBudget {
    const totalMinutes = windowDays * 24 * 60;
    const errorBudgetPercentage = 1 - sloTarget;
    const allowedDowntimeMinutes = totalMinutes * errorBudgetPercentage;
    
    return {
      sloTarget,
      windowDays,
      totalMinutes,
      errorBudgetPercentage,
      allowedDowntimeMinutes,
      allowedDowntimeFormatted: this.formatDuration(allowedDowntimeMinutes)
    };
  }

  calculateRemainingBudget(
    sloTarget: number,
    windowDays: number,
    currentAvailability: number
  ): RemainingBudget {
    const totalErrorBudget = 1 - sloTarget;
    const usedErrorBudget = 1 - currentAvailability;
    const remainingBudget = totalErrorBudget - usedErrorBudget;
    const remainingPercentage = remainingBudget / totalErrorBudget;

    return {
      total: totalErrorBudget,
      used: usedErrorBudget,
      remaining: remainingBudget,
      remainingPercentage,
      status: remainingPercentage > 0.5 ? "healthy" :
              remainingPercentage > 0.25 ? "at_risk" : "critical"
    };
  }

  calculateBurnRate(
    currentErrorRate: number,
    sloTarget: number,
    windowDays: number
  ): number {
    const normalErrorRate = 1 - sloTarget;
    return currentErrorRate / normalErrorRate;
  }

  private formatDuration(minutes: number): string {
    if (minutes < 60) return `${minutes.toFixed(1)} minutes`;
    if (minutes < 1440) return `${(minutes / 60).toFixed(1)} hours`;
    return `${(minutes / 1440).toFixed(1)} days`;
  }
}

interface ErrorBudget {
  sloTarget: number;
  windowDays: number;
  totalMinutes: number;
  errorBudgetPercentage: number;
  allowedDowntimeMinutes: number;
  allowedDowntimeFormatted: string;
}

interface RemainingBudget {
  total: number;
  used: number;
  remaining: number;
  remainingPercentage: number;
  status: "healthy" | "at_risk" | "critical";
}
```

---

## ขั้นตอนที่ 1307: Alerting Rules

```yaml
# prometheus-alerts.yaml

groups:
  - name: availability
    interval: 1m
    rules:
      # Error rate > 1%
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) /
          sum(rate(http_requests_total[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} (>1%)"
          runbook_url: "https://wiki.example.com/runbooks/high-error-rate"
      
      # p95 latency > 500ms
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_ms_bucket[5m])) by (le)
          ) > 500
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "High p95 latency"
          description: "p95 latency is {{ $value }}ms"
      
      # SLO burn rate
      - alert: SLOBurnRateCritical
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m])) /
            sum(rate(http_requests_total[5m]))
          ) > (14.4 * 0.001)  # 14.4x burn rate, SLO = 99.9%
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "SLO burn rate critical"
          description: "Burning through error budget 14.4x faster than allowed"
      
  - name: business
    rules:
      # Order failure rate
      - alert: HighOrderFailureRate
        expr: |
          sum(rate(orders_total{status="failed"}[5m])) /
          sum(rate(orders_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
          team: business
        annotations:
          summary: "High order failure rate"
          description: "{{ $value | humanizePercentage }} of orders failing"
      
      # No orders (possible service down)
      - alert: NoOrders
        expr: sum(rate(orders_total[10m])) == 0
        for: 5m
        labels:
          severity: critical
          team: business
        annotations:
          summary: "No orders in last 10 minutes"
          description: "Order processing may be down"
```

---

## ขั้นตอนที่ 1308: Grafana Dashboards

```json
{
  "dashboard": {
    "title": "Service Overview",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (status_code)",
            "legendFormat": "{{status_code}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status_code=~'5..'}[5m])) / sum(rate(http_requests_total[5m]))",
            "legendFormat": "Error Rate"
          }
        ],
        "thresholds": [
          {"color": "green", "value": 0},
          {"color": "yellow", "value": 0.001},
          {"color": "red", "value": 0.01}
        ]
      },
      {
        "title": "Latency Percentiles",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_ms_bucket[5m])) by (le))",
            "legendFormat": "p50"
          },
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_ms_bucket[5m])) by (le))",
            "legendFormat": "p95"
          },
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_ms_bucket[5m])) by (le))",
            "legendFormat": "p99"
          }
        ]
      },
      {
        "title": "Error Budget Remaining",
        "type": "gauge",
        "targets": [
          {
            "expr": "1 - (sum(rate(http_requests_total{status_code=~'5..'}[30d])) / sum(rate(http_requests_total[30d]))) / 0.001",
            "legendFormat": "Error Budget"
          }
        ],
        "min": 0,
        "max": 1,
        "thresholds": [
          {"color": "red", "value": 0},
          {"color": "yellow", "value": 0.25},
          {"color": "green", "value": 0.5}
        ]
      }
    ]
  }
}
```

---

## ขั้นตอนที่ 1309: Log Aggregation with ELK

```yaml
# docker-compose.yml - ELK Stack
version: "3.8"

services:
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9200"]
      interval: 30s

  logstash:
    image: logstash:8.11.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - "5044:5044"  # Beats input
      - "5000:5000"  # TCP input (for apps)
    depends_on:
      - elasticsearch

  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

  filebeat:
    image: elastic/filebeat:8.11.0
    user: root
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - logstash

volumes:
  es_data:
```

```yaml
# filebeat.yml - Collect Docker logs
filebeat.inputs:
  - type: container
    paths:
      - "/var/lib/docker/containers/*/*.log"
    json.message_key: log
    json.keys_under_root: true
    
processors:
  - add_docker_metadata: ~
  - decode_json_fields:
      fields: ["message"]
      target: ""
      overwrite_keys: true

output.logstash:
  hosts: ["logstash:5044"]
```

---

## ขั้นตอนที่ 1310: Distributed Tracing Integration

```typescript
// src/tracing/tracing-interceptor.ts

import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from "@nestjs/common";
import { Observable } from "rxjs";
import { tap } from "rxjs/operators";
import { trace, context, SpanStatusCode } from "@opentelemetry/api";

@Injectable()
export class TracingInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<any> {
    const tracer = trace.getTracer("api");
    const request = ctx.switchToHttp().getRequest();
    
    const spanName = `${request.method} ${request.route?.path || request.path}`;
    
    const span = tracer.startSpan(spanName, {
      attributes: {
        "http.method": request.method,
        "http.url": request.url,
        "http.route": request.route?.path,
        "user.id": request.user?.id
      }
    });

    return context.with(trace.setSpan(context.active(), span), () => {
      return next.handle().pipe(
        tap({
          next: () => {
            span.setStatus({ code: SpanStatusCode.OK });
            span.end();
          },
          error: (error) => {
            span.setStatus({
              code: SpanStatusCode.ERROR,
              message: error.message
            });
            span.recordException(error);
            span.end();
          }
        })
      );
    });
  }
}
```

---

## ขั้นตอนที่ 1311: Health Check with Details

```typescript
// src/health/health.controller.ts

import { Controller, Get } from "@nestjs/common";
import { InjectDataSource } from "@nestjs/typeorm";
import { DataSource } from "typeorm";
import Redis from "ioredis";

export interface HealthStatus {
  status: "ok" | "degraded" | "down";
  version: string;
  uptime: number;
  timestamp: string;
  checks: {
    [key: string]: HealthCheck;
  };
}

interface HealthCheck {
  status: "ok" | "error";
  latencyMs?: number;
  details?: any;
  error?: string;
}

@Controller("health")
export class HealthController {
  private readonly startTime = Date.now();

  constructor(
    @InjectDataSource() private readonly dataSource: DataSource,
    private readonly redis: Redis
  ) {}

  @Get()
  async check(): Promise<HealthStatus> {
    const checks = await Promise.allSettled([
      this.checkDatabase(),
      this.checkRedis(),
      this.checkDiskSpace(),
      this.checkMemory()
    ]);

    const results = {
      database: this.extractResult(checks[0]),
      redis: this.extractResult(checks[1]),
      disk: this.extractResult(checks[2]),
      memory: this.extractResult(checks[3])
    };

    const allOk = Object.values(results).every(r => r.status === "ok");
    const anyDown = Object.values(results).some(r => r.status === "error");

    return {
      status: allOk ? "ok" : anyDown ? "down" : "degraded",
      version: process.env.APP_VERSION ?? "0.0.0",
      uptime: Date.now() - this.startTime,
      timestamp: new Date().toISOString(),
      checks: results
    };
  }

  private async checkDatabase(): Promise<HealthCheck> {
    const start = Date.now();
    await this.dataSource.query("SELECT 1");
    return { status: "ok", latencyMs: Date.now() - start };
  }

  private async checkRedis(): Promise<HealthCheck> {
    const start = Date.now();
    await this.redis.ping();
    return { status: "ok", latencyMs: Date.now() - start };
  }

  private async checkDiskSpace(): Promise<HealthCheck> {
    const { checkDiskSpace } = await import("check-disk-space");
    const disk = await checkDiskSpace("/");
    const usedPercent = (1 - disk.free / disk.size) * 100;
    
    return {
      status: usedPercent < 90 ? "ok" : "error",
      details: {
        total: disk.size,
        free: disk.free,
        used_percent: usedPercent.toFixed(1)
      }
    };
  }

  private checkMemory(): HealthCheck {
    const memUsage = process.memoryUsage();
    const heapUsedPercent = memUsage.heapUsed / memUsage.heapTotal * 100;
    
    return {
      status: heapUsedPercent < 85 ? "ok" : "error",
      details: {
        heap_used_mb: Math.round(memUsage.heapUsed / 1024 / 1024),
        heap_total_mb: Math.round(memUsage.heapTotal / 1024 / 1024),
        rss_mb: Math.round(memUsage.rss / 1024 / 1024),
        used_percent: heapUsedPercent.toFixed(1)
      }
    };
  }

  private extractResult(result: PromiseSettledResult<HealthCheck>): HealthCheck {
    if (result.status === "fulfilled") return result.value;
    return { status: "error", error: result.reason?.message };
  }
}
```

---

## ขั้นตอนที่ 1312: Alertmanager Configuration

```yaml
# alertmanager.yml

global:
  resolve_timeout: 5m
  slack_api_url: "https://hooks.slack.com/services/XXXXX"

route:
  group_by: ["alertname", "service"]
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 4h
  receiver: "default"
  
  routes:
    # Critical alerts go to PagerDuty
    - match:
        severity: critical
      receiver: pagerduty-critical
      continue: true
    
    # All alerts also go to Slack
    - match_re:
        severity: "critical|warning"
      receiver: slack-alerts
    
    # Business alerts to business team
    - match:
        team: business
      receiver: business-team-slack

receivers:
  - name: "default"
    slack_configs:
      - channel: "#alerts-all"
        send_resolved: true

  - name: "pagerduty-critical"
    pagerduty_configs:
      - service_key: "${PAGERDUTY_SERVICE_KEY}"
        severity: critical
        description: "{{ .GroupLabels.alertname }}: {{ range .Alerts }}{{ .Annotations.description }}{{ end }}"

  - name: "slack-alerts"
    slack_configs:
      - channel: "#alerts-platform"
        send_resolved: true
        title: "{{ .GroupLabels.alertname }}"
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Labels.alertname }}
          *Severity:* {{ .Labels.severity }}
          *Summary:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Runbook:* {{ .Annotations.runbook_url }}
          {{ end }}
        color: |
          {{ if eq .Status "firing" }}danger{{ else }}good{{ end }}

  - name: "business-team-slack"
    slack_configs:
      - channel: "#business-alerts"
        send_resolved: true

inhibit_rules:
  # If critical alert firing, inhibit warnings for same service
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ["alertname", "service"]
```

---

## ขั้นตอนที่ 1313: Custom Metrics Decorator

```typescript
// src/metrics/track-metrics.decorator.ts

import { MetricsService } from "./metrics.service";

export function TrackMetrics(options?: {
  operation?: string;
  alertOnSlowMs?: number;
}) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    const operation = options?.operation ?? `${target.constructor.name}.${propertyKey}`;

    descriptor.value = async function (...args: any[]) {
      const metrics: MetricsService = (this as any).metricsService;
      const startTime = Date.now();
      
      try {
        const result = await originalMethod.apply(this, args);
        
        const duration = Date.now() - startTime;
        if (metrics) {
          metrics.recordDbQuery(operation, "unknown", duration);
        }
        
        if (options?.alertOnSlowMs && duration > options.alertOnSlowMs) {
          console.warn(`Slow operation: ${operation} took ${duration}ms`);
        }
        
        return result;
      } catch (error) {
        const duration = Date.now() - startTime;
        if (metrics) {
          metrics.recordDbError(operation, (error as any).code ?? "unknown");
        }
        throw error;
      }
    };

    return descriptor;
  };
}

// Usage
class UserService {
  @TrackMetrics({ operation: "findUser", alertOnSlowMs: 500 })
  async findById(userId: string) {
    // This method will be automatically tracked
  }
}
```

---

## ขั้นตอนที่ 1314: Correlation IDs

```typescript
// src/context/request-context.ts

import { AsyncLocalStorage } from "async_hooks";
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";
import { v4 as uuidv4 } from "uuid";

export interface RequestContext {
  requestId: string;
  userId?: string;
  sessionId?: string;
  tenantId?: string;
  traceId?: string;
  spanId?: string;
}

export const requestContextStorage = new AsyncLocalStorage<RequestContext>();

@Injectable()
export class RequestContextMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const context: RequestContext = {
      requestId: req.headers["x-request-id"] as string ?? uuidv4(),
      userId: (req as any).user?.id,
      sessionId: req.headers["x-session-id"] as string,
      tenantId: req.headers["x-tenant-id"] as string,
      traceId: req.headers["x-trace-id"] as string
    };

    // Set response headers
    res.setHeader("x-request-id", context.requestId);

    // Run rest of request handling with this context
    requestContextStorage.run(context, () => {
      next();
    });
  }
}

// Helper to get current context
export function getCurrentContext(): RequestContext | undefined {
  return requestContextStorage.getStore();
}

// Logger that automatically includes request context
export class ContextualLogger {
  log(message: string, data?: any) {
    const ctx = getCurrentContext();
    console.log(JSON.stringify({
      level: "info",
      message,
      requestId: ctx?.requestId,
      userId: ctx?.userId,
      tenantId: ctx?.tenantId,
      traceId: ctx?.traceId,
      ...data,
      timestamp: new Date().toISOString()
    }));
  }
}
```

---

## ขั้นตอนที่ 1315: Observability for Microservices

```typescript
// src/observability/service-graph.ts
// Track service dependencies

import * as promClient from "prom-client";

const serviceCallsTotal = new promClient.Counter({
  name: "service_calls_total",
  help: "Total calls between services",
  labelNames: ["from_service", "to_service", "method", "status"]
});

const serviceCallDuration = new promClient.Histogram({
  name: "service_call_duration_ms",
  help: "Duration of service calls",
  labelNames: ["from_service", "to_service", "method"],
  buckets: [5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000]
});

export function trackServiceCall(
  fromService: string,
  toService: string
) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function (...args: any[]) {
      const startTime = Date.now();
      let status = "success";
      
      try {
        const result = await originalMethod.apply(this, args);
        return result;
      } catch (error) {
        status = "error";
        throw error;
      } finally {
        const duration = Date.now() - startTime;
        
        serviceCallsTotal.inc({
          from_service: fromService,
          to_service: toService,
          method: propertyKey,
          status
        });
        
        serviceCallDuration.observe({
          from_service: fromService,
          to_service: toService,
          method: propertyKey
        }, duration);
      }
    };
    
    return descriptor;
  };
}

// Usage
class OrderService {
  @trackServiceCall("order-service", "payment-service")
  async processPayment(orderId: string) {
    // This call is automatically tracked
  }
}
```

---

## ขั้นตอนที่ 1316: Prometheus Rules and Recording

```yaml
# recording-rules.yaml - Pre-compute expensive queries

groups:
  - name: recording_rules
    interval: 1m
    rules:
      # Pre-compute error rate (avoid computing for every dashboard load)
      - record: job:http_error_rate:rate5m
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (job) /
          sum(rate(http_requests_total[5m])) by (job)
      
      # Pre-compute p95 latency
      - record: job:http_p95_latency:rate5m
        expr: |
          histogram_quantile(0.95, 
            sum(rate(http_request_duration_ms_bucket[5m])) by (le, job)
          )
      
      # Pre-compute request rate
      - record: job:http_request_rate:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
      
      # Pre-compute error budget consumption
      - record: job:slo_error_budget_consumed:rate30d
        expr: |
          1 - (
            sum(rate(http_requests_total{status_code!~"5.."}[30d])) by (job) /
            sum(rate(http_requests_total[30d])) by (job)
          )
```

---

## ขั้นตอนที่ 1317: Incident Response Dashboard

```typescript
// src/incident/incident-detector.ts

import { Injectable } from "@nestjs/common";
import { StructuredLoggerService } from "../logger/logger.service";

interface IncidentSignal {
  type: string;
  severity: "critical" | "high" | "medium";
  message: string;
  metrics: Record<string, number>;
  timestamp: Date;
}

@Injectable()
export class IncidentDetector {
  private activeIncidents: Map<string, IncidentSignal> = new Map();

  constructor(private readonly logger: StructuredLoggerService) {}

  async evaluate(metrics: any): Promise<IncidentSignal[]> {
    const newIncidents: IncidentSignal[] = [];

    // Check error rate
    if (metrics.errorRate > 0.05) {
      const signal: IncidentSignal = {
        type: "HIGH_ERROR_RATE",
        severity: metrics.errorRate > 0.1 ? "critical" : "high",
        message: `Error rate ${(metrics.errorRate * 100).toFixed(1)}% exceeds threshold`,
        metrics: { errorRate: metrics.errorRate },
        timestamp: new Date()
      };
      newIncidents.push(signal);
      
      if (!this.activeIncidents.has("HIGH_ERROR_RATE")) {
        this.activeIncidents.set("HIGH_ERROR_RATE", signal);
        this.logger.error("INCIDENT DETECTED: High error rate", {
          incident: signal
        });
      }
    } else {
      this.activeIncidents.delete("HIGH_ERROR_RATE");
    }

    // Check latency
    if (metrics.p99LatencyMs > 2000) {
      const signal: IncidentSignal = {
        type: "HIGH_LATENCY",
        severity: "high",
        message: `p99 latency ${metrics.p99LatencyMs}ms exceeds 2000ms threshold`,
        metrics: { p99LatencyMs: metrics.p99LatencyMs },
        timestamp: new Date()
      };
      newIncidents.push(signal);
    }

    return newIncidents;
  }
}
```

---

## ขั้นตอนที่ 1318: OpenTelemetry Collector

```yaml
# otel-collector-config.yaml

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  
  prometheus:
    config:
      scrape_configs:
        - job_name: "node-services"
          scrape_interval: 15s
          static_configs:
            - targets: ["product-service:9090", "order-service:9090"]

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128
    check_interval: 5s
  
  # Add service resource attributes
  resource:
    attributes:
      - key: environment
        value: production
        action: upsert

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  
  prometheus:
    endpoint: "0.0.0.0:8889"
  
  logging:
    loglevel: debug

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource]
      exporters: [jaeger]
    
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
    
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [logging]
```

---

## ขั้นตอนที่ 1319: Service Level Agreement

```typescript
// src/slo/sla-calculator.ts
// Calculate whether SLA is being met

export interface SLAReport {
  period: string;
  slaTarget: number;  // 99.9%
  actualAvailability: number;
  withinSLA: boolean;
  downtimeMinutes: number;
  credits: SLACredit[];
}

export interface SLACredit {
  reason: string;
  period: string;
  creditPercentage: number;
}

export class SLACalculator {
  
  calculateSLA(
    uptimeMinutes: number,
    totalMinutes: number,
    slaTarget: number = 0.999
  ): SLAReport {
    const actualAvailability = uptimeMinutes / totalMinutes;
    const downtimeMinutes = totalMinutes - uptimeMinutes;
    const withinSLA = actualAvailability >= slaTarget;

    const credits = withinSLA ? [] : this.calculateCredits(actualAvailability);

    return {
      period: "monthly",
      slaTarget,
      actualAvailability,
      withinSLA,
      downtimeMinutes,
      credits
    };
  }

  private calculateCredits(actualAvailability: number): SLACredit[] {
    const credits: SLACredit[] = [];

    if (actualAvailability < 0.95) {
      credits.push({
        reason: "Availability below 95%",
        period: "monthly",
        creditPercentage: 100  // 100% service credit
      });
    } else if (actualAvailability < 0.99) {
      credits.push({
        reason: "Availability 95-99%",
        period: "monthly",
        creditPercentage: 25
      });
    } else if (actualAvailability < 0.999) {
      credits.push({
        reason: "Availability 99-99.9%",
        period: "monthly",
        creditPercentage: 10
      });
    }

    return credits;
  }
}
```

---

## ขั้นตอนที่ 1320: Full Observability Stack Setup

```bash
#!/bin/bash
# scripts/setup-observability.sh

# Deploy Prometheus
kubectl apply -f k8s/monitoring/prometheus/

# Deploy Grafana
kubectl apply -f k8s/monitoring/grafana/

# Deploy Jaeger for tracing
kubectl apply -f k8s/monitoring/jaeger/

# Deploy Loki for log aggregation
kubectl apply -f k8s/monitoring/loki/

# Deploy Alertmanager
kubectl apply -f k8s/monitoring/alertmanager/

# Deploy OpenTelemetry Collector
kubectl apply -f k8s/monitoring/otel-collector/

# Configure Prometheus service monitor for auto-discovery
kubectl apply -f k8s/monitoring/service-monitors/

# Import Grafana dashboards
for dashboard in grafana-dashboards/*.json; do
  curl -X POST http://admin:admin@localhost:3000/api/dashboards/import \
    -H "Content-Type: application/json" \
    -d @$dashboard
done

echo "Observability stack deployed successfully!"
echo ""
echo "Access URLs:"
echo "  Grafana: http://localhost:3000 (admin/admin)"
echo "  Prometheus: http://localhost:9090"
echo "  Jaeger: http://localhost:16686"
echo "  AlertManager: http://localhost:9093"
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: Structured Logging
1. implement StructuredLoggerService
2. เพิ่ม request ID ใน all logs
3. ทดสอบ log output format

### แบบฝึกหัดที่ 2: SLO Definition
1. กำหนด SLO สำหรับ API
2. คำนวณ error budget
3. สร้าง Prometheus alerts

### แบบฝึกหัดที่ 3: Dashboard
1. สร้าง Grafana dashboard
2. เพิ่ม panels สำหรับ key metrics
3. ตั้งค่า thresholds

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- Three pillars of observability
- Structured logging
- Metrics collection ด้วย Prometheus
- SLO/SLA/SLI definitions
- Error budget calculation
- Alerting rules
- Log aggregation ด้วย ELK stack
- Distributed tracing integration
- Full observability stack setup

**Part ถัดไป**: Cost Optimization
