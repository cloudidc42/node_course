# ขั้นตอนที่ 701-800 จาก 1000
# Part 08: Async Programming ใน Node.js

---

## สารบัญ

1. [บทนำ: Asynchronous Programming](#บทนำ)
2. [Callbacks](#callbacks)
3. [Promises](#promises)
4. [async/await](#async-await)
5. [Promise Combinators](#promise-combinators)
6. [Error Handling Patterns](#error-handling-patterns)
7. [Async Iterators](#async-iterators)
8. [Practical Patterns](#practical-patterns)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## บทนำ: Asynchronous Programming {#บทนำ}

Node.js ทำงานบน single thread แต่ใช้ event loop เพื่อจัดการ I/O แบบ non-blocking

```
Synchronous (รอ):                 Asynchronous (ไม่รอ):
─────────────────────             ─────────────────────────────
อ่านไฟล์    →  รอ...             อ่านไฟล์  →  ทำงานอื่น
                                             ↓
           ↓                      [เสร็จแล้ว] ← callback/promise
         ทำต่อ                              ↓
                                          ทำต่อ
```

### วิวัฒนาการของ Async ใน JavaScript

```
Callbacks (Node.js 0.x)
  ↓ ซับซ้อนเกิน, callback hell
Promises (ES6 / 2015)
  ↓ ดีขึ้น แต่ chain ยาว
async/await (ES8 / 2017)
  ↓ อ่านง่ายที่สุด แต่ต้องเข้าใจ underneath
```

---

## Callbacks {#callbacks}

Callback คือ function ที่ส่งเป็น argument ให้ function อื่น และถูกเรียกเมื่องานเสร็จ

### Node.js Error-First Callback Convention

```javascript
// รูปแบบมาตรฐานของ Node.js:
// callback(error, result)
// - error: null ถ้าสำเร็จ, Error object ถ้าล้มเหลว
// - result: ผลลัพธ์ (ถ้าสำเร็จ)

const fs = require('fs');

// การอ่านไฟล์แบบ callback
fs.readFile('./data.txt', 'utf8', (err, data) => {
  if (err) {
    // จัดการ error ก่อนเสมอ
    console.error('ไม่สามารถอ่านไฟล์:', err.message);
    return; // หยุดการทำงาน
  }
  // ถ้าไม่มี error ค่อยจัดการ data
  console.log('เนื้อหาไฟล์:', data);
});

console.log('โค้ดนี้ทำงานก่อน readFile เสร็จ!'); // ทำงานก่อน
```

### Callback Hell (ปัญหาหลัก)

```javascript
// ❌ Callback Hell - อ่านยาก, debug ยาก
fs.readFile('./config.json', 'utf8', (err, configData) => {
  if (err) return console.error(err);

  const config = JSON.parse(configData);

  db.connect(config.db, (err, connection) => {
    if (err) return console.error(err);

    connection.query('SELECT * FROM users', (err, users) => {
      if (err) return console.error(err);

      users.forEach(user => {
        emailService.send(user.email, 'สวัสดี!', (err, result) => {
          if (err) return console.error(err);

          console.log(`ส่ง email ให้ ${user.name} แล้ว`);

          db.updateUser(user.id, { emailSent: true }, (err) => {
            if (err) return console.error(err);
            console.log(`อัพเดต user ${user.id} แล้ว`);
          });
        });
      });
    });
  });
});
// ปัญหา: ยากต่อการอ่าน, error handling ซ้ำๆ,
//        ยากต่อการ test, ยากต่อการ refactor
```

### การหนีจาก Callback Hell

```javascript
// ✅ วิธีแก้: แยก functions ออก + ตั้งชื่อ
function readConfig(callback) {
  fs.readFile('./config.json', 'utf8', (err, data) => {
    if (err) return callback(new Error(`อ่าน config ล้มเหลว: ${err.message}`));
    try {
      callback(null, JSON.parse(data));
    } catch (e) {
      callback(new Error('JSON ไม่ถูกต้อง'));
    }
  });
}

function connectDB(config, callback) {
  db.connect(config.db, (err, conn) => {
    if (err) return callback(new Error(`เชื่อมต่อ DB ล้มเหลว: ${err.message}`));
    callback(null, conn);
  });
}

function getUsers(connection, callback) {
  connection.query('SELECT * FROM users', callback);
}

function sendEmailToUser(user, callback) {
  emailService.send(user.email, 'สวัสดี!', callback);
}

// การใช้งานที่อ่านง่ายกว่า
readConfig((err, config) => {
  if (err) return console.error(err);
  connectDB(config, (err, conn) => {
    if (err) return console.error(err);
    getUsers(conn, (err, users) => {
      if (err) return console.error(err);
      // ยังมี nesting อยู่ แต่ดีขึ้น
    });
  });
});
```

### Promisify Callbacks

```javascript
const { promisify } = require('util');
const fs = require('fs');

// แปลง callback-based function เป็น Promise
const readFileAsync = promisify(fs.readFile);
const writeFileAsync = promisify(fs.writeFile);

// ใช้งานกับ Promise
readFileAsync('./data.txt', 'utf8')
  .then(data => console.log(data))
  .catch(err => console.error(err));

// หรือ manual promisify
function promisifyManual(fn) {
  return function(...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (err, result) => {
        if (err) reject(err);
        else resolve(result);
      });
    });
  };
}

const myReadFile = promisifyManual(fs.readFile);
```

---

## Promises {#promises}

Promise คือ object ที่แทน "ผลลัพธ์ที่จะเกิดขึ้นในอนาคต"

```
Promise States:
  pending  →  fulfilled (มีค่า)
  pending  →  rejected  (มี error)
```

### การสร้างและใช้งาน Promise

```javascript
// สร้าง Promise
function delay(ms) {
  return new Promise((resolve, reject) => {
    if (ms < 0) {
      reject(new Error('ms ต้องเป็นค่าบวก'));
      return;
    }
    setTimeout(resolve, ms);
  });
}

// ใช้งาน Promise chain
delay(1000)
  .then(() => {
    console.log('รอ 1 วินาที');
    return delay(500);
  })
  .then(() => {
    console.log('รออีก 0.5 วินาที');
    return 'เสร็จแล้ว!';
  })
  .then(result => {
    console.log(result);
  })
  .catch(err => {
    console.error('Error:', err.message);
  })
  .finally(() => {
    console.log('ทำงานเสมอ ไม่ว่าจะสำเร็จหรือล้มเหลว');
  });
```

### Promise Chaining

```javascript
// ตัวอย่างจริง: ดึงข้อมูลผู้ใช้และ orders
function fetchUser(userId) {
  return fetch(`/api/users/${userId}`)
    .then(res => {
      if (!res.ok) throw new Error(`HTTP error! status: ${res.status}`);
      return res.json();
    });
}

function fetchOrders(userId) {
  return fetch(`/api/users/${userId}/orders`).then(res => res.json());
}

function calculateTotal(orders) {
  return orders.reduce((sum, order) => sum + order.total, 0);
}

// Promise chain ที่อ่านได้
fetchUser(123)
  .then(user => {
    console.log('User:', user.name);
    return fetchOrders(user.id); // คืน Promise ใหม่
  })
  .then(orders => {
    console.log(`Orders จำนวน: ${orders.length}`);
    return calculateTotal(orders); // คืนค่าปกติ (ถูกห่อใน Promise อัตโนมัติ)
  })
  .then(total => {
    console.log(`ยอดรวม: ${total} บาท`);
  })
  .catch(err => {
    // catch ทุก error ใน chain
    console.error('ข้อผิดพลาด:', err.message);
  });
```

### Promise States และ Anti-Patterns

```javascript
// ❌ Anti-pattern: Nested Promises (Promise Hell)
fetchUser(123).then(user => {
  fetchOrders(user.id).then(orders => { // อย่าทำแบบนี้!
    calculateTotal(orders).then(total => {
      console.log(total);
    });
  });
});

// ✅ ถูกต้อง: Return Promises เพื่อ chain
fetchUser(123)
  .then(user => fetchOrders(user.id))  // return Promise
  .then(orders => calculateTotal(orders))
  .then(console.log);

// ❌ Anti-pattern: ลืม return
fetchUser(123)
  .then(user => {
    fetchOrders(user.id); // ลืม return! orders จะไม่ส่งไปบรรทัดถัดไป
  })
  .then(orders => {
    // orders จะเป็น undefined!
    console.log(orders);
  });

// ❌ Anti-pattern: สร้าง Promise ซ้อน Promise
function bad() {
  return new Promise((resolve, reject) => {
    fetchUser(123).then(user => resolve(user)); // ไม่จำเป็น
  });
}

// ✅ ถูกต้อง: คืน Promise โดยตรง
function good() {
  return fetchUser(123); // ง่ายกว่ามาก
}
```

---

## async/await {#async-await}

`async/await` คือ syntax sugar ของ Promises ทำให้โค้ด async อ่านเหมือน synchronous

### พื้นฐาน async/await

```javascript
// async function คืนค่าเป็น Promise เสมอ
async function greet(name) {
  return `สวัสดี, ${name}!`; // ถูกห่อใน Promise อัตโนมัติ
}

// เหมือนกับ:
function greetPromise(name) {
  return Promise.resolve(`สวัสดี, ${name}!`);
}

// await ใช้ได้แค่ใน async function
async function main() {
  const message = await greet('สมชาย');
  console.log(message); // สวัสดี, สมชาย!

  // await หยุดรอ Promise จนสำเร็จ
  const result = await delay(1000);
  console.log('รอ 1 วินาทีเสร็จแล้ว');
}

main();
```

### เปรียบเทียบ Promise กับ async/await

```javascript
const fs = require('fs').promises;

// แบบ Promise chain
function readAndProcess_Promise(filename) {
  return fs.readFile(filename, 'utf8')
    .then(data => data.split('\n'))
    .then(lines => lines.filter(line => line.trim()))
    .then(lines => lines.map(line => line.toUpperCase()))
    .then(lines => lines.join('\n'))
    .then(result => {
      console.log('ผลลัพธ์:', result);
      return result;
    })
    .catch(err => {
      console.error('Error:', err.message);
      throw err;
    });
}

// แบบ async/await (อ่านง่ายกว่ามาก!)
async function readAndProcess_Async(filename) {
  try {
    const data = await fs.readFile(filename, 'utf8');
    const lines = data.split('\n');
    const nonEmpty = lines.filter(line => line.trim());
    const upperCased = nonEmpty.map(line => line.toUpperCase());
    const result = upperCased.join('\n');
    console.log('ผลลัพธ์:', result);
    return result;
  } catch (err) {
    console.error('Error:', err.message);
    throw err;
  }
}
```

### Sequential vs Parallel Execution

```javascript
const delay = (ms, val) =>
  new Promise(r => setTimeout(() => r(val), ms));

// ❌ Sequential (ช้า - ทำทีละอย่าง)
async function sequential() {
  const start = Date.now();
  const a = await delay(1000, 'A'); // รอ 1 วินาที
  const b = await delay(1000, 'B'); // รอ 1 วินาทีอีก
  const c = await delay(1000, 'C'); // รอ 1 วินาทีอีก
  console.log(`${a}, ${b}, ${c} - ใช้เวลา: ${Date.now() - start}ms`);
  // ใช้เวลา ~3000ms
}

// ✅ Parallel (เร็ว - ทำพร้อมกัน)
async function parallel() {
  const start = Date.now();
  const [a, b, c] = await Promise.all([
    delay(1000, 'A'),
    delay(1000, 'B'),
    delay(1000, 'C')
  ]);
  console.log(`${a}, ${b}, ${c} - ใช้เวลา: ${Date.now() - start}ms`);
  // ใช้เวลา ~1000ms
}

sequential(); // ~3000ms
parallel();   // ~1000ms
```

### Async ใน Loops

```javascript
// ❌ forEach ไม่รองรับ async (ไม่รอ!)
async function badLoop(ids) {
  ids.forEach(async (id) => {
    const data = await fetchUser(id); // ไม่รอก่อนทำรายการถัดไป
    console.log(data);
  });
  console.log('เสร็จแล้ว?'); // แสดงก่อนที่ async operations จะเสร็จ!
}

// ✅ ใช้ for...of สำหรับ sequential
async function sequentialLoop(ids) {
  for (const id of ids) {
    const data = await fetchUser(id); // รอทีละรายการ
    console.log(data);
  }
  console.log('เสร็จจริงๆ'); // แสดงหลังทุกอย่างเสร็จ
}

// ✅ ใช้ Promise.all สำหรับ parallel
async function parallelLoop(ids) {
  const results = await Promise.all(ids.map(id => fetchUser(id)));
  results.forEach(data => console.log(data));
  console.log('เสร็จจริงๆ (เร็วกว่า)');
}

// ✅ Batch processing (ทำเป็นกลุ่ม)
async function batchLoop(ids, batchSize = 5) {
  const results = [];
  for (let i = 0; i < ids.length; i += batchSize) {
    const batch = ids.slice(i, i + batchSize);
    const batchResults = await Promise.all(batch.map(id => fetchUser(id)));
    results.push(...batchResults);
    console.log(`ประมวลผล batch ${Math.ceil(i/batchSize) + 1}/${Math.ceil(ids.length/batchSize)}`);
  }
  return results;
}
```

### Top-level await

```javascript
// Node.js 14.8+ รองรับ top-level await ใน ES modules (.mjs)
// หรือ "type": "module" ใน package.json

// file: app.mjs
import { readFile } from 'fs/promises';

// await โดยตรง ไม่ต้องอยู่ใน async function
const config = JSON.parse(
  await readFile('./config.json', 'utf8')
);

console.log('Config โหลดแล้ว:', config);

// ใช้ในการ initialize modules
const db = await Database.connect(config.db);
const server = await createServer(config.server);
```

---

## Promise Combinators {#promise-combinators}

### Promise.all() - รอทุกอัน

```javascript
// รอทุก Promise สำเร็จ
// ถ้าอันใดอัน reject ทั้งหมด reject ทันที

async function fetchAllData() {
  try {
    const [users, products, orders] = await Promise.all([
      fetch('/api/users').then(r => r.json()),
      fetch('/api/products').then(r => r.json()),
      fetch('/api/orders').then(r => r.json())
    ]);

    return { users, products, orders };
  } catch (err) {
    // ถ้า request ใดล้มเหลว จะ catch ที่นี่
    throw err;
  }
}

// ตัวอย่างที่ล้มเหลว
const promises = [
  Promise.resolve('A'),
  Promise.reject(new Error('B ล้มเหลว')),
  Promise.resolve('C')
];

Promise.all(promises)
  .then(results => console.log(results))
  .catch(err => console.error('ล้มเหลว:', err.message));
// ล้มเหลว: B ล้มเหลว  ← C ถูกยกเลิก!
```

### Promise.allSettled() - รอทุกอัน ไม่ว่าผลลัพธ์จะเป็นอะไร

```javascript
// รอทุก Promise ไม่ว่า fulfilled หรือ rejected
// ไม่ throw แม้มีบาง Promise reject

const promises = [
  Promise.resolve('สำเร็จ A'),
  Promise.reject(new Error('ล้มเหลว B')),
  Promise.resolve('สำเร็จ C')
];

const results = await Promise.allSettled(promises);

results.forEach((result, i) => {
  if (result.status === 'fulfilled') {
    console.log(`Promise ${i}: ✅ ${result.value}`);
  } else {
    console.log(`Promise ${i}: ❌ ${result.reason.message}`);
  }
});
// Promise 0: ✅ สำเร็จ A
// Promise 1: ❌ ล้มเหลว B
// Promise 2: ✅ สำเร็จ C

// ประโยชน์: ส่ง emails หลายอัน บางอันอาจล้มเหลว แต่ต้องการรายงานทั้งหมด
async function sendBulkEmails(recipients) {
  const results = await Promise.allSettled(
    recipients.map(email => sendEmail(email))
  );

  const succeeded = results.filter(r => r.status === 'fulfilled').length;
  const failed = results.filter(r => r.status === 'rejected').length;

  console.log(`ส่งสำเร็จ: ${succeeded}, ล้มเหลว: ${failed}`);

  return results;
}
```

### Promise.race() - เอาอันที่เร็วที่สุด

```javascript
// คืนผลลัพธ์ของ Promise แรกที่ settle (fulfilled หรือ rejected)

// ตัวอย่าง: Timeout pattern
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(`Timeout หลังจาก ${ms}ms`)), ms)
  );
  return Promise.race([promise, timeout]);
}

// ใช้งาน
try {
  const result = await withTimeout(fetchUser(123), 5000);
  console.log(result);
} catch (err) {
  if (err.message.includes('Timeout')) {
    console.error('ใช้เวลานานเกินไป!');
  }
}

// ตัวอย่าง: First responder (CDN fallback)
async function fetchFromFastest(url) {
  const mirrors = [
    `https://cdn1.example.com/${url}`,
    `https://cdn2.example.com/${url}`,
    `https://cdn3.example.com/${url}`
  ];

  return Promise.race(mirrors.map(mirror => fetch(mirror)));
}
```

### Promise.any() - เอาอันแรกที่สำเร็จ (ES2021)

```javascript
// คืนผลของ Promise แรกที่ fulfilled
// ถ้าทุกอัน reject จะ throw AggregateError

// ตัวอย่าง: ลองหลาย endpoints
async function fetchWithFallback(id) {
  try {
    const response = await Promise.any([
      fetch(`https://api1.example.com/data/${id}`),
      fetch(`https://api2.example.com/data/${id}`),
      fetch(`https://backup.example.com/data/${id}`)
    ]);
    return response.json();
  } catch (err) {
    if (err instanceof AggregateError) {
      // ทุก endpoint ล้มเหลว
      throw new Error('ไม่สามารถเชื่อมต่อ API ได้เลย');
    }
    throw err;
  }
}

// เปรียบเทียบ:
// Promise.all     = ทุกอันต้องสำเร็จ (AND)
// Promise.any     = อย่างน้อยหนึ่งอันต้องสำเร็จ (OR)
// Promise.race    = อันแรกที่ settle (resolved หรือ rejected)
// Promise.allSettled = รอทุกอัน รายงานทุกผล
```

---

## Error Handling Patterns {#error-handling-patterns}

### Try/Catch ใน async/await

```javascript
// รูปแบบพื้นฐาน
async function riskyOperation() {
  try {
    const data = await fetchData();
    const processed = await processData(data);
    await saveData(processed);
    return processed;
  } catch (err) {
    // catch ทุก error ใน try block
    console.error('เกิดข้อผิดพลาด:', err.message);
    throw err; // re-throw ถ้าต้องการให้ caller จัดการต่อ
  } finally {
    // ทำเสมอ (cleanup)
    await cleanup();
  }
}
```

### Error Type Checking

```javascript
async function handleSpecificErrors(userId) {
  try {
    const user = await fetchUser(userId);
    return user;
  } catch (err) {
    // จัดการตาม error type
    if (err instanceof NotFoundError) {
      return null; // User ไม่มีอยู่ ไม่ใช่ข้อผิดพลาดร้ายแรง
    }
    if (err instanceof NetworkError) {
      // ลองอีกครั้ง
      await delay(1000);
      return fetchUser(userId);
    }
    if (err instanceof AuthError) {
      // redirect ไป login
      redirectToLogin();
      return null;
    }
    // error อื่นๆ throw ต่อไป
    throw err;
  }
}
```

### Async Error Helper

```javascript
// Utility function สำหรับ clean error handling
async function tryCatch(promise) {
  try {
    const data = await promise;
    return [null, data];
  } catch (err) {
    return [err, null];
  }
}

// การใช้งาน - ไม่ต้องเขียน try/catch ซ้ำๆ
async function main() {
  const [userErr, user] = await tryCatch(fetchUser(123));
  if (userErr) {
    console.error('ดึง user ล้มเหลว:', userErr.message);
    return;
  }

  const [orderErr, orders] = await tryCatch(fetchOrders(user.id));
  if (orderErr) {
    console.error('ดึง orders ล้มเหลว:', orderErr.message);
    return;
  }

  console.log(`User ${user.name} มี ${orders.length} orders`);
}
```

### Retry Pattern

```javascript
async function withRetry(fn, options = {}) {
  const {
    maxRetries = 3,
    delay = 1000,
    backoff = 2,        // exponential backoff
    retryOn = () => true // function กำหนดว่า error ไหนควร retry
  } = options;

  let lastError;
  let currentDelay = delay;

  for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;

      if (attempt > maxRetries || !retryOn(err)) {
        throw err;
      }

      console.log(`ลองครั้งที่ ${attempt}/${maxRetries} ล้มเหลว: ${err.message}`);
      console.log(`รอ ${currentDelay}ms ก่อนลองใหม่...`);

      await new Promise(r => setTimeout(r, currentDelay));
      currentDelay *= backoff; // exponential backoff
    }
  }

  throw lastError;
}

// ตัวอย่างการใช้งาน
const result = await withRetry(
  () => fetch('https://api.example.com/data'),
  {
    maxRetries: 3,
    delay: 500,
    backoff: 2,
    retryOn: (err) => err.message.includes('Network') || err.status >= 500
  }
);
```

### Circuit Breaker Pattern

```javascript
class CircuitBreaker {
  constructor(fn, options = {}) {
    this.fn = fn;
    this.failureThreshold = options.failureThreshold || 5;
    this.successThreshold = options.successThreshold || 2;
    this.timeout = options.timeout || 60000; // 1 นาที

    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.failureCount = 0;
    this.successCount = 0;
    this.nextAttempt = Date.now();
  }

  async call(...args) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit Breaker: OPEN - ระบบยังไม่พร้อม');
      }
      // ลอง HALF_OPEN
      this.state = 'HALF_OPEN';
    }

    try {
      const result = await this.fn(...args);
      this._onSuccess();
      return result;
    } catch (err) {
      this._onFailure();
      throw err;
    }
  }

  _onSuccess() {
    this.failureCount = 0;
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.successThreshold) {
        this.state = 'CLOSED';
        this.successCount = 0;
        console.log('Circuit Breaker: CLOSED - ระบบกลับมาปกติ');
      }
    }
  }

  _onFailure() {
    this.failureCount++;
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
      console.log(`Circuit Breaker: OPEN - จะลองอีกครั้งใน ${this.timeout/1000}s`);
    }
  }

  get status() {
    return {
      state: this.state,
      failures: this.failureCount,
      nextAttempt: this.state === 'OPEN' ? new Date(this.nextAttempt) : null
    };
  }
}

