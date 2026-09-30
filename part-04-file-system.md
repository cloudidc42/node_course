# Part 04: File System (fs module)
## ขั้นตอนที่ 91-140: จัดการไฟล์และ Directory อย่างมืออาชีพ

---

## 🎯 เป้าหมายของ Part นี้

1. อ่าน/เขียนไฟล์ทั้งแบบ Sync และ Async ได้
2. จัดการ Directory ได้
3. ใช้ Streams สำหรับไฟล์ขนาดใหญ่ได้
4. Watch ไฟล์และ Directory ได้
5. ใช้ fs/promises (modern API) ได้
6. สร้าง file-based storage ได้

---

## ขั้นตอนที่ 91: อ่านไฟล์

```javascript
// read-files.js
const fs = require('fs');
const { promises: fsPromises } = require('fs');
const path = require('path');

// ═══════════════════════════════════════════
// Synchronous Reading (blocking)
// ═══════════════════════════════════════════

// อ่านทั้งไฟล์
const content = fs.readFileSync('./example.txt', 'utf8');
console.log('Sync content:', content);

// อ่านเป็น Buffer
const buffer = fs.readFileSync('./image.png');
console.log('Buffer size:', buffer.length, 'bytes');

// อ่านพร้อม options
const data = fs.readFileSync('./data.json', {
  encoding: 'utf8',
  flag: 'r'  // r = read, r+ = read+write
});

// ═══════════════════════════════════════════
// Asynchronous Reading (non-blocking) - Callback
// ═══════════════════════════════════════════

fs.readFile('./example.txt', 'utf8', (err, data) => {
  if (err) {
    if (err.code === 'ENOENT') {
      console.error('File not found');
    } else {
      console.error('Error reading file:', err.message);
    }
    return;
  }
  console.log('Async content:', data);
});

// ═══════════════════════════════════════════
// Modern: fs.promises (แนะนำ)
// ═══════════════════════════════════════════

// วิธีที่ 1: fs.promises
async function readWithPromises() {
  try {
    const content = await fsPromises.readFile('./example.txt', 'utf8');
    console.log('Promise content:', content);
  } catch (err) {
    if (err.code === 'ENOENT') {
      console.error('File not found:', err.path);
    } else {
      throw err;
    }
  }
}

// วิธีที่ 2: require('fs/promises') - Node.js 14+
const fs2 = require('fs/promises');
async function readModern() {
  const content = await fs2.readFile('./example.txt', 'utf8');
  return content;
}

// ═══════════════════════════════════════════
// อ่าน JSON File
// ═══════════════════════════════════════════

async function readJSON(filePath) {
  const content = await fsPromises.readFile(filePath, 'utf8');
  return JSON.parse(content);
}

// ═══════════════════════════════════════════
// อ่านทีละบรรทัด (สำหรับไฟล์ใหญ่)
// ═══════════════════════════════════════════
const readline = require('readline');

async function readLineByLine(filePath) {
  const fileStream = fs.createReadStream(filePath);
  const rl = readline.createInterface({
    input: fileStream,
    crlfDelay: Infinity  // รองรับทั้ง \n และ \r\n
  });
  
  let lineNumber = 0;
  for await (const line of rl) {
    lineNumber++;
    console.log(`Line ${lineNumber}: ${line}`);
    
    // หยุดหลัง 100 บรรทัด
    if (lineNumber >= 100) break;
  }
  
  return lineNumber;
}

// ═══════════════════════════════════════════
// อ่านส่วนหนึ่งของไฟล์ (offset + length)
// ═══════════════════════════════════════════

async function readPartial(filePath, start, length) {
  const fd = await fsPromises.open(filePath, 'r');
  try {
    const buffer = Buffer.alloc(length);
    const { bytesRead } = await fd.read(buffer, 0, length, start);
    return buffer.slice(0, bytesRead).toString('utf8');
  } finally {
    await fd.close();
  }
}
```

---

## ขั้นตอนที่ 92: เขียนไฟล์

