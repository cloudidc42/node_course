# Part 70 | ขั้นตอนที่ 1201-1220 จาก 1000+

# Distributed Tracing

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. เข้าใจ Distributed Tracing concepts
2. ใช้ OpenTelemetry กับ Node.js
3. ตั้งค่า Jaeger สำหรับ trace visualization
4. implement Correlation IDs
5. trace requests ข้าม microservices
6. วิเคราะห์ performance bottlenecks

---

## ขั้นตอนที่ 1201: Distributed Tracing คืออะไร?

ใน microservices architecture, request หนึ่งอาจผ่านหลาย services distributed tracing ช่วยติดตาม request นั้นตลอด journey

```
Distributed Trace Example:
                                                    
User Request (100ms total)                          
│                                                   
├── API Gateway (5ms)                               
│                                                   
├── User Service (10ms)                             
│   ├── DB Query (8ms)                              
│   └── Cache Check (1ms)                           
│                                                   
├── Order Service (60ms)                            
│   ├── DB Query (20ms)                             
│   ├── Payment Service (30ms)                      
│   │   └── External API (25ms) ← Bottleneck!       
│   └── Inventory Check (8ms)                       
│                                                   
└── Notification Service (20ms)                     
    └── Email API (18ms)                            

Trace ID: abc123
Spans: 12 spans total
```

---

## ขั้นตอนที่ 1202: การติดตั้ง OpenTelemetry

```bash
# Core packages
npm install @opentelemetry/sdk-node
npm install @opentelemetry/auto-instrumentations-node
npm install @opentelemetry/exporter-jaeger
npm install @opentelemetry/exporter-otlp-grpc
npm install @opentelemetry/semantic-conventions

# Additional
npm install @opentelemetry/sdk-trace-base
npm install @opentelemetry/sdk-metrics
npm install @opentelemetry/resources
npm install @opentelemetry/instrumentation-http
npm install @opentelemetry/instrumentation-express
npm install @opentelemetry/instrumentation-pg
npm install @opentelemetry/instrumentation-redis
npm install @opentelemetry/instrumentation-mongodb

# Run Jaeger
docker run -d \
  --name jaeger \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 14250:14250 \
  jaegertracing/all-in-one:1.50
  
# Jaeger UI: http://localhost:16686
```

---

## ขั้นตอนที่ 1203: OpenTelemetry Setup

```typescript
// src/tracing/tracer.ts
// Must be loaded BEFORE other imports
import { NodeSDK } from "@opentelemetry/sdk-node";
import { Resource } from "@opentelemetry/resources";
import { SemanticResourceAttributes } from "@opentelemetry/semantic-conventions";
import { JaegerExporter } from "@opentelemetry/exporter-jaeger";
import { OTLPTraceExporter } from "@opentelemetry/exporter-otlp-grpc";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { BatchSpanProcessor } from "@opentelemetry/sdk-trace-base";
import { PeriodicExportingMetricReader } from "@opentelemetry/sdk-metrics";
import { PrometheusExporter } from "@opentelemetry/exporter-prometheus";

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: process.env.SERVICE_NAME ?? "my-service",
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.npm_package_version ?? "1.0.0",
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV ?? "development",
    "service.team": "backend",
    "service.owner": "engineering"
  }),
  
  // Trace exporter
  spanProcessor: new BatchSpanProcessor(
    new JaegerExporter({
      endpoint: process.env.JAEGER_ENDPOINT ?? "http://localhost:14268/api/traces"
    })
  ),
  
  // Metrics exporter
  metricReader: new PeriodicExportingMetricReader({
    exporter: new PrometheusExporter({ port: 9464 }),
    exportIntervalMillis: 10000
  }),
  
  // Auto-instrumentation
  instrumentations: [
    getNodeAutoInstrumentations({
      "@opentelemetry/instrumentation-fs": { enabled: false },
      "@opentelemetry/instrumentation-http": {
        requestHook: (span, request) => {
          span.setAttribute("http.request.body.size", 
            request.headers["content-length"] ?? 0);
        }
      },
      "@opentelemetry/instrumentation-express": {
        requestHook: (span, info) => {
          span.setAttribute("http.route", info.route);
        }
      }
    })
  ]
});

sdk.start();
console.log("OpenTelemetry SDK started");

process.on("SIGTERM", async () => {
  await sdk.shutdown();
});

export default sdk;
```