// ใช้งาน
const breaker = new CircuitBreaker(fetchUser, {
  failureThreshold: 3,
  timeout: 30000
});

try {
  const user = await breaker.call(123);
  console.log(user);
} catch (err) {
  console.error(err.message);
  console.log('สถานะ:', breaker.status);
}
```

---

## Async Iterators {#async-iterators}

### Async Generator

```javascript
// Async Generator: function ที่ yield ค่า async
async function* generateNumbers(start, end) {
  for (let i = start; i <= end; i++) {
    // จำลอง async operation
    await new Promise(r => setTimeout(r, 10));
    yield i;
  }
}

// ใช้ for await...of
async function main() {
  for await (const num of generateNumbers(1, 5)) {
    console.log(num); // 1, 2, 3, 4, 5
  }
}
```

### Async Iterator สำหรับ Pagination

```javascript
// ดึงข้อมูลแบบ pagination อัตโนมัติ
async function* fetchAllPages(url) {
  let nextUrl = url;

  while (nextUrl) {
    const response = await fetch(nextUrl);
    const { data, next } = await response.json();

    for (const item of data) {
      yield item; // ส่งทีละ item
    }

    nextUrl = next; // URL หน้าถัดไป (null ถ้าหมดแล้ว)
  }
}

// ใช้งาน: ประมวลผลทุก item โดยไม่ต้องโหลดทั้งหมดมาก่อน
async function processAllUsers() {
  let count = 0;
  for await (const user of fetchAllPages('/api/users?page=1&limit=100')) {
    await processUser(user);
    count++;
    if (count % 100 === 0) console.log(`ประมวลผลแล้ว ${count} users`);
  }
  console.log(`ประมวลผลทั้งหมด ${count} users`);
}
```

### Async Iterator สำหรับ Stream Processing

```javascript
const { Readable } = require('stream');
const readline = require('readline');
const fs = require('fs');