```javascript
// write-files.js
const fs = require('fs');
const fsPromises = require('fs/promises');
const path = require('path');

// ═══════════════════════════════════════════
// Synchronous Writing
// ═══════════════════════════════════════════

// เขียนใหม่ทับ (overwrite)
fs.writeFileSync('./output.txt', 'Hello, World!\n', 'utf8');

// เขียน Buffer
fs.writeFileSync('./output.bin', Buffer.from([0x48, 0x65, 0x6C, 0x6C, 0x6F]));

// เพิ่มต่อท้าย (append)
fs.appendFileSync('./log.txt', `${new Date().toISOString()} - Server started\n`);

// ═══════════════════════════════════════════
// Asynchronous Writing
// ═══════════════════════════════════════════

// writeFile - เขียนทับ
async function writeFile(filePath, content) {
  await fsPromises.writeFile(filePath, content, {
    encoding: 'utf8',
    flag: 'w',   // w = write (overwrite), a = append, wx = fail if exists
    mode: 0o644  // file permissions (Unix)
  });
}

// appendFile - เพิ่มต่อท้าย
async function appendToFile(filePath, content) {
  await fsPromises.appendFile(filePath, content + '\n', 'utf8');
}

// ═══════════════════════════════════════════
// เขียน JSON
// ═══════════════════════════════════════════

async function writeJSON(filePath, data, pretty = true) {
  const content = pretty 
    ? JSON.stringify(data, null, 2) 
    : JSON.stringify(data);
  await fsPromises.writeFile(filePath, content, 'utf8');
}

// ═══════════════════════════════════════════
// เขียนแบบ Atomic (ป้องกันข้อมูลเสียหาย)
// ═══════════════════════════════════════════

async function writeAtomic(filePath, content) {
  // เขียนไปไฟล์ temp ก่อน แล้วค่อย rename
  // ถ้า process crash ระหว่างเขียน ไฟล์ต้นฉบับยังปลอดภัย
  const tmpFile = filePath + '.tmp.' + process.pid;
  
  try {
    await fsPromises.writeFile(tmpFile, content, 'utf8');
    await fsPromises.rename(tmpFile, filePath);
  } catch (err) {
    // ล้าง temp file ถ้าเกิด error
    try { await fsPromises.unlink(tmpFile); } catch {}
    throw err;
  }
}

// ═══════════════════════════════════════════
// เขียนทีละ chunk (สำหรับข้อมูลขนาดใหญ่)
// ═══════════════════════════════════════════

async function writeChunked(filePath, dataGenerator) {
  const writeStream = fs.createWriteStream(filePath, { encoding: 'utf8' });
  
  return new Promise((resolve, reject) => {
    writeStream.on('finish', resolve);
    writeStream.on('error', reject);
    
    for (const chunk of dataGenerator()) {
      writeStream.write(chunk);
    }
    
    writeStream.end();
  });
}

// ═══════════════════════════════════════════
// Logger - เขียน log ไฟล์
// ═══════════════════════════════════════════

class FileLogger {
  constructor(logFile, options = {}) {
    this.logFile = logFile;
    this.maxSize = options.maxSize || 10 * 1024 * 1024; // 10MB default
    this.maxFiles = options.maxFiles || 5;
    
    // Ensure directory exists
    fs.mkdirSync(path.dirname(logFile), { recursive: true });
  }
  
  async log(level, message, meta = {}) {
    const entry = JSON.stringify({
      timestamp: new Date().toISOString(),
      level,
      message,
      ...meta
    }) + '\n';
    
    await this.rotateIfNeeded();
    await fsPromises.appendFile(this.logFile, entry, 'utf8');
  }
  
  async rotateIfNeeded() {
    try {
      const stat = await fsPromises.stat(this.logFile);
      if (stat.size >= this.maxSize) {
        await this.rotate();
      }
    } catch (err) {
      if (err.code !== 'ENOENT') throw err;
    }
  }
  
  async rotate() {
    // ลบไฟล์เก่าสุด
    const oldest = `${this.logFile}.${this.maxFiles}`;
    try { await fsPromises.unlink(oldest); } catch {}
    
    // เลื่อน files
    for (let i = this.maxFiles - 1; i >= 1; i--) {
      const from = i === 1 ? this.logFile : `${this.logFile}.${i}`;
      const to = `${this.logFile}.${i + 1}`;
      try { await fsPromises.rename(from, to); } catch {}
    }
  }
  
  info(message, meta) { return this.log('INFO', message, meta); }
  warn(message, meta) { return this.log('WARN', message, meta); }
  error(message, meta) { return this.log('ERROR', message, meta); }
}

// ใช้งาน
const logger = new FileLogger('./logs/app.log', {
  maxSize: 5 * 1024 * 1024,  // 5MB
  maxFiles: 10
});

async function demoLogger() {
  await logger.info('Server started', { port: 3000 });
  await logger.warn('High memory usage', { usage: '85%' });
  await logger.error('Database connection failed', { 
    host: 'localhost', 
    port: 5432 
  });
}
```

---

## ขั้นตอนที่ 93: Directory Operations

