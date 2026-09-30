# Part 01: แนะนำ Node.js และการติดตั้ง
## ขั้นตอนที่ 1-50: รู้จัก Node.js ก่อนเริ่มต้น

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. อธิบายได้ว่า Node.js คืออะไรและทำงานอย่างไร
2. ติดตั้ง Node.js และ npm ได้
3. รันโปรแกรม JavaScript บน Node.js ได้
4. เข้าใจ Event Loop ของ Node.js
5. ใช้ REPL ของ Node.js ได้
6. สร้างโปรแกรม "Hello World" แรกได้

---

## ขั้นตอนที่ 1: Node.js คืออะไร?

### ประวัติความเป็นมา

**Node.js** ถูกสร้างขึ้นในปี 2009 โดย **Ryan Dahl** เขาต้องการแก้ปัญหาการเขียน Web Server ที่สามารถจัดการ Request จำนวนมากได้พร้อมกัน โดยไม่ต้องใช้ Thread จำนวนมาก

```
ปัญหาเดิม (ก่อน Node.js):
┌─────────────────────────────────────┐
│  Request 1 → Thread 1 (รอ I/O...)  │
│  Request 2 → Thread 2 (รอ I/O...)  │
│  Request 3 → Thread 3 (รอ I/O...)  │
│  Request N → Thread N (รอ I/O...)  │
│                                     │
│  ปัญหา: Thread มีจำกัด, ใช้ RAM มาก │
└─────────────────────────────────────┘

วิธีแก้ของ Node.js:
┌─────────────────────────────────────┐
│  1 Thread (Event Loop)              │
│  Request 1 → ส่งไป I/O → ทำงานอื่น │
│  Request 2 → ส่งไป I/O → ทำงานอื่น │
│  Request 3 → ส่งไป I/O → ทำงานอื่น │
│  I/O เสร็จ → Callback → ส่งคืน    │
└─────────────────────────────────────┘
```

### Node.js คือ JavaScript Runtime

```
Node.js = V8 Engine (Chrome) + libuv + Node.js APIs

┌─────────────────────────────────────────────┐
│              Node.js                        │
│  ┌─────────┐  ┌────────┐  ┌─────────────┐  │
│  │   V8    │  │ libuv  │  │  Node APIs  │  │
│  │ (JS     │  │(async  │  │  (fs, http, │  │
│  │ Engine) │  │ I/O)   │  │  crypto...) │  │
│  └─────────┘  └────────┘  └─────────────┘  │
└─────────────────────────────────────────────┘
```

**V8 Engine:** แปลง JavaScript เป็น Machine Code (เร็วมาก)  
**libuv:** จัดการ I/O แบบ Asynchronous (ไม่บล็อก)  
**Node APIs:** ฟังก์ชันสำหรับเข้าถึง OS เช่น file, network, process

---

## ขั้นตอนที่ 2: Node.js ต่างจาก Browser JavaScript อย่างไร?

```
Browser JavaScript:              Node.js:
┌──────────────────┐             ┌──────────────────┐
│ window object    │             │ global object    │
│ document (DOM)   │             │ process object   │
│ localStorage     │             │ __dirname        │
│ fetch API        │             │ __filename       │
│ alert/confirm    │             │ require()        │
│ XMLHttpRequest   │             │ fs module        │
│                  │             │ http module      │
│ ไม่มี:           │             │ path module      │
│ - fs (ไฟล์)     │             │ os module        │
│ - process        │             │ child_process    │
│ - crypto native  │             │                  │
└──────────────────┘             └──────────────────┘
```

### ตัวอย่างความแตกต่าง

```javascript
// ❌ ทำใน Browser ได้ แต่ Node.js ไม่รู้จัก
document.getElementById('title').innerHTML = 'Hello';
window.location.href = 'https://example.com';
localStorage.setItem('key', 'value');

// ✅ ทำได้เฉพาะ Node.js
const fs = require('fs');
fs.readFileSync('./file.txt', 'utf8');
process.exit(0);
console.log(__dirname);  // path ของ directory ปัจจุบัน

// ✅ ทำได้ทั้ง Browser และ Node.js
console.log('Hello World');
const arr = [1, 2, 3].map(x => x * 2);
setTimeout(() => console.log('delayed'), 1000);
```

---

## ขั้นตอนที่ 3: ทำไมต้องใช้ Node.js?

### จุดแข็ง

```
1. ⚡ เร็ว
   - V8 Engine คอมไพล์ JS เป็น Machine Code
   - Non-blocking I/O รับ Request ได้พร้อมกันมาก

2. 🔄 Asynchronous
   - ไม่รอ I/O เสร็จก่อนทำงานต่อ
   - เหมาะกับ Real-time apps, APIs

3. 📦 npm Ecosystem
   - Package มากกว่า 2 ล้านตัว
   - ใหญ่ที่สุดในโลก

4. 💡 JavaScript ทั้ง Frontend และ Backend
   - เรียนภาษาเดียวใช้ได้ทั้งคู่
   - แชร์ Code บางส่วนได้

5. 🌐 Community
   - ชุมชนใหญ่มาก
   - อัพเดตบ่อย
   - บริษัทใหญ่ใช้ (Netflix, LinkedIn, Uber)
```

### จุดอ่อน

