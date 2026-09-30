# Part 02: Node.js Core Modules
## ขั้นตอนที่ 21-70: Built-in Modules ที่ต้องรู้จัก

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. ใช้ `path` module จัดการ file paths ได้อย่างถูกต้อง
2. ใช้ `os` module ดึงข้อมูล OS ได้
3. ใช้ `url` module parse และสร้าง URL ได้
4. ใช้ `util` module ช่วยงานทั่วไปได้
5. ใช้ `crypto` module เข้ารหัสข้อมูลพื้นฐานได้
6. ใช้ `zlib` module compress/decompress ได้
7. ใช้ `child_process` รัน command ภายนอกได้
8. ใช้ `worker_threads` สำหรับ CPU-intensive tasks ได้

---

## ขั้นตอนที่ 21: path module

`path` module ช่วยจัดการ file paths อย่างถูกต้องในทุก OS

```javascript
// path-demo.js
const path = require('path');

// ══════════════════════════════════════
// path.join() - รวม paths ให้ถูก OS
// ══════════════════════════════════════
const p1 = path.join('/home', 'user', 'documents', 'file.txt');
console.log('join:', p1);
// Linux/macOS: /home/user/documents/file.txt
// Windows: \home\user\documents\file.txt

// แก้ปัญหา path separator ต่างกัน
const p2 = path.join(__dirname, 'data', 'users.json');
console.log('join with __dirname:', p2);

// ═══════════════════════════════════════
// path.resolve() - สร้าง absolute path
// ═══════════════════════════════════════
console.log('\n--- path.resolve() ---');
console.log(path.resolve('file.txt'));
// /current/working/directory/file.txt

console.log(path.resolve('/home', 'user', 'file.txt'));
// /home/user/file.txt (ถ้า segment เป็น absolute, reset จากตรงนั้น)

console.log(path.resolve('/home', '/etc', 'nginx.conf'));
// /etc/nginx.conf (เริ่มใหม่ตั้งแต่ /etc)

// ════════════════════════════════════════
// path.parse() - แยกส่วนประกอบของ path
// ════════════════════════════════════════
console.log('\n--- path.parse() ---');
const parsed = path.parse('/home/user/documents/report.pdf');
console.log(parsed);
// {
//   root: '/',
//   dir: '/home/user/documents',
//   base: 'report.pdf',
//   ext: '.pdf',
//   name: 'report'
// }

console.log('root:', parsed.root);   // /
console.log('dir:', parsed.dir);     // /home/user/documents
console.log('base:', parsed.base);   // report.pdf
console.log('ext:', parsed.ext);     // .pdf
console.log('name:', parsed.name);   // report

// ════════════════════════════════════════
// path.format() - รวมส่วนประกอบเป็น path
// ════════════════════════════════════════
console.log('\n--- path.format() ---');
const formatted = path.format({
  dir: '/home/user/documents',
  name: 'report',
  ext: '.pdf'
});
console.log(formatted);
// /home/user/documents/report.pdf

// ════════════════════════════════════════
// path.basename() - เอาชื่อไฟล์
// ════════════════════════════════════════
console.log('\n--- path.basename() ---');
console.log(path.basename('/home/user/file.txt'));        // file.txt
console.log(path.basename('/home/user/file.txt', '.txt')); // file (ตัด extension)
console.log(path.basename('/home/user/'));                 // user

// ════════════════════════════════════════
// path.dirname() - เอา directory
// ════════════════════════════════════════
console.log('\n--- path.dirname() ---');
console.log(path.dirname('/home/user/file.txt'));  // /home/user
console.log(path.dirname('/home/user/'));          // /home

// ════════════════════════════════════════
// path.extname() - เอา extension
// ════════════════════════════════════════
console.log('\n--- path.extname() ---');
console.log(path.extname('file.txt'));      // .txt
console.log(path.extname('archive.tar.gz')); // .gz
console.log(path.extname('Makefile'));      // '' (ไม่มี extension)
console.log(path.extname('.gitignore'));    // '' (ชื่อขึ้นต้นด้วย dot)

// ════════════════════════════════════════
// path.relative() - relative path ระหว่างสอง paths
// ════════════════════════════════════════
console.log('\n--- path.relative() ---');
const rel = path.relative('/home/user/documents', '/home/user/downloads/file.txt');
console.log(rel);  // ../downloads/file.txt

// ════════════════════════════════════════
// path.normalize() - แก้ path ที่ไม่ถูกต้อง
// ════════════════════════════════════════
console.log('\n--- path.normalize() ---');
console.log(path.normalize('/home//user/../user/./file.txt'));
// /home/user/file.txt

// ════════════════════════════════════════
// path constants
// ════════════════════════════════════════
console.log('\n--- constants ---');
console.log('sep:', path.sep);        // / บน Linux/macOS, \ บน Windows
console.log('delimiter:', path.delimiter); // : บน Linux/macOS, ; บน Windows
console.log('isAbsolute:', path.isAbsolute('/home/user'));  // true
console.log('isAbsolute:', path.isAbsolute('relative/path')); // false
```

### Path Module - Use Cases จริง