```javascript
// directory-ops.js
const fs = require('fs');
const fsPromises = require('fs/promises');
const path = require('path');

// ═══════════════════════════════════════════
// สร้าง Directory
// ═══════════════════════════════════════════

// สร้างเดียว
fs.mkdirSync('./new-dir');

// สร้าง nested directories
fs.mkdirSync('./a/b/c/d', { recursive: true });

// Async
await fsPromises.mkdir('./new-dir', { recursive: true });

// ═══════════════════════════════════════════
// อ่าน Directory
// ═══════════════════════════════════════════

// อ่านแบบ simple (ชื่อไฟล์เท่านั้น)
const entries = fs.readdirSync('./');
console.log('Entries:', entries);

// อ่านพร้อม Dirent (ข้อมูลเพิ่มเติม)
const detailedEntries = await fsPromises.readdir('./', { withFileTypes: true });
detailedEntries.forEach(entry => {
  console.log(
    entry.isDirectory() ? '📁' : '📄',
    entry.name
  );
});

// ═══════════════════════════════════════════
// อ่าน Directory แบบ Recursive
// ═══════════════════════════════════════════

async function readDirRecursive(dirPath, options = {}) {
  const {
    include = null,    // regex หรือ extension list
    exclude = null,    // regex หรือ folder name list
    maxDepth = Infinity
  } = options;
  
  const results = [];
  
  async function traverse(currentPath, depth = 0) {
    if (depth > maxDepth) return;
    
    const entries = await fsPromises.readdir(currentPath, { withFileTypes: true });
    
    for (const entry of entries) {
      const fullPath = path.join(currentPath, entry.name);
      const relativePath = path.relative(dirPath, fullPath);
      
      // ตรวจสอบ exclude
      if (exclude) {
        if (Array.isArray(exclude) && exclude.includes(entry.name)) continue;
        if (exclude instanceof RegExp && exclude.test(entry.name)) continue;
      }
      
      if (entry.isDirectory()) {
        if (exclude !== 'node_modules' || entry.name !== 'node_modules') {
          await traverse(fullPath, depth + 1);
        }
      } else {
        // ตรวจสอบ include
        if (include) {
          if (Array.isArray(include) && !include.some(ext => entry.name.endsWith(ext))) continue;
          if (include instanceof RegExp && !include.test(entry.name)) continue;
        }
        
        const stat = await fsPromises.stat(fullPath);
        results.push({
          name: entry.name,
          path: fullPath,
          relativePath,
          size: stat.size,
          modified: stat.mtime,
          extension: path.extname(entry.name)
        });
      }
    }
  }
  
  await traverse(dirPath);
  return results;
}

// ตัวอย่างใช้งาน
async function findJSFiles() {
  const files = await readDirRecursive('./src', {
    include: ['.js', '.ts'],
    exclude: ['node_modules', '.git']
  });
  
  console.log(`Found ${files.length} JS/TS files:`);
  files.forEach(f => {
    console.log(`  ${f.relativePath} (${formatBytes(f.size)})`);
  });
}

// ═══════════════════════════════════════════
// คัดลอก Directory
// ═══════════════════════════════════════════

async function copyDir(src, dest) {
  await fsPromises.mkdir(dest, { recursive: true });
  
  const entries = await fsPromises.readdir(src, { withFileTypes: true });
  
  for (const entry of entries) {
    const srcPath = path.join(src, entry.name);
    const destPath = path.join(dest, entry.name);
    
    if (entry.isDirectory()) {
      await copyDir(srcPath, destPath);
    } else {
      await fsPromises.copyFile(srcPath, destPath);
    }
  }
}

// Node.js 16.7+ มี cp command
// await fsPromises.cp('./src', './dest', { recursive: true });

// ═══════════════════════════════════════════
// ลบ Directory
// ═══════════════════════════════════════════

// ลบ directory ว่าง
await fsPromises.rmdir('./empty-dir');

// ลบ directory พร้อมเนื้อหา (Node.js 14+)
await fsPromises.rm('./dir-with-files', { recursive: true, force: true });

// ═══════════════════════════════════════════
// ตรวจสอบการมีอยู่ของไฟล์/Directory
// ═══════════════════════════════════════════

// วิธีที่ 1: try/catch (แนะนำ)
async function exists(filePath) {
  try {
    await fsPromises.access(filePath, fs.constants.F_OK);
    return true;
  } catch {
    return false;
  }
}

// วิธีที่ 2: stat (ได้ข้อมูลเพิ่มเติม)
async function getStats(filePath) {
  try {
    const stat = await fsPromises.stat(filePath);
    return {
      exists: true,
      isFile: stat.isFile(),
      isDirectory: stat.isDirectory(),
      size: stat.size,
      created: stat.birthtime,
      modified: stat.mtime,
      mode: stat.mode
    };
  } catch (err) {
    if (err.code === 'ENOENT') return { exists: false };
    throw err;
  }
}

// ตรวจสอบ permissions
async function checkPermissions(filePath) {
  const checks = {
    readable: fs.constants.R_OK,
    writable: fs.constants.W_OK,
    executable: fs.constants.X_OK
  };
  
  const result = {};
  for (const [name, mode] of Object.entries(checks)) {
    try {
      await fsPromises.access(filePath, mode);
      result[name] = true;
    } catch {
      result[name] = false;
    }
  }
  
  return result;
}

function formatBytes(bytes) {
  if (bytes < 1024) return bytes + ' B';
  if (bytes < 1024 ** 2) return (bytes / 1024).toFixed(1) + ' KB';
  if (bytes < 1024 ** 3) return (bytes / 1024 ** 2).toFixed(1) + ' MB';
  return (bytes / 1024 ** 3).toFixed(1) + ' GB';
}
```

---

## ขั้นตอนที่ 94: File Streams

