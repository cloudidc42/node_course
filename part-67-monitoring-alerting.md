# Part 67: Monitoring & Alerting
## ขั้นตอนที่ 661-670 จาก 1000

---

## ทำไมต้อง Monitor?

Monitoring ช่วยให้เราเข้าใจสถานะของ application ในขณะ runtime ตรวจพบปัญหาก่อนที่ user จะรายงาน และตัดสินใจ scale infrastructure ได้ถูกต้อง

---

## 1. Prometheus Metrics

### ติดตั้ง prom-client

```bash
npm install prom-client express
```

### ตั้งค่า Prometheus Registry

```javascript
// monitoring/metrics.js
const prometheus = require('prom-client');

// สร้าง registry แยก (ดีกว่าใช้ defaultRegistry)
const registry = new prometheus.Registry();

// เพิ่ม default metrics (CPU, memory, GC, etc.)
prometheus.collectDefaultMetrics({ 
  register: registry,
  prefix: 'node_app_'
});

// Custom Metrics

// Counter - นับ requests
const httpRequestsTotal = new prometheus.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [registry]
});

// Histogram - วัด response time
const httpRequestDurationMs = new prometheus.Histogram({
  name: 'http_request_duration_ms',
  help: 'HTTP request duration in milliseconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000],
  registers: [registry]
});

// Gauge - ค่าที่ขึ้นลงได้
const activeConnections = new prometheus.Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
  registers: [registry]
});

const dbConnectionPool = new prometheus.Gauge({
  name: 'db_connection_pool_size',
  help: 'Database connection pool size',
  labelNames: ['status'],
  registers: [registry]
});

// Summary - percentile calculations
const httpRequestSummary = new prometheus.Summary({
  name: 'http_request_summary_ms',
  help: 'HTTP request latency summary',
  labelNames: ['method', 'route'],
  percentiles: [0.5, 0.9, 0.95, 0.99],
  registers: [registry]
});

// Business metrics
const orderCreated = new prometheus.Counter({
  name: 'orders_created_total',
  help: 'Total orders created',
  labelNames: ['payment_method'],
  registers: [registry]
});

const orderValue = new prometheus.Histogram({
  name: 'order_value_baht',
  help: 'Order value in Baht',
  buckets: [100, 500, 1000, 5000, 10000, 50000],
  registers: [registry]
});

module.exports = {
  registry,
  httpRequestsTotal,
  httpRequestDurationMs,
  activeConnections,
  dbConnectionPool,
  httpRequestSummary,
  orderCreated,
  orderValue
};
```

### Metrics Middleware

```javascript
// middleware/metrics.js
const { httpRequestsTotal, httpRequestDurationMs, httpRequestSummary } = require('../monitoring/metrics');

function metricsMiddleware(req, res, next) {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    const route = req.route?.path || req.path || 'unknown';
    const labels = {
      method: req.method,
      route: normalizeRoute(route),
      status_code: res.statusCode.toString()
    };

    httpRequestsTotal.inc(labels);
    httpRequestDurationMs.observe(labels, duration);
    httpRequestSummary.observe(
      { method: req.method, route: normalizeRoute(route) },
      duration
    );
  });

  next();
}

function normalizeRoute(path) {
  // แปลง /users/123 เป็น /users/:id
  return path.replace(/\/[0-9a-f]{24}/gi, '/:id')
             .replace(/\/\d+/g, '/:id');
}

module.exports = metricsMiddleware;
```

### Metrics Endpoint