```typescript
// src/main.ts - Load tracing FIRST
import "./tracing/tracer";
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}

bootstrap();
```

---

## ขั้นตอนที่ 1204: Custom Spans

```typescript
// src/tracing/tracing.service.ts
import { Injectable } from "@nestjs/common";
import {
  trace,
  context,
  SpanStatusCode,
  SpanKind,
  Attributes,
  Span
} from "@opentelemetry/api";

@Injectable()
export class TracingService {
  private readonly tracer = trace.getTracer("my-service", "1.0.0");

  // Create a span for an operation
  async trace<T>(
    spanName: string,
    attributes: Attributes,
    operation: (span: Span) => Promise<T>
  ): Promise<T> {
    return this.tracer.startActiveSpan(
      spanName,
      { kind: SpanKind.INTERNAL, attributes },
      async (span) => {
        try {
          const result = await operation(span);
          span.setStatus({ code: SpanStatusCode.OK });
          return result;
        } catch (error) {
          span.setStatus({
            code: SpanStatusCode.ERROR,
            message: (error as Error).message
          });
          span.recordException(error as Error);
          throw error;
        } finally {
          span.end();
        }
      }
    );
  }

  // Add attributes to current span
  setAttribute(key: string, value: string | number | boolean): void {
    const span = trace.getActiveSpan();
    if (span) {
      span.setAttribute(key, value);
    }
  }

  // Add event to current span
  addEvent(name: string, attributes?: Attributes): void {
    const span = trace.getActiveSpan();
    if (span) {
      span.addEvent(name, attributes);
    }
  }

  // Get current trace context for propagation
  getCurrentTraceId(): string | undefined {
    const span = trace.getActiveSpan();
    return span?.spanContext().traceId;
  }

  // Create a child span
  createChildSpan(name: string, attributes?: Attributes): Span {
    return this.tracer.startSpan(name, {
      kind: SpanKind.INTERNAL,
      attributes
    });
  }
}
```

---

## ขั้นตอนที่ 1205: Correlation IDs

```typescript
// src/common/middleware/correlation.middleware.ts
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";
import { v4 as uuidv4 } from "uuid";
import { context, propagation, trace } from "@opentelemetry/api";

@Injectable()
export class CorrelationMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // Use existing correlation ID or create new one
    const correlationId = 
      req.headers["x-correlation-id"] as string ?? 
      req.headers["x-request-id"] as string ?? 
      uuidv4();
    
    // Set on request
    req.headers["x-correlation-id"] = correlationId;
    
    // Set on response
    res.setHeader("x-correlation-id", correlationId);
    
    // Add to active span
    const activeSpan = trace.getActiveSpan();
    if (activeSpan) {
      activeSpan.setAttribute("correlation.id", correlationId);
      activeSpan.setAttribute("http.request.id", correlationId);
    }
    
    next();
  }
}
```

```typescript
// src/common/context/async-context.ts
import { AsyncLocalStorage } from "async_hooks";

interface RequestContext {
  correlationId: string;
  userId?: string;
  sessionId?: string;
  traceId?: string;
}

class AsyncContextService {
  private storage = new AsyncLocalStorage<RequestContext>();

  run<T>(context: RequestContext, fn: () => T): T {
    return this.storage.run(context, fn);
  }

  get(): RequestContext | undefined {
    return this.storage.getStore();
  }

  getCorrelationId(): string | undefined {
    return this.storage.getStore()?.correlationId;
  }

  set(key: keyof RequestContext, value: string): void {
    const store = this.storage.getStore();
    if (store) {
      (store as any)[key] = value;
    }
  }
}

export const asyncContext = new AsyncContextService();
```