```
1. ⚠️ CPU-intensive Tasks
   - การคำนวณหนักจะบล็อก Event Loop
   - ต้องใช้ Worker Threads หรือ Child Process

2. ⚠️ Callback Hell (แต่แก้ได้ด้วย async/await)

3. ⚠️ Single Threaded
   - ถ้า Error ไม่ถูก Handle → Server Crash

4. ⚠️ ไม่เหมาะกับงาน CPU-intensive
   - เช่น Video encoding, Image processing หนักๆ
```

---

## ขั้นตอนที่ 4: บริษัทที่ใช้ Node.js

```
🎬 Netflix    - Streaming service, startup time ลด 70%
💼 LinkedIn   - Mobile backend
🚗 Uber       - Matching algorithm, real-time
💰 PayPal     - API layer, ลดเวลา response 35%
🛒 Walmart    - Black Friday traffic
💬 Slack      - Real-time messaging
🐦 Twitter    - Social graph API
```

---

## ขั้นตอนที่ 5: ติดตั้ง Node.js

### วิธีที่ 1: ติดตั้งโดยตรงจาก nodejs.org (แนะนำสำหรับผู้เริ่มต้น)

```bash
# ไปที่ https://nodejs.org
# เลือก LTS (Long Term Support) version
# Download และติดตั้งตาม OS ของคุณ

# ตรวจสอบการติดตั้ง
node --version    # ควรแสดง v20.x.x หรือใหม่กว่า
npm --version     # ควรแสดง 10.x.x หรือใหม่กว่า
```

### วิธีที่ 2: ใช้ nvm (Node Version Manager) - แนะนำสำหรับ Developer

```bash
# ติดตั้ง nvm บน macOS/Linux
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# หรือบน Windows ใช้ nvm-windows
# ดาวน์โหลดจาก: https://github.com/coreybutler/nvm-windows

# หลังติดตั้ง nvm แล้ว
nvm install --lts          # ติดตั้ง Node.js LTS ล่าสุด
nvm install 20             # ติดตั้ง Node.js version 20
nvm use 20                 # ใช้ Node.js version 20
nvm ls                     # แสดง version ที่มี
nvm ls-remote              # แสดง version ที่ download ได้
nvm alias default 20       # ตั้ง default version

# ข้อดีของ nvm: สามารถสลับ version ได้ง่าย
nvm use 18    # สลับไปใช้ Node.js 18
nvm use 20    # สลับกลับมาใช้ Node.js 20
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Node.js
node --version
# Output: v20.10.0

# ตรวจสอบ npm
npm --version
# Output: 10.2.3

# ตรวจสอบ npx
npx --version
# Output: 10.2.3

# ดู path ของ Node.js
which node
# macOS/Linux: /usr/local/bin/node
# หรือ: /home/user/.nvm/versions/node/v20.10.0/bin/node
```

---

## ขั้นตอนที่ 6: โปรแกรมแรก - Hello World

### สร้างไฟล์แรก

```bash
# สร้าง folder สำหรับฝึก
mkdir nodejs-course
cd nodejs-course

# สร้างไฟล์ hello.js
touch hello.js
```

```javascript
// hello.js - โปรแกรมแรกของคุณ

console.log('Hello, World!');
console.log('สวัสดี Node.js!');
console.log('ยินดีต้อนรับสู่การเขียน Backend!');
```

```bash
# รันโปรแกรม
node hello.js

# Output:
# Hello, World!
# สวัสดี Node.js!
# ยินดีต้อนรับสู่การเขียน Backend!
```

### เพิ่ม Logic เบื้องต้น

```javascript
// hello-advanced.js

// แสดงข้อมูล Environment
console.log('Node.js Version:', process.version);
console.log('Platform:', process.platform);
console.log('Architecture:', process.arch);
console.log('Current Directory:', process.cwd());

// การคำนวณ
const num1 = 10;
const num2 = 5;
console.log(`\n${num1} + ${num2} = ${num1 + num2}`);
console.log(`${num1} * ${num2} = ${num1 * num2}`);

// Array และ Loop
const fruits = ['มะม่วง', 'กล้วย', 'ส้ม', 'แอปเปิ้ล'];
console.log('\nรายการผลไม้:');
fruits.forEach((fruit, index) => {
  console.log(`  ${index + 1}. ${fruit}`);
});

// Object
const person = {
  name: 'สมชาย',
  age: 25,
  job: 'Backend Developer'
};
console.log('\nข้อมูลผู้ใช้:');
console.log(`  ชื่อ: ${person.name}`);
console.log(`  อายุ: ${person.age} ปี`);
console.log(`  อาชีพ: ${person.job}`);
```

---

## ขั้นตอนที่ 7: Node.js REPL (Read-Eval-Print Loop)

REPL คือ Interactive Console สำหรับทดสอบ JavaScript

```bash
# เปิด REPL
node

# จะเห็น prompt: >
```

```javascript
// ใน REPL พิมพ์:
> 1 + 1
2

> 'Hello' + ' World'
'Hello World'

> const arr = [1, 2, 3, 4, 5]
undefined

> arr.filter(x => x > 2)
[ 3, 4, 5 ]

> arr.map(x => x * 2)
[ 2, 4, 6, 8, 10 ]

> arr.reduce((sum, x) => sum + x, 0)
15

// Object
> const obj = { name: 'Node.js', version: 20 }
undefined

> obj
{ name: 'Node.js', version: 20 }

> obj.name
'Node.js'

// ใช้ _ เพื่อเข้าถึงผลลัพธ์ล่าสุด
> 2 + 2
4
> _ * 3
12

// คำสั่ง REPL พิเศษ
> .help       // แสดง commands ทั้งหมด
> .exit       // ออกจาก REPL (หรือ Ctrl+C สองครั้ง)
> .clear      // ล้าง context
> .save file.js   // บันทึก session ลงไฟล์
> .load file.js   // โหลดไฟล์เข้า REPL
```