```javascript
// path-use-cases.js
const path = require('path');
const fs = require('fs');

// ═══════════════════════════════════════════
// 1. หา path ของ config file อย่างถูกต้อง
// ═══════════════════════════════════════════
function getConfigPath(filename) {
  // ต้องใช้ __dirname ไม่ใช้ relative path
  // เพราะ relative path เปลี่ยนตาม cwd
  return path.join(__dirname, 'config', filename);
}

const dbConfig = getConfigPath('database.json');
const appConfig = getConfigPath('app.json');
console.log('DB config:', dbConfig);
console.log('App config:', appConfig);

// ═══════════════════════════════════════════
// 2. สร้าง path สำหรับ file uploads
// ═══════════════════════════════════════════
function getUploadPath(userId, filename) {
  // Sanitize filename (ป้องกัน path traversal)
  const sanitized = path.basename(filename);  // ตัด directory separators
  const ext = path.extname(sanitized);
  const name = path.basename(sanitized, ext);
  const safeName = name.replace(/[^a-zA-Z0-9-_]/g, '_') + ext;
  
  return path.join(__dirname, 'uploads', String(userId), safeName);
}

console.log(getUploadPath(123, 'profile.jpg'));
// /project/uploads/123/profile.jpg

console.log(getUploadPath(123, '../../../etc/passwd'));
// /project/uploads/123/passwd (ป้องกัน path traversal แล้ว)

// ═══════════════════════════════════════════
// 3. เปลี่ยน extension ของไฟล์
// ═══════════════════════════════════════════
function changeExtension(filePath, newExt) {
  const { dir, name } = path.parse(filePath);
  return path.format({ dir, name, ext: newExt });
}

console.log(changeExtension('/src/component.jsx', '.js'));
// /src/component.js

console.log(changeExtension('/docs/readme.md', '.html'));
// /docs/readme.html

// ═══════════════════════════════════════════
// 4. หา relative import path
// ═══════════════════════════════════════════
function getRelativeImport(from, to) {
  const fromDir = path.dirname(from);
  let rel = path.relative(fromDir, to);
  if (!rel.startsWith('.')) rel = './' + rel;
  return rel.replace(/\\/g, '/');  // ใช้ / เสมอสำหรับ imports
}

console.log(getRelativeImport('/src/components/Button.jsx', '/src/utils/helpers.js'));
// ../utils/helpers.js

console.log(getRelativeImport('/src/index.js', '/src/components/Button.jsx'));
// ./components/Button.jsx
```

---

## ขั้นตอนที่ 22: os module

```javascript
// os-demo.js
const os = require('os');

console.log('═══════════════════════════════');
console.log('  System Information');
console.log('═══════════════════════════════');

// ข้อมูลระบบปฏิบัติการ
console.log('\n📊 OS Info:');
console.log('  Platform:', os.platform());  // linux, darwin, win32
console.log('  Type:', os.type());           // Linux, Darwin, Windows_NT
console.log('  Release:', os.release());     // kernel version
console.log('  Architecture:', os.arch());   // x64, arm64, ia32
console.log('  Hostname:', os.hostname());   // ชื่อเครื่อง

// ข้อมูล CPU
console.log('\n🖥️ CPU Info:');
const cpus = os.cpus();
console.log('  Number of CPUs:', cpus.length);
console.log('  CPU Model:', cpus[0].model);
console.log('  CPU Speed:', cpus[0].speed, 'MHz');

// แสดงข้อมูล CPU แต่ละ core
cpus.forEach((cpu, index) => {
  const total = Object.values(cpu.times).reduce((a, b) => a + b, 0);
  const idle = cpu.times.idle;
  const usage = ((total - idle) / total * 100).toFixed(1);
  console.log(`  Core ${index}: ${usage}% usage`);
});

// ข้อมูล Memory
console.log('\n💾 Memory Info:');
const totalMem = os.totalmem();
const freeMem = os.freemem();
const usedMem = totalMem - freeMem;

console.log('  Total:', formatBytes(totalMem));
console.log('  Free:', formatBytes(freeMem));
console.log('  Used:', formatBytes(usedMem));
console.log('  Usage:', ((usedMem / totalMem) * 100).toFixed(1) + '%');

// Network interfaces
console.log('\n🌐 Network Interfaces:');
const interfaces = os.networkInterfaces();
for (const [name, addrs] of Object.entries(interfaces)) {
  console.log(`  ${name}:`);
  addrs.forEach(addr => {
    if (addr.family === 'IPv4') {
      console.log(`    IPv4: ${addr.address} (${addr.internal ? 'internal' : 'external'})`);
    }
  });
}

// User Info
console.log('\n👤 User Info:');
const userInfo = os.userInfo();
console.log('  Username:', userInfo.username);
console.log('  Home dir:', userInfo.homedir);
console.log('  Shell:', userInfo.shell);

// System Uptime
console.log('\n⏱️ System Uptime:');
const uptime = os.uptime();
const hours = Math.floor(uptime / 3600);
const minutes = Math.floor((uptime % 3600) / 60);
console.log(`  ${hours}h ${minutes}m`);

// Load Average (Linux/macOS)
if (os.platform() !== 'win32') {
  console.log('\n📈 Load Average (1, 5, 15 min):');
  const loads = os.loadavg();
  console.log('  ', loads.map(l => l.toFixed(2)).join(', '));
}

// Temp directory
console.log('\n📁 Temp Directory:', os.tmpdir());

// EOL (End of Line)
console.log('\n↩️ EOL:', os.EOL === '\r\n' ? 'CRLF (Windows)' : 'LF (Unix)');

// Helper function
function formatBytes(bytes) {
  if (bytes === 0) return '0 B';
  const k = 1024;
  const sizes = ['B', 'KB', 'MB', 'GB', 'TB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i];
}
```

### ใช้ os module ในทางปฏิบัติ