---

## ขั้นตอนที่ 1206: Trace Propagation across services

```typescript
// src/http/traced-http.service.ts
import { Injectable } from "@nestjs/common";
import { HttpService } from "@nestjs/axios";
import { context, propagation } from "@opentelemetry/api";
import { W3CTraceContextPropagator } from "@opentelemetry/core";

@Injectable()
export class TracedHttpService {
  constructor(private readonly httpService: HttpService) {}

  async get<T>(url: string, headers?: Record<string, string>): Promise<T> {
    // Inject trace context into headers
    const carrier: Record<string, string> = {};
    propagation.inject(context.active(), carrier);
    
    const response = await this.httpService.axiosRef.get<T>(url, {
      headers: {
        ...headers,
        ...carrier  // Includes traceparent, tracestate headers
      }
    });
    
    return response.data;
  }

  async post<T>(url: string, data: any, headers?: Record<string, string>): Promise<T> {
    const carrier: Record<string, string> = {};
    propagation.inject(context.active(), carrier);
    
    const response = await this.httpService.axiosRef.post<T>(url, data, {
      headers: { ...headers, ...carrier }
    });
    
    return response.data;
  }
}
```

---

## ขั้นตอนที่ 1207: Database Tracing

```typescript
// src/tracing/db-tracer.ts
import { PrismaClient } from "@prisma/client";
import { trace, SpanStatusCode, SpanKind } from "@opentelemetry/api";

export function withTracing(prisma: PrismaClient): PrismaClient {
  const tracer = trace.getTracer("prisma");
  
  return prisma.$extends({
    query: {
      $allModels: {
        async $allOperations({ operation, model, args, query }) {
          const span = tracer.startSpan(`db.${model}.${operation}`, {
            kind: SpanKind.CLIENT,
            attributes: {
              "db.system": "postgresql",
              "db.operation": operation,
              "db.model": model,
              "db.statement": JSON.stringify(args)
            }
          });
          
          try {
            const result = await query(args);
            span.setStatus({ code: SpanStatusCode.OK });
            return result;
          } catch (error) {
            span.setStatus({
              code: SpanStatusCode.ERROR,
              message: (error as Error).message
            });
            span.recordException(error as Error);
            throw error;
          } finally {
            span.end();
          }
        }
      }
    }
  }) as PrismaClient;
}
```

---

## ขั้นตอนที่ 1208: NestJS Interceptor สำหรับ Tracing

```typescript
// src/tracing/tracing.interceptor.ts
import {
  Injectable, NestInterceptor, ExecutionContext, CallHandler
} from "@nestjs/common";
import { Observable } from "rxjs";
import { tap } from "rxjs/operators";
import { trace, SpanStatusCode, SpanKind, context, propagation } from "@opentelemetry/api";
import { Request } from "express";

@Injectable()
export class TracingInterceptor implements NestInterceptor {
  private readonly tracer = trace.getTracer("http-server");

  intercept(ctx: ExecutionContext, next: CallHandler): Observable<any> {
    const request = ctx.switchToHttp().getRequest<Request>();
    const { method, url, headers } = request;
    
    // Extract parent context from incoming request
    const parentContext = propagation.extract(context.active(), headers);
    
    const span = this.tracer.startSpan(
      `${method} ${url}`,
      {
        kind: SpanKind.SERVER,
        attributes: {
          "http.method": method,
          "http.url": url,
          "http.target": url,
          "http.user_agent": headers["user-agent"],
          "http.correlation_id": headers["x-correlation-id"]
        }
      },
      parentContext
    );
    
    return context.with(trace.setSpan(parentContext, span), () =>
      next.handle().pipe(
        tap({
          next: () => {
            const response = ctx.switchToHttp().getResponse();
            span.setAttribute("http.status_code", response.statusCode);
            span.setStatus({ code: SpanStatusCode.OK });
            span.end();
          },
          error: (error) => {
            span.recordException(error);
            span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
            span.end();
          }
        })
      )
    );
  }
}
```