```javascript
// file-streams.js
const fs = require('fs');
const path = require('path');
const { pipeline } = require('stream/promises');
const { Transform, PassThrough } = require('stream');
const zlib = require('zlib');
const crypto = require('crypto');

// ═══════════════════════════════════════════
// ReadStream
// ═══════════════════════════════════════════

function readWithStream(filePath) {
  const readStream = fs.createReadStream(filePath, {
    encoding: 'utf8',
    highWaterMark: 64 * 1024  // chunk size = 64KB
  });
  
  let totalSize = 0;
  let chunkCount = 0;
  
  readStream.on('data', chunk => {
    totalSize += chunk.length;
    chunkCount++;
    process.stdout.write('.');  // แสดง progress
  });
  
  readStream.on('end', () => {
    console.log(`\nRead ${totalSize} bytes in ${chunkCount} chunks`);
  });
  
  readStream.on('error', err => {
    console.error('Read error:', err.message);
  });
  
  return readStream;
}

// ═══════════════════════════════════════════
// WriteStream
// ═══════════════════════════════════════════

function writeWithStream(filePath, content) {
  return new Promise((resolve, reject) => {
    const writeStream = fs.createWriteStream(filePath, {
      flags: 'w',
      encoding: 'utf8'
    });
    
    writeStream.on('finish', resolve);
    writeStream.on('error', reject);
    
    // เขียน content ที่มีขนาดใหญ่ทีละ chunk
    const chunkSize = 64 * 1024;  // 64KB
    let offset = 0;
    
    function writeChunk() {
      while (offset < content.length) {
        const chunk = content.slice(offset, offset + chunkSize);
        offset += chunkSize;
        
        // drain event จะบอกเมื่อ buffer พร้อมรับ data อีก
        if (!writeStream.write(chunk)) {
          writeStream.once('drain', writeChunk);
          return;
        }
      }
      writeStream.end();
    }
    
    writeChunk();
  });
}

// ═══════════════════════════════════════════
// Pipeline - แนะนำสำหรับ production
// ═══════════════════════════════════════════

// คัดลอกไฟล์ด้วย pipeline
async function copyFile(src, dest) {
  await pipeline(
    fs.createReadStream(src),
    fs.createWriteStream(dest)
  );
  console.log(`Copied ${src} → ${dest}`);
}

// คัดลอกพร้อม compress
async function copyAndCompress(src, dest) {
  await pipeline(
    fs.createReadStream(src),
    zlib.createGzip({ level: 9 }),
    fs.createWriteStream(dest)
  );
}

// คัดลอก + progress tracking
async function copyWithProgress(src, dest) {
  const stat = fs.statSync(src);
  const totalSize = stat.size;
  let transferred = 0;
  
  const progress = new Transform({
    transform(chunk, encoding, callback) {
      transferred += chunk.length;
      const percent = (transferred / totalSize * 100).toFixed(1);
      process.stdout.write(`\rProgress: ${percent}% (${formatBytes(transferred)}/${formatBytes(totalSize)})`);
      callback(null, chunk);
    }
  });
  
  await pipeline(
    fs.createReadStream(src),
    progress,
    fs.createWriteStream(dest)
  );
  
  console.log('\nCopy complete!');
}

// คัดลอก + hash verification
async function copyWithHash(src, dest) {
  const hash = crypto.createHash('sha256');
  
  const hashTransform = new Transform({
    transform(chunk, encoding, callback) {
      hash.update(chunk);
      callback(null, chunk);
    },
    flush(callback) {
      this.emit('hash', hash.digest('hex'));
      callback();
    }
  });
  
  hashTransform.on('hash', (h) => {
    console.log('SHA256:', h);
  });
  
  await pipeline(
    fs.createReadStream(src),
    hashTransform,
    fs.createWriteStream(dest)
  );
}

// ═══════════════════════════════════════════
// Process large CSV file
// ═══════════════════════════════════════════

const { createInterface } = require('readline');

async function processCSV(inputFile, outputFile, transform) {
  const readStream = fs.createReadStream(inputFile, 'utf8');
  const writeStream = fs.createWriteStream(outputFile, 'utf8');
  
  const rl = createInterface({
    input: readStream,
    crlfDelay: Infinity
  });
  
  let isFirstLine = true;
  let processedRows = 0;
  
  for await (const line of rl) {
    if (isFirstLine) {
      writeStream.write(line + '\n');  // เขียน header
      isFirstLine = false;
      continue;
    }
    
    const row = line.split(',');
    const transformedRow = transform(row);
    
    if (transformedRow !== null) {
      writeStream.write(transformedRow.join(',') + '\n');
      processedRows++;
    }
  }
  
  await new Promise(resolve => writeStream.end(resolve));
  console.log(`Processed ${processedRows} rows`);
}

// ตัวอย่าง: filter และ transform CSV
await processCSV(
  './data/users.csv',
  './output/active-users.csv',
  (row) => {
    // row = ['id', 'name', 'email', 'status', 'age']
    if (row[3] !== 'active') return null;  // filter inactive
    row[4] = parseInt(row[4]) + 1;          // increment age
    return row;
  }
);

function formatBytes(bytes) {
  if (bytes < 1024) return bytes + ' B';
  if (bytes < 1024 ** 2) return (bytes / 1024).toFixed(1) + ' KB';
  return (bytes / 1024 ** 2).toFixed(1) + ' MB';
}
```

---

## ขั้นตอนที่ 95: File Watching