### REPL Mode พิเศษ

```bash
# เปิด REPL พร้อม require ไม่ต้องพิมพ์ require()
node -e "console.log('Hello from -e flag')"

# รันสคริปต์แบบ one-liner
node -e "const sum = [1,2,3].reduce((a,b)=>a+b,0); console.log('Sum:', sum)"

# ดู Node.js documentation ใน Terminal
node --help
```

---

## ขั้นตอนที่ 8: Global Objects ใน Node.js

### process object

```javascript
// process.js - ทดสอบ process object

// Version information
console.log('Node version:', process.version);
console.log('V8 version:', process.versions.v8);
console.log('npm version:', process.env.npm_package_version);

// Platform info
console.log('Platform:', process.platform);      // 'linux', 'darwin', 'win32'
console.log('Architecture:', process.arch);       // 'x64', 'arm64'
console.log('PID:', process.pid);                 // Process ID

// Memory usage
const memUsage = process.memoryUsage();
console.log('\nMemory Usage:');
console.log('  RSS:', Math.round(memUsage.rss / 1024 / 1024), 'MB');
console.log('  Heap Total:', Math.round(memUsage.heapTotal / 1024 / 1024), 'MB');
console.log('  Heap Used:', Math.round(memUsage.heapUsed / 1024 / 1024), 'MB');

// CPU usage
const cpuUsage = process.cpuUsage();
console.log('\nCPU Usage:');
console.log('  User:', cpuUsage.user, 'microseconds');
console.log('  System:', cpuUsage.system, 'microseconds');

// Current directory
console.log('\nCurrent Directory:', process.cwd());

// Environment variables
console.log('\nEnvironment Variables:');
console.log('  HOME:', process.env.HOME);
console.log('  PATH (first 50 chars):', process.env.PATH?.substring(0, 50));
console.log('  NODE_ENV:', process.env.NODE_ENV || 'not set');

// Command line arguments
console.log('\nCommand Line Args:', process.argv);
// process.argv[0] = path to node
// process.argv[1] = path to script
// process.argv[2+] = actual arguments

// Uptime
console.log('\nProcess uptime:', process.uptime(), 'seconds');
```

```bash
# รันและดูผลลัพธ์
node process.js

# ส่ง arguments
node process.js arg1 arg2 arg3
```

### __dirname และ __filename

```javascript
// paths.js

console.log('__dirname:', __dirname);
// Output: /home/user/nodejs-course (path ของโฟลเดอร์)

console.log('__filename:', __filename);
// Output: /home/user/nodejs-course/paths.js (path ของไฟล์)

// ใช้ประโยชน์จาก __dirname
const path = require('path');
const configFile = path.join(__dirname, 'config', 'settings.json');
console.log('Config path:', configFile);
// Output: /home/user/nodejs-course/config/settings.json
```

### global object

```javascript
// global.js - global namespace ใน Node.js

// ตัวแปร global ที่มีอยู่แล้ว
console.log('setTimeout:', typeof setTimeout);    // function
console.log('setInterval:', typeof setInterval);  // function
console.log('clearTimeout:', typeof clearTimeout); // function
console.log('console:', typeof console);           // object
console.log('process:', typeof process);           // object
console.log('Buffer:', typeof Buffer);             // function

// สร้าง global variable (ไม่แนะนำ แต่ทำได้)
global.myGlobalVar = 'This is global';

// เข้าถึงได้จากไฟล์อื่น (ถ้า require ไม่ได้ใช้ module scope)
console.log(global.myGlobalVar);

// setTimeout ใน Node.js
console.log('Starting...');
setTimeout(() => {
  console.log('After 1 second');
}, 1000);
console.log('This runs before setTimeout callback!');
// Output:
// Starting...
// This runs before setTimeout callback!
// After 1 second
```

---

## ขั้นตอนที่ 9: Event Loop ของ Node.js

Event Loop เป็นหัวใจสำคัญของ Node.js ที่ทำให้มันเป็น Non-blocking

```
Event Loop Phases:
┌──────────────────────────────────────────────┐
│                  Event Loop                  │
│                                              │
│  ┌──────────┐                                │
│  │  timers  │  ← setTimeout, setInterval     │
│  └─────┬────┘                                │
│        │                                     │
│  ┌─────▼────────┐                            │
│  │  pending     │  ← I/O callbacks           │
│  │  callbacks   │                            │
│  └─────┬────────┘                            │
│        │                                     │
│  ┌─────▼────────┐                            │
│  │  idle,       │  ← Internal use            │
│  │  prepare     │                            │
│  └─────┬────────┘                            │
│        │                                     │
│  ┌─────▼────────┐                            │
│  │    poll      │  ← Wait for I/O events     │
│  └─────┬────────┘                            │
│        │                                     │
│  ┌─────▼────────┐                            │
│  │    check     │  ← setImmediate             │
│  └─────┬────────┘                            │
│        │                                     │
│  ┌─────▼────────────┐                        │
│  │  close callbacks │  ← socket.on('close')  │
│  └──────────────────┘                        │
└──────────────────────────────────────────────┘

Special Queues (ทำก่อน ทุก phase):
- nextTick Queue (process.nextTick)
- Microtask Queue (Promise.resolve)
```