---

## ขั้นตอนที่ 1209: Custom Instrumentations

```typescript
// src/tracing/instrumentations.ts

// Instrument Redis operations
import { createClient } from "redis";
import { trace, SpanKind, SpanStatusCode } from "@opentelemetry/api";

export function instrumentRedis(client: ReturnType<typeof createClient>) {
  const tracer = trace.getTracer("redis-client");
  const originalGet = client.get.bind(client);
  const originalSet = client.set.bind(client);
  
  client.get = async (key: string) => {
    const span = tracer.startSpan("redis.get", {
      kind: SpanKind.CLIENT,
      attributes: {
        "db.system": "redis",
        "db.operation": "GET",
        "db.redis.key": key
      }
    });
    
    try {
      const result = await originalGet(key);
      span.setAttribute("db.redis.hit", result !== null);
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.recordException(error as Error);
      span.setStatus({ code: SpanStatusCode.ERROR });
      throw error;
    } finally {
      span.end();
    }
  };
  
  return client;
}

// Instrument external API calls
export function instrumentAxios(instance: any) {
  const tracer = trace.getTracer("http-client");
  
  instance.interceptors.request.use((config: any) => {
    const span = tracer.startSpan(`HTTP ${config.method?.toUpperCase()} ${config.url}`, {
      kind: SpanKind.CLIENT
    });
    
    config._span = span;
    
    // Propagate trace context
    const carrier: Record<string, string> = {};
    propagation.inject(context.active(), carrier);
    config.headers = { ...config.headers, ...carrier };
    
    return config;
  });
  
  instance.interceptors.response.use(
    (response: any) => {
      response.config._span?.end();
      return response;
    },
    (error: any) => {
      error.config?._span?.recordException(error);
      error.config?._span?.end();
      throw error;
    }
  );
}

import { context, propagation } from "@opentelemetry/api";
```

---

## ขั้นตอนที่ 1210: Trace Context in Logs

```typescript
// src/logging/logger.service.ts
import { Injectable } from "@nestjs/common";
import { createLogger, format, transports } from "winston";
import { trace, context } from "@opentelemetry/api";

@Injectable()
export class LoggerService {
  private logger = createLogger({
    format: format.combine(
      format.timestamp(),
      format.errors({ stack: true }),
      format.json()
    ),
    transports: [
      new transports.Console(),
      new transports.File({ filename: "logs/app.log" })
    ]
  });

  private getTraceContext() {
    const span = trace.getActiveSpan();
    if (!span) return {};
    
    const ctx = span.spanContext();
    return {
      traceId: ctx.traceId,
      spanId: ctx.spanId,
      traceFlags: ctx.traceFlags
    };
  }

  info(message: string, meta?: object) {
    this.logger.info(message, {
      ...this.getTraceContext(),
      ...meta
    });
  }

  error(message: string, error?: Error, meta?: object) {
    this.logger.error(message, {
      ...this.getTraceContext(),
      error: error ? {
        message: error.message,
        stack: error.stack,
        name: error.name
      } : undefined,
      ...meta
    });
  }

  warn(message: string, meta?: object) {
    this.logger.warn(message, {
      ...this.getTraceContext(),
      ...meta
    });
  }

  debug(message: string, meta?: object) {
    this.logger.debug(message, {
      ...this.getTraceContext(),
      ...meta
    });
  }
}
```

---

## ขั้นตอนที่ 1211: Jaeger Configuration