```javascript
// system-monitor.js
const os = require('os');

class SystemMonitor {
  constructor(intervalMs = 1000) {
    this.interval = intervalMs;
    this.history = [];
    this.maxHistory = 60;  // เก็บ 60 samples
  }
  
  getCPUUsage() {
    const cpus = os.cpus();
    let totalIdle = 0;
    let totalTick = 0;
    
    cpus.forEach(cpu => {
      for (const type in cpu.times) {
        totalTick += cpu.times[type];
      }
      totalIdle += cpu.times.idle;
    });
    
    return 100 - (totalIdle / totalTick * 100);
  }
  
  getMemoryUsage() {
    const total = os.totalmem();
    const free = os.freemem();
    const used = total - free;
    
    return {
      total,
      free,
      used,
      percentage: (used / total * 100).toFixed(1)
    };
  }
  
  sample() {
    const data = {
      timestamp: Date.now(),
      cpu: this.getCPUUsage().toFixed(1),
      memory: this.getMemoryUsage()
    };
    
    this.history.push(data);
    if (this.history.length > this.maxHistory) {
      this.history.shift();
    }
    
    return data;
  }
  
  start() {
    console.log('🖥️ System Monitor Started');
    console.log('Press Ctrl+C to stop\n');
    
    this.timer = setInterval(() => {
      const data = this.sample();
      
      // Clear line และแสดงข้อมูล
      process.stdout.write('\r');
      process.stdout.write(
        `CPU: ${data.cpu}% | ` +
        `RAM: ${data.memory.percentage}% ` +
        `(${this.formatBytes(data.memory.used)}/${this.formatBytes(data.memory.total)})`
      );
    }, this.interval);
    
    // Handle Ctrl+C
    process.on('SIGINT', () => {
      clearInterval(this.timer);
      console.log('\n\nMonitor stopped. Final stats:');
      this.printSummary();
      process.exit(0);
    });
  }
  
  printSummary() {
    if (this.history.length === 0) return;
    
    const cpuValues = this.history.map(h => parseFloat(h.cpu));
    const memValues = this.history.map(h => parseFloat(h.memory.percentage));
    
    console.log(`CPU  - Avg: ${avg(cpuValues).toFixed(1)}%, Peak: ${Math.max(...cpuValues).toFixed(1)}%`);
    console.log(`RAM  - Avg: ${avg(memValues).toFixed(1)}%, Peak: ${Math.max(...memValues).toFixed(1)}%`);
    console.log(`Samples: ${this.history.length}`);
  }
  
  formatBytes(bytes) {
    const gb = bytes / 1024 / 1024 / 1024;
    if (gb >= 1) return gb.toFixed(1) + ' GB';
    return (bytes / 1024 / 1024).toFixed(0) + ' MB';
  }
}

function avg(arr) {
  return arr.reduce((a, b) => a + b, 0) / arr.length;
}

const monitor = new SystemMonitor(1000);
monitor.start();
```

---

## ขั้นตอนที่ 23: url module

```javascript
// url-demo.js
const { URL, URLSearchParams } = require('url');

// ═══════════════════════════════════════════
// สร้างและ parse URL
// ═══════════════════════════════════════════
const url = new URL('https://user:password@example.com:8080/path/to/page?name=สมชาย&age=25&tag=nodejs&tag=express#section1');

console.log('Full URL:', url.href);
console.log('\nURL Components:');
console.log('  protocol:', url.protocol);   // https:
console.log('  username:', url.username);   // user
console.log('  password:', url.password);   // password
console.log('  hostname:', url.hostname);   // example.com
console.log('  port:', url.port);           // 8080
console.log('  host:', url.host);           // example.com:8080
console.log('  origin:', url.origin);       // https://example.com:8080
console.log('  pathname:', url.pathname);   // /path/to/page
console.log('  search:', url.search);       // ?name=...&age=25...
console.log('  hash:', url.hash);           // #section1

// ═══════════════════════════════════════════
// URLSearchParams - จัดการ query string
// ═══════════════════════════════════════════
console.log('\n--- URLSearchParams ---');
const params = url.searchParams;

console.log('name:', params.get('name'));    // สมชาย
console.log('age:', params.get('age'));      // 25
console.log('tags:', params.getAll('tag')); // ['nodejs', 'express']
console.log('has name:', params.has('name')); // true
console.log('has email:', params.has('email')); // false

// วน loop ผ่าน params
console.log('\nAll params:');
for (const [key, value] of params) {
  console.log(`  ${key}: ${value}`);
}

// แก้ไข params
params.set('age', '26');
params.append('tag', 'javascript');
params.delete('name');

console.log('\nModified URL:', url.href);

// ═══════════════════════════════════════════
// สร้าง URL จาก parts
// ═══════════════════════════════════════════
console.log('\n--- Building URLs ---');

// วิธีที่ 1: สร้างจาก base + path
const apiBase = 'https://api.example.com';
const endpoint = new URL('/users/123/posts', apiBase);
endpoint.searchParams.set('page', '1');
endpoint.searchParams.set('limit', '10');
console.log('API URL:', endpoint.href);
// https://api.example.com/users/123/posts?page=1&limit=10

// วิธีที่ 2: สร้าง query string
function buildQueryString(params) {
  const sp = new URLSearchParams(params);
  return sp.toString();
}

const query = buildQueryString({
  search: 'Node.js tutorial',
  category: 'programming',
  sort: 'newest',
  page: 1
});
console.log('Query string:', query);
// search=Node.js+tutorial&category=programming&sort=newest&page=1

// ═══════════════════════════════════════════
// URL Validation
// ═══════════════════════════════════════════
function isValidURL(string) {
  try {
    new URL(string);
    return true;
  } catch {
    return false;
  }
}

console.log('\n--- URL Validation ---');
console.log(isValidURL('https://example.com'));        // true
console.log(isValidURL('http://localhost:3000'));       // true
console.log(isValidURL('not a url'));                  // false
console.log(isValidURL('ftp://files.example.com'));    // true

// ═══════════════════════════════════════════
// ใช้กับ Express route
// ═══════════════════════════════════════════
function parseRequestURL(req) {
  // req.url เป็น path + query string เท่านั้น
  // ต้องสร้าง full URL ก่อน
  const fullURL = new URL(req.url, `http://${req.headers.host}`);
  
  return {
    path: fullURL.pathname,
    query: Object.fromEntries(fullURL.searchParams),
    hash: fullURL.hash
  };
}
```

---

## ขั้นตอนที่ 24: util module

```javascript
// util-demo.js
const util = require('util');
const fs = require('fs');

// ═══════════════════════════════════════════
// util.promisify - แปลง callback เป็น Promise
// ═══════════════════════════════════════════
console.log('--- util.promisify ---');