// อ่านไฟล์ทีละบรรทัดด้วย async iterator
async function processLargeFile(filename) {
  const fileStream = fs.createReadStream(filename);
  const rl = readline.createInterface({ input: fileStream });

  let lineCount = 0;
  let errorCount = 0;

  // readline ใช้ Symbol.asyncIterator
  for await (const line of rl) {
    lineCount++;
    try {
      const data = JSON.parse(line);
      await processRecord(data);
    } catch (err) {
      errorCount++;
      console.warn(`บรรทัด ${lineCount} ไม่ถูกต้อง:`, err.message);
    }
  }

  console.log(`ประมวลผล ${lineCount} บรรทัด, พบข้อผิดพลาด ${errorCount} บรรทัด`);
}

// Custom Async Iterable
class DatabaseCursor {
  constructor(query, params) {
    this.query = query;
    this.params = params;
    this.offset = 0;
    this.pageSize = 50;
    this.done = false;
  }

  [Symbol.asyncIterator]() {
    return this;
  }

  async next() {
    if (this.done) return { done: true };

    // ดึงข้อมูลจาก DB (จำลอง)
    const rows = await db.query(
      `${this.query} LIMIT ${this.pageSize} OFFSET ${this.offset}`,
      this.params
    );

    if (rows.length === 0) {
      this.done = true;
      return { done: true };
    }

    this.offset += rows.length;

    if (rows.length < this.pageSize) {
      this.done = true;
    }

    return { value: rows, done: false };
  }
}