```javascript
// routes/monitoring.js
const express = require('express');
const router = express.Router();
const { registry } = require('../monitoring/metrics');

// Prometheus scrape endpoint
router.get('/metrics', async (req, res) => {
  // ตรวจสอบ authentication (ป้องกัน public access)
  const token = req.headers['x-metrics-token'];
  if (token !== process.env.METRICS_TOKEN) {
    return res.status(403).send('Forbidden');
  }

  res.set('Content-Type', registry.contentType);
  res.end(await registry.metrics());
});

// Health check endpoint
router.get('/health', async (req, res) => {
  const health = await checkHealth();
  const statusCode = health.status === 'healthy' ? 200 : 503;
  res.status(statusCode).json(health);
});

// Readiness check
router.get('/ready', async (req, res) => {
  const ready = await checkReadiness();
  res.status(ready ? 200 : 503).json({ ready });
});

async function checkHealth() {
  const checks = await Promise.allSettled([
    checkDatabase(),
    checkRedis(),
    checkExternalAPIs()
  ]);

  const results = {
    database: checks[0].status === 'fulfilled' ? checks[0].value : { status: 'error', error: checks[0].reason?.message },
    redis: checks[1].status === 'fulfilled' ? checks[1].value : { status: 'error', error: checks[1].reason?.message },
    externalAPIs: checks[2].status === 'fulfilled' ? checks[2].value : { status: 'error', error: checks[2].reason?.message }
  };

  const allHealthy = Object.values(results).every(r => r.status === 'ok');

  return {
    status: allHealthy ? 'healthy' : 'degraded',
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION || '1.0.0',
    uptime: process.uptime(),
    checks: results
  };
}

async function checkDatabase() {
  const mongoose = require('mongoose');
  const state = mongoose.connection.readyState;
  if (state !== 1) throw new Error('Database not connected');
  return { status: 'ok', latency: await measureDbLatency() };
}

async function measureDbLatency() {
  const start = Date.now();
  await mongoose.connection.db.command({ ping: 1 });
  return Date.now() - start;
}

module.exports = router;
```

---

## 2. Grafana Dashboards

### Prometheus Configuration (prometheus.yml)

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'nodejs-app'
    static_configs:
      - targets: ['localhost:3000']
    metrics_path: '/metrics'
    bearer_token: 'your-metrics-token'
    
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
```

### Docker Compose สำหรับ Monitoring Stack

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=secret
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - "3001:3000"
    depends_on:
      - prometheus

  alertmanager:
    image: prom/alertmanager:latest
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"

volumes:
  prometheus_data:
  grafana_data:
```

### Grafana Dashboard JSON

```json
{
  "title": "Node.js App Dashboard",
  "panels": [
    {
      "title": "Request Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "rate(http_requests_total[5m])",
          "legendFormat": "{{method}} {{route}} {{status_code}}"
        }
      ]
    },
    {
      "title": "Response Time P95",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, rate(http_request_duration_ms_bucket[5m]))",
          "legendFormat": "P95 {{route}}"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "stat",
      "targets": [
        {
          "expr": "rate(http_requests_total{status_code=~'5..'}[5m]) / rate(http_requests_total[5m]) * 100",
          "legendFormat": "Error %"
        }
      ]
    },
    {
      "title": "Memory Usage",
      "type": "graph",
      "targets": [
        {
          "expr": "node_app_process_resident_memory_bytes / 1024 / 1024",
          "legendFormat": "RSS MB"
        }
      ]
    }
  ]
}
```

---

## 3. Health Checks

```javascript
// monitoring/health-check.js
class HealthChecker {
  constructor() {
    this.checks = new Map();
  }

  register(name, checkFn, options = {}) {
    this.checks.set(name, {
      fn: checkFn,
      timeout: options.timeout || 5000,
      critical: options.critical !== false
    });
    return this;
  }

  async runAll() {
    const results = {};
    let overallStatus = 'healthy';

    for (const [name, check] of this.checks) {
      try {
        const start = Date.now();
        
        const result = await Promise.race([
          check.fn(),
          new Promise((_, reject) => 
            setTimeout(() => reject(new Error('Check timed out')), check.timeout)
          )
        ]);

        results[name] = {
          status: 'ok',
          latency: Date.now() - start,
          ...result
        };
      } catch (error) {
        results[name] = {
          status: 'error',
          error: error.message
        };

        if (check.critical) {
          overallStatus = 'unhealthy';
        } else if (overallStatus === 'healthy') {
          overallStatus = 'degraded';
        }
      }
    }

    return {
      status: overallStatus,
      timestamp: new Date().toISOString(),
      checks: results
    };
  }
}

// ตั้งค่า health checks
const healthChecker = new HealthChecker();

healthChecker
  .register('database', async () => {
    const latency = await measureDbLatency();
    if (latency > 1000) throw new Error('Database too slow');
    return { latency };
  }, { critical: true, timeout: 3000 })
  
  .register('redis', async () => {
    const redis = require('../redis');
    const start = Date.now();
    await redis.ping();
    return { latency: Date.now() - start };
  }, { critical: false })
  
  .register('disk', async () => {
    const { checkDisk } = require('diskusage');
    const info = await checkDisk('/');
    const usedPercent = ((info.total - info.free) / info.total) * 100;
    if (usedPercent > 90) throw new Error('Disk usage too high');
    return { usedPercent: usedPercent.toFixed(1) };
  }, { critical: false });

module.exports = healthChecker;
```