// fs.readFile ปกติใช้ callback
fs.readFile('./package.json', 'utf8', (err, data) => {
  if (err) console.error(err);
  else console.log('Callback version:', data.substring(0, 20));
});

// แปลงเป็น Promise ด้วย promisify
const readFileAsync = util.promisify(fs.readFile);

async function demo() {
  try {
    const data = await readFileAsync('./package.json', 'utf8');
    console.log('Promise version:', data.substring(0, 20));
  } catch (err) {
    console.error(err);
  }
  
  // ═════════════════════════════════════════
  // util.inspect - แสดง object อย่างละเอียด
  // ═════════════════════════════════════════
  console.log('\n--- util.inspect ---');
  
  const complexObject = {
    name: 'Node.js',
    nested: {
      deep: {
        value: 42,
        array: [1, 2, 3]
      }
    },
    fn: function() {},  // ฟังก์ชัน
    sym: Symbol('test'),  // Symbol
    buf: Buffer.from('hello')  // Buffer
  };
  
  // inspect แบบ default
  console.log(util.inspect(complexObject));
  
  // inspect แบบละเอียด (unlimited depth, colors)
  console.log(util.inspect(complexObject, {
    depth: null,
    colors: true,
    showHidden: false,
    compact: false
  }));
  
  // ═════════════════════════════════════════
  // util.format - สร้าง string (เหมือน printf)
  // ═════════════════════════════════════════
  console.log('\n--- util.format ---');
  
  console.log(util.format('Hello %s! You are %d years old.', 'World', 25));
  // Hello World! You are 25 years old.
  
  console.log(util.format('Object: %j', { name: 'test', value: 42 }));
  // Object: {"name":"test","value":42}
  
  console.log(util.format('Multiple:', 1, 'two', [3, 4], { five: 5 }));
  // Multiple: 1 two [ 3, 4 ] { five: 5 }
  
  // ═════════════════════════════════════════
  // util.types - ตรวจสอบ types
  // ═════════════════════════════════════════
  console.log('\n--- util.types ---');
  
  console.log('isPromise:', util.types.isPromise(Promise.resolve()));  // true
  console.log('isRegExp:', util.types.isRegExp(/abc/));                // true
  console.log('isDate:', util.types.isDate(new Date()));              // true
  console.log('isMap:', util.types.isMap(new Map()));                 // true
  console.log('isSet:', util.types.isSet(new Set()));                 // true
  console.log('isUint8Array:', util.types.isUint8Array(new Uint8Array())); // true
  
  // ═════════════════════════════════════════
  // util.deprecate - เตือน deprecated function
  // ═════════════════════════════════════════
  console.log('\n--- util.deprecate ---');
  
  const oldFunction = util.deprecate(
    function(x) { return x * 2; },
    'oldFunction() is deprecated. Use newFunction() instead.',
    'DEP0001'
  );
  
  console.log(oldFunction(5));  // 10 (แต่แสดง deprecation warning)
  
  // ═════════════════════════════════════════
  // util.isDeepStrictEqual
  // ═════════════════════════════════════════
  console.log('\n--- util.isDeepStrictEqual ---');
  
  const obj1 = { a: 1, b: { c: 2 } };
  const obj2 = { a: 1, b: { c: 2 } };
  const obj3 = { a: 1, b: { c: 3 } };
  
  console.log(util.isDeepStrictEqual(obj1, obj2));  // true
  console.log(util.isDeepStrictEqual(obj1, obj3));  // false
  console.log(obj1 === obj2);  // false (reference comparison)
}

demo();
```

---

## ขั้นตอนที่ 25: crypto module

```javascript
// crypto-demo.js
const crypto = require('crypto');

// ═══════════════════════════════════════════
// Hash Functions
// ═══════════════════════════════════════════
console.log('--- Hash Functions ---');

function hash(algorithm, data) {
  return crypto.createHash(algorithm).update(data).digest('hex');
}

const text = 'Hello, Node.js!';
console.log('MD5:', hash('md5', text));
console.log('SHA1:', hash('sha1', text));
console.log('SHA256:', hash('sha256', text));
console.log('SHA512:', hash('sha512', text));

// Hash ไฟล์
const fs = require('fs');
function hashFile(filePath, algorithm = 'sha256') {
  return new Promise((resolve, reject) => {
    const hash = crypto.createHash(algorithm);
    const stream = fs.createReadStream(filePath);
    stream.on('data', chunk => hash.update(chunk));
    stream.on('end', () => resolve(hash.digest('hex')));
    stream.on('error', reject);
  });
}

// ═══════════════════════════════════════════
// HMAC - Hash-based Message Authentication Code
// ═══════════════════════════════════════════
console.log('\n--- HMAC ---');

const secret = 'my-secret-key';
const message = 'Important data';

const hmac = crypto.createHmac('sha256', secret)
  .update(message)
  .digest('hex');

console.log('HMAC:', hmac);

// Verify HMAC
function verifyHMAC(message, key, providedHmac) {
  const expectedHmac = crypto.createHmac('sha256', key)
    .update(message)
    .digest('hex');
  
  // ใช้ timingSafeEqual ป้องกัน timing attacks
  const expectedBuffer = Buffer.from(expectedHmac);
  const providedBuffer = Buffer.from(providedHmac);
  
  if (expectedBuffer.length !== providedBuffer.length) return false;
  return crypto.timingSafeEqual(expectedBuffer, providedBuffer);
}

console.log('HMAC valid:', verifyHMAC(message, secret, hmac));  // true
console.log('HMAC tampered:', verifyHMAC(message + '!', secret, hmac));  // false

// ═══════════════════════════════════════════
// Random Data Generation
// ═══════════════════════════════════════════
console.log('\n--- Random Data ---');

// สร้าง random bytes
const randomBytes = crypto.randomBytes(16);
console.log('Random bytes (hex):', randomBytes.toString('hex'));
console.log('Random bytes (base64):', randomBytes.toString('base64'));