```yaml
# jaeger-config.yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
spec:
  strategy: production
  collector:
    maxReplicas: 5
    resources:
      limits:
        cpu: 100m
        memory: 128Mi
  query:
    replicas: 2
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: https://elasticsearch:9200
    secretName: jaeger-secret
  sampling:
    options:
      default_strategy:
        type: probabilistic
        param: 0.1  # 10% sampling rate
      service_strategies:
        - service: payment-service
          type: probabilistic
          param: 1.0  # 100% for critical services
        - service: user-service
          type: ratelimiting
          param: 100  # 100 traces per second
```

---

## ขั้นตอนที่ 1212: Sampling Strategies

```typescript
// src/tracing/sampling.ts
import {
  Sampler,
  SamplingResult,
  SamplingDecision,
  SpanKind
} from "@opentelemetry/sdk-trace-base";

// Custom sampler
class CustomSampler implements Sampler {
  shouldSample(
    context: any,
    traceId: string,
    spanName: string,
    spanKind: SpanKind,
    attributes: any,
    links: any
  ): SamplingResult {
    // Always sample errors
    if (attributes["error"] === true) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }
    
    // Always sample slow operations
    if (attributes["duration_ms"] > 1000) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }
    
    // Sample health checks at 1%
    if (spanName.includes("/health")) {
      return Math.random() < 0.01
        ? { decision: SamplingDecision.RECORD_AND_SAMPLED }
        : { decision: SamplingDecision.NOT_RECORD };
    }
    
    // Default: sample 10%
    return Math.random() < 0.1
      ? { decision: SamplingDecision.RECORD_AND_SAMPLED }
      : { decision: SamplingDecision.NOT_RECORD };
  }
  
  toString(): string {
    return "CustomSampler";
  }
}
```

---

## ขั้นตอนที่ 1213: B3 Propagation สำหรับ Zipkin

```typescript
// src/tracing/zipkin.setup.ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { ZipkinExporter } from "@opentelemetry/exporter-zipkin";
import { B3Propagator } from "@opentelemetry/propagator-b3";
import { CompositePropagator, W3CTraceContextPropagator } from "@opentelemetry/core";

const sdk = new NodeSDK({
  traceExporter: new ZipkinExporter({
    url: process.env.ZIPKIN_URL ?? "http://localhost:9411/api/v2/spans"
  }),
  
  // Support multiple propagation formats
  textMapPropagator: new CompositePropagator({
    propagators: [
      new W3CTraceContextPropagator(),  // W3C format (traceparent)
      new B3Propagator()               // Zipkin B3 format
    ]
  })
});
```

---

## ขั้นตอนที่ 1214: Trace Analysis

```typescript
// src/tracing/analyzer.ts
// Analyze traces to find performance issues

interface TraceSpan {
  spanId: string;
  parentSpanId?: string;
  operationName: string;
  serviceName: string;
  startTime: number;
  duration: number;
  tags: Record<string, any>;
  logs: Array<{ timestamp: number; fields: Record<string, any> }>;
}

interface TraceAnalysis {
  totalDuration: number;
  criticalPath: TraceSpan[];
  bottlenecks: TraceSpan[];
  errors: TraceSpan[];
  parallelOps: TraceSpan[][];
}

function analyzeTrace(spans: TraceSpan[]): TraceAnalysis {
  const rootSpan = spans.find(s => !s.parentSpanId)!;
  const totalDuration = rootSpan.duration;
  
  // Find critical path (longest sequential path)
  const criticalPath = findCriticalPath(spans, rootSpan);
  
  // Find bottlenecks (spans > 20% of total duration)
  const bottlenecks = spans.filter(
    s => s.duration > totalDuration * 0.2 && s !== rootSpan
  );
  
  // Find errors
  const errors = spans.filter(s => s.tags.error === true);
  
  // Find parallel operations
  const parallelOps = findParallelOps(spans);
  
  return { totalDuration, criticalPath, bottlenecks, errors, parallelOps };
}

function findCriticalPath(spans: TraceSpan[], current: TraceSpan): TraceSpan[] {
  const children = spans.filter(s => s.parentSpanId === current.spanId);
  
  if (children.length === 0) return [current];
  
  const longestChild = children.reduce((longest, child) => {
    const childPath = findCriticalPath(spans, child);
    const longestPath = findCriticalPath(spans, longest);
    
    const childDuration = childPath.reduce((sum, s) => sum + s.duration, 0);
    const longestDuration = longestPath.reduce((sum, s) => sum + s.duration, 0);
    
    return childDuration > longestDuration ? child : longest;
  });
  
  return [current, ...findCriticalPath(spans, longestChild)];
}

function findParallelOps(spans: TraceSpan[]): TraceSpan[][] {
  const groups: TraceSpan[][] = [];
  const processed = new Set<string>();
  
  for (const span of spans) {
    if (processed.has(span.spanId)) continue;
    
    // Find spans that overlap with this one
    const parallel = spans.filter(other =>
      other.spanId !== span.spanId &&
      other.parentSpanId === span.parentSpanId &&
      other.startTime < span.startTime + span.duration &&
      other.startTime + other.duration > span.startTime
    );
    
    if (parallel.length > 0) {
      groups.push([span, ...parallel]);
      parallel.forEach(s => processed.add(s.spanId));
    }
    
    processed.add(span.spanId);
  }
  
  return groups;
}
```

