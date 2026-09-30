# Part 68: Distributed Tracing
## ขั้นตอนที่ 671-680 จาก 1000

---

## Distributed Tracing คืออะไร?

ใน microservices architecture, request เดียวอาจผ่าน services หลายตัว Distributed Tracing ช่วย track เส้นทาง request ทั้งหมดผ่าน services ต่างๆ ทำให้ debug latency problems ง่ายขึ้น

---

## 1. OpenTelemetry

### ติดตั้ง

```bash
npm install @opentelemetry/api
npm install @opentelemetry/sdk-node
npm install @opentelemetry/auto-instrumentations-node
npm install @opentelemetry/exporter-jaeger
npm install @opentelemetry/exporter-otlp-http
```

### Setup OpenTelemetry

```javascript
// tracing.js (ต้อง require ก่อน app code)
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

const exporter = new JaegerExporter({
  endpoint: process.env.JAEGER_ENDPOINT || 'http://localhost:14268/api/traces',
});

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: process.env.SERVICE_NAME || 'my-app',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION || '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'development',
  }),
  traceExporter: exporter,
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': { enabled: true },
      '@opentelemetry/instrumentation-express': { enabled: true },
      '@opentelemetry/instrumentation-mongoose': { enabled: true },
      '@opentelemetry/instrumentation-redis': { enabled: true },
    })
  ],
});

sdk.start();

process.on('SIGTERM', () => {
  sdk.shutdown()
    .then(() => console.log('Tracing terminated'))
    .catch((error) => console.error('Error terminating tracing', error))
    .finally(() => process.exit(0));
});

module.exports = sdk;
```

```javascript
// index.js (ต้อง require tracing ก่อน)
require('./tracing');
const app = require('./app');

app.listen(3000);
```

---

## 2. Jaeger Setup

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    environment:
      - COLLECTOR_ZIPKIN_HOST_PORT=:9411
    ports:
      - "5775:5775/udp"
      - "6831:6831/udp"
      - "6832:6832/udp"
      - "5778:5778"
      - "16686:16686"   # Jaeger UI
      - "14268:14268"   # Jaeger collector
      - "14250:14250"
      - "9411:9411"
```

---

## 3. Spans และ Traces

### Manual Span Creation

```javascript
// services/order.service.js
const { trace, context, SpanStatusCode } = require('@opentelemetry/api');

const tracer = trace.getTracer('order-service');

class OrderService {
  async createOrder(data) {
    // สร้าง span หลัก
    return tracer.startActiveSpan('order.create', async (span) => {
      try {
        span.setAttribute('order.userId', data.userId);
        span.setAttribute('order.itemCount', data.items.length);

        // Validate items
        const validationSpan = tracer.startSpan('order.validate');
        await this.validateItems(data.items);
        validationSpan.end();

        // Calculate total
        const total = await tracer.startActiveSpan('order.calculateTotal', async (calcSpan) => {
          const result = await this.calculateTotal(data.items);
          calcSpan.setAttribute('order.total', result);
          calcSpan.end();
          return result;
        });

        // Save to database
        const order = await tracer.startActiveSpan('order.save', async (saveSpan) => {
          const result = await Order.create({ ...data, total });
          saveSpan.setAttribute('order.id', result.id);
          saveSpan.end();
          return result;
        });

        span.setAttribute('order.id', order.id);
        span.setStatus({ code: SpanStatusCode.OK });
        
        return order;
      } catch (error) {
        span.recordException(error);
        span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
        throw error;
      } finally {
        span.end();
      }
    });
  }
}
```

### Tracing Middleware

```javascript
// middleware/tracing.js
const { trace, context, propagation } = require('@opentelemetry/api');