// สร้าง random UUID v4
const uuid = crypto.randomUUID();
console.log('UUID v4:', uuid);

// สร้าง random token (สำหรับ password reset, API keys)
function generateToken(bytes = 32) {
  return crypto.randomBytes(bytes).toString('hex');
}

function generateApiKey() {
  const prefix = 'sk_live_';
  const token = crypto.randomBytes(24).toString('base64url');
  return prefix + token;
}

console.log('Random token:', generateToken(32));
console.log('API Key:', generateApiKey());

// ═══════════════════════════════════════════
// Password Hashing (ควรใช้ bcrypt ในโปรดักชัน)
// ═══════════════════════════════════════════
console.log('\n--- Password Hashing with scrypt ---');

// สร้าง hash password
async function hashPassword(password) {
  const salt = crypto.randomBytes(16).toString('hex');
  
  return new Promise((resolve, reject) => {
    crypto.scrypt(password, salt, 64, (err, hash) => {
      if (err) reject(err);
      else resolve(`${salt}:${hash.toString('hex')}`);
    });
  });
}

// ตรวจสอบ password
async function verifyPassword(password, storedHash) {
  const [salt, hash] = storedHash.split(':');
  
  return new Promise((resolve, reject) => {
    crypto.scrypt(password, salt, 64, (err, derivedKey) => {
      if (err) reject(err);
      else {
        const hashBuffer = Buffer.from(hash, 'hex');
        resolve(crypto.timingSafeEqual(derivedKey, hashBuffer));
      }
    });
  });
}

async function passwordDemo() {
  const password = 'MySecurePassword123!';
  
  console.log('Hashing password...');
  const stored = await hashPassword(password);
  console.log('Stored hash:', stored.substring(0, 40) + '...');
  
  const valid = await verifyPassword(password, stored);
  const invalid = await verifyPassword('wrongpassword', stored);
  
  console.log('Correct password:', valid);    // true
  console.log('Wrong password:', invalid);    // false
}

passwordDemo();

// ═══════════════════════════════════════════
// Symmetric Encryption (AES-256-GCM)
// ═══════════════════════════════════════════
console.log('\n--- AES-256-GCM Encryption ---');

class AESCipher {
  constructor(key) {
    // key ต้องเป็น 32 bytes สำหรับ AES-256
    this.key = crypto.scryptSync(key, 'salt', 32);
  }
  
  encrypt(text) {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv('aes-256-gcm', this.key, iv);
    
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag().toString('hex');
    
    return `${iv.toString('hex')}:${authTag}:${encrypted}`;
  }
  