### ตัวอย่าง Event Loop ในทางปฏิบัติ

```javascript
// event-loop-demo.js

console.log('1. Synchronous - Start');

// setTimeout - Timers phase
setTimeout(() => {
  console.log('4. setTimeout (0ms)');
}, 0);

// setImmediate - Check phase
setImmediate(() => {
  console.log('5. setImmediate');
});

// Promise - Microtask Queue
Promise.resolve().then(() => {
  console.log('3. Promise.then (Microtask)');
});

// process.nextTick - nextTick Queue (ก่อน Microtask)
process.nextTick(() => {
  console.log('2. process.nextTick');
});

console.log('1. Synchronous - End');

// Expected Output Order:
// 1. Synchronous - Start
// 1. Synchronous - End
// 2. process.nextTick
// 3. Promise.then (Microtask)
// 4. setTimeout (0ms)    <- หรือ setImmediate อาจมาก่อน
// 5. setImmediate        <- ขึ้นอยู่กับ I/O context
```

### ทำไม Non-blocking ถึงสำคัญ?

```javascript
// blocking-vs-nonblocking.js

// ❌ Blocking (ห้ามทำในโปรแกรมจริง)
function blockingOperation() {
  const start = Date.now();
  // จำลองงานหนักที่ใช้เวลา 3 วินาที
  while (Date.now() - start < 3000) {
    // รอ... บล็อก Event Loop ทั้งหมด
  }
  return 'done';
}

// ถ้าทำแบบนี้ ในระหว่างที่รอ 3 วินาที
// Server จะไม่สามารถรับ Request อื่นได้เลย!

// ✅ Non-blocking (ทำแบบนี้)
function nonBlockingOperation() {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve('done');
    }, 3000);
  });
}

// ในระหว่างที่รอ 3 วินาที Event Loop ยังทำงานได้
// รับ Request อื่นได้ปกติ

async function main() {
  console.log('Starting multiple operations...');
  
  // รัน 3 operations พร้อมกัน
  const [result1, result2, result3] = await Promise.all([
    nonBlockingOperation(),
    nonBlockingOperation(),
    nonBlockingOperation()
  ]);
  
  console.log('All done:', result1, result2, result3);
  // รวมเวลาแค่ ~3 วินาที ไม่ใช่ 9 วินาที!
}

main();
```

---

## ขั้นตอนที่ 10: Module System ใน Node.js

### CommonJS (require/module.exports) - ระบบดั้งเดิม

```javascript
// math.js - สร้าง Module
function add(a, b) {
  return a + b;
}

function subtract(a, b) {
  return a - b;
}

function multiply(a, b) {
  return a * b;
}

function divide(a, b) {
  if (b === 0) throw new Error('Cannot divide by zero');
  return a / b;
}

// Export เฉพาะที่ต้องการ
module.exports = {
  add,
  subtract,
  multiply,
  divide
};
```

```javascript
// main.js - ใช้ Module
const math = require('./math');

console.log('10 + 5 =', math.add(10, 5));       // 15
console.log('10 - 5 =', math.subtract(10, 5));  // 5
console.log('10 * 5 =', math.multiply(10, 5));  // 50
console.log('10 / 5 =', math.divide(10, 5));    // 2

// Destructuring import
const { add, multiply } = require('./math');
console.log('3 + 4 =', add(3, 4));  // 7
```

### วิธี Export แบบต่างๆ

```javascript
// export-styles.js

// สไตล์ที่ 1: Object export
module.exports = {
  greet: (name) => `Hello, ${name}!`,
  farewell: (name) => `Goodbye, ${name}!`
};

// สไตล์ที่ 2: Function export
module.exports = function greet(name) {
  return `Hello, ${name}!`;
};

// สไตล์ที่ 3: Class export
module.exports = class Calculator {
  add(a, b) { return a + b; }
  multiply(a, b) { return a * b; }
};

// สไตล์ที่ 4: Multiple exports ทีละตัว
module.exports.greet = (name) => `Hello, ${name}!`;
module.exports.VERSION = '1.0.0';
module.exports.MAX_SIZE = 100;
```

### ES Modules (import/export) - ระบบใหม่

```javascript
// เปิดใช้ ES Modules: ต้องมี "type": "module" ใน package.json
// หรือใช้นามสกุล .mjs

// math.mjs
export function add(a, b) {
  return a + b;
}

export function multiply(a, b) {
  return a * b;
}

export default function calculate(operation, a, b) {
  switch(operation) {
    case 'add': return add(a, b);
    case 'multiply': return multiply(a, b);
    default: throw new Error('Unknown operation');
  }
}

// main.mjs
import calculate, { add, multiply } from './math.mjs';

console.log(add(1, 2));          // 3
console.log(multiply(3, 4));     // 12
console.log(calculate('add', 5, 6));  // 11
```

---

## ขั้นตอนที่ 11: การรับ Arguments จาก Command Line