---

## ขั้นตอนที่ 1215: Alerting บน Traces

```typescript
// src/tracing/alerting.service.ts
import { Injectable } from "@nestjs/common";
import { HttpService } from "@nestjs/axios";

interface TraceAlert {
  type: "high_latency" | "error_spike" | "timeout";
  service: string;
  operation: string;
  value: number;
  threshold: number;
  traceId: string;
}

@Injectable()
export class TraceAlertingService {
  private readonly thresholds = {
    latency: {
      warning: 500,   // 500ms
      critical: 2000  // 2s
    },
    errorRate: {
      warning: 0.01,  // 1%
      critical: 0.05  // 5%
    }
  };

  private recentTraces: Array<{
    service: string;
    operation: string;
    duration: number;
    hasError: boolean;
    timestamp: Date;
  }> = [];

  recordTrace(trace: typeof this.recentTraces[0]): void {
    this.recentTraces.push(trace);
    
    // Keep only last 5 minutes of traces
    const fiveMinutesAgo = new Date(Date.now() - 5 * 60 * 1000);
    this.recentTraces = this.recentTraces.filter(
      t => t.timestamp > fiveMinutesAgo
    );
    
    // Check for alerts
    this.checkLatency(trace);
    this.checkErrorRate(trace.service, trace.operation);
  }

  private checkLatency(trace: typeof this.recentTraces[0]): void {
    if (trace.duration > this.thresholds.latency.critical) {
      this.sendAlert({
        type: "high_latency",
        service: trace.service,
        operation: trace.operation,
        value: trace.duration,
        threshold: this.thresholds.latency.critical,
        traceId: ""
      });
    }
  }

  private checkErrorRate(service: string, operation: string): void {
    const recent = this.recentTraces.filter(
      t => t.service === service && t.operation === operation
    );
    
    if (recent.length < 10) return; // Need enough data
    
    const errorRate = recent.filter(t => t.hasError).length / recent.length;
    
    if (errorRate > this.thresholds.errorRate.critical) {
      this.sendAlert({
        type: "error_spike",
        service,
        operation,
        value: errorRate,
        threshold: this.thresholds.errorRate.critical,
        traceId: ""
      });
    }
  }

  private async sendAlert(alert: TraceAlert): Promise<void> {
    console.error("TRACE ALERT:", alert);
    // Send to PagerDuty, Slack, etc.
  }
}
```

---

## ขั้นตอนที่ 1216: Service Map Visualization