// ใช้งาน
const cursor = new DatabaseCursor('SELECT * FROM large_table WHERE active = ?', [true]);
for await (const rows of cursor) {
  for (const row of rows) {
    await processRow(row);
  }
}
```

---

## Practical Patterns {#practical-patterns}

### Pattern 1: Task Queue

```javascript
class AsyncQueue {
  constructor(concurrency = 1) {
    this.concurrency = concurrency;
    this.running = 0;
    this.queue = [];
  }

  async add(task) {
    return new Promise((resolve, reject) => {
      this.queue.push({ task, resolve, reject });
      this._run();
    });
  }

  async _run() {
    if (this.running >= this.concurrency || this.queue.length === 0) return;

    this.running++;
    const { task, resolve, reject } = this.queue.shift();

    try {
      const result = await task();
      resolve(result);
    } catch (err) {
      reject(err);
    } finally {
      this.running--;
      this._run(); // ทำงานถัดไป
    }
  }
}

// ใช้งาน: จำกัด concurrent API calls
const queue = new AsyncQueue(3); // ทำงาน 3 อย่างพร้อมกัน

const tasks = Array.from({ length: 10 }, (_, i) => () =>
  fetch(`/api/item/${i}`).then(r => r.json())
);

const results = await Promise.all(tasks.map(task => queue.add(task)));
console.log('ผลลัพธ์ทั้งหมด:', results.length);
```

### Pattern 2: Memoization ของ Async Functions

```javascript
function memoizeAsync(fn, options = {}) {
  const cache = new Map();
  const { ttl = Infinity, maxSize = 100 } = options;

  return async function(...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      const { value, expiresAt } = cache.get(key);
      if (Date.now() < expiresAt) {
        return value; // คืนจาก cache
      }
      cache.delete(key);
    }

    const result = await fn(...args);

    // จัดการขนาด cache
    if (cache.size >= maxSize) {
      const firstKey = cache.keys().next().value;
      cache.delete(firstKey);
    }

    cache.set(key, {
      value: result,
      expiresAt: Date.now() + ttl
    });

    return result;
  };
}