```javascript
// args.js - รับ arguments

// process.argv = ['node', 'args.js', 'arg1', 'arg2', ...]
const args = process.argv.slice(2);  // ตัด node และ script path ออก

console.log('Arguments:', args);
console.log('Number of args:', args.length);

// ตัวอย่างการใช้งาน
if (args.length === 0) {
  console.log('กรุณาระบุ arguments');
  console.log('Usage: node args.js <name> <age>');
  process.exit(1);
}

const [name, age] = args;
console.log(`สวัสดี ${name}! คุณอายุ ${age} ปี`);
```

```bash
# รัน
node args.js สมชาย 25
# Output:
# Arguments: [ 'สมชาย', '25' ]
# Number of args: 2
# สวัสดี สมชาย! คุณอายุ 25 ปี
```

### ใช้ minimist หรือ yargs สำหรับ CLI Arguments ที่ซับซ้อน

```bash
npm install minimist
```

```javascript
// cli-args.js
const minimist = require('minimist');

const argv = minimist(process.argv.slice(2), {
  string: ['name', 'output'],
  number: ['age', 'port'],
  boolean: ['verbose', 'help'],
  default: {
    port: 3000,
    verbose: false
  },
  alias: {
    h: 'help',
    v: 'verbose',
    n: 'name',
    p: 'port'
  }
});

if (argv.help) {
  console.log(`
  Usage: node cli-args.js [options]
  
  Options:
    -n, --name     ชื่อผู้ใช้
    --age          อายุ
    -p, --port     Port number (default: 3000)
    -v, --verbose  แสดงข้อมูลเพิ่มเติม
    -h, --help     แสดง help
  `);
  process.exit(0);
}

console.log('Name:', argv.name);
console.log('Age:', argv.age);
console.log('Port:', argv.port);
console.log('Verbose:', argv.verbose);
```

```bash
# รัน
node cli-args.js --name สมชาย --age 25 -p 8080 -v
# Output:
# Name: สมชาย
# Age: 25
# Port: 8080
# Verbose: true
```

---

## ขั้นตอนที่ 12: Console Methods

```javascript
// console-methods.js

// console.log - ข้อมูลทั่วไป
console.log('Normal message');
console.log('With value:', 42);
console.log('With object:', { name: 'Node', version: 20 });

// console.error - Error (แสดงที่ stderr)
console.error('Error message');
console.error(new Error('Something went wrong'));

// console.warn - Warning
console.warn('Warning: Deprecated function used');

// console.info - Information (เหมือน log)
console.info('Server started on port 3000');

// console.debug - Debug info
console.debug('Debug info:', { data: 'test' });

// console.table - แสดงเป็นตาราง
const users = [
  { id: 1, name: 'สมชาย', age: 25 },
  { id: 2, name: 'สมหญิง', age: 30 },
  { id: 3, name: 'สมศรี', age: 22 }
];
console.table(users);

// console.dir - แสดง properties ของ object
const obj = {
  name: 'test',
  nested: {
    deep: {
      value: 42
    }
  }
};
console.dir(obj, { depth: null, colors: true });

// console.time / timeEnd - วัดเวลา
console.time('operation');
let sum = 0;
for (let i = 0; i < 1000000; i++) {
  sum += i;
}
console.timeEnd('operation');
// Output: operation: 5.123ms

// console.count - นับจำนวนครั้ง
for (let i = 0; i < 5; i++) {
  console.count('loop');
}
// Output:
// loop: 1
// loop: 2
// loop: 3
// loop: 4
// loop: 5

// console.group / groupEnd - จัดกลุ่ม output
console.group('User Info');
console.log('Name: สมชาย');
console.log('Age: 25');
console.group('Address');
console.log('City: Bangkok');
console.log('Country: Thailand');
console.groupEnd();
console.groupEnd();

// console.assert - แสดง error ถ้า condition เป็น false
console.assert(1 === 1, 'This should not show');
console.assert(1 === 2, 'This WILL show: 1 is not 2');

// console.clear - ล้างหน้าจอ terminal
// console.clear();
```

---

## ขั้นตอนที่ 13: การจัดการ Environment Variables

```javascript
// env-demo.js

// อ่าน environment variable
const port = process.env.PORT || 3000;
const dbHost = process.env.DB_HOST || 'localhost';
const nodeEnv = process.env.NODE_ENV || 'development';

console.log('Port:', port);
console.log('DB Host:', dbHost);
console.log('Environment:', nodeEnv);

// ตั้งค่าผ่าน command line:
// PORT=8080 DB_HOST=prod-server node env-demo.js
```

### ใช้ dotenv package

```bash
npm install dotenv
```

```bash
# .env file
PORT=3000
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp
DB_USER=admin
DB_PASSWORD=secret123
NODE_ENV=development
JWT_SECRET=my-super-secret-key
```

```javascript
// app.js
require('dotenv').config();  // โหลด .env ก่อนเสมอ

const port = process.env.PORT;
const dbConfig = {
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  name: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD
};

console.log('App running on port:', port);
console.log('Database config:', dbConfig);

// ⚠️ อย่าลืม: ห้าม commit ไฟล์ .env ขึ้น Git!
// เพิ่มใน .gitignore:
// .env
// .env.local
// .env.production
```

---

## ขั้นตอนที่ 14: สร้าง package.json

```bash
# สร้าง package.json แบบ interactive
npm init

# สร้าง package.json อัตโนมัติ (ใช้ defaults)
npm init -y
```

