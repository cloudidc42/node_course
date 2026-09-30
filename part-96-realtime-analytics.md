# Part 96 | ขั้นตอนที่ 1721-1740 จาก 1000+

## Real-time Analytics และ Stream Processing

ในส่วนนี้เราจะเรียนรู้การสร้าง real-time analytics systems ด้วย ClickHouse, TimescaleDB, และ stream processing สำหรับ Node.js applications

---

## ขั้นตอนที่ 1721: Real-time Analytics Architecture

```
Events → Kafka → Stream Processor → ClickHouse/TimescaleDB → Dashboard
                      ↓
                  Redis (hot data)
```

```javascript
// analytics-pipeline.js
// Real-time analytics pipeline

const { Kafka } = require('kafkajs');
const { createClient } = require('@clickhouse/client');
const Redis = require('ioredis');

class AnalyticsPipeline {
  constructor(config) {
    this.kafka = new Kafka({
      clientId: 'analytics-processor',
      brokers: config.kafka.brokers
    });
    
    this.clickhouse = createClient({
      host: config.clickhouse.host,
      database: config.clickhouse.database,
      username: config.clickhouse.username,
      password: config.clickhouse.password
    });
    
    this.redis = new Redis(config.redis);
    this.batchSize = config.batchSize || 1000;
    this.batchTimeout = config.batchTimeout || 5000; // 5 seconds
    this.buffer = [];
    this.bufferTimer = null;
  }

  async start() {
    const consumer = this.kafka.consumer({ groupId: 'analytics-group' });
    await consumer.connect();
    await consumer.subscribe({ topics: ['user-events', 'page-views', 'transactions'] });
    
    await consumer.run({
      eachBatch: async ({ batch, resolveOffset, heartbeat }) => {
        for (const message of batch.messages) {
          const event = JSON.parse(message.value.toString());
          await this.processEvent(event);
          resolveOffset(message.offset);
          await heartbeat();
        }
      }
    });
  }

  async processEvent(event) {
    // Enrich event
    const enriched = this.enrichEvent(event);
    
    // Update real-time counters in Redis
    await this.updateRealtimeCounters(enriched);
    
    // Buffer for batch insert to ClickHouse
    this.buffer.push(enriched);
    
    if (this.buffer.length >= this.batchSize) {
      await this.flushToClickHouse();
    } else if (!this.bufferTimer) {
      this.bufferTimer = setTimeout(() => this.flushToClickHouse(), this.batchTimeout);
    }
  }

  enrichEvent(event) {
    return {
      ...event,
      timestamp: new Date(event.timestamp || Date.now()),
      date: new Date().toISOString().split('T')[0],
      hour: new Date().getHours(),
      dayOfWeek: new Date().getDay()
    };
  }

  async updateRealtimeCounters(event) {
    const pipeline = this.redis.pipeline();
    const minute = Math.floor(Date.now() / 60000);
    
    // Per-minute counters (expire after 1 hour)
    pipeline.incr(`events:${event.type}:${minute}`);
    pipeline.expire(`events:${event.type}:${minute}`, 3600);
    
    // Per-user activity
    if (event.userId) {
      pipeline.zadd('active_users', Date.now(), event.userId);
      pipeline.expire('active_users', 300); // 5 min window
    }
    
    // Per-page counters
    if (event.page) {
      pipeline.incr(`page_views:${event.page}`);
      pipeline.zincrby('top_pages', 1, event.page);
    }
    
    await pipeline.exec();
  }

  async flushToClickHouse() {
    if (this.buffer.length === 0) return;
    
    const events = [...this.buffer];
    this.buffer = [];
    
    if (this.bufferTimer) {
      clearTimeout(this.bufferTimer);
      this.bufferTimer = null;
    }
    
    try {
      await this.clickhouse.insert({
        table: 'events',
        values: events,
        format: 'JSONEachRow'
      });
      
      console.log(`Flushed ${events.length} events to ClickHouse`);
    } catch (err) {
      console.error('ClickHouse insert failed:', err.message);
      // Put back in buffer for retry
      this.buffer.unshift(...events);
    }
  }
}

module.exports = { AnalyticsPipeline };
```

---

## ขั้นตอนที่ 1722: ClickHouse Schema Design

```sql
-- ClickHouse schema for analytics

-- Events table with MergeTree for high throughput
CREATE TABLE IF NOT EXISTS events (
    id UUID DEFAULT generateUUIDv4(),
    timestamp DateTime64(3) DEFAULT now(),
    date Date DEFAULT toDate(timestamp),
    hour UInt8 DEFAULT toHour(timestamp),
    event_type LowCardinality(String),
    user_id String,
    session_id String,
    page String,
    properties JSON,
    country LowCardinality(String),
    device_type LowCardinality(String),
    INDEX idx_event_type event_type TYPE bloom_filter GRANULARITY 1,
    INDEX idx_user_id user_id TYPE bloom_filter GRANULARITY 1
) ENGINE = MergeTree()
PARTITION BY date
ORDER BY (date, event_type, user_id)
TTL date + INTERVAL 90 DAY;

-- Materialized view for hourly aggregates
CREATE MATERIALIZED VIEW hourly_stats
ENGINE = SummingMergeTree()
PARTITION BY date
ORDER BY (date, hour, event_type)
AS SELECT
    date,
    hour,
    event_type,
    count() as event_count,
    uniq(user_id) as unique_users,
    uniq(session_id) as unique_sessions
FROM events
GROUP BY date, hour, event_type;

-- Real-time DAU (Daily Active Users)
CREATE VIEW dau AS
SELECT
    date,
    uniq(user_id) as dau,
    count() as total_events
FROM events
WHERE date >= today() - 30
GROUP BY date
ORDER BY date;
```