```typescript
// src/tracing/service-map.service.ts
@Injectable()
export class ServiceMapService {
  private edges = new Map<string, Set<string>>();
  private callCounts = new Map<string, number>();
  private errorCounts = new Map<string, number>();
  private latencies = new Map<string, number[]>();

  recordCall(
    from: string,
    to: string,
    duration: number,
    hasError: boolean
  ): void {
    const key = `${from}->${to}`;
    
    // Track edges
    if (!this.edges.has(from)) this.edges.set(from, new Set());
    this.edges.get(from)!.add(to);
    
    // Track call counts
    this.callCounts.set(key, (this.callCounts.get(key) ?? 0) + 1);
    
    // Track errors
    if (hasError) {
      this.errorCounts.set(key, (this.errorCounts.get(key) ?? 0) + 1);
    }
    
    // Track latencies
    if (!this.latencies.has(key)) this.latencies.set(key, []);
    this.latencies.get(key)!.push(duration);
  }

  getServiceMap() {
    const nodes: string[] = [];
    const links: Array<{
      source: string;
      target: string;
      callCount: number;
      errorRate: number;
      avgLatency: number;
    }> = [];
    
    for (const [from, tos] of this.edges.entries()) {
      if (!nodes.includes(from)) nodes.push(from);
      
      for (const to of tos) {
        if (!nodes.includes(to)) nodes.push(to);
        
        const key = `${from}->${to}`;
        const calls = this.callCounts.get(key) ?? 0;
        const errors = this.errorCounts.get(key) ?? 0;
        const durations = this.latencies.get(key) ?? [];
        
        links.push({
          source: from,
          target: to,
          callCount: calls,
          errorRate: calls > 0 ? errors / calls : 0,
          avgLatency: durations.length > 0
            ? durations.reduce((a, b) => a + b, 0) / durations.length
            : 0
        });
      }
    }
    
    return { nodes, links };
  }
}
```

---

## ขั้นตอนที่ 1217: OpenTelemetry Collector

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  
  memory_limiter:
    limit_mib: 1500
  
  # Add environment info
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
    endpoint: 0.0.0.0:8889
  
  logging:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource]
      exporters: [jaeger, logging]
    
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

---

## ขั้นตอนที่ 1218: Trace-based Testing

```typescript
// src/testing/trace-test.helper.ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { InMemorySpanExporter, SimpleSpanProcessor } from "@opentelemetry/sdk-trace-base";

export class TraceTestHelper {
  private exporter = new InMemorySpanExporter();
  
  setup() {
    const sdk = new NodeSDK({
      spanProcessor: new SimpleSpanProcessor(this.exporter)
    });
    sdk.start();
    return sdk;
  }

  getSpans() {
    return this.exporter.getFinishedSpans();
  }

  getSpanByName(name: string) {
    return this.exporter.getFinishedSpans().find(s => s.name === name);
  }

  reset() {
    this.exporter.reset();
  }

  // Assert span attributes
  assertSpanAttributes(spanName: string, expectedAttributes: Record<string, any>) {
    const span = this.getSpanByName(spanName);
    if (!span) throw new Error(`Span "${spanName}" not found`);
    
    for (const [key, value] of Object.entries(expectedAttributes)) {
      const actual = span.attributes[key];
      if (actual !== value) {
        throw new Error(
          `Span "${spanName}" attribute "${key}": expected ${value}, got ${actual}`
        );
      }
    }
  }
}

// Usage in tests
describe("UsersService", () => {
  const traceHelper = new TraceTestHelper();
  
  beforeAll(() => traceHelper.setup());
  afterEach(() => traceHelper.reset());

  it("should create trace span for findById", async () => {
    await userService.findById("123");
    
    const span = traceHelper.getSpanByName("db.User.findUnique");
    expect(span).toBeDefined();
    expect(span?.attributes["db.operation"]).toBe("findUnique");
  });
});
```

---

## ขั้นตอนที่ 1219: Performance Profiling