```json
{
  "name": "my-node-app",
  "version": "1.0.0",
  "description": "แอพ Node.js แรกของฉัน",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "jest",
    "lint": "eslint .",
    "build": "node build.js"
  },
  "keywords": ["nodejs", "api"],
  "author": "ชื่อคุณ <email@example.com>",
  "license": "MIT",
  "dependencies": {
    "express": "^4.18.2",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.1",
    "jest": "^29.7.0",
    "eslint": "^8.50.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

### npm Scripts

```bash
# รัน script
npm start         # รัน "start" script
npm test          # รัน "test" script
npm run dev       # รัน "dev" script (ต้องใช้ run สำหรับชื่อที่ไม่ใช่ start/test)
npm run lint      # รัน "lint" script
npm run build     # รัน "build" script

# npm scripts สามารถใช้ร่วมกันได้
{
  "scripts": {
    "clean": "rm -rf dist",
    "compile": "tsc",
    "build": "npm run clean && npm run compile",
    "prestart": "npm run build",  // รันก่อน start อัตโนมัติ
    "start": "node dist/index.js",
    "poststart": "echo 'Server started!'"  // รันหลัง start อัตโนมัติ
  }
}
```

---

## ขั้นตอนที่ 15: ติดตั้งและใช้ nodemon (Development Tool)

```bash
# ติดตั้ง nodemon (auto-restart server เมื่อไฟล์เปลี่ยน)
npm install -D nodemon

# รัน
npx nodemon index.js

# หรือใส่ใน package.json scripts
# "dev": "nodemon index.js"
# แล้วรัน
npm run dev
```

```json
// nodemon.json - configuration
{
  "watch": ["src"],
  "ext": "js,json",
  "ignore": ["src/**/*.test.js"],
  "exec": "node src/index.js",
  "env": {
    "NODE_ENV": "development"
  },
  "delay": "500"
}
```

---

## ขั้นตอนที่ 16: โปรเจกต์แรก - Simple Calculator CLI

```bash
# สร้างโฟลเดอร์
mkdir calculator-cli
cd calculator-cli
npm init -y
```

```javascript
// calculator.js - Calculator CLI

const readline = require('readline');

// สร้าง interface สำหรับรับ input
const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout
});

// ฟังก์ชันถามคำถาม
function question(prompt) {
  return new Promise(resolve => {
    rl.question(prompt, resolve);
  });
}

// ฟังก์ชันคำนวณ
function calculate(num1, operator, num2) {
  switch (operator) {
    case '+': return num1 + num2;
    case '-': return num1 - num2;
    case '*': return num1 * num2;
    case '/':
      if (num2 === 0) throw new Error('ไม่สามารถหารด้วย 0 ได้');
      return num1 / num2;
    case '%': return num1 % num2;
    case '**': return Math.pow(num1, num2);
    default: throw new Error(`ไม่รู้จัก operator: ${operator}`);
  }
}

// Main function
async function main() {
  console.log('╔════════════════════════╗');
  console.log('║   Calculator CLI v1.0  ║');
  console.log('╚════════════════════════╝');
  console.log('Operations: + - * / % **');
  console.log("Type 'quit' to exit\n");

  while (true) {
    try {
      // รับ input
      const num1Str = await question('ตัวเลขที่ 1: ');
      
      if (num1Str.toLowerCase() === 'quit') break;
      
      const operator = await question('เครื่องหมาย (+, -, *, /, %, **): ');
      const num2Str = await question('ตัวเลขที่ 2: ');
      
      // แปลงเป็นตัวเลข
      const num1 = parseFloat(num1Str);
      const num2 = parseFloat(num2Str);
      
      // ตรวจสอบว่าเป็นตัวเลข
      if (isNaN(num1) || isNaN(num2)) {
        console.log('❌ กรุณาใส่ตัวเลขที่ถูกต้อง\n');
        continue;
      }
      
      // คำนวณ
      const result = calculate(num1, operator, num2);
      
      // แสดงผล
      console.log(`\n✅ ${num1} ${operator} ${num2} = ${result}\n`);
      
    } catch (error) {
      console.log(`❌ Error: ${error.message}\n`);
    }
  }
  
  console.log('\nขอบคุณที่ใช้ Calculator! 👋');
  rl.close();
}

main();
```

```bash
# รัน
node calculator.js

# ลองใช้:
# ตัวเลขที่ 1: 10
# เครื่องหมาย: *
# ตัวเลขที่ 2: 5
# ✅ 10 * 5 = 50
```

---

## ขั้นตอนที่ 17: โปรเจกต์ที่ 2 - Temperature Converter

```javascript
// temp-converter.js - แปลงอุณหภูมิ

// Conversion functions
const conversions = {
  celsiusToFahrenheit: (c) => (c * 9/5) + 32,
  fahrenheitToCelsius: (f) => (f - 32) * 5/9,
  celsiusToKelvin: (c) => c + 273.15,
  kelvinToCelsius: (k) => k - 273.15,
  fahrenheitToKelvin: (f) => (f + 459.67) * 5/9,
  kelvinToFahrenheit: (k) => (k * 9/5) - 459.67
};

function convertTemperature(value, from, to) {
  const key = `${from.toLowerCase()}To${to.charAt(0).toUpperCase()}${to.slice(1).toLowerCase()}`;
  
  if (from.toLowerCase() === to.toLowerCase()) {
    return value;
  }
  
  if (!conversions[key]) {
    throw new Error(`Cannot convert from ${from} to ${to}`);
  }
  
  return conversions[key](value);
}

