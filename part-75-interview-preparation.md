# Part 75: Interview Preparation
## ขั้นตอนที่ 741-750 จาก 1000

---

## เตรียมพร้อมสำหรับ Node.js Interview

---

## 1. Top 50 Node.js Interview Questions

### พื้นฐาน (1-15)

**Q1: Node.js ทำงานอย่างไร? อธิบาย Event Loop**

```
Node.js ใช้ Single-threaded event loop ที่ non-blocking I/O
Event Loop มี phases:
1. timers: setTimeout, setInterval callbacks
2. pending callbacks: I/O error callbacks
3. idle/prepare: internal use
4. poll: ดึง I/O events
5. check: setImmediate callbacks
6. close callbacks: socket.on('close')

ตัวอย่าง:
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));

// Output: nextTick → promise → timeout/immediate (depends on context)
```

**Q2: Callback vs Promise vs Async/Await**

```javascript
// Callback
fs.readFile('file.txt', (err, data) => {
  if (err) throw err;
  console.log(data);
});

// Promise
fs.promises.readFile('file.txt')
  .then(data => console.log(data))
  .catch(err => console.error(err));

// Async/Await (แนะนำสุด)
async function readFile() {
  try {
    const data = await fs.promises.readFile('file.txt');
    console.log(data);
  } catch (err) {
    console.error(err);
  }
}
```

**Q3: อธิบาย require() vs import**

```javascript
// CommonJS (require) - synchronous
const express = require('express');
module.exports = { myFunction };

// ES Modules (import) - asynchronous
import express from 'express';
export { myFunction };

// Dynamic import
const module = await import('./module.js');
```

**Q4: process.nextTick() vs setImmediate() vs setTimeout()**

```javascript
// process.nextTick: ทำงานก่อน event loop phases ถัดไป
// setImmediate: ทำงานใน check phase ของ event loop
// setTimeout(fn, 0): ทำงานใน timers phase

process.nextTick(() => console.log('1 nextTick'));
setImmediate(() => console.log('2 setImmediate'));
setTimeout(() => console.log('3 setTimeout'), 0);
Promise.resolve().then(() => console.log('4 Promise'));

// Output: 1, 4, 3, 2
```

**Q5: Streams ใน Node.js คืออะไร?**

```javascript
// Streams = การอ่าน/เขียน data แบบ chunk-by-chunk
// ประหยัด memory สำหรับ large files

const fs = require('fs');

// Readable Stream
const readStream = fs.createReadStream('large-file.txt');
readStream.on('data', chunk => console.log('Received:', chunk.length));
readStream.on('end', () => console.log('Done'));

// Pipe streams
fs.createReadStream('input.txt')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('output.gz'));

// Transform Stream
const { Transform } = require('stream');
const upperCase = new Transform({
  transform(chunk, encoding, callback) {
    callback(null, chunk.toString().toUpperCase());
  }
});
```

**Q6: Cluster module ทำงานอย่างไร?**

```javascript
const cluster = require('cluster');
const os = require('os');

if (cluster.isPrimary) {
  // สร้าง worker ตามจำนวน CPU cores
  for (let i = 0; i < os.cpus().length; i++) {
    cluster.fork();
  }
  
  cluster.on('exit', worker => {
    console.log(`Worker ${worker.pid} died. Restarting...`);
    cluster.fork();
  });
} else {
  // Worker process
  require('./app').listen(3000);
}
```

**Q7: Middleware ใน Express คืออะไร?**

```javascript
// Middleware = function ที่รับ (req, res, next) และทำงานระหว่าง request-response cycle

// Application middleware
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next(); // ต้องเรียก next() เพื่อส่งต่อ
});

// Route middleware
app.get('/protected', 
  authenticate,  // middleware 1
  authorize,     // middleware 2
  handler        // final handler
);

// Error middleware (4 params)
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});
```

**Q8: อธิบาย Buffer ใน Node.js**