  decrypt(encryptedData) {
    const [ivHex, authTagHex, encrypted] = encryptedData.split(':');
    
    const iv = Buffer.from(ivHex, 'hex');
    const authTag = Buffer.from(authTagHex, 'hex');
    
    const decipher = crypto.createDecipheriv('aes-256-gcm', this.key, iv);
    decipher.setAuthTag(authTag);
    
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}

const cipher = new AESCipher('my-secret-encryption-key');
const plaintext = 'Sensitive data: credit card 1234-5678-9012-3456';

const encrypted = cipher.encrypt(plaintext);
const decrypted = cipher.decrypt(encrypted);

console.log('Original:', plaintext);
console.log('Encrypted:', encrypted.substring(0, 40) + '...');
console.log('Decrypted:', decrypted);
console.log('Match:', plaintext === decrypted);
```

---

## ขั้นตอนที่ 26: zlib module

```javascript
// zlib-demo.js
const zlib = require('zlib');
const fs = require('fs');
const { promisify } = require('util');

const gzip = promisify(zlib.gzip);
const gunzip = promisify(zlib.gunzip);
const deflate = promisify(zlib.deflate);
const inflate = promisify(zlib.inflate);
const brotliCompress = promisify(zlib.brotliCompress);
const brotliDecompress = promisify(zlib.brotliDecompress);

async function compressionDemo() {
  const text = `
    Lorem ipsum dolor sit amet, consectetur adipiscing elit.
    Sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
    Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris.
    นี่คือข้อมูลที่เราจะ compress เพื่อทดสอบ
    สวัสดี Node.js! การ compress ข้อมูลช่วยลดขนาดได้มาก
  `.repeat(100);  // ซ้ำ 100 รอบ เพื่อให้เห็นความต่าง
  
  const originalSize = Buffer.byteLength(text, 'utf8');
  console.log('Original size:', formatBytes(originalSize));
  
  // ═══════════════════════════════════════════
  // GZIP Compression
  // ═══════════════════════════════════════════
  console.log('\n--- GZIP ---');
  const gzipped = await gzip(text);
  console.log('Compressed size:', formatBytes(gzipped.length));
  console.log('Ratio:', ((1 - gzipped.length / originalSize) * 100).toFixed(1) + '% smaller');
  
  const ungzipped = await gunzip(gzipped);
  console.log('Decompressed match:', ungzipped.toString() === text);
  
  // ═══════════════════════════════════════════
  // Deflate Compression
  // ═══════════════════════════════════════════
  console.log('\n--- Deflate ---');
  const deflated = await deflate(text);
  console.log('Compressed size:', formatBytes(deflated.length));
  console.log('Ratio:', ((1 - deflated.length / originalSize) * 100).toFixed(1) + '% smaller');
  
  const inflated = await inflate(deflated);
  console.log('Decompressed match:', inflated.toString() === text);
  
  // ═══════════════════════════════════════════
  // Brotli Compression (ดีกว่า GZIP ประมาณ 20-30%)
  // ═══════════════════════════════════════════
  console.log('\n--- Brotli ---');
  const brotli = await brotliCompress(text, {
    params: {
      [zlib.constants.BROTLI_PARAM_QUALITY]: 11  // สูงสุด 11
    }
  });
  console.log('Compressed size:', formatBytes(brotli.length));
  console.log('Ratio:', ((1 - brotli.length / originalSize) * 100).toFixed(1) + '% smaller');
  
  const unbrotli = await brotliDecompress(brotli);
  console.log('Decompressed match:', unbrotli.toString() === text);
  
  // ═══════════════════════════════════════════
  // Compress ไฟล์ด้วย Stream
  // ═══════════════════════════════════════════
  console.log('\n--- File Compression with Streams ---');
  
  // สร้างไฟล์ทดสอบ
  const testFile = '/tmp/test-data.txt';
  fs.writeFileSync(testFile, text);
  
  await new Promise((resolve, reject) => {
    const readStream = fs.createReadStream(testFile);
    const writeStream = fs.createWriteStream(testFile + '.gz');
    const gzipStream = zlib.createGzip({ level: 9 });
    
    readStream
      .pipe(gzipStream)
      .pipe(writeStream)
      .on('finish', () => {
        const original = fs.statSync(testFile).size;
        const compressed = fs.statSync(testFile + '.gz').size;
        console.log(`File compressed: ${formatBytes(original)} → ${formatBytes(compressed)}`);
        resolve();
      })
      .on('error', reject);
  });
  
  // ═══════════════════════════════════════════
  // ใช้กับ HTTP Response (Express middleware)
  // ═══════════════════════════════════════════
  console.log('\n--- HTTP Compression Example ---');
  console.log(`
// ใน Express.js:
const compression = require('compression');  // npm install compression
const express = require('express');
const app = express();

// Enable GZIP/Brotli compression
app.use(compression({
  filter: (req, res) => {
    if (req.headers['x-no-compression']) return false;
    return compression.filter(req, res);
  },
  level: 6,  // 0-9, ค่า default
  threshold: 1024  // compress เฉพาะ response > 1KB
}));
  `);
}

function formatBytes(bytes) {
  if (bytes < 1024) return bytes + ' B';
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB';
  return (bytes / 1024 / 1024).toFixed(1) + ' MB';
}

compressionDemo();
```

---

## ขั้นตอนที่ 27: child_process module

```javascript
// child-process-demo.js
const { exec, execSync, spawn, spawnSync, fork } = require('child_process');
const { promisify } = require('util');

const execAsync = promisify(exec);

// ═══════════════════════════════════════════
// exec - รัน shell command (simple)
// ═══════════════════════════════════════════
console.log('--- exec ---');

// Async version
exec('ls -la', (error, stdout, stderr) => {
  if (error) {
    console.error('Error:', error.message);
    return;
  }
  if (stderr) {
    console.error('Stderr:', stderr);
  }
  console.log('Directory listing:\n', stdout);
});

// Promise version
async function runCommand() {
  try {
    const { stdout, stderr } = await execAsync('node --version');
    console.log('Node version:', stdout.trim());
    
    // ได้รับ stdout และ stderr แยกกัน
    const { stdout: npmVersion } = await execAsync('npm --version');
    console.log('npm version:', npmVersion.trim());
    
  } catch (error) {
    console.error('Command failed:', error.message);
    console.error('Exit code:', error.code);
  }
}

runCommand();

// ═══════════════════════════════════════════
// execSync - Synchronous (blocking)
// ═══════════════════════════════════════════
console.log('\n--- execSync ---');

try {
  const output = execSync('echo "Hello from shell"', { encoding: 'utf8' });
  console.log('Output:', output.trim());
  
  const date = execSync('date', { encoding: 'utf8' });
  console.log('Current date:', date.trim());
  
} catch (error) {
  console.error('Error:', error.message);
}

// ═══════════════════════════════════════════
// spawn - รัน process (สำหรับ long-running tasks)
// ═══════════════════════════════════════════
console.log('\n--- spawn ---');

function runScript(scriptPath, args = []) {
  return new Promise((resolve, reject) => {
    const child = spawn('node', [scriptPath, ...args], {
      stdio: ['pipe', 'pipe', 'pipe']
    });
    
    let stdout = '';
    let stderr = '';
    
    child.stdout.on('data', (data) => {
      stdout += data;
      process.stdout.write(data);  // แสดงแบบ real-time
    });
    
    child.stderr.on('data', (data) => {
      stderr += data;
      process.stderr.write(data);
    });
    
    child.on('close', (code) => {
      if (code === 0) {
        resolve({ stdout, stderr });
      } else {
        reject(new Error(`Process exited with code ${code}\n${stderr}`));
      }
    });
    
    child.on('error', reject);
  });
}

// ═══════════════════════════════════════════
// fork - สำหรับ Node.js processes
// ═══════════════════════════════════════════
console.log('\n--- fork (IPC between processes) ---');

// parent.js
// ส่ง message ไป child process

/*
// worker.js (child process)
process.on('message', (msg) => {
  console.log('Worker received:', msg);
  
  if (msg.type === 'compute') {
    // ทำงานหนัก
    const result = heavyComputation(msg.data);
    process.send({ type: 'result', data: result });
  }
  
  if (msg.type === 'exit') {
    process.exit(0);
  }
});

function heavyComputation(n) {
  let result = 0;
  for (let i = 0; i < n; i++) {
    result += Math.sqrt(i);
  }
  return result;
}
*/

// parent.js
/*
const { fork } = require('child_process');
const worker = fork('./worker.js');

worker.on('message', (msg) => {
  console.log('Parent received:', msg);
  if (msg.type === 'result') {
    console.log('Computation result:', msg.data);
    worker.send({ type: 'exit' });
  }
});

worker.send({ type: 'compute', data: 1000000 });
*/

// ═══════════════════════════════════════════
// ตัวอย่างจริง: รัน ffmpeg สำหรับ video processing
// ═══════════════════════════════════════════

async function getVideoInfo(inputFile) {
  return new Promise((resolve, reject) => {
    const ffprobe = spawn('ffprobe', [
      '-v', 'quiet',
      '-print_format', 'json',
      '-show_format',
      '-show_streams',
      inputFile
    ]);
    
    let output = '';
    ffprobe.stdout.on('data', data => { output += data; });
    
    ffprobe.on('close', (code) => {
      if (code === 0) {
        resolve(JSON.parse(output));
      } else {
        reject(new Error(`ffprobe failed with code ${code}`));
      }
    });
  });
}

// ═══════════════════════════════════════════
// ตัวอย่างจริง: Git operations
// ═══════════════════════════════════════════

async function gitStatus(repoPath) {
  const { stdout } = await execAsync('git status --porcelain', { cwd: repoPath });
  const files = stdout.trim().split('\n').filter(Boolean);
  
  return files.map(line => ({
    status: line.substring(0, 2).trim(),
    file: line.substring(3)
  }));
}

async function gitLog(repoPath, limit = 10) {
  const format = '%H|%an|%ae|%ai|%s';
  const { stdout } = await execAsync(
    `git log --oneline -${limit} --format="${format}"`,
    { cwd: repoPath }
  );
  
  return stdout.trim().split('\n').filter(Boolean).map(line => {
    const [hash, author, email, date, ...subject] = line.split('|');
    return { hash, author, email, date, subject: subject.join('|') };
  });
}
```

---

## ขั้นตอนที่ 28: worker_threads module

```javascript
// worker-threads-demo.js
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

// Worker Threads ใช้สำหรับ CPU-intensive tasks
// เพื่อไม่ให้บล็อก Event Loop

if (isMainThread) {
  // ═══════════════════════════════════════════
  // Main Thread
  // ═══════════════════════════════════════════
  console.log('Main thread running...');
  
  // สร้าง Worker Pool
  class WorkerPool {
    constructor(numWorkers, workerFile) {
      this.workers = [];
      this.queue = [];
      this.activeWorkers = 0;
      
      for (let i = 0; i < numWorkers; i++) {
        this.addWorker(workerFile);
      }
    }
    
    addWorker(workerFile) {
      const worker = new Worker(workerFile);
      
      worker.on('message', (result) => {
        this.activeWorkers--;
        
        if (result.error) {
          result.reject(new Error(result.error));
        } else {
          result.resolve(result.data);
        }
        
        // รันงานถัดไปใน queue
        if (this.queue.length > 0) {
          const { data, resolve, reject } = this.queue.shift();
          this.runWorker(worker, data, resolve, reject);
        }
      });
      
      this.workers.push(worker);
    }
    
    run(data) {
      return new Promise((resolve, reject) => {
        const availableWorker = this.workers.find(
          w => !w._activeTask
        );
        
        if (availableWorker) {
          this.runWorker(availableWorker, data, resolve, reject);
        } else {
          this.queue.push({ data, resolve, reject });
        }
      });
    }
    
    runWorker(worker, data, resolve, reject) {
      worker._activeTask = true;
      this.activeWorkers++;
      
      // ส่งงานไปให้ worker
      worker.postMessage({ ...data, resolve: null, reject: null });
      
      // เก็บ callbacks
      worker._resolve = resolve;
      worker._reject = reject;
    }
    
    terminate() {
      this.workers.forEach(w => w.terminate());
    }
  }
  
  // ตัวอย่าง: คำนวณ prime numbers แบบ parallel
  async function findPrimes(start, end, numWorkers = 4) {
    console.log(`Finding primes from ${start} to ${end} using ${numWorkers} workers...`);
    const startTime = Date.now();
    
    // แบ่งงานออกเป็น chunks
    const chunkSize = Math.ceil((end - start) / numWorkers);
    const chunks = [];
    
    for (let i = 0; i < numWorkers; i++) {
      const chunkStart = start + i * chunkSize;
      const chunkEnd = Math.min(start + (i + 1) * chunkSize - 1, end);
      chunks.push({ start: chunkStart, end: chunkEnd });
    }
    
    // รัน workers แบบ parallel
    const results = await Promise.all(
      chunks.map(chunk => {
        return new Promise((resolve, reject) => {
          const worker = new Worker(__filename, {
            workerData: { task: 'findPrimes', ...chunk }
          });
          
          worker.on('message', resolve);
          worker.on('error', reject);
          worker.on('exit', (code) => {
            if (code !== 0) reject(new Error(`Worker stopped with exit code ${code}`));
          });
        });
      })
    );
    
    const primes = results.flat().sort((a, b) => a - b);
    const elapsed = Date.now() - startTime;
    
    console.log(`Found ${primes.length} primes in ${elapsed}ms`);
    return primes;
  }
  
  findPrimes(1, 100000, 4).then(primes => {
    console.log('First 10 primes:', primes.slice(0, 10));
    console.log('Last 10 primes:', primes.slice(-10));
  });
  
} else {
  // ═══════════════════════════════════════════
  // Worker Thread
  // ═══════════════════════════════════════════
  const { task, start, end } = workerData;
  
  if (task === 'findPrimes') {
    const primes = [];
    
    for (let n = start; n <= end; n++) {
      if (isPrime(n)) primes.push(n);
    }
    
    parentPort.postMessage(primes);
  }
  
  function isPrime(n) {
    if (n < 2) return false;
    if (n === 2) return true;
    if (n % 2 === 0) return false;
    
    for (let i = 3; i <= Math.sqrt(n); i += 2) {
      if (n % i === 0) return false;
    }
    return true;
  }
}
```

---

## ขั้นตอนที่ 29: timer functions

```javascript
// timers-demo.js

// ═══════════════════════════════════════════
// setTimeout - รันครั้งเดียวหลัง delay
// ═══════════════════════════════════════════
console.log('--- setTimeout ---');

const timer1 = setTimeout(() => {
  console.log('Executed after 1 second');
}, 1000);

// ยกเลิก timeout
clearTimeout(timer1);  // ยกเลิกแล้ว ไม่รัน

// ส่ง arguments
setTimeout((name, greeting) => {
  console.log(`${greeting}, ${name}!`);
}, 500, 'World', 'Hello');

// ═══════════════════════════════════════════
// setInterval - รันซ้ำทุก interval
// ═══════════════════════════════════════════
console.log('\n--- setInterval ---');

let count = 0;
const timer2 = setInterval(() => {
  count++;
  process.stdout.write(`\rCount: ${count}`);
  
  if (count >= 5) {
    clearInterval(timer2);
    console.log('\nInterval stopped');
  }
}, 200);

// ═══════════════════════════════════════════
// setImmediate - รันหลัง I/O events ใน current iteration
// ═══════════════════════════════════════════
console.log('\n--- setImmediate ---');

setImmediate(() => {
  console.log('setImmediate: runs after I/O events');
});

process.nextTick(() => {
  console.log('nextTick: runs before setImmediate');
});

// ═══════════════════════════════════════════
// Practical Timers: Countdown
// ═══════════════════════════════════════════

function countdown(seconds, onTick, onComplete) {
  let remaining = seconds;
  
  onTick(remaining);  // แสดงเริ่มต้น
  
  const timer = setInterval(() => {
    remaining--;
    onTick(remaining);
    
    if (remaining === 0) {
      clearInterval(timer);
      onComplete();
    }
  }, 1000);
  
  return timer;  // คืน timer เพื่อให้ยกเลิกได้
}

// ═══════════════════════════════════════════
// Practical Timers: Retry with backoff
// ═══════════════════════════════════════════

async function retryWithBackoff(fn, maxRetries = 3, baseDelay = 1000) {
  let lastError;
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      
      if (attempt < maxRetries) {
        const delay = baseDelay * Math.pow(2, attempt);  // exponential backoff
        const jitter = Math.random() * delay * 0.1;     // เพิ่ม jitter
        const waitTime = delay + jitter;
        
        console.log(`Attempt ${attempt + 1} failed. Retrying in ${waitTime.toFixed(0)}ms...`);
        await new Promise(resolve => setTimeout(resolve, waitTime));
      }
    }
  }
  
  throw new Error(`Failed after ${maxRetries} retries: ${lastError.message}`);
}