---

## ขั้นตอนที่ 1723: TimescaleDB for Time-Series Metrics

```javascript
// timescale-metrics.js
// TimescaleDB for time-series application metrics

const { Pool } = require('pg');

class MetricsStore {
  constructor(config) {
    this.pool = new Pool(config);
  }

  async initialize() {
    await this.pool.query(`
      CREATE TABLE IF NOT EXISTS app_metrics (
        time TIMESTAMPTZ NOT NULL,
        metric_name TEXT NOT NULL,
        value DOUBLE PRECISION NOT NULL,
        labels JSONB DEFAULT '{}'
      );
      
      -- Convert to hypertable for automatic partitioning
      SELECT create_hypertable('app_metrics', 'time', if_not_exists => TRUE);
      
      -- Continuous aggregate for per-minute averages
      CREATE MATERIALIZED VIEW IF NOT EXISTS metrics_1min
      WITH (timescaledb.continuous) AS
      SELECT
        time_bucket('1 minute', time) as bucket,
        metric_name,
        labels,
        avg(value) as avg_value,
        min(value) as min_value,
        max(value) as max_value,
        count(*) as sample_count
      FROM app_metrics
      GROUP BY bucket, metric_name, labels;
      
      -- Add retention policy (90 days)
      SELECT add_retention_policy('app_metrics', INTERVAL '90 days', if_not_exists => TRUE);
    `);
  }

  async record(metricName, value, labels = {}) {
    await this.pool.query(
      `INSERT INTO app_metrics (time, metric_name, value, labels) VALUES (NOW(), $1, $2, $3)`,
      [metricName, value, JSON.stringify(labels)]
    );
  }

  async batchRecord(metrics) {
    const values = metrics.map((m, i) => `(NOW(), $${i*3+1}, $${i*3+2}, $${i*3+3})`).join(', ');
    const params = metrics.flatMap(m => [m.name, m.value, JSON.stringify(m.labels || {})]);
    
    await this.pool.query(
      `INSERT INTO app_metrics (time, metric_name, value, labels) VALUES ${values}`,
      params
    );
  }

  async query(metricName, startTime, endTime, interval = '1 minute') {
    const { rows } = await this.pool.query(`
      SELECT
        time_bucket($1::interval, time) as bucket,
        avg(value) as avg_value,
        percentile_cont(0.95) WITHIN GROUP (ORDER BY value) as p95,
        percentile_cont(0.99) WITHIN GROUP (ORDER BY value) as p99,
        count(*) as sample_count
      FROM app_metrics
      WHERE metric_name = $2 AND time BETWEEN $3 AND $4
      GROUP BY bucket
      ORDER BY bucket
    `, [interval, metricName, startTime, endTime]);
    
    return rows;
  }

  // Anomaly detection query
  async detectAnomalies(metricName, window = '1 hour', threshold = 3) {
    const { rows } = await this.pool.query(`
      WITH stats AS (
        SELECT
          time,
          value,
          avg(value) OVER (ORDER BY time RANGE BETWEEN INTERVAL '1 hour' PRECEDING AND CURRENT ROW) as rolling_avg,
          stddev(value) OVER (ORDER BY time RANGE BETWEEN INTERVAL '1 hour' PRECEDING AND CURRENT ROW) as rolling_stddev
        FROM app_metrics
        WHERE metric_name = $1
        AND time > NOW() - $2::interval
      )
      SELECT 
        time,
        value,
        rolling_avg,
        rolling_stddev,
        ABS(value - rolling_avg) / NULLIF(rolling_stddev, 0) as z_score
      FROM stats
      WHERE ABS(value - rolling_avg) / NULLIF(rolling_stddev, 0) > $3
      ORDER BY time DESC
    `, [metricName, window, threshold]);
    
    return rows;
  }
}

module.exports = { MetricsStore };
```

---

## ขั้นตอนที่ 1724-1740: Real-time Dashboard API