```javascript
// Buffer = fixed-size chunk of memory สำหรับ binary data
const buf = Buffer.from('Hello World');
console.log(buf); // <Buffer 48 65 6c 6c 6f 20 57 6f 72 6c 64>

const buf2 = Buffer.alloc(10); // Buffer ขนาด 10 bytes
buf2.write('Hello');

// แปลง Buffer เป็น string
console.log(buf.toString('utf8'));
console.log(buf.toString('base64'));
console.log(buf.toString('hex'));
```

**Q9: Event Emitter คืออะไร?**

```javascript
const EventEmitter = require('events');

class MyEmitter extends EventEmitter {}
const emitter = new MyEmitter();

emitter.on('event', (data) => console.log('Received:', data));
emitter.emit('event', { message: 'Hello' });

// once - รับครั้งเดียว
emitter.once('login', (user) => console.log('First login:', user));

// removeListener
const handler = () => console.log('Event!');
emitter.on('test', handler);
emitter.removeListener('test', handler);
```

**Q10: อธิบาย REPL ใน Node.js**

```
REPL = Read-Eval-Print-Loop
เป็น interactive shell สำหรับ Node.js

ใช้คำสั่ง: node
> 1 + 1
2
> const x = 'Hello'
> x.toUpperCase()
'HELLO'
```

**Q11: Node.js child_process**

```javascript
const { exec, spawn, fork } = require('child_process');

// exec: ใช้ shell, buffer output
exec('ls -la', (error, stdout) => {
  console.log(stdout);
});

// spawn: stream output, ไม่ใช้ shell (ปลอดภัยกว่า)
const child = spawn('node', ['script.js']);
child.stdout.on('data', data => console.log(data.toString()));

// fork: สร้าง Node.js child process พร้อม IPC
const worker = fork('worker.js');
worker.send({ task: 'process' });
worker.on('message', msg => console.log('Worker result:', msg));
```

**Q12: JWT Token ทำงานอย่างไร?**

```javascript
// JWT = Header.Payload.Signature
// Header: algorithm + type
// Payload: claims (data)
// Signature: HMAC(base64(header) + '.' + base64(payload), secret)

const jwt = require('jsonwebtoken');

// Sign
const token = jwt.sign(
  { userId: '123', role: 'admin' },
  process.env.JWT_SECRET,
  { expiresIn: '1h' }
);

// Verify
const decoded = jwt.verify(token, process.env.JWT_SECRET);
console.log(decoded); // { userId: '123', role: 'admin', iat: ..., exp: ... }
```

**Q13: อธิบาย Mongoose middleware (hooks)**

```javascript
// Pre hooks
userSchema.pre('save', async function() {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
});

// Post hooks
userSchema.post('save', function(doc) {
  console.log('User saved:', doc._id);
});

// Query middleware
postSchema.pre('find', function() {
  this.where({ deletedAt: null });
});
```

**Q14: Memory Leak ใน Node.js คืออะไร และแก้อย่างไร?**

```javascript
// สาเหตุ Memory Leak:
// 1. Global variables ที่ไม่ได้ clear
// 2. Event listeners ที่ไม่ได้ remove
// 3. Closures ที่ reference ข้อมูลขนาดใหญ่
// 4. setTimeout/setInterval ที่ไม่ได้ clear

// ปัญหา
const listeners = [];
app.get('/subscribe', (req, res) => {
  listeners.push(res); // ไม่เคย remove!
});

// แก้ไข
const listeners = new Set();
app.get('/subscribe', (req, res) => {
  listeners.add(res);
  req.on('close', () => listeners.delete(res)); // Remove เมื่อ disconnect
});

// ตรวจสอบ memory
process.memoryUsage();
// { rss, heapTotal, heapUsed, external }
```

**Q15: Node.js Worker Threads**

```javascript
// Worker Threads สำหรับ CPU-intensive tasks
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
  const worker = new Worker(__filename);
  worker.on('message', result => console.log('Result:', result));
  worker.postMessage({ n: 40 });
} else {
  parentPort.on('message', ({ n }) => {
    const result = fibonacci(n);
    parentPort.postMessage(result);
  });
}

function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
```

---

### กลาง (16-30)

**Q16-20: Performance & Optimization**

