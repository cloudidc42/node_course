# ขั้นตอนที่ 601-700 จาก 1000
# Part 07: Streams และ Buffers ใน Node.js

---

## สารบัญ

1. [Buffer Fundamentals](#buffer-fundamentals)
2. [Readable Streams](#readable-streams)
3. [Writable Streams](#writable-streams)
4. [Transform Streams](#transform-streams)
5. [Duplex Streams](#duplex-streams)
6. [Pipeline](#pipeline)
7. [Practical: File Processing Pipeline](#practical-file-pipeline)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Buffer Fundamentals {#buffer-fundamentals}

`Buffer` คือ class ใน Node.js สำหรับจัดการข้อมูล binary โดยตรง เป็น fixed-size chunk of memory นอก V8 heap

### ทำไมต้อง Buffer?

```
ปัญหา: JavaScript ทำงานกับ strings (text) เป็นหลัก
แต่ในโลกจริงต้องจัดการ:
  - ไฟล์ภาพ, วิดีโอ, เสียง (binary data)
  - Network packets
  - Database binary fields
  - Encrypted data
  - Compressed data

Buffer ช่วยให้จัดการ binary data ได้โดยตรงและมีประสิทธิภาพ
```

### การสร้าง Buffer

```javascript
// วิธีที่ 1: Buffer.alloc() - สร้าง buffer ที่มี size กำหนด (เริ่มเป็น 0)
const buf1 = Buffer.alloc(10);
console.log(buf1);           // <Buffer 00 00 00 00 00 00 00 00 00 00>
console.log(buf1.length);    // 10

// วิธีที่ 2: Buffer.alloc() พร้อมกำหนดค่าเริ่มต้น
const buf2 = Buffer.alloc(10, 0xFF);
console.log(buf2);           // <Buffer ff ff ff ff ff ff ff ff ff ff>

// วิธีที่ 3: Buffer.allocUnsafe() - เร็วกว่าแต่ไม่ initialize (ข้อมูลเก่าอาจติดมา)
const buf3 = Buffer.allocUnsafe(10);
console.log(buf3);           // <Buffer [ข้อมูลสุ่ม]>
// ⚠️ ใช้ระวัง: ข้อมูลใน buffer อาจมีข้อมูลเก่า

// วิธีที่ 4: Buffer.from() จาก string
const buf4 = Buffer.from('สวัสดี Node.js', 'utf8');
console.log(buf4);           // <Buffer e0 b8 aa e0 b8 a7 e0 b8 b1 e0 b8 aa e0 b8 94 ...>
console.log(buf4.length);    // ขนาดใน bytes (ไม่ใช่จำนวนตัวอักษร!)
// ภาษาไทยใช้ 3 bytes ต่อตัวอักษรใน UTF-8

// วิธีที่ 5: Buffer.from() จาก array
const buf5 = Buffer.from([0x48, 0x65, 0x6c, 0x6c, 0x6f]);
console.log(buf5.toString()); // Hello

// วิธีที่ 6: Buffer.from() จาก ArrayBuffer
const arrayBuffer = new ArrayBuffer(8);
const buf6 = Buffer.from(arrayBuffer);
```

### การอ่านและเขียน Buffer

```javascript
const buf = Buffer.alloc(16);

// เขียนข้อมูลชนิดต่างๆ
buf.writeUInt8(255, 0);          // เขียน unsigned int 8-bit ที่ offset 0
buf.writeUInt16BE(1000, 1);      // เขียน unsigned int 16-bit big-endian ที่ offset 1
buf.writeUInt32LE(100000, 3);    // เขียน unsigned int 32-bit little-endian ที่ offset 3
buf.writeFloatBE(3.14, 7);       // เขียน float 32-bit big-endian ที่ offset 7
buf.writeDoubleBE(3.14159, 8);   // เขียน double 64-bit big-endian ที่ offset 8

console.log(buf);
// อ่านข้อมูลกลับ
console.log(buf.readUInt8(0));          // 255
console.log(buf.readUInt16BE(1));       // 1000
console.log(buf.readUInt32LE(3));       // 100000
console.log(buf.readFloatBE(7).toFixed(2));  // 3.14
```

### การแปลง Buffer เป็น String

```javascript
const buf = Buffer.from('Hello, สวัสดี!', 'utf8');

// แปลงกลับเป็น string
console.log(buf.toString('utf8'));    // Hello, สวัสดี!
console.log(buf.toString('hex'));     // ในรูปแบบ hexadecimal
console.log(buf.toString('base64')); // ในรูปแบบ base64

// แปลงเฉพาะส่วน
console.log(buf.toString('utf8', 0, 5)); // Hello

// JSON
const data = { name: 'สมชาย', age: 25 };
const jsonBuf = Buffer.from(JSON.stringify(data));
const parsed = JSON.parse(jsonBuf.toString());
console.log(parsed); // { name: 'สมชาย', age: 25 }
```

### Buffer Operations

```javascript
// Concatenate buffers
const buf1 = Buffer.from('Hello ');
const buf2 = Buffer.from('World');
const buf3 = Buffer.concat([buf1, buf2]);
console.log(buf3.toString()); // Hello World

// Copy buffer
const source = Buffer.from('Node.js Rocks!');
const dest = Buffer.alloc(source.length);
source.copy(dest);
console.log(dest.toString()); // Node.js Rocks!

// Slice buffer (reference, not copy!)
const original = Buffer.from('0123456789');
const slice = original.slice(2, 7); // '23456'
console.log(slice.toString()); // 23456

// เปลี่ยน slice จะเปลี่ยน original ด้วย!
slice[0] = 0x58; // 'X'
console.log(original.toString()); // 01X3456789

// ใช้ subarray แทน slice (ปลอดภัยกว่า)
const sub = original.subarray(2, 7);

// Compare buffers
const a = Buffer.from('abc');
const b = Buffer.from('abc');
const c = Buffer.from('xyz');
console.log(a.equals(b));    // true
console.log(Buffer.compare(a, c)); // -1 (a < c)

// Fill buffer
const filled = Buffer.alloc(10);
filled.fill(0xAB);
console.log(filled); // <Buffer ab ab ab ab ab ab ab ab ab ab>

// indexOf
const text = Buffer.from('Hello World Hello');
console.log(text.indexOf('Hello'));    // 0
console.log(text.indexOf('Hello', 1)); // 12 (เริ่มหาจาก index 1)
```

### TextEncoder และ TextDecoder (Web API ใน Node.js)

```javascript
// TextEncoder / TextDecoder เป็น Web API standard
const { TextEncoder, TextDecoder } = require('util');

const encoder = new TextEncoder();
const decoder = new TextDecoder('utf-8');

// encode string → Uint8Array
const encoded = encoder.encode('สวัสดี Node.js');
console.log(encoded instanceof Uint8Array); // true

// decode Uint8Array → string
const decoded = decoder.decode(encoded);
console.log(decoded); // สวัสดี Node.js

// รองรับ encoding อื่น
const tis620Decoder = new TextDecoder('windows-874'); // TIS-620/Windows-874
```

---

## Readable Streams {#readable-streams}

Readable Stream คือ stream ที่ใช้อ่านข้อมูล มีสอง modes:

```
Flowing mode: ข้อมูลไหลออกมาอัตโนมัติ (push)
  emitter.on('data', handler)

Paused mode: ต้องเรียก .read() เองเพื่อดึงข้อมูล (pull)
  readStream.read()
```

### การใช้ Readable Stream จาก File

```javascript
const fs = require('fs');

// สร้าง readable stream จากไฟล์
const readStream = fs.createReadStream('./largefile.txt', {
  encoding: 'utf8',
  highWaterMark: 64 * 1024  // 64KB chunks
});

let totalBytes = 0;
let chunkCount = 0;

// Flowing mode: ข้อมูลไหลออกมาผ่าน 'data' event
readStream.on('data', (chunk) => {
  chunkCount++;
  totalBytes += Buffer.byteLength(chunk);
  console.log(`Chunk ${chunkCount}: ${chunk.length} characters`);
});

readStream.on('end', () => {
  console.log(`อ่านเสร็จ: ${chunkCount} chunks, ${totalBytes} bytes`);
});

readStream.on('error', (err) => {
  console.error('เกิดข้อผิดพลาด:', err.message);
});
```

### การสร้าง Custom Readable Stream

```javascript
const { Readable } = require('stream');

// วิธีที่ 1: สืบทอดจาก Readable
class CounterStream extends Readable {
  constructor(start, end, options = {}) {
    super(options);
    this.current = start;
    this.end = end;
  }

  _read() {
    if (this.current <= this.end) {
      // push ข้อมูลออกไป
      this.push(`${this.current}\n`);
      this.current++;
    } else {
      // null = สัญญาณสิ้นสุด stream
      this.push(null);
    }
  }
}

const counter = new CounterStream(1, 5);
counter.on('data', (chunk) => process.stdout.write(chunk));
counter.on('end', () => console.log('นับเสร็จแล้ว!'));
// 1
// 2
// 3
// 4
// 5
// นับเสร็จแล้ว!
```

```javascript
// วิธีที่ 2: Readable.from() - จาก async generator
const { Readable } = require('stream');

async function* generateData() {
  const items = ['แอปเปิ้ล', 'กล้วย', 'ส้ม', 'มะม่วง', 'ลิ้นจี่'];
  for (const item of items) {
    // จำลองการดึงข้อมูล async
    await new Promise(r => setTimeout(r, 100));
    yield `${item}\n`;
  }
}

const fruitStream = Readable.from(generateData());

fruitStream.on('data', (chunk) => process.stdout.write(chunk));
fruitStream.on('end', () => console.log('ส่งข้อมูลผลไม้ครบแล้ว'));
```

```javascript
// วิธีที่ 3: ใช้ object mode
const { Readable } = require('stream');

class DatabaseReader extends Readable {
  constructor(data) {
    super({ objectMode: true }); // รับ-ส่ง objects ได้โดยตรง
    this.data = [...data];
    this.index = 0;
  }

  _read() {
    if (this.index < this.data.length) {
      this.push(this.data[this.index++]);
    } else {
      this.push(null);
    }
  }
}

const dbReader = new DatabaseReader([
  { id: 1, name: 'สมชาย', age: 25 },
  { id: 2, name: 'สมหญิง', age: 30 },
  { id: 3, name: 'สมศักดิ์', age: 22 }
]);

dbReader.on('data', (record) => {
  // ได้รับเป็น object โดยตรง
  console.log(`ID: ${record.id}, ชื่อ: ${record.name}`);
});
```

### Paused Mode

```javascript
const { Readable } = require('stream');
const stream = Readable.from(['a', 'b', 'c', 'd', 'e']);

// Paused mode: อ่านด้วยตนเอง
stream.on('readable', () => {
  let chunk;
  while (null !== (chunk = stream.read())) {
    console.log('อ่านได้:', chunk);
  }
});

// หรือใช้ for await...of
async function readAll() {
  for await (const chunk of stream) {
    console.log('chunk:', chunk);
  }
}
```

---

## Writable Streams {#writable-streams}

Writable Stream คือ stream ที่ใช้เขียนข้อมูล

### การใช้ Writable Stream จาก File

```javascript
const fs = require('fs');

const writeStream = fs.createWriteStream('./output.txt', {
  encoding: 'utf8',
  flags: 'w' // 'w' = เขียนใหม่, 'a' = ต่อท้าย
});

// เขียนข้อมูล
writeStream.write('บรรทัดที่ 1\n');
writeStream.write('บรรทัดที่ 2\n');
writeStream.write('บรรทัดที่ 3\n');

// เขียนข้อมูลสุดท้ายแล้วปิด stream
writeStream.end('บรรทัดสุดท้าย\n');

writeStream.on('finish', () => {
  console.log('เขียนไฟล์เสร็จแล้ว');
});

writeStream.on('error', (err) => {
  console.error('ข้อผิดพลาดในการเขียน:', err.message);
});
```

### Backpressure

```javascript
const fs = require('fs');

// Backpressure: ป้องกันการเขียนเร็วกว่าที่อ่านได้
const writeStream = fs.createWriteStream('./output.txt');

function writeData(data, callback) {
  // write() คืน false เมื่อ internal buffer เต็ม
  const ok = writeStream.write(data);

  if (!ok) {
    // รอจน drain (buffer ว่าง) ก่อนเขียนต่อ
    writeStream.once('drain', callback);
  } else {
    // Buffer ยังมีที่ว่าง เขียนต่อได้ทันที
    process.nextTick(callback);
  }
}

// เขียนข้อมูลจำนวนมากโดยไม่ overflow
let count = 0;
function writeLoop() {
  if (count < 10000) {
    writeData(`บรรทัดที่ ${count++}\n`, writeLoop);
  } else {
    writeStream.end();
  }
}

writeLoop();
```

### การสร้าง Custom Writable Stream

```javascript
const { Writable } = require('stream');

// Custom Writable: เก็บข้อมูลใน array
class ArrayWriter extends Writable {
  constructor(options = {}) {
    super(options);
    this.data = [];
    this.bytesWritten = 0;
  }

  _write(chunk, encoding, callback) {
    // chunk คือ Buffer หรือ string
    const str = chunk.toString();
    this.data.push(str);
    this.bytesWritten += chunk.length;

    // เรียก callback เมื่อเสร็จ (หรือส่ง error เป็น argument)
    callback();
  }

  _final(callback) {
    // เรียกเมื่อ stream กำลังจะปิด
    console.log(`บันทึก ${this.data.length} รายการ, ${this.bytesWritten} bytes`);
    callback();
  }
}

const writer = new ArrayWriter();
writer.write('รายการที่ 1\n');
writer.write('รายการที่ 2\n');
writer.write('รายการที่ 3\n');
writer.end();

writer.on('finish', () => {
  console.log('ข้อมูลทั้งหมด:', writer.data);
});
```

```javascript
// Custom Writable: Database writer
const { Writable } = require('stream');

class DatabaseWriter extends Writable {
  constructor(db, tableName, options = {}) {
    super({ ...options, objectMode: true });
    this.db = db;
    this.tableName = tableName;
    this.batch = [];
    this.batchSize = options.batchSize || 100;
    this.inserted = 0;
  }

  async _write(record, encoding, callback) {
    this.batch.push(record);

    if (this.batch.length >= this.batchSize) {
      try {
        await this._flushBatch();
        callback();
      } catch (err) {
        callback(err);
      }
    } else {
      callback();
    }
  }

  async _final(callback) {
    try {
      if (this.batch.length > 0) {
        await this._flushBatch();
      }
      console.log(`บันทึกข้อมูลทั้งหมด ${this.inserted} records`);
      callback();
    } catch (err) {
      callback(err);
    }
  }

  async _flushBatch() {
    // จำลองการบันทึกฐานข้อมูล
    console.log(`บันทึก batch: ${this.batch.length} records`);
    this.inserted += this.batch.length;
    this.batch = [];
    await new Promise(r => setTimeout(r, 10));
  }
}
```

---

## Transform Streams {#transform-streams}

Transform Stream คือ stream ที่ทั้งรับและส่งข้อมูล โดยแปลงข้อมูลระหว่างทาง

```
Input ──→ [Transform] ──→ Output
           แปลงข้อมูล
```

### Transform Stream พื้นฐาน

```javascript
const { Transform } = require('stream');

// Uppercase Transform
class UpperCaseTransform extends Transform {
  _transform(chunk, encoding, callback) {
    // แปลงข้อมูลและส่งต่อ
    this.push(chunk.toString().toUpperCase());
    callback();
  }
}

const upper = new UpperCaseTransform();
process.stdin.pipe(upper).pipe(process.stdout);
// พิมพ์ข้อความอะไรก็ได้ จะแสดงเป็นตัวพิมพ์ใหญ่
```

### CSV Parser Transform

```javascript
const { Transform } = require('stream');

class CSVParser extends Transform {
  constructor(options = {}) {
    super({ ...options, objectMode: true });
    this.buffer = '';
    this.headers = null;
    this.lineCount = 0;
  }

  _transform(chunk, encoding, callback) {
    this.buffer += chunk.toString();
    const lines = this.buffer.split('\n');

    // เก็บบรรทัดสุดท้ายไว้ (อาจยังไม่สมบูรณ์)
    this.buffer = lines.pop();

    for (const line of lines) {
      if (!line.trim()) continue;

      const values = this._parseLine(line);

      if (!this.headers) {
        // บรรทัดแรกคือ headers
        this.headers = values;
      } else {
        // แปลงเป็น object
        const record = {};
        this.headers.forEach((header, i) => {
          record[header] = values[i] || '';
        });
        this.lineCount++;
        this.push(record);
      }
    }

    callback();
  }

  _flush(callback) {
    // ประมวลผลข้อมูลที่เหลือใน buffer
    if (this.buffer.trim() && this.headers) {
      const values = this._parseLine(this.buffer);
      const record = {};
      this.headers.forEach((header, i) => {
        record[header] = values[i] || '';
      });
      this.push(record);
    }
    console.log(`แปลง CSV ได้ ${this.lineCount} records`);
    callback();
  }

  _parseLine(line) {
    // จัดการ quoted values
    const result = [];
    let current = '';
    let inQuotes = false;

    for (const char of line) {
      if (char === '"') {
        inQuotes = !inQuotes;
      } else if (char === ',' && !inQuotes) {
        result.push(current.trim());
        current = '';
      } else {
        current += char;
      }
    }
    result.push(current.trim());
    return result;
  }
}

// ตัวอย่างการใช้งาน
const { Readable } = require('stream');

const csvData = `id,name,age,email
1,สมชาย,25,somchai@example.com
2,สมหญิง,30,somying@example.com
3,สมศักดิ์,22,"somsakdi@example.com"
`;

const source = Readable.from([csvData]);
const parser = new CSVParser();

source.pipe(parser);

parser.on('data', (record) => {
  console.log('Record:', record);
});

parser.on('end', () => {
  console.log('แปลง CSV เสร็จแล้ว');
});
```

### Compression Transform

```javascript
const { createGzip, createGunzip } = require('zlib');
const { Transform } = require('stream');
const fs = require('fs');

// บีบอัดไฟล์
const compress = (inputFile, outputFile) => {
  return new Promise((resolve, reject) => {
    const readStream = fs.createReadStream(inputFile);
    const writeStream = fs.createWriteStream(outputFile);
    const gzip = createGzip({ level: 9 }); // level 1-9

    readStream
      .pipe(gzip)
      .pipe(writeStream)
      .on('finish', resolve)
      .on('error', reject);
  });
};

// คลายการบีบอัด
const decompress = (inputFile, outputFile) => {
  return new Promise((resolve, reject) => {
    const readStream = fs.createReadStream(inputFile);
    const writeStream = fs.createWriteStream(outputFile);
    const gunzip = createGunzip();

    readStream
      .pipe(gunzip)
      .pipe(writeStream)
      .on('finish', resolve)
      .on('error', reject);
  });
};

// ใช้งาน
async function main() {
  console.log('กำลังบีบอัด...');
  await compress('./data.txt', './data.txt.gz');
  console.log('บีบอัดเสร็จแล้ว');

  console.log('กำลังคลายการบีบอัด...');
  await decompress('./data.txt.gz', './data_uncompressed.txt');
  console.log('คลายการบีบอัดเสร็จแล้ว');
}
```

### Encryption Transform

```javascript
const { Transform } = require('stream');
const crypto = require('crypto');

class EncryptTransform extends Transform {
  constructor(key, iv) {
    super();
    this.cipher = crypto.createCipheriv('aes-256-cbc', key, iv);
  }

  _transform(chunk, encoding, callback) {
    const encrypted = this.cipher.update(chunk);
    this.push(encrypted);
    callback();
  }

  _flush(callback) {
    const final = this.cipher.final();
    if (final.length > 0) this.push(final);
    callback();
  }
}

class DecryptTransform extends Transform {
  constructor(key, iv) {
    super();
    this.decipher = crypto.createDecipheriv('aes-256-cbc', key, iv);
  }

  _transform(chunk, encoding, callback) {
    const decrypted = this.decipher.update(chunk);
    this.push(decrypted);
    callback();
  }

  _flush(callback) {
    const final = this.decipher.final();
    if (final.length > 0) this.push(final);
    callback();
  }
}

// ตัวอย่างการใช้งาน
const fs = require('fs');
const key = crypto.randomBytes(32);
const iv = crypto.randomBytes(16);

// เข้ารหัสไฟล์
fs.createReadStream('./secret.txt')
  .pipe(new EncryptTransform(key, iv))
  .pipe(fs.createWriteStream('./secret.enc'));

// ถอดรหัสไฟล์
fs.createReadStream('./secret.enc')
  .pipe(new DecryptTransform(key, iv))
  .pipe(fs.createWriteStream('./secret_decrypted.txt'));
```

---

## Duplex Streams {#duplex-streams}

Duplex Stream ทำหน้าที่เป็นทั้ง Readable และ Writable พร้อมกัน แต่ไม่มีการเชื่อมต่อระหว่าง input และ output

```
Input ──→ [Duplex] ──→ Output
           (อ่านและเขียนแยกกัน)
```

```javascript
const { Duplex } = require('stream');

// ตัวอย่าง: Network Socket (เป็น Duplex ตามธรรมชาติ)
class MockSocket extends Duplex {
  constructor(options = {}) {
    super(options);
    this._readBuffer = [];
    this._readPending = false;
  }

  // ส่วน Readable
  _read(size) {
    if (this._readBuffer.length > 0) {
      this.push(this._readBuffer.shift());
    } else {
      this._readPending = true;
    }
  }

  // ส่วน Writable
  _write(chunk, encoding, callback) {
    console.log('ส่งข้อมูล:', chunk.toString());
    // จำลองการรับ response
    setTimeout(() => {
      const response = `ACK: ${chunk.toString()}`;
      this._readBuffer.push(response);
      if (this._readPending) {
        this._readPending = false;
        this.push(this._readBuffer.shift());
      }
      callback();
    }, 100);
  }
}

const socket = new MockSocket();

socket.on('data', (data) => {
  console.log('ได้รับ response:', data.toString());
});

socket.write('สวัสดี');
socket.write('ข้อความที่ 2');
```

---

## Pipeline {#pipeline}

`stream.pipeline()` เป็นวิธีที่แนะนำสำหรับการเชื่อม streams เพราะจัดการ cleanup และ error handling ให้อัตโนมัติ

### pipeline vs pipe

```javascript
const { pipeline } = require('stream');
const { promisify } = require('util');
const fs = require('fs');
const zlib = require('zlib');

const pipelineAsync = promisify(pipeline);

// ❌ วิธีเก่า: pipe (ไม่จัดการ error ให้)
fs.createReadStream('./input.txt')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('./output.gz'))
  .on('error', (err) => console.error(err));
// ปัญหา: ถ้า stream กลางเกิด error, streams อื่นไม่ถูก destroy อัตโนมัติ

// ✅ วิธีใหม่: pipeline (จัดการ cleanup อัตโนมัติ)
async function compress() {
  try {
    await pipelineAsync(
      fs.createReadStream('./input.txt'),
      zlib.createGzip(),
      fs.createWriteStream('./output.gz')
    );
    console.log('บีบอัดสำเร็จ!');
  } catch (err) {
    console.error('เกิดข้อผิดพลาด:', err.message);
  }
}
```

### Node.js 16+: stream.pipeline() with async generators

```javascript
const { pipeline } = require('stream/promises');
const fs = require('fs');

async function processFile() {
  await pipeline(
    // 1. อ่านไฟล์
    fs.createReadStream('./data.csv'),

    // 2. แปลงเป็น chunks ทีละบรรทัด
    async function* (source) {
      let buffer = '';
      for await (const chunk of source) {
        buffer += chunk;
        const lines = buffer.split('\n');
        buffer = lines.pop();
        for (const line of lines) {
          if (line.trim()) yield line + '\n';
        }
      }
      if (buffer.trim()) yield buffer;
    },

    // 3. แปลงเป็น uppercase
    async function* (source) {
      for await (const line of source) {
        yield line.toUpperCase();
      }
    },

    // 4. เขียนออกไฟล์
    fs.createWriteStream('./output.txt')
  );

  console.log('ประมวลผลเสร็จแล้ว');
}

processFile();
```

### การ Monitor Progress ใน Pipeline

```javascript
const { Transform, pipeline } = require('stream');
const { promisify } = require('util');
const fs = require('fs');

const pipelineAsync = promisify(pipeline);

class ProgressTransform extends Transform {
  constructor(totalSize, options = {}) {
    super(options);
    this.totalSize = totalSize;
    this.processed = 0;
    this.startTime = Date.now();
    this.lastReport = 0;
  }

  _transform(chunk, encoding, callback) {
    this.processed += chunk.length;
    const percent = Math.round((this.processed / this.totalSize) * 100);

    // รายงานทุก 5%
    if (percent - this.lastReport >= 5) {
      this.lastReport = percent;
      const elapsed = (Date.now() - this.startTime) / 1000;
      const speed = (this.processed / elapsed / 1024 / 1024).toFixed(2);
      process.stdout.write(`\r Progress: ${percent}% | Speed: ${speed} MB/s`);
    }

    this.push(chunk);
    callback();
  }

  _flush(callback) {
    process.stdout.write('\n');
    const elapsed = ((Date.now() - this.startTime) / 1000).toFixed(2);
    console.log(`เสร็จสมบูรณ์! ใช้เวลา ${elapsed} วินาที`);
    callback();
  }
}

// ใช้งาน
async function copyWithProgress(src, dest) {
  const { size } = fs.statSync(src);
  const progress = new ProgressTransform(size);

  await pipelineAsync(
    fs.createReadStream(src),
    progress,
    fs.createWriteStream(dest)
  );
}
```

---

## Practical: File Processing Pipeline {#practical-file-pipeline}

สร้าง pipeline สำหรับประมวลผลไฟล์ CSV ขนาดใหญ่:

```javascript
const { Transform, pipeline } = require('stream');
const { promisify } = require('util');
const fs = require('fs');
const zlib = require('zlib');

const pipelineAsync = promisify(pipeline);

// 1. CSV Line Parser
class LineParser extends Transform {
  constructor() {
    super({ objectMode: true });
    this.buffer = '';
    this.lineNumber = 0;
    this.headers = null;
  }

  _transform(chunk, encoding, callback) {
    this.buffer += chunk.toString();
    const lines = this.buffer.split('\n');
    this.buffer = lines.pop();

    for (const line of lines) {
      if (!line.trim()) continue;
      this.lineNumber++;
      const fields = line.split(',').map(f => f.trim().replace(/^"|"$/g, ''));

      if (!this.headers) {
        this.headers = fields;
      } else {
        const record = {};
        this.headers.forEach((h, i) => record[h] = fields[i] || '');
        this.push(record);
      }
    }
    callback();
  }

  _flush(callback) {
    if (this.buffer.trim() && this.headers) {
      const fields = this.buffer.split(',').map(f => f.trim());
      const record = {};
      this.headers.forEach((h, i) => record[h] = fields[i] || '');
      this.push(record);
    }
    callback();
  }
}

// 2. Data Validator
class DataValidator extends Transform {
  constructor(schema) {
    super({ objectMode: true });
    this.schema = schema;
    this.validCount = 0;
    this.invalidCount = 0;
    this.errors = [];
  }

  _transform(record, encoding, callback) {
    const errors = this._validate(record);
    if (errors.length === 0) {
      this.validCount++;
      this.push(record);
    } else {
      this.invalidCount++;
      this.errors.push({ record, errors });
      this.emit('invalid', { record, errors });
    }
    callback();
  }

  _validate(record) {
    const errors = [];
    for (const [field, rules] of Object.entries(this.schema)) {
      const value = record[field];

      if (rules.required && (!value || value === '')) {
        errors.push(`${field} จำเป็นต้องมีค่า`);
        continue;
      }

      if (value && rules.type === 'number' && isNaN(Number(value))) {
        errors.push(`${field} ต้องเป็นตัวเลข`);
      }

      if (value && rules.minLength && value.length < rules.minLength) {
        errors.push(`${field} ต้องมีความยาวอย่างน้อย ${rules.minLength} ตัวอักษร`);
      }

      if (value && rules.pattern && !rules.pattern.test(value)) {
        errors.push(`${field} มีรูปแบบไม่ถูกต้อง`);
      }
    }
    return errors;
  }
}

// 3. Data Transformer
class DataTransformer extends Transform {
  constructor(transformFn) {
    super({ objectMode: true });
    this.transformFn = transformFn;
    this.count = 0;
  }

  _transform(record, encoding, callback) {
    try {
      const transformed = this.transformFn(record);
      this.count++;
      this.push(transformed);
      callback();
    } catch (err) {
      callback(err);
    }
  }
}

// 4. JSON Serializer
class JSONSerializer extends Transform {
  constructor() {
    super({ writableObjectMode: true });
    this.first = true;
    this.push('[\n');
  }

  _transform(record, encoding, callback) {
    const prefix = this.first ? '  ' : ',\n  ';
    this.first = false;
    this.push(prefix + JSON.stringify(record));
    callback();
  }

  _flush(callback) {
    this.push('\n]\n');
    callback();
  }
}

// ========== สร้าง test data ==========
function createTestCSV(filename, rows = 1000) {
  const ws = fs.createWriteStream(filename);
  ws.write('id,name,age,email,salary\n');

  const names = ['สมชาย', 'สมหญิง', 'สมศักดิ์', 'วิชัย', 'มาลี'];

  for (let i = 1; i <= rows; i++) {
    const name = names[i % names.length];
    ws.write(`${i},${name},${20 + (i % 40)},user${i}@example.com,${30000 + i * 100}\n`);
  }
  ws.end();
  return new Promise(r => ws.on('finish', r));
}

// ========== Run Pipeline ==========
async function runPipeline() {
  const inputFile = '/tmp/test_data.csv';
  const outputFile = '/tmp/processed_data.json';

  console.log('สร้างข้อมูลทดสอบ...');
  await createTestCSV(inputFile, 100);
  console.log('สร้างข้อมูลทดสอบเสร็จแล้ว');

  const validator = new DataValidator({
    id:     { required: true, type: 'number' },
    name:   { required: true, minLength: 2 },
    age:    { required: true, type: 'number' },
    email:  { required: true, pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/ },
    salary: { type: 'number' }
  });

  const transformer = new DataTransformer((record) => ({
    ...record,
    id: parseInt(record.id),
    age: parseInt(record.age),
    salary: parseFloat(record.salary),
    fullInfo: `${record.name} (${record.age} ปี) - ${record.email}`,
    processedAt: new Date().toISOString()
  }));

  // ติดตาม validation errors
  validator.on('invalid', ({ record, errors }) => {
    console.warn(`⚠️ Invalid record ID ${record.id}:`, errors.join(', '));
  });

  console.log('เริ่มประมวลผล...');
  const startTime = Date.now();

  try {
    await pipelineAsync(
      fs.createReadStream(inputFile),
      new LineParser(),
      validator,
      transformer,
      new JSONSerializer(),
      fs.createWriteStream(outputFile)
    );

    const elapsed = Date.now() - startTime;
    console.log(`\nเสร็จสมบูรณ์! ใช้เวลา ${elapsed}ms`);
    console.log(`  ✓ ข้อมูลถูกต้อง: ${validator.validCount} records`);
    console.log(`  ✗ ข้อมูลไม่ถูกต้อง: ${validator.invalidCount} records`);
    console.log(`  แปลงข้อมูล: ${transformer.count} records`);
    console.log(`  บันทึกที่: ${outputFile}`);
  } catch (err) {
    console.error('Pipeline error:', err.message);
  }
}

runPipeline();
```

---

## แบบฝึกหัด {#แบบฝึกหัด}

### แบบฝึกหัดที่ 1: Log File Analyzer

สร้าง stream pipeline ที่:
1. อ่านไฟล์ log ขนาดใหญ่ (จำลองเองได้)
2. แยก parse log entries
3. กรองเฉพาะ ERROR level
4. สรุปจำนวน errors ตาม type
5. เขียนผลลัพธ์เป็น JSON

```javascript
// Format ของ log:
// [2024-01-15T10:30:00.000Z] INFO [App] Server started
// [2024-01-15T10:30:01.000Z] ERROR [Database] Connection failed: timeout
// [2024-01-15T10:30:02.000Z] WARN [API] Rate limit approaching

// ผลลัพธ์ที่ต้องการ:
// {
//   totalErrors: 150,
//   byType: { Database: 50, API: 30, ... },
//   firstError: "...",
//   lastError: "..."
// }
```

### แบบฝึกหัดที่ 2: Image Thumbnail Generator (แนวคิด)

สร้าง Transform stream ที่ resize ภาพ:
- รับ image data เป็น stream
- แปลงเป็น thumbnail ขนาดเล็ก
- บันทึกลงไฟล์

### แบบฝึกหัดที่ 3: Real-time Data Stream

สร้าง Readable stream ที่:
- ส่งข้อมูล sensor (อุณหภูมิ, ความชื้น) แบบ real-time
- มี Transform ที่คำนวณค่าเฉลี่ยทุก 10 readings
- ส่งแจ้งเตือนเมื่อค่าเกิน threshold

---

## สรุป

| Stream Type | หน้าที่ | ตัวอย่าง |
|-------------|---------|---------|
| Buffer | เก็บข้อมูล binary | image data, network packets |
| Readable | ส่งข้อมูลออก | fs.createReadStream, HTTP request |
| Writable | รับข้อมูลเข้า | fs.createWriteStream, HTTP response |
| Transform | แปลงข้อมูล | gzip, csv parser, encryption |
| Duplex | อ่านและเขียน | TCP socket, WebSocket |

---

## ก้าวต่อไป

➡️ **Part 08: Async Programming** - Callbacks, Promises, async/await

---
*Node.js/Express.js Course - Part 07 of 20*
*ขั้นตอนที่ 601-700 จาก 1000*