// รับ arguments จาก command line
const args = process.argv.slice(2);

if (args.length !== 3) {
  console.log('Usage: node temp-converter.js <value> <from> <to>');
  console.log('Example: node temp-converter.js 100 celsius fahrenheit');
  console.log('Units: celsius, fahrenheit, kelvin');
  process.exit(1);
}

const [valueStr, from, to] = args;
const value = parseFloat(valueStr);

if (isNaN(value)) {
  console.error('Error: value must be a number');
  process.exit(1);
}

try {
  const result = convertTemperature(value, from, to);
  console.log(`${value}° ${from} = ${result.toFixed(2)}° ${to}`);
} catch (error) {
  console.error('Error:', error.message);
  process.exit(1);
}
```

```bash
# ทดสอบ
node temp-converter.js 100 celsius fahrenheit
# Output: 100° celsius = 212.00° fahrenheit

node temp-converter.js 98.6 fahrenheit celsius
# Output: 98.6° fahrenheit = 37.00° celsius

node temp-converter.js 300 kelvin celsius
# Output: 300° kelvin = 26.85° celsius
```

---

## ขั้นตอนที่ 18: โปรเจกต์ที่ 3 - Simple Todo CLI

```javascript
// todo-cli.js - Todo List ง่ายๆ ใน Terminal

const fs = require('fs');
const path = require('path');
const readline = require('readline');

const TODO_FILE = path.join(__dirname, 'todos.json');

// โหลด todos จากไฟล์
function loadTodos() {
  if (!fs.existsSync(TODO_FILE)) {
    return [];
  }
  const data = fs.readFileSync(TODO_FILE, 'utf8');
  return JSON.parse(data);
}

// บันทึก todos ลงไฟล์
function saveTodos(todos) {
  fs.writeFileSync(TODO_FILE, JSON.stringify(todos, null, 2));
}

// แสดง todos ทั้งหมด
function listTodos(todos) {
  if (todos.length === 0) {
    console.log('📭 ยังไม่มี Todo');
    return;
  }
  
  console.log('\n📋 รายการ Todo:');
  console.log('─'.repeat(50));
  
  todos.forEach((todo, index) => {
    const status = todo.done ? '✅' : '⬜';
    const text = todo.done ? `\x1b[9m${todo.text}\x1b[0m` : todo.text;  // strikethrough ถ้าเสร็จ
    console.log(`${status} ${index + 1}. ${text}`);
  });
  
  const done = todos.filter(t => t.done).length;
  console.log('─'.repeat(50));
  console.log(`รวม: ${todos.length} งาน | เสร็จ: ${done} | ค้าง: ${todos.length - done}\n`);
}

// เพิ่ม todo
function addTodo(todos, text) {
  todos.push({
    id: Date.now(),
    text: text,
    done: false,
    createdAt: new Date().toISOString()
  });
  saveTodos(todos);
  console.log(`✅ เพิ่ม: "${text}"`);
}

// ทำ todo เสร็จ
function completeTodo(todos, index) {
  if (index < 1 || index > todos.length) {
    console.log('❌ ไม่พบ todo ที่ระบุ');
    return;
  }
  todos[index - 1].done = !todos[index - 1].done;
  const status = todos[index - 1].done ? 'เสร็จแล้ว' : 'ยังไม่เสร็จ';
  saveTodos(todos);
  console.log(`✅ "${todos[index - 1].text}" - ${status}`);
}

// ลบ todo
function removeTodo(todos, index) {
  if (index < 1 || index > todos.length) {
    console.log('❌ ไม่พบ todo ที่ระบุ');
    return todos;
  }
  const removed = todos.splice(index - 1, 1)[0];
  saveTodos(todos);
  console.log(`🗑️ ลบแล้ว: "${removed.text}"`);
  return todos;
}

// Main
async function main() {
  let todos = loadTodos();
  
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout
  });
  
  function question(prompt) {
    return new Promise(resolve => rl.question(prompt, resolve));
  }
  
  console.log('╔══════════════════════════╗');
  console.log('║   Todo CLI Manager v1.0  ║');
  console.log('╚══════════════════════════╝');
  console.log('Commands: list, add, done, remove, clear, quit\n');
  
  while (true) {
    const input = await question('> ');
    const parts = input.trim().split(' ');
    const command = parts[0].toLowerCase();
    const rest = parts.slice(1).join(' ');
    
    switch (command) {
      case 'list':
      case 'ls':
        listTodos(todos);
        break;
        
      case 'add':
      case 'a':
        if (!rest) {
          console.log('Usage: add <task description>');
        } else {
          addTodo(todos, rest);
        }
        break;
        
      case 'done':
      case 'd':
        const doneIndex = parseInt(rest);
        if (isNaN(doneIndex)) {
          console.log('Usage: done <number>');
        } else {
          completeTodo(todos, doneIndex);
        }
        break;
        
      case 'remove':
      case 'rm':
        const rmIndex = parseInt(rest);
        if (isNaN(rmIndex)) {
          console.log('Usage: remove <number>');
        } else {
          todos = removeTodo(todos, rmIndex);
        }
        break;
        
      case 'clear':
        todos = [];
        saveTodos(todos);
        console.log('🗑️ ล้าง todo ทั้งหมดแล้ว');
        break;
        
      case 'quit':
      case 'q':
      case 'exit':
        console.log('\nBye! 👋');
        rl.close();
        process.exit(0);
        break;
        
      case '':
        break;
        
      default:
        console.log(`ไม่รู้จักคำสั่ง: ${command}`);
        console.log('Commands: list, add <text>, done <n>, remove <n>, clear, quit');
    }
  }
}