```javascript
// Q16: Caching strategies
// Memory cache: node-cache, lru-cache
// Redis cache: ioredis
// HTTP cache: Cache-Control headers

// Q17: Database query optimization
// - ใช้ indexes
// - Projection (select only needed fields)
// - Lean queries (toObject ไม่ต้องเรียก)
// - Pagination แทน load all

const users = await User.find().lean().select('name email').limit(20);

// Q18: Connection pooling
// MongoDB: maxPoolSize option
// PostgreSQL: pg pool

// Q19: N+1 Query problem
// ปัญหา
const posts = await Post.find();
for (const post of posts) {
  post.author = await User.findById(post.userId); // N queries!
}

// แก้: populate หรือ join
const posts = await Post.find().populate('author', 'name avatar');

// Q20: Memory optimization
// ใช้ streams สำหรับ large data
// Avoid storing large objects in memory
// Use Buffer pooling
```

**Q21-25: Testing**

```javascript
// Q21: Unit testing
describe('UserService', () => {
  it('should hash password on create', async () => {
    const service = new UserService();
    const user = await service.create({ password: '12345678' });
    expect(user.password).not.toBe('12345678');
    expect(bcrypt.compareSync('12345678', user.password)).toBe(true);
  });
});

// Q22: Integration testing
it('POST /api/users should create user', async () => {
  const response = await request(app)
    .post('/api/users')
    .send({ name: 'Test', email: 'test@test.com', password: '12345678' });
  
  expect(response.status).toBe(201);
  expect(response.body).toHaveProperty('id');
});

// Q23: Mocking
jest.mock('../services/email.service');
const emailService = require('../services/email.service');
emailService.send.mockResolvedValue({ success: true });

// Q24: Test database
// ใช้ mongodb-memory-server สำหรับ test
const { MongoMemoryServer } = require('mongodb-memory-server');

// Q25: Coverage
// jest --coverage
// Istanbul
```

**Q26-30: Architecture**

```javascript
// Q26: MVC Pattern
// Model: business logic + data
// View: presentation layer (สำหรับ API = JSON response)
// Controller: handle requests, coordinate

// Q27: Repository Pattern
class UserRepository {
  async findById(id) { return User.findById(id); }
  async save(user) { return user.save(); }
}

// Q28: Dependency Injection
class UserService {
  constructor(userRepository, emailService) {
    this.userRepository = userRepository;
    this.emailService = emailService;
  }
}

// Q29: Microservices vs Monolith
// Monolith: ง่าย, ราคาถูก, deploy ง่าย แต่ scale ยาก
// Microservices: scale ได้ดี แต่ complex มากกว่า

// Q30: API versioning
app.use('/api/v1', v1Router);
app.use('/api/v2', v2Router);
```

### ขั้นสูง (31-50)

```javascript
// Q31: CQRS - Command Query Responsibility Segregation
// Q32: Event Sourcing
// Q33: Saga Pattern สำหรับ distributed transactions
// Q34: Circuit Breaker Pattern
// Q35: Rate limiting algorithms (token bucket, leaky bucket)
// Q36: CAP Theorem
// Q37: Eventual consistency
// Q38: Message queues (RabbitMQ, Kafka, SQS)
// Q39: GraphQL vs REST
// Q40: gRPC
// Q41: WebSocket scaling
// Q42: Database sharding
// Q43: Read replicas
// Q44: Cache invalidation strategies
// Q45: Blue-green deployment
// Q46: A/B testing
// Q47: Feature flags
// Q48: Zero-downtime migration
// Q49: Observability (metrics, logs, traces)
// Q50: SLO/SLI/SLA
```

---

## 2. System Design Questions

### Q: Design Twitter/X

```
Scale: 100M users, 500M tweets/day

Components:
1. API Gateway
2. User Service
3. Tweet Service
4. Timeline Service (Fan-out)
5. Notification Service
6. Search Service

Key decisions:
- Fan-out on write vs read (hybrid for celebrities)
- Timeline caching ด้วย Redis
- Sharding tweets by tweet_id
- CDN สำหรับ media
- Kafka สำหรับ async processing
```