function tracingMiddleware(req, res, next) {
  const tracer = trace.getTracer('http');
  
  // Extract trace context จาก incoming headers
  const ctx = propagation.extract(context.active(), req.headers);
  
  const span = tracer.startSpan(
    `${req.method} ${req.path}`,
    {
      kind: 1, // SERVER
      attributes: {
        'http.method': req.method,
        'http.url': req.url,
        'http.route': req.route?.path,
        'http.user_agent': req.get('user-agent'),
      }
    },
    ctx
  );

  // เพิ่ม trace ID ใน response header เพื่อ debug ง่าย
  const traceId = span.spanContext().traceId;
  res.setHeader('X-Trace-Id', traceId);

  context.with(trace.setSpan(context.active(), span), () => {
    res.on('finish', () => {
      span.setAttribute('http.status_code', res.statusCode);
      
      if (res.statusCode >= 500) {
        span.setStatus({ code: 2, message: 'Server error' });
      }
      
      span.end();
    });

    next();
  });
}

module.exports = tracingMiddleware;
```

---

## 4. Context Propagation

### HTTP Headers Propagation

```javascript
// services/payment.service.js (downstream service call)
const { context, propagation } = require('@opentelemetry/api');
const axios = require('axios');

async function processPayment(orderData) {
  // Inject trace context ไปยัง outgoing request
  const headers = {};
  propagation.inject(context.active(), headers);

  const response = await axios.post(
    `${process.env.PAYMENT_SERVICE_URL}/charge`,
    orderData,
    { headers }
  );

  return response.data;
}
```

### Custom Attributes สำหรับ Business Context

```javascript
// helpers/tracing-helpers.js
const { trace } = require('@opentelemetry/api');

function addBusinessContext(attributes) {
  const span = trace.getActiveSpan();
  if (span) {
    Object.entries(attributes).forEach(([key, value]) => {
      span.setAttribute(key, value);
    });
  }
}

function recordError(error, attributes = {}) {
  const span = trace.getActiveSpan();
  if (span) {
    span.recordException(error);
    span.setStatus({ code: 2, message: error.message });
    Object.entries(attributes).forEach(([key, value]) => {
      span.setAttribute(key, value);
    });
  }
}

// ใช้งาน
async function processOrder(req, res) {
  addBusinessContext({
    'user.id': req.user.id,
    'user.plan': req.user.plan,
    'order.source': req.headers['x-order-source'] || 'web'
  });

  try {
    const order = await orderService.create(req.body);
    res.json(order);
  } catch (error) {
    recordError(error, { 'order.failureReason': error.code });
    throw error;
  }
}
```

---

## 5. Tracing Best Practices

```javascript
// Trace Sampling
const { TraceIdRatioBased, ParentBasedSampler } = require('@opentelemetry/sdk-trace-base');

// Sample 10% ใน production
const sampler = new ParentBasedSampler({
  root: new TraceIdRatioBased(0.1)
});

const sdk = new NodeSDK({
  sampler,
  // ... other config
});

// Force trace สำหรับ errors
class ErrorForceSampler {
  shouldSample(context, traceId, spanName, spanKind, attributes) {
    // Force sample ถ้ามี error
    if (attributes['error'] === true) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLE };
    }
    
    // Else use ratio sampling
    return { decision: SamplingDecision.NOT_RECORD };
  }
}
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
ตั้งค่า OpenTelemetry:
- Auto instrumentation
- Jaeger exporter
- View traces ใน Jaeger UI

### ระดับ 2: กลาง
เพิ่ม manual tracing:
- Custom spans สำหรับ business operations
- Business attributes
- Error recording

### ระดับ 3: ขั้นสูง
Distributed tracing:
- Context propagation ระหว่าง services
- Sampling strategy
- Custom exporters

---

## สรุป

Distributed Tracing ช่วย debug performance issues ใน microservices ได้อย่างมีประสิทธิภาพ OpenTelemetry เป็น standard ที่ vendor-neutral รองรับ exporters หลายตัว ควรใช้ auto-instrumentation เป็น baseline และเพิ่ม manual spans เฉพาะจุดที่สำคัญ

> ขั้นตอนต่อไป: Part 69 - Load Balancing