```javascript
// analytics-api.js
// API endpoints for analytics dashboard

const express = require('express');
const { AnalyticsPipeline } = require('./analytics-pipeline');
const { MetricsStore } = require('./timescale-metrics');
const Redis = require('ioredis');
const WebSocket = require('ws');

class AnalyticsDashboardAPI {
  constructor(config) {
    this.app = express();
    this.app.use(express.json());
    
    this.redis = new Redis(config.redis);
    this.metrics = new MetricsStore(config.timescale);
    this.clickhouse = config.clickhouse;
    
    this.setupRoutes();
    this.setupWebSocket();
  }

  setupRoutes() {
    // Real-time stats (from Redis)
    this.app.get('/stats/realtime', async (req, res) => {
      try {
        const [activeUsers, topPages, eventCounts] = await Promise.all([
          this.redis.zcount('active_users', Date.now() - 300000, Date.now()),
          this.redis.zrevrange('top_pages', 0, 9, 'WITHSCORES'),
          this.getEventCounts()
        ]);
        
        res.json({
          activeUsers,
          topPages: this.parseZSetResponse(topPages),
          eventCounts,
          timestamp: new Date().toISOString()
        });
      } catch (err) {
        res.status(500).json({ error: err.message });
      }
    });

    // Historical metrics
    this.app.get('/metrics/:name', async (req, res) => {
      const { name } = req.params;
      const { start, end, interval } = req.query;
      
      try {
        const data = await this.metrics.query(
          name,
          new Date(start || Date.now() - 3600000),
          new Date(end || Date.now()),
          interval || '1 minute'
        );
        
        res.json({ metric: name, data });
      } catch (err) {
        res.status(500).json({ error: err.message });
      }
    });

    // Funnel analysis
    this.app.get('/funnel', async (req, res) => {
      const { steps, startDate, endDate } = req.query;
      const stepList = JSON.parse(steps || '[]');
      
      try {
        const results = await this.analyzeFunnel(stepList, startDate, endDate);
        res.json({ funnel: results });
      } catch (err) {
        res.status(500).json({ error: err.message });
      }
    });
  }

  setupWebSocket() {
    this.wss = new WebSocket.Server({ noServer: true });
    
    // Push real-time updates every 5 seconds
    setInterval(async () => {
      if (this.wss.clients.size === 0) return;
      
      try {
        const stats = await this.getRealtimeStats();
        const message = JSON.stringify({ type: 'stats', data: stats });
        
        this.wss.clients.forEach(client => {
          if (client.readyState === WebSocket.OPEN) {
            client.send(message);
          }
        });
      } catch (err) {
        console.error('WebSocket broadcast failed:', err.message);
      }
    }, 5000);
    
    this.wss.on('connection', (ws) => {
      ws.send(JSON.stringify({ type: 'connected', timestamp: Date.now() }));
    });
  }

  async getRealtimeStats() {
    const [activeUsers, topPages] = await Promise.all([
      this.redis.zcount('active_users', Date.now() - 300000, Date.now()),
      this.redis.zrevrange('top_pages', 0, 4, 'WITHSCORES')
    ]);
    
    return {
      activeUsers,
      topPages: this.parseZSetResponse(topPages),
      timestamp: Date.now()
    };
  }

  async getEventCounts() {
    const minute = Math.floor(Date.now() / 60000);
    const keys = ['page_view', 'click', 'purchase'].map(type => `events:${type}:${minute}`);
    const values = await this.redis.mget(...keys);
    
    return Object.fromEntries(
      ['page_view', 'click', 'purchase'].map((type, i) => [type, parseInt(values[i]) || 0])
    );
  }

  async analyzeFunnel(steps, startDate, endDate) {
    const results = [];
    let previousCount = null;
    
    for (const step of steps) {
      const { rows } = await this.clickhouse.query({
        query: `
          SELECT COUNT(DISTINCT user_id) as users
          FROM events
          WHERE event_type = {step:String}
          AND date BETWEEN {startDate:Date} AND {endDate:Date}
        `,
        query_params: { step, startDate, endDate }
      }).then(r => r.json());
      
      const count = rows[0].users;
      const conversionRate = previousCount ? (count / previousCount * 100).toFixed(1) : 100;
      
      results.push({ step, users: count, conversionRate: parseFloat(conversionRate) });
      previousCount = count;
    }
    
    return results;
  }

  parseZSetResponse(data) {
    const result = [];
    for (let i = 0; i < data.length; i += 2) {
      result.push({ name: data[i], score: parseInt(data[i + 1]) });
    }
    return result;
  }

  listen(port = 3000) {
    const server = this.app.listen(port, () => {
      console.log(`Analytics API running on port ${port}`);
    });
    
    server.on('upgrade', (request, socket, head) => {
      this.wss.handleUpgrade(request, socket, head, ws => {
        this.wss.emit('connection', ws, request);
      });
    });
    
    return server;
  }
}

module.exports = { AnalyticsDashboardAPI };
```

---

## แบบฝึกหัด

### Exercise 1: Cohort Analysis
สร้าง ClickHouse query ที่วิเคราะห์ user retention cohorts รายสัปดาห์

### Exercise 2: Real-time Alerting
เพิ่ม alerting system ที่ trigger เมื่อ metrics เกิน threshold ที่กำหนด

### Exercise 3: A/B Test Analytics
สร้าง analytics service ที่คำนวณ statistical significance ของ A/B test results

### คำถามทบทวน
1. ClickHouse เหมาะกับ use case อะไร เมื่อเทียบกับ PostgreSQL?
2. TimescaleDB ช่วยแก้ปัญหาอะไรใน time-series data?
3. Materialized views ใน ClickHouse ทำงานอย่างไร?

---

*ต่อไป: Part 97 - Technical Leadership*