### Q: Design Rate Limiter

```javascript
// Token Bucket Algorithm
class TokenBucket {
  constructor(capacity, refillRate) {
    this.capacity = capacity;
    this.tokens = capacity;
    this.refillRate = refillRate;
    this.lastRefill = Date.now();
  }

  consume(tokens = 1) {
    this.refill();
    
    if (this.tokens >= tokens) {
      this.tokens -= tokens;
      return true;
    }
    
    return false;
  }

  refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    const refill = elapsed * this.refillRate;
    
    this.tokens = Math.min(this.capacity, this.tokens + refill);
    this.lastRefill = now;
  }
}

// Redis-based distributed rate limiter
async function isRateLimited(key, limit, windowMs) {
  const now = Date.now();
  const window = Math.floor(now / windowMs);
  const redisKey = `rl:${key}:${window}`;
  
  const count = await redis.incr(redisKey);
  
  if (count === 1) {
    await redis.pexpire(redisKey, windowMs * 2);
  }
  
  return count > limit;
}
```

---

## 3. Coding Challenges

```javascript
// Challenge 1: Implement debounce
function debounce(fn, delay) {
  let timer;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Challenge 2: Implement Promise.all
async function myPromiseAll(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let completed = 0;
    
    if (promises.length === 0) return resolve([]);
    
    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(result => {
          results[index] = result;
          if (++completed === promises.length) {
            resolve(results);
          }
        })
        .catch(reject);
    });
  });
}

// Challenge 3: LRU Cache
class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.cache = new Map();
  }

  get(key) {
    if (!this.cache.has(key)) return -1;
    
    const value = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, value); // Move to end
    return value;
  }

  put(key, value) {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.capacity) {
      this.cache.delete(this.cache.keys().next().value);
    }
    
    this.cache.set(key, value);
  }
}
```

---

## 4. Best Practices Recap

```javascript
// 1. Error handling
process.on('uncaughtException', (err) => {
  logger.error('Uncaught Exception:', err);
  process.exit(1);
});

process.on('unhandledRejection', (reason) => {
  logger.error('Unhandled Rejection:', reason);
  process.exit(1);
});

// 2. Graceful shutdown
process.on('SIGTERM', async () => {
  await server.close();
  await mongoose.connection.close();
  process.exit(0);
});

// 3. Environment configuration
const config = {
  port: parseInt(process.env.PORT) || 3000,
  db: {
    url: process.env.DATABASE_URL,
    poolSize: parseInt(process.env.DB_POOL_SIZE) || 10
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN || '1h'
  }
};

// 4. Structured logging
const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [new winston.transports.Console()]
});
```

---

## 5. Career Advice

### สิ่งที่ interviewer มองหา

```
Technical Skills:
- เข้าใจ fundamentals (async/await, event loop, streams)
- Clean code, proper error handling
- Security awareness
- Testing mindset

Soft Skills:
- Communication ชัดเจน
- Problem-solving approach
- อธิบาย trade-offs ได้
- เรียนรู้จาก mistakes

System Design:
- ถามข้อมูลก่อน design
- Start simple แล้วค่อย scale
- พูด trade-offs ทุกการตัดสินใจ
- เข้าใจ bottlenecks
```

### Tips สำหรับ Interview

```
1. อ่านโจทย์ให้ละเอียดก่อน code
2. พูดความคิดออกมาตลอด
3. ถามเพื่อ clarify requirements
4. เริ่ม brute force แล้วค่อย optimize
5. Test code ด้วย edge cases
6. พูดถึง time/space complexity
```

---

## สรุป

Node.js interview ต้องการความเข้าใจลึกใน event loop, async programming, และ Node.js-specific concepts ควบคู่กับ general software engineering principles เช่น SOLID, design patterns, และ system design การฝึก coding challenges เป็นประจำและอ่าน documentation อย่างสม่ำเสมอจะทำให้ interview ผ่านได้ง่ายขึ้น

> ขั้นตอนต่อไป: Part 76 - Prisma ORM