```javascript
// file-watcher.js
const fs = require('fs');
const fsPromises = require('fs/promises');
const path = require('path');

// ═══════════════════════════════════════════
// fs.watch - Native (เร็ว แต่ API ไม่สวย)
// ═══════════════════════════════════════════

const watcher = fs.watch('./src', { recursive: true }, (eventType, filename) => {
  if (filename) {
    console.log(`Event: ${eventType} | File: ${filename}`);
  }
});

// หยุด watching
watcher.close();

// ═══════════════════════════════════════════
// fs.watchFile - Polling (compatible ทุก OS)
// ═══════════════════════════════════════════

fs.watchFile('./config.json', { interval: 1000 }, (current, previous) => {
  if (current.mtime > previous.mtime) {
    console.log('Config file changed!');
    // reload config...
  }
});

// หยุด watching
fs.unwatchFile('./config.json');

// ═══════════════════════════════════════════
// สร้าง Custom File Watcher
// ═══════════════════════════════════════════

class FileWatcher {
  constructor(options = {}) {
    this.watchers = new Map();
    this.handlers = new Map();
    this.debounceTime = options.debounceTime || 100;
    this.debounceTimers = new Map();
  }
  
  watch(filePath, events, handler) {
    const absPath = path.resolve(filePath);
    
    if (!this.handlers.has(absPath)) {
      this.handlers.set(absPath, new Map());
    }
    
    const eventHandlers = this.handlers.get(absPath);
    
    const eventList = Array.isArray(events) ? events : [events];
    eventList.forEach(event => {
      if (!eventHandlers.has(event)) {
        eventHandlers.set(event, []);
      }
      eventHandlers.get(event).push(handler);
    });
    
    if (!this.watchers.has(absPath)) {
      const watcher = fs.watch(absPath, { recursive: true }, (event, filename) => {
        this.handleEvent(absPath, event, filename);
      });
      
      this.watchers.set(absPath, watcher);
    }
    
    return this;
  }
  
  handleEvent(watchPath, event, filename) {
    const key = `${watchPath}:${filename || ''}`;
    
    // Debounce
    if (this.debounceTimers.has(key)) {
      clearTimeout(this.debounceTimers.get(key));
    }
    
    this.debounceTimers.set(key, setTimeout(async () => {
      this.debounceTimers.delete(key);
      
      const filePath = filename ? path.join(watchPath, filename) : watchPath;
      
      // หา handler ที่ตรงกัน
      const handlers = this.handlers.get(watchPath);
      if (!handlers) return;
      
      const eventHandlers = handlers.get(event) || [];
      const allHandlers = handlers.get('all') || [];
      
      const context = {
        event,
        filename,
        filePath,
        watchPath,
        timestamp: new Date()
      };
      
      for (const handler of [...eventHandlers, ...allHandlers]) {
        try {
          await handler(context);
        } catch (err) {
          console.error('Watcher handler error:', err);
        }
      }
    }, this.debounceTime));
  }
  
  unwatch(filePath) {
    const absPath = path.resolve(filePath);
    const watcher = this.watchers.get(absPath);
    if (watcher) {
      watcher.close();
      this.watchers.delete(absPath);
      this.handlers.delete(absPath);
    }
  }
  
  close() {
    this.watchers.forEach(w => w.close());
    this.watchers.clear();
    this.handlers.clear();
  }
}

// ตัวอย่างใช้งาน
const watcher2 = new FileWatcher({ debounceTime: 200 });

watcher2.watch('./src', 'change', async ({ filename, filePath }) => {
  console.log(`Changed: ${filename}`);
  // auto-compile, hot-reload, etc.
});

watcher2.watch('./config', 'all', async ({ event, filename }) => {
  console.log(`Config ${event}: ${filename}`);
  // reload configuration
});

// ═══════════════════════════════════════════
// Hot Module Reloading (สำหรับ Node.js)
// ═══════════════════════════════════════════

class HotReloader {
  constructor(entryPoint) {
    this.entryPoint = path.resolve(entryPoint);
    this.server = null;
    this.watcher = null;
  }
  
  async start() {
    await this.loadApp();
    this.watch();
  }
  
  async loadApp() {
    // ล้าง require cache สำหรับ module ทั้งหมด
    this.clearCache(this.entryPoint);
    
    try {
      const appModule = require(this.entryPoint);
      
      if (this.server) {
        this.server.close();
      }
      
      this.server = appModule.listen ? appModule.listen(3000) : null;
      console.log('✅ App reloaded');
    } catch (err) {
      console.error('❌ Failed to reload:', err.message);
    }
  }
  
  clearCache(modulePath) {
    const resolvedPath = require.resolve(modulePath);
    
    if (require.cache[resolvedPath]) {
      const mod = require.cache[resolvedPath];
      
      // ล้าง cache ของ dependencies ด้วย
      mod.children.forEach(child => {
        if (!child.id.includes('node_modules')) {
          this.clearCache(child.id);
        }
      });
      
      delete require.cache[resolvedPath];
    }
  }
  
  watch() {
    const srcDir = path.dirname(this.entryPoint);
    
    this.watcher = fs.watch(srcDir, { recursive: true }, (event, filename) => {
      if (filename && /\.[jt]s$/.test(filename)) {
        console.log(`\n📝 Changed: ${filename}`);
        this.loadApp();
      }
    });
    
    console.log('👁️ Watching for changes...');
  }
  
  stop() {
    if (this.watcher) this.watcher.close();
    if (this.server) this.server.close();
  }
}
```

---

## ขั้นตอนที่ 96: File-based Database (JSON Database)