main();
```

---

## ขั้นตอนที่ 19: ทำความเข้าใจ Node.js Versions

```bash
# ดู version ปัจจุบัน
node --version

# ตรวจสอบ version ที่ support
# LTS = Long Term Support (แนะนำสำหรับ Production)
# Current = version ล่าสุด (มีฟีเจอร์ใหม่ แต่อาจไม่เสถียร)

# Node.js version lifecycle:
# - Active LTS: 30 เดือน (support เต็มที่)
# - Maintenance: 18 เดือน (security patches เท่านั้น)
# - End of Life: หยุด support แล้ว

# ดูข้อมูลเพิ่มเติมที่: https://nodejs.org/en/about/releases/
```

### Node.js Features ตาม Version

```javascript
// Node.js 18+ features

// Fetch API (ไม่ต้อง install node-fetch อีกต่อไป!)
const response = await fetch('https://api.example.com/data');
const data = await response.json();

// Test Runner built-in
import { test } from 'node:test';
test('example test', () => {
  assert.strictEqual(1 + 1, 2);
});

// Node.js 20+ features
// Permission Model
node --experimental-permission --allow-fs-read=. index.js

// Node.js 22+ features  
// require(ESM) - import ES Modules ด้วย require ได้แล้ว!
```

---

## ขั้นตอนที่ 20: สรุป Part 01 และ Exercise

### สิ่งที่เรียนรู้ใน Part 01

```
✅ Node.js คืออะไรและทำงานอย่างไร
✅ ความแตกต่างระหว่าง Browser JS และ Node.js
✅ การติดตั้ง Node.js และ nvm
✅ การรัน JavaScript ด้วย node command
✅ REPL - Interactive JavaScript console
✅ Global objects: process, __dirname, __filename
✅ Event Loop - หัวใจของ Node.js
✅ Module system: require/exports และ import/export
✅ package.json และ npm scripts
✅ Environment variables และ dotenv
✅ nodemon สำหรับ development
✅ สร้าง CLI applications จริง
```

### 📝 Exercise - ทำเองให้ได้!

**Exercise 1: แก้ไขโปรแกรม Hello World**
```
สร้างไฟล์ greet.js ที่:
1. รับชื่อจาก command line argument
2. ถ้าไม่มี argument ให้ถามชื่อผ่าน stdin
3. แสดงข้อความทักทายเป็นภาษาไทย
4. แสดงวันเวลาปัจจุบัน
5. แสดง Node.js version ที่ใช้
```

**Exercise 2: สร้าง Unit Converter**
```
สร้าง converter.js ที่แปลงหน่วยได้:
1. ระยะทาง: กิโลเมตร ↔ ไมล์
2. น้ำหนัก: กิโลกรัม ↔ ปอนด์
3. พื้นที่: ตารางเมตร ↔ ตารางฟุต
รับ arguments: node converter.js 100 km miles
```

**Exercise 3: สร้าง Password Generator**
```
สร้าง password-gen.js ที่:
1. รับความยาวที่ต้องการ (default: 12)
2. รับ options: --uppercase, --numbers, --symbols
3. สร้าง password แบบ random
4. แสดง password strength
```

**Exercise 4: สร้าง File Info Tool**
```
สร้าง file-info.js ที่:
1. รับ path ของไฟล์จาก argument
2. แสดงข้อมูล: ชื่อไฟล์, ขนาด, วันสร้าง, วันแก้ไข
3. แสดงจำนวนบรรทัด (ถ้าเป็นข้อความ)
4. แสดง MIME type ของไฟล์
```

### ✅ เฉลย Exercise 1

```javascript
// greet.js
const readline = require('readline');

async function getInput(prompt) {
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout
  });
  
  return new Promise(resolve => {
    rl.question(prompt, (answer) => {
      rl.close();
      resolve(answer);
    });
  });
}

async function main() {
  let name = process.argv[2];
  
  if (!name) {
    name = await getInput('กรุณาใส่ชื่อของคุณ: ');
  }
  
  const now = new Date();
  const dateStr = now.toLocaleDateString('th-TH', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  });
  const timeStr = now.toLocaleTimeString('th-TH');
  
  const hour = now.getHours();
  let greeting;
  if (hour < 12) greeting = 'สวัสดีตอนเช้า';
  else if (hour < 17) greeting = 'สวัสดีตอนบ่าย';
  else greeting = 'สวัสดีตอนเย็น';
  
  console.log(`\n${greeting} คุณ${name}! 👋`);
  console.log(`📅 วันนี้: ${dateStr}`);
  console.log(`🕐 เวลา: ${timeStr}`);
  console.log(`⚙️ Node.js: ${process.version}`);
}

main().catch(console.error);
```

---

## 🔜 Part ถัดไป

**Part 02: Node.js Core Modules** จะครอบคลุม:
- `path` module - จัดการ file paths
- `os` module - ข้อมูล Operating System
- `url` module - จัดการ URLs
- `util` module - utility functions
- `crypto` module - การเข้ารหัส
- `zlib` module - compression

---

*Part 01 สมบูรณ์ | ขั้นตอนที่ 1-20 จาก 1000*
