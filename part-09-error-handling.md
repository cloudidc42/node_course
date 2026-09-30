# ขั้นตอนที่ 801-900 จาก 1000
# Part 09: Error Handling ใน Node.js

---

## สารบัญ

1. [Error Types ใน Node.js](#error-types)
2. [Try/Catch](#try-catch)
3. [Custom Error Classes](#custom-error-classes)
4. [Async Error Handling](#async-error-handling)
5. [Global Error Handlers](#global-error-handlers)
6. [HTTP Error Responses](#http-error-responses)
7. [Logging Errors](#logging-errors)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Error Types ใน Node.js {#error-types}

### ประเภทของ Errors

```javascript
// 1. SyntaxError - โค้ดผิดไวยากรณ์
// eval('สิ่งที่ไม่ใช่ JavaScript!');
// → SyntaxError: Unexpected token

// 2. ReferenceError - อ้างถึงตัวแปรที่ไม่มี
try {
  console.log(undeclaredVariable);
} catch (err) {
  console.log(err instanceof ReferenceError); // true
  console.log(err.name);    // 'ReferenceError'
  console.log(err.message); // 'undeclaredVariable is not defined'
}

// 3. TypeError - ใช้ค่าผิดประเภท
try {
  null.property; // ไม่สามารถอ่าน property จาก null ได้
} catch (err) {
  console.log(err instanceof TypeError); // true
}

// 4. RangeError - ค่าอยู่นอกช่วงที่ถูกต้อง
try {
  new Array(-1); // ขนาด array ติดลบ
} catch (err) {
  console.log(err instanceof RangeError); // true
}

// 5. URIError - URI ไม่ถูกต้อง
try {
  decodeURIComponent('%');
} catch (err) {
  console.log(err instanceof URIError); // true
}

// 6. EvalError - ปัญหาใน eval()
// ไม่ค่อยพบในโค้ดสมัยใหม่

// 7. Error (base class) - สร้าง error เอง
const err = new Error('บางอย่างผิดพลาด');
console.log(err.message); // 'บางอย่างผิดพลาด'
console.log(err.stack);   // stack trace
```

### System Errors (Node.js เฉพาะ)

```javascript
const fs = require('fs');

// System errors มี .code property
fs.readFile('./ไม่มีไฟล์นี้.txt', (err) => {
  if (err) {
    console.log(err.code);    // 'ENOENT'
    console.log(err.message); // "ENOENT: no such file or directory, open './ไม่มีไฟล์นี้.txt'"
    console.log(err.syscall); // 'open'
    console.log(err.path);    // './ไม่มีไฟล์นี้.txt'
  }
});

// Error codes ที่พบบ่อย:
const errorCodes = {
  'ENOENT':     'ไม่พบไฟล์หรือ directory',
  'EACCES':     'ไม่มีสิทธิ์เข้าถึง',
  'EEXIST':     'ไฟล์/directory มีอยู่แล้ว',
  'EISDIR':     'เป็น directory ไม่ใช่ไฟล์',
  'ENOTDIR':    'ไม่ใช่ directory',
  'EADDRINUSE': 'Port นี้ถูกใช้งานอยู่แล้ว',
  'ECONNREFUSED': 'ปฏิเสธการเชื่อมต่อ',
  'ETIMEDOUT':  'หมดเวลาเชื่อมต่อ',
  'ECONNRESET': 'การเชื่อมต่อถูกรีเซ็ต'
};

function handleFSError(err) {
  const description = errorCodes[err.code] || 'ข้อผิดพลาดระบบไฟล์';
  return new Error(`${description}: ${err.path || ''}`);
}
```

### Error Properties

```javascript
const err = new Error('ตัวอย่าง error');

// Properties มาตรฐาน
console.log(err.name);     // 'Error'
console.log(err.message);  // 'ตัวอย่าง error'
console.log(err.stack);    // stack trace (multiline string)

// Parse stack trace
const stackLines = err.stack.split('\n');
console.log(stackLines[0]);  // Error: ตัวอย่าง error
console.log(stackLines[1]);  // at Object.<anonymous> (/path/to/file.js:1:13)

// เพิ่ม custom properties
err.code = 'MY_ERROR';
err.statusCode = 400;
err.details = { field: 'email', issue: 'invalid format' };
```

---

## Try/Catch {#try-catch}

### พื้นฐาน Try/Catch/Finally

```javascript
// รูปแบบพื้นฐาน
function divide(a, b) {
  try {
    if (typeof a !== 'number' || typeof b !== 'number') {
      throw new TypeError('ต้องการตัวเลข');
    }
    if (b === 0) {
      throw new RangeError('ไม่สามารถหารด้วยศูนย์');
    }
    return a / b;
  } catch (err) {
    if (err instanceof TypeError) {
      console.error('ประเภทข้อมูลไม่ถูกต้อง:', err.message);
      return null;
    }
    if (err instanceof RangeError) {
      console.error('ค่าผิดช่วง:', err.message);
      return Infinity;
    }
    // re-throw errors ที่ไม่ได้คาดไว้
    throw err;
  } finally {
    // ทำงานเสมอ ไม่ว่า try หรือ catch จะทำงาน
    console.log('การหารเสร็จสิ้น');
  }
}

console.log(divide(10, 2));    // การหารเสร็จสิ้น → 5
console.log(divide(10, 0));    // การหารเสร็จสิ้น → Infinity
console.log(divide('a', 2));   // การหารเสร็จสิ้น → null
```

### Try/Catch ใน async Functions

```javascript
const fs = require('fs').promises;

async function readConfig(filePath) {
  let fileHandle = null;
  try {
    fileHandle = await fs.open(filePath, 'r');
    const content = await fileHandle.readFile({ encoding: 'utf8' });
    return JSON.parse(content);
  } catch (err) {
    if (err.code === 'ENOENT') {
      console.warn(`ไม่พบไฟล์ config: ${filePath}, ใช้ค่า default`);
      return {}; // คืน default config
    }
    if (err instanceof SyntaxError) {
      throw new Error(`ไฟล์ config ไม่ใช่ JSON ที่ถูกต้อง: ${filePath}`);
    }
    throw err;
  } finally {
    // ปิด file handle เสมอ แม้เกิด error
    if (fileHandle) {
      await fileHandle.close().catch(console.error);
    }
  }
}
```

### การ Re-throw Errors อย่างถูกต้อง

```javascript
// ❌ ผิด: กลืน error (swallow error)
async function bad() {
  try {
    await riskyOperation();
  } catch (err) {
    // ทำอะไรไม่ถูกต้อง: catch แล้วไม่ทำอะไร
    // หรือ console.log แล้วไม่ throw
    console.log('เกิด error'); // caller ไม่รู้ว่าล้มเหลว!
  }
}

// ✅ ถูกต้อง: จัดการ error ที่จัดการได้ re-throw ที่จัดการไม่ได้
async function good() {
  try {
    await riskyOperation();
  } catch (err) {
    // จัดการเฉพาะ errors ที่เรารับผิดชอบ
    if (err.code === 'NETWORK_TIMEOUT') {
      // ลองใหม่
      return await riskyOperation();
    }
    // re-throw errors อื่นๆ
    throw err;
  }
}

// ✅ Error wrapping: เพิ่ม context ก่อน re-throw
async function withContext() {
  try {
    const data = await fetchUserData();
    return processData(data);
  } catch (err) {
    // เพิ่ม context ให้ error
    const wrappedError = new Error(`ล้มเหลวในการโหลดข้อมูลผู้ใช้: ${err.message}`);
    wrappedError.cause = err; // ES2022: เก็บ original error
    wrappedError.code = err.code;
    throw wrappedError;
  }
}
```

---

## Custom Error Classes {#custom-error-classes}

### การสร้าง Custom Error Classes

```javascript
// Base Custom Error
class AppError extends Error {
  constructor(message, options = {}) {
    super(message);
    this.name = this.constructor.name;  // ใช้ชื่อ class
    this.statusCode = options.statusCode || 500;
    this.code = options.code || 'INTERNAL_ERROR';
    this.isOperational = options.isOperational !== false; // คาดว่าจะเกิด
    this.details = options.details || null;
    this.timestamp = new Date().toISOString();

    // ซ่อน class ออกจาก stack trace
    if (Error.captureStackTrace) {
      Error.captureStackTrace(this, this.constructor);
    }
  }

  toJSON() {
    return {
      name: this.name,
      message: this.message,
      code: this.code,
      statusCode: this.statusCode,
      details: this.details,
      timestamp: this.timestamp
    };
  }
}

// HTTP Errors
class HttpError extends AppError {
  constructor(statusCode, message, options = {}) {
    super(message, { ...options, statusCode });
    this.code = options.code || `HTTP_${statusCode}`;
  }
}

class BadRequestError extends HttpError {
  constructor(message = 'คำร้องขอไม่ถูกต้อง', options = {}) {
    super(400, message, { ...options, code: options.code || 'BAD_REQUEST' });
  }
}

class UnauthorizedError extends HttpError {
  constructor(message = 'ไม่ได้รับการยืนยันตัวตน', options = {}) {
    super(401, message, { ...options, code: options.code || 'UNAUTHORIZED' });
  }
}

class ForbiddenError extends HttpError {
  constructor(message = 'ไม่มีสิทธิ์เข้าถึง', options = {}) {
    super(403, message, { ...options, code: options.code || 'FORBIDDEN' });
  }
}

class NotFoundError extends HttpError {
  constructor(resource = 'ทรัพยากร', options = {}) {
    super(404, `ไม่พบ${resource}`, { ...options, code: options.code || 'NOT_FOUND' });
    this.resource = resource;
  }
}

class ConflictError extends HttpError {
  constructor(message = 'ข้อมูลซ้ำกัน', options = {}) {
    super(409, message, { ...options, code: options.code || 'CONFLICT' });
  }
}

class ValidationError extends BadRequestError {
  constructor(errors, message = 'ข้อมูลไม่ถูกต้อง') {
    super(message, {
      code: 'VALIDATION_ERROR',
      details: errors
    });
    this.errors = errors;
  }
}

class DatabaseError extends AppError {
  constructor(message, options = {}) {
    super(message, {
      ...options,
      statusCode: 500,
      code: 'DATABASE_ERROR'
    });
  }
}

class NetworkError extends AppError {
  constructor(message, options = {}) {
    super(message, {
      ...options,
      statusCode: 503,
      code: 'NETWORK_ERROR'
    });
  }
}

// ตัวอย่างการใช้งาน
function getUserById(id) {
  if (!id || typeof id !== 'number') {
    throw new ValidationError([
      { field: 'id', message: 'id ต้องเป็นตัวเลขที่มากกว่า 0' }
    ]);
  }

  const user = db.findById(id);
  if (!user) {
    throw new NotFoundError(`ผู้ใช้ ID ${id}`);
  }

  return user;
}

// การตรวจสอบ error type
try {
  const user = getUserById('invalid');
} catch (err) {
  if (err instanceof ValidationError) {
    console.log('Validation errors:', err.errors);
  } else if (err instanceof NotFoundError) {
    console.log('ไม่พบข้อมูล:', err.message);
  } else if (err instanceof AppError && err.isOperational) {
    console.log('Operational error:', err.message);
  } else {
    // Programmer error - ต้อง investigate
    console.error('Unexpected error:', err);
    process.exit(1);
  }
}
```

### Error Factory

```javascript
// Factory สำหรับสร้าง errors อย่างสม่ำเสมอ
class ErrorFactory {
  static validation(errors) {
    return new ValidationError(errors);
  }

  static notFound(resource, id) {
    return new NotFoundError(`${resource} (ID: ${id})`);
  }

  static unauthorized(reason) {
    return new UnauthorizedError(reason);
  }

  static forbidden(action, resource) {
    return new ForbiddenError(
      `ไม่มีสิทธิ์ ${action} ${resource}`
    );
  }

  static database(operation, originalError) {
    const err = new DatabaseError(
      `Database ${operation} ล้มเหลว: ${originalError.message}`
    );
    err.cause = originalError;
    return err;
  }

  static fromCode(code, message, details) {
    const errorMap = {
      'VALIDATION_ERROR': () => new ValidationError(details, message),
      'NOT_FOUND': () => new NotFoundError(message),
      'UNAUTHORIZED': () => new UnauthorizedError(message),
      'FORBIDDEN': () => new ForbiddenError(message),
      'CONFLICT': () => new ConflictError(message),
    };

    const factory = errorMap[code];
    if (!factory) {
      return new AppError(message, { code });
    }
    return factory();
  }
}
```

---

## Async Error Handling {#async-error-handling}

### Promise Error Handling

```javascript
// ✅ ถูกต้อง: ต้อง handle errors เสมอ
async function fetchData(url) {
  const response = await fetch(url);

  if (!response.ok) {
    throw new HttpError(
      response.status,
      `HTTP Error: ${response.statusText}`
    );
  }

  return response.json();
}

// การใช้งาน
fetchData('/api/users')
  .then(users => console.log(users))
  .catch(err => {
    if (err instanceof HttpError && err.statusCode === 404) {
      console.log('ไม่พบข้อมูล');
    } else {
      console.error('เกิดข้อผิดพลาด:', err.message);
    }
  });
```

### Error Boundary สำหรับ Express

```javascript
const express = require('express');
const app = express();

// Async Error Wrapper: ห่อ async route handlers
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// ใช้งาน
app.get('/users/:id', asyncHandler(async (req, res) => {
  // Error จะถูกส่งไปที่ error middleware อัตโนมัติ
  const user = await getUserById(parseInt(req.params.id));
  res.json(user);
}));

// Error Handling Middleware
app.use((err, req, res, next) => {
  // Log error
  console.error(err);

  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      success: false,
      error: {
        code: err.code,
        message: err.message,
        ...(err.details && { details: err.details })
      }
    });
  }

  // Unknown error
  res.status(500).json({
    success: false,
    error: {
      code: 'INTERNAL_SERVER_ERROR',
      message: process.env.NODE_ENV === 'production'
        ? 'เกิดข้อผิดพลาดภายในระบบ'
        : err.message
    }
  });
});
```

### การจัดการ Error ใน Event Handlers

```javascript
const EventEmitter = require('events');

class SafeEmitter extends EventEmitter {
  safeEmit(event, ...args) {
    try {
      // ถ้าไม่มี 'error' listener และ emit 'error' จะ crash!
      if (event === 'error' && !this.listenerCount('error')) {
        throw args[0]; // re-throw โดยตรง
      }
      return this.emit(event, ...args);
    } catch (err) {
      if (this.listenerCount('error')) {
        return this.emit('error', err);
      }
      throw err;
    }
  }
}

// ✅ ต้องมี error handler สำหรับ EventEmitter เสมอ
const emitter = new EventEmitter();

// ถ้าไม่มี handler นี้ และ 'error' ถูก emit จะ crash!
emitter.on('error', (err) => {
  console.error('Emitter error:', err.message);
});
```

---

## Global Error Handlers {#global-error-handlers}

### Process-level Error Handlers

```javascript
// global-error-handler.js

// 1. Uncaught Exceptions (synchronous errors ที่ไม่ถูก catch)
process.on('uncaughtException', (err) => {
  // ⚠️ สถานะ application อาจไม่ stable หลังจากนี้
  logger.fatal({
    type: 'uncaughtException',
    error: err.message,
    stack: err.stack
  });

  // ปิด gracefully แล้ว exit
  gracefulShutdown(() => process.exit(1));
});

// 2. Unhandled Promise Rejections
process.on('unhandledRejection', (reason, promise) => {
  logger.error({
    type: 'unhandledRejection',
    reason: reason instanceof Error ? reason.message : reason,
    stack: reason instanceof Error ? reason.stack : undefined
  });

  // ใน Node.js 15+: unhandledRejection จะ crash โดยอัตโนมัติ
  // ควร handle ทุก rejection อย่างชัดเจน
});

// 3. Warning Events
process.on('warning', (warning) => {
  logger.warn({
    type: 'processWarning',
    name: warning.name,
    message: warning.message,
    stack: warning.stack
  });
});

// Graceful Shutdown Function
async function gracefulShutdown(callback) {
  console.log('กำลัง Graceful Shutdown...');

  const shutdownTimeout = setTimeout(() => {
    console.error('Graceful shutdown timeout - Force exit');
    process.exit(1);
  }, 10000); // 10 วินาที timeout

  try {
    // ปิด connections ต่างๆ
    await Promise.all([
      server?.close(),
      dbConnection?.close(),
      redisClient?.quit()
    ]);
    console.log('ปิด connections ทั้งหมดแล้ว');
    clearTimeout(shutdownTimeout);
    callback?.();
  } catch (err) {
    console.error('Error during shutdown:', err);
    clearTimeout(shutdownTimeout);
    callback?.();
  }
}

// Handle SIGTERM (docker stop, kubernetes pod termination)
process.on('SIGTERM', async () => {
  console.log('ได้รับ SIGTERM');
  await gracefulShutdown(() => process.exit(0));
});

// Handle SIGINT (Ctrl+C)
process.on('SIGINT', async () => {
  console.log('\nได้รับ SIGINT');
  await gracefulShutdown(() => process.exit(0));
});
```

### Domain (เก่า - ไม่แนะนำ แต่ควรรู้จัก)

```javascript
// Domain เป็น legacy feature ที่ยังพบในโค้ดเก่า
// ไม่แนะนำในโค้ดใหม่ แต่ควรรู้จัก
const domain = require('domain');
const d = domain.create();

d.on('error', (err) => {
  console.error('Domain caught:', err.message);
});

d.run(() => {
  // async operations ใน domain นี้จะถูก catch
  setTimeout(() => {
    throw new Error('async error ใน domain');
  }, 100);
});
```

---

## HTTP Error Responses {#http-error-responses}

### Express Error Middleware

```javascript
const express = require('express');
const app = express();

app.use(express.json());

// Routes
app.get('/users/:id', async (req, res, next) => {
  try {
    const id = parseInt(req.params.id);

    if (isNaN(id) || id <= 0) {
      throw new ValidationError([{
        field: 'id',
        message: 'id ต้องเป็นตัวเลขที่มากกว่า 0'
      }]);
    }

    const user = await userService.findById(id);

    if (!user) {
      throw new NotFoundError(`ผู้ใช้ ID ${id}`);
    }

    res.json({ success: true, data: user });
  } catch (err) {
    next(err); // ส่งไป error middleware
  }
});

// 404 Handler (ต้องอยู่หลัง routes ทั้งหมด)
app.use((req, res, next) => {
  next(new NotFoundError(`เส้นทาง ${req.path}`));
});

// Error Handler Middleware (ต้องมี 4 parameters!)
app.use((err, req, res, next) => {
  const isDev = process.env.NODE_ENV === 'development';

  // Log error
  if (err.statusCode >= 500 || !err.isOperational) {
    logger.error({
      error: err.message,
      stack: err.stack,
      url: req.originalUrl,
      method: req.method,
      ip: req.ip
    });
  }

  // ตอบ client
  const statusCode = err.statusCode || 500;

  res.status(statusCode).json({
    success: false,
    error: {
      code: err.code || 'INTERNAL_ERROR',
      message: err.isOperational
        ? err.message
        : (isDev ? err.message : 'เกิดข้อผิดพลาดภายในระบบ'),
      ...(err.details && { details: err.details }),
      ...(isDev && { stack: err.stack })
    },
    meta: {
      timestamp: new Date().toISOString(),
      requestId: req.id // ถ้าใช้ express-request-id
    }
  });
});
```

### Standard HTTP Error Response Format

```javascript
// รูปแบบ error response ที่สม่ำเสมอ
function createErrorResponse(err, req) {
  const response = {
    success: false,
    error: {
      code: err.code || 'UNKNOWN_ERROR',
      message: err.message,
      timestamp: new Date().toISOString()
    }
  };

  // เพิ่มข้อมูลเพิ่มเติมสำหรับ validation errors
  if (err instanceof ValidationError) {
    response.error.fields = err.errors.reduce((acc, e) => {
      acc[e.field] = e.message;
      return acc;
    }, {});
  }

  // เพิ่ม debug info ใน development
  if (process.env.NODE_ENV === 'development') {
    response.debug = {
      stack: err.stack,
      method: req.method,
      url: req.originalUrl,
      body: req.body
    };
  }

  return response;
}

// ตัวอย่าง responses:
// 400 Bad Request:
// {
//   "success": false,
//   "error": {
//     "code": "VALIDATION_ERROR",
//     "message": "ข้อมูลไม่ถูกต้อง",
//     "fields": {
//       "email": "รูปแบบ email ไม่ถูกต้อง",
//       "age": "ต้องมีอายุมากกว่า 18 ปี"
//     }
//   }
// }

// 404 Not Found:
// {
//   "success": false,
//   "error": {
//     "code": "NOT_FOUND",
//     "message": "ไม่พบผู้ใช้ ID 999",
//     "timestamp": "2024-01-15T10:30:00.000Z"
//   }
// }
```

### Input Validation Middleware

```javascript
const { body, param, query, validationResult } = require('express-validator');

// Validation rules
const validateCreateUser = [
  body('name')
    .trim()
    .isLength({ min: 2, max: 100 })
    .withMessage('ชื่อต้องมีความยาว 2-100 ตัวอักษร'),

  body('email')
    .isEmail()
    .normalizeEmail()
    .withMessage('รูปแบบ email ไม่ถูกต้อง'),

  body('age')
    .isInt({ min: 18, max: 120 })
    .withMessage('อายุต้องอยู่ระหว่าง 18-120 ปี'),

  body('password')
    .isLength({ min: 8 })
    .matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
    .withMessage('รหัสผ่านต้องมีอย่างน้อย 8 ตัว มีตัวพิมพ์ใหญ่ ตัวพิมพ์เล็ก และตัวเลข')
];

// Validation middleware
const validate = (validations) => async (req, res, next) => {
  await Promise.all(validations.map(v => v.run(req)));

  const errors = validationResult(req);
  if (errors.isEmpty()) {
    return next();
  }

  const formattedErrors = errors.array().map(err => ({
    field: err.path,
    message: err.msg,
    value: err.value
  }));

  throw new ValidationError(formattedErrors);
};

// ใช้งาน
app.post('/users',
  validate(validateCreateUser),
  async (req, res) => {
    const user = await userService.create(req.body);
    res.status(201).json({ success: true, data: user });
  }
);
```

---

## Logging Errors {#logging-errors}

### โครงสร้าง Error Log ที่ดี

```javascript
const winston = require('winston');

// สร้าง logger ที่มีโครงสร้างชัดเจน
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'my-app',
    version: process.env.APP_VERSION || '1.0.0',
    environment: process.env.NODE_ENV || 'development'
  },
  transports: [
    // Console (development)
    new winston.transports.Console({
      format: process.env.NODE_ENV === 'development'
        ? winston.format.combine(
            winston.format.colorize(),
            winston.format.simple()
          )
        : winston.format.json()
    }),

    // Error log file
    new winston.transports.File({
      filename: './logs/error.log',
      level: 'error',
      maxsize: 10 * 1024 * 1024, // 10MB
      maxFiles: 5
    }),

    // Combined log file
    new winston.transports.File({
      filename: './logs/combined.log',
      maxsize: 50 * 1024 * 1024, // 50MB
      maxFiles: 10
    })
  ]
});

// Helper functions สำหรับ logging errors
function logError(err, context = {}) {
  const errorInfo = {
    message: err.message,
    code: err.code,
    stack: err.stack,
    isOperational: err.isOperational,
    ...context
  };

  if (err.statusCode >= 500 || !err.isOperational) {
    logger.error('Server Error', errorInfo);
  } else if (err.statusCode >= 400) {
    logger.warn('Client Error', errorInfo);
  } else {
    logger.info('Error', errorInfo);
  }
}

function logRequest(req, res, responseTime) {
  logger.info('HTTP Request', {
    method: req.method,
    url: req.originalUrl,
    statusCode: res.statusCode,
    responseTime: `${responseTime}ms`,
    userAgent: req.headers['user-agent'],
    ip: req.ip,
    userId: req.user?.id
  });
}
```

### Request Logging Middleware

```javascript
const morgan = require('morgan');
const { v4: uuidv4 } = require('uuid');

// เพิ่ม request ID
app.use((req, res, next) => {
  req.id = uuidv4();
  res.setHeader('X-Request-ID', req.id);
  next();
});

// Log request/response
app.use((req, res, next) => {
  const start = Date.now();

  // log เมื่อ response ส่งไป
  res.on('finish', () => {
    const duration = Date.now() - start;
    logger.info('request', {
      requestId: req.id,
      method: req.method,
      url: req.originalUrl,
      status: res.statusCode,
      duration,
      userId: req.user?.id,
      ip: req.ip
    });
  });

  next();
});

// Error logging middleware
app.use((err, req, res, next) => {
  logError(err, {
    requestId: req.id,
    method: req.method,
    url: req.originalUrl,
    userId: req.user?.id,
    body: req.method !== 'GET' ? req.body : undefined
  });

  // ส่ง response
  const statusCode = err.statusCode || 500;
  res.status(statusCode).json(createErrorResponse(err, req));
});
```

### Error Monitoring (Sentry example)

```javascript
const Sentry = require('@sentry/node');

// Initialize Sentry
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.APP_VERSION,
  tracesSampleRate: 0.1, // 10% sampling
  beforeSend(event, hint) {
    const err = hint.originalException;
    // ไม่ส่ง operational errors ไป Sentry
    if (err instanceof AppError && err.isOperational) {
      return null;
    }
    return event;
  }
});

// ใช้งานใน Express
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// Routes...

app.use(Sentry.Handlers.errorHandler({
  shouldHandleError(error) {
    // ส่งเฉพาะ 5xx errors ไป Sentry
    return !error.statusCode || error.statusCode >= 500;
  }
}));

// Error middleware ของเราเอง
app.use((err, req, res, next) => {
  // Sentry จัดการแล้ว ตอบ client ต่อได้เลย
  const statusCode = err.statusCode || 500;
  res.status(statusCode).json(createErrorResponse(err, req));
});
```

---

## แบบฝึกหัด {#แบบฝึกหัด}

### แบบฝึกหัดที่ 1: Error Hierarchy

สร้าง Error class hierarchy สำหรับระบบ e-commerce:

```javascript
// ต้องการ errors:
// - ProductError (base)
//   - ProductNotFoundError
//   - OutOfStockError (มี quantity field)
//   - InvalidProductError (มี validation errors)
// - OrderError (base)
//   - OrderNotFoundError
//   - InvalidOrderStatusTransitionError (from, to states)
//   - PaymentError
//     - InsufficientFundsError
//     - PaymentDeclinedError

// ทดสอบ
function processOrder(orderId, userId) {
  // ใช้ errors ที่สร้างใน scenarios ต่างๆ
}
```

### แบบฝึกหัดที่ 2: Global Error Handler

สร้าง Express app ที่มี:
- Custom error classes ครบถ้วน
- Request logging ด้วย request ID
- Error middleware ที่แยก development/production responses
- Graceful shutdown handler
- ทดสอบด้วย routes ที่ throw errors ต่างๆ

### แบบฝึกหัดที่ 3: Error Recovery System

สร้าง middleware ที่:
- Auto-retry เมื่อเกิด database errors (สูงสุด 3 ครั้ง)
- Circuit breaker สำหรับ external API calls
- Fallback response เมื่อ service ไม่พร้อม
- Log ทุก retry attempt

---

## สรุป

| หัวข้อ | สิ่งสำคัญ |
|--------|-----------|
| Error Types | Standard JS errors + System errors (code property) |
| Custom Errors | extend Error, ตั้ง name, code, statusCode, isOperational |
| Try/Catch | จัดการ errors ที่คาดได้, re-throw ที่ไม่คาด |
| Async Errors | ใช้ try/catch ใน async, handle unhandledRejection |
| Global Handlers | uncaughtException, unhandledRejection, Graceful shutdown |
| HTTP Errors | Standard format, error middleware, validation |
| Logging | Structured logs, request ID, Sentry/monitoring |

---

## ก้าวต่อไป

➡️ **Part 10: Debugging** - เครื่องมือและเทคนิคการ debug Node.js

---
*Node.js/Express.js Course - Part 09 of 20*
*ขั้นตอนที่ 801-900 จาก 1000*