// ใช้งาน
const cachedFetchUser = memoizeAsync(fetchUser, { ttl: 60000 }); // cache 1 นาที

// เรียกซ้ำๆ แต่ fetch จริงแค่ครั้งเดียว
const user1 = await cachedFetchUser(123); // fetch จริง
const user2 = await cachedFetchUser(123); // จาก cache (เร็วกว่ามาก)
```

### Pattern 3: Async Event Emitter

```javascript
const EventEmitter = require('events');

class AsyncEventEmitter extends EventEmitter {
  // emit แบบ async และรอให้ handler ทั้งหมดเสร็จ
  async emitAsync(event, ...args) {
    const listeners = this.listeners(event);
    const results = await Promise.allSettled(
      listeners.map(listener => Promise.resolve(listener(...args)))
    );

    const errors = results
      .filter(r => r.status === 'rejected')
      .map(r => r.reason);

    if (errors.length > 0) {
      throw new AggregateError(errors, `${errors.length} listener(s) threw errors`);
    }

    return results
      .filter(r => r.status === 'fulfilled')
      .map(r => r.value);
  }
}

// ใช้งาน
const emitter = new AsyncEventEmitter();

emitter.on('process', async (data) => {
  await delay(100);
  console.log('Handler 1:', data);
  return `result1`;
});

