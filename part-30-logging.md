# Part 30: Logging

> ขั้นตอนที่ 30-30 จาก 1000 — Production Logging ด้วย Winston, Morgan และ Structured Logging

---

## สารบัญ

1. [Logging Levels](#logging-levels)
2. [Winston Logger](#winston-logger)
3. [Morgan HTTP Logger](#morgan-http-logger)
4. [Structured Logging](#structured-logging)
5. [Log Rotation](#log-rotation)
6. [Remote Logging](#remote-logging)
7. [Practical: Production Logging Setup](#practical-production-logging-setup)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Logging Levels

### RFC 5424 Syslog Levels

| Level | ค่า | ใช้เมื่อ |
|-------|-----|---------|
| `error` | 0 | Production errors, exceptions |
| `warn` | 1 | Deprecations, retry attempts |
| `info` | 2 | ข้อมูลทั่วไป (startup, shutdown) |
| `http` | 3 | HTTP request logs |
| `verbose` | 4 | ข้อมูลเพิ่มเติม |
| `debug` | 5 | Debug information |
| `silly` | 6 | ข้อมูลมาก (development only) |

### กฎการเลือก Level

```javascript
// ใช้แต่ละ level อย่างเหมาะสม
logger.error('Database connection failed', { error });          // error เสมอ
logger.warn('Redis unavailable, falling back to memory cache'); // บอก degraded
logger.info('Server started on port 3000');                     // สถานะสำคัญ
logger.http('GET /api/users 200 45ms');                        // HTTP access log
logger.debug('Cache hit for key: user:123');                   // dev ใช้
```

---

## Winston Logger

### ติดตั้ง

```bash
npm install winston winston-daily-rotate-file
npm install @google-cloud/logging-winston  # ถ้าใช้ GCP
npm install winston-transport-sentry-node  # ถ้าส่งไป Sentry
```

### Winston Setup พื้นฐาน

```javascript
// utils/logger.js
const winston = require('winston');
const DailyRotateFile = require('winston-daily-rotate-file');
const path = require('path');

// Custom format
const { combine, timestamp, errors, json, colorize, printf, splat } = winston.format;

// Format สำหรับ console (development)
const consoleFormat = combine(
  colorize({ all: true }),
  timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
  errors({ stack: true }),
  printf(({ level, message, timestamp, stack, ...meta }) => {
    let log = `${timestamp} [${level}]: ${message}`;
    
    // แสดง meta data ถ้ามี
    if (Object.keys(meta).length > 0) {
      log += `\n${JSON.stringify(meta, null, 2)}`;
    }
    
    // แสดง stack trace ถ้ามี
    if (stack) {
      log += `\n${stack}`;
    }
    
    return log;
  })
);

// Format สำหรับ file (JSON)
const fileFormat = combine(
  timestamp(),
  errors({ stack: true }),
  splat(),
  json()
);

// Transport สำหรับ file rotation
const fileTransport = new DailyRotateFile({
  dirname: path.join(process.cwd(), 'logs'),
  filename: '%DATE%-app.log',
  datePattern: 'YYYY-MM-DD',
  zippedArchive: true,         // compress ไฟล์เก่า
  maxSize: '20m',              // สูงสุด 20 MB ต่อไฟล์
  maxFiles: '30d',             // เก็บ 30 วัน
  format: fileFormat,
  handleExceptions: true,
  handleRejections: true,
});

// Error log file แยก
const errorFileTransport = new DailyRotateFile({
  dirname: path.join(process.cwd(), 'logs'),
  filename: '%DATE%-error.log',
  datePattern: 'YYYY-MM-DD',
  level: 'error',
  zippedArchive: true,
  maxSize: '10m',
  maxFiles: '90d',
  format: fileFormat,
});

// สร้าง logger
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || (process.env.NODE_ENV === 'production' ? 'info' : 'debug'),
  
  defaultMeta: {
    service: process.env.APP_NAME || 'api',
    environment: process.env.NODE_ENV,
    version: process.env.APP_VERSION,
  },
  
  transports: [
    // Console transport
    new winston.transports.Console({
      format: process.env.NODE_ENV === 'production' ? fileFormat : consoleFormat,
      handleExceptions: true,
      handleRejections: true,
    }),
  ],
  
  exitOnError: false,  // ไม่ exit เมื่อ logging fails
});

// เพิ่ม file transports ใน production/staging
if (['production', 'staging'].includes(process.env.NODE_ENV)) {
  logger.add(fileTransport);
  logger.add(errorFileTransport);
}

// ดักจับ unhandled errors
logger.exceptions.handle(
  new winston.transports.File({
    filename: path.join(process.cwd(), 'logs', 'exceptions.log'),
  })
);

logger.rejections.handle(
  new winston.transports.File({
    filename: path.join(process.cwd(), 'logs', 'rejections.log'),
  })
);

module.exports = logger;
```

### Child Logger

```javascript
// สร้าง child logger พร้อม context
const requestLogger = logger.child({
  requestId: req.id,
  userId: req.user?._id,
  method: req.method,
  url: req.originalUrl,
});

requestLogger.info('Processing request');
requestLogger.error('Request failed', { error: err.message });
// logs จะมี requestId, userId ทุกครั้งอัตโนมัติ
```

### Logger Profiles

```javascript
// วัดเวลา
logger.profile('database-query');

const results = await User.find({});

logger.profile('database-query');
// output: database-query: 45ms
```

---

## Morgan HTTP Logger

Morgan บันทึก HTTP access logs

### Setup Morgan

```javascript
// middleware/requestLogger.js
const morgan = require('morgan');
const logger = require('../utils/logger');

// Custom format string
const FORMAT = process.env.NODE_ENV === 'production'
  ? ':remote-addr - :method :url HTTP/:http-version :status :res[content-length] ":referrer" ":user-agent" :response-time ms'
  : ':method :url :status :response-time ms';

// Stream ไปยัง Winston
const stream = {
  write: (message) => {
    logger.http(message.trim());
  },
};

// Skip healthcheck requests
const skip = (req) => {
  return req.url === '/health' || req.url === '/favicon.ico';
};

module.exports = morgan(FORMAT, {
  stream,
  skip,
  immediate: false,  // log หลัง response ส่งไปแล้ว
});
```

### Custom Morgan Tokens

```javascript
// เพิ่ม custom tokens
morgan.token('user-id', (req) => req.user?._id || 'anonymous');
morgan.token('request-id', (req) => req.id);
morgan.token('body-size', (req) => {
  const len = req.headers['content-length'];
  return len ? `${Math.ceil(len / 1024)}KB` : '-';
});

const customFormat = ':request-id :user-id :method :url :status :response-time ms :body-size';

module.exports = morgan(customFormat, { stream });
```

---

## Structured Logging

Structured logging เก็บ logs เป็น JSON แทน plaintext — ทำให้ query และ analyze ได้ง่าย

### JSON Log Format

```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "info",
  "message": "User logged in",
  "service": "api",
  "environment": "production",
  "requestId": "abc123",
  "userId": "user-456",
  "email": "user@example.com",
  "ip": "203.0.113.1",
  "duration": 45
}
```

### Context-aware Logger

```javascript
// utils/contextLogger.js
const logger = require('./logger');
const { AsyncLocalStorage } = require('async_hooks');

// Store request context ไว้ใน async local storage
const asyncLocalStorage = new AsyncLocalStorage();

/**
 * Middleware ตั้งค่า request context
 */
exports.requestContextMiddleware = (req, res, next) => {
  const context = {
    requestId: req.id,
    method: req.method,
    url: req.originalUrl,
    ip: req.ip,
    userId: null,  // จะ set ทีหลังเมื่อ authenticate
  };
  
  asyncLocalStorage.run(context, () => {
    next();
  });
};

/**
 * อัปเดต context (เช่น เมื่อ user authenticate)
 */
exports.setContext = (updates) => {
  const context = asyncLocalStorage.getStore();
  if (context) {
    Object.assign(context, updates);
  }
};

/**
 * Logger ที่ใส่ context อัตโนมัติ
 */
const contextLogger = {
  log: (level, message, meta = {}) => {
    const context = asyncLocalStorage.getStore() || {};
    logger[level](message, { ...context, ...meta });
  },
  
  error: (message, meta) => contextLogger.log('error', message, meta),
  warn: (message, meta) => contextLogger.log('warn', message, meta),
  info: (message, meta) => contextLogger.log('info', message, meta),
  debug: (message, meta) => contextLogger.log('debug', message, meta),
  http: (message, meta) => contextLogger.log('http', message, meta),
};

module.exports = contextLogger;
```

### ใช้งาน Context Logger

```javascript
// middleware/auth.js
const { setContext } = require('../utils/contextLogger');

exports.authenticate = async (req, res, next) => {
  // ... verify token ...
  
  req.user = user;
  
  // อัปเดต log context
  setContext({
    userId: user._id.toString(),
    userEmail: user.email,
    userRole: user.role,
  });
  
  next();
};

// controller
const contextLogger = require('../utils/contextLogger');

exports.createOrder = async (req, res) => {
  contextLogger.info('Creating order', {
    itemCount: req.body.items.length,
    total: req.body.total,
  });
  
  // ... create order ...
  
  contextLogger.info('Order created', { orderId: order._id });
};
```

---

## Log Rotation

### Winston Daily Rotate File

```javascript
// Advanced rotation config
const DailyRotateFile = require('winston-daily-rotate-file');

const transport = new DailyRotateFile({
  dirname: './logs',
  filename: 'app-%DATE%.log',
  datePattern: 'YYYY-MM-DD-HH',  // rotate ทุกชั่วโมง
  zippedArchive: true,
  maxSize: '50m',
  maxFiles: '14d',              // เก็บ 14 วัน
  
  // Events
  auditFile: './logs/.audit.json',  // track rotated files
});

transport.on('rotate', (oldFilename, newFilename) => {
  logger.info('Log rotated', { from: oldFilename, to: newFilename });
});

transport.on('new', (newFilename) => {
  logger.info('New log file created', { filename: newFilename });
});
```

### Logrotate (Linux/Unix)

```bash
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 644 nodejs nodejs
    sharedscripts
    postrotate
        # ส่ง signal ให้ app reload log file
        kill -USR1 `cat /var/run/myapp.pid`
    endscript
}
```

---

## Remote Logging

### Elasticsearch (ELK Stack)

```bash
npm install winston-elasticsearch
```

```javascript
// config ส่ง logs ไป Elasticsearch
const { ElasticsearchTransport } = require('winston-elasticsearch');

const esTransport = new ElasticsearchTransport({
  level: 'info',
  clientOpts: {
    node: process.env.ELASTICSEARCH_URL,
    auth: {
      username: process.env.ES_USER,
      password: process.env.ES_PASS,
    },
  },
  index: `logs-${process.env.APP_NAME}-${process.env.NODE_ENV}`,
  buffering: true,
  bufferLimit: 100,
  flushInterval: 2000,  // flush ทุก 2 วินาที
});

logger.add(esTransport);
```

### Loki (Grafana Stack)

```bash
npm install winston-loki
```

```javascript
const LokiTransport = require('winston-loki');

const lokiTransport = new LokiTransport({
  host: process.env.LOKI_HOST || 'http://localhost:3100',
  labels: {
    app: process.env.APP_NAME,
    env: process.env.NODE_ENV,
  },
  json: true,
  format: winston.format.json(),
  replaceTimestamp: false,
  onConnectionError: (err) => console.error('Loki connection error:', err),
});

logger.add(lokiTransport);
```

### Datadog

```bash
npm install datadog-winston
```

```javascript
const DatadogWinston = require('datadog-winston');

logger.add(
  new DatadogWinston({
    apiKey: process.env.DATADOG_API_KEY,
    hostname: process.env.HOSTNAME || 'localhost',
    service: process.env.APP_NAME,
    ddsource: 'nodejs',
    ddtags: `env:${process.env.NODE_ENV}`,
    level: 'info',
  })
);
```

---

## Practical: Production Logging Setup

### ไฟล์โครงสร้าง

```
myapp/
├── logs/
│   ├── .audit.json      — rotation audit
│   ├── 2024-01-15-app.log.gz
│   └── 2024-01-15-error.log.gz
├── utils/
│   ├── logger.js        — Winston setup
│   └── contextLogger.js — Context-aware logger
└── middleware/
    └── requestLogger.js — Morgan setup
```

### Complete Logger Setup

```javascript
// utils/logger.js — Production-ready
const winston = require('winston');
const DailyRotateFile = require('winston-daily-rotate-file');

const LOG_DIR = process.env.LOG_DIR || './logs';
const LOG_LEVEL = process.env.LOG_LEVEL || (
  process.env.NODE_ENV === 'production' ? 'info' : 'debug'
);

// Sensitive fields ที่ต้อง mask
const SENSITIVE_FIELDS = ['password', 'token', 'secret', 'authorization', 'creditCard'];

const maskSensitiveData = winston.format((info) => {
  const mask = (obj) => {
    if (!obj || typeof obj !== 'object') return obj;
    
    return Object.fromEntries(
      Object.entries(obj).map(([key, value]) => {
        if (SENSITIVE_FIELDS.some((f) => key.toLowerCase().includes(f))) {
          return [key, '[MASKED]'];
        }
        if (typeof value === 'object') {
          return [key, mask(value)];
        }
        return [key, value];
      })
    );
  };
  
  return mask(info);
});

const logger = winston.createLogger({
  level: LOG_LEVEL,
  
  format: winston.format.combine(
    maskSensitiveData(),
    winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss.SSS' }),
    winston.format.errors({ stack: true }),
  ),
  
  defaultMeta: {
    service: process.env.APP_NAME || 'api',
    env: process.env.NODE_ENV,
    version: process.env.APP_VERSION || '1.0.0',
    pid: process.pid,
    hostname: require('os').hostname(),
  },
  
  transports: [
    // Console
    new winston.transports.Console({
      format: process.env.NODE_ENV !== 'production'
        ? winston.format.combine(
            winston.format.colorize(),
            winston.format.printf(({ level, message, timestamp, ...meta }) => {
              const metaStr = Object.keys(meta).length
                ? `\n${JSON.stringify(meta, null, 2)}`
                : '';
              return `${timestamp} ${level}: ${message}${metaStr}`;
            })
          )
        : winston.format.json(),
    }),
  ],
});

// Production transports
if (process.env.NODE_ENV !== 'test') {
  // All logs
  logger.add(new DailyRotateFile({
    dirname: LOG_DIR,
    filename: 'app-%DATE%.log',
    datePattern: 'YYYY-MM-DD',
    zippedArchive: true,
    maxSize: '50m',
    maxFiles: '30d',
    format: winston.format.json(),
  }));
  
  // Error logs only
  logger.add(new DailyRotateFile({
    dirname: LOG_DIR,
    filename: 'error-%DATE%.log',
    datePattern: 'YYYY-MM-DD',
    level: 'error',
    zippedArchive: true,
    maxSize: '20m',
    maxFiles: '90d',
    format: winston.format.json(),
  }));
}

module.exports = logger;
```

### Performance Logging

```javascript
// utils/performanceLogger.js
const logger = require('./logger');

/**
 * Log slow operations
 */
exports.measureTime = async (name, fn, warnThreshold = 500) => {
  const start = process.hrtime.bigint();
  
  try {
    const result = await fn();
    const durationMs = Number(process.hrtime.bigint() - start) / 1_000_000;
    
    const logData = { operation: name, durationMs: Math.round(durationMs) };
    
    if (durationMs > warnThreshold) {
      logger.warn(`Slow operation: ${name}`, logData);
    } else {
      logger.debug(`Operation completed: ${name}`, logData);
    }
    
    return result;
  } catch (error) {
    const durationMs = Number(process.hrtime.bigint() - start) / 1_000_000;
    logger.error(`Operation failed: ${name}`, {
      operation: name,
      durationMs: Math.round(durationMs),
      error: error.message,
    });
    throw error;
  }
};

// ใช้งาน
const users = await measureTime(
  'find-users-with-posts',
  () => User.find({}).populate('posts'),
  200  // warn ถ้าเกิน 200ms
);
```

### Audit Logger

```javascript
// utils/auditLogger.js
const logger = require('./logger');

const auditLogger = logger.child({ type: 'audit' });

/**
 * บันทึก audit log สำหรับ actions สำคัญ
 */
exports.logAction = (userId, action, resource, details = {}) => {
  auditLogger.info('User action', {
    userId,
    action,                    // 'CREATE', 'UPDATE', 'DELETE', 'VIEW'
    resource,                  // 'User', 'Order', 'Product'
    resourceId: details.id,
    changes: details.changes,  // old vs new values
    ip: details.ip,
    userAgent: details.userAgent,
    timestamp: new Date().toISOString(),
  });
};

// ใช้งาน
exports.logAction(
  req.user._id,
  'DELETE',
  'User',
  {
    id: deletedUser._id,
    ip: req.ip,
    userAgent: req.headers['user-agent'],
  }
);
```

### app.js — ใช้ Logger

```javascript
// app.js
const express = require('express');
const logger = require('./utils/logger');
const contextLogger = require('./utils/contextLogger');
const morganLogger = require('./middleware/requestLogger');
const { v4: uuidv4 } = require('uuid');

const app = express();

// Request ID
app.use((req, res, next) => {
  req.id = req.headers['x-request-id'] || uuidv4();
  res.setHeader('X-Request-ID', req.id);
  next();
});

// Context middleware
app.use(contextLogger.requestContextMiddleware);

// Morgan HTTP logging
app.use(morganLogger);

// ... routes ...

const PORT = process.env.PORT || 3000;
const server = app.listen(PORT, () => {
  logger.info('Server started', {
    port: PORT,
    env: process.env.NODE_ENV,
    pid: process.pid,
  });
});

server.on('error', (error) => {
  logger.error('Server error', { error: error.message, code: error.code });
  process.exit(1);
});
```

---

## แบบฝึกหัด

### Exercise 1: Log Analysis

1. สร้าง script วิเคราะห์ log files
2. หา top 10 slowest endpoints
3. หา error rate ต่อ endpoint
4. สร้าง daily summary report

### Exercise 2: Real-time Log Dashboard

1. Stream logs ผ่าน WebSocket
2. Frontend dashboard แสดง live logs
3. Filter ตาม level, service, userId
4. Alert เมื่อ error rate สูง

### Exercise 3: Distributed Tracing

1. สร้าง trace ID ที่ propagate ข้าม services
2. Log correlation ระหว่าง services
3. ใช้ OpenTelemetry standard
4. Visualize traces ด้วย Jaeger

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **Logging Levels** — เลือกใช้ให้เหมาะสม
- **Winston** — flexible, production-ready logger
- **Morgan** — HTTP access logging
- **Structured Logging** — JSON format สำหรับ analysis
- **Log Rotation** — จัดการขนาดไฟล์
- **Remote Logging** — ELK, Loki, Datadog
- **Context Logger** — trace request ตลอด lifecycle

---

## สรุปภาพรวม Part 21-30

เราได้ครอบคลุม:

| Part | หัวข้อ | สิ่งสำคัญ |
|------|--------|----------|
| 21 | MongoDB + Mongoose | NoSQL, Schema, CRUD, Aggregation |
| 22 | PostgreSQL + Sequelize | Relational DB, Migrations, Transactions |
| 23 | Authentication Basics | Session, bcrypt, OAuth2 |
| 24 | JWT Authentication | Access/Refresh tokens, Blacklist |
| 25 | Security & Passwords | argon2, Lockout, 2FA |
| 26 | File Uploads | Multer, S3, Sharp |
| 27 | Email Sending | Nodemailer, SendGrid, Queue |
| 28 | Input Validation | Joi, Zod, Sanitization |
| 29 | Error Handling | Custom Errors, Sentry |
| 30 | Logging | Winston, Morgan, Structured Logs |

> **ต่อไป:** Part 31 — Caching with Redis