// ตัวอย่างใช้งาน
async function unreliableOperation() {
  if (Math.random() < 0.7) {  // 70% chance of failure
    throw new Error('Temporary network error');
  }
  return 'Success!';
}

retryWithBackoff(unreliableOperation, 3, 500)
  .then(result => console.log('Result:', result))
  .catch(err => console.error('All retries failed:', err.message));

// ═══════════════════════════════════════════
// Practical Timers: Debounce & Throttle
// ═══════════════════════════════════════════

// Debounce: รอให้หยุดเรียกก่อน แล้วค่อยรัน
function debounce(fn, delay) {
  let timer;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Throttle: รันไม่เกิน X ครั้งต่อ interval
function throttle(fn, limit) {
  let inThrottle;
  return function(...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => { inThrottle = false; }, limit);
    }
  };
}

// ตัวอย่าง: search as you type
const handleSearch = debounce((query) => {
  console.log('Searching for:', query);
  // call API
}, 300);

// จะ call API ครั้งเดียวหลังพิมพ์เสร็จ 300ms
handleSearch('N');
handleSearch('No');
handleSearch('Nod');
handleSearch('Node');
handleSearch('Node.');
// จะ search "Node." เพียงครั้งเดียว
```

---

## ขั้นตอนที่ 30: สรุป Part 02 และ Exercise

### สิ่งที่เรียนรู้ใน Part 02

```
✅ path module - จัดการ file paths อย่างถูกต้อง
✅ os module - ข้อมูล OS, CPU, Memory
✅ url module - parse และสร้าง URLs
✅ util module - promisify, inspect, format
✅ crypto module - hash, HMAC, encryption, random
✅ zlib module - compression/decompression
✅ child_process - exec, spawn, fork
✅ worker_threads - parallel CPU tasks
✅ Timer functions - setTimeout, setInterval, debounce, throttle
```

### 📝 Exercise

**Exercise 1: System Info Tool**
```
สร้าง system-info.js ที่แสดง:
1. ข้อมูล OS ทั้งหมด
2. CPU usage แต่ละ core
3. Memory usage พร้อม progress bar
4. Network interfaces ทั้งหมด
5. Disk space (ใช้ child_process รัน df)
Format output ให้สวยงาม
```

**Exercise 2: File Hash Checker**
```
สร้าง hash-checker.js ที่:
1. รับ path ของไฟล์หรือ directory จาก argument
2. คำนวณ hash ของไฟล์ทุกไฟล์
3. บันทึก hash ลงใน checksums.json
4. ตรวจสอบว่าไฟล์เปลี่ยนแปลงหรือไม่
รัน: node hash-checker.js ./src --check
```

**Exercise 3: URL Analyzer**
```
สร้าง url-analyzer.js ที่:
1. รับ URL จาก argument
2. แสดงทุก component
3. Decode query string parameters
4. ตรวจสอบว่า URL valid หรือไม่
5. สร้าง modified URL ตาม flags ที่ส่งมา
```

**Exercise 4: Mini Build Tool**
```
สร้าง build.js ที่:
1. อ่านไฟล์ .js ทุกไฟล์ใน src/
2. Concatenate รวมกัน
3. Minify (ลบ whitespace ที่ไม่จำเป็น)
4. GZIP compress
5. เขียนไปที่ dist/bundle.js.gz
6. แสดง file size ก่อนและหลัง
```

---

## 🔜 Part ถัดไป

**Part 03: npm และการจัดการ Package** จะครอบคลุม:
- npm basics และ commands สำคัญ
- semantic versioning
- package-lock.json
- npm scripts advanced
- สร้าง npm package เอง
- npx usage
- private packages

---

*Part 02 สมบูรณ์ | ขั้นตอนที่ 21-50 จาก 1000*