```typescript
// src/tracing/profiler.ts
import { trace, SpanStatusCode } from "@opentelemetry/api";
import { performance } from "perf_hooks";

export class PerformanceProfiler {
  private tracer = trace.getTracer("profiler");
  
  // Profile a function and create a span
  async profile<T>(name: string, fn: () => Promise<T>): Promise<T> {
    const startTime = performance.now();
    
    return this.tracer.startActiveSpan(name, async (span) => {
      try {
        const result = await fn();
        const duration = performance.now() - startTime;
        
        span.setAttribute("duration.ms", Math.round(duration));
        span.setStatus({ code: SpanStatusCode.OK });
        
        return result;
      } catch (error) {
        const duration = performance.now() - startTime;
        span.setAttribute("duration.ms", Math.round(duration));
        span.recordException(error as Error);
        span.setStatus({ code: SpanStatusCode.ERROR });
        throw error;
      } finally {
        span.end();
      }
    });
  }
  
  // Measure N iterations
  async benchmark(name: string, iterations: number, fn: () => Promise<void>) {
    const durations: number[] = [];
    
    for (let i = 0; i < iterations; i++) {
      const start = performance.now();
      await fn();
      durations.push(performance.now() - start);
    }
    
    const avg = durations.reduce((a, b) => a + b, 0) / iterations;
    const min = Math.min(...durations);
    const max = Math.max(...durations);
    const p99 = durations.sort((a, b) => a - b)[Math.floor(iterations * 0.99)];
    
    console.log(`${name}: avg=${avg.toFixed(2)}ms min=${min.toFixed(2)}ms max=${max.toFixed(2)}ms p99=${p99.toFixed(2)}ms`);
    
    return { avg, min, max, p99 };
  }
}
```

---

## ขั้นตอนที่ 1220: Distributed Tracing Dashboard

```typescript
// src/tracing/dashboard.controller.ts
import { Controller, Get, Query, UseGuards } from "@nestjs/common";
import { JwtAuthGuard } from "../auth/guards/jwt-auth.guard";
import { ServiceMapService } from "./service-map.service";

@Controller("tracing")
@UseGuards(JwtAuthGuard)
export class TracingDashboardController {
  constructor(
    private readonly serviceMapService: ServiceMapService
  ) {}

  @Get("service-map")
  getServiceMap() {
    return this.serviceMapService.getServiceMap();
  }

  @Get("traces")
  async getTraces(
    @Query("service") service?: string,
    @Query("operation") operation?: string,
    @Query("minDuration") minDuration?: number,
    @Query("hasError") hasError?: boolean,
    @Query("limit") limit: number = 20
  ) {
    // Query from Jaeger API
    const jaegerUrl = process.env.JAEGER_QUERY_URL ?? "http://localhost:16686";
    
    const params = new URLSearchParams({
      limit: String(limit),
      ...(service && { service }),
      ...(operation && { operation }),
      ...(minDuration && { minDuration: String(minDuration * 1000) }),
      ...(hasError !== undefined && { tags: `error=${hasError}` })
    });
    
    const response = await fetch(`${jaegerUrl}/api/traces?${params}`);
    return response.json();
  }
}
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic Tracing
1. ติดตั้ง OpenTelemetry ใน Express application
2. เพิ่ม custom spans สำหรับ database operations
3. ดู traces ใน Jaeger UI

### แบบฝึกหัดที่ 2: Cross-service Tracing
1. สร้าง 2 services
2. implement trace propagation ระหว่าง services
3. ดู distributed trace ใน Jaeger

### แบบฝึกหัดที่ 3: Performance Analysis
1. สร้าง load test สำหรับ API
2. วิเคราะห์ traces เพื่อหา bottlenecks
3. optimize ตาม findings

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- Distributed Tracing concepts
- OpenTelemetry SDK setup
- Custom spans และ attributes
- Correlation IDs
- Trace propagation ข้าม services
- Database และ HTTP client tracing
- Jaeger visualization
- Sampling strategies
- Performance profiling

**Part ถัดไป**: Feature Flags
