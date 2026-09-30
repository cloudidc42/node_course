# Part 29: Advanced Error Handling

> ขั้นตอนที่ 29-30 จาก 1000 — การจัดการ Error อย่างมืออาชีพสำหรับ Production

---

## สารบัญ

1. [Error Taxonomy](#error-taxonomy)
2. [Custom Error Classes](#custom-error-classes)
3. [Operational vs Programmer Errors](#operational-vs-programmer-errors)
4. [Centralized Error Handler](#centralized-error-handler)
5. [Async Error Wrapping](#async-error-wrapping)
6. [Error Monitoring (Sentry)](#error-monitoring-sentry)
7. [Practical: Production Error Handling](#practical-production-error-handling)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Error Taxonomy

### ประเภทของ Errors

**1. Operational Errors** — สิ่งที่คาดเดาได้ในระหว่าง runtime:
- ผู้ใช้ส่งข้อมูลผิด (400 Bad Request)
- ไม่พบ resource (404 Not Found)
- Database connection ล้มเหลว
- External API ไม่ตอบสนอง
- ไม่มีสิทธิ์ (401/403)

**2. Programmer Errors** — bugs ในโค้ด:
- TypeError: Cannot read property of undefined
- ReferenceError
- Logic errors
- Stack overflow

### HTTP Status Codes

```javascript
// utils/httpStatus.js
module.exports = {
  // 2xx Success
  OK: 200,
  CREATED: 201,
  NO_CONTENT: 204,
  
  // 3xx Redirect
  MOVED_PERMANENTLY: 301,
  NOT_MODIFIED: 304,
  
  // 4xx Client errors
  BAD_REQUEST: 400,         // ข้อมูลไม่ถูกต้อง
  UNAUTHORIZED: 401,        // ไม่ได้ login
  FORBIDDEN: 403,           // ไม่มีสิทธิ์
  NOT_FOUND: 404,           // ไม่พบ resource
  METHOD_NOT_ALLOWED: 405,
  CONFLICT: 409,            // ข้อมูลซ้ำ
  GONE: 410,                // resource ถูกลบแล้ว
  UNPROCESSABLE: 422,       // validation error
  TOO_MANY_REQUESTS: 429,   // rate limit
  
  // 5xx Server errors
  INTERNAL_ERROR: 500,
  BAD_GATEWAY: 502,
  SERVICE_UNAVAILABLE: 503,
  GATEWAY_TIMEOUT: 504,
};
```

---

## Custom Error Classes

### Base AppError

```javascript
// utils/errors.js
'use strict';

/**
 * Base class สำหรับ operational errors
 */
class AppError extends Error {
  constructor(message, statusCode = 500, code = null) {
    super(message);
    
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true;  // flag สำหรับแยก operational vs programmer
    this.timestamp = new Date().toISOString();
    
    // Capture stack trace ยกเว้น constructor เอง
    Error.captureStackTrace(this, this.constructor);
  }
  
  toJSON() {
    return {
      name: this.name,
      message: this.message,
      statusCode: this.statusCode,
      code: this.code,
      timestamp: this.timestamp,
    };
  }
}

/**
 * 400 Bad Request — ข้อมูลไม่ถูกต้อง
 */
class BadRequestError extends AppError {
  constructor(message = 'ข้อมูลไม่ถูกต้อง', errors = null) {
    super(message, 400, 'BAD_REQUEST');
    this.errors = errors;  // validation errors array
  }
}

/**
 * 401 Unauthorized — ไม่ได้ยืนยันตัวตน
 */
class UnauthorizedError extends AppError {
  constructor(message = 'กรุณาเข้าสู่ระบบก่อน') {
    super(message, 401, 'UNAUTHORIZED');
  }
}

/**
 * 403 Forbidden — ไม่มีสิทธิ์
 */
class ForbiddenError extends AppError {
  constructor(message = 'คุณไม่มีสิทธิ์ดำเนินการนี้') {
    super(message, 403, 'FORBIDDEN');
  }
}

/**
 * 404 Not Found — ไม่พบ resource
 */
class NotFoundError extends AppError {
  constructor(resource = 'Resource') {
    super(`ไม่พบ${resource}`, 404, 'NOT_FOUND');
    this.resource = resource;
  }
}

/**
 * 409 Conflict — ข้อมูลซ้ำ
 */
class ConflictError extends AppError {
  constructor(message = 'ข้อมูลนี้มีอยู่แล้ว', field = null) {
    super(message, 409, 'CONFLICT');
    this.field = field;
  }
}

/**
 * 422 Unprocessable — Validation Error
 */
class ValidationError extends AppError {
  constructor(errors) {
    super('ข้อมูลไม่ถูกต้อง', 422, 'VALIDATION_ERROR');
    this.errors = Array.isArray(errors) ? errors : [{ message: errors }];
  }
}

/**
 * 429 Too Many Requests
 */
class RateLimitError extends AppError {
  constructor(message = 'คำขอมากเกินไป กรุณารอสักครู่', retryAfter = 60) {
    super(message, 429, 'RATE_LIMIT');
    this.retryAfter = retryAfter;
  }
}

/**
 * 500 Internal Server Error
 */
class InternalError extends AppError {
  constructor(message = 'เกิดข้อผิดพลาดภายในระบบ') {
    super(message, 500, 'INTERNAL_ERROR');
    this.isOperational = false;  // programmer error
  }
}

/**
 * Service Unavailable
 */
class ServiceUnavailableError extends AppError {
  constructor(service = 'Service') {
    super(`${service} ไม่พร้อมใช้งาน`, 503, 'SERVICE_UNAVAILABLE');
  }
}

/**
 * Database Error
 */
class DatabaseError extends AppError {
  constructor(message = 'Database error', originalError = null) {
    super(message, 500, 'DATABASE_ERROR');
    this.originalError = originalError;
  }
}

module.exports = {
  AppError,
  BadRequestError,
  UnauthorizedError,
  ForbiddenError,
  NotFoundError,
  ConflictError,
  ValidationError,
  RateLimitError,
  InternalError,
  ServiceUnavailableError,
  DatabaseError,
};
```

### ใช้งาน Custom Errors

```javascript
// controllers/userController.js
const {
  NotFoundError,
  ConflictError,
  ValidationError,
  ForbiddenError,
} = require('../utils/errors');

exports.getUser = async (req, res, next) => {
  try {
    const user = await User.findById(req.params.id);
    
    if (!user) {
      throw new NotFoundError('ผู้ใช้งาน');
    }
    
    res.json({ success: true, data: user });
  } catch (error) {
    next(error);  // ส่งไปยัง error handler
  }
};

exports.createUser = async (req, res, next) => {
  try {
    const existing = await User.findOne({ email: req.body.email });
    if (existing) {
      throw new ConflictError('Email นี้มีผู้ใช้แล้ว', 'email');
    }
    
    const user = await User.create(req.body);
    res.status(201).json({ success: true, data: user });
  } catch (error) {
    next(error);
  }
};
```

---

## Operational vs Programmer Errors

### จำแนก Errors

```javascript
// utils/errorClassifier.js
const { AppError, DatabaseError } = require('./errors');

/**
 * แปลง framework/library errors เป็น AppError
 */
exports.classifyError = (error) => {
  // ถ้าเป็น AppError อยู่แล้ว
  if (error instanceof AppError) return error;
  
  // Mongoose Validation Error
  if (error.name === 'ValidationError') {
    const errors = Object.values(error.errors).map((e) => ({
      field: e.path,
      message: e.message,
    }));
    return new ValidationError(errors);
  }
  
  // Mongoose Duplicate Key
  if (error.code === 11000) {
    const field = Object.keys(error.keyValue)[0];
    return new ConflictError(`${field} นี้มีอยู่แล้ว`, field);
  }
  
  // Mongoose CastError (invalid ObjectId)
  if (error.name === 'CastError') {
    return new BadRequestError(`ID ไม่ถูกต้อง: ${error.value}`);
  }
  
  // JWT Errors
  if (error.name === 'JsonWebTokenError') {
    return new UnauthorizedError('Token ไม่ถูกต้อง');
  }
  if (error.name === 'TokenExpiredError') {
    return new UnauthorizedError('Token หมดอายุแล้ว');
  }
  
  // Sequelize Errors
  if (error.name === 'SequelizeValidationError') {
    const errors = error.errors.map((e) => ({
      field: e.path,
      message: e.message,
    }));
    return new ValidationError(errors);
  }
  if (error.name === 'SequelizeUniqueConstraintError') {
    const field = Object.keys(error.fields)[0];
    return new ConflictError(`${field} นี้มีอยู่แล้ว`, field);
  }
  if (error.name === 'SequelizeForeignKeyConstraintError') {
    return new BadRequestError('ข้อมูล reference ไม่ถูกต้อง');
  }
  
  // Database connection errors
  if (error.code === 'ECONNREFUSED' || error.message?.includes('connect')) {
    return new DatabaseError('ไม่สามารถเชื่อมต่อฐานข้อมูล');
  }
  
  // Multer errors
  if (error.name === 'MulterError') {
    return new BadRequestError(error.message);
  }
  
  // ไม่รู้จัก — programmer error
  return error;
};
```

---

## Centralized Error Handler

```javascript
// middleware/errorHandler.js
const { AppError } = require('../utils/errors');
const { classifyError } = require('../utils/errorClassifier');
const logger = require('../utils/logger');
const Sentry = require('@sentry/node');

/**
 * Global error handler middleware
 * ต้องมี 4 parameters (err, req, res, next)
 */
const errorHandler = (err, req, res, next) => {
  // แปลง error เป็น AppError ถ้ายังไม่ใช่
  const error = classifyError(err);
  
  // Log error
  const logData = {
    message: error.message,
    statusCode: error.statusCode,
    code: error.code,
    isOperational: error.isOperational,
    url: req.originalUrl,
    method: req.method,
    ip: req.ip,
    userId: req.user?._id,
    userAgent: req.headers['user-agent'],
    requestId: req.id,
  };
  
  if (!error.isOperational) {
    // Programmer error — log เต็มๆ พร้อม stack trace
    logger.error('PROGRAMMER ERROR', { ...logData, stack: error.stack });
    
    // ส่งไป Sentry
    Sentry.captureException(err, {
      user: { id: req.user?._id, email: req.user?.email },
      extra: { url: req.originalUrl, method: req.method },
    });
  } else {
    // Operational error — log ระดับ warn หรือ info
    if (error.statusCode >= 500) {
      logger.error('OPERATIONAL ERROR 5xx', logData);
    } else if (error.statusCode >= 400) {
      logger.warn('CLIENT ERROR 4xx', logData);
    }
  }
  
  // สร้าง response
  const response = {
    success: false,
    message: error.message,
    code: error.code,
    ...(error.errors && { errors: error.errors }),
    ...(error.retryAfter && { retryAfter: error.retryAfter }),
  };
  
  // เพิ่ม debug info ใน development
  if (process.env.NODE_ENV === 'development') {
    response.stack = error.stack;
    response.originalError = err !== error ? {
      name: err.name,
      message: err.message,
    } : undefined;
  }
  
  // ถ้า headers ถูกส่งไปแล้ว
  if (res.headersSent) {
    return next(err);
  }
  
  res.status(error.statusCode || 500).json(response);
};

/**
 * 404 handler (ต้องใส่ก่อน errorHandler)
 */
const notFoundHandler = (req, res, next) => {
  const { NotFoundError } = require('../utils/errors');
  next(new NotFoundError(`เส้นทาง ${req.originalUrl}`));
};

module.exports = { errorHandler, notFoundHandler };
```

### ลงทะเบียน Error Handlers ใน app.js

```javascript
// app.js
const express = require('express');
const { errorHandler, notFoundHandler } = require('./middleware/errorHandler');

const app = express();

// ... middleware และ routes ...

// 404 handler — ต้องอยู่หลัง routes ทั้งหมด
app.use(notFoundHandler);

// Error handler — ต้องอยู่สุดท้าย
app.use(errorHandler);

// Unhandled promise rejections
process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled Rejection:', { reason, promise });
  // ใน production ควร crash และ restart
  if (process.env.NODE_ENV === 'production') {
    process.exit(1);
  }
});

// Uncaught exceptions
process.on('uncaughtException', (error) => {
  logger.error('Uncaught Exception:', { error: error.message, stack: error.stack });
  // ต้อง exit เสมอ — state ของ process ไม่น่าไว้ใจแล้ว
  process.exit(1);
});
```

---

## Async Error Wrapping

### asyncHandler Wrapper

```javascript
// utils/asyncHandler.js

/**
 * Wrap async functions เพื่อ forward errors ไปยัง Express error handler
 */
const asyncHandler = (fn) => {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};

module.exports = asyncHandler;

// ใช้งาน
const asyncHandler = require('../utils/asyncHandler');

exports.getUser = asyncHandler(async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) throw new NotFoundError('ผู้ใช้งาน');
  res.json({ success: true, data: user });
  // ไม่ต้องใช้ try/catch
});
```

### Express 5 (Native Async Support)

```bash
npm install express@next  # Express 5 รองรับ async natively
```

```javascript
// Express 5 — async errors ถูก catch อัตโนมัติ
app.get('/users/:id', async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) throw new NotFoundError('ผู้ใช้งาน');
  res.json(user);
});
```

### Retry Logic สำหรับ Transient Errors

```javascript
// utils/retry.js

/**
 * Retry ฟังก์ชันที่ fail ด้วย exponential backoff
 */
exports.withRetry = async (fn, options = {}) => {
  const {
    maxAttempts = 3,
    baseDelay = 1000,
    maxDelay = 10000,
    retryOn = (error) => error.statusCode >= 500 || error.code === 'ECONNREFUSED',
  } = options;
  
  let lastError;
  
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      
      if (attempt === maxAttempts || !retryOn(error)) {
        throw error;
      }
      
      const delay = Math.min(baseDelay * Math.pow(2, attempt - 1), maxDelay);
      const jitter = Math.random() * 0.3 * delay;
      
      logger.warn(`Attempt ${attempt}/${maxAttempts} failed, retrying in ${delay + jitter}ms`);
      
      await new Promise((resolve) => setTimeout(resolve, delay + jitter));
    }
  }
  
  throw lastError;
};

// ใช้งาน
const result = await withRetry(
  () => externalAPI.fetchData(),
  { maxAttempts: 3, baseDelay: 500 }
);
```

---

## Error Monitoring (Sentry)

```bash
npm install @sentry/node @sentry/profiling-node
```

### Sentry Setup

```javascript
// config/sentry.js
const Sentry = require('@sentry/node');
const { nodeProfilingIntegration } = require('@sentry/profiling-node');

if (process.env.SENTRY_DSN && process.env.NODE_ENV === 'production') {
  Sentry.init({
    dsn: process.env.SENTRY_DSN,
    environment: process.env.NODE_ENV,
    release: process.env.APP_VERSION,
    
    integrations: [
      nodeProfilingIntegration(),
      new Sentry.Integrations.Http({ tracing: true }),
      new Sentry.Integrations.Express({ app }),
    ],
    
    tracesSampleRate: 0.1,    // 10% ของ requests
    profilesSampleRate: 0.1,
    
    // กรอง sensitive data
    beforeSend(event, hint) {
      // ลบ password จาก request body
      if (event.request?.data?.password) {
        event.request.data.password = '[FILTERED]';
      }
      
      // ไม่ส่ง 404 errors
      if (event.exception?.values?.[0]?.type === 'NotFoundError') {
        return null;
      }
      
      return event;
    },
  });
}

module.exports = Sentry;
```

### Sentry ใน Express

```javascript
// app.js
const Sentry = require('./config/sentry');
const express = require('express');

const app = express();

// Sentry request handler ต้องเป็นอันแรก
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// ... routes ...

// Sentry error handler ต้องอยู่ก่อน custom error handler
app.use(Sentry.Handlers.errorHandler({
  shouldHandleError(error) {
    // ส่งเฉพาะ 5xx errors
    return error.status >= 500;
  },
}));

app.use(errorHandler);
```

### Manual Error Reporting

```javascript
// ส่ง error พร้อม context
try {
  await riskyOperation();
} catch (error) {
  Sentry.withScope((scope) => {
    scope.setUser({ id: req.user._id, email: req.user.email });
    scope.setTag('operation', 'payment');
    scope.setExtra('orderId', order._id);
    scope.setLevel('error');
    
    Sentry.captureException(error);
  });
  
  throw error;
}
```

---

## Practical: Production Error Handling

### Complete Setup

```javascript
// app.js — Production-ready setup
require('dotenv').config();
const express = require('express');
const Sentry = require('./config/sentry');
const { errorHandler, notFoundHandler } = require('./middleware/errorHandler');
const logger = require('./utils/logger');
const { v4: uuidv4 } = require('uuid');

const app = express();

// Request ID middleware
app.use((req, res, next) => {
  req.id = req.headers['x-request-id'] || uuidv4();
  res.setHeader('X-Request-ID', req.id);
  next();
});

// Sentry
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// ... other middleware ...
// ... routes ...

// 404
app.use(notFoundHandler);

// Sentry error handler
app.use(Sentry.Handlers.errorHandler());

// Custom error handler
app.use(errorHandler);

// Graceful shutdown
const gracefulShutdown = async (signal) => {
  logger.info(`${signal} received, shutting down gracefully`);
  
  server.close(async () => {
    logger.info('HTTP server closed');
    
    await mongoose.connection.close();
    logger.info('Database connection closed');
    
    await Sentry.close(2000);
    
    process.exit(0);
  });
  
  // Force shutdown หลัง 10 วินาที
  setTimeout(() => {
    logger.error('Forced shutdown');
    process.exit(1);
  }, 10000);
};

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

process.on('unhandledRejection', (reason) => {
  logger.error('Unhandled Rejection', { reason });
  if (process.env.NODE_ENV === 'production') {
    gracefulShutdown('UNHANDLED_REJECTION');
  }
});

process.on('uncaughtException', (error) => {
  logger.error('Uncaught Exception', { error: error.message, stack: error.stack });
  gracefulShutdown('UNCAUGHT_EXCEPTION');
});
```

### Health Check Endpoint

```javascript
// routes/health.js
const express = require('express');
const mongoose = require('mongoose');
const { sequelize } = require('../models');
const router = express.Router();

router.get('/health', async (req, res) => {
  const checks = {
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    services: {},
  };
  
  // ตรวจสอบ MongoDB
  try {
    await mongoose.connection.db.admin().ping();
    checks.services.mongodb = { status: 'ok' };
  } catch (error) {
    checks.services.mongodb = { status: 'error', message: error.message };
    checks.status = 'degraded';
  }
  
  // ตรวจสอบ PostgreSQL
  try {
    await sequelize.authenticate();
    checks.services.postgres = { status: 'ok' };
  } catch (error) {
    checks.services.postgres = { status: 'error', message: error.message };
    checks.status = 'degraded';
  }
  
  const statusCode = checks.status === 'ok' ? 200 : 503;
  res.status(statusCode).json(checks);
});

module.exports = router;
```

---

## แบบฝึกหัด

### Exercise 1: Error Recovery

1. สร้าง circuit breaker สำหรับ external API calls
2. Fallback เมื่อ service ไม่พร้อม
3. Health check endpoint
4. Alert เมื่อ error rate สูงเกิน threshold

### Exercise 2: Error Reporting Dashboard

1. Log errors ไปยัง MongoDB
2. สร้าง admin endpoint ดู error stats
3. Top 10 error types
4. Error trend (รายชั่วโมง/วัน)

### Exercise 3: Integration Testing

1. ทดสอบ error handling ทุก error type
2. ตรวจสอบ status codes ถูกต้อง
3. ตรวจสอบ error response format
4. ทดสอบ sensitive data ไม่รั่วไหล

---

## สรุป

- **Custom Error Classes** — structured, meaningful errors
- **Operational vs Programmer errors** — จัดการต่างกัน
- **Centralized handler** — consistent error responses
- **asyncHandler** — ไม่ต้องใช้ try/catch ทุกที่
- **Sentry** — monitor errors ใน production

> **บทถัดไป:** Part 30 — Logging