```javascript
// json-db.js - Simple JSON Database

const fs = require('fs');
const fsPromises = require('fs/promises');
const path = require('path');
const crypto = require('crypto');

class JSONDatabase {
  constructor(dbPath, options = {}) {
    this.dbPath = path.resolve(dbPath);
    this.options = {
      autoSave: options.autoSave !== false,
      pretty: options.pretty !== false,
      saveDelay: options.saveDelay || 100
    };
    
    this.data = {};
    this.saveTimer = null;
    this.loaded = false;
    
    // สร้าง directory ถ้าไม่มี
    fs.mkdirSync(path.dirname(this.dbPath), { recursive: true });
  }
  
  async load() {
    try {
      const content = await fsPromises.readFile(this.dbPath, 'utf8');
      this.data = JSON.parse(content);
    } catch (err) {
      if (err.code === 'ENOENT') {
        this.data = {};
      } else {
        throw err;
      }
    }
    this.loaded = true;
    return this;
  }
  
  async save() {
    const content = this.options.pretty
      ? JSON.stringify(this.data, null, 2)
      : JSON.stringify(this.data);
    
    // Atomic write
    const tmpFile = this.dbPath + '.tmp';
    await fsPromises.writeFile(tmpFile, content, 'utf8');
    await fsPromises.rename(tmpFile, this.dbPath);
  }
  
  scheduleSave() {
    if (!this.options.autoSave) return;
    
    clearTimeout(this.saveTimer);
    this.saveTimer = setTimeout(() => this.save(), this.options.saveDelay);
  }
  
  // ═══════════════════════════
  // Collection Operations
  // ═══════════════════════════
  
  collection(name) {
    if (!this.data[name]) {
      this.data[name] = [];
    }
    return new Collection(this, name);
  }
  
  // ═══════════════════════════
  // Key-Value Operations
  // ═══════════════════════════
  
  get(key) { return this.data[key]; }
  
  set(key, value) {
    this.data[key] = value;
    this.scheduleSave();
    return this;
  }
  
  delete(key) {
    delete this.data[key];
    this.scheduleSave();
    return this;
  }
  
  has(key) { return key in this.data; }
  
  keys() { return Object.keys(this.data); }
}

class Collection {
  constructor(db, name) {
    this.db = db;
    this.name = name;
  }
  
  get data() { return this.db.data[this.name]; }
  
  // สร้าง record ใหม่
  insert(doc) {
    const newDoc = {
      _id: crypto.randomUUID(),
      _createdAt: new Date().toISOString(),
      _updatedAt: new Date().toISOString(),
      ...doc
    };
    
    this.data.push(newDoc);
    this.db.scheduleSave();
    return newDoc;
  }
  
  // ค้นหา records
  find(query = {}) {
    if (Object.keys(query).length === 0) {
      return [...this.data];
    }
    
    return this.data.filter(doc => this.matchesQuery(doc, query));
  }
  
  // ค้นหา record เดียว
  findOne(query) {
    return this.data.find(doc => this.matchesQuery(doc, query)) || null;
  }
  
  // ค้นหาด้วย ID
  findById(id) {
    return this.data.find(doc => doc._id === id) || null;
  }
  
  // อัพเดท records
  update(query, update) {
    let count = 0;
    
    this.data.forEach((doc, index) => {
      if (this.matchesQuery(doc, query)) {
        this.data[index] = {
          ...doc,
          ...update,
          _id: doc._id,
          _createdAt: doc._createdAt,
          _updatedAt: new Date().toISOString()
        };
        count++;
      }
    });
    
    if (count > 0) this.db.scheduleSave();
    return count;
  }
  
  // อัพเดทด้วย ID
  updateById(id, update) {
    return this.update({ _id: id }, update);
  }
  
  // ลบ records
  delete(query) {
    const before = this.data.length;
    
    const remaining = this.data.filter(doc => !this.matchesQuery(doc, query));
    this.db.data[this.name] = remaining;
    
    const deleted = before - remaining.length;
    if (deleted > 0) this.db.scheduleSave();
    return deleted;
  }
  
  // ลบด้วย ID
  deleteById(id) {
    return this.delete({ _id: id });
  }
  
  // นับ records
  count(query = {}) {
    return this.find(query).length;
  }
  
  // Sort
  sort(field, direction = 'asc') {
    const sorted = [...this.data].sort((a, b) => {
      if (a[field] < b[field]) return direction === 'asc' ? -1 : 1;
      if (a[field] > b[field]) return direction === 'asc' ? 1 : -1;
      return 0;
    });
    return sorted;
  }
  
  // Pagination
  paginate(query = {}, page = 1, limit = 10) {
    const all = this.find(query);
    const total = all.length;
    const offset = (page - 1) * limit;
    const data = all.slice(offset, offset + limit);
    
    return {
      data,
      pagination: {
        total,
        page,
        limit,
        pages: Math.ceil(total / limit),
        hasNext: page * limit < total,
        hasPrev: page > 1
      }
    };
  }
  
  // ตรวจสอบว่า document ตรงกับ query
  matchesQuery(doc, query) {
    for (const [key, value] of Object.entries(query)) {
      // Support nested operators
      if (typeof value === 'object' && value !== null) {
        for (const [op, opValue] of Object.entries(value)) {
          switch (op) {
            case '$eq': if (doc[key] !== opValue) return false; break;
            case '$ne': if (doc[key] === opValue) return false; break;
            case '$gt': if (doc[key] <= opValue) return false; break;
            case '$gte': if (doc[key] < opValue) return false; break;
            case '$lt': if (doc[key] >= opValue) return false; break;
            case '$lte': if (doc[key] > opValue) return false; break;
            case '$in': if (!opValue.includes(doc[key])) return false; break;
            case '$nin': if (opValue.includes(doc[key])) return false; break;
            case '$regex': if (!new RegExp(opValue).test(doc[key])) return false; break;
          }
        }
      } else {
        if (doc[key] !== value) return false;
      }
    }
    return true;
  }
}

// ═══════════════════════════════════════════
// ตัวอย่างใช้งาน
// ═══════════════════════════════════════════

async function demoDatabase() {
  const db = new JSONDatabase('./data/mydb.json');
  await db.load();
  
  const users = db.collection('users');
  const posts = db.collection('posts');
  
  // Insert users
  const alice = users.insert({
    name: 'Alice',
    email: 'alice@example.com',
    age: 28,
    role: 'admin'
  });
  
  const bob = users.insert({
    name: 'Bob', 
    email: 'bob@example.com',
    age: 32,
    role: 'user'
  });
  
  console.log('Inserted:', alice._id, bob._id);
  
  // Find
  const allUsers = users.find();
  console.log('All users:', allUsers.length);
  
  const admins = users.find({ role: 'admin' });
  console.log('Admins:', admins.length);
  
  const over30 = users.find({ age: { $gte: 30 } });
  console.log('Users 30+:', over30.length);
  
  // Update
  users.updateById(alice._id, { age: 29 });
  
  // Find with pagination
  const page1 = users.paginate({}, 1, 10);
  console.log('Page 1:', page1);
  
  // Insert posts
  posts.insert({
    title: 'Hello World',
    content: 'First post',
    authorId: alice._id,
    tags: ['intro', 'nodejs']
  });
  
  // Save
  await db.save();
  console.log('Database saved!');
}

demoDatabase();
```