---

## 4. Alerting

### Prometheus Alerting Rules

```yaml
# alerts.yml
groups:
  - name: nodejs_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status_code=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"

      - alert: SlowResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_ms_bucket[5m])) > 2000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow response times"
          description: "95th percentile response time is {{ $value }}ms"

      - alert: HighMemoryUsage
        expr: node_app_process_resident_memory_bytes / 1024 / 1024 > 512
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value }}MB"

      - alert: AppDown
        expr: up{job="nodejs-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Application is down"
```

### AlertManager Configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 10m
  repeat_interval: 1h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true
    - match:
        severity: warning
      receiver: 'slack'

receivers:
  - name: 'default'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#alerts'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'pagerduty'
    pagerduty_configs:
      - routing_key: 'YOUR_PAGERDUTY_KEY'
        description: '{{ .CommonAnnotations.summary }}'

  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#monitoring'
```

### Custom Alert via Code

```javascript
// monitoring/alerting.js
const axios = require('axios');

class AlertService {
  constructor() {
    this.slackWebhook = process.env.SLACK_WEBHOOK_URL;
    this.cooldowns = new Map();
  }

  async sendAlert(alert) {
    const key = `${alert.type}:${alert.service}`;
    const lastSent = this.cooldowns.get(key);
    const cooldownMs = 5 * 60 * 1000; // 5 minutes

    if (lastSent && Date.now() - lastSent < cooldownMs) {
      return; // Skip (cooldown)
    }

    this.cooldowns.set(key, Date.now());

    if (this.slackWebhook) {
      await this.sendSlack(alert);
    }

    // Log the alert
    console.error('ALERT:', JSON.stringify(alert));
  }

  async sendSlack(alert) {
    const color = alert.severity === 'critical' ? 'danger' : 'warning';
    
    await axios.post(this.slackWebhook, {
      attachments: [{
        color,
        title: `[${alert.severity.toUpperCase()}] ${alert.title}`,
        text: alert.message,
        fields: [
          { title: 'Service', value: alert.service, short: true },
          { title: 'Time', value: new Date().toISOString(), short: true }
        ]
      }]
    });
  }
}

const alertService = new AlertService();

// ใช้ใน error handler
app.use(async (err, req, res, next) => {
  if (err.statusCode >= 500) {
    await alertService.sendAlert({
      type: 'server_error',
      severity: 'critical',
      service: 'api',
      title: 'Server Error',
      message: err.message
    });
  }
  
  res.status(err.statusCode || 500).json({
    error: err.message
  });
});
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
ตั้งค่า basic monitoring:
- prom-client metrics
- /metrics endpoint
- /health endpoint
- Docker Compose สำหรับ Prometheus + Grafana

### ระดับ 2: กลาง
เพิ่ม custom metrics:
- Business metrics (orders, revenue)
- Database query duration
- Cache hit/miss rate
- Grafana dashboard

### ระดับ 3: ขั้นสูง
สร้าง complete alerting system:
- Alert rules
- AlertManager ด้วย Slack/PagerDuty
- Custom alert service
- SLI/SLO tracking

---

## สรุป

Monitoring เป็นส่วนสำคัญของ production system ที่ดี Prometheus + Grafana เป็น stack ยอดนิยมที่ cost-effective และ flexible สูง ควรวัด RED metrics (Rate, Errors, Duration) เป็นขั้นต่ำ และเพิ่ม business metrics ตามความต้องการ

> ขั้นตอนต่อไป: Part 68 - Distributed Tracing
