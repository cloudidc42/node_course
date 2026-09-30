# ขั้นตอนที่ 501-600 จาก 1000
# Part 06: Events และ EventEmitter ใน Node.js

---

## สารบัญ

1. [บทนำ: Event-Driven Architecture](#บทนำ)
2. [EventEmitter Class](#eventemitter-class)
3. [Custom Events](#custom-events)
4. [Built-in Events ใน Node.js](#built-in-events)
5. [Event Patterns](#event-patterns)
6. [Memory Leak Prevention](#memory-leak-prevention)
7. [Practical Examples: Logger System](#practical-logger)
8. [Practical Examples: PubSub System](#practical-pubsub)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## บทนำ: Event-Driven Architecture {#บทนำ}

Node.js ถูกออกแบบมาบนพื้นฐานของ **Event-Driven Architecture** ซึ่งหมายความว่าแทนที่จะรอผลลัพธ์แบบ synchronous โปรแกรมจะ "ฟัง" (listen) เหตุการณ์ต่างๆ และตอบสนองเมื่อเหตุการณ์นั้นเกิดขึ้น

### ทำไมต้อง Event-Driven?

```
แบบ Synchronous (บล็อก):
────────────────────────────────────────
อ่านไฟล์ → รอ → รอ → รอ → ได้ข้อมูล → ทำต่อ
         [ไม่ทำอะไรเลยระหว่างรอ]

แบบ Event-Driven (ไม่บล็อก):
────────────────────────────────────────
อ่านไฟล์ → ทำงานอื่น → ทำงานอื่น → [เหตุการณ์: อ่านเสร็จแล้ว!] → จัดการข้อมูล
           [ทำงานอื่นได้ระหว่างรอ]
```

### Event Loop ของ Node.js

```
          ┌───────────────────────────┐
      ┌─> │        timers             │  ← setTimeout, setInterval
      │   └─────────────┬─────────────┘
      │   ┌─────────────┴─────────────┐
      │   │     pending callbacks     │  ← I/O errors
      │   └─────────────┬─────────────┘
      │   ┌─────────────┴─────────────┐
      │   │       idle, prepare       │  ← internal use
      │   └─────────────┬─────────────┘
      │   ┌─────────────┴─────────────┐
      │   │           poll            │  ← I/O events, เหตุการณ์ใหม่
      │   └─────────────┬─────────────┘
      │   ┌─────────────┴─────────────┐
      │   │           check           │  ← setImmediate
      │   └─────────────┬─────────────┘
      │   ┌─────────────┴─────────────┐
      └── │      close callbacks      │  ← .on('close', ...)
          └───────────────────────────┘
```

---

## EventEmitter Class {#eventemitter-class}

`EventEmitter` เป็น class หลักของ Node.js ที่ใช้จัดการ events ทั้งหมด อยู่ใน module `events`

### การ import และใช้งานเบื้องต้น

```javascript
// วิธีที่ 1: ใช้ EventEmitter โดยตรง
const EventEmitter = require('events');

const emitter = new EventEmitter();

// ลงทะเบียน listener สำหรับ event 'greet'
emitter.on('greet', (name) => {
  console.log(`สวัสดี, ${name}!`);
});

// ปล่อย (emit) event 'greet' พร้อม argument
emitter.emit('greet', 'สมชาย');
// ผลลัพธ์: สวัสดี, สมชาย!
```

```javascript
// วิธีที่ 2: สืบทอด (inherit) จาก EventEmitter
const EventEmitter = require('events');

class MyService extends EventEmitter {
  constructor() {
    super();
    this.data = [];
  }

  addItem(item) {
    this.data.push(item);
    // ปล่อย event เมื่อเพิ่มข้อมูล
    this.emit('itemAdded', item, this.data.length);
  }

  removeItem(item) {
    const index = this.data.indexOf(item);
    if (index !== -1) {
      this.data.splice(index, 1);
      this.emit('itemRemoved', item);
    }
  }
}

const service = new MyService();

// ฟัง event 'itemAdded'
service.on('itemAdded', (item, count) => {
  console.log(`เพิ่ม: ${item}, จำนวนทั้งหมด: ${count}`);
});

service.addItem('แอปเปิ้ล');  // เพิ่ม: แอปเปิ้ล, จำนวนทั้งหมด: 1
service.addItem('กล้วย');      // เพิ่ม: กล้วย, จำนวนทั้งหมด: 2
```

### Methods หลักของ EventEmitter

#### 1. `.on(event, listener)` - ลงทะเบียน listener แบบถาวร

```javascript
const emitter = new EventEmitter();

// listener นี้จะถูกเรียกทุกครั้งที่ event 'data' ถูก emit
emitter.on('data', (value) => {
  console.log('ได้รับข้อมูล:', value);
});

emitter.emit('data', 'ครั้งที่ 1');  // ได้รับข้อมูล: ครั้งที่ 1
emitter.emit('data', 'ครั้งที่ 2');  // ได้รับข้อมูล: ครั้งที่ 2
emitter.emit('data', 'ครั้งที่ 3');  // ได้รับข้อมูล: ครั้งที่ 3
```

#### 2. `.once(event, listener)` - ลงทะเบียน listener แบบครั้งเดียว

```javascript
const emitter = new EventEmitter();

// listener นี้จะถูกเรียกแค่ครั้งเดียว แล้วถูกลบออกอัตโนมัติ
emitter.once('connect', () => {
  console.log('เชื่อมต่อครั้งแรกแล้ว!');
});

emitter.emit('connect');  // เชื่อมต่อครั้งแรกแล้ว!
emitter.emit('connect');  // (ไม่มีผลลัพธ์ - listener ถูกลบแล้ว)
emitter.emit('connect');  // (ไม่มีผลลัพธ์)
```

#### 3. `.off(event, listener)` / `.removeListener()` - ลบ listener

```javascript
const emitter = new EventEmitter();

// ต้องเก็บ reference ของ function ไว้เพื่อลบ
const handler = (data) => {
  console.log('handler:', data);
};

emitter.on('message', handler);
emitter.emit('message', 'ทดสอบ 1');  // handler: ทดสอบ 1

// ลบ listener
emitter.off('message', handler);
// หรือ: emitter.removeListener('message', handler);

emitter.emit('message', 'ทดสอบ 2');  // (ไม่มีผลลัพธ์)
```

#### 4. `.removeAllListeners([event])` - ลบ listener ทั้งหมด

```javascript
const emitter = new EventEmitter();

emitter.on('event1', () => console.log('event1 - listener 1'));
emitter.on('event1', () => console.log('event1 - listener 2'));
emitter.on('event2', () => console.log('event2 - listener 1'));

// ลบ listeners ทั้งหมดของ event1
emitter.removeAllListeners('event1');

// ลบ listeners ทั้งหมดของทุก event
// emitter.removeAllListeners();

emitter.emit('event1');  // (ไม่มีผลลัพธ์)
emitter.emit('event2');  // event2 - listener 1
```

#### 5. `.emit(event, ...args)` - ปล่อย event

```javascript
const emitter = new EventEmitter();

emitter.on('calculate', (a, b, operation) => {
  let result;
  switch(operation) {
    case 'add': result = a + b; break;
    case 'sub': result = a - b; break;
    case 'mul': result = a * b; break;
    case 'div': result = a / b; break;
  }
  console.log(`${a} ${operation} ${b} = ${result}`);
});

// emit พร้อมส่ง arguments หลายตัว
emitter.emit('calculate', 10, 5, 'add');  // 10 add 5 = 15
emitter.emit('calculate', 10, 5, 'mul');  // 10 mul 5 = 50
```

#### 6. `.listeners(event)` - ดู listeners ที่ลงทะเบียนไว้

```javascript
const emitter = new EventEmitter();

emitter.on('test', () => {});
emitter.on('test', () => {});
emitter.on('test', () => {});

console.log(emitter.listeners('test').length);  // 3
console.log(emitter.listenerCount('test'));      // 3
```

#### 7. `.prependListener()` - เพิ่ม listener ไว้ต้นคิว

```javascript
const emitter = new EventEmitter();

emitter.on('process', () => console.log('ขั้นตอนที่ 2'));
emitter.on('process', () => console.log('ขั้นตอนที่ 3'));
emitter.prependListener('process', () => console.log('ขั้นตอนที่ 1 (prepend)'));

emitter.emit('process');
// ขั้นตอนที่ 1 (prepend)
// ขั้นตอนที่ 2
// ขั้นตอนที่ 3
```

#### 8. `.eventNames()` - ดูชื่อ events ทั้งหมด

```javascript
const emitter = new EventEmitter();

emitter.on('connect', () => {});
emitter.on('data', () => {});
emitter.on('error', () => {});
emitter.on('close', () => {});

console.log(emitter.eventNames());
// ['connect', 'data', 'error', 'close']
```

---

## Custom Events {#custom-events}

### การออกแบบ Custom Events ที่ดี

```javascript
const EventEmitter = require('events');

// ตัวอย่าง: ระบบจัดการ Order
class OrderManager extends EventEmitter {
  constructor() {
    super();
    this.orders = new Map();
    this.nextOrderId = 1;
  }

  createOrder(customer, items) {
    const orderId = `ORD-${String(this.nextOrderId++).padStart(5, '0')}`;
    const order = {
      id: orderId,
      customer,
      items,
      status: 'pending',
      createdAt: new Date(),
      total: items.reduce((sum, item) => sum + item.price * item.qty, 0)
    };

    this.orders.set(orderId, order);

    // ปล่อย event พร้อมข้อมูล order
    this.emit('orderCreated', { ...order });
    return orderId;
  }

  processOrder(orderId) {
    const order = this.orders.get(orderId);
    if (!order) {
      this.emit('error', new Error(`ไม่พบ order: ${orderId}`));
      return;
    }

    order.status = 'processing';
    this.emit('orderProcessing', { orderId, customer: order.customer });

    // จำลองการประมวลผล async
    setTimeout(() => {
      order.status = 'completed';
      order.completedAt = new Date();
      this.emit('orderCompleted', { ...order });
    }, 1000);
  }

  cancelOrder(orderId, reason) {
    const order = this.orders.get(orderId);
    if (!order) {
      this.emit('error', new Error(`ไม่พบ order: ${orderId}`));
      return;
    }

    const previousStatus = order.status;
    order.status = 'cancelled';
    order.cancelReason = reason;

    this.emit('orderCancelled', {
      orderId,
      previousStatus,
      reason,
      customer: order.customer
    });
  }
}

// การใช้งาน
const orderManager = new OrderManager();

// ลงทะเบียน listeners
orderManager.on('orderCreated', (order) => {
  console.log(`[สร้าง] Order ${order.id} สำหรับ ${order.customer}`);
  console.log(`  รายการ: ${order.items.length} รายการ`);
  console.log(`  ยอดรวม: ${order.total} บาท`);
});

orderManager.on('orderProcessing', ({ orderId, customer }) => {
  console.log(`[ดำเนินการ] กำลังประมวลผล Order ${orderId} ของ ${customer}`);
});

orderManager.on('orderCompleted', (order) => {
  const duration = order.completedAt - order.createdAt;
  console.log(`[เสร็จสิ้น] Order ${order.id} เสร็จแล้ว (ใช้เวลา ${duration}ms)`);
});

orderManager.on('orderCancelled', ({ orderId, reason }) => {
  console.log(`[ยกเลิก] Order ${orderId} ถูกยกเลิก: ${reason}`);
});

orderManager.on('error', (err) => {
  console.error('[ข้อผิดพลาด]', err.message);
});

// ทดสอบ
const id1 = orderManager.createOrder('สมชาย', [
  { name: 'แล็ปท็อป', price: 35000, qty: 1 },
  { name: 'เมาส์', price: 500, qty: 2 }
]);

orderManager.processOrder(id1);

const id2 = orderManager.createOrder('สมหญิง', [
  { name: 'คีย์บอร์ด', price: 1500, qty: 1 }
]);
orderManager.cancelOrder(id2, 'เปลี่ยนใจ');
```

### การใช้ Symbol เป็น Event Name (ป้องกัน collision)

```javascript
const EventEmitter = require('events');

// ใช้ Symbol แทน string เพื่อป้องกัน name collision
const Events = {
  DATA_RECEIVED: Symbol('dataReceived'),
  DATA_PROCESSED: Symbol('dataProcessed'),
  ERROR: Symbol('error')
};

class DataPipeline extends EventEmitter {
  receive(data) {
    console.log('รับข้อมูล:', data);
    this.emit(Events.DATA_RECEIVED, data);
  }

  process(data) {
    const processed = data.toString().toUpperCase();
    this.emit(Events.DATA_PROCESSED, processed);
  }
}

const pipeline = new DataPipeline();

pipeline.on(Events.DATA_RECEIVED, (data) => {
  console.log('จัดการข้อมูลที่รับมา:', data);
  pipeline.process(data);
});

pipeline.on(Events.DATA_PROCESSED, (data) => {
  console.log('ข้อมูลหลังประมวลผล:', data);
});

pipeline.receive('hello world');
```

---

## Built-in Events ใน Node.js {#built-in-events}

### process Events

```javascript
// process เป็น EventEmitter ระดับ global

// 1. uncaughtException - ข้อผิดพลาดที่ไม่ถูก catch
process.on('uncaughtException', (error) => {
  console.error('Uncaught Exception:', error);
  // ควร log แล้ว exit gracefully
  process.exit(1);
});

// 2. unhandledRejection - Promise rejection ที่ไม่ถูก handle
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Promise Rejection:', reason);
  process.exit(1);
});

// 3. SIGTERM - สัญญาณหยุดโปรแกรม (เช่น จาก kill command)
process.on('SIGTERM', () => {
  console.log('ได้รับ SIGTERM กำลัง graceful shutdown...');
  // ปิด database connections, finish pending requests ฯลฯ
  process.exit(0);
});

// 4. SIGINT - Ctrl+C
process.on('SIGINT', () => {
  console.log('\nได้รับ SIGINT (Ctrl+C) กำลังหยุดทำงาน...');
  process.exit(0);
});

// 5. exit - เมื่อ process กำลังจะออก
process.on('exit', (code) => {
  // synchronous เท่านั้น! ไม่มี async ที่นี่
  console.log(`โปรแกรมจบการทำงาน รหัส: ${code}`);
});

// 6. beforeExit - เมื่อ event loop ว่าง (ก่อน exit)
process.on('beforeExit', (code) => {
  console.log('beforeExit:', code);
  // ยังทำ async ได้ที่นี่
});
```

### HTTP Server Events

```javascript
const http = require('http');

const server = http.createServer();

// 1. request - เมื่อมี HTTP request เข้ามา
server.on('request', (req, res) => {
  console.log(`${req.method} ${req.url}`);
  res.end('สวัสดี!');
});

// 2. connection - เมื่อมี TCP connection ใหม่
server.on('connection', (socket) => {
  console.log('Connection ใหม่:', socket.remoteAddress);
});

// 3. close - เมื่อ server ปิด
server.on('close', () => {
  console.log('Server ปิดแล้ว');
});

// 4. error - เมื่อเกิดข้อผิดพลาด
server.on('error', (err) => {
  if (err.code === 'EADDRINUSE') {
    console.error('Port นี้ถูกใช้งานอยู่แล้ว!');
  } else {
    console.error('Server error:', err);
  }
});

// 5. listening - เมื่อ server เริ่มรับ connections
server.on('listening', () => {
  const { port } = server.address();
  console.log(`Server ทำงานที่ port ${port}`);
});

server.listen(3000);
```

### Stream Events

```javascript
const fs = require('fs');

const readStream = fs.createReadStream('./largefile.txt');

// 1. data - เมื่อได้รับข้อมูล chunk
readStream.on('data', (chunk) => {
  console.log(`ได้รับ ${chunk.length} bytes`);
});

// 2. end - เมื่ออ่านข้อมูลหมดแล้ว
readStream.on('end', () => {
  console.log('อ่านไฟล์เสร็จแล้ว');
});

// 3. error - เมื่อเกิดข้อผิดพลาด
readStream.on('error', (err) => {
  console.error('เกิดข้อผิดพลาด:', err.message);
});

// 4. close - เมื่อ stream ปิด
readStream.on('close', () => {
  console.log('Stream ปิดแล้ว');
});

// 5. readable - เมื่อมีข้อมูลพร้อมอ่าน
readStream.on('readable', () => {
  let chunk;
  while (null !== (chunk = readStream.read())) {
    console.log(`อ่านได้ ${chunk.length} bytes`);
  }
});
```

---

## Event Patterns {#event-patterns}

### Pattern 1: Event-based State Machine

```javascript
const EventEmitter = require('events');

class TrafficLight extends EventEmitter {
  constructor() {
    super();
    this.state = 'red';
    this.transitions = {
      red: 'green',
      green: 'yellow',
      yellow: 'red'
    };
    this.durations = {
      red: 3000,    // 3 วินาที
      green: 4000,  // 4 วินาที
      yellow: 1000  // 1 วินาที
    };
  }

  start() {
    this.emit('stateChange', this.state);
    this._scheduleNext();
  }

  _scheduleNext() {
    setTimeout(() => {
      const nextState = this.transitions[this.state];
      this.state = nextState;
      this.emit('stateChange', nextState);
      this._scheduleNext();
    }, this.durations[this.state]);
  }
}

const light = new TrafficLight();

const colors = { red: '🔴', green: '🟢', yellow: '🟡' };

light.on('stateChange', (state) => {
  console.log(`${colors[state]} ${state.toUpperCase()}`);
});

light.start();
// หลังจากนั้นจะวนเป็น red → green → yellow → red → ...
```

### Pattern 2: Event-based Observer

```javascript
const EventEmitter = require('events');

// Observable data store
class Store extends EventEmitter {
  constructor(initialState = {}) {
    super();
    this._state = { ...initialState };
  }

  get state() {
    return { ...this._state };
  }

  setState(newState) {
    const oldState = { ...this._state };
    this._state = { ...this._state, ...newState };

    // หา keys ที่เปลี่ยนแปลง
    const changedKeys = Object.keys(newState).filter(
      key => oldState[key] !== newState[key]
    );

    changedKeys.forEach(key => {
      this.emit(`change:${key}`, this._state[key], oldState[key]);
    });

    if (changedKeys.length > 0) {
      this.emit('change', this._state, oldState);
    }
  }
}

// ตัวอย่างการใช้งาน
const userStore = new Store({
  name: 'สมชาย',
  age: 25,
  email: 'somchai@example.com',
  loggedIn: false
});

// ฟังการเปลี่ยนแปลงทุกอย่าง
userStore.on('change', (newState, oldState) => {
  console.log('State เปลี่ยนแปลง:', Object.keys(newState)
    .filter(k => newState[k] !== oldState[k])
    .map(k => `${k}: ${oldState[k]} → ${newState[k]}`)
    .join(', ')
  );
});

// ฟังการเปลี่ยนแปลงเฉพาะ field
userStore.on('change:loggedIn', (newVal, oldVal) => {
  if (newVal) {
    console.log(`${userStore.state.name} เข้าสู่ระบบแล้ว`);
  } else {
    console.log(`${userStore.state.name} ออกจากระบบแล้ว`);
  }
});

userStore.setState({ loggedIn: true });
userStore.setState({ age: 26 });
userStore.setState({ name: 'สมชาย จันทร์ดี', age: 27 });
```

### Pattern 3: Event Queue / Batching

```javascript
const EventEmitter = require('events');

class BatchProcessor extends EventEmitter {
  constructor(batchSize = 5, flushInterval = 1000) {
    super();
    this.batch = [];
    this.batchSize = batchSize;
    this.flushInterval = flushInterval;
    this.timer = null;
  }

  add(item) {
    this.batch.push(item);
    this.emit('itemAdded', item);

    if (this.batch.length >= this.batchSize) {
      this._flush();
    } else if (!this.timer) {
      // ตั้ง timer สำหรับ auto-flush
      this.timer = setTimeout(() => this._flush(), this.flushInterval);
    }
  }

  _flush() {
    if (this.timer) {
      clearTimeout(this.timer);
      this.timer = null;
    }

    if (this.batch.length === 0) return;

    const items = [...this.batch];
    this.batch = [];

    this.emit('flush', items);
  }

  stop() {
    this._flush();
    this.emit('stopped');
  }
}

const processor = new BatchProcessor(3, 2000);

processor.on('flush', (items) => {
  console.log(`ประมวลผล batch: [${items.join(', ')}]`);
});

processor.on('stopped', () => {
  console.log('BatchProcessor หยุดทำงานแล้ว');
});

// เพิ่มข้อมูล
for (let i = 1; i <= 10; i++) {
  processor.add(`item${i}`);
}

setTimeout(() => processor.stop(), 3000);
```

### Pattern 4: Middleware Chain ผ่าน Events

```javascript
const EventEmitter = require('events');

class Pipeline extends EventEmitter {
  constructor() {
    super();
    this.middlewares = [];
  }

  use(name, fn) {
    this.middlewares.push({ name, fn });
    return this; // สำหรับ chaining
  }

  async execute(context) {
    this.emit('start', context);

    for (const { name, fn } of this.middlewares) {
      try {
        this.emit('before', name, context);
        await fn(context);
        this.emit('after', name, context);
      } catch (err) {
        this.emit('error', err, name, context);
        return;
      }
    }

    this.emit('complete', context);
  }
}

// ตัวอย่าง: Request processing pipeline
const pipeline = new Pipeline();

pipeline
  .use('authenticate', async (ctx) => {
    // จำลองการตรวจสอบ token
    await new Promise(r => setTimeout(r, 10));
    ctx.user = { id: 1, name: 'สมชาย', role: 'admin' };
    console.log('  ✓ Authentication สำเร็จ');
  })
  .use('authorize', async (ctx) => {
    if (ctx.user.role !== 'admin') {
      throw new Error('ไม่มีสิทธิ์เข้าถึง');
    }
    console.log('  ✓ Authorization สำเร็จ');
  })
  .use('validate', async (ctx) => {
    if (!ctx.data) {
      throw new Error('ข้อมูลไม่ครบถ้วน');
    }
    console.log('  ✓ Validation สำเร็จ');
  })
  .use('process', async (ctx) => {
    ctx.result = `ประมวลผลข้อมูล: ${ctx.data}`;
    console.log('  ✓ Processing สำเร็จ');
  });

pipeline.on('start', (ctx) => console.log('เริ่ม Pipeline:', ctx));
pipeline.on('complete', (ctx) => console.log('Pipeline เสร็จสมบูรณ์:', ctx.result));
pipeline.on('error', (err, stage) => console.error(`ข้อผิดพลาดใน ${stage}:`, err.message));

pipeline.execute({ data: 'ข้อมูลสำคัญ' });
```

---

## Memory Leak Prevention {#memory-leak-prevention}

### ปัญหา Memory Leak จาก EventEmitter

```javascript
const EventEmitter = require('events');

// ❌ ปัญหา: เพิ่ม listener ซ้ำๆ โดยไม่ลบ
function createLeaky() {
  const emitter = new EventEmitter();

  // ทุกครั้งที่เรียก function นี้ จะเพิ่ม listener ใหม่
  // แต่ไม่ลบ listener เก่า
  setInterval(() => {
    emitter.on('event', () => {
      // handler ที่เพิ่มขึ้นเรื่อยๆ
    });
  }, 100);

  return emitter;
}
```

```javascript
// Node.js จะแจ้งเตือนเมื่อ listeners เกิน maxListeners
// (default: 10)
const emitter = new EventEmitter();

for (let i = 0; i < 15; i++) {
  emitter.on('test', () => {});
}
// คำเตือน: MaxListenersExceededWarning: Possible EventEmitter memory leak
// detected. 11 test listeners added. Use emitter.setMaxListeners() to
// increase limit
```

### วิธีป้องกัน Memory Leak

```javascript
const EventEmitter = require('events');

// 1. ตั้ง maxListeners ให้เหมาะสม
const emitter = new EventEmitter();
emitter.setMaxListeners(20); // เพิ่มขีดจำกัด

// หรือตั้งเป็น 0 = ไม่จำกัด (ระวัง memory leak!)
emitter.setMaxListeners(0);

// 2. ใช้ once() แทน on() เมื่อต้องการ handler ครั้งเดียว
function waitForEvent(emitter, event) {
  return new Promise((resolve, reject) => {
    // ✅ ใช้ once แทน on เพื่อป้องกัน accumulation
    emitter.once(event, resolve);
    emitter.once('error', reject);
  });
}

// 3. ลบ listener เมื่อไม่ใช้
class Component {
  constructor(emitter) {
    this.emitter = emitter;
    // เก็บ reference ของ handler ไว้เพื่อลบ
    this._onData = this._handleData.bind(this);
    this._onError = this._handleError.bind(this);

    this.emitter.on('data', this._onData);
    this.emitter.on('error', this._onError);
  }

  _handleData(data) {
    console.log('ข้อมูล:', data);
  }

  _handleError(err) {
    console.error('ข้อผิดพลาด:', err);
  }

  // ✅ ลบ listeners เมื่อ component ถูก destroy
  destroy() {
    this.emitter.off('data', this._onData);
    this.emitter.off('error', this._onError);
    console.log('Component destroyed, listeners ถูกลบแล้ว');
  }
}

// 4. ใช้ WeakRef สำหรับ listener objects
const EventEmitter = require('events');

class SafeListener {
  constructor(emitter, event, handler) {
    this.weakRef = new WeakRef(handler);
    
    this.wrapper = (...args) => {
      const fn = this.weakRef.deref();
      if (fn) {
        fn(...args);
      } else {
        // Handler ถูก garbage collect แล้ว ลบ listener
        emitter.off(event, this.wrapper);
      }
    };
    
    emitter.on(event, this.wrapper);
  }
}
```

### การตรวจจับ Memory Leak

```javascript
// ติดตาม memory usage
const monitorMemory = () => {
  const used = process.memoryUsage();
  console.log({
    rss: `${Math.round(used.rss / 1024 / 1024)} MB`,
    heapUsed: `${Math.round(used.heapUsed / 1024 / 1024)} MB`,
    heapTotal: `${Math.round(used.heapTotal / 1024 / 1024)} MB`,
    external: `${Math.round(used.external / 1024 / 1024)} MB`
  });
};

// ตรวจสอบจำนวน listeners
const EventEmitter = require('events');
const emitter = new EventEmitter();

// Override emit เพื่อ debug
const originalEmit = emitter.emit.bind(emitter);
emitter.emit = function(event, ...args) {
  const count = this.listenerCount(event);
  if (count > 10) {
    console.warn(`⚠️ Event "${event}" มี ${count} listeners - อาจเป็น memory leak`);
  }
  return originalEmit(event, ...args);
};
```

---

## Practical Examples: Logger System {#practical-logger}

```javascript
const EventEmitter = require('events');
const fs = require('fs');
const path = require('path');

// ระดับ log
const LOG_LEVELS = {
  DEBUG: 0,
  INFO: 1,
  WARN: 2,
  ERROR: 3,
  FATAL: 4
};

class Logger extends EventEmitter {
  constructor(options = {}) {
    super();
    this.name = options.name || 'App';
    this.level = LOG_LEVELS[options.level] || LOG_LEVELS.DEBUG;
    this.transports = [];
    this._setupDefaultTransports(options);
  }

  _setupDefaultTransports(options) {
    // Console transport
    if (options.console !== false) {
      this.addTransport(new ConsoleTransport());
    }

    // File transport
    if (options.file) {
      this.addTransport(new FileTransport(options.file));
    }
  }

  addTransport(transport) {
    this.transports.push(transport);
    // เชื่อม events ระหว่าง Logger กับ Transport
    this.on('log', (entry) => transport.write(entry));
    return this;
  }

  _log(level, message, meta = {}) {
    if (LOG_LEVELS[level] < this.level) return;

    const entry = {
      timestamp: new Date().toISOString(),
      level,
      name: this.name,
      message,
      meta,
      pid: process.pid
    };

    this.emit('log', entry);
    this.emit(`log:${level.toLowerCase()}`, entry);

    if (level === 'FATAL') {
      this.emit('fatal', entry);
    }

    return entry;
  }

  debug(message, meta) { return this._log('DEBUG', message, meta); }
  info(message, meta)  { return this._log('INFO', message, meta); }
  warn(message, meta)  { return this._log('WARN', message, meta); }
  error(message, meta) { return this._log('ERROR', message, meta); }
  fatal(message, meta) { return this._log('FATAL', message, meta); }

  // สร้าง child logger
  child(context) {
    const child = new Logger({ name: `${this.name}:${context}`, console: false });
    // ส่ง logs ต่อไปยัง parent
    child.on('log', (entry) => this.emit('log', { ...entry, context }));
    return child;
  }
}

class ConsoleTransport {
  write(entry) {
    const colors = {
      DEBUG: '\x1b[36m',  // Cyan
      INFO:  '\x1b[32m',  // Green
      WARN:  '\x1b[33m',  // Yellow
      ERROR: '\x1b[31m',  // Red
      FATAL: '\x1b[35m'   // Magenta
    };
    const reset = '\x1b[0m';
    const color = colors[entry.level] || '';

    const metaStr = Object.keys(entry.meta).length > 0
      ? ` ${JSON.stringify(entry.meta)}`
      : '';

    console.log(
      `${color}[${entry.timestamp}] ${entry.level} [${entry.name}] ${entry.message}${metaStr}${reset}`
    );
  }
}

class FileTransport {
  constructor(filePath) {
    this.filePath = filePath;
    this.stream = fs.createWriteStream(filePath, { flags: 'a' });
  }

  write(entry) {
    this.stream.write(JSON.stringify(entry) + '\n');
  }
}

// ตัวอย่างการใช้งาน
const logger = new Logger({
  name: 'MyApp',
  level: 'DEBUG',
  console: true,
  // file: './app.log'  // เปิด comment เพื่อบันทึกไฟล์
});

// ฟัง events พิเศษ
logger.on('log:error', (entry) => {
  // ส่ง alert เมื่อมี error
  console.log('📧 ส่ง Alert Email สำหรับ error:', entry.message);
});

logger.on('fatal', (entry) => {
  // บันทึก fatal error แบบพิเศษ
  console.log('🚨 FATAL ERROR - ต้องการการดูแลทันที!');
  // process.exit(1);
});

// ใช้งาน
logger.debug('เริ่มต้นโปรแกรม', { version: '1.0.0' });
logger.info('Server เริ่มทำงาน', { port: 3000 });
logger.warn('Memory ใกล้เต็ม', { used: '85%' });
logger.error('Database connection failed', { host: 'localhost', port: 5432 });

// Child logger
const dbLogger = logger.child('Database');
dbLogger.info('Connected to database');
dbLogger.debug('Query executed', { sql: 'SELECT * FROM users', rows: 50 });
```

---

## Practical Examples: PubSub System {#practical-pubsub}

```javascript
const EventEmitter = require('events');

// Publish-Subscribe System
class PubSub extends EventEmitter {
  constructor() {
    super();
    this._subscriptions = new Map();
    this._channels = new Map();
    this._messageHistory = new Map();
    this._historySize = 100;
  }

  // Subscribe ไปยัง channel
  subscribe(channel, subscriber, handler) {
    if (!this._subscriptions.has(subscriber)) {
      this._subscriptions.set(subscriber, new Map());
    }

    const subscriberChannels = this._subscriptions.get(subscriber);

    // Unsubscribe เก่าก่อน (ถ้ามี)
    if (subscriberChannels.has(channel)) {
      this.unsubscribe(channel, subscriber);
    }

    // สร้าง wrapper handler
    const wrappedHandler = (message) => handler(message);
    subscriberChannels.set(channel, wrappedHandler);

    // ลงทะเบียน listener
    this.on(`channel:${channel}`, wrappedHandler);

    // เพิ่ม subscriber ใน channel list
    if (!this._channels.has(channel)) {
      this._channels.set(channel, new Set());
    }
    this._channels.get(channel).add(subscriber);

    this.emit('subscribed', { channel, subscriber });

    // ส่ง message ที่ค้างอยู่ (history)
    const history = this._messageHistory.get(channel) || [];
    history.forEach(msg => handler({ ...msg, fromHistory: true }));

    return () => this.unsubscribe(channel, subscriber);
  }

  // Unsubscribe จาก channel
  unsubscribe(channel, subscriber) {
    const subscriberChannels = this._subscriptions.get(subscriber);
    if (!subscriberChannels) return false;

    const handler = subscriberChannels.get(channel);
    if (!handler) return false;

    this.off(`channel:${channel}`, handler);
    subscriberChannels.delete(channel);

    const channelSubscribers = this._channels.get(channel);
    if (channelSubscribers) {
      channelSubscribers.delete(subscriber);
      if (channelSubscribers.size === 0) {
        this._channels.delete(channel);
      }
    }

    this.emit('unsubscribed', { channel, subscriber });
    return true;
  }

  // Publish ไปยัง channel
  publish(channel, data, options = {}) {
    const message = {
      id: `msg_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      channel,
      data,
      timestamp: new Date().toISOString(),
      publisher: options.publisher || 'anonymous'
    };

    // บันทึก history
    if (options.saveHistory !== false) {
      if (!this._messageHistory.has(channel)) {
        this._messageHistory.set(channel, []);
      }
      const history = this._messageHistory.get(channel);
      history.push(message);
      if (history.length > this._historySize) {
        history.shift(); // ลบ message เก่าสุด
      }
    }

    const subscriberCount = this._channels.get(channel)?.size || 0;
    this.emit(`channel:${channel}`, message);
    this.emit('published', { ...message, subscriberCount });

    return message.id;
  }

  // Publish ไปยังหลาย channels (broadcast)
  broadcast(channels, data, options = {}) {
    return channels.map(channel => this.publish(channel, data, options));
  }

  // ดูสถิติ
  getStats() {
    const stats = {
      totalChannels: this._channels.size,
      totalSubscribers: 0,
      channels: {}
    };

    this._channels.forEach((subscribers, channel) => {
      stats.channels[channel] = {
        subscribers: subscribers.size,
        history: (this._messageHistory.get(channel) || []).length
      };
      stats.totalSubscribers += subscribers.size;
    });

    return stats;
  }
}

// ตัวอย่างการใช้งาน: ระบบแชท
const pubsub = new PubSub();

// Monitor events
pubsub.on('subscribed', ({ channel, subscriber }) => {
  console.log(`📌 ${subscriber} subscribe ${channel}`);
});
pubsub.on('unsubscribed', ({ channel, subscriber }) => {
  console.log(`📌 ${subscriber} unsubscribe ${channel}`);
});
pubsub.on('published', ({ channel, subscriberCount, data }) => {
  console.log(`📤 Publish → ${channel} (${subscriberCount} ผู้รับ):`, data.text || data);
});

// สร้าง subscribers
const unsubAlice = pubsub.subscribe('general', 'Alice', (msg) => {
  if (!msg.fromHistory) console.log(`  Alice ได้รับ [${msg.channel}]: ${msg.data.text}`);
});

const unsubBob = pubsub.subscribe('general', 'Bob', (msg) => {
  if (!msg.fromHistory) console.log(`  Bob ได้รับ [${msg.channel}]: ${msg.data.text}`);
});

pubsub.subscribe('news', 'Alice', (msg) => {
  if (!msg.fromHistory) console.log(`  Alice ได้รับข่าว: ${msg.data.headline}`);
});

// Publish messages
pubsub.publish('general', { text: 'สวัสดีทุกคน!' }, { publisher: 'System' });
pubsub.publish('general', { text: 'ยินดีต้อนรับ!' }, { publisher: 'Admin' });
pubsub.publish('news', { headline: 'ข่าวสำคัญประจำวัน' }, { publisher: 'News Bot' });

// Bob ออกจาก general
setTimeout(() => {
  unsubBob();
  pubsub.publish('general', { text: 'Bob ออกไปแล้ว แต่ Alice ยังอยู่' });
  console.log('\nสถิติ:', JSON.stringify(pubsub.getStats(), null, 2));
}, 100);
```

---

## แบบฝึกหัด {#แบบฝึกหัด}

### แบบฝึกหัดที่ 1: Task Queue with Events

สร้าง `TaskQueue` class ที่ใช้ EventEmitter และมีฟีเจอร์:
- `add(task)` - เพิ่ม task เข้าคิว
- `process()` - ประมวลผล task แบบ async
- Events: `taskAdded`, `taskStarted`, `taskCompleted`, `taskFailed`, `queueEmpty`
- รองรับ concurrency (ทำงานหลาย task พร้อมกันได้)
- มี retry mechanism เมื่อ task ล้มเหลว

```javascript
// โครงสร้างที่ควรได้
class TaskQueue extends EventEmitter {
  constructor(options = {}) {
    super();
    this.concurrency = options.concurrency || 1;
    this.maxRetries = options.maxRetries || 3;
    this.queue = [];
    this.running = 0;
  }

  add(taskFn, options = {}) {
    // TODO: เพิ่ม task เข้าคิว
    // TODO: emit 'taskAdded'
  }

  _processNext() {
    // TODO: ดึง task จากคิวมาทำงาน
    // TODO: จัดการ concurrency
    // TODO: retry เมื่อล้มเหลว
  }
}

// ทดสอบ
const queue = new TaskQueue({ concurrency: 2, maxRetries: 2 });

queue.on('taskCompleted', ({ id, result, duration }) => {
  console.log(`✅ Task ${id} เสร็จ (${duration}ms):`, result);
});

queue.on('taskFailed', ({ id, error, retries }) => {
  console.log(`❌ Task ${id} ล้มเหลว (ลองแล้ว ${retries} ครั้ง):`, error.message);
});

queue.on('queueEmpty', () => {
  console.log('✨ ทุก task เสร็จสิ้น');
});
```

### แบบฝึกหัดที่ 2: Event-Driven Chat Room

สร้างระบบ Chat Room ด้วย EventEmitter:
- รองรับหลาย room
- ส่ง private message ได้
- มีระบบ typing indicator
- บันทึก message history
- ระบบ ban user

### แบบฝึกหัดที่ 3: Monitoring System

สร้าง monitoring system ที่:
- ตรวจสอบ CPU, Memory, Disk ทุก X วินาที
- ปล่อย event เมื่อ metric เกิน threshold
- มี alert levels: WARNING, CRITICAL
- รองรับ custom thresholds
- บันทึก log เมื่อเกิด alert

---

## สรุป

ในบทนี้เราได้เรียนรู้:

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| EventEmitter | Class พื้นฐาน, methods สำคัญ |
| Custom Events | การออกแบบ events ที่ดี, Symbol events |
| Built-in Events | process, HTTP, Stream events |
| Event Patterns | State Machine, Observer, Queue, Middleware |
| Memory Leak | การป้องกันและตรวจจับ |
| Logger | ระบบ logging ด้วย events |
| PubSub | Publish-Subscribe pattern |

---

## ก้าวต่อไป

➡️ **Part 07: Streams และ Buffers** - เรียนรู้การจัดการข้อมูลขนาดใหญ่อย่างมีประสิทธิภาพ

---
*Node.js/Express.js Course - Part 06 of 20*
*ขั้นตอนที่ 501-600 จาก 1000*