---

## ขั้นตอนที่ 97: โปรเจกต์จริง - File Manager CLI

```javascript
// file-manager.js - CLI File Manager

const fs = require('fs');
const fsPromises = require('fs/promises');
const path = require('path');
const readline = require('readline');
const crypto = require('crypto');

class FileManager {
  constructor() {
    this.currentDir = process.cwd();
    this.clipboard = null;
    this.history = [this.currentDir];
  }
  
  // แสดงไฟล์และโฟลเดอร์
  async list(dirPath = this.currentDir) {
    const entries = await fsPromises.readdir(dirPath, { withFileTypes: true });
    
    const items = await Promise.all(
      entries.map(async entry => {
        const fullPath = path.join(dirPath, entry.name);
        const stat = await fsPromises.stat(fullPath).catch(() => null);
        
        return {
          name: entry.name,
          isDir: entry.isDirectory(),
          size: stat?.size || 0,
          modified: stat?.mtime || null
        };
      })
    );
    
    // แสดงโฟลเดอร์ก่อน
    items.sort((a, b) => {
      if (a.isDir !== b.isDir) return a.isDir ? -1 : 1;
      return a.name.localeCompare(b.name);
    });
    
    console.log(`\n📁 ${dirPath}\n`);
    console.log('Name'.padEnd(40) + 'Size'.padEnd(12) + 'Modified');
    console.log('─'.repeat(70));
    
    items.forEach(item => {
      const icon = item.isDir ? '📁' : '📄';
      const name = (icon + ' ' + item.name).padEnd(40);
      const size = item.isDir ? '<DIR>'.padEnd(12) : this.formatBytes(item.size).padEnd(12);
      const date = item.modified ? item.modified.toLocaleDateString('th-TH') : '';
      console.log(name + size + date);
    });
    
    console.log();
    return items;
  }
  
  // เปลี่ยน directory
  async cd(target) {
    let newPath;
    
    if (target === '..') {
      newPath = path.dirname(this.currentDir);
    } else if (path.isAbsolute(target)) {
      newPath = target;
    } else {
      newPath = path.join(this.currentDir, target);
    }
    
    const stat = await fsPromises.stat(newPath).catch(() => null);
    if (!stat?.isDirectory()) {
      throw new Error(`Not a directory: ${target}`);
    }
    
    this.history.push(this.currentDir);
    this.currentDir = newPath;
    return newPath;
  }
  
  // คัดลอกไฟล์
  async copy(src, dest) {
    const srcPath = path.resolve(this.currentDir, src);
    const destPath = path.resolve(this.currentDir, dest);
    
    const stat = await fsPromises.stat(srcPath);
    
    if (stat.isDirectory()) {
      await this.copyDir(srcPath, destPath);
    } else {
      await fsPromises.copyFile(srcPath, destPath);
    }
    
    console.log(`✅ Copied: ${src} → ${dest}`);
  }
  
  async copyDir(src, dest) {
    await fsPromises.mkdir(dest, { recursive: true });
    const entries = await fsPromises.readdir(src, { withFileTypes: true });
    
    for (const entry of entries) {
      const srcPath = path.join(src, entry.name);
      const destPath = path.join(dest, entry.name);
      
      if (entry.isDirectory()) {
        await this.copyDir(srcPath, destPath);
      } else {
        await fsPromises.copyFile(srcPath, destPath);
      }
    }
  }
  
  // ย้ายไฟล์
  async move(src, dest) {
    const srcPath = path.resolve(this.currentDir, src);
    const destPath = path.resolve(this.currentDir, dest);
    
    await fsPromises.rename(srcPath, destPath);
    console.log(`✅ Moved: ${src} → ${dest}`);
  }
  
  // ลบไฟล์
  async remove(target) {
    const targetPath = path.resolve(this.currentDir, target);
    const stat = await fsPromises.stat(targetPath);
    
    if (stat.isDirectory()) {
      await fsPromises.rm(targetPath, { recursive: true });
    } else {
      await fsPromises.unlink(targetPath);
    }
    
    console.log(`✅ Deleted: ${target}`);
  }
  
  // ดูเนื้อหาไฟล์
  async cat(filename) {
    const filePath = path.resolve(this.currentDir, filename);
    const content = await fsPromises.readFile(filePath, 'utf8');
    console.log(content);
    return content;
  }
  
  // หาไฟล์
  async find(pattern, dirPath = this.currentDir) {
    const results = [];
    const regex = new RegExp(pattern.replace('*', '.*'), 'i');
    
    async function search(dir) {
      const entries = await fsPromises.readdir(dir, { withFileTypes: true });
      
      for (const entry of entries) {
        const fullPath = path.join(dir, entry.name);
        
        if (regex.test(entry.name)) {
          results.push(fullPath);
        }
        
        if (entry.isDirectory() && entry.name !== 'node_modules') {
          await search(fullPath);
        }
      }
    }
    
    await search(dirPath);
    
    if (results.length === 0) {
      console.log('No files found');
    } else {
      results.forEach(r => console.log(r));
    }
    
    return results;
  }
  
  // ดู disk usage
  async du(dirPath = this.currentDir) {
    let totalSize = 0;
    
    async function calculateSize(dir) {
      const entries = await fsPromises.readdir(dir, { withFileTypes: true });
      
      for (const entry of entries) {
        const fullPath = path.join(dir, entry.name);
        
        if (entry.isDirectory()) {
          await calculateSize(fullPath);
        } else {
          const stat = await fsPromises.stat(fullPath).catch(() => ({ size: 0 }));
          totalSize += stat.size;
        }
      }
    }
    
    await calculateSize(dirPath);
    console.log(`Total size: ${this.formatBytes(totalSize)}`);
    return totalSize;
  }
  
  // Hash ไฟล์
  async hash(filename, algorithm = 'sha256') {
    const filePath = path.resolve(this.currentDir, filename);
    
    return new Promise((resolve, reject) => {
      const hash = crypto.createHash(algorithm);
      const stream = fs.createReadStream(filePath);
      
      stream.on('data', chunk => hash.update(chunk));
      stream.on('end', () => {
        const result = hash.digest('hex');
        console.log(`${algorithm}: ${result}`);
        resolve(result);
      });
      stream.on('error', reject);
    });
  }
  
  formatBytes(bytes) {
    if (bytes === 0) return '0 B';
    const k = 1024;
    const sizes = ['B', 'KB', 'MB', 'GB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return (bytes / Math.pow(k, i)).toFixed(1) + ' ' + sizes[i];
  }
}

// Main REPL
async function main() {
  const fm = new FileManager();
  
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
    prompt: () => `${fm.currentDir}> `
  });
  
  console.log('File Manager CLI - Type "help" for commands\n');
  
  rl.prompt();
  
  rl.on('line', async (input) => {
    const parts = input.trim().split(' ');
    const cmd = parts[0];
    const args = parts.slice(1);
    
    try {
      switch (cmd) {
        case 'ls': case 'dir':
          await fm.list(args[0] || fm.currentDir);
          break;
        case 'cd':
          await fm.cd(args[0] || '~');
          break;
        case 'pwd':
          console.log(fm.currentDir);
          break;
        case 'cat':
          await fm.cat(args[0]);
          break;
        case 'cp':
          await fm.copy(args[0], args[1]);
          break;
        case 'mv':
          await fm.move(args[0], args[1]);
          break;
        case 'rm':
          await fm.remove(args[0]);
          break;
        case 'mkdir':
          await fsPromises.mkdir(path.resolve(fm.currentDir, args[0]), { recursive: true });
          console.log(`✅ Created: ${args[0]}`);
          break;
        case 'find':
          await fm.find(args[0]);
          break;
        case 'du':
          await fm.du(args[0] || fm.currentDir);
          break;
        case 'hash':
          await fm.hash(args[0], args[1] || 'sha256');
          break;
        case 'help':
          console.log(`
