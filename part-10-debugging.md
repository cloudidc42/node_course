# ขั้นตอนที่ 901-1000 จาก 1000
# Part 10: Debugging ใน Node.js

---

## สารบัญ

1. [Console Debugging](#console-debugging)
2. [Node.js Debugger](#nodejs-debugger)
3. [VS Code Debugging](#vscode-debugging)
4. [Chrome DevTools](#chrome-devtools)
5. [Performance Profiling](#performance-profiling)
6. [Memory Leak Detection](#memory-leak-detection)
7. [Common Debugging Techniques](#common-debugging-techniques)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Console Debugging {#console-debugging}

### console methods ทั้งหมด

```javascript
// 1. console.log - ทั่วไป
console.log('สวัสดี', 'Node.js');
console.log('User:', { id: 1, name: 'สมชาย' });
console.log('Array:', [1, 2, 3]);

// 2. console.error - ส่งไป stderr
console.error('ข้อผิดพลาด:', new Error('something went wrong'));

// 3. console.warn - คำเตือน (ใน terminal มักเป็นสีเหลือง)
console.warn('⚠️ คำเตือน: deprecation');

// 4. console.info - ข้อมูล (เหมือน log แต่ semantic ต่างกัน)
console.info('ℹ️ Server เริ่มทำงาน');

// 5. console.dir - แสดงโครงสร้าง object อย่างละเอียด
const obj = { a: { b: { c: { d: 'deep' } } } };
console.dir(obj, { depth: null, colors: true }); // depth: null = ไม่จำกัด

// 6. console.table - แสดงแบบตาราง
const users = [
  { id: 1, name: 'สมชาย', age: 25, role: 'admin' },
  { id: 2, name: 'สมหญิง', age: 30, role: 'user' },
  { id: 3, name: 'สมศักดิ์', age: 22, role: 'user' }
];
console.table(users);
// ┌─────────┬────┬──────────┬─────┬─────────┐
// │ (index) │ id │   name   │ age │  role   │
// ├─────────┼────┼──────────┼─────┼─────────┤
// │    0    │ 1  │ 'สมชาย'  │ 25  │ 'admin' │
// │    1    │ 2  │ 'สมหญิง' │ 30  │ 'user'  │
// │    2    │ 3  │ 'สมศักดิ์'│ 22  │ 'user'  │
// └─────────┴────┴──────────┴─────┴─────────┘

// 7. console.time / console.timeEnd - วัดเวลา
console.time('operation');
for (let i = 0; i < 1000000; i++) {}
console.timeEnd('operation'); // operation: 2.345ms

// 8. console.timeLog - บันทึกเวลากลางทาง
console.time('process');
// ... ทำงาน ...
console.timeLog('process', 'step 1 done'); // process: 100ms step 1 done
// ... ทำงาน ...
console.timeEnd('process'); // process: 250ms

// 9. console.count / console.countReset - นับจำนวนครั้ง
function onClick(element) {
  console.count(`click ${element}`);
}
onClick('button'); // click button: 1
onClick('button'); // click button: 2
onClick('link');   // click link: 1
console.countReset('button');
onClick('button'); // click button: 1

// 10. console.group / console.groupEnd - จัดกลุ่ม
console.group('HTTP Request');
console.log('Method: GET');
console.log('URL: /api/users');
console.group('Headers');
console.log('Content-Type: application/json');
console.log('Authorization: Bearer ...');
console.groupEnd();
console.log('Status: 200 OK');
console.groupEnd();

// 11. console.assert - assert แบบง่าย
const value = 42;
console.assert(value > 0, 'ค่าต้องมากกว่า 0'); // ไม่แสดงอะไร (true)
console.assert(value < 0, 'ค่าต้องน้อยกว่า 0'); // Assertion failed: ค่าต้องน้อยกว่า 0

// 12. console.trace - แสดง stack trace
function inner() {
  console.trace('เรียกจากที่ไหน?');
}
function outer() {
  inner();
}
outer();
// Trace: เรียกจากที่ไหน?
//   at inner (/app.js:2:11)
//   at outer (/app.js:5:3)
//   at Object.<anonymous> (/app.js:7:1)

// 13. console.clear - ล้างหน้าจอ
// console.clear();
```

### การสร้าง Debug Logger ที่ดี

```javascript
// debug package (ยอดนิยมมากใน Node.js ecosystem)
// npm install debug
const debug = require('debug');

// สร้าง debug namespaces
const debugHTTP = debug('app:http');
const debugDB = debug('app:db');
const debugCache = debug('app:cache');
const debugAuth = debug('app:auth');

// ใช้งาน
debugHTTP('GET /users - user: %O', { id: 1, role: 'admin' });
debugDB('Query: %s params: %j', 'SELECT * FROM users', [1, 2, 3]);
debugCache('Cache hit: %s', 'user:123');

// เปิด/ปิดด้วย environment variable:
// DEBUG=app:* node app.js     ← แสดงทุก namespace
// DEBUG=app:http node app.js  ← แสดงเฉพาะ http
// DEBUG=app:* node app.js  ← แสดงทุก namespace ยกเว้น db
```

```javascript
// Custom debug utility ที่มีสีสัน
const colors = {
  reset: '\x1b[0m',
  red: '\x1b[31m',
  green: '\x1b[32m',
  yellow: '\x1b[33m',
  blue: '\x1b[34m',
  cyan: '\x1b[36m',
  magenta: '\x1b[35m',
  gray: '\x1b[90m'
};

const DEBUG_ENABLED = process.env.NODE_ENV !== 'production';

function createDebugger(namespace, color = 'cyan') {
  return function(...args) {
    if (!DEBUG_ENABLED) return;
    const timestamp = new Date().toISOString().split('T')[1].replace('Z', '');
    const prefix = `${colors[color]}[${timestamp}] [${namespace}]${colors.reset}`;
    console.log(prefix, ...args);
  };
}

const log = {
  http: createDebugger('HTTP', 'blue'),
  db: createDebugger('DB', 'green'),
  cache: createDebugger('Cache', 'yellow'),
  auth: createDebugger('Auth', 'magenta'),
  error: createDebugger('Error', 'red')
};

// ใช้งาน
log.http('GET /api/users', { userId: 123 });
log.db('Query executed', { sql: 'SELECT...', rows: 50, time: '12ms' });
log.error('Connection failed', { host: 'localhost', code: 'ECONNREFUSED' });
```

---

## Node.js Debugger {#nodejs-debugger}

### Built-in Debugger (CLI)

```bash
# เริ่ม debugger ผ่าน CLI
node inspect app.js

# หรือ
node --inspect app.js         # เปิด inspector (port 9229)
node --inspect-brk app.js     # หยุดที่บรรทัดแรก (รอ connection)
node --inspect=0.0.0.0:9229 app.js  # เปิดให้ remote เชื่อมต่อได้
```

```javascript
// การใช้ debugger statement ในโค้ด
function calculateTax(income, rate) {
  debugger; // หยุดที่นี่เมื่อ debugger เชื่อมต่อ

  const tax = income * rate;
  const netIncome = income - tax;

  debugger; // หยุดอีกครั้ง

  return { tax, netIncome };
}

const result = calculateTax(50000, 0.2);
console.log(result);
```

### Node.js Inspector Commands (CLI debugger)

```
คำสั่งสำคัญใน Node.js CLI Debugger (node inspect):
  c, cont      → ทำงานต่อจนถึง breakpoint ถัดไป
  n, next      → ทำบรรทัดถัดไป (step over)
  s, step      → เข้าไปใน function (step into)
  o, out       → ออกจาก function ปัจจุบัน (step out)
  bt, backtrace → แสดง call stack
  list(n)      → แสดง n บรรทัดรอบๆ ตำแหน่งปัจจุบัน
  setBreakpoint(line)  → ตั้ง breakpoint
  clearBreakpoint(line) → ลบ breakpoint
  watch('expr')  → ติดตามการเปลี่ยนแปลงของ expression
  repl         → เปิด REPL สำหรับ inspect values
  exec('expr') → รัน expression และแสดงผล
  restart      → restart script
  kill         → หยุด script
  quit         → ออกจาก debugger
```

---

## VS Code Debugging {#vscode-debugging}

### การตั้งค่า launch.json

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Node.js App",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/src/index.js",
      "env": {
        "NODE_ENV": "development",
        "PORT": "3000"
      },
      "envFile": "${workspaceFolder}/.env",
      "sourceMaps": true,
      "restart": true,
      "console": "integratedTerminal"
    },
    {
      "name": "Debug Express with nodemon",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "nodemon",
      "program": "${workspaceFolder}/src/index.js",
      "restart": true,
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen"
    },
    {
      "name": "Attach to Process",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "restart": true
    },
    {
      "name": "Debug Jest Tests",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/jest",
      "args": ["--runInBand", "--no-coverage"],
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen"
    },
    {
      "name": "Debug TypeScript",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/src/index.ts",
      "runtimeArgs": ["-r", "ts-node/register"],
      "sourceMaps": true,
      "outFiles": ["${workspaceFolder}/dist/**/*.js"]
    }
  ]
}
```

### VS Code Debugging Features

```
VS Code Debugger Features:
  F5           → Start/Continue debugging
  F10          → Step Over (ข้ามฟังก์ชัน)
  F11          → Step Into (เข้าไปในฟังก์ชัน)
  Shift+F11    → Step Out (ออกจากฟังก์ชัน)
  F9           → Toggle Breakpoint
  Ctrl+Shift+F5 → Restart
  Shift+F5     → Stop

Breakpoint Types:
  Line Breakpoint   → หยุดที่บรรทัด
  Conditional Breakpoint → หยุดเมื่อเงื่อนไขเป็น true
  Hit Count Breakpoint   → หยุดหลังจากผ่านมา N ครั้ง
  Logpoint → Log โดยไม่หยุดทำงาน
  Exception Breakpoint → หยุดเมื่อ exception เกิด
```

```javascript
// Conditional Breakpoints ตัวอย่าง:
// คลิกขวาที่ breakpoint → Edit Breakpoint → Condition

// หยุดเมื่อ user.role === 'admin'
function processUser(user) {
  // ตั้ง conditional breakpoint ที่บรรทัดนี้
  // Condition: user.role === 'admin'
  return doSomething(user);
}

// หยุดเมื่อ loop ถึง index 50
for (let i = 0; i < 100; i++) {
  // Condition: i === 50
  processItem(items[i]);
}
```

---

## Chrome DevTools {#chrome-devtools}

### การเชื่อมต่อ Chrome DevTools กับ Node.js

```bash
# เริ่ม Node.js ด้วย --inspect
node --inspect-brk server.js

# จะเห็น output:
# Debugger listening on ws://127.0.0.1:9229/...
# For help, see: https://nodejs.org/en/docs/inspector

# เปิด Chrome แล้วไปที่:
# chrome://inspect
# แล้วคลิก "inspect" ใต้ Remote Target
```

### Chrome DevTools Features สำหรับ Node.js

```javascript
// Sources Panel:
// - เปิดไฟล์ source
// - ตั้ง breakpoints
// - Step through code
// - Watch expressions
// - Call stack

// Console Panel:
// - ทดสอบ expressions แบบ interactive
// - ดู console output
// - Live evaluation

// Memory Panel:
// - Heap snapshot
// - Allocation profiler
// - Heap timeline

// Profiler Panel:
// - CPU profiling
// - Performance flame chart

// ตัวอย่าง: debug ด้วย Chrome DevTools
async function processLargeDataset(data) {
  const results = [];

  for (let i = 0; i < data.length; i++) {
    // ตั้ง breakpoint ที่นี่ใน DevTools
    const item = data[i];
    const processed = await transform(item);
    results.push(processed);
  }

  return results;
}
```

### Inspector Protocol

```javascript
// ใช้ inspector module สำหรับ programmatic debugging
const inspector = require('inspector');

// เปิด inspector session
const session = new inspector.Session();
session.connect();

// Enable CPU profiler
session.post('Profiler.enable', () => {
  session.post('Profiler.start', () => {
    // รันโค้ดที่ต้องการ profile
    runExpensiveOperation();

    // หยุด profiler และดึงผลลัพธ์
    session.post('Profiler.stop', (err, { profile }) => {
      if (!err) {
        require('fs').writeFileSync(
          './profile.cpuprofile',
          JSON.stringify(profile)
        );
        console.log('บันทึก CPU profile แล้ว');
      }
    });
  });
});

// ดู heap snapshot
function takeHeapSnapshot() {
  return new Promise((resolve, reject) => {
    session.post('HeapProfiler.enable', () => {
      session.post('HeapProfiler.takeHeapSnapshot', null, (err, result) => {
        if (err) return reject(err);
        resolve(result);
      });
    });
  });
}
```

---

## Performance Profiling {#performance-profiling}

### Performance Hooks (perf_hooks)

```javascript
const { performance, PerformanceObserver } = require('perf_hooks');

// 1. performance.now() - เวลาที่แม่นยำ
const start = performance.now();
// ... รันโค้ด ...
const end = performance.now();
console.log(`ใช้เวลา: ${(end - start).toFixed(3)}ms`);

// 2. performance.mark() และ performance.measure()
performance.mark('A');
doOperation1();
performance.mark('B');
doOperation2();
performance.mark('C');

// วัดช่วง A ถึง B
performance.measure('Operation 1', 'A', 'B');
// วัดช่วง B ถึง C
performance.measure('Operation 2', 'B', 'C');
// วัดทั้งหมด
performance.measure('Total', 'A', 'C');

// ดูผลลัพธ์
const measures = performance.getEntriesByType('measure');
measures.forEach(m => {
  console.log(`${m.name}: ${m.duration.toFixed(3)}ms`);
});

// 3. PerformanceObserver - ติดตาม performance entries แบบ real-time
const obs = new PerformanceObserver((list) => {
  list.getEntries().forEach(entry => {
    console.log(`[Performance] ${entry.name}: ${entry.duration.toFixed(3)}ms`);
  });
});

obs.observe({ entryTypes: ['measure', 'function'] });

// ล้าง marks และ measures
performance.clearMarks();
performance.clearMeasures();
obs.disconnect();
```

### Benchmarking

```javascript
const { performance } = require('perf_hooks');

// Benchmark function helper
async function benchmark(name, fn, iterations = 1000) {
  // warm up
  for (let i = 0; i < 10; i++) await fn();

  const times = [];
  for (let i = 0; i < iterations; i++) {
    const start = performance.now();
    await fn();
    times.push(performance.now() - start);
  }

  times.sort((a, b) => a - b);

  const sum = times.reduce((a, b) => a + b, 0);
  const mean = sum / times.length;
  const median = times[Math.floor(times.length / 2)];
  const p95 = times[Math.floor(times.length * 0.95)];
  const p99 = times[Math.floor(times.length * 0.99)];
  const min = times[0];
  const max = times[times.length - 1];

  console.log(`\n📊 Benchmark: ${name} (${iterations} iterations)`);
  console.log(`  Mean:   ${mean.toFixed(3)}ms`);
  console.log(`  Median: ${median.toFixed(3)}ms`);
  console.log(`  P95:    ${p95.toFixed(3)}ms`);
  console.log(`  P99:    ${p99.toFixed(3)}ms`);
  console.log(`  Min:    ${min.toFixed(3)}ms`);
  console.log(`  Max:    ${max.toFixed(3)}ms`);

  return { mean, median, p95, p99, min, max };
}

// เปรียบเทียบ algorithm
async function runComparisons() {
  const data = Array.from({ length: 1000 }, () => Math.random() * 1000);

  await benchmark('Array.sort (built-in)', () => {
    [...data].sort((a, b) => a - b);
  });

  await benchmark('Bubble Sort', () => {
    const arr = [...data];
    for (let i = 0; i < arr.length; i++) {
      for (let j = 0; j < arr.length - i - 1; j++) {
        if (arr[j] > arr[j + 1]) {
          [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
        }
      }
    }
  });
}

runComparisons();
```

### CPU Profiling

```javascript
// สร้างไฟล์ start-profile.js
const { Session } = require('inspector');
const fs = require('fs');

// เริ่ม CPU profiling
const session = new Session();
session.connect();

console.log('เริ่ม CPU Profiling...');
session.post('Profiler.enable', () => {
  session.post('Profiler.start', () => {
    // โหลด app หลังจาก profiler เริ่ม
    require('./app.js');

    // หยุดหลังจาก 30 วินาที
    setTimeout(() => {
      session.post('Profiler.stop', (err, { profile }) => {
        if (err) throw err;

        const filename = `cpu-profile-${Date.now()}.cpuprofile`;
        fs.writeFileSync(filename, JSON.stringify(profile));
        console.log(`บันทึก CPU profile: ${filename}`);
        console.log('เปิดด้วย Chrome DevTools → Performance tab → Load profile');

        session.disconnect();
        process.exit(0);
      });
    }, 30000);
  });
});
```

### Clinic.js Tools

```bash
# ติดตั้ง clinic
npm install -g clinic

# 1. clinic doctor - วิเคราะห์ health ของ app
clinic doctor -- node app.js

# 2. clinic flame - สร้าง flame graph สำหรับ CPU
clinic flame -- node app.js

# 3. clinic bubbleprof - วิเคราะห์ async operations
clinic bubbleprof -- node app.js

# 4. autocannon - load testing
npm install -g autocannon
autocannon -c 100 -d 5 http://localhost:3000/api/users
# -c 100 = 100 concurrent connections
# -d 5   = รัน 5 วินาที
```

---

## Memory Leak Detection {#memory-leak-detection}

### การตรวจสอบ Memory Usage

```javascript
// ตรวจสอบ memory usage แบบ real-time
function getMemoryUsage() {
  const used = process.memoryUsage();
  return {
    rss:       `${(used.rss / 1024 / 1024).toFixed(2)} MB`,       // Resident Set Size
    heapTotal: `${(used.heapTotal / 1024 / 1024).toFixed(2)} MB`,  // V8 heap size
    heapUsed:  `${(used.heapUsed / 1024 / 1024).toFixed(2)} MB`,   // ใช้จริง
    external:  `${(used.external / 1024 / 1024).toFixed(2)} MB`,   // C++ objects
    arrayBuffers: `${(used.arrayBuffers / 1024 / 1024).toFixed(2)} MB`
  };
}

// ตรวจสอบทุก 5 วินาที
const monitorInterval = setInterval(() => {
  const mem = getMemoryUsage();
  console.log(`[Memory] RSS: ${mem.rss} | Heap: ${mem.heapUsed}/${mem.heapTotal}`);
}, 5000);

// ยกเลิกเมื่อเสร็จ
process.on('SIGINT', () => {
  clearInterval(monitorInterval);
  process.exit();
});
```

### Memory Leak Patterns และวิธีแก้

```javascript
// 1. ❌ Global Variables Accumulation
const cache = new Map(); // global cache ที่โตเรื่อยๆ

function getUser(id) {
  if (cache.has(id)) return cache.get(id);
  const user = db.findUser(id);
  cache.set(id, user); // ไม่เคยลบ!
  return user;
}

// ✅ ใช้ LRU Cache หรือตั้ง size limit
const LRU = require('lru-cache');
const lruCache = new LRU({ max: 1000, ttl: 1000 * 60 * 5 }); // 5 นาที

// 2. ❌ Event Listener Accumulation
function setupConnection(socket) {
  // ทุกครั้งที่เรียก function นี้ เพิ่ม listener ใหม่
  // แต่ไม่ลบของเก่า!
  process.on('exit', () => socket.close());
}

// ✅ ลบ listener เมื่อไม่ใช้
function setupConnection(socket) {
  const cleanup = () => socket.close();
  process.on('exit', cleanup);

  socket.on('close', () => {
    process.removeListener('exit', cleanup); // ลบเมื่อ socket ปิด
  });
}

// 3. ❌ Closure Memory Leak
function createLeak() {
  const bigData = new Array(1000000).fill('leak'); // 1M items

  return function() {
    // ถึงแม้จะไม่ใช้ bigData
    // แต่ closure นี้ยังถือ reference ไว้
    return 'something';
  };
}

// ✅ ตัด reference ออก
function createFixed() {
  let bigData = new Array(1000000).fill('data');
  const processed = processData(bigData);
  bigData = null; // ปล่อยให้ GC เก็บ

  return function() {
    return processed; // ใช้เฉพาะข้อมูลที่ process แล้ว
  };
}

// 4. ❌ Circular References (เก่า - V8 จัดการได้แล้ว แต่ยังควรระวัง)
function createCircular() {
  const obj = {};
  obj.self = obj; // อ้างถึงตัวเอง
  return obj;
}

// 5. ❌ Timer และ Interval ที่ไม่ถูก clear
class DataFetcher {
  start() {
    this.interval = setInterval(() => {
      this.fetchData(); // ถ้า fetchData เก็บ reference ไว้
    }, 1000);
  }

  // ❌ ขาด stop method!
}

// ✅
class DataFetcherFixed {
  start() {
    this.interval = setInterval(() => this.fetchData(), 1000);
  }

  stop() {
    clearInterval(this.interval); // หยุดเมื่อไม่ใช้
    this.interval = null;
  }
}
```

### Heap Snapshot Analysis

```javascript
// สร้าง heap snapshot
const { writeHeapSnapshot } = require('v8');

// บันทึก snapshot
const filename = writeHeapSnapshot('./snapshots/');
console.log(`Heap snapshot บันทึกที่: ${filename}`);

// หรือใช้ inspector
const inspector = require('inspector');
const fs = require('fs');
const session = new inspector.Session();

function takeSnapshot(filename) {
  return new Promise((resolve, reject) => {
    const chunks = [];

    session.on('HeapProfiler.addHeapSnapshotChunk', ({ params }) => {
      chunks.push(params.chunk);
    });

    session.post('HeapProfiler.enable', () => {
      session.post('HeapProfiler.takeHeapSnapshot', null, (err) => {
        if (err) return reject(err);

        fs.writeFileSync(filename, chunks.join(''));
        console.log(`Snapshot บันทึกที่: ${filename}`);
        resolve(filename);
      });
    });
  });
}

// เปิดด้วย Chrome DevTools → Memory → Load profile
// หรือ Firefox → about:memory

// ตรวจสอบ memory growth
async function detectLeak(duration = 60000, interval = 5000) {
  session.connect();
  const snapshots = [];
  const start = Date.now();

  console.log(`ตรวจสอบ memory leak ${duration/1000} วินาที...`);

  const check = async () => {
    const used = process.memoryUsage().heapUsed / 1024 / 1024;
    snapshots.push({ time: Date.now() - start, heapMB: used });
    console.log(`[${snapshots.length}] Heap: ${used.toFixed(2)} MB`);
  };

  await check();
  const intervalId = setInterval(check, interval);

  await new Promise(r => setTimeout(r, duration));
  clearInterval(intervalId);

  // วิเคราะห์ trend
  const first = snapshots[0].heapMB;
  const last = snapshots[snapshots.length - 1].heapMB;
  const growth = last - first;

  console.log(`\nผลการวิเคราะห์:`);
  console.log(`  เริ่มต้น: ${first.toFixed(2)} MB`);
  console.log(`  สิ้นสุด: ${last.toFixed(2)} MB`);
  console.log(`  การเติบโต: ${growth.toFixed(2)} MB`);

  if (growth > 50) {
    console.warn('⚠️ Memory เพิ่มขึ้นมาก - อาจมี memory leak!');
  } else {
    console.log('✅ Memory ดูปกติ');
  }

  session.disconnect();
}
```

### memwatch-next สำหรับ Memory Leak Detection

```javascript
// npm install @airbnb/node-memwatch
const memwatch = require('@airbnb/node-memwatch');

// ตรวจจับ memory leak
memwatch.on('leak', (info) => {
  console.error('🚨 Memory leak detected!', info);
  // info = { start, end, growth, reason }
});

// ตรวจสอบ garbage collection
memwatch.on('stats', (stats) => {
  console.log('GC stats:', {
    num_full_gc: stats.num_full_gc,
    num_inc_gc: stats.num_inc_gc,
    heap_compactions: stats.heap_compactions,
    estimated_base: `${(stats.estimated_base / 1024 / 1024).toFixed(2)} MB`,
    current_base: `${(stats.current_base / 1024 / 1024).toFixed(2)} MB`,
    min: `${(stats.min / 1024 / 1024).toFixed(2)} MB`,
    max: `${(stats.max / 1024 / 1024).toFixed(2)} MB`
  });
});

// Heap Diff: เปรียบเทียบ heap ก่อน-หลัง
const hd = new memwatch.HeapDiff();

// รันโค้ดที่สงสัย
for (let i = 0; i < 10000; i++) {
  const obj = { data: new Array(100).fill(i) };
  // ลืมทำ cleanup!
  leakyCache.set(i, obj);
}

const diff = hd.end();
console.log('Heap diff:', JSON.stringify(diff, null, 2));
// จะแสดง objects ที่เพิ่มขึ้น
```

---

## Common Debugging Techniques {#common-debugging-techniques}

### 1. Binary Search Debugging

```javascript
// เมื่อไม่รู้ว่า bug อยู่ที่ไหน ใช้ binary search
// แทนที่จะดูทีละบรรทัด ให้หาจุดกึ่งกลาง

// ตัวอย่าง: process 1000 items แล้ว crash ที่ไหนสักที่
async function processItems(items) {
  // เพิ่ม log ที่จุดกึ่งกลาง
  console.log(`ประมวลผล 0-${items.length/2}...`);
  for (let i = 0; i < items.length / 2; i++) {
    await processItem(items[i]);
  }

  console.log('ถึงจุดกึ่งกลาง - ยังไม่ crash');

  console.log(`ประมวลผล ${items.length/2}-${items.length}...`);
  for (let i = items.length / 2; i < items.length; i++) {
    await processItem(items[i]);
  }
}
```

### 2. Rubber Duck Debugging

อธิบายโค้ดทีละบรรทัดให้ "เพื่อน" ฟัง (หรืออธิบายกับตัวเอง) มักพบ bug ระหว่างอธิบาย เพราะต้องคิดอย่างเป็นระบบ

### 3. Logging Strategy

```javascript
// สร้าง logging ที่ช่วย debug ได้จริง

// ❌ ไม่ดี
console.log(data);
console.log('here');
console.log(x);

// ✅ ดี: มีบริบทชัดเจน
console.log('[UserService.create] input:', JSON.stringify(userData));
console.log('[UserService.create] validation passed');
console.log('[UserService.create] db.insert result:', result);

// ✅ ดีกว่า: ใช้ structured logging
logger.debug({
  context: 'UserService.create',
  step: 'validation',
  input: userData,
  valid: true
});

// เพิ่ม timing
const startTime = Date.now();
logger.debug({ step: 'db.insert', startTime });
const result = await db.insert(userData);
logger.debug({
  step: 'db.insert.complete',
  duration: Date.now() - startTime,
  result
});
```

### 4. การ Debug Async Issues

```javascript
// ปัญหา: Promise ที่ไม่ resolve/reject
function debugPromise(promise, name = 'unnamed') {
  const timeout = setTimeout(() => {
    console.warn(`⚠️ Promise "${name}" ยังไม่ resolve หลังจาก 5 วินาที`);
    console.trace('สร้างจากที่นี่:');
  }, 5000);

  return promise.finally(() => clearTimeout(timeout));
}

// ใช้งาน
const result = await debugPromise(
  fetchUser(123),
  'fetchUser(123)'
);

// ติดตาม Promise chain
async function debugChain() {
  console.log('Step 1: เริ่มต้น');
  const data = await fetchData();

  console.log('Step 2: ได้ข้อมูลมาแล้ว:', data?.length, 'items');
  const processed = await processData(data);

  console.log('Step 3: ประมวลผลเสร็จ:', processed?.length, 'items');
  const saved = await saveData(processed);

  console.log('Step 4: บันทึกเสร็จ:', saved?.id);
  return saved;
}
```

### 5. การ Debug Race Conditions

```javascript
// Race condition: ผลลัพธ์ขึ้นอยู่กับลำดับการทำงาน

// ❌ Race condition
let count = 0;
async function increment() {
  const current = count;     // อ่านค่า
  await delay(1);            // อาจมี operation อื่นแทรก!
  count = current + 1;       // เขียนค่าที่อาจ stale
}

// รัน 100 ครั้งพร้อมกัน
await Promise.all(Array(100).fill(null).map(() => increment()));
console.log(count); // อาจได้ค่าน้อยกว่า 100!

// ✅ แก้ด้วย Mutex
class Mutex {
  constructor() {
    this._queue = [];
    this._locked = false;
  }

  async acquire() {
    if (!this._locked) {
      this._locked = true;
      return;
    }
    await new Promise(resolve => this._queue.push(resolve));
  }

  release() {
    if (this._queue.length > 0) {
      const next = this._queue.shift();
      next();
    } else {
      this._locked = false;
    }
  }

  async runExclusive(fn) {
    await this.acquire();
    try {
      return await fn();
    } finally {
      this.release();
    }
  }
}

const mutex = new Mutex();
let safeCount = 0;

async function safeIncrement() {
  await mutex.runExclusive(async () => {
    const current = safeCount;
    await delay(1);
    safeCount = current + 1;
  });
}

await Promise.all(Array(100).fill(null).map(() => safeIncrement()));
console.log(safeCount); // 100 ถูกต้องแน่นอน
```

### 6. การ Debug Network Issues

```javascript
// ดักจับ HTTP requests ทั้งหมด
const http = require('http');
const https = require('https');

function interceptRequests() {
  const originalHttpRequest = http.request;
  const originalHttpsRequest = https.request;

  function logRequest(options, protocol) {
    const url = `${protocol}://${options.hostname}${options.path}`;
    console.log(`[HTTP] ${options.method || 'GET'} ${url}`);
  }

  http.request = function(options, callback) {
    logRequest(options, 'http');
    return originalHttpRequest.call(this, options, callback);
  };

  https.request = function(options, callback) {
    logRequest(options, 'https');
    return originalHttpsRequest.call(this, options, callback);
  };
}

interceptRequests();
```

### 7. Debugging Tools Summary

```javascript
// เครื่องมือที่แนะนำ:

// 1. ndb - Chrome DevTools สำหรับ Node.js
// npm install -g ndb
// ndb app.js

// 2. node-clinic - Performance analysis
// npm install -g clinic
// clinic doctor -- node app.js

// 3. 0x - Flame graphs
// npm install -g 0x
// 0x app.js

// 4. why-is-node-running - หา events ที่ค้างอยู่
// npm install -g why-is-node-running
const wtf = require('why-is-node-running');
// เรียก setTimeout(() => wtf.dump(), 5000) เมื่อ node ไม่ exit

// 5. v8-profiler-next - Heap profiling
// npm install v8-profiler-next

// 6. heapdump - Manual heap snapshots
// npm install heapdump
const heapdump = require('heapdump');
process.on('SIGUSR2', () => {
  heapdump.writeSnapshot('./heapdump-' + Date.now() + '.heapsnapshot');
});
// ส่ง signal: kill -USR2 <pid>
```

---

## แบบฝึกหัด {#แบบฝึกหัด}

### แบบฝึกหัดที่ 1: Debug Buggy Code

โค้ดด้านล่างมี bugs หลายอย่าง ให้หาและแก้ไข:

```javascript
// bugs.js - มีปัญหาหลายอย่าง หาและแก้ไข
const EventEmitter = require('events');
const cache = {};

class UserService extends EventEmitter {
  constructor() {
    super();
    this.users = [];
  }

  async createUser(userData) {
    // Bug 1: ไม่ validate input
    const user = {
      id: this.users.length, // Bug 2: id ซ้ำได้เมื่อลบ user
      ...userData,
      createdAt: Date.now
    };

    this.users.push(user);
    this.emit('userCreated', user);

    // Bug 3: ไม่ return
    cache[user.id] = user;
  }

  async getUserById(id) {
    // Bug 4: ไม่ handle cache miss
    if (cache[id]) return cache[id];
    return this.users.find(u => u.id == id); // Bug 5: == แทน ===
  }

  async deleteUser(id) {
    const index = this.users.findIndex(u => u.id === id);
    this.users.splice(index, 1); // Bug 6: ถ้าไม่พบ index = -1
    delete cache[id];
    // Bug 7: ไม่ emit event
  }

  getStats() {
    return {
      total: this.users.length,
      // Bug 8: ล้มเหลวถ้า users ว่าง
      latest: this.users[this.users.length - 1].name
    };
  }
}

module.exports = UserService;
```

### แบบฝึกหัดที่ 2: Performance Investigation

สร้าง Express API ที่ช้า และใช้เครื่องมือ profiling หาจุดที่ช้า:
1. สร้าง route ที่ทำงานช้า (จงใจ)
2. ใช้ `clinic flame` วิเคราะห์
3. แก้ไขให้เร็วขึ้น
4. เปรียบเทียบผลลัพธ์ด้วย autocannon

### แบบฝึกหัดที่ 3: Memory Leak Hunt

```javascript
// memory-leak-app.js - มี memory leak ซ่อนอยู่
// ใช้เครื่องมือที่เรียนมาหาและแก้ไข

const express = require('express');
const app = express();

const requestHistory = []; // ปัญหาที่ 1?
const eventEmitter = new EventEmitter();

app.use((req, res, next) => {
  requestHistory.push({
    url: req.url,
    time: Date.now(),
    headers: req.headers // ปัญหาที่ 2?
  });
  next();
});

app.get('/data', (req, res) => {
  const processData = (items) => {
    const bigBuffer = Buffer.alloc(1024 * 1024); // 1MB
    // bigBuffer ถูก close over แต่ไม่ใช้ - ปัญหาที่ 3?

    return items.map(item => ({
      ...item,
      processed: true
    }));
  };

  eventEmitter.on('process', processData); // ปัญหาที่ 4?

  const data = Array(1000).fill({ id: 1, name: 'test' });
  res.json(processData(data));
});
```

---

## เครื่องมือที่ควรมีใน Toolkit

| เครื่องมือ | ประเภท | ใช้เมื่อ |
|-----------|--------|---------|
| VS Code Debugger | IDE | Debug ทั่วไป, step through code |
| Chrome DevTools | Browser | Profiling, Heap analysis |
| node --inspect | CLI | Quick debug |
| clinic.js | CLI | Performance analysis |
| 0x | CLI | Flame graphs |
| autocannon | CLI | Load testing |
| debug package | npm | Conditional logging |
| heapdump | npm | Manual heap snapshots |
| why-is-node-running | npm | หา hanging operations |

---

## สรุปทั้งหมด Part 06-10

| Part | หัวข้อ | สิ่งสำคัญ |
|------|--------|-----------|
| 06 | Events | EventEmitter, patterns, memory leak prevention |
| 07 | Streams | Buffer, Readable/Writable/Transform, pipeline |
| 08 | Async | Callbacks → Promises → async/await, combinators |
| 09 | Errors | Custom classes, global handlers, HTTP responses |
| 10 | Debugging | Console, VS Code, Chrome DevTools, profiling |

---

## ก้าวต่อไป

➡️ **Part 11: HTTP และ Express.js Basics** - เริ่มสร้าง Web Server จริงๆ

---
*Node.js/Express.js Course - Part 10 of 20*
*ขั้นตอนที่ 901-1000 จาก 1000*