emitter.on('process', async (data) => {
  await delay(200);
  console.log('Handler 2:', data);
  return `result2`;
});

const results = await emitter.emitAsync('process', 'ข้อมูล');
console.log('ผลลัพธ์:', results); // ['result1', 'result2']
```

### Pattern 4: Promise Pool (Throttled Concurrency)

```javascript
async function promisePool(tasks, concurrency) {
  const results = new Array(tasks.length);
  const executing = new Set();

  for (let i = 0; i < tasks.length; i++) {
    const promise = Promise.resolve().then(() => tasks[i]());

    results[i] = promise;
    executing.add(promise);

    const cleanup = () => executing.delete(promise);
    promise.then(cleanup, cleanup);

    // รอถ้า concurrent tasks เต็ม
    if (executing.size >= concurrency) {
      await Promise.race(executing);
    }
  }

  return Promise.all(results);
}

// ใช้งาน
const urls = Array.from({ length: 20 }, (_, i) => `/api/item/${i}`);
const tasks = urls.map(url => () => fetch(url).then(r => r.json()));

// ดาวน์โหลด 20 items แต่ทำพร้อมกันได้แค่ 5
const results = await promisePool(tasks, 5);
```

---

## แบบฝึกหัด {#แบบฝึกหัด}

### แบบฝึกหัดที่ 1: Async Data Pipeline

สร้าง pipeline ที่:
1. ดึง list ของ user IDs จาก API
2. สำหรับแต่ละ user: ดึง profile และ orders พร้อมกัน
3. คำนวณยอดรวมของแต่ละ user
4. กรองเฉพาะ users ที่มียอดเกิน threshold
5. บันทึกผลลัพธ์ลงไฟล์

```javascript
async function analyzeTopCustomers(threshold) {
  // TODO: implement
  // Hint: ใช้ Promise.all สำหรับ parallel fetching
  // ใช้ withRetry สำหรับ network errors
  // ใช้ batch processing เพื่อไม่ให้ flood API
}
```

### แบบฝึกหัดที่ 2: Rate Limiter

สร้าง `RateLimiter` class ที่:
- กำหนด max requests ต่อหน่วยเวลา
- Queue requests ที่เกิน limit
- ให้ countdown เมื่อถูก throttle

### แบบฝึกหัดที่ 3: Async State Machine

สร้าง state machine สำหรับ order processing:
- States: pending → paid → processing → shipped → delivered
- แต่ละ transition เป็น async operation
- มี timeout สำหรับแต่ละ state
- รองรับ cancellation

---

## สรุป

| เทคนิค | ใช้เมื่อ | ข้อดี | ข้อเสีย |
|--------|---------|------|--------|
| Callbacks | Node.js APIs เก่า | เร็ว, ง่าย | Callback hell |
| Promises | Sequential chains | Chain ได้, Error handling ดี | Verbose |
| async/await | ทั่วไป | อ่านง่าย, Debug ง่าย | ต้องระวัง parallel |
| Promise.all | Parallel tasks | เร็วที่สุด | ถ้า 1 อัน fail ทั้งหมด fail |
| Promise.allSettled | Parallel + รายงานทุก result | ครบถ้วน | ต้อง check status เอง |
| Promise.race | Timeout/Fallback | ยืดหยุ่น | ยากต่อ cleanup |
| Promise.any | Primary + fallbacks | Resilient | ต้องระวัง AggregateError |

---

## ก้าวต่อไป

➡️ **Part 09: Error Handling** - การจัดการข้อผิดพลาดอย่างมีประสิทธิภาพ

---
*Node.js/Express.js Course - Part 08 of 20*
*ขั้นตอนที่ 701-800 จาก 1000*