Commands:
  ls [dir]          List directory contents
  cd <dir>          Change directory
  pwd               Print working directory
  cat <file>        Display file contents
  cp <src> <dest>   Copy file or directory
  mv <src> <dest>   Move/rename file or directory
  rm <path>         Delete file or directory
  mkdir <dir>       Create directory
  find <pattern>    Find files matching pattern
  du [dir]          Show disk usage
  hash <file> [algo] Calculate file hash
  exit              Exit
          `);
          break;
        case 'exit': case 'quit':
          console.log('Bye!');
          rl.close();
          process.exit(0);
          break;
        case '':
          break;
        default:
          console.log(`Unknown command: ${cmd}. Type "help" for help.`);
      }
    } catch (err) {
      console.error(`Error: ${err.message}`);
    }
    
    rl.setPrompt(`${fm.currentDir}> `);
    rl.prompt();
  });
}

main();
```

---

## ขั้นตอนที่ 98: สรุป Part 04 และ Exercise

### สิ่งที่เรียนรู้ใน Part 04

```
✅ อ่านไฟล์: readFileSync, readFile, fs.promises, readline
✅ เขียนไฟล์: writeFile, appendFile, atomic write
✅ Directory: mkdir, readdir, recursive operations
✅ File Streams: ReadStream, WriteStream, pipeline
✅ File Watching: fs.watch, fs.watchFile, custom watcher
✅ File-based Database: JSON database with CRUD
✅ Security: path traversal prevention, atomic write
✅ โปรเจกต์จริง: File Manager CLI
```

### 📝 Exercise

**Exercise 1: Log Analyzer**
```
สร้าง log-analyzer.js ที่:
1. อ่าน log file ขนาดใหญ่ด้วย streams
2. Parse log entries (timestamp, level, message)
3. นับจำนวนแต่ละ level
4. หา errors ที่เกิดบ่อยที่สุด
5. สร้าง summary report
6. Export เป็น CSV หรือ JSON
```

**Exercise 2: File Backup Tool**
```
สร้าง backup.js ที่:
1. Copy source directory ไปยัง backup directory
2. เพิ่ม timestamp ในชื่อ backup
3. ตรวจสอบ checksums
4. Compress backup (gzip)
5. ลบ backups เก่ากว่า N วัน
6. แสดง progress
```

**Exercise 3: Config Manager**
```
สร้าง ConfigManager ที่:
1. โหลด config จาก JSON/YAML
2. Merge กับ environment variables
3. Validate schema
4. Watch ไฟล์ config
5. Reload อัตโนมัติเมื่อ config เปลี่ยน
6. Support nested keys (get('database.host'))
```

---

## 🔜 Part ถัดไป

**Part 05: HTTP Module** จะครอบคลุม:
- สร้าง HTTP Server ด้วย http module
- Routing พื้นฐาน
- Request parsing
- Response handling
- HTTPS
- HTTP/2

---

*Part 04 สมบูรณ์ | ขั้นตอนที่ 91-140 จาก 1000*
